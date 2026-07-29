---
title: "Standing Up TNP: Kubernetes & Observability (Part 4)"
published: false
tags: kubernetes, prometheus, grafana, devops
series: Standing Up TNP
canonical_url: https://github.com/tnptommy/standing-up-tnp/blob/main/posts/04-kubernetes-observability.md
---

> 🧭 This is Part 4 of **Standing Up TNP**. Catch up on [Part 3 — Self-Hosted CI/CD](#) if you missed it.

## TL;DR

- k3d — a 4-node cluster (1 server, 3 agents) running entirely inside Docker on `devbox`, no separate VMs needed.
- The full observability stack — Prometheus, Grafana, Alertmanager, Loki, Promtail — deployed via Helm, in dedicated namespaces.
- Two failures that had nothing to do with Kubernetes itself: a k3d flag that silently broke the kubeconfig, and an inotify limit that took the host down before it took a single pod down.
- Self-signed HTTPS via cert-manager, because `tnp.internal` is a domain nobody's issuing real certificates for.

---

## Why k3d instead of a "real" cluster

`devbox` didn't need a fleet of VMs to get a working multi-node Kubernetes cluster — k3d runs each node as a Docker container, which means a 4-node cluster is four containers, not four virtual machines. For a lab that's meant to teach Kubernetes concepts rather than simulate exact production topology, that trade-off is the right one: fast to create, fast to tear down, and light enough to coexist with everything else already running on the same box.

```bash
k3d cluster create tnp-cluster --servers 1 --agents 3
k3d kubeconfig merge tnp-cluster --kubeconfig-switch-context
kubectl get nodes
```

```
NAME                       STATUS   ROLES           VERSION
k3d-tnp-cluster-agent-0    Ready    <none>          v1.36.2+k3s1
k3d-tnp-cluster-agent-1    Ready    <none>          v1.36.2+k3s1
k3d-tnp-cluster-agent-2    Ready    <none>          v1.36.2+k3s1
k3d-tnp-cluster-server-0   Ready    control-plane   v1.36.2+k3s1
```

### The flag that looked helpful and wasn't

The first attempt included `--port "6443:6443@loadbalancer"`, on the assumption that explicitly mapping the API server port was the responsible thing to do. It broke the cluster instead:

```
kubectl get nodes
dial tcp 0.0.0.0:34423: connect: connection refused
```

`docker ps` showed the loadbalancer container correctly publishing `6443`, but the kubeconfig k3d generated pointed at `34423` — a port nothing was actually bound to. The `--port` flag overrides how k3d exposes the loadbalancer, but it doesn't reconcile that override with the port the API server address in the kubeconfig actually needs. Deleting the cluster and recreating it *without* that flag fixed it immediately — k3d's default port handling is correct on its own; the "helpful" override was the bug.

```bash
k3d cluster delete tnp-cluster
k3d cluster create tnp-cluster --servers 1 --agents 3
k3d kubeconfig merge tnp-cluster --kubeconfig-switch-context
```

---

## Helm v4, not v3

Helm hit a major version bump in late 2025 — v3 is maintenance-only now, v4 is current. Building from source instead of installing a stale binary meant catching that shift naturally:

```bash
git clone https://github.com/helm/helm.git ~/src/helm
cd ~/src/helm
git checkout v4.2.3
make build
sudo cp bin/helm /usr/local/bin/helm
```

Nothing about the observability stack below needed v3-specific behavior, so this was a non-event in practice — worth mentioning mainly because most tutorials floating around still assume v3 by default.

---

## Prometheus, Grafana, Alertmanager

```bash
kubectl create namespace monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=<redacted>
```

One chart, three components, all namespaced together. This is the point where a lab starts to feel like it has an actual nervous system — every subsequent thing deployed to the cluster is now something Grafana can show you the health of.

## Loki + Promtail — and the failure that took down more than Loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set promtail.enabled=true
```

Three of four Promtail pods immediately went into CrashLoopBackOff:

```
level=error msg="error creating promtail" error="failed to make file target manager: too many open files"
```

The obvious suspect — `fs.file-max` — turned out to be a red herring; the host had 2 million file descriptors available and was using roughly 2,300 of them. The actual limit was `fs.inotify.max_user_instances`, defaulted to **128** on Ubuntu. That's an instance count, not a watch count — every process that opens an inotify handle (kubelet, containerd, each Promtail pod tailing logs) consumes one, and 128 disappears fast once a handful of pods are running.

It got worse before the fix: partway through investigating, `sudo systemctl daemon-reload` itself started failing with `Failed to allocate directory watch: Too many open files` — confirming the whole host, not just Promtail, was starved of inotify instances.

```bash
echo 'fs.inotify.max_user_instances=1024' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
kubectl delete pod -n monitoring -l app=promtail
```

```bash
cat /proc/sys/fs/inotify/max_user_instances   # 1024
kubectl get pods -n monitoring | grep promtail   # all 1/1 Running
```

> If you're running Kubernetes locally with several DaemonSets — k3d, minikube, kind, doesn't matter which — and you hit an unexplained CrashLoopBackOff on anything that watches files, check `max_user_instances` before anything else. It's a five-second check that saves an hour of chasing the wrong metric.

---

## A Helm chart conflict that looked like a Grafana bug

Once Loki was stable, Grafana itself started CrashLoopBackOff — a much stranger failure, because it had been running fine minutes earlier:

```
Datasource provisioning error: datasource.yaml config is invalid.
Only one datasource per organization can be marked as default
```

`kube-prometheus-stack`'s own datasource ConfigMap was clean — Prometheus as the only default, Alertmanager alongside it, no conflict. The second offender turned out to be a ConfigMap the `loki-stack` chart creates on its own:

```bash
kubectl get configmap -n monitoring -l grafana_datasource=1
# kube-prometheus-stack-grafana-datasource
# loki-loki-stack   <- the extra one
```

```bash
kubectl get configmap -n monitoring loki-loki-stack -o yaml
# isDefault: true   <- conflicts with Prometheus
```

Grafana's sidecar auto-discovers *every* ConfigMap in the namespace carrying the `grafana_datasource: "1"` label — not just the one belonging to the chart that "owns" Grafana. `loki-stack` ships its own, `isDefault: true`, regardless of `grafana.enabled=false` — that flag only skips installing a second Grafana instance, it does nothing to stop the chart from provisioning its own datasource.

```bash
helm upgrade loki grafana/loki-stack \
  --namespace monitoring \
  --reuse-values \
  --set grafana.sidecar.datasources.enabled=false

kubectl delete pod -n monitoring -l app.kubernetes.io/name=grafana
```

Loki was then re-added the correct way — through the chart that actually owns Grafana, instead of letting a second chart provision its own competing datasource:

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
  --namespace monitoring --reuse-values -f loki-datasource-values.yaml
```

> Whenever two Helm charts each ship a Grafana-sidecar-discoverable ConfigMap, assume they'll collide on `isDefault` until proven otherwise. Wiring additional data sources through the chart that already owns Grafana is the safer default, not just the fix for this one collision.

---

## HTTPS for a domain nobody can issue certificates for

`tnp.internal` doesn't exist on the public internet, which means Let's Encrypt is a non-starter — there's no way to prove domain ownership for something that isn't resolvable outside a host-only network. The answer is a self-signed CA, issued locally via cert-manager:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

```yaml
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
    - hosts: [grafana.tnp.internal]
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
                port: {number: 80}
```

```bash
kubectl get certificate -n monitoring   # READY: True
```

Visiting `https://grafana.tnp.internal` still triggers a browser warning — expected and correct, since no public CA vouches for a certificate issued for a domain that only exists on a private network. Click through, and everything from that point on is real TLS, not an unencrypted fallback.

---

## What's running by the end of this part

```
monitoring/         Prometheus, Grafana, Alertmanager, Loki, Promtail
ingress-nginx/      Ingress controller
cert-manager/       Self-signed ClusterIssuer + certs
```

Four k3d nodes, one Helm-managed observability stack, HTTPS on internal services that will never touch a real CA. Small, but it's the layer everything from here on gets measured against — the next thing deployed into this cluster (`tnp-pay-api`, `tnp-pay-web`) inherits metrics, logs, and dashboards for free.

**Next up: Part 5 — Infrastructure as Code, with a Real Target**
Terraform against an actual AWS account, because a lab that only ever runs `terraform apply` against local containers never teaches the parts that matter in production.
