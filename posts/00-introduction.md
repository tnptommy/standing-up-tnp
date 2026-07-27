---
title: "Building TNP: Why I Built an Enterprise-Realistic DevOps Lab (Part 0)"
published: false
tags: devops, kubernetes, terraform, homelab
series: Standing Up TNP
canonical_url: https://github.com/tnptommy/standing-up-tnp/blob/main/posts/00-introduction.md
---

> 🧭 **This is Part 0 of a 10-part series** documenting a self-built, enterprise-realistic DevOps lab — modeled after a fictional fintech company, not a pile of unrelated tools.

## TL;DR

- Most homelab tutorials teach tools in isolation. This series teaches them **working together**, inside a fictional company with real constraints.
- Meet **TNP Technologies** — a fictional fintech with 150 employees, hybrid infrastructure, and a product called **TNP Pay**.
- The lab runs entirely on **one 64GB Windows machine**, no cloud dependency for the core loop.
- **10 parts**, from infrastructure setup to chaos engineering — each one builds on the last.

---

## The problem with most homelab tutorials

Most DevOps homelab guides follow the same pattern:

```
install Docker → install Kubernetes → install Prometheus → done
```

You end up with a pile of tools running side by side. Each one works. None of them talk to each other the way they would inside a real company.

That's not what a DevOps engineer actually does. The job isn't *"know 15 tools."* It's **"make 15 tools work together to ship and operate software safely."**

So instead of building a lab that teaches tools in isolation, I built one around a fictional company — with fictional constraints, fictional teams, and a fictional product that actually needed to run.

This series documents that build, end to end.

---

## Meet TNP Technologies

**TNP Technologies JSC** is a fictional fintech company — a digital banking platform with about 150 employees and a **hybrid infrastructure**:

| | |
|---|---|
| 🇻🇳 On-premises (Vietnam) | Data residency & compliance for financial data |
| ☁️ AWS | Everything that isn't sensitive |

I picked fintech on purpose. It's not a neutral choice — it forces real constraints onto the lab that a generic *"todo app in Kubernetes"* tutorial never has to deal with:

- 📝 Every infrastructure change needs an **audit trail**
- 🔐 Secrets can't sit around as static values forever
- 🧱 Environments (dev/staging/prod) need to be **genuinely isolated**, not just namespaced by convention
- 🛡️ Code doesn't reach production without passing **security scans**
- ⏱️ Uptime matters enough to justify writing **runbooks** and running **failure drills**

> If a lab can survive those constraints, the skills transfer to almost any serious engineering org — fintech or not.

### The product: TNP Pay

A small digital wallet app with three components:

| Component | Role |
|---|---|
| `tnp-pay-web` | Frontend — Nginx serving static assets |
| `tnp-pay-api` | Backend service — connects to the database |
| `tnp-pay-db` | PostgreSQL — also used for migration practice with Flyway |

It's deliberately simple. The point was never to build a complicated app — it's to have *something real* moving through the pipeline, so every tool in the lab has an actual job to do instead of sitting there as a checkbox.

---

## The team that doesn't exist (but shapes everything)

Here's a confession: TNP Technologies has no employees. It's just me, one laptop, and a lot of `kubectl apply`. But the moment I started setting up RBAC, I ran into the same wall every solo builder hits — **"admin for everything" isn't a security model, it's the absence of one.** You can't design real access control for a company of one, because there's nothing to control access *from*.

So I invented an org chart. Six roles, on paper, with nobody behind most of them:

| Team | Responsibility | Access level |
|---|---|---|
| **Platform Engineering** | Owns infrastructure | Cluster-admin, GitLab Owner |
| **Payment Engineering** | Ships TNP Pay to dev/staging | Developer, namespace-scoped |
| **QA/QE Team** | Owns test automation & quality gates | Read-only on SonarQube reports |
| **Security Team** | Owns vulnerability scanning & policy | Read-only on Trivy/gitleaks, manages OPA policy |
| **SRE On-call** | Handles incidents, keeps the lights on | Read-only on Prometheus/Grafana, break-glass only |
| **Release Manager** | Approves production deploys | Approve-only, no direct deploy rights |

A few of these were tempting to merge. QA and Security look like the same "quality" bucket at first glance — until you remember they answer different questions. QA asks *"does this work?"* Security asks *"can this be abused?"* Different concerns, different owners, in any compliance-driven org. Fold them together and you've quietly decided that a working feature and a secure feature are the same thing. They're not.

**Release Manager** was the role I almost cut, and glad I didn't. The whole point of that seat is uncomfortable by design: the person who writes the infrastructure code should *not* be the person who waves it into production. Merge those two hats and you've built a system where the fastest way to ship a mistake is to be really good at your job.

None of this changes what commands I type. I'm still the one running every playbook. But designing *as if* six different people with six different incentives were watching over my shoulder turned RBAC from a checkbox — `verbs: ["*"]`, ship it — into an actual design problem: **who should be able to do what, and why.** That question doesn't go away just because the org chart is fictional.

---

## What the stack looks like

The whole lab runs on a **single Windows machine** — 64GB RAM, VMware Workstation Pro, no cloud dependency for the core learning loop.

```
Windows Host (64GB RAM, VMware Workstation Pro)
│
├── devbox      Docker · k3d · Terraform · Ansible
│               Prometheus · Grafana · Loki
│
├── cicd        GitLab CE · Harbor · SonarQube
│
└── 4x Linux VMs   Ansible targets
    ├── rocky-01 / rocky-02    Rocky Linux 9
    └── ubuntu-01 / ubuntu-02  Ubuntu 24.04
```

The four Ansible targets are deliberately **not identical** — two Rocky, two Ubuntu — so playbooks have to actually handle differences: SELinux vs AppArmor, firewalld vs ufw, dnf vs apt.

AWS enters the picture only where it has to: Terraform needs something real to provision against, and *"real"* here means an actual cloud account, not another local container pretending to be one.

---

## What this series covers

Eleven parts, roughly in the order I actually built things:

1. **Infrastructure Foundation** — hypervisor choice, VM sizing, networking
2. **Multi-OS Configuration Management** — Ansible across Rocky and Ubuntu
3. **Self-Hosted CI/CD** — GitLab CE, Harbor, SonarQube
4. **Kubernetes & Observability** — k3d, Prometheus, Grafana, Loki
5. **Infrastructure as Code, with a Real Target** — Terraform against AWS, MinIO remote state
6. **Shift-Left Security** — gitleaks, SAST, dependency & IaC scanning
7. **Supply Chain Security** — SBOM, image signing with Cosign, policy gates with OPA
8. **GitOps at Scale** — ArgoCD, multi-environment promotion
9. **Enterprise Practices** — RBAC, SLOs, runbooks, chaos engineering
10. **Retrospective** — what actually worked, what I'd do differently, and what this lab still can't teach you

Each part stands on its own, but they build on each other the same way a real platform does — nothing here is a toy example rebuilt from scratch each time.

---

## What this series won't pretend to be

This is a homelab, not a production system, and I'm not going to write as if it is. There's no real user traffic, no real incident history, no real compliance audit.

What it *does* give you is the **shape** of how these pieces fit together — usually the hardest part to learn from documentation alone, because official docs explain one tool at a time and never show you the seams.

> If you're learning DevOps and tired of tutorials that stop the moment a `kubectl apply` succeeds, this series is for the part that comes after that — where the real questions start.

---

**Next up: Part 1 — Infrastructure Foundation**
Why I ended up on VMware Workstation instead of Hyper-V or nested ESXi, and how I sized four VMs against a single 64GB machine without any of them starving the others.
