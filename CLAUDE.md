# solar-lab-devops — Development Conventions

DevOps tooling and documentation for cloning and deploying the NTUA Chemeng
Solar Lab WordPress site into a new pre-production environment.

**Topology:**

- **Production** (untouched, stays live as-is): `https://solar.chemeng.ntua.gr/`
- **Pre-prod target host:** Debian Trixie server running Apache HTTP Server,
  serving a full clone of the WordPress site.
- **Pre-prod public entry point:** `https://preprod.solar.poland-contabo.fyi`,
  fronted by a Caddy server running on the **same host** as Apache. Caddy
  terminates TLS on 443 and reverse-proxies to Apache listening on a local
  port (e.g. `127.0.0.1:8080`).

**Clone strategy:** full-site clone (files + database) via a free WordPress
migration/backup plugin — Duplicator is the current leading candidate —
rather than a manual export/import.

**Stack:** WordPress (PHP/MySQL or MariaDB), Apache HTTP Server, Debian
Trixie, Caddy.

---

## Commit Messages

Every commit's **subject line is at most 3 words.** Put everything else —
the full description, rationale, and any bullet list of changes — in the
body, separated from the subject by a blank line.

```
Add user auth

Implement JWT-based authentication with refresh tokens.

- Add /auth/login and /auth/refresh endpoints
- Store hashed tokens in Redis with a 24-hour TTL
- Reject expired tokens with 401 rather than 403
```

Never add `Co-Authored-By: Claude` or any similar AI-attribution trailer to
commits. Enforce globally via `~/.claude/settings.json`:

```json
{ "attribution": { "commit": "" } }
```

---

## Command Logging & Formatting

Every shell command run against this repo (locally or relayed for someone to
run on a server over SSH) must be:

1. **Logged** in a Markdown file at
   `doc/command-log/YYYY/YYYY-MM/YYYY-MM-DD/YYYY-MM-DD-NNN-<slug>.md` — three
   directory levels (year, year-month, year-month-day), with the sequence
   number and a short descriptive slug in the filename. `NNN` is a sequential
   3-digit number per day, starting at `001`, always using the current date.
2. **Described in detail**: every command (or chained sequence) in the log
   entry is accompanied by a clear explanation of what it is trying to achieve
   and how it works — not just the command text and its output. State the goal,
   walk through each flag/step, then give the result/verification.
3. **Formatted for readability**: multiline, using trailing `\` continuations
   indented **4 spaces**, with sequential steps joined by `&&` — both when
   executing and when transcribing into the log. Avoid dense single-line
   commands with multiple chained parts.

```bash
git status --short \
    && git diff --staged
```

> What this does: `git status --short` lists modified/untracked files in the
> compact `XY path` format; `&&` chains a second command that only runs if the
> first succeeds — `git diff --staged` shows the exact line-by-line changes
> already staged for commit.

The same level of detail applies to deploy commands documented in release
files and `DEPLOY.md` files.

---

## Technical & UX Analysis Log

Any non-trivial design decision — a CSS layout choice, an architecture
trade-off, a DevOps approach, a UX behaviour — must be captured as a
structured Markdown entry **before or alongside** the code that implements it,
not reconstructed from memory later.

**File path:** `doc/analysis/YYYY/YYYY-MM/YYYY-MM-DD/YYYY-MM-DD-NNN-<slug>.md`

Same three-level hierarchy and per-day sequential numbering (`001`, `002`, …)
as the command log.

**When to write one:** whenever the answer to "why did we do it this way and
not the obvious alternative?" is non-trivial. If a reader of the diff would
ask "why not just X?", write the entry. Routine copy edits or one-line fixes
with obvious intent don't need one.

**What each entry must contain:**

1. **Problem** — user-facing symptom or technical constraint, with enough
   context that someone unfamiliar with the codebase can follow.
2. **Root cause** — the specific technical reason the symptom exists.
3. **Options considered** — at least two alternatives, each with a clear
   trade-off assessment. "We only thought of one approach" is never accurate —
   document the alternatives even if they were rejected quickly.
4. **Decision** — which option was chosen and the decisive reason.
5. **Implementation summary** — the files changed and the key lines that encode
   the decision, so the entry and the diff cross-reference each other.

The analysis file is staged and committed in the **same commit** as the code
it describes. It is not a post-hoc documentation step.

---

## Release Analysis (Unreleased Staging)

Non-trivial root-cause writeups accumulate under
`releases/UNRELEASED/analysis/` as the work happens — not reconstructed from
memory once a version number exists. At tag time, the whole `UNRELEASED/`
directory is renamed to `releases/vX.Y.Z/`.

```
releases/
  UNRELEASED/
    analysis/
      001-fix-session-expiry-race.md
      002-move-uploads-to-s3.md
  v1.2.0/
    README.md
    analysis/
      001-…
