---
title: "Standing Up TNP: Building a Self-Hosted CI/CD Platform (Part 3)"
published: false
tags: gitlab, cicd, docker, devops
series: Standing Up TNP
canonical_url: https://github.com/tnptommy/standing-up-tnp/blob/main/posts/03-self-hosted-cicd.md
---

> 🧭 This is Part 3 of **Standing Up TNP**. Catch up on [Part 2 — Multi-OS Ansible](#) if you missed it.

## TL;DR

- GitLab CE, Harbor, and SonarQube — all self-hosted on one VM (`cicd`), all behind self-signed HTTPS.
- Getting a GitLab Runner to actually pick up and pass a job took longer than getting GitLab itself running: DNS inside job containers, a cert reconfigure that silently didn't apply, and a Docker-in-Docker daemon that doesn't inherit anything from its host.
- End state: a real `.gitlab-ci.yml` that installs, type-checks, scans with SonarQube, and pushes a Docker image to a private registry — triggered automatically on every merge to `main`.

---

## Three services, one VM, one recurring lesson

`cicd` runs three things: GitLab CE (source control, merge requests, CI orchestration), Harbor (container registry, with Trivy scanning built in), and SonarQube (static analysis). Getting each one installed was mechanical. Getting them to *trust each other's certificates* was where most of the real time went — and it's worth calling out up front, because the same root cause bit three separate times in three separate disguises.

### GitLab CE

```bash
sudo docker run --detach \
  --hostname gitlab.tnp.internal \
  --publish 8443:443 --publish 8080:80 --publish 2222:22 \
  --name gitlab --restart always \
  --volume /srv/gitlab/config:/etc/gitlab \
  --volume /srv/gitlab/logs:/var/log/gitlab \
  --volume /srv/gitlab/data:/var/opt/gitlab \
  --shm-size 256m \
  gitlab/gitlab-ce:latest
```

Configured for HTTPS via `gitlab.rb`, self-signed cert generated the same way as everything else in this lab. Created the `tnp-technologies` group, four repos, and — the part that actually matters for an "enterprise-realistic" lab — turned on branch protection: no direct pushes to `main`, merge only, pipeline must succeed, all threads resolved.

> **GitLab CE doesn't have Approval Rules** (requiring N reviewers) — that's a Premium/Ultimate feature. "Pipelines must succeed" + "all threads resolved" is the closest CE-native equivalent, and honestly a decent one for a solo lab.

### Harbor

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.15.2/harbor-online-installer-v2.15.2.tgz
```

First cert attempt used only a Common Name (`-subj "/CN=harbor.tnp.internal"`). Browsers tolerate that. Docker's TLS stack (Go's, underneath) does not, and refuses with:

```
x509: certificate relies on legacy Common Name field, use SANs instead
```

Regenerating with an explicit SAN fixed the *file*:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout harbor.tnp.internal.key -out harbor.tnp.internal.crt \
  -config harbor-openssl.cnf -extensions v3_req
```

But swapping the cert file on disk didn't fix the *running service* — Harbor caches its runtime config via a `prepare` step, and skipping it means nginx keeps serving whatever it loaded at last actual startup:

```bash
cd ~/harbor
sudo ./prepare
sudo docker compose down && sudo docker compose up -d
```

This is lesson #1, and it repeats below in a different shape: **regenerating a cert file is not the same as making the service that serves it aware of the change.**

### SonarQube

```yaml
# docker-compose.yml
services:
  sonarqube:
    image: sonarqube:community
    volumes:
      - /srv/sonarqube/data:/opt/sonarqube/data
      - /srv/sonarqube/logs:/opt/sonarqube/logs
      - /srv/sonarqube/extensions:/opt/sonarqube/extensions
```

CrashLoopBackOff on first boot:

```
Failed to create temporary configuration directory [/opt/sonarqube/data/es8/config]
```

The host directories were created with `sudo mkdir` — owned by root. SonarQube's embedded Elasticsearch runs as UID `1000`, not root, and can't write into a root-owned bind mount.

```bash
sudo chown -R 1000:1000 /srv/sonarqube/data /srv/sonarqube/logs /srv/sonarqube/extensions
```

Same underlying category of mistake as the cert issue above, different layer: **the thing you configured on disk isn't automatically the thing the container can actually use.** Always check UID/GID against what's inside the image, not what created the host directory.

---

## Getting a GitLab Runner to work — the actual hard part

Everything above was mechanical once you knew the fix. Getting CI to run a job end to end took a genuinely long troubleshooting chain, and it's worth walking through in order because each fix revealed the next problem.

### 1. The job container can't resolve GitLab's hostname

```bash
sudo docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

Registration worked. Running an actual job didn't:

```
fatal: unable to access 'https://gitlab.tnp.internal/...':
Could not resolve host: gitlab.tnp.internal
```

Passing `--add-host=gitlab.tnp.internal:192.168.10.11` to the `gitlab-runner` container itself does nothing here, because **the job runs in a separate, sibling container** that the runner spins up per-job — it doesn't inherit host entries from its parent. The fix lives in the runner's own Docker executor config:

```toml
# config.toml
[runners.docker]
  extra_hosts = ["gitlab.tnp.internal:192.168.10.11", "harbor.tnp.internal:192.168.10.11"]
```

### 2. TLS hostname mismatch, caused by a missing port in `external_url`

Fixed DNS, hit a new error:

```
SSL: no alternative certificate subject name matches target hostname 'gitlab.tnp.internal'
```

GitLab generates the clone URL for CI jobs from `external_url` in `gitlab.rb`. At the time, that was set without the `:8443` port the container actually exposes — so the job tried the wrong port, hit whatever was (or wasn't) listening on 443, and got a cert that didn't match.

### 3. Fixing the port broke the port mapping

Adding `:8443` to `external_url` made GitLab's internal nginx start listening on `8443` *inside the container* — but the Docker port mapping was still `8443(host) → 443(container)`. Now nothing was listening where the mapping expected. `Connection refused`.

The actual fix: keep `external_url` without a port, and force nginx to listen on 443 explicitly regardless:

```ruby
external_url 'https://gitlab.tnp.internal'
nginx['listen_port'] = 443
nginx['listen_https'] = true
```

Three fixes, three different failure modes, one underlying pattern: **changing one config value can silently move where a service listens, and the fix that seems obvious (add the port everywhere) creates a new mismatch instead of resolving the old one.**

### 4. Runner concurrency starving new jobs

Several earlier failed pipelines were still sitting in "Running" indefinitely. `concurrent = 1` in the runner config meant exactly one job slot, permanently occupied by zombies from earlier debugging. New jobs queued forever.

```toml
concurrent = 4
```

```bash
sudo docker container prune -f
```

### 5. Docker-in-Docker doesn't trust anything the host trusts

Job container passes DNS, GitLab connects — then the actual build stage:

```
Cannot connect to the Docker daemon at tcp://docker:2375
```

Missing `DOCKER_HOST`/`DOCKER_TLS_CERTDIR` for the DinD service:

```yaml
variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_TLS_CERTDIR: ""
```

Fixed that, hit `docker login` failing with `certificate signed by unknown authority` against Harbor. Copying the Harbor cert into the *client* container's `/etc/docker/certs.d/` (the fix that worked for `devbox` weeks earlier) did nothing here — because the actual TLS verification happens in the **DinD daemon container**, a completely separate process from the client issuing `docker login`. It never reads the client's cert store.

The fix that actually matches the architecture: tell the daemon itself to skip verification for this one internal registry, rather than trying to get a cert into a container that isn't the one doing the verifying.

```yaml
services:
  - name: docker:27-dind
    command: ["--insecure-registry=harbor.tnp.internal"]
```

---

## The pipeline that finally worked

```yaml
stages: [install, test, scan, push]

install:
  stage: install
  image: node:22-alpine
  script: [npm ci]
  artifacts:
    paths: [node_modules/]

typecheck:
  stage: test
  image: node:22-alpine
  script: [npm run build]
  dependencies: [install]

sonarqube-check:
  stage: scan
  image: node:22-alpine
  script:
    - npm install -D @sonar/scan
    - npx sonar-scanner-npm -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.token=$SONAR_TOKEN -Dsonar.projectKey=tnp-pay-api
  allow_failure: true
  dependencies: [install]

docker-build-push:
  stage: push
  image: docker:27
  services:
    - name: docker:27-dind
      command: ["--insecure-registry=harbor.tnp.internal"]
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
  before_script:
    - docker login harbor.tnp.internal -u "$HARBOR_USER" -p "$HARBOR_PASSWORD"
  script:
    - docker build -t $HARBOR_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $HARBOR_IMAGE:$CI_COMMIT_SHORT_SHA
  only: [main]
```

`Job succeeded` — four stages, no manual steps. Every merge to `main` now builds, scans, and pushes a versioned image to Harbor on its own.

### One deliberate gap, left in on purpose

`SONAR_TOKEN`, `HARBOR_USER`, and `HARBOR_PASSWORD` are all marked **Protected** in GitLab's CI/CD variables — meaning they're only injected into pipelines running on protected branches (`main`), not on ordinary feature branches. That means `sonarqube-check` reliably fails with a `401` on every feature branch, and only passes after merge.

That's not a bug I left unfixed — it's GitLab correctly refusing to hand a Sonar token to a pipeline running on a branch anyone could have pushed to. The alternative (unprotecting the variables so every branch gets full credentials) is objectively more convenient and objectively worse practice. Keeping it protected means the "pipeline is fully green" moment only ever happens on `main` — which, if you think about what these credentials actually gate, is exactly where that moment should happen.

---

## What actually took the time here

Nothing above was conceptually hard. Every fix was one or two lines. What made this section long was the number of **layers that don't share context with each other by default** — a GitLab container doesn't share DNS with the runner container it triggers; a DinD daemon doesn't share a cert store with the client that logs into it; a `reconfigure` command doesn't necessarily restart the service whose config it just rewrote. None of that is a GitLab or Docker flaw exactly — it's what you get from composing several isolated, disposable environments and expecting them to behave like one machine. Getting comfortable debugging *at the boundary* between those environments turned out to be most of what this part actually taught.

**Next up: Part 4 — Kubernetes & Observability**
Standing up k3d, Prometheus, Grafana, and Loki — and a Grafana datasource conflict that looked like a Helm bug and wasn't.
