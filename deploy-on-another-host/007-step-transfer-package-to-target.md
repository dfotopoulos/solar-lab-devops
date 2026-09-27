# 007 — Transfer the Package to the Target Server

## Goal

Copy the `installer.php` and archive `.zip` downloaded from Duplicator into
the target server's document root, where the installer wizard will run in
the next step.

## Placeholders used in this runbook

- `<DEPLOY_USER>` — sudo-capable Linux user on the target host
- `<TARGET_IP>` — public IP address of the new Debian Trixie server
- `<DOCROOT>` — document root path configured in
  [004-step-configure-apache-vhost.md](004-step-configure-apache-vhost.md),
  e.g. `/var/www/preprod-solar`
- `<PACKAGE_NAME>` — the Duplicator package name chosen in
  [006-step-create-duplicator-package.md](006-step-create-duplicator-package.md)

## Steps

### 1. Copy both files to the target

```bash
scp \
    installer.php \
    <PACKAGE_NAME>_archive.zip \
    <DEPLOY_USER>@<TARGET_IP>:/tmp/
```

> What this does: `scp` copies both local files over SSH into `/tmp/` on the
> target, using the same connection/auth already set up in
> [001-step-prepare-target-server.md](001-step-prepare-target-server.md).
> `/tmp/` is used as a neutral landing spot before moving the files into the
> web root with correct ownership.

### 2. Move the files into the document root with correct ownership

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo mv /tmp/installer.php /tmp/<PACKAGE_NAME>_archive.zip <DOCROOT>/ \
    && sudo chown www-data:www-data <DOCROOT>/installer.php <DOCROOT>/<PACKAGE_NAME>_archive.zip"
```

> What this does: moves both files into `<DOCROOT>` (the vhost's
> `DocumentRoot`) and sets ownership to `www-data` so the PHP process
> running as Apache's user can read the archive and write extracted files
> and `wp-config.php` during the installer step.

## Verification

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "ls -la <DOCROOT>/installer.php <DOCROOT>/<PACKAGE_NAME>_archive.zip"
```

Expect both files listed, owned by `www-data:www-data`, with sizes matching
the local downloads from
[006-step-create-duplicator-package.md](006-step-create-duplicator-package.md).
