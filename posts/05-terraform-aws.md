---
title: "Standing Up TNP: Infrastructure as Code, with a Real Target (Part 5)"
published: false
tags: terraform, aws, iac, devops
series: Standing Up TNP
canonical_url: https://github.com/tnptommy/standing-up-tnp/blob/main/posts/05-terraform-aws.md
---

> 🧭 This is Part 5 of **Standing Up TNP**. Catch up on [Part 4 — Kubernetes & Observability](#) if you missed it.

## TL;DR

- Terraform against local containers teaches you the HCL syntax. Terraform against a real cloud account teaches you the parts that actually matter — credentials, state, and the ways two independent SDK clients can disagree about who they are.
- MinIO as an S3-compatible remote state backend, self-hosted on `devbox`, because state on a laptop disk is a single point of failure waiting to happen.
- Three failures in a row, each one revealing the next: a Terraform binary that silently wasn't a release build, a registry policy change that broke the "obvious" way to create credentials, and — the one that took the longest to find — two separate AWS SDK clients inside the same Terraform run that don't share configuration with each other.

---

## Why a local-only Terraform lab doesn't teach the real thing

It's possible to learn the entire HCL syntax — resources, variables, modules, outputs — without ever touching a cloud provider, by pointing Terraform at local Docker containers or a mock provider. That's a legitimate way to learn the language. It's also a way to never encounter the problems that actually define working with Terraform professionally: real IAM credentials that can be wrong in several different ways, state that has to live somewhere durable and shared, and provider behavior that only shows up against a real API.

So this part uses an actual AWS account, not a simulation of one.

## Getting an AWS account that isn't free

Signing up hit an unexpected wall immediately: the new account got flagged as ineligible for the Free Plan — *"Your information is associated with an existing or previously registered AWS account"* — and was pushed to a Paid Plan instead. AWS's eligibility check runs on more than just the email address; payment card, phone number, and device fingerprinting all factor in.

The practical difference turned out to be smaller than it sounds: **Paid Plan accounts still receive the same $100–200 in signup credit** as Free Plan. The real difference is what happens when that credit runs out — Free Plan accounts auto-suspend, Paid Plan accounts start billing the card on file. For a lab where resources get destroyed after each session, that's a manageable risk, not a dealbreaker.

Non-negotiable setup before creating a single resource:

```
1. MFA on the root user
2. Budget alerts at $1 and $10
3. A dedicated IAM user for Terraform — never root
```

```bash
aws configure
# Access Key ID / Secret from IAM user "terraform-tnp"
# region: ap-southeast-1
# output: json

aws sts get-caller-identity
```

The IAM user (`terraform-tnp`) got `PowerUserAccess`, not `AdministratorAccess` — full access to build infrastructure, no access to IAM itself. Least privilege, even in a lab of one.

---

## The first module: a VPC that actually exists

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "tnp" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "tnp-${var.environment}-vpc", Project = "TNP" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.tnp.id
  cidr_block               = var.public_subnet_cidr
  map_public_ip_on_launch   = true
}

resource "aws_internet_gateway" "tnp" {
  vpc_id = aws_vpc.tnp.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.tnp.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.tnp.id
  }
}
```

```bash
cd environments/dev
terraform init
terraform plan   # 5 to add
terraform apply  # yes
```

A real VPC, a real subnet, a real internet gateway, sitting in `ap-southeast-1` on an actual AWS account — not a diagram, not a local approximation.

---

## Remote state, and the Terraform release that wasn't one

State living in `terraform.tfstate` on `devbox`'s disk is fine for a single person on a single machine, and wrong for basically everything past that. MinIO, self-hosted and S3-compatible, is the fix:

```bash
docker run -d --name minio --restart always \
  -p 9000:9000 -p 9001:9001 \
  -v ~/minio-data:/data \
  -e "MINIO_ROOT_USER=tnpadmin" \
  -e "MINIO_ROOT_PASSWORD=<redacted>" \
  minio/minio server /data --console-address ":9001"
