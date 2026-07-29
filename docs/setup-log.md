# Setup Log — devbox

Raw command log for standing up `devbox`, in the actual order things were run.
This is reference material for rebuilding the lab or writing the `posts/` series —
not polished prose. See `docs/troubleshooting.md` for issues hit along the way.

---

## 1. Host prep (Windows)

Disabled Hyper-V entirely to run VMware Workstation Pro natively:

```powershell
bcdedit /set hypervisorlaunchtype off

dism /online /disable-feature /featurename:Microsoft-Hyper-V-All /norestart
dism /online /disable-feature /featurename:VirtualMachinePlatform /norestart
dism /online /disable-feature /featurename:Microsoft-Windows-Subsystem-Linux /norestart
```

Also disabled **Core Isolation → Memory Integrity** in Windows Security
(re-enables a hypervisor silently otherwise). Rebooted, then verified:

```powershell
systeminfo | findstr /i "Hyper-V"
```

Workstation Pro memory setting: **Edit → Preferences → Memory** →
"Fit all virtual machine memory into reserved host RAM", set to 56GB.

---

## 2. VM creation — devbox

- Custom (advanced) → Ubuntu 64-bit
- 10 vCPU, 32768 MB RAM
- NVMe disk, 300GB, thin provisioned (not pre-allocated)
- Two NICs: VMnet8 (NAT) + VMnet1 (host-only, `192.168.10.0/24`)

`.vmx` additions before first boot:

```ini
ethernet0.virtualDev = "vmxnet3"
ethernet1.virtualDev = "vmxnet3"
disk.EnableUUID = "TRUE"
mainMem.useNamedFile = "FALSE"
```

### Ubuntu 26.04 LTS install

- Storage: LVM full disk — expanded `ubuntu-lv` to use all ~298GB
  (installer defaults to 100GB and leaves the rest unallocated; had to
  manually resize the logical volume before continuing)
- No swap partition created here — added as a swap **file** post-install instead
- Hostname: `devbox`
- Username: `tnpadmin`
- **Ticked "Install OpenSSH server"** — required, easy to miss
- Static IP on the VMnet1 interface: `192.168.10.10/24`

---

## 3. Post-install base setup

```bash
sudo apt update && sudo apt upgrade -y
```

### Swap file (16GB)

```bash
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### SSH key (for later Ansible use against the four lab VMs)

```bash
ssh-keygen -t ed25519 -C "tnpadmin@devbox" -f ~/.ssh/id_ed25519 -N ""
```

---

## 4. Docker Engine

Installed via the official script — **not** built from source. Docker/containerd
are treated as an exception to the "build from source" rule for this lab (too
many moving parts, high version-mismatch risk, low learning value relative to
time cost).

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker tnpadmin
# logout/login for group membership to take effect
docker run hello-world
```

---

## 5. k3d + kubectl (built from source / official binaries — simple Go tools)

### Go toolchain

```bash
wget https://go.dev/dl/go1.24.0.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.24.0.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
```

### k3d — built from source

```bash
git clone https://github.com/k3d-io/k3d.git ~/src/k3d
cd ~/src/k3d
git checkout v5.9.0   # latest tag at time of build
make build
sudo cp bin/k3d /usr/local/bin/k3d
```

### kubectl — official upstream binary

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Cluster creation

```bash
k3d cluster create tnp-cluster --servers 1 --agents 3
```

> Do **not** pass `--port "6443:6443@loadbalancer"` here — it broke kubeconfig
> generation (server pointed at a stale random port). See troubleshooting log.

```bash
k3d kubeconfig merge tnp-cluster --kubeconfig-switch-context
kubectl get nodes
```

---

## 6. Helm (built from source)

```bash
git clone https://github.com/helm/helm.git ~/src/helm
cd ~/src/helm
git checkout v4.2.3   # Helm 4 is current stable; v3 is maintenance-only
make build
sudo cp bin/helm /usr/local/bin/helm
helm version
```

---

## 7. kube-prometheus-stack

```bash
kubectl create namespace monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=changeme123
```

---

## 8. Ingress + cert-manager (self-signed, for `*.tnp.internal`)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

Self-signed ClusterIssuer:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF
```

Ingress for Grafana with TLS (`grafana.tnp.internal`):

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-issuer
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - grafana.tnp.internal
      secretName: grafana-tls
  rules:
    - host: grafana.tnp.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-stack-grafana
                port:
                  number: 80
EOF
```

Access via port-forward into the Ingress Controller and a `hosts` entry on the
Windows host pointing `grafana.tnp.internal` at `192.168.10.10`. Self-signed
cert means the browser will warn — expected, click through.

---

## 9. Loki + Promtail

```bash
helm repo add grafana https://grafana.github.io/helm-charts

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set promtail.enabled=true
```

