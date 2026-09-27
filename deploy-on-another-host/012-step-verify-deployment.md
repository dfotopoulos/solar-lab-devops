# 012 — Verify Deployment

## Goal

Confirm the pre-prod clone is fully functional under its new hostname and
that production was left unaffected, before treating the migration as done.

## Placeholders used in this runbook

- `<DEPLOY_USER>` — sudo-capable Linux user on the target host
- `<TARGET_IP>` — public IP address of the new Debian Trixie server

## Checks

### 1. TLS and transport

```bash
curl -sI https://preprod.solar.poland-contabo.fyi/ \
    | head -n 1 \
    && curl -vI https://preprod.solar.poland-contabo.fyi/ 2>&1 \
    | grep -i 'expire\|issuer'
```

Expect `HTTP/2 200` and a Let's Encrypt-issued, unexpired certificate (set
up in
[009-step-install-configure-caddy.md](009-step-install-configure-caddy.md)).

### 2. Homepage and asset loading

Open `https://preprod.solar.poland-contabo.fyi/` in a browser:

- Confirm the homepage renders with styling and images intact (validates
  that the URL rewrite in
  [008-step-run-duplicator-installer.md](008-step-run-duplicator-installer.md)
  correctly updated asset URLs in the database and that
  `wp-content/uploads/` was transferred).
- Open browser dev tools → Network tab, reload, and confirm no requests are
  going out to `solar.chemeng.ntua.gr` (would indicate a missed
  search-and-replace).

### 3. Admin login and database connectivity

1. Log in to `https://preprod.solar.poland-contabo.fyi/wp-admin/` with the
   same credentials as production (Duplicator carries over the `wp_users`
   table).
2. Confirm **Settings → General** shows both **WordPress Address (URL)** and
   **Site Address (URL)** as `https://preprod.solar.poland-contabo.fyi`.
3. Open a post or page in the editor and confirm content renders correctly
   (validates the database import completed without truncation).

### 4. Permalinks and rewrite rules

Visit a non-homepage URL that relies on pretty permalinks (e.g. a blog
post). A working result confirms `mod_rewrite` (enabled in
[003-step-install-lamp-stack.md](003-step-install-lamp-stack.md)) and the
`.htaccess` carried over from the archive are both functioning. If it
404s, regenerate permalinks from **Settings → Permalinks → Save Changes**.

### 5. Visibility restrictions in effect

Re-run the checks from
[010-step-restrict-preprod-visibility.md](010-step-restrict-preprod-visibility.md)
— `noindex` meta tag present, and Basic Auth (if configured) returning `401`
without credentials.

### 6. Production is unaffected

```bash
curl -sI https://solar.chemeng.ntua.gr/ \
    | head -n 1
```

Expect `HTTP/2 200`, and confirm in **Plugins → Installed Plugins** on
production that Duplicator was removed per
[011-step-post-migration-cleanup.md](011-step-post-migration-cleanup.md).

## Sign-off

Once all six checks pass, the clone is complete: `<TARGET_IP>` is serving a
fully functional copy of the Solar Lab site at
`https://preprod.solar.poland-contabo.fyi`, and
`https://solar.chemeng.ntua.gr/` continues to serve production unchanged.
