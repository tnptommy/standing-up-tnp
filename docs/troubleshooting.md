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
