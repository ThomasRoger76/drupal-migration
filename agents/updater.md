---
name: drupal-migration-updater
description: Executes the actual Drupal update or major version migration. Runs Composer update with correct version constraints, drush updatedb, config import, cache rebuild. Generates the final detailed migration report committed to the Drupal project repo.
---

# Drupal Migration Updater

You are the fifth agent in the drupal-migration pipeline. You execute the actual update. This is where packages are upgraded, database updates run, and config is synced. **After this agent completes, the test-runner validates the result.**

## Prerequisites

Read `.drupal-migration/environment.json`. `CMD` = `environment.json > command_prefix`.

Also read:
- `.drupal-migration/compatibility-summary.json` — blocked modules to skip
- `.drupal-migration/test-baseline.json` — pre-migration test baseline

## Inputs from orchestrator

- `migration_mode`: `patch` | `minor` | `major`
- `source_version`: e.g., 10
- `target_version`: e.g., 11
- `start_time`: ISO8601 timestamp when migration started

## Step 1 — Capture pre-update module versions

Before touching anything, snapshot current package versions from `composer.lock`. Store as `pre_update_versions` map: `{package_name: version}`.

## Step 2 — Enable maintenance mode

```bash
${CMD} drush state:set system.maintenance_mode 1 --input-format=integer
${CMD} drush cache:rebuild
```

Print: `🔧 Maintenance mode enabled`

## Step 2b — Drush version check and upgrade

For major mode (D10→D11+), check if Drush needs upgrading:

```bash
${CMD} composer show drush/drush 2>/dev/null | grep "^versions" | grep -o "\* [0-9]*\.[0-9]*" | head -1
```

- D11 target: require `drush/drush: ^13`
- If current is `^12`: add Drush upgrade to the composer.json changes (update in same run as core)
- Drush must be updated SIMULTANEOUSLY with core, not separately, to avoid incompatible drush/core combinations during the update window

```json
// Add to composer.json require if upgrading D10→D11:
"drush/drush": "^13"
```

## Step 2c — Lock file repair strategy

If `composer.lock` is in an inconsistent state (partial previous update, conflicting path repositories, etc.):

```bash
# Check lock file consistency
${CMD} composer validate 2>&1 | grep -E "ERROR|lock file"
```

If lock file errors found:
1. Try: `${CMD} composer update --lock 2>&1` (regenerates lock without installing)
2. If still broken: `${CMD} composer install --no-scripts 2>&1` (installs from existing lock)
3. If still broken: `rm composer.lock && ${CMD} composer install 2>&1` (nuclear option — note in report)

**Never use path repositories for packages managed by Composer** — they cause "Failed to realpath" errors when Composer removes then tries to re-install. Instead, patch the package's `composer.json` via the `patches` mechanism.

## Step 3 — Execute update

### For patch/minor mode:

```bash
${CMD} composer update drupal/core-recommended drupal/core-composer-scaffold --with-all-dependencies --no-interaction 2>&1
```

### For major mode:

First, update core version constraint in `composer.json` using Edit tool:
- `"drupal/core-recommended": "^SOURCE"` → `"drupal/core-recommended": "^TARGET"`
- `"drupal/core-composer-scaffold": "^SOURCE"` → `"drupal/core-composer-scaffold": "^TARGET"`
- `"drupal/core-dev": "^SOURCE"` → `"drupal/core-dev": "^TARGET"` (if present)
- `"drush/drush": "^12"` → `"drush/drush": "^13"` (if upgrading to D11)

Then:
```bash
${CMD} composer update --with-all-dependencies --no-interaction 2>&1
```

**Watch for patch failures during composer update:**
```
Could not apply patch! Skipping.
```
This is a silent failure that leaves contrib modules un-patched. After the update completes, verify each patch from `composer.json > extra.patches` was actually applied. See patch-manager agent (post-migration phase) for validation.

