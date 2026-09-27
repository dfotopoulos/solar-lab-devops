# 009 — Install and Configure Caddy

## Goal

Install Caddy on the target host and configure it as the public-facing
reverse proxy: it terminates TLS for `preprod.solar.poland-contabo.fyi` on
443 and forwards requests to Apache on `127.0.0.1:8080`.

## Placeholders used in this runbook

- `<DEPLOY_USER>` — sudo-capable Linux user on the target host
- `<TARGET_IP>` — public IP address of the new Debian Trixie server

## Prerequisites

- DNS for `preprod.solar.poland-contabo.fyi` resolves to `<TARGET_IP>` (see
  [002-step-point-dns-to-target.md](002-step-point-dns-to-target.md)) — required
  for Caddy's automatic TLS to complete its ACME HTTP-01 challenge.
- Ports 80/443 open in the firewall (see
  [001-step-prepare-target-server.md](001-step-prepare-target-server.md)).
- Apache serving the site correctly on `127.0.0.1:8080` (see
  [008-step-run-duplicator-installer.md](008-step-run-duplicator-installer.md)).

## Steps

### 1. Add the official Caddy repository and install

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl \
    && curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
    | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg \
    && curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
    | sudo tee /etc/apt/sources.list.d/caddy-stable.list \
    && sudo apt update \
    && sudo apt install -y caddy"
```

> What this does: adds Caddy's official Cloudsmith APT repository (signed
> with its own keyring, avoiding a global `apt-key add`) and installs the
> `caddy` package from it — the current recommended install method on
> Debian per Caddy's own documentation.

### 2. Write the Caddyfile

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo tee /etc/caddy/Caddyfile > /dev/null" <<'EOF'
preprod.solar.poland-contabo.fyi {
    reverse_proxy 127.0.0.1:8080
}
EOF
```

> What this does: defines a single site block. Caddy automatically obtains
> and renews a Let's Encrypt certificate for the hostname (no explicit `tls`
> directive needed — this is Caddy's default behavior for any site with a
> real hostname) and forwards all traffic to Apache on `127.0.0.1:8080`.

### 3. Reload Caddy

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo systemctl reload caddy"
```

> What this does: applies the new Caddyfile. On first load for this
> hostname, Caddy performs the ACME HTTP-01 challenge over port 80 and
> obtains the certificate before serving HTTPS on 443.

## Verification

```bash
curl -sI https://preprod.solar.poland-contabo.fyi/ \
    | head -n 1
```

Expect `HTTP/2 200`. Also confirm the certificate is valid and
Let's-Encrypt-issued:

```bash
curl -vI https://preprod.solar.poland-contabo.fyi/ 2>&1 \
    | grep -i 'issuer\|subject'
```

If the HTTPS request fails or times out, check
`sudo journalctl -u caddy --since '10 min ago'` on the target for ACME
challenge errors — the most common cause is DNS not yet resolving to
`<TARGET_IP>` (recheck
[002-step-point-dns-to-target.md](002-step-point-dns-to-target.md)).
