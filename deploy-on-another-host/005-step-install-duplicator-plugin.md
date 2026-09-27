# 005 — Install Duplicator on Production

## Goal

Install the free Duplicator plugin on the live production site so it can
build a full-site export package (files + database) in the next step.

> **Scope note:** this is the one step that touches production
> (`https://solar.chemeng.ntua.gr/`). Per the project's topology, production
> otherwise stays live and untouched — this step adds a plugin used only to
> read/export the site, performs no destructive action, and
> [011-step-post-migration-cleanup.md](011-step-post-migration-cleanup.md)
> removes it again once the clone is verified.

## Prerequisites

- Administrator login on `https://solar.chemeng.ntua.gr/wp-admin/`.

## Steps

### 1. Install the plugin

1. Log in to `https://solar.chemeng.ntua.gr/wp-admin/`.
2. Navigate to **Plugins → Add New**.
3. Search for **"Duplicator"** (published by Snap Creek).
4. Click **Install Now**, then **Activate** once installation finishes.

### 2. Confirm the free edition is sufficient

The free edition supports full site packages (files + database) up to the
size Duplicator's own scan step flags as safe for shared/typical hosting.
Before building the package in
[006-step-create-duplicator-package.md](006-step-create-duplicator-package.md),
note the site's approximate size:

1. Navigate to **Duplicator → Packages → Create New**.
2. Let the **Scan** step run and note the reported archive size and any
   warnings (large `uploads/` directories, oversized tables, PHP memory/time
   limits).
3. If the scan reports a size or resource warning, address it before
   building (e.g. exclude cache directories — configured in the next step) —
   don't proceed on a scan that reports a hard failure.

## Verification

On **Plugins → Installed Plugins**, confirm **Duplicator** is listed and
**Active**.
