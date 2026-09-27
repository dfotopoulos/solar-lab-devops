# 001 — Prepare Target Server

## Goal

Bring the new Debian Trixie host to a baseline state: fully updated, a
non-root administrative user, and a firewall that only allows the traffic
this deployment needs (SSH, HTTP, HTTPS).

## Placeholders used in this runbook

- `<TARGET_IP>` — public IP address of the new Debian Trixie server
- `<DEPLOY_USER>` — the sudo-capable Linux user used to administer the target
  host (created in this step if it doesn't already exist)

## Prerequisites

- Root (or sudo) SSH access to the freshly provisioned Debian Trixie host.

## Steps

### 1. Update the base system

```bash
ssh root@<TARGET_IP> \
    "apt update \
    && apt full-upgrade -y \
    && apt autoremove -y"
```

> What this does: connects as root and runs three chained steps — `apt
> update` refreshes the package index, `apt full-upgrade -y` installs all
> available updates (including ones that add/remove dependencies, safe on a
> fresh Trixie install), and `apt autoremove -y` clears packages that are no
> longer needed as a result of the upgrade.

### 2. Create a non-root administrative user

```bash
ssh root@<TARGET_IP> \
    "adduser --disabled-password --gecos '' <DEPLOY_USER> \
    && usermod -aG sudo <DEPLOY_USER>"
```

> What this does: `adduser --disabled-password --gecos ''` creates the user
> without prompting for a password interactively (SSH key auth is added
> next) and without the interactive GECOS prompts; `usermod -aG sudo` adds
> the user to the `sudo` group so they can run privileged commands.

### 3. Authorize your SSH key for the new user

```bash
ssh-copy-id <DEPLOY_USER>@<TARGET_IP>
```

> What this does: appends your local public key to
> `~<DEPLOY_USER>/.ssh/authorized_keys` on the target so future steps can
> connect as `<DEPLOY_USER>` without a password.

### 4. Configure the firewall

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo apt install -y ufw \
    && sudo ufw allow OpenSSH \
    && sudo ufw allow 80/tcp \
    && sudo ufw allow 443/tcp \
    && sudo ufw --force enable"
```

> What this does: installs `ufw` (Uncomplicated Firewall), allows the
> `OpenSSH` service (port 22) so the current session isn't locked out, opens
> ports 80 and 443 for Caddy's HTTP-01 challenge and HTTPS traffic, then
> enables the firewall. `--force` skips the interactive
> "this may disconnect you" confirmation, which is safe here because SSH is
> explicitly allowed first.

Apache itself will only ever listen on `127.0.0.1` (see
[003-step-install-lamp-stack.md](003-step-install-lamp-stack.md) and
[004-step-configure-apache-vhost.md](004-step-configure-apache-vhost.md)), so
it does not need its own firewall rule — only Caddy is exposed externally.

## Verification

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo ufw status verbose"
```

Expect `OpenSSH`, `80/tcp`, and `443/tcp` listed as `ALLOW`, and the firewall
status as `active`.
