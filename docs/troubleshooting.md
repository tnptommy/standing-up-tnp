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
