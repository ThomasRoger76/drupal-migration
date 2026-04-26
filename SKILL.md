---
name: drupal-migration
description: Automates Drupal updates and major version migrations (D9->D10, D10->D11, D11->D12+) in local environments. Multi-agent pipeline with backup, compatibility analysis, auto-fix, testing, and rollback. Supports DDEV, Lando, Docker, and classic local installs. Triggers on: /drupal-migrate, /drupal-update, /drupal-status, /drupal-rollback, "migrer drupal", "mise à jour drupal", "upgrade drupal", "montée de version drupal".
---

# Drupal Migration Skill

Automate Drupal security patches, minor updates, and major version migrations with zero manual intervention and guaranteed rollback.

## When This Skill Is Invoked

**Slash commands:**
- `/drupal-migrate` — Major version upgrade (prompts for target version)
- `/drupal-update` — Security patch or minor update (auto-detects what's available)
- `/drupal-status` — Dry-run analysis only (no modifications)
- `/drupal-rollback` — Restore from last snapshot
- `/drupal-dry-run` — Full simulation: what WOULD happen (no changes, generates a plan)
- `/drupal-cleanup-tests` — Delete all AgentIA test content from the site
- `/drupal-resume` — Resume an interrupted migration from last completed step

**Auto-trigger keywords:**
- "migrer drupal", "migration drupal", "upgrade drupal"
- "mettre à jour drupal", "mise à jour drupal", "drupal update"
- "D9 vers D10", "D10 vers D11", "D11 vers D12"
- "montée de version drupal", "modules drupal à jour"
- "sécurité drupal", "drupal outdated"

## Safety Check — ALWAYS FIRST

Before doing anything else, verify we are NOT in production by checking ANY of:
1. `$settings['environment']` = `'production'` in settings.php
2. Drush alias contains `@prod` or `@production`
3. `.env` file contains `APP_ENV=production` or `DRUPAL_ENV=production`

If ANY condition is true:
```
🚫 REFUSED: Production environment detected.
This skill only runs in local environments.
To override (dangerous): set DRUPAL_MIGRATION_ALLOW_PROD=1
```

## Mode Detection

| Command/Keyword | Mode |
|---|---|
| `/drupal-status` or "drupal status" | `status` |
| `/drupal-rollback` | `rollback` |
| `/drupal-update` or "mise à jour sécurité" | `update` |
| `/drupal-migrate` or "migration" or "vers D11" | `migrate` |

If unclear, ask:
> What would you like to do?
> [1] Check status only (no changes)
> [2] Apply security/minor updates
> [3] Major version migration (D10→D11, etc.)
> [4] Rollback to last snapshot
> [5] Dry-run — simulate migration without changes
> [6] Cleanup test content (AgentIA nodes)
> [7] Resume interrupted migration

## Pipeline state tracking

After each completed agent step, update `.drupal-migration/environment.json > migration.state`:

```json
{
  "migration": {
    "state": "env-detected|preflight|patches-validated|compatibility|code-fixed|baseline|updating|db-health|config-doctor|patch-post|testing|complete",
    "last_completed_step": "preflight",
    "last_completed_at": "ISO8601",
    "target_version": 11
  }
}
```

**On `/drupal-resume`**: Read `migration.last_completed_step`, skip already-completed steps, resume from the next one. Print what was already done and where we're resuming.

## MODE: dry-run

Triggered by `/drupal-dry-run` or "que se passerait-il si":

1. **env-detector** — detect current state
2. **compatibility-analyzer** — full scan, no modifications
3. **patch-manager** (phase: pre-migration, dry-run only) — validate patches
4. Run `composer update --dry-run` to see what would change
5. Generate a migration plan document WITHOUT making any changes

Output: `.drupal-migration/dry-run-plan-YYYY-MM-DD.md` with:
- What packages would be updated (from composer dry-run)
- What patches would fail
- What custom code would need fixing
- Estimated manual effort required

No state changes, no git branch, no DB dump.

## MODE: cleanup-tests

Triggered by `/drupal-cleanup-tests` or "nettoyer les noeuds de test":

```bash
${CMD} drush php:eval "
\$uid = \\Drupal::entityQuery('user')->condition('name','AgentIA')->accessCheck(FALSE)->execute();
if (empty(\$uid)) { echo 'AgentIA user not found'; exit; }
\$uid = reset(\$uid);
\$nids = \\Drupal::entityQuery('node')->condition('uid', \$uid)->accessCheck(FALSE)->execute();
\$pids = \\Drupal::entityQuery('paragraph')->accessCheck(FALSE)->execute();
// Note: paragraphs don't have uid, so delete only nodes owned by AgentIA
\\Drupal::entityTypeManager()->getStorage('node')->delete(
  \\Drupal::entityTypeManager()->getStorage('node')->loadMultiple(\$nids)
);
echo 'Deleted ' . count(\$nids) . ' test nodes owned by AgentIA' . PHP_EOL;
" 2>/dev/null
```

Also clean up:
- `.drupal-migration/baseline/` directory
- `.drupal-migration/validate/` directory  
- `.drupal-migration/test-baseline.json`

Print: `✅ Cleaned N test nodes. Baseline snapshots removed.`

## MODE: status

Run env-detector agent. Present environment.json summary. List available updates. No modifications.

Invoke env-detector by reading `~/.claude/skills/drupal-migration/agents/env-detector.md` and following its instructions in the current Drupal project directory.

After detection, run:
```bash
CMD drush pm:security
CMD composer outdated drupal/
CMD drush upgrade_status:analyze --all 2>/dev/null || echo "upgrade_status not installed — run /drupal-update to install it"
```

Present a clean report. No changes made.

## MODE: rollback

Check `.drupal-migration/environment.json` for last snapshot.

If no snapshot found:
```
❌ No migration snapshot found. Cannot rollback.
Run /drupal-status first, or ensure a migration was previously started with this skill.
```

If snapshot found: read `~/.claude/skills/drupal-migration/agents/rollback-manager.md` and follow its instructions with:
- `failed_agent`: "manual rollback request"
- `failed_step`: "user requested rollback"
- `error_output`: "Manual rollback requested by user"
- `changes_made`: "Unknown — manual rollback"

## MODE: update (patch/minor)

Full pipeline:

1. **env-detector** — read `~/.claude/skills/drupal-migration/agents/env-detector.md`
2. **pre-flight** — read `~/.claude/skills/drupal-migration/agents/pre-flight.md` (target = same major, next minor)
3. **compatibility-analyzer** — read `~/.claude/skills/drupal-migration/agents/compatibility-analyzer.md` with `migration_mode: minor`
4. Present report, ask: "Apply N updates? [Y/N]"
5. If Y: **test-runner (baseline)** — read `~/.claude/skills/drupal-migration/agents/test-runner.md` with `phase: baseline`
6. **updater** — read `~/.claude/skills/drupal-migration/agents/updater.md` with `migration_mode: minor`
7. **test-runner (validate)** — read `~/.claude/skills/drupal-migration/agents/test-runner.md` with `phase: validate`, pass `report_file` from updater
8. If critical tests fail → **rollback-manager**
9. Final commit with report in Drupal project (done by test-runner step V7)

## MODE: migrate (major)

Ask for target version if not specified:
> Target version? (e.g., 11 for D10→D11, 12 for D11→D12)

Determine `deprecations_key`:
- D9→D10: `D9_to_D10`
- D10→D11: `D10_to_D11`
- D11→D12: `D11_to_D12` (will be added to known-deprecations.json when D12 releases)

Full pipeline:
1.  **env-detector** — read `agents/env-detector.md` (includes DB health pre-check via db-health agent)
    → Update `migration.state: env-detected`
2.  **pre-flight** — read `agents/pre-flight.md` with source/target versions
    → Disk space check, CSS/JS aggregation disable, DB dump, config export
    → Update `migration.state: preflight`
3.  **patch-manager** (phase: pre-migration) — read `agents/patch-manager.md`
    → Validate all patches against target versions
    → Update `migration.state: patches-validated`
4.  **compatibility-analyzer** — read `agents/compatibility-analyzer.md` with `migration_mode: major`, `deprecations_key`
    → Includes Symfony 7 scan, Views handler check, known module bugs, patch pre-validation
    → Update `migration.state: compatibility`
5.  Present blocking issues, wait for user decisions per issue (skip if autonomous mode)
6.  If fixable issues: → **code-fixer** — read `agents/code-fixer.md`
    → Update `migration.state: code-fixed`
7.  **test-runner (baseline)** — read `agents/test-runner.md` with `phase: baseline`
    → Creates test content + paragraphs + navigation crawl + HTML snapshots
    → Update `migration.state: baseline`
8.  **updater** — read `agents/updater.md` with `migration_mode: major`
    → Composer update + Drush upgrade + DB updates + config-doctor + db-health + patch-manager post + deployment checklist
    → Update `migration.state: updating`
9.  **test-runner (validate)** — read `agents/test-runner.md` with `phase: validate`
    → HTML diff, creation/edit API test, navigation replay + comparison
    → Update `migration.state: testing`
10. If critical tests fail → **rollback-manager**
11. Final commit with full report in Drupal project (done by test-runner step V10)
    → Update `migration.state: complete`

### Multi-hop migrations (e.g., D9 → D11)

If source + target span more than one major:
1. Ask: "D9→D11 requires two passes: D9→D10 then D10→D11. Proceed?"
2. Run full pipeline for D9→D10
3. If successful, run full pipeline for D10→D11

## MODE: analyze (dry-run complet)

Triggered by `/drupal-status --full`, "analyser compatibilité drupal", "vérifier compatibilité":

1. **env-detector** — read `~/.claude/skills/drupal-migration/agents/env-detector.md` (no modifications)
2. **compatibility-analyzer** — read `~/.claude/skills/drupal-migration/agents/compatibility-analyzer.md` (no pre-flight, no modifications)
3. Present full `compatibility-report.md`
4. No changes to the site

This is the safe way to preview what a migration would require.

## How to invoke agents

Each agent file is read with the Read tool then its instructions are followed directly. Always pass the environment.json content as context.

Pattern:
1. Read `~/.claude/skills/drupal-migration/agents/AGENT-NAME.md`
2. Read `.drupal-migration/environment.json` from the Drupal project
3. Follow the agent's instructions with full context

## On any critical error

Immediately read and follow `~/.claude/skills/drupal-migration/agents/rollback-manager.md` with:
- `failed_agent`: name of the step that failed
- `failed_step`: exact description
- `error_output`: full error text
- `changes_made`: list everything modified before the error
