# Standing Up TNP

A hands-on, enterprise-realistic DevOps lab — built around **TNP Technologies**, a fictional fintech company, instead of a pile of isolated tool demos.

Most homelab tutorials teach Docker, then Kubernetes, then Prometheus, each in its own silo. This lab does something different: everything is built to serve a fictional but coherent business — a digital banking platform with real constraints (compliance, audit trails, environment isolation, security gates) — so every tool has an actual job to do, not just a checkbox to tick.

📖 **Full write-up series:** [Standing Up TNP on dev.to](#) — 11 parts, from bare infrastructure to chaos engineering *(link added as parts are published)*

---

## The company

**TNP Technologies JSC** — a fictional fintech / digital banking platform, ~150 employees, hybrid infrastructure (on-prem Vietnam for compliance + AWS for everything else).

**Product:** TNP Pay — a small digital wallet app

| Component | Role |
|---|---|
| `tnp-pay-web` | Frontend — Nginx serving static assets |
| `tnp-pay-api` | Backend service |
| `tnp-pay-db` | PostgreSQL, also used for Flyway migration practice |

**Engineering org (fictional, used to make RBAC mean something):**

| Team | Responsibility |
|---|---|
| Platform Engineering | Owns infrastructure |
| Payment Engineering | Ships TNP Pay to dev/staging |
| QA/QE Team | Test automation & quality gates |
| Security Team | Vulnerability scanning & policy |
| SRE On-call | Incident response |
| Release Manager | Approves production deploys |

---

## The lab

Runs entirely on a single Windows 11 host — 64GB RAM, VMware Workstation Pro, no cloud dependency for the core learning loop.

```
Windows Host (64GB RAM, VMware Workstation Pro)
│
├── devbox      Docker · k3d · Terraform · Ansible
│               Prometheus · Grafana · Loki
│
├── cicd        GitLab CE · Harbor · SonarQube
│
└── 4x Linux VMs   Ansible targets
    ├── web-rocky / db-rocky    Rocky Linux 10
    └── web-ubuntu / db-ubuntu  Ubuntu 26.04 LTS
```

AWS Free Tier is used as the real provisioning target for Terraform — because a lab that only ever runs `terraform apply` against local containers never teaches the parts that actually matter in production.

---

## Series roadmap

| Part | Title | Status |
|---|---|---|
| 0 | Introduction: Why I Built an Enterprise-Realistic DevOps Lab | ✅ Draft ready |
| 1 | Infrastructure Foundation | ✅ Draft ready |
| 2 | Multi-OS Configuration Management with Ansible | ✅ Draft ready |
| 3 | Building a Self-Hosted CI/CD Platform | ✅ Draft ready |
| 4 | Kubernetes & Observability | ✅ Draft ready |
| 5 | Infrastructure as Code, with a Real Target | ⏳ Not started |
| 6 | Shift-Left Security: Designing the DevSecOps Pipeline | ⏳ Not started |
| 7 | Supply Chain Security: SBOM, Image Signing & Policy Gates | ⏳ Not started |
| 8 | GitOps at Scale with ArgoCD | ⏳ Not started |
| 9 | Enterprise Practices: RBAC, SLO & Chaos Engineering | ⏳ Not started |
| 10 | Retrospective: What Building TNP Actually Taught Me | ⏳ Not started |

---

## Repository structure

```
standing-up-tnp/
├── posts/              the accompanying article series (Markdown, dev.to-ready)
├── docs/               setup log and troubleshooting notes (raw, unpolished)
├── terraform/          IaC modules (tnp-infra)
├── ansible/             playbooks and inventory (tnp-ansible)
├── k8s-manifests/       Kubernetes manifests / GitOps configs
└── README.md
```

---

## Disclaimer

TNP Technologies, TNP Pay, and everyone on the org chart above are fictional. This is a personal homelab project for learning purposes, not a real company or product.