**On Composer conflict:**
1. Read full error output, identify conflicting packages
2. Try updating conflicting packages individually
3. If conflict involves blocked module from compatibility-summary.json: `composer remove drupal/MODULE --no-update` then retry
4. If still failing after 2 attempts → call rollback-manager with full error

On success: print `✅ Composer update complete`

## Step 4 — Run database updates

```bash
${CMD} drush updatedb --no-interaction -y 2>&1
```

- "already up to date" → fine, continue
- actual error → call rollback-manager

On success: print `✅ Database updates applied`

## Step 5 — Import configuration

```bash
${CMD} drush config:import -y 2>&1
```

"No changes to import" → fine. Config mismatch → note in report, continue.

## Step 6 — Rebuild caches

```bash
${CMD} drush cache:rebuild 2>&1
```

## Step 6b — Run config-doctor (post-update)

Read and follow `~/.claude/skills/drupal-migration/agents/config-doctor.md`. This validates:
- Entity definition updates (critical — data corruption if skipped)
- Orphaned schema entries
- Broken Views handlers
- config_split health

## Step 6c — Run db-health (post-update)

Read and follow `~/.claude/skills/drupal-migration/agents/db-health.md` with `phase: post`.

## Step 6d — Run patch-manager (post-update)

Read and follow `~/.claude/skills/drupal-migration/agents/patch-manager.md` with `phase: post-migration`.
This verifies patches applied, archives obsolete ones, and formalizes manual fixes as patches.

## Step 7 — Re-enable CSS/JS aggregation

If pre-flight disabled CSS/JS aggregation (`environment.json > aggregation_disabled == true`):

```bash
${CMD} drush config:set system.performance css.preprocess 1 -y 2>/dev/null
${CMD} drush config:set system.performance js.preprocess 1 -y 2>/dev/null
```

## Step 7b — Disable maintenance mode

```bash
${CMD} drush state:set system.maintenance_mode 0 --input-format=integer
${CMD} drush cache:rebuild
```

## Step 8 — Verify site responds

```bash
${CMD} drush status --format=json 2>/dev/null
```

Check: `bootstrap` = "Successful", `drupal-version` matches target. If site not responding → call rollback-manager.

## Step 9 — Capture post-update module versions

Read updated `composer.lock`, store as `post_update_versions` map.

## Step 9b — Generate deployment checklist

Create `DEPLOYMENT-CHECKLIST-YYYY-MM-DD.md` in the project root. This is the handoff document for deploying the migration to production:

```markdown
# Deployment Checklist — D{SOURCE}→D{TARGET} — YYYY-MM-DD

## Pre-deployment requirements
- [ ] PHP VERSION_REQUIRED available on production server
- [ ] MariaDB/MySQL VERSION_REQUIRED on production
- [ ] At least 500MB free disk space on production
- [ ] Maintenance window scheduled (estimated: 15-30 min)
- [ ] Production DB backup taken immediately before deployment
- [ ] Team notified of maintenance window

## Deployment commands (in order)

\`\`\`bash
# 1. Deploy code
git pull origin migration/D{SOURCE}-to-D{TARGET}-YYYY-MM-DD

# 2. Install dependencies
composer install --no-dev --optimize-autoloader

# 3. Enable maintenance mode
drush state:set system.maintenance_mode 1

# 4. Run database updates
drush updatedb -y

# 5. Import configuration
drush config:import -y

# 6. Rebuild caches
drush cache:rebuild

# 7. Disable maintenance mode
drush state:set system.maintenance_mode 0
drush cache:rebuild
\`\`\`

## Manual steps required on production
[List from compatibility-summary.json > warnings and patch-report.md]
PLACEHOLDER_MANUAL_STEPS

## Post-deployment validation (run within 5 minutes)
- [ ] Homepage loads (HTTP 200)
- [ ] Admin login works
- [ ] `drush status` shows Drupal TARGET_VERSION
- [ ] `drush updatedb:status` shows "No database updates required"
- [ ] Search API still indexed
- [ ] Key Views pages return rows (not empty)

## Rollback procedure (if needed)
\`\`\`bash
drush state:set system.maintenance_mode 1
drush sql:drop -y
drush sql:cli < <(gunzip -c /path/to/pre-migration-backup.sql.gz)
git checkout pre-migration-TAG -- composer.json composer.lock
composer install
drush cache:rebuild
drush state:set system.maintenance_mode 0
\`\`\`

## Notes
- Migration branch: migration/D{SOURCE}-to-D{TARGET}-YYYY-MM-DD
- Git tag (pre-migration): pre-migration-YYYY-MM-DD
- DB backup: backups/pre-migration-YYYY-MM-DD/db.sql.gz.gz
- Migration report: MIGRATION-REPORT-YYYY-MM-DD.md
```

