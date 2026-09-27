# 006 — Create the Duplicator Package

## Goal

Build the full-site export on production — an archive of all files plus a
database dump — and download the two artifacts needed to install it on the
target server.

## Prerequisites

- Duplicator installed and active on production (see
  [005-step-install-duplicator-plugin.md](005-step-install-duplicator-plugin.md)).

## Steps

### 1. Start a new package

1. In `wp-admin`, navigate to **Duplicator → Packages → Create New**.
2. Give it a recognizable name, e.g. `solar-preprod-clone-YYYY-MM-DD`.

### 2. Configure the scan (exclusions)

Before building, exclude anything that doesn't need to travel to the
pre-prod clone and would only inflate transfer time:

- Cache plugin directories (e.g. `wp-content/cache/`, `wp-content/wflogs/`)
  under **Archive → Files → File & Folders**.
- Large log files unrelated to WordPress content.

Do **not** exclude `wp-content/uploads/` — that's the media library and is
part of "full-site clone".

### 3. Run the scan and resolve warnings

Click **Next** to run the **Scan** step. Duplicator checks PHP settings,
directory permissions, and package size, and flags:

- **Good** — proceed.
- **Warn** — reviewable but not blocking (e.g. large `uploads/` size); note
  it and continue if it's expected for this site.
- **Bad** — must be resolved before building (e.g. a PHP `max_execution_time`
  too low for the archive size, or unwritable paths). Fix at the hosting
  level and re-run the scan.

### 4. Build the package

1. Click **Build**. Duplicator archives the files and dumps the database.
2. When it finishes, download both artifacts from the **Packages** list:
   - **Installer** (`installer.php`)
   - **Archive** (`<package-name>_archive.zip`)

Keep both files together locally — the installer references the archive by
name and both are required on the target server in
[007-step-transfer-package-to-target.md](007-step-transfer-package-to-target.md).

## Verification

Confirm both downloaded files are non-zero size and that the archive's size
roughly matches what the scan step reported. If the archive size is
suspiciously small (e.g. a few KB), the build likely failed silently —
re-run the build rather than proceeding.
