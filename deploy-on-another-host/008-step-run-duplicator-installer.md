# 008 — Run the Duplicator Installer

## Goal

Extract the archive, import the database, and rewrite all URLs from the
production domain to the pre-prod hostname — all done through Duplicator's
browser-based installer wizard.

Apache only listens on `127.0.0.1:8080` (per
[003-step-install-lamp-stack.md](003-step-install-lamp-stack.md)) and Caddy
isn't configured yet (that's
[009-step-install-configure-caddy.md](009-step-install-configure-caddy.md)),
so the installer is reached through a local SSH tunnel rather than exposing
Apache publicly.

## Placeholders used in this runbook

- `<DEPLOY_USER>` — sudo-capable Linux user on the target host
- `<TARGET_IP>` — public IP address of the new Debian Trixie server
- `<DB_NAME>`, `<DB_USER>`, `<DB_PASS>` — database credentials created in
  [003-step-install-lamp-stack.md](003-step-install-lamp-stack.md)

## Steps

### 1. Open an SSH tunnel to Apache

```bash
ssh -N -L 8080:127.0.0.1:8080 <DEPLOY_USER>@<TARGET_IP>
```

> What this does: `-L 8080:127.0.0.1:8080` forwards local port 8080 to
> `127.0.0.1:8080` as seen from the target server — i.e. to Apache. `-N`
> means "don't run a remote command, just forward the port." Leave this
> running in its own terminal for the duration of this step.

### 2. Run the installer in a browser

1. With the tunnel open, browse to `http://localhost:8080/installer.php`.
2. Duplicator runs its own pre-flight checks (PHP version, extensions,
   archive integrity) — resolve any that fail before continuing.
3. **Setup — Database**: enter `localhost`, `<DB_NAME>`, `<DB_USER>`,
   `<DB_PASS>` (created in
   [003-step-install-lamp-stack.md](003-step-install-lamp-stack.md)).
   Choose **Empty Database** — the schema and data come entirely from the
   archive.
4. **Setup — Site URLs**: this is the key step for moving to a new hostname.
   Set both the new site URL and path to:
   - URL: `https://preprod.solar.poland-contabo.fyi`
   - Path: `<DOCROOT>` (as configured in
     [004-step-configure-apache-vhost.md](004-step-configure-apache-vhost.md))

   Duplicator rewrites `siteurl`/`home` in the `wp_options` table and runs a
   search-and-replace across serialized post content, widget data, and
   plugin settings, swapping `solar.chemeng.ntua.gr` for
   `preprod.solar.poland-contabo.fyi`.
5. Click **Run Deployment**. Duplicator extracts the archive into
   `<DOCROOT>`, imports the SQL dump, writes a new `wp-config.php` with the
   database credentials from step 3, and performs the URL replacement.
6. On the final **Test** screen, click through to confirm the front end and
   `wp-admin` load, then remove the installer per that screen's own cleanup
   prompt (this is reinforced in
   [011-step-post-migration-cleanup.md](011-step-post-migration-cleanup.md)
   in case it's skipped here).

### 3. Close the tunnel

Once the installer confirms success, `Ctrl+C` the `ssh -L` session from
step 1 — it's no longer needed.

## Verification

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo mysql <DB_NAME> -e \
    \"SELECT option_value FROM wp_options WHERE option_name IN ('siteurl','home');\""
```

Expect both rows to show `https://preprod.solar.poland-contabo.fyi` — if
either still shows `solar.chemeng.ntua.gr`, the URL rewrite step in the
wizard was skipped or entered incorrectly, and step 2 above should be
re-run against a fresh archive extraction.
