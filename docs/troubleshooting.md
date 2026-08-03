# Troubleshooting Log

Real issues hit while building this lab, kept here for quick lookup and as
raw material for the `posts/` series. Format: symptom → root cause → fix →
verify.

---

## k3d cluster: `kubectl get nodes` fails with "connection refused" after `--port` flag

**Symptom:**

```
kubectl get nodes
E... dial tcp 0.0.0.0:34423: connect: connection refused
```

Cluster showed as created successfully in `k3d cluster create` output, but
`kubectl` couldn't reach it.

**Root cause:** Passed `--port "6443:6443@loadbalancer"` at cluster creation
time. This overrides the port k3d publishes for the load balancer, but the
kubeconfig it generates still points at a different, randomly-assigned port
that was never actually bound. `docker ps` showed `6443` correctly mapped on
the `serverlb` container, while `kubectl config view --minify` showed
`server: https://0.0.0.0:34423` — a port nothing was listening on.

**Fix:** Delete and recreate without the `--port` flag; let k3d handle the
API server port itself.

```bash
k3d cluster delete tnp-cluster
k3d cluster create tnp-cluster --servers 1 --agents 3
k3d kubeconfig merge tnp-cluster --kubeconfig-switch-context
```

**Verify:**

```bash
kubectl config view --minify   # server: should match the port actually bound
kubectl get nodes              # all nodes Ready
```

Ingress/LoadBalancer port exposure for real traffic is handled separately,
later, via an Ingress Controller — not by forcing a port at cluster creation.

---

## Promtail: CrashLoopBackOff — "too many open files"

**Symptom:** After `helm install loki grafana/loki-stack`, 3 of 4 Promtail
pods stuck cycling:

```
kubectl get pods -n monitoring | grep promtail
loki-promtail-7tkfb   0/1   CrashLoopBackOff
loki-promtail-7v9v7   1/1   Running
loki-promtail-glmzd   0/1   CrashLoopBackOff
loki-promtail-p2hfz   0/1   CrashLoopBackOff
```

```
kubectl logs -n monitoring loki-promtail-7tkfb
level=error msg="error creating promtail" error="failed to make file target manager: too many open files"
```

**Root cause:** Not `fs.file-max` (host had 2,097,152, using only ~2,300 —
nowhere close to the limit). The actual bottleneck was
`fs.inotify.max_user_instances`, defaulted to **128** on Ubuntu. Each
container needing an inotify watch (kubelet, containerd, every Promtail pod
tailing logs) consumes an instance, not a watch — and 128 is exhausted fast
once k3d is running a handful of pods. `fs.inotify.max_user_watches` was
already fine at 250,418 and was not the issue.

Symptom escalated further while investigating: `sudo systemctl daemon-reload`
itself started failing with `Failed to allocate directory watch: Too many
open files` — confirming the whole host, not just Promtail, was starved of
inotify instances.

**Fix:**

```bash
echo 'fs.inotify.max_user_instances=1024' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
kubectl delete pod -n monitoring -l app=promtail
```

**Verify:**

```bash
cat /proc/sys/fs/inotify/max_user_instances   # 1024
sudo systemctl daemon-reload                    # no more errors
kubectl get pods -n monitoring | grep promtail  # all 1/1 Running
```

This is a common failure mode running Kubernetes locally (k3d, minikube,
kind) with several DaemonSets that each need their own inotify watch —
worth checking `max_user_instances` early if you see unexplained
CrashLoopBackOff on log-shipping or file-watching pods.

---

## Ubuntu installer: LVM leaves ~200GB unallocated by default

**Symptom:** Storage configuration screen shows `/` sized at only 100GB
despite a 300GB disk, with the remainder sitting as unused "free space" in
the volume group.

**Root cause:** Ubuntu's installer (Subiquity) defaults the LVM logical
volume to a conservative size rather than the full disk, even when
"Use an entire disk" is selected.

**Fix:** On the storage configuration screen, edit the `ubuntu-lv` entry and
either clear the size field (to use all available space) or set it explicitly
to the volume group's full size before confirming.

**Verify:** File system summary should show `/` at the full ~298GB with no
"free space" line left in Available Devices.

---

