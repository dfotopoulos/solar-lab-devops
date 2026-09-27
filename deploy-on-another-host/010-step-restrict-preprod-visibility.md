# 010 — Restrict Pre-Prod Visibility

## Goal

Prevent the pre-prod clone from being indexed by search engines or mistaken
for the production site, since it's a byte-for-byte content copy reachable
on the public internet.

## Placeholders used in this runbook

- `<DEPLOY_USER>` — sudo-capable Linux user on the target host
- `<TARGET_IP>` — public IP address of the new Debian Trixie server
- `<DOCROOT>` — document root path, e.g. `/var/www/preprod-solar`
- `<BASIC_AUTH_USER>`, `<BASIC_AUTH_PASS>` — credentials for the optional
  access gate in step 3

## Steps

### 1. Block search engine indexing at the WordPress level

1. Log in to `https://preprod.solar.poland-contabo.fyi/wp-admin/`.
2. Navigate to **Settings → Reading**.
3. Check **Discourage search engines from indexing this site**.
4. Save changes.

> This sets `blog_public` to `0`, which makes WordPress emit a `noindex,
> nofollow` meta tag and a restrictive virtual `robots.txt`. It's a signal
> search engines are expected to respect, not an access control — combine
> with step 3 below for stronger isolation.

### 2. Confirm the noindex tag is present

```bash
curl -s https://preprod.solar.poland-contabo.fyi/ \
    | grep -i 'noindex'
```

Expect a `<meta name="robots" content="noindex, nofollow"/>` tag in the
output.

### 3. (Recommended) Gate access with HTTP Basic Auth at Caddy

Since the discourage-indexing setting relies on well-behaved crawlers,
add an access gate in front of the whole site for real restriction:

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "caddy hash-password --plaintext '<BASIC_AUTH_PASS>'"
```

> What this does: outputs a bcrypt hash of `<BASIC_AUTH_PASS>` for use in the
> Caddyfile — Caddy never stores or compares plaintext passwords.

Add a `basicauth` block to the site definition from
[009-step-install-configure-caddy.md](009-step-install-configure-caddy.md):

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo tee /etc/caddy/Caddyfile > /dev/null" <<EOF
preprod.solar.poland-contabo.fyi {
    basicauth {
        <BASIC_AUTH_USER> <BCRYPT_HASH_FROM_PREVIOUS_COMMAND>
    }
    reverse_proxy 127.0.0.1:8080
}
EOF
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo systemctl reload caddy"
```

> What this does: requires the browser-level Basic Auth credentials before
> Caddy forwards any request to Apache, so the site content itself is never
> reachable by an unauthenticated crawler or visitor — a stronger guarantee
> than the WordPress-level setting alone.

## Verification

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
    https://preprod.solar.poland-contabo.fyi/
```

If Basic Auth was configured in step 3, expect `401`. Then confirm access
works with credentials:

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
    -u '<BASIC_AUTH_USER>:<BASIC_AUTH_PASS>' \
    https://preprod.solar.poland-contabo.fyi/
```

Expect `200`.
