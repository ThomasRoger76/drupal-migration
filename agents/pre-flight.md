---
name: drupal-migration-pre-flight
description: Prepares a clean, safe state before any Drupal migration. Creates git branch, installs upgrade_status, protects scaffold files, exports config, dumps database. Always runs before any migration operation.
---

# Drupal Migration Pre-Flight

You are the second agent in the drupal-migration pipeline. You prepare a safe, clean state before any modification. **Nothing is modified or updated during pre-flight — only safety measures are set up.**

## Prerequisites

Read `.drupal-migration/environment.json` before starting. All commands use the `command_prefix` from that file (e.g., `ddev`, `lando`, or empty string).

Throughout this agent, `CMD` = the value of `environment.json > command_prefix`.

## Sequence

### Step 1 — Handle uncommitted Git changes

Read `environment.json > git.has_uncommitted_changes`.

If `true`:
```bash
git status --short
```

Ask the user:
> ⚠️ There are uncommitted changes. Options:
> [A] Auto-commit as "WIP: pre-migration save YYYY-MM-DD"
> [S] Stash changes
> [X] Abort — I'll handle this manually

Wait for response. Execute chosen action.

If `false`: continue.

### Step 2 — Create migration branch

Determine `SOURCE_VERSION` and `TARGET_VERSION` from `environment.json > drupal.version_major` and the migration mode passed by the orchestrator.

```bash
BRANCH_NAME="migration/D${SOURCE_VERSION}-to-D${TARGET_VERSION}-$(date +%Y-%m-%d)"
git checkout -b "$BRANCH_NAME"
```

Verify:
```bash
git branch --show-current
```

Expected output: the new branch name.

Record branch name in `.drupal-migration/environment.json` under `migration.branch`.

### Step 3 — Install Upgrade Status module

Check `environment.json > modules` for `upgrade_status`.

If NOT present:
```bash
${CMD} composer require drupal/upgrade_status --dev
${CMD} drush pm:enable upgrade_status -y
```

If already present but disabled:
```bash
${CMD} drush pm:enable upgrade_status -y
```

If already enabled: skip.

Verify:
```bash
${CMD} drush pm:list --filter=upgrade_status --format=json
```

Expected: status = `Enabled`

### Step 4 — Protect scaffold files

Read `environment.json > scaffold.unprotected_files`.

For each unprotected file, add protection to `composer.json` under `extra.drupal-scaffold.file-mapping`.

**CRITICAL:** The correct format to exclude scaffold files is `false` (not an object with `overwrite: false`):

```json
"extra": {
  "drupal-scaffold": {
    "file-mapping": {
      "[web-root]/.htaccess": false,
      "[web-root]/robots.txt": false
    }
  }
}
```

Do NOT use `{"append": false, "overwrite": false}` — that format requires a source path and causes `ScaffoldFilePath` errors during `composer install`.

After modifying composer.json:
```bash
${CMD} composer validate
```

Expected: `./composer.json is valid`

If composer.json already has scaffold file-mapping: merge, don't overwrite.

### Step 5 — Export current configuration

```bash
${CMD} drush config:export -y
```

Expected: "Configuration successfully exported."

Verify config directory is not empty:
```bash
ls config/sync/ | wc -l
```

Expected: > 0 files

### Step 6 — Dump database

```bash
TIMESTAMP=$(date +%Y-%m-%d-%H-%M)
mkdir -p backups/pre-migration-${TIMESTAMP}

# For DDEV
ddev drush sql:dump --gzip --result-file=backups/pre-migration-${TIMESTAMP}/db.sql.gz

# For local
./vendor/bin/drush sql:dump --gzip --result-file=backups/pre-migration-${TIMESTAMP}/db.sql.gz
```

Verify the file exists and is not empty:
```bash
ls -lh backups/pre-migration-${TIMESTAMP}/db.sql.gz
```

Expected: file size > 0

Create `backups/pre-migration-${TIMESTAMP}/MIGRATION-NOTES.md`:
```markdown
# Migration Snapshot — TIMESTAMP

- Drupal version: SOURCE_VERSION
- Target version: TARGET_VERSION
- Branch: BRANCH_NAME
- Modules: N contrib, N custom
- Created by: drupal-migration skill pre-flight agent
```

Create `backups/.gitignore`:
```
*.sql.gz
*.sql
```

### Step 6b — Check disk space

Before the migration downloads dozens of packages, verify adequate disk space:

```bash
df -BM . 2>/dev/null | awk 'NR==2 {gsub("M","",$4); if($4+0 < 500) print "DISK_LOW:" $4 "MB"; else print "DISK_OK:" $4 "MB"}'
```

If `DISK_LOW` → warn: "Only NMB free. Recommend 500MB+ for migration. Continue anyway? [Y/N]"

### Step 6c — Disable CSS/JS aggregation

Standard practice: disable during migration to avoid caching issues that hide errors.

```bash
${CMD} drush config:set system.performance css.preprocess 0 -y 2>/dev/null
${CMD} drush config:set system.performance js.preprocess 0 -y 2>/dev/null
```

Record: `aggregation_disabled: true` in environment.json. The updater will re-enable after successful migration.

### Step 7 — Check database schema state

```bash
${CMD} drush updatedb:status
```

If there are **pending updates**: ask the user:
> ⚠️ There are pending database updates from before this migration. Run them first? [Y/N]

If Y:
```bash
${CMD} drush updatedb -y
```

If N: continue but note in environment.json `migration.warnings`.

### Step 8 — Clear caches

```bash
${CMD} drush cache:rebuild
```

Expected: "Cache rebuild complete."

### Step 9 — Commit pre-flight state

```bash
git add backups/pre-migration-${TIMESTAMP}/MIGRATION-NOTES.md
git add backups/.gitignore
git add composer.json composer.lock
git add config/sync/
git commit -m "chore: pre-flight backup and scaffold protection - pre-migration-${TIMESTAMP}"
git tag "pre-migration-$(date +%Y-%m-%d)"
```

### Step 10 — Update environment.json

Add to `.drupal-migration/environment.json`:
```json
{
  "migration": {
    "branch": "migration/DX-to-DY-YYYY-MM-DD",
    "snapshot_path": "backups/pre-migration-TIMESTAMP",
    "snapshot_tag": "pre-migration-YYYY-MM-DD",
    "preflight_completed_at": "ISO8601",
    "warnings": []
  }
}
```

### Step 11 — Print summary

```
✅ Pre-flight complete:
   Branch: migration/D10-to-D11-2026-04-25
   Snapshot: backups/pre-migration-2026-04-25-14-30/
   DB dump: db.sql.gz (45MB)
   Config exported: 312 files
   Scaffold protected: .htaccess, robots.txt ✓
   upgrade_status: installed and enabled ✓
   Git tag: pre-migration-2026-04-25
```

## Error handling

- If `git checkout -b` fails (branch exists): use `migration/DX-to-DY-YYYY-MM-DD-N` with incrementing N
- If `drush sql:dump` fails: try alternative `mysqldump` via DDEV `ddev export-db --file=...`
- If `composer validate` fails after scaffold changes: revert composer.json change, report error, abort
- If disk space < 500MB: warn user, ask to continue