Hit a CrashLoopBackOff on 3/4 Promtail pods here — see
`docs/troubleshooting.md` for the `fs.inotify.max_user_instances` fix.

Added Loki as a Grafana data source manually (`http://loki:3100`),
verified log ingestion via **Explore** with query `{namespace="monitoring"}`.

---

## Status as of this log

- [x] devbox created, sized, networked
- [x] Docker Engine
- [x] k3d cluster `tnp-cluster` (1 server, 3 agents)
- [x] Helm v4.2.3
- [x] kube-prometheus-stack (Prometheus, Grafana, Alertmanager)
- [x] Ingress-nginx + cert-manager, self-signed HTTPS for Grafana
- [x] Loki + Promtail
- [ ] cicd VM (GitLab CE, Harbor, SonarQube)
- [ ] rocky-01/02, ubuntu-01/02 (Ansible lab targets)
- [ ] Terraform + AWS Free Plan target
- [ ] DevSecOps pipeline (gitleaks, Trivy, checkov/tfsec wired into CI)

---

# Setup Log — cicd

Same format as above. VM created identically to devbox (see Part 1 /
`posts/01-infrastructure-foundation.md`), just resized: 20GB RAM, 6 vCPU,
150GB disk, static IP `192.168.10.11`, hostname `cicd`.

## Post-install base setup

```bash
sudo apt update && sudo apt upgrade -y

# Swap — 10GB (50% of RAM, same ratio used on devbox)
sudo fallocate -l 10G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Applied the inotify fix from devbox proactively this time —
# GitLab (Sidekiq, Puma, Gitaly) also needs plenty of inotify watches
echo 'fs.inotify.max_user_instances=1024' | sudo tee -a /etc/sysctl.conf
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker tnpadmin
```

## GitLab CE

```bash
sudo mkdir -p /srv/gitlab/config /srv/gitlab/logs /srv/gitlab/data

sudo docker run --detach \
  --hostname gitlab.tnp.internal \
  --publish 8443:443 --publish 8080:80 --publish 2222:22 \
  --name gitlab \
  --restart always \
  --volume /srv/gitlab/config:/etc/gitlab \
  --volume /srv/gitlab/logs:/var/log/gitlab \
  --volume /srv/gitlab/data:/var/opt/gitlab \
  --shm-size 256m \
  gitlab/gitlab-ce:latest
```

Self-signed cert for `gitlab.tnp.internal`, then inside the container:

```bash
docker exec -it gitlab bash
vi /etc/gitlab/gitlab.rb
# external_url 'https://gitlab.tnp.internal'
# nginx['redirect_http_to_https'] = true
# nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.tnp.internal.crt"
# nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.tnp.internal.key"
exit
docker exec -it gitlab gitlab-ctl reconfigure
```

Post-setup: disabled public sign-up, created group `tnp-technologies`,
created 4 projects (`tnp-pay-api`, `tnp-pay-web`, `tnp-infra`,
`tnp-ansible`). On each: protected `main` (no one can push directly,
merge via MR only), merge method set to "merge commit with semi-linear
history" + automatic rebase, "Pipelines must succeed" and "All threads
must be resolved" enabled as merge gates.

> Note: GitLab CE (free) does not include Approval Rules (requiring N
> reviewers) — that's a Premium/Ultimate feature. The two merge checks
> above are the closest CE equivalent.

Verified end-to-end: SSH key for `tnpadmin` added to GitLab, cloned
`tnp-pay-api` from `devbox` over SSH (port 2222, configured via
`~/.ssh/config`), confirmed direct push to `main` is rejected
(`pre-receive hook declined`).

## Harbor v2.15.2

```bash
cd ~
wget https://github.com/goharbor/harbor/releases/download/v2.15.2/harbor-online-installer-v2.15.2.tgz
tar xzvf harbor-online-installer-v2.15.2.tgz
cd harbor

sudo mkdir -p /srv/harbor/ssl /srv/harbor/data
cd /srv/harbor/ssl
# see troubleshooting.md — first attempt used CN-only cert, had to redo
# with SAN via an openssl config file
cd ~/harbor

cp harbor.yml.tmpl harbor.yml
# hostname: harbor.tnp.internal
# http.port: 80, https.port: 443 (kept as defaults — GitLab uses 8443/8080,
# no conflict, and skipping the port in every docker/curl command is nicer)

sudo ./install.sh
```

Created project `tnp-pay` (private). Verified with a real push:

```bash
docker login harbor.tnp.internal
docker pull hello-world
docker tag hello-world harbor.tnp.internal/tnp-pay/hello-world:test
docker push harbor.tnp.internal/tnp-pay/hello-world:test
```

## SonarQube (Community Build)