```

Each file is numbered sequentially (`NNN-slug.md`). When the release is
tagged, `UNRELEASED/` is renamed to `vX.Y.Z/` and a fresh `README.md` is
written for that version. The accumulated `analysis/` files move over
unchanged. If a release ends up with no analysis files, it stays a flat
`vX.Y.Z.md`; delete the empty `UNRELEASED/analysis/` rather than carrying it
forward.

Routine asset swaps, copy fixes, and other changes with nothing non-obvious to
explain don't need an entry.

---

## Release File Structure

Each release file contains these sections in order:

| Section                   | Purpose                                                               |
| ------------------------- | --------------------------------------------------------------------- |
| **Header**                | Version, date, one-line title                                         |
| **What changed**          | Bullet summary of modified files and why                              |
| **Analysis** _(optional)_ | Root-cause reasoning — only when real diagnostic reasoning exists     |
| **Impact**                | What this changes for the running environment                         |
| **Prerequisites**         | Anything that must be true before deploying                           |
| **Step 1 — Local**        | Build, commit, push                                                   |
| **Step 2 — Server**       | Pull, migrate, restart                                                |
| **Verification**          | Commands that prove it worked                                         |
| **Rollback**              | Exact steps to undo if something is wrong                             |

Never edit a released file. If a step was wrong, document the correction in
the next patch release.

---

## Deploy Documentation Rule

Any shell command proposed for a fresh-install or update deploy workflow must
be added to the corresponding `DEPLOY.md` file(s) under
`doc/deployment-environments/`. Add to every file whose workflow the command
affects.

Adapt the directory structure to the environments this project uses, e.g.:

```
doc/deployment-environments/
  001-production/
    001-fresh-install/DEPLOY.md
    002-update/DEPLOY.md
  002-staging/
    001-fresh-install/DEPLOY.md
    002-update/DEPLOY.md
```

When writing a new release file, review and update **all** deployment
documentation files if the release changes anything that affects the install
or update workflow — new dependencies, new seed steps, changed commands, new
verification rows, new troubleshooting entries. Pure content or style changes
that leave the deploy procedure identical do not require updates.

---

## Code Formatting (Prettier)

Before committing any Prettier-supported files, run the formatter from the
repo root.

**Setup** — add to the root `package.json`:

```json
{
    "devDependencies": {
        "prettier": "^3.8.4",
        "prettier-plugin-sh": "^0.18.1"
    },
    "scripts": {
        "format:md":   "prettier \"**/*.md\"   --config .prettierrc.json --write",
        "format:json": "prettier \"**/*.json\" --config .prettierrc.json --write",
        "format:yaml": "prettier \"**/*.yaml\" --config .prettierrc.json --write",
        "format:sh":   "prettier \"**/*.sh\"   --config .prettierrc.json --write",
        "format:path": "bash -c 'prettier --config .prettierrc.json --write \"$@\"' _",
        "format":      "npm run format:json && npm run format:md && npm run format:yaml && npm run format:sh"
    }
}
```

**`.prettierrc.json`:**

```json
{
    "tabWidth": 4,
    "useTabs": false,
    "proseWrap": "always",
    "plugins": ["prettier-plugin-sh"]
}
```

**Usage:**

- Format a specific glob: `npm run format:path -- '<glob>'`
  (e.g. `'doc/**/*.md'`, `'bin/*.sh'`, `'**/*.json'`)
- Format all Markdown: `npm run format:md`
- Format everything: `npm run format`

Supported types: Markdown, JSON, YAML, JS/TS, CSS/SCSS, shell scripts
(via `prettier-plugin-sh`).

---

## Indentation

All code — in every language used in the project — uses **4-space
indentation**. Tabs are not used in source files.

---

## General Coding Rules

- **Idempotent operations.** Seed / import / migration functions guard every
  insert with a check for existing content by slug, ID, or unique key. Safe
  to run repeatedly without side effects.
- **Seed-first rule.** If the project uses seed files as the authoritative
  source of structured data (demo content, reference data, static content),
  a record does not exist as far as the deploy pipeline is concerned until it
  has a corresponding entry in the seed file. Admin UIs and one-off scripts
  are allowed as drafting tools, but any content created through them must be
  reflected in the seed file before — or in the same commit as — the next
  push.
- No inline styles; all styles live in stylesheet partials or component files.
- All user-facing strings support internationalisation from the start, even if
  only one language is planned.
- Don't add features, refactor, or introduce abstractions beyond what the task
  requires. Don't design for hypothetical future requirements.
- No error handling for scenarios that cannot happen. Validate only at system
  boundaries (user input, external APIs, IPC).
- Default to writing no comments. Only add one when the _why_ is non-obvious:
  a hidden constraint, a subtle invariant, a workaround for a specific bug.
