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