```bash
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

sudo mkdir -p /srv/sonarqube/data /srv/sonarqube/logs /srv/sonarqube/extensions /srv/sonarqube/postgresql
```

`docker-compose.yml`:

```yaml
services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    depends_on:
      - db
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: <redacted>
    volumes:
      - /srv/sonarqube/data:/opt/sonarqube/data
      - /srv/sonarqube/logs:/opt/sonarqube/logs
      - /srv/sonarqube/extensions:/opt/sonarqube/extensions
    ports:
      - "9000:9000"
    restart: always

  db:
    image: postgres:17
    container_name: sonarqube-db
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: <redacted>
      POSTGRES_DB: sonar
    volumes:
      - /srv/sonarqube/postgresql:/var/lib/postgresql/data
    restart: always
```

Hit the UID/permission CrashLoop here — see `troubleshooting.md`. Fixed with
`chown -R 1000:1000` on the mounted directories, then:

```bash
docker compose up -d
```

Created project `tnp-pay-api` as a **local project** (not "Import from
GitLab" — that path needs the GitLab self-signed cert to be trusted by
SonarQube first, unnecessary friction for a repo that doesn't have code
yet). Generated an analysis token, installed the scanner:

```bash
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

npm install -g @sonar/scan
```

> The package is `@sonar/scan`, but the actual binary it installs is
> `sonar-scanner-npm` — not `sonar` or `sonar-scanner`. Check
> `package.json`'s `bin` field if a fresh install ever renames it again.

Ran a connectivity test from `devbox` (repo only had a README at this
point, so 0 files analyzed — the point was just confirming the scanner
reaches the server):

```bash
sonar-scanner-npm \
  -Dsonar.host.url=http://192.168.10.11:9000 \
  -Dsonar.token=<redacted> \
  -Dsonar.projectKey=tnp-pay-api
```

Result: `ANALYSIS SUCCESSFUL`.

## Status as of this log

- [x] GitLab CE — HTTPS, 4 repos, branch protection, merge gates verified
- [x] Harbor — HTTPS (SAN cert), project `tnp-pay`, push verified
- [x] SonarQube — project `tnp-pay-api`, scanner connectivity verified
- [ ] Real backend code for `tnp-pay-api` (Node.js 22 + TypeScript + Express)
- [ ] Real frontend code for `tnp-pay-web` (React/Vue, served via Nginx)
- [ ] GitLab CI pipeline wiring SonarQube + Trivy + Harbor push
- [ ] rocky-01/02, ubuntu-01/02 (Ansible lab targets)
- [ ] Terraform + AWS Free Plan target

---

# Setup Log — Ansible lab (web-rocky, web-ubuntu, db-rocky, db-ubuntu)

Renamed from the OS-only scheme (rocky-01/02, ubuntu-01/02) to a
role+OS scheme before cloning — keeps the multi-OS learning value (name
still tells you Rocky vs Ubuntu) while adding role context.

## Templates

`tmpl-rocky10` (Rocky Linux 10 minimal) and `tmpl-ubuntu26`
(Ubuntu 26.04 minimal), each 2GB RAM / 2 vCPU / 16GB disk, NAT only.

Prepped identically before snapshotting:

```bash
# Rocky
sudo dnf install -y python3 open-vm-tools   # NOT qemu-guest-agent — that's
sudo systemctl enable --now vmtoolsd         # for QEMU/KVM, not VMware

# Ubuntu
sudo apt install -y python3 open-vm-tools
sudo cloud-init clean --logs
```

Both, before shutdown + snapshot `base`:

```bash
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo mkdir -p /var/lib/dbus            # dir didn't exist on Rocky by default
sudo ln -sf /etc/machine-id /var/lib/dbus/machine-id
sudo rm -f /etc/ssh/ssh_host_*          # regenerated fresh on next boot
```

SSH public key for `tnpadmin` (from devbox) copied into
`~/.ssh/authorized_keys` on each template before snapshotting.

## Cloning + networking

Linked-cloned 4 VMs from the two `base` snapshots, added a second NIC
(VMnet1) to each, then per VM:

```bash
# Rocky (nmcli)
sudo hostnamectl set-hostname web-rocky
sudo nmcli con mod <iface> ipv4.addresses 192.168.10.21/24
sudo nmcli con mod <iface> ipv4.method manual
sudo nmcli con up <iface>

# Ubuntu (netplan)
sudo hostnamectl set-hostname web-ubuntu
# edit /etc/netplan/50-cloud-init.yaml, add the vmnet1 iface block
sudo netplan apply
```

| VM | OS | IP |
|---|---|---|
| web-rocky | Rocky Linux 10 | 192.168.10.21 |
| web-ubuntu | Ubuntu 26.04 | 192.168.10.22 |
| db-rocky | Rocky Linux 10 | 192.168.10.23 |
| db-ubuntu | Ubuntu 26.04 | 192.168.10.24 |

