# 011 — Post-Migration Cleanup

## Goal

Remove the temporary migration artifacts on both ends so neither the
production site nor the pre-prod clone is left carrying files or a plugin
that were only needed to perform the clone.

## Placeholders used in this runbook

- `<DEPLOY_USER>` — sudo-capable Linux user on the target host
- `<TARGET_IP>` — public IP address of the new Debian Trixie server
- `<DOCROOT>` — document root path, e.g. `/var/www/preprod-solar`
- `<PACKAGE_NAME>` — the Duplicator package name from
  [006-step-create-duplicator-package.md](006-step-create-duplicator-package.md)

## Steps

### 1. Remove the installer and archive from the target

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo rm -f \
    <DOCROOT>/installer.php \
    <DOCROOT>/<PACKAGE_NAME>_archive.zip \
    <DOCROOT>/dup-installer*.log \
    && sudo find <DOCROOT> -maxdepth 1 -iname '*dup-installer*' -exec rm -rf {} +"
```

> What this does: deletes `installer.php`, the source archive, and any
> Duplicator installer log/backup directories it leaves behind (named with
> a `dup-installer` prefix) from the document root. Leaving `installer.php`
> in place is a known security risk — anyone who finds the URL could
> re-run a site wipe/reinstall.

### 2. Deactivate and remove Duplicator from production

1. Log in to `https://solar.chemeng.ntua.gr/wp-admin/`.
2. Navigate to **Plugins → Installed Plugins**.
3. **Deactivate**, then **Delete** the **Duplicator** plugin.

> This restores production to its pre-clone state, per the project's
> "production stays untouched" principle — the plugin installed in
> [005-step-install-duplicator-plugin.md](005-step-install-duplicator-plugin.md)
> was only needed to build the export package.

### 3. Remove the local copies of the package

Delete the local `installer.php` and `<PACKAGE_NAME>_archive.zip` downloaded
in
[006-step-create-duplicator-package.md](006-step-create-duplicator-package.md)
once the clone is verified (see
[012-step-verify-deployment.md](012-step-verify-deployment.md)) — they
contain a full database dump and shouldn't be left on a workstation or
uploaded anywhere.

## Verification

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "curl -s -o /dev/null -w '%{http_code}\n' https://preprod.solar.poland-contabo.fyi/installer.php"
```

Expect `404`. Also confirm on production's **Plugins** page that Duplicator
no longer appears in the installed plugins list.