## Grafana: CrashLoopBackOff — "Only one datasource per organization can be marked as default"

**Symptom:** After installing `loki-stack` alongside `kube-prometheus-stack`
in the same namespace, the Grafana pod goes into CrashLoopBackOff (2/3
containers ready) and stays there indefinitely — deleting the pod doesn't
help, a brand new pod hits the exact same error on next start.

```
kubectl logs -n monitoring <grafana-pod> -c grafana --previous
logger=provisioning ... error="Datasource provisioning error: datasource.yaml
config is invalid. Only one datasource per organization can be marked as default"
```

**Root cause:** Grafana's sidecar auto-discovers *any* ConfigMap labeled
`grafana_datasource: "1"` in the namespace and merges them all at startup —
not just the one belonging to `kube-prometheus-stack`. The `loki-stack` chart
creates its own ConfigMap (`loki-loki-stack`) with the same label and
`isDefault: true`, even when installed with `grafana.enabled=false`. That
flag only skips installing a *second Grafana instance* — it does nothing to
stop the chart from creating its own datasource ConfigMap. Two ConfigMaps,
two datasources both claiming default, Grafana refuses to start.

Confirmed by comparing the two ConfigMaps directly:

```bash
kubectl get configmap -n monitoring -l grafana_datasource=1
# NAME                                       AGE
# kube-prometheus-stack-grafana-datasource   ...
# loki-loki-stack                            ...   <- the extra one

kubectl get configmap -n monitoring loki-loki-stack -o yaml
# isDefault: true   <- conflicts with Prometheus's isDefault: true
```

Restarting the pod alone does not fix this — the ConfigMap causing the
conflict is still there and gets re-read on every startup.

**Fix:** Disable the `loki-stack` chart's own datasource sidecar, since a
second, independently-managed data source ConfigMap is the wrong pattern
here regardless of the default-flag conflict:

```bash
helm upgrade loki grafana/loki-stack \
  --namespace monitoring \
  --reuse-values \
  --set grafana.sidecar.datasources.enabled=false

kubectl delete pod -n monitoring -l app.kubernetes.io/name=grafana
```

Then add Loki as a data source the correct way — through
`kube-prometheus-stack`'s own values, which already owns the datasource
ConfigMap Grafana is provisioned from:

```yaml
# loki-datasource-values.yaml
grafana:
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki:3100
      access: proxy
      isDefault: false
```

```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --reuse-values \
  -f loki-datasource-values.yaml
```

**Verify:**

```bash
kubectl get pods -n monitoring | grep grafana   # 3/3 Running, RESTARTS not climbing
```

Check **Connections → Data sources** in the Grafana UI — Loki should be
listed, `isDefault` unchecked, Prometheus still the default.

**Lesson:** Whenever two Helm charts each ship a Grafana-sidecar-discoverable
ConfigMap (the `grafana_datasource: "1"` label pattern), assume they will
collide on `isDefault` unless proven otherwise. Prefer wiring additional
data sources through the chart that already owns Grafana
(`additionalDataSources` in `kube-prometheus-stack`) instead of installing a
second chart with its own auto-provisioning sidecar.

---

## Harbor: Docker login fails — "certificate relies on legacy Common Name field"

**Symptom:** Browsers accept the self-signed cert for Harbor (after clicking
through the warning), but `docker login harbor.tnp.internal` fails:

```
Error response from daemon: Get "https://harbor.tnp.internal/v2/":
tls: failed to verify certificate: x509: certificate relies on legacy
Common Name field, use SANs instead
```

**Root cause:** The cert was generated with only a Common Name
(`-subj "/CN=harbor.tnp.internal"`), no Subject Alternative Name. Browsers
still tolerate CN-only certs; Go's TLS stack (which Docker's daemon uses)
does not — SAN has been required for hostname verification for years now,
and CN-only certs are silently rejected regardless of trust store.

**Fix:** Regenerate the certificate with an explicit SAN via an OpenSSL
config file:

```bash
cat > harbor-openssl.cnf <<EOF
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
x509_extensions = v3_req

[dn]
CN = harbor.tnp.internal

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = harbor.tnp.internal
IP.1 = 192.168.10.11
EOF

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout harbor.tnp.internal.key \
  -out harbor.tnp.internal.crt \
  -config harbor-openssl.cnf \
  -extensions v3_req
```

