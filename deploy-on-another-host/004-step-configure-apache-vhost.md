# 004 — Configure Apache Virtual Host

## Goal

Create the document root and Apache virtual host that will serve the cloned
WordPress site on `127.0.0.1:8080`, ready to receive the Duplicator package
files in the next step.

## Placeholders used in this runbook

- `<DEPLOY_USER>` — sudo-capable Linux user on the target host
- `<TARGET_IP>` — public IP address of the new Debian Trixie server
- `<DOCROOT>` — document root path, e.g. `/var/www/preprod-solar`

## Steps

### 1. Create the document root

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo mkdir -p <DOCROOT> \
    && sudo chown -R www-data:www-data <DOCROOT>"
```

> What this does: creates the directory that will hold the WordPress files
> once unpacked by the Duplicator installer, and gives ownership to
> `www-data` (the user Apache runs as), which the installer needs in order
> to write `wp-config.php` and extract the archive.

### 2. Write the virtual host file

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo tee /etc/apache2/sites-available/preprod-solar.conf > /dev/null" <<'EOF'
<VirtualHost 127.0.0.1:8080>
    ServerName preprod.solar.poland-contabo.fyi
    DocumentRoot <DOCROOT>

    <Directory <DOCROOT>>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/preprod-solar-error.log
    CustomLog ${APACHE_LOG_DIR}/preprod-solar-access.log combined
</VirtualHost>
EOF
```

> What this does: defines a virtual host bound to `127.0.0.1:8080` (matching
> the `Listen` directive from
> [003-step-install-lamp-stack.md](003-step-install-lamp-stack.md)) for the
> `preprod.solar.poland-contabo.fyi` hostname, points it at `<DOCROOT>`, and
> sets `AllowOverride All` so WordPress's `.htaccess` permalink rules work.
> Separate log files make it easy to isolate this site's log output from
> Apache's default logs.

### 3. Enable the site and reload Apache

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo a2ensite preprod-solar.conf \
    && sudo apache2ctl configtest \
    && sudo systemctl reload apache2"
```

> What this does: `a2ensite` symlinks the config into
> `sites-enabled/`; `apache2ctl configtest` validates the syntax before
> touching the running server (fails loudly instead of taking Apache down);
> `systemctl reload apache2` applies the change without dropping existing
> connections.

## Verification

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "curl -s -o /dev/null -w '%{http_code}\n' \
    -H 'Host: preprod.solar.poland-contabo.fyi' \
    http://127.0.0.1:8080/"
```

Expect `200` or `403` (403 is fine at this stage — the document root is
still empty; it confirms Apache is routing the vhost correctly rather than
falling back to the default site). A `000` or connection-refused result
means Apache isn't listening on `127.0.0.1:8080` — recheck
[003-step-install-lamp-stack.md](003-step-install-lamp-stack.md) step 5.