## Step 10 — Generate final migration report

**This report is generated IN THE DRUPAL PROJECT, not in ~/.claude.**

Create `MIGRATION-REPORT-YYYY-MM-DD.md` in the project root:

```markdown
# Migration Report — D{SOURCE} → D{TARGET} — TIMESTAMP

## Summary
- **Status:** ✅ SUCCESS
- **Duration:** N minutes
- **Source:** Drupal SOURCE_VERSION
- **Target:** Drupal TARGET_VERSION

---

## Modules Updated (contrib)

| Module | Before | After | Status |
|--------|--------|-------|--------|
[For each package where pre_version != post_version and name starts with drupal/:]
| drupal/MODULE | BEFORE | AFTER | ✅ Updated |
[For each blocked module from compatibility-summary:]
| drupal/MODULE | VERSION | — | ⚠️ Skipped (user choice) |

**Total contrib:** N updated, N skipped

---

## Custom Modules Modified

| Module | Changes Applied |
|--------|----------------|
[For each module in compatibility-summary.modules_fixed:]
| MODULE_NAME | list of fixes applied |

---

## Core Update

| Component | Before | After |
|-----------|--------|-------|
| drupal/core-recommended | SOURCE_VERSION | TARGET_VERSION |
| PHP | PHP_VERSION | PHP_VERSION |

---

## Tests Results

TESTS_PLACEHOLDER

---

## Git History (this migration)

[output of: git log pre-migration-DATE..HEAD --oneline]

---

## Snapshot Available

- DB backup: `backups/pre-migration-TIMESTAMP/db.sql.gz`
- Git tag: `pre-migration-DATE`

---

## Warnings

[list any warnings: blocked modules, config mismatches, etc.]

---

*Generated by drupal-migration skill — TIMESTAMP*
```

## Step 11 — Preliminary commit (without test results)

```bash
git add MIGRATION-REPORT-YYYY-MM-DD.md
git commit -m "chore: preliminary migration report D{SOURCE}→D{TARGET} YYYY-MM-DD (tests pending)"
```

## Step 12 — Signal completion to orchestrator

Return:
```
{
  "status": "update_complete",
  "actual_version": "VERSION_FROM_DRUSH_STATUS",
  "report_file": "MIGRATION-REPORT-YYYY-MM-DD.md",
  "pre_versions": {...},
  "post_versions": {...},
  "duration_minutes": N
}
```

## Error handling

- Composer timeout (>10min) → wait, do NOT rollback automatically
- `drush updatedb` hook failure → capture, present to user before rollback decision
- Config import mismatch → note in report, non-blocking
- Site 500 after update → immediate rollback-manager invocation

## Validation

```bash
wc -l /home/thomasroger/.claude/skills/drupal-migration/agents/updater.md
head -3 /home/thomasroger/.claude/skills/drupal-migration/agents/updater.md
```

## Commit

```bash
git -C /home/thomasroger/.claude add skills/drupal-migration/agents/updater.md
git -C /home/thomasroger/.claude commit -m "feat: add updater agent for drupal-migration skill"
```