```

Getting there took a detour through the "build from source" principle this lab has followed for every simple Go CLI tool — and Terraform turned out not to belong in that category, for a subtle reason.

### The `-dev` suffix that wouldn't go away

```bash
git clone https://github.com/hashicorp/terraform.git
cd terraform
go build -o ~/.local/bin/terraform
terraform version
# Terraform v1.17.0-dev
```

`git clone` checks out `main` by default — HashiCorp's active development branch, not a release. The fix looked obvious: find the latest real tag and check it out.

```bash
git tag --sort=-creatordate | head -5
# v1.17.0-alpha20260729
# v1.16.0-beta1
# v1.15.8          <- first one without alpha/beta/dev
```

```bash
git checkout v1.15.8
go build -o ~/.local/bin/terraform
terraform version
# Terraform v1.15.8-dev
```

Still `-dev`. Checking out the exact tag HashiCorp used for their `v1.15.8` release still produced a binary reporting itself as a dev build. The reason: **HashiCorp injects the version string via `ldflags` in their own release pipeline**, not through a plain `go build`. A manually compiled binary from the correct source tag is functionally identical to the release — but it will never claim to be one, because the metadata that says so is added by tooling outside the repository itself.

This is the same category of exception already made for Docker and GitLab CE: multi-step, non-trivial release processes don't compile down to "clone and build." Terraform joined that list — official binaries from `releases.hashicorp.com`, not a self-compiled copy, still the latest version, just not self-built.

```bash
wget https://releases.hashicorp.com/terraform/1.15.8/terraform_1.15.8_linux_amd64.zip
unzip terraform_1.15.8_linux_amd64.zip
mv terraform ~/.local/bin/terraform
terraform version
# Terraform v1.15.8
```

### MinIO's access key policy quietly changed

Creating a scoped access key for Terraform (rather than reusing the MinIO root credential) used to be a UI operation. It no longer is — MinIO Community Edition removed access key creation from the console; that capability now lives behind the paid AIStor tier. The CLI still works:

```bash
mc alias set local http://192.168.10.10:9000 tnpadmin '<root password>'
mc admin accesskey create local
```

Worth calling out as its own small lesson: a tool's *documented* capability can silently move behind a paywall between when a tutorial was written and when you follow it. Checking the CLI path first would have saved a detour through a UI element that no longer exists.

---

## Two AWS clients, one error message, and a fix that has to be applied twice

With a real Terraform binary and a real access key, `terraform init -migrate-state` still failed:

```
Error: Retrieving AWS account details: AWS account ID not previously found
and failed retrieving via all available methods.

* retrieving caller identity from STS: ... InvalidClientTokenId: The
  security token included in the request is invalid.
```

The confusing part: `aws sts get-caller-identity` on the CLI, using the same credentials, worked perfectly — correct account, correct user ARN, no error. Something inside Terraform was authenticating with different (or no) credentials than the CLI was.

Adding `skip_requesting_account_id = true` to the `provider "aws"` block — the fix the error message itself links to — changed nothing. Same error, same request IDs pattern, every time.

The actual structure of the problem: **`provider "aws"` and `backend "s3"` are two entirely separate AWS SDK clients inside the same Terraform run.** The backend isn't a feature of the provider — it's configured independently, authenticates independently, and in recent Terraform versions occasionally probes AWS APIs (like account ID lookup for state locking) on its own initiative. Setting `skip_requesting_account_id` on the provider tells one client to stop asking. It says nothing to the other.

```hcl
# main.tf
provider "aws" {
  region                      = "ap-southeast-1"
  skip_requesting_account_id  = true
}
```

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket                       = "tnp-terraform-state"
    key                          = "dev/terraform.tfstate"
    region                       = "ap-southeast-1"
    endpoints                    = { s3 = "http://192.168.10.10:9000" }
    access_key                   = "<minio access key>"
    secret_key                   = "<minio secret key>"
    skip_credentials_validation  = true
    skip_metadata_api_check      = true
    skip_region_validation       = true
    skip_requesting_account_id   = true   # <- the missing half
    use_path_style                = true
  }
}
```

```bash
rm -rf .terraform .terraform.lock.hcl
terraform init -migrate-state
# Initializing the backend...
# Successfully configured the backend "s3"!
```

```bash
terraform plan
# No changes. Your infrastructure matches the configuration.
```

State now lives in MinIO, verified against real infrastructure the CLI can reach independently of Terraform, matching exactly.

---

## Why this one took longer than it should have

The error message pointed at the provider. The fix belonged to the backend. Nothing about the message distinguished which of the two independent AWS clients was actually failing — and the natural instinct, apply the documented fix, check if it worked, repeat, doesn't surface that distinction either. What did was going back to first principles: if the CLI authenticates fine with these exact credentials, the problem isn't the credentials — it's *something inside Terraform not using them*. That reframing is what led to treating the backend as its own client rather than an extension of the provider.

Worth keeping as a general instinct for infrastructure tooling: when a single command orchestrates multiple network clients under the hood, a shared-looking config block doesn't guarantee shared behavior. Check whether each client is actually reading what you think it's reading before assuming a fix "should" have applied everywhere.

**Next up: Part 6 — Shift-Left Security: Designing the DevSecOps Pipeline**
Wiring gitleaks, dependency scanning, and IaC scanning into the pipeline as blocking stages — not optional reports nobody reads.
