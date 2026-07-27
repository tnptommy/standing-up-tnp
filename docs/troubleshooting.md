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
