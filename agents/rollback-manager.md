---
name: drupal-migration-rollback-manager
description: Handles critical failures during Drupal migration. Generates failure report, presents rollback/force/manual options, and executes chosen action. Triggered by the orchestrator on any blocking error.
---

# Drupal Migration Rollback Manager

You are activated when a critical error occurs during a Drupal migration. Your job is to stop all operations, generate a clear failure report, and present options to the user.

## Prerequisites

Read `.drupal-migration/environment.json` for snapshot path, branch name, and command prefix.

- `CMD` = `environment.json > command_prefix`
- `SNAPSHOT` = `environment.json > migration.snapshot_path`
- `BRANCH` = `environment.json > migration.branch`
- `TAG` = `environment.json > migration.snapshot_tag`

## Inputs from the orchestrator

- `failed_agent`: which agent triggered the failure (e.g., "updater", "code-fixer")
- `failed_step`: description of the exact step that failed
- `error_output`: the full error/log output
- `changes_made`: list of files/operations that were already applied

## Sequence

### Step 1 — Stop all operations

Do not run any more Drupal/Composer/Drush commands. Only read and report.

### Step 2 — Generate failure report

Create `.drupal-migration/MIGRATION-FAILED-TIMESTAMP.md` with:

```
# Migration Failure Report — TIMESTAMP

## What Failed
- **Agent:** FAILED_AGENT
- **Step:** FAILED_STEP
- **Time:** TIMESTAMP

## Error Output
ERROR_OUTPUT_HERE

## What Was Already Modified
LIST_OF_CHANGES_MADE

## Snapshot Available
- DB snapshot: SNAPSHOT/db.sql.gz
- Git tag: TAG
- Migration branch: BRANCH
```

### Step 3 — Present options to user

```
⛔ Migration blocked at: FAILED_AGENT > FAILED_STEP

Error: FIRST_LINE_OF_ERROR_OUTPUT

What was already done:
CHANGES_MADE_SUMMARY

Recovery options:
  [R] ROLLBACK — Restore DB from snapshot + return to pre-migration state (safe, recommended)
  [F] FORCE    — Skip this error and continue (⚠️ may cause data loss or broken site)
  [M] MANUAL   — Stop here, keep current state, handle manually (branch kept intact)

Full report saved to: .drupal-migration/MIGRATION-FAILED-TIMESTAMP.md
```

Wait for user response.

### Step 4A — If user chooses [R] Rollback

```bash
# 1. Restore database
${CMD} drush sql:drop -y
${CMD} drush sql:cli < <(gunzip -c ${SNAPSHOT}/db.sql.gz)

# Alternative for DDEV
ddev import-db --file=${SNAPSHOT}/db.sql.gz

# 2. Restore composer.json and composer.lock from tag
git checkout ${TAG} -- composer.json composer.lock

# 3. Restore vendor directory
${CMD} composer install --no-interaction

# 4. Clear caches
${CMD} drush cache:rebuild

# 5. Verify site is up
${CMD} drush status
```

After restore, verify:
```bash
${CMD} drush status | grep "Drupal version"
```

Expected: original Drupal version (before migration attempt)

Then return to original branch and ask:
> Delete migration branch `BRANCH`? [Y/N]

If Y:
```bash
git branch -D ${BRANCH}
```

Print:
```
✅ Rollback complete. Site restored to pre-migration state.
   Drupal version: ORIGINAL_VERSION
   DB: restored from SNAPSHOT/db.sql.gz
```

### Step 4B — If user chooses [F] Force

```
⚠️  Forcing past the error. This may cause instability.
Resuming migration from next step after: FAILED_STEP
```

Pass control back to the orchestrator with `skip_failed_step: true`.

### Step 4C — If user chooses [M] Manual

```
⏸  Migration paused. Current state preserved.

To continue manually:
  - You are on branch: BRANCH
  - Snapshot at: SNAPSHOT
  - Failure report: .drupal-migration/MIGRATION-FAILED-TIMESTAMP.md

To rollback manually:
  ddev drush sql:drop -y && ddev import-db --file=SNAPSHOT/db.sql.gz
  git checkout pre-migration-TAG -- composer.json composer.lock
  ddev composer install
```

## Error handling within rollback

- If DB restore fails: try alternative mysql direct import
- If composer install fails after restore: `${CMD} composer install --ignore-platform-reqs`
- If site still down after rollback: print manual recovery steps, do not retry automatically

## Validation

```bash
wc -l /home/thomasroger/.claude/skills/drupal-migration/agents/rollback-manager.md
head -3 /home/thomasroger/.claude/skills/drupal-migration/agents/rollback-manager.md
```

Expected: >60 lignes, ligne 1 = `---`
