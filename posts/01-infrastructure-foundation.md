---
title: "Standing Up TNP: Infrastructure Foundation (Part 1)"
published: false
tags: devops, vmware, homelab, virtualization
series: Standing Up TNP
canonical_url: https://github.com/tnptommy/standing-up-tnp/blob/main/posts/01-infrastructure-foundation.md
---

> 🧭 This is Part 1 of **Standing Up TNP** — an enterprise-realistic DevOps lab built around a fictional fintech company. [Start from Part 0](#) if you haven't already.

## TL;DR

- One Windows 11 machine, 64GB RAM, 12–16 cores — no cloud dependency for the core loop.
- **VMware Workstation Pro**, not Hyper-V, not ESXi. Hyper-V is fully disabled.
- Two main VMs (`devbox`, `cicd`) plus four small Ansible targets — sized by actual workload, not guesswork.
- A host-only network with static IPs, because DHCP for a lab you need to reason about is asking for pain later.

---

## The hypervisor decision

The obvious options for a Windows host are Hyper-V (built in, free) or VMware Workstation Pro (also free since late 2024, even for commercial use). I went with Workstation, and it wasn't a close call once I looked at what I actually needed.

Hyper-V and WSL2 share the same virtualization layer. That's convenient right up until you want to run something Hyper-V doesn't handle well — nested virtualization, in particular, gets flaky the moment Hyper-V is also managing WSL2 in the background. Rather than fight that interaction, I disabled Hyper-V entirely and let Workstation run natively.

```powershell
# PowerShell, admin
bcdedit /set hypervisorlaunchtype off

dism /online /disable-feature /featurename:Microsoft-Hyper-V-All /norestart
dism /online /disable-feature /featurename:VirtualMachinePlatform /norestart
dism /online /disable-feature /featurename:Microsoft-Windows-Subsystem-Linux /norestart
```

> ⚠️ **The gotcha almost everyone hits:** Windows Security → Core Isolation → *Memory Integrity* silently re-enables a hypervisor even after `bcdedit` says it's off. If Workstation still feels like it's running through a compatibility layer instead of natively, this is why. Turn it off, then verify:

```powershell
systeminfo | findstr /i "Hyper-V"
```

If that command still lists Hyper-V requirements, it isn't actually off.

The trade-off: no WSL2, no Docker Desktop, no Windows Sandbox. None of that mattered here — every real workload in this lab runs inside Linux VMs anyway, not on the Windows host itself.

---

## Sizing the VMs

64GB sounds like a lot until you start adding up what a realistic DevOps toolchain needs concurrently. Two VMs carry almost all of the weight:

| VM | OS | RAM | vCPU | Disk | Role |
|---|---|---|---|---|---|
| **devbox** | Ubuntu 26.04 LTS | 32GB | 10 | 300GB thin | Docker, k3d, Terraform, Ansible, Prometheus/Grafana/Loki |
| **cicd** | Ubuntu 26.04 LTS | 20GB | 6 | 150GB thin | GitLab CE, Harbor, SonarQube |
| Windows host | — | 8GB | 2 | — | Hypervisor, VS Code Remote-SSH, browser |

Plus four small Ansible targets — deliberately not identical:

| VM | OS | RAM |
|---|---|---|
| rocky-01, rocky-02 | Rocky Linux 10 | 1GB each |
| ubuntu-01, ubuntu-02 | Ubuntu 26.04 LTS | 1GB each |

Total when everything is running: **64GB**, fully used, no headroom left to spare. That's intentional — I'd rather size tight and know exactly where the ceiling is than leave vague slack I can't account for.

### Why devbox is the bigger VM

`devbox` runs two Docker Compose profiles that can be toggled independently:

```yaml
# core profile — always on
k3d server + 3 agents      ~6GB
Prometheus, Grafana, Loki  ~4GB
Alertmanager, exporters    ~1GB

# heavy profile — enabled when needed
Elasticsearch, Logstash, Kibana   ~7GB
Istio, ArgoCD, Vault              ~4GB
```

`core` alone runs comfortably in 11GB. `core + heavy` together land around 22GB — well inside the 32GB budget, with room to spare for build caches and whatever Kubernetes decides to do that day.

### Why cicd is separate from devbox

GitLab CE alone recommends 8GB minimum just for itself. Bundling GitLab, Harbor, and SonarQube into the same VM as the Kubernetes/observability stack would mean either starving one side or overprovisioning both. Splitting them into two VMs means `cicd` can be **shut down entirely** when I'm not doing CI/CD work — freeing 20GB instantly, something a single monolithic VM can't do.

### Why the Ansible targets are 1GB each

They don't need to do anything except exist, accept SSH connections, and run whatever a playbook throws at them. Any more RAM here would be wasted — the point of these VMs is repetition and disposability, not capacity. I use linked clones from a snapshotted template for both distros, so resetting a target back to a clean state takes seconds, not a rebuild.

---

## Networking

Two virtual networks, doing two very different jobs:

| vmnet | Type | Subnet | DHCP | Purpose |
|---|---|---|---|---|
| vmnet8 | NAT | 192.168.100.0/24 | On | Internet access for every VM |
| vmnet1 | Host-only | 192.168.110.0/24 | **Off** | Lab network, static IPs |

Every VM gets two NICs — one on each network. DHCP is deliberately off on the lab network. A lab you're reasoning about (which service talks to which, what's the IP for the GitLab runner, why can't devbox reach cicd) is not a place where you want addresses that might change on you.

```
devbox        192.168.110.10
cicd          192.168.110.11
rocky-01      192.168.110.21
rocky-02      192.168.110.22
ubuntu-01     192.168.110.23
ubuntu-02     192.168.110.24
```

A local `hosts` file entry on the Windows host makes the dashboards reachable by name instead of IP:

```
192.168.110.10  devbox grafana.tnp.internal prom.tnp.internal
192.168.110.11  gitlab.tnp.internal harbor.tnp.internal sonar.tnp.internal
```

Opening `http://grafana.tnp.internal:3000` from Chrome on the host works exactly like it would on a real internal network — no port-forwarding gymnastics required, because host-only networking already gives Windows direct routing to every VM's static IP.

---

## The maintenance detail nobody mentions upfront

Thin-provisioned disks grow and **do not shrink back on their own**. A VM that starts at 300GB "provisioned, thin" will genuinely creep toward that ceiling over months of Docker image churn — pulling images, building, pruning, repeating — even as the actual data inside stays much smaller.

The fix is periodic, not automatic:

```powershell
vmware-vdiskmanager -k "D:\vmware\devbox\devbox.vmdk"
```

Run this from Workstation's install directory with the VM powered off. It's easy to forget until a disk alert shows up somewhere you didn't expect one.

---

## What's next

With the hypervisor decided, the VMs sized, and the network laid out, the next problem is making four VMs that are deliberately *not* identical behave the same way under automation — which is exactly what Ansible is for.

**Next up: Part 2 — Multi-OS Configuration Management with Ansible**
Why Rocky Linux and Ubuntu were chosen specifically for their differences, and the SELinux error almost everyone hits on their first playbook run.