**Critical follow-up step, easy to miss:** regenerating the cert file on disk
is not enough. Harbor has a `prepare` step that reads `harbor.yml` and
writes the actual runtime config (including certs) that containers use.
Just restarting containers reuses the stale runtime config:

```bash
cd ~/harbor
sudo ./prepare
sudo docker compose down
sudo docker compose up -d
```

Then re-copy the new cert to any Docker client that needs to trust it:

```bash
# on the client (e.g. devbox)
sudo nano /etc/docker/certs.d/harbor.tnp.internal/ca.crt   # paste new cert
sudo systemctl restart docker
```

**Verify:**

```bash
echo | openssl s_client -connect harbor.tnp.internal:443 \
  -servername harbor.tnp.internal 2>/dev/null \
  | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
# should show DNS + IP entries

docker login harbor.tnp.internal   # Login Succeeded
```

**Lesson:** For any self-hosted service that has a "prepare / generate
config" step in its install process (Harbor, and likely others), changing
the cert file on disk and restarting the container is not sufficient —
the prepare step must be re-run first, or the service keeps serving
whatever it baked into its runtime config at install time.

---

## SonarQube: CrashLoop — "Failed to create temporary configuration directory [/opt/sonarqube/data/es8/config]"

**Symptom:** SonarQube container stuck restarting, logs show Elasticsearch
repeatedly failing to launch:

```
java.lang.IllegalStateException: Failed to create temporary configuration
directory [/opt/sonarqube/data/es8/config]
```

`vm.max_map_count` was already correctly set to 262144 — not the cause here,
despite being the most commonly cited fix for this exact error message online.

**Root cause:** The host directories mounted into the container
(`/srv/sonarqube/data`, `/logs`, `/extensions`) were created with `sudo
mkdir`, so they're owned by `root`. The SonarQube container runs as UID
`1000`, not root, and can't write to root-owned directories — so
Elasticsearch can't create its config directory at startup.

**Fix:**

```bash
sudo chown -R 1000:1000 /srv/sonarqube/data /srv/sonarqube/logs /srv/sonarqube/extensions
docker compose down
docker compose up -d
```

Note: the PostgreSQL data directory (`/srv/sonarqube/postgresql`) does **not**
need this — that container manages its own UID (`999`) internally and
already had correct ownership.

**Verify:**

```bash
docker compose logs -f sonarqube   # wait for "SonarQube is operational"
curl -I http://localhost:9000       # HTTP 200
```

**Lesson:** Any host directory bind-mounted into a container needs its
ownership checked against the UID the container actually runs as — not
assumed to be root or the host user. This bites hardest with JVM-based
images (SonarQube, Elasticsearch, and similar) which very commonly run
as a fixed non-root UID baked into the image.

---

## Ansible: "Timeout waiting for privilege escalation prompt" — Ubuntu only, Rocky fine

**Symptom:** `--ask-become-pass` (or any interactive sudo become) times out
against the two Ubuntu 26.04 lab VMs, every time, with every workaround —
but works immediately against the Rocky 10 VMs on the same inventory, same
playbook, same password:

```
[ERROR]: Task failed: Timeout (12s) waiting for privilege escalation prompt:
```

Ruled out along the way, in order: wrong password (sudo works fine
interactively), stale SSH ControlPersist sockets (cleared `~/.ansible/cp/*`,
no change), parallel fork race condition (`-f 1`, no change), password
passed via `-e ansible_become_pass=...` instead of prompt (no change), MOTD
hanging the SSH session (timed a plain `ssh -tt`, under half a second).

**Root cause:** Ubuntu 26.04 ships **sudo-rs** — Canonical's Rust rewrite of
sudo, made the default starting with 25.10 — instead of GNU sudo. Ansible
sends a custom prompt string via `sudo -S -p "<marker>"` and scans the
output for that exact marker to know when to feed the password. sudo-rs
handles `-p`/`-S` differently: a manual test showed the prompt coming out
wrapped as `[sudo: TESTPROMPT:] Password:` instead of the bare string
Ansible asked for. Ansible's marker-matching logic never sees what it's
looking for, and just waits until it times out — even though the password
is correct and interactive `sudo` works perfectly.