Added all four to `/etc/hosts` on devbox, cicd, and the Windows host.

## Ansible setup (on devbox)

```bash
sudo apt install -y pipx
pipx ensurepath
pipx install --include-deps ansible   # core 2.21.2
```

`~/tnp-ansible-work/inventory.ini`:

```ini
[rocky]
web-rocky ansible_host=192.168.10.21
db-rocky ansible_host=192.168.10.23

[ubuntu]
web-ubuntu ansible_host=192.168.10.22
db-ubuntu ansible_host=192.168.10.24

[all:vars]
ansible_user=tnpadmin
ansible_python_interpreter=/usr/bin/python3
```

```bash
ssh-keyscan -H 192.168.10.21 >> ~/.ssh/known_hosts
ssh-keyscan -H 192.168.10.22 >> ~/.ssh/known_hosts
ssh-keyscan -H 192.168.10.23 >> ~/.ssh/known_hosts
ssh-keyscan -H 192.168.10.24 >> ~/.ssh/known_hosts

ansible all -i inventory.ini -m ping   # all 4 SUCCESS
```

Hit the sudo-rs prompt-matching issue on both Ubuntu hosts here — see
`troubleshooting.md`. Fixed with manual NOPASSWD setup per host, then ran
the first real playbook (`site.yml`): installs Nginx, opens the firewall
(firewalld on Rocky, ufw on Ubuntu), deploys a per-host index page.
`PLAY RECAP` came back `failed=0` across all four hosts, verified with
`curl` against each IP showing the correct hostname/OS/distribution.

---

# Setup Log — tnp-pay-api (first real code)

Stack: Node.js 22 + TypeScript + Express + `pg`, on `devbox`.

```bash
cd ~/tnp-pay-api
npm init -y
npm install express pg dotenv
npm install -D typescript @types/node @types/express ts-node-dev
```

Swapped `ts-node-dev` for `tsx` immediately — `npm audit` showed 5 high
severity vulnerabilities, all originating from `ts-node-dev`'s outdated
`rimraf`/`glob`/`minimatch`/`brace-expansion` chain:

```bash
npm uninstall ts-node-dev
npm install -D tsx
npm audit   # 0 vulnerabilities after the swap
```

Structure:

```
src/
├── config/db.ts     # pg Pool
├── routes/health.ts # GET /api/health
└── index.ts
```

Hit the `dotenv.config()` ordering bug here (see `troubleshooting.md`) —
had it only in `index.ts`, `db.ts`'s `Pool` was constructed with an empty
`process.env` because ES Module imports are hoisted above other top-level
code. Fixed by calling `dotenv.config()` at the top of `db.ts` directly.

PostgreSQL via a separate compose file, not bundled with the app:

```yaml
# docker-compose.db.yml
services:
  tnp-pay-db:
    image: postgres:17
    environment:
      POSTGRES_DB: tnp_pay
      POSTGRES_USER: tnp_pay
      POSTGRES_PASSWORD: '<redacted>'   # single-quoted — value has $ in it
    ports:
      - "5432:5432"
    volumes:
      - ~/tnp-pay-db-data:/var/lib/postgresql/data
```

Hit a password-mismatch issue between `.env` and the compose file — see
`troubleshooting.md`. Root cause was partly a stale volume (changed
`POSTGRES_PASSWORD` after first init, container ignored the new value)
and partly an unquoted `$` in the password. Fixed by wiping the volume and
re-syncing both files with a single, quoted value.

**Verified end to end:**

```bash
curl http://localhost:3000/api/health
# {"status":"ok","service":"tnp-pay-api","db":"connected"}
```

## Status as of this log

- [x] devbox, cicd fully set up
- [x] 4 Ansible lab VMs, first playbook run successfully
- [x] `tnp-pay-api` — real code, connects to Postgres, `/health` verified
- [x] Committed `tnp-pay-api` to GitLab via MR (branch protection verified —
      direct push to `main` rejected as expected)
- [x] Real SonarQube scan against actual TypeScript code
- [x] Docker image built and pushed to Harbor
      (`harbor.tnp.internal/tnp-pay/tnp-pay-api:v0.1.0`)
- [x] Fixed missing `@types/pg` (caught by `tsc` in the Docker build, not
      by `tsx` in dev mode) via a second MR
- [ ] `tnp-pay-web` frontend
- [ ] `.gitlab-ci.yml` wiring build → scan → push into an actual pipeline
- [ ] Terraform + AWS Free Plan target

Manual chain now verified end to end: write code on `devbox` → push →
GitLab MR (branch protection + merge gates enforced) → SonarQube scan →
`docker build` → `docker push` to Harbor. Next step is automating this
chain instead of running each stage by hand.
