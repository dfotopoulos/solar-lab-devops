# 003 — Install Apache, PHP, and MariaDB

## Goal

Install the stack WordPress needs — Apache HTTP Server, PHP with the modules
WordPress requires, and MariaDB — and configure Apache to listen only on
localhost, since Caddy (not Apache) is the public-facing server.

## Placeholders used in this runbook

- `<DEPLOY_USER>` — sudo-capable Linux user on the target host
- `<TARGET_IP>` — public IP address of the new Debian Trixie server
- `<DB_NAME>`, `<DB_USER>`, `<DB_PASS>` — database name/credentials for the
  cloned WordPress site

## Steps

### 1. Install Apache and PHP with required modules

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo apt install -y \
    apache2 \
    php \
    libapache2-mod-php \
    php-mysql \
    php-curl \
    php-gd \
    php-mbstring \
    php-xml \
    php-zip \
    php-intl"
```

> What this does: installs Apache, PHP, the Apache↔PHP bridge module, and
> the PHP extensions WordPress core and common plugins expect (`mysqli` via
> `php-mysql`, image handling via `php-gd`, string/XML handling for
> import/export, and `php-zip` — needed both by WordPress's own updater and
> by the Duplicator installer to unpack the archive in
> [008-step-run-duplicator-installer.md](008-step-run-duplicator-installer.md)).

### 2. Install MariaDB and secure it

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo apt install -y mariadb-server \
    && sudo mysql_secure_installation"
```

> What this does: installs the MariaDB server, then runs the interactive
> hardening wizard (set root password, remove anonymous users, disallow
> remote root login, remove the test database) — answer "yes" to each
> prompt.

### 3. Create the WordPress database and user

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo mysql -e \
    \"CREATE DATABASE <DB_NAME> CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci; \
    CREATE USER '<DB_USER>'@'localhost' IDENTIFIED BY '<DB_PASS>'; \
    GRANT ALL PRIVILEGES ON <DB_NAME>.* TO '<DB_USER>'@'localhost'; \
    FLUSH PRIVILEGES;\""
```

> What this does: creates an empty database with the `utf8mb4` charset
> (matches modern WordPress defaults) and a dedicated MySQL user scoped to
> `localhost` with full privileges on that one database only — this is what
> the Duplicator installer will import into in
> [008-step-run-duplicator-installer.md](008-step-run-duplicator-installer.md).

### 4. Enable required Apache modules

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo a2enmod rewrite \
    && sudo systemctl restart apache2"
```

> What this does: enables `mod_rewrite`, required for WordPress's pretty
> permalinks (`.htaccess` rewrite rules), then restarts Apache to load it.

### 5. Make Apache listen only on localhost

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "sudo sed -i 's/^Listen 80/Listen 127.0.0.1:8080/' /etc/apache2/ports.conf \
    && sudo systemctl restart apache2"
```

> What this does: rewrites Apache's `Listen` directive in `ports.conf` from
> the default `Listen 80` (all interfaces) to `Listen 127.0.0.1:8080` — so
> Apache is reachable only from the same host, on port 8080. Caddy is the
> only process that will ever bind to the public interface on 80/443 (see
> [009-step-install-configure-caddy.md](009-step-install-configure-caddy.md)),
> and it reverse-proxies to `127.0.0.1:8080`.

## Verification

```bash
ssh <DEPLOY_USER>@<TARGET_IP> \
    "php -v \
    && sudo mysql -e 'SHOW DATABASES;' \
    && sudo ss -tlnp | grep apache2"
```

Expect a PHP version banner, `<DB_NAME>` listed among the databases, and the
`ss` output showing Apache bound to `127.0.0.1:8080` (not `0.0.0.0:80` or
`:::80`).