```bash
# confirms the wrapping behavior directly
ssh -tt user@ubuntu-host 'sudo -H -S -p "TESTPROMPT:" whoami'
# prompt appears as: [sudo: TESTPROMPT:] Password:
```

**Fix:** Don't fight sudo-rs's prompt handling — remove the need for
Ansible to parse a sudo prompt at all, by configuring passwordless sudo
directly over a manual SSH session:

```bash
ssh tnpadmin@<ubuntu-host>
echo "tnpadmin ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/tnpadmin
sudo chmod 440 /etc/sudoers.d/tnpadmin
exit
```

Once NOPASSWD is set, Ansible never sends a password through `become` in
the first place, so the sudo-rs prompt-format mismatch becomes irrelevant.

**Verify:**

```bash
ansible all -i inventory.ini -b -m shell -a 'whoami'
# all hosts return root, no --ask-become-pass needed
```

**Lesson:** When an OS swaps a core utility for a rewrite (sudo → sudo-rs,
in this case), tools that scrape specific output formats from that utility
— not just its exit code — can break in ways that look like a connection
or credentials problem, even though the underlying command works fine by
hand. Worth checking "did a core utility change implementation" before
assuming a config or network issue when only one OS in a mixed inventory
misbehaves.

---

## Node/TypeScript: `db: unreachable` despite a correct `.env` file

**Symptom:** `tnp-pay-api`'s `/api/health` endpoint always reports
`"db": "unreachable"`, even after confirming: PostgreSQL container is
running, `psql` connects fine manually, the right port is listening,
`.env` has the correct host/port/credentials, and the dev server was
restarted after every `.env` change.

**Root cause:** `dotenv.config()` was called at the top of `index.ts`, with
`pool = new Pool({...})` living in a separately-imported `db.ts`. In an ES
Module, **all `import` statements are hoisted and execute before any other
top-level code in the file** — regardless of where they appear textually.
So the real execution order was:

```
import express
import dotenv
import healthRouter  →  which imports db.ts  →  new Pool() runs HERE,
                         with process.env still empty
dotenv.config()          ← runs last, too late for db.ts
```

`db.ts`'s `Pool` constructor ran before `dotenv.config()` had populated
`process.env`, so it silently fell back to hardcoded defaults —
never actually reading the real `.env` values.

**Fix:** Call `dotenv.config()` at the very top of every file that reads
`process.env` at module load time, not just the entrypoint:

```typescript
// src/config/db.ts
import { Pool } from 'pg';
import dotenv from 'dotenv';

dotenv.config();   // must be here, not just in index.ts

export const pool = new Pool({ /* ... */ });
```

**Verify:** Add a temporary `console.log` of the resolved config values
right after `dotenv.config()` and confirm they match `.env` — not the
hardcoded fallback defaults.

**Lesson:** In any module that does environment-dependent work at import
time (not inside a function), assume `dotenv.config()` in a *different*
file will not have run yet. Load env in the module that needs it, not just
once at the entrypoint.

---

## PostgreSQL (Docker): "password authentication failed" despite matching `.env`

**Symptom:** `pg` throws `password authentication failed for user "..."`
even though the password in `.env` was carefully copied from the same
place as `POSTGRES_PASSWORD` in `docker-compose.db.yml`.

**Root cause:** Two separate issues, both worth checking:

1. **Stale volume.** `POSTGRES_PASSWORD` is only read once — the first time
   the Postgres container initializes an *empty* data directory. If the
   password in `docker-compose.db.yml` is changed after the container has
   already run once, the running database still has the *original*
   password baked in; the compose file value is now just wrong-but-unused.
2. **Unquoted special characters in YAML.** A generated password containing
   `$` (e.g. `openssl rand -base64` output) can be partially interpreted as
   a shell/YAML variable reference if left unquoted, silently mangling the
   value Postgres actually receives.

**Fix:**

```bash
docker compose -f docker-compose.db.yml down
rm -rf ~/tnp-pay-db-data   # wipes the stale volume — fine for a dev DB with no real data yet
```

Set the same password in both files, single-quoted in the compose file:

```yaml
POSTGRES_PASSWORD: 'gJ7]AdY.u5|EVGyb=[u='
```

```bash
docker compose -f docker-compose.db.yml up -d
```

**Verify:**

```bash
docker exec -it tnp-pay-db psql -U tnp_pay -d tnp_pay -c "SELECT 1;"
curl http://localhost:3000/api/health   # {"status":"ok", ..., "db":"connected"}
```

**Lesson:** Prefer generating passwords without shell/YAML-special
characters for anything going into a compose file or `.env`:

```bash
openssl rand -base64 24 | tr -dc 'a-zA-Z0-9' | head -c 32
```

And remember that changing `POSTGRES_PASSWORD` on an already-initialized
volume is a no-op — the fix is always to reset the volume, not just the
config.

---

## dotenv: unexpected "tip" message in console output — not a supply-chain issue

**Symptom:** `npm run dev` prints an unfamiliar line alongside the expected
`dotenv` output, referencing an external domain:

```
◇ injected env (0) from .env // tip: ⌁ auth for agents [www.vestauth.com]
```

This looks alarming out of context — an unfamiliar domain name printed by
a dependency is a classic supply-chain-compromise red flag, and it's worth
treating that way by default.

**Investigation:** Rather than assume it's malicious or dismiss it,
searched for the exact string across `node_modules`:

```bash
grep -r "vestauth" node_modules/ -l
# node_modules/dotenv/CHANGELOG.md
# node_modules/dotenv/lib/main.js
```

Both hits are inside `dotenv`'s own official package files, not a nested
or typosquatted dependency — confirming the message originates from the
real, official `dotenv` package.

**Conclusion:** Not a compromise. `dotenv`'s maintainer has a known history
of printing promotional "tip" messages for their commercial product
(`dotenvx`) directly in console output. Noisy and arguably bad practice for
a widely-used library, but not malicious.

**Lesson:** Treat unfamiliar strings/domains from a dependency as suspicious
by default, but verify before reacting — `grep` the string across
`node_modules` to see exactly which package prints it and whether it's
coming from the real package's own source (self-promotion) versus an
injected or typosquatted one (compromise). The check takes seconds and
turns a guess into a fact either way.

---

## Docker build: `tsc` fails on missing `@types/pg`, but `npm run dev` never caught it

**Symptom:** `npm run dev` (via `tsx`) works fine against `db.ts` importing
`pg`. The Dockerfile's build stage, running `tsc` via `npm run build`,
fails:

```
src/config/db.ts(1,22): error TS7016: Could not find a declaration file
for module 'pg'. '/app/node_modules/pg/esm/index.mjs' implicitly has an
'any' type.
```

**Root cause:** `tsx` transpiles TypeScript without running a full type
check — it strips types and runs the JavaScript, so a missing `@types/*`
package for a dependency doesn't block it. `tsc` (used for the actual
production build) does full type checking and refuses to compile with an
implicit `any` from an untyped module. `@types/pg` was never installed
alongside `pg` itself.

**Fix:**

```bash
npm install -D @types/pg
```

**Verify:**

```bash
docker build -t harbor.tnp.internal/tnp-pay/tnp-pay-api:v0.1.0 .
# build stage's `npm run build` now succeeds
```

**Lesson:** `tsx`/`ts-node`-style dev runners are not a substitute for
actually running `tsc` before considering code done — they optimize for
fast iteration, not correctness. Building the Docker image (or running
`npm run build` directly) is what actually catches missing type
declarations, strict-mode violations, and other issues dev mode silently
tolerates. Worth doing a real build periodically during development, not
just at the end.

---

## Harbor: all containers except `harbor-log` silently stopped

**Symptom:** `docker login harbor.tnp.internal` fails with
`connect: connection refused` on port 443, despite Harbor having worked
fine in a previous session. `docker ps` on the Harbor host shows only one
container running:

```
harbor-log   Up 2 hours (healthy)
```

Core, db, portal, registry, proxy/nginx, trivy-adapter, redis, and
jobservice are all gone.

