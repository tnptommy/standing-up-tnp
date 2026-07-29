---
title: "Standing Up TNP: Multi-OS Configuration Management with Ansible (Part 2)"
published: false
tags: ansible, linux, devops, automation
series: Standing Up TNP
canonical_url: https://github.com/tnptommy/standing-up-tnp/blob/main/posts/02-multi-os-ansible.md
---

> 🧭 This is Part 2 of **Standing Up TNP**. [Start from Part 0](#) if you're new here, or catch up on [Part 1 — Infrastructure Foundation](#).

## TL;DR

- Four lab VMs, deliberately split across two distro families — two Rocky Linux 10, two Ubuntu 26.04 — to force real cross-platform Ansible.
- Renamed them by **role**, not just OS (`web-rocky`, `db-ubuntu`, etc.) so the inventory reads like a real fleet, not a science experiment.
- Hit a genuinely new failure mode: **Ubuntu 26.04 ships `sudo-rs`**, a Rust rewrite of sudo, and Ansible's privilege-escalation prompt matching doesn't know how to talk to it yet.
- First real playbook — install Nginx, open the firewall, deploy a per-host landing page — ran clean across all four hosts once that was sorted out.

---

## Why bother with two distros at all

It would have been faster to clone four identical Ubuntu VMs and call it a day. That's also exactly the setup that teaches you nothing about writing Ansible that survives contact with a real company, where the fleet is never one distro.

So the four lab targets are split on purpose:

```
web-rocky    Rocky Linux 10     192.168.10.21
web-ubuntu   Ubuntu 26.04 LTS   192.168.10.22
db-rocky     Rocky Linux 10     192.168.10.23
db-ubuntu    Ubuntu 26.04 LTS   192.168.10.24
```

Naming them by role instead of just distro (`web-rocky` rather than `rocky-01`) was a small but deliberate choice — the name now tells you both *what it's for* and *what OS it runs*, without needing a separate lookup table in your head.

## Building the templates once, cloning four times

Each distro got one template VM — `tmpl-rocky10` and `tmpl-ubuntu26` — set up identically in spirit, differently in syntax:

```bash
# Rocky — python3 required for Ansible, open-vm-tools for VMware
# (NOT qemu-guest-agent — that's for QEMU/KVM, not VMware, and the
# service will just sit there failing a dependency it can never satisfy)
sudo dnf install -y python3 open-vm-tools
sudo systemctl enable --now vmtoolsd

# Ubuntu — same idea, different package manager
sudo apt install -y python3 open-vm-tools
```

Before snapshotting either template, three things get cleaned so linked clones don't inherit duplicate identity:

```bash
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -sf /etc/machine-id /var/lib/dbus/machine-id
sudo rm -f /etc/ssh/ssh_host_*
```

Skip this and every clone shares the same `machine-id` and SSH host key — harmless until something (DHCP, `known_hosts`, monitoring) gets confused about which host is actually which.

From there: linked clone each template twice, add a static IP on the lab network (`nmcli` for Rocky, netplan for Ubuntu), and copy the same SSH public key into all four `authorized_keys` files. Four VMs, two templates, a couple of minutes of cloning instead of four fresh installs.

## First contact: `ansible all -m ping`

```ini
# inventory.ini
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
ansible all -i inventory.ini -m ping
```

First run failed with `Host key verification failed` — expected, since none of these hosts had been touched from `devbox` before:

```bash
ssh-keyscan -H 192.168.10.21 >> ~/.ssh/known_hosts
ssh-keyscan -H 192.168.10.22 >> ~/.ssh/known_hosts
ssh-keyscan -H 192.168.10.23 >> ~/.ssh/known_hosts
ssh-keyscan -H 192.168.10.24 >> ~/.ssh/known_hosts
```

After that, all four came back `SUCCESS` / `pong`. Encouraging — and, as it turned out, not the interesting part of this post.

---

## The real problem: sudo isn't sudo anymore on one of these

Writing a playbook that needs `become: true` should be routine. Here it wasn't — but only on the two Ubuntu hosts:

```
ansible ubuntu -i inventory.ini -b --ask-become-pass -m shell -a 'whoami'

[ERROR]: Task failed: Timeout (12s) waiting for privilege escalation prompt
```

Rocky: instant success. Ubuntu: timeout, every time, regardless of password correctness. The obvious suspects got ruled out one by one: wrong password (interactive `sudo` worked fine by hand), stale SSH multiplexing sockets (cleared, no change), parallel execution racing on the password prompt (`-f 1`, no change), passing the password via `-e` instead of an interactive prompt (no change).

The actual test that cracked it was reproducing Ansible's exact `sudo` invocation by hand:

```bash
ssh -tt tnpadmin@192.168.10.22 \
  'sudo -H -S -p "TESTPROMPT:" -u root /bin/sh -c "echo SUCCESS"'
```

The prompt came back as:

```
[sudo: TESTPROMPT:] Password:
```

Not `TESTPROMPT:`. Wrapped in `[sudo: ... ]`. Ansible sends a unique marker string via `-p` and scans the output for that exact string to know when to feed the password — and this wrapping meant it never found what it was looking for.

**Ubuntu 26.04 ships `sudo-rs`** — Canonical's Rust rewrite of sudo, the default since 25.10 — instead of GNU sudo. It handles `-p`/`-S` differently enough that Ansible's prompt-matching breaks, even though `sudo` itself works perfectly for a human typing a password by hand.

### The fix: stop asking sudo for a password at all

```bash
echo "tnpadmin ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/tnpadmin
sudo chmod 440 /etc/sudoers.d/tnpadmin
```

Run once per Ubuntu host, over a plain SSH session. With NOPASSWD in place, Ansible never sends a password through `become` in the first place — the sudo-rs prompt format never comes into play, because there's no prompt to match.

> If you're used to blaming "OS swapped a core tool" failures on containers or bleeding-edge distros, file this one away: it's exactly the kind of thing that looks like a connection problem, behaves like a credentials problem, and turns out to be neither.

---

## The first real playbook

With auth sorted, the actual config management part was almost anticlimactic — which is the point. A single playbook, branching on `ansible_os_family`, installs Nginx, opens the right firewall, and drops a landing page that identifies the host:

```yaml
---
- name: Install and configure Nginx across mixed OS lab
  hosts: all
  become: true

  tasks:
    - name: Install Nginx (RedHat family)
      ansible.builtin.dnf:
        name: nginx
        state: present
      when: ansible_os_family == "RedHat"

    - name: Install Nginx (Debian family)
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
      when: ansible_os_family == "Debian"

    - name: Ensure Nginx is started and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Open firewall for HTTP (firewalld - RedHat family)
      ansible.posix.firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: true
      when: ansible_os_family == "RedHat"

    - name: Open firewall for HTTP (ufw - Debian family)
      community.general.ufw:
        rule: allow
        port: '80'
      when: ansible_os_family == "Debian"

    - name: Deploy a custom index page identifying the host
      ansible.builtin.copy:
        content: |
          <html><body>
            <h1>{{ inventory_hostname }}</h1>
            <p>{{ ansible_distribution }} {{ ansible_distribution_version }}</p>
          </body></html>
        dest: "{{ '/usr/share/nginx/html/index.html' if ansible_os_family == 'RedHat' else '/var/www/html/index.html' }}"
```

```bash
ansible-playbook -i inventory.ini site.yml
```

```
PLAY RECAP
web-rocky    : ok=6  changed=5  failed=0
web-ubuntu   : ok=6  changed=5  failed=0
db-rocky     : ok=6  changed=5  failed=0
db-ubuntu    : ok=6  changed=5  failed=0
```

```bash
curl http://192.168.10.21   # web-rocky
curl http://192.168.10.22   # web-ubuntu
```

Each returns HTML naming the correct host, distro, and version. Same playbook, two package managers, two firewall tools, zero branching left to the person running it.

---

## What this actually taught

The Rocky/Ubuntu split wasn't just about learning two syntaxes for installing a package. It surfaced a real, current compatibility problem — sudo-rs vs. Ansible — that a single-distro lab would never have exposed. That's the value of forcing heterogeneity into a lab on purpose: the interesting failures only show up at the seams.

**Next up: Part 3 — Building a Self-Hosted CI/CD Platform**
GitLab CE, Harbor, and SonarQube on a single VM — and the first real branch-protected merge request.