**Root cause:** Not fully diagnosed — Harbor's Compose stack stopped on its
own between sessions. Possibly related to host resource pressure, a Docker
daemon restart, or the VM being suspended/resumed rather than cleanly
rebooted (a VM suspend can leave containers in an inconsistent state that
Docker doesn't automatically recover from).

**Fix:**

```bash
cd ~/harbor
sudo docker compose up -d
docker compose ps   # wait for all services to report healthy/running
```

**Verify:**

```bash
docker ps | grep harbor   # should show ~8-9 containers again
```

From another host:

```bash
docker login harbor.tnp.internal   # Login Succeeded
```

**Lesson:** Self-hosted Docker Compose stacks on a lab VM aren't guaranteed
to survive VM suspend/resume or host reboots the same way a properly
`restart: always`-configured systemd-managed service would. Worth checking
`docker ps` on any Compose-based service (Harbor, GitLab if run outside its
own init system, SonarQube) at the start of a session before assuming it's
still up from last time — and worth investigating whether `restart: always`
in the compose file is actually being honored across VM power-cycles, not
just container crashes.

---

## GitLab Omnibus: replacing a self-signed cert doesn't take effect after `reconfigure`

**Symptom:** Same SAN-related TLS error as the earlier Harbor entry, this
time on GitLab: `gitlab-runner register` and `gitlab-runner verify` both
fail with `certificate relies on legacy Common Name field, use SANs
instead`, even after generating a proper SAN cert and running
`gitlab-ctl reconfigure`. `openssl s_client` against the GitLab port
consistently returns nothing usable — the connection isn't presenting the
new cert at all.

**Root cause:** `gitlab-ctl reconfigure` is a Chef run that only touches
services whose *configuration* it detects as changed. Replacing the `.crt`
file on disk via `docker cp` doesn't change `gitlab.rb` itself, so Chef has
nothing to diff against and doesn't restart nginx — the running nginx
process keeps serving whatever certificate it loaded at its last actual
start, regardless of what's now sitting on disk.

**Fix:** Restart nginx explicitly instead of relying on `reconfigure` to
infer that it should:

```bash
docker exec -it gitlab gitlab-ctl restart nginx
```

**Verify:**

```bash
openssl s_client -connect <gitlab-host>:8443 -servername gitlab.tnp.internal \
  </dev/null 2>/dev/null | openssl x509 -noout -text \
  | grep -A 2 "Subject Alternative Name"
```

**Lesson:** Same underlying lesson as the Harbor cert issue, one layer
deeper: `gitlab-ctl reconfigure` is not equivalent to "restart everything
and pick up new files." Chef-driven reconfigure tools infer what needs
restarting from tracked config changes — replacing a file it doesn't watch
(a cert dropped in via `docker cp` rather than referenced by a new
`gitlab.rb` value) can leave the actual running service untouched. When a
cert swap doesn't take effect after the "official" reload step, restart the
specific service directly rather than trusting the higher-level reconfigure
command to have covered it.

---

## GitLab Runner: "is removed" / 403 Forbidden despite a valid-looking token

**Symptom:** `gitlab-runner register` fails with `Verifying runner... is
removed` and a `403 Forbidden`, using a registration token that was copied
from the GitLab UI shortly before.

**Root cause:** Two compounding issues from repeated failed registration
attempts while debugging the TLS problem above:

1. Each interrupted or failed `register` call had still created a runner
   record on the GitLab side (visible under Settings → CI/CD → Runners),
   even though the local `gitlab-runner` container never successfully
   registered. Re-running registration with an old token pointed at a
   runner record that either belonged to a different attempt or had
   already been revoked.
2. Separately, manually retyping a long `glrt-...` token introduced a
   single missing character at the end, producing `Verifying runner... is
   not valid` — a different, more specific error than the 403.

**Fix:**

1. Delete all stale/duplicate runner entries from Settings → CI/CD →
   Runners in the GitLab UI — leave none, to avoid ambiguity about which
   token belongs to which runner record.
2. Create exactly one new runner, and copy the registration token using the
   UI's copy button — never retype it by hand.

```bash
sudo docker exec -it gitlab-runner gitlab-runner register \
  --url https://gitlab.tnp.internal:8443 \
  --token <copied, not typed> \
  --executor docker \
  --docker-image docker:27 \
  --docker-privileged
```

**Verify:**

```bash
sudo docker exec -it gitlab-runner gitlab-runner verify
# "is valid"
```

**Lesson:** GitLab runner authentication tokens (`glrt-...`) are long
enough that manual transcription is a real source of failure — always copy
them programmatically. And when debugging one problem (TLS, in this case)
causes several registration attempts to fail partway through, check for
orphaned runner records left behind by those attempts before assuming a
fresh attempt with a "new" token will behave cleanly — stale, ambiguous
state from earlier failures can produce misleading errors that look
unrelated to the original problem.

---

## HTTPS for SonarQube, and why one self-signed cert per service stopped scaling

**Symptom / motivation:** GitLab and Harbor already had HTTPS via
individually self-signed certificates. Adding a third — SonarQube, sitting
behind an Nginx reverse proxy since it has no built-in TLS termination —
meant a third cert to generate, a third cert to trust on every client, and
a third thing to re-copy every time a runner or host needed to talk to it.

**Decision:** Instead of a fourth self-signed cert, generate one internal
root CA and sign all three services' certificates with it. Trust the CA
once, everywhere, instead of trusting three unrelated certificates.

```bash
# on cicd
mkdir -p ~/tnp-ca && cd ~/tnp-ca
openssl genrsa -out tnp-ca.key 4096
openssl req -x509 -new -nodes -key tnp-ca.key -sha256 -days 3650 \
  -out tnp-ca.crt -subj "/CN=TNP Internal Root CA"

# per service (gitlab, harbor, sonar), signed by the CA instead of self-signed
openssl req -new -key SERVICE.key -out SERVICE.csr -config SERVICE.cnf
openssl x509 -req -in SERVICE.csr -CA tnp-ca.crt -CAkey tnp-ca.key \
  -CAcreateserial -out SERVICE.crt -days 365 -extensions v3_req -extfile SERVICE.cnf
```

Applied to each service (GitLab needs `gitlab-ctl reconfigure` + explicit
nginx restart, same as the Part 3 lesson; Harbor needs `./prepare` re-run;
the SonarQube Nginx proxy just needs a restart), trusted at the OS level on
both `devbox` and `cicd` via `update-ca-certificates`, and — critically —
copied into the GitLab Runner's mounted volume so job containers can trust
it too:

```toml
# config.toml
[runners.docker]
  tls-ca-file = "/etc/gitlab-runner/certs/tnp-ca.crt"   # the CA, not a per-service cert
  volumes = ["/cache", "...", "/srv/gitlab-runner/config/ca:/etc/gitlab-runner/ca:ro"]
```

**A basic prerequisite that's easy to skip:** the reverse proxy for
SonarQube needs its own port, since Harbor already owns 443/80 on the same
host. Nginx also ships a default site listening on 80 by default — leaving
it enabled causes a bind conflict even when the new config only listens on
a different port (9443, in this case). `rm
/etc/nginx/sites-enabled/default` before anything else.

**Lesson:** A single internal CA scales better than N self-signed
certificates the moment N is more than one — every new service reuses the
same trust relationship instead of adding a new one to distribute.

---

## Trusting a self-signed CA inside a CI job container — four layers, four separate failures

**Symptom:** With the CA created and every server-side service updated,
`sonarqube-check` still failed — but each fix revealed a new, different
failure underneath, all inside the same `node:22-alpine` job container.

### Layer 1 — DNS

```
getaddrinfo ENOTFOUND sonar.tnp.internal
```
Same root cause as the Part 3 GitLab/Harbor DNS issue: `extra_hosts` in
`config.toml` needs every internal domain the job might reach, and a newly
added service (SonarQube's new HTTPS domain) doesn't inherit trust from
domains added earlier.

```toml
extra_hosts = ["gitlab.tnp.internal:...", "harbor.tnp.internal:...", "sonar.tnp.internal:..."]
```

### Layer 2 — Alpine doesn't have the destination directory yet

```
cp: can't create '/usr/local/share/ca-certificates/tnp-ca.crt': No such file or directory
```
`node:22-alpine` is minimal enough that `/usr/local/share/ca-certificates/`
doesn't exist until the `ca-certificates` package is actually installed —
copying a cert there before installing the package that owns that
directory fails, unlike on Debian-based images where the path pre-exists.

```bash
apk add --no-cache ca-certificates openjdk21-jre   # first
cp tnp-ca.crt /usr/local/share/ca-certificates/     # then this works
update-ca-certificates
```

### Layer 3 — `keytool` succeeds, then fails, because `$JAVA_HOME` is empty

```
Certificate was added to keystore
keytool error: java.io.FileNotFoundException: /lib/security/cacerts (No such file or directory)
```
The import itself worked — against the wrong (empty-prefixed) path.
Installing `openjdk21-jre` via `apk` doesn't set `$JAVA_HOME`, unlike some
other distros' Java packages. `$JAVA_HOME/lib/security/cacerts` silently
became `/lib/security/cacerts` once `$JAVA_HOME` evaluated to nothing.

```bash
export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))
```

### Layer 4 — Java trusts the CA now; Node.js still doesn't

```
[ERROR] Bootstrapper: ... unable to verify the first certificate
```
`sonar-scanner-npm` has two halves: a Node.js "Bootstrapper" that fetches
and launches the actual Java-based Scanner Engine. Getting the Java half to
trust the CA (Layer 3) does nothing for the Node half — Node ships its own
bundled CA store and doesn't read the OS certificate store by default,
regardless of what `update-ca-certificates` did system-wide.

```yaml
variables:
  NODE_EXTRA_CA_CERTS: /etc/gitlab-runner/ca/tnp-ca.crt
```

**Final correct order**, all four layers together:

```yaml
before_script:
  - apk add --no-cache ca-certificates openjdk21-jre
  - export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))
  - cp /etc/gitlab-runner/ca/tnp-ca.crt /usr/local/share/ca-certificates/tnp-ca.crt
  - update-ca-certificates
  - keytool -import -trustcacerts -alias tnp-ca -file /etc/gitlab-runner/ca/tnp-ca.crt \
      -keystore $JAVA_HOME/lib/security/cacerts -storepass changeit -noprompt
variables:
  NODE_EXTRA_CA_CERTS: /etc/gitlab-runner/ca/tnp-ca.crt
```

**Lesson:** "Trust this CA" is not one operation inside a container running
multiple language runtimes — it's one operation *per runtime*, because
each one (OS/OpenSSL, JVM, Node.js) maintains its own independent
certificate store and none of them automatically defer to the others. A
tool that's secretly two processes in a trenchcoat (Node bootstrapper +
Java engine, in this case) inherits both runtimes' trust requirements, not
either one alone.

---

## Git: `main` looked out of date because it actually was — three false "conflicts" in a row

**Symptom:** After merging a fix, `git log --oneline main -5` on `devbox`
kept showing only `Initial commit` — no sign of any of the merges just
completed on GitLab. Creating a new branch from what looked like `main`
kept producing merge conflicts and MRs stuck "N commits behind," even
immediately after merging.

**Root cause:** Not a Git bug and not a real conflict — `git checkout main`
alone does not fetch or update anything. Every time a new feature branch
was cut with `git checkout -b` from a stale local `main`, it inherited
that staleness, and a subsequent MR against the real `origin/main`
legitimately had diverging history to reconcile.

```bash
git log --oneline main -5   # only ever showed the local, stale main
```

vs. the fix:

```bash
git checkout main
git fetch origin
git reset --hard origin/main   # force local main to match remote exactly
git log --oneline -5           # now shows the real history
```

A second, related trap: after `git branch -D` on a local branch whose
remote counterpart was already deleted by GitLab's "delete source branch"
option, `git push origin --delete` on that same branch name correctly
errors with "remote ref does not exist" — that's not a failure, it's
confirmation the branch is already gone. `git fetch --prune` clears the
stale `remotes/origin/...` references left behind after this kind of
GitLab-side auto-deletion.

**Lesson:** `git checkout <branch>` switches HEAD; it does not sync
anything with the remote. Any workflow that creates branches from "main"
without an explicit `git fetch` + `reset --hard origin/main` (or at
minimum `git pull`) immediately before doing so is building on
potentially stale ground — and the resulting conflicts look like content
problems even though the actual cause is simply an out-of-date local
ref.

