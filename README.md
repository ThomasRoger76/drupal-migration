# drupal-migration — Claude Code Skill

> Automated Drupal security patches, minor updates, and major version migrations with zero manual intervention and guaranteed rollback.

---

## Summary

**drupal-migration** is a [Claude Code](https://claude.ai/code) skill that automates the full lifecycle of Drupal upgrades — from `D9 → D10 → D11 → D12` — directly from your local development environment.

It operates as a **multi-agent pipeline** of 10 specialized agents that handle every aspect of the migration: environment detection, patch validation, compatibility analysis, code auto-fixing, database health checks, content-based regression testing, configuration validation, and production deployment checklist generation.

### At a glance

| Property | Value |
|----------|-------|
| Skill trigger | `/drupal-migrate`, `/drupal-update`, `/drupal-status` |
| Agents | 10 |
| Config files | 1 (`known-deprecations.json`) |
| Supported environments | DDEV, Lando, Docker, local PHP |
| Supported Drupal versions | 9, 10, 11 (D12 ready) |
| Rollback | Automatic on critical failure |

### What it does that manual migrations don't

- **Pre-validates every composer patch** against the target module version before starting
- **Detects silent failures**: a patch that "Skips" during `composer update` without error
- **Catches Symfony 7 breaking changes** in custom code before they cause a 500
- **Captures HTML snapshots** of every page before migration, then diffs them after
- **Tracks Views row counts** — a View with 42 rows before that returns 0 after is flagged
- **Tests content creation AND editing** via Drupal API before and after, compares results
- **Generates a production deployment checklist** tailored to what actually changed

---

## Agents

### Pipeline overview (major migration mode)

```
1. env-detector        Detect environment, PHP/DB versions, modules, CI, theme tooling
2. pre-flight          DB dump, config export, CSS/JS disable, scaffold protection
3. patch-manager       Pre-validate composer patches against target versions
4. compatibility-analyzer  Scan custom code: deprecated APIs, Symfony 7, module bugs
5. code-fixer          Auto-fix deprecated PHP, annotations→attributes, Twig, return types
6. test-runner         Baseline: create test content + paragraphs + navigation crawl
7. updater             composer update + Drush upgrade + DB updates + deployment checklist
   └─ config-doctor    Entity definitions, broken Views, config_split health
   └─ db-health        DB version, JSON support, orphaned entity tables
   └─ patch-manager    Verify patches applied, archive obsolete, formalize manual fixes
8. test-runner         Validate: HTML diff, create/edit API regression, nav comparison
9. rollback-manager    Auto-invoked on critical failure — DB restore + git checkout
```

---

### `env-detector`

Detects everything about the local environment before any operation.

**What it detects:**
- Environment type: DDEV / Lando / Docker / local PHP
- Drupal version, PHP version, Drush version, database driver
- All contrib, custom, and core modules (with **correct custom module detection** — finds modules in `sites/default/modules/` and other non-standard paths)
- All themes (contrib + custom)
- Special modules: Paragraphs, Search API, Webform, Encrypt, Group, config_split
- Git state: branch, uncommitted changes, recent tags
- Scaffold file protection status
- **PHP version compatibility** against target Drupal requirements
- **MariaDB/MySQL version** with minimum requirements check (D11 requires MariaDB 10.6+)
- Composer version (requires Composer 2)
- CI files: `.gitlab-ci.yml`, `.github/workflows/`, etc.
- Theme compilation: webpack, gulp, npm-scripts

**Outputs:** `.drupal-migration/environment.json`

---

### `pre-flight`

Prepares a safe, clean state before any modification.

**Steps:**
1. Handle uncommitted git changes (auto-commit WIP or stash)
2. Create migration branch: `migration/D10-to-D11-YYYY-MM-DD`
3. Install and enable `drupal/upgrade_status`
4. Protect scaffold files in `composer.json` (correct format: `false` value, not object)
5. **Disk space check** (warns if < 500MB free)
6. **Disable CSS/JS aggregation** (re-enabled after successful migration)
7. Export current config to sync directory
8. Dump database to `backups/pre-migration-YYYY-MM-DD/db.sql.gz`
9. Check for pre-existing pending DB updates
10. Rebuild caches
11. Commit pre-flight state + git tag

**Outputs:** DB dump, config export, migration branch, git tag

---

### `patch-manager`

Manages composer patches across the entire migration lifecycle.

**Phase: pre-migration**
- Validates each patch in `composer.json > extra.patches` against the target module version
- Classifies patches as: `VALID` | `OBSOLETE` (fix included in target version) | `WILL_FAIL`
- Removes obsolete patches automatically; flags failing ones for manual resolution
- Prevents the silent "Could not apply patch! Skipping." scenario

**Phase: post-migration**
- Verifies each patch was actually applied (not silently skipped)
- Generates `.patch` files from manual fixes applied to contrib modules during migration
- Archives patches that are no longer needed

**Outputs:** `.drupal-migration/patch-report.md`, updated `composer.json`

---

### `compatibility-analyzer`

Full compatibility scan before any code is touched.

**What it scans:**

| Check | Severity |
|-------|----------|
| Security vulnerabilities (`composer audit`) | 🔴/🟡/🟢 |
| Available module updates | 🟡 |
| Upgrade Status analysis | 🔴/🟡/🟢 |
| Contrib module D{TARGET} version availability | 🔴 if no release |
| **Custom modules** (all paths, not just `modules/custom/`) | 🔴/🟡/🟢 |
| PHP deprecated functions | 🔴 |
| PHP return types missing (`:void` enforcement D11) | 🔴 |
| **Symfony 7 breaking changes** (GetResponseEvent, etc.) | 🔴 |
| Annotations → PHP 8 Attributes | 🟡 |
| `.info.yml` core_version_requirement | 🟡 |
| Twig deprecated syntax | 🔴 |
| PHPExcel API (abandoned, incompatible with phpspreadsheet 2.x) | 🔴 |
| **Known contrib module bugs** | 🔴 auto-fixable |
| Views broken handlers (pre-existing) | 🔴 |
| **Patch compatibility pre-check** | 🔴 |
| Modules removed from D11 core | 🔴 |

**Outputs:** `.drupal-migration/compatibility-report.md`, `.drupal-migration/compatibility-summary.json`

---

### `code-fixer`

Automatically fixes deprecated APIs in custom modules and themes. One commit per module.

**Auto-fixes:**

| Pattern | Fix |
|---------|-----|
| `drupal_set_message()` | `\Drupal::messenger()->addMessage()` |
| `db_query/insert/update/delete()` | `\Drupal::database()->query()...` |
| `node_load()`, `user_load()` etc. | `Node::load()`, `User::load()` etc. |
| `@Block(`, `@Action(`, etc. annotations | `#[Block(`, `#[Action(` PHP 8 attributes |
| `{% spaceless %}` / `{% endspaceless %}` | `{% apply spaceless %}...{% endapply %}` |
| `sameas` / `divisibleby` Twig operators | `same as` / `divisible by` |
| `core_version_requirement: ^10` in `.info.yml` | `^10 \|\| ^11` |
| Missing `: void` return type on `validateExposed()` etc. | Added automatically |
| `PHPExcel_IOFactory` namespace | `PhpOffice\PhpSpreadsheet\IOFactory` |
| Known contrib module bugs (xls_views_data_export, etc.) | Auto-patched |

**PHP syntax verification** after each fix: if a file breaks, it's reverted automatically.

---

### `db-health`

Validates database compatibility and integrity.

**Phase: pre-migration**
- Checks MariaDB/MySQL version against target Drupal requirements
- Verifies native JSON support (required for D11 `updatedb`)
- Checks charset/collation (`utf8mb4` recommended)
- Detects pre-existing pending DB updates

**Phase: post-migration**
- Verifies all update hooks ran successfully
- Applies pending entity definition updates
- Detects orphaned field tables (field data for deleted field configs)

**Minimum DB versions:**

| Target | MariaDB | MySQL |
|--------|---------|-------|
| D10 | 10.3.7+ | 5.7.8+ |
| D11 | 10.6+ | 8.0+ |

**Outputs:** `environment.json > db_health`

---

### `config-doctor`

Post-migration configuration validator. Catches silent regressions that HTTP 200 status codes don't reveal.

**Checks:**
1. **Entity definition updates** — applies pending updates; failure here causes data corruption
2. **Orphaned schema entries** — modules with schema entries but missing from filesystem; auto-cleans them
3. **Views broken handlers** — views that return 200 but render empty due to missing plugin
4. **Config schema validation** — detects configs that reference removed/renamed plugins
5. **config_split health** — verifies split folders exist and splits are active/inactive as expected
6. **Missing formatters/widgets** — field display configs referencing non-existent formatter plugins

**Outputs:** `.drupal-migration/config-report.md`

---

### `test-runner`

Two-phase testing engine: captures state **before** migration, validates and compares **after**.

#### PHASE: baseline (before migration)

1. Discovers all content types, paragraph types, field mappings
2. Creates a dedicated **AgentIA** user (all test content owned by this user for easy cleanup)
3. Creates one test node per content type, with:
   - A **real GD-generated test image** (800×500px, created on disk with GD) for image fields
   - All paragraph fields filled with paragraphs of every allowed type, with all fields populated (text, images, links, dates, etc.)
4. Creates taxonomy terms for all vocabularies
5. Runs Search API indexing
6. **Navigation crawl**: discovers all Views page displays + menu links, crawls up to 50 URLs, captures HTML and Views row counts
7. Captures HTML snapshots of rendered nodes, creation forms, and edit forms
8. Tests creation and editing via Drupal API (baseline pass/fail recorded)

#### PHASE: validate (after migration)

1. Re-renders the **same nodes** (same nids — not recreated)
2. Re-crawls the **same URLs** (exact same list from baseline)
3. Diffs HTML: baseline vs validate
4. Compares Views row counts: if a View had 42 rows before and 0 after → **VIEWS_EMPTY** regression
5. Re-runs creation/edit API tests and compares with baseline:
   - `pre=OK, post=OK` → ✅
   - `pre=OK, post=ERROR` → 🔴 REGRESSION
6. Detects PHP errors embedded in rendered HTML

**Regression classifications:**

| Code | Meaning |
|------|---------|
| `OK` | HTTP same, rows same, no PHP errors, diff < 50 lines |
| `HTTP_CHANGED` | 200→403 (access broke), 200→500 (crash) |
| `VIEWS_EMPTY` | View lost more than half its rows |
| `ERROR_PHP` | Fatal error / Exception visible in HTML |
| `*_BIGDIFF` | > 50 lines changed in rendered HTML |

**Cleanup command:** `/drupal-cleanup-tests` removes all AgentIA test content.

---

### `updater`

Executes the actual Drupal update.

**Steps:**
1. Capture pre-update package versions (snapshot)
2. **Drush version detection** — upgrades Drush 12→13 simultaneously with D11 core (required)
3. **Lock file repair** strategy for inconsistent states
4. Enable maintenance mode
5. Update `composer.json` constraints for target version
6. Run `composer update --with-all-dependencies`
   - Watches for silent patch failures ("Could not apply patch! Skipping")
7. Run `drush updatedb -y`
8. Run `drush config:import -y`
9. **config-doctor** (entity definitions, broken Views, config_split)
10. **db-health** post-migration check
11. **patch-manager** post-migration (verify + archive + generate patches for manual fixes)
12. Re-enable CSS/JS aggregation
13. Disable maintenance mode + cache rebuild
14. Verify site responds (bootstrap successful, version matches target)
15. **Generate `DEPLOYMENT-CHECKLIST-YYYY-MM-DD.md`** for production deployment
16. Generate preliminary migration report

**Lock file repair strategy:**
- `composer update --lock` → regenerate lock without installing
- `composer install --no-scripts` → install from existing lock
- `rm composer.lock && composer install` → nuclear option (noted in report)

---

### `rollback-manager`

Auto-invoked on any critical failure. Never runs automatically without presenting options.

**Recovery options:**

| Option | What it does |
|--------|-------------|
| `[R] ROLLBACK` | Restore DB from snapshot + `git checkout TAG -- composer.json composer.lock` + `composer install` |
| `[F] FORCE` | Skip the failed step and continue (risky, requires explicit confirmation) |
| `[M] MANUAL` | Pause migration, preserve current state, print manual recovery steps |

Always generates `.drupal-migration/MIGRATION-FAILED-TIMESTAMP.md` before presenting options.

---

## Slash Commands

| Command | Description |
|---------|-------------|
| `/drupal-migrate` | Full major version upgrade (prompts for target version) |
| `/drupal-update` | Security patches and minor updates |
| `/drupal-status` | Read-only analysis — detect current state, available updates, no changes |
| `/drupal-rollback` | Restore from last snapshot |
| `/drupal-dry-run` | **Simulate** what a migration would do — `composer update --dry-run` + compatibility report, zero changes |
| `/drupal-cleanup-tests` | Delete all AgentIA test nodes and baseline snapshots |
| `/drupal-resume` | Resume an interrupted migration from the last completed step |

---

## State Tracking & Resumability

Every pipeline step records its completion in `.drupal-migration/environment.json > migration.state`. If the skill is interrupted (crash, manual stop, connection loss), `/drupal-resume` reads the last completed step and resumes from the next one.

```json
{
  "migration": {
    "state": "preflight",
    "last_completed_step": "preflight",
    "last_completed_at": "2026-04-26T16:35:00Z",
    "target_version": 11
  }
}
```

**States:** `env-detected` → `preflight` → `patches-validated` → `compatibility` → `code-fixed` → `baseline` → `updating` → `testing` → `complete`

---

## Known Deprecations Database

`config/known-deprecations.json` contains a curated database of deprecated patterns and known bugs, updated from real-world migrations.

### D9 → D10
- 14 deprecated PHP functions (`drupal_set_message`, `db_query`, `node_load`, etc.)
- 2 removed hooks
- Twig `{{ dump() }}` warning

### D10 → D11
- PHP return type enforcement (`:void` on `validateExposed`, `submitForm`, `validateForm`)
- **Symfony 7 breaking changes**: `GetResponseEvent`, `ContainerAwareCommand`, `Symfony\Component\EventDispatcher\Event` namespace changes
- **PHPExcel → PhpSpreadsheet** library migration (namespace mapping table)
- **Known contrib module bugs** discovered from production migrations:
  - `drupal/xls_views_data_export` 3.x and 4.x: `buildResponse()` signature incompatibility
  - `drupal/token_views_filter` 2.x: missing `:void` return type
- **Removed from D11 core**: tour, ckeditor, aggregator, color, rdf, hal, quickedit, classy, stable, sdc
- Drush minimum version: 13
- Annotations → PHP 8 Attributes (8 plugin types, with correct `use` statements)
- Twig: `{% spaceless %}`, `sameas`, `divisibleby`
- Scaffold file mapping format: `false` value (not object with `overwrite: false`)

### D11 → D12 (placeholder)
- All annotations become mandatory PHP Attributes (were deprecated in D11)

---

## Requirements

### Claude Code
- [Claude Code CLI](https://claude.ai/code)
- Any Claude model (Sonnet 4.6+ recommended)

### Local environment
- One of: DDEV, Lando, Docker Compose, or local PHP
- PHP 8.1+ (8.3+ recommended for D11 target)
- Composer 2.x
- Git
- `curl` and `diff` available in the container/shell
- 500MB+ free disk space

### Drupal project
- Composer-based Drupal installation
- Drush installed as a Composer dependency (`drush/drush`)
- Git repository initialized

---

## Installation

### Via Claude Code plugin system

Add to your Claude Code `settings.json`:
```json
{
  "skills": [
    "drupal-migration"
  ]
}
```

Or copy the `skills/drupal-migration/` directory to your Claude Code skills folder:
```bash
cp -r drupal-migration ~/.claude/skills/
```

### Manual installation
```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/drupal-migration-skill.git

# Copy to Claude Code skills directory
cp -r drupal-migration-skill ~/.claude/skills/drupal-migration
```

---

## Usage

### Check current status (safe, no changes)
```
/drupal-status
```

### Preview what would happen (dry-run)
```
/drupal-dry-run
```

### Apply security updates
```
/drupal-update
```

### Migrate to Drupal 11
```
/drupal-migrate
> Target version? 11
```

### Resume an interrupted migration
```
/drupal-resume
```

### Clean up test content after validation
```
/drupal-cleanup-tests
```

---

## What gets created

After a successful migration, the project contains:

```
.drupal-migration/
├── environment.json          # Full environment snapshot + migration state
├── compatibility-report.md   # Pre-migration analysis
├── compatibility-summary.json
├── test-baseline.json        # Test content inventory
├── patch-report.md           # Patch validation results
├── config-report.md          # Post-migration config health
├── comparison-report.md      # Before/after diff summary
├── baseline/
│   ├── rendered/             # HTML of rendered nodes (pre-migration)
│   ├── create-forms/         # HTML of creation forms (pre-migration)
│   ├── edit-forms/           # HTML of edit forms (pre-migration)
│   └── navigation/           # HTML of Views + menu pages (pre-migration)
└── validate/
    ├── rendered/             # HTML of rendered nodes (post-migration)
    ├── navigation/           # HTML of Views + menu pages (post-migration)
    └── diff-*.diff           # Line-by-line diffs

backups/
└── pre-migration-YYYY-MM-DD/
    ├── db.sql.gz             # Full database dump
    └── MIGRATION-NOTES.md

MIGRATION-REPORT-YYYY-MM-DD.md   # Full migration report
DEPLOYMENT-CHECKLIST-YYYY-MM-DD.md  # Production deployment guide
```

---

## Architecture

```
SKILL.md (orchestrator)
├── agents/
│   ├── env-detector.md        (step 1 — read-only)
│   ├── pre-flight.md          (step 2 — backup + branch)
│   ├── patch-manager.md       (step 3 + post — patch lifecycle)
│   ├── compatibility-analyzer.md  (step 4 — analysis)
│   ├── code-fixer.md          (step 5 — auto-fix)
│   ├── test-runner.md         (step 6 baseline + step 8 validate)
│   ├── updater.md             (step 7 — composer + drush)
│   ├── config-doctor.md       (sub-agent of updater)
│   ├── db-health.md           (sub-agent of env-detector + updater)
│   └── rollback-manager.md    (on critical failure)
└── config/
    └── known-deprecations.json  (D9→D10, D10→D11, D11→D12)
```

### How agents communicate

Each agent:
1. Reads `~/.claude/skills/drupal-migration/agents/AGENT-NAME.md`
2. Reads `.drupal-migration/environment.json` from the Drupal project
3. Executes its steps
4. Writes its output (JSON or Markdown) to `.drupal-migration/`
5. Updates `environment.json > migration.state`

Agents do NOT call each other directly — the orchestrator (`SKILL.md`) sequences them and passes context.

---

## Safety guarantees

1. **Production protection** — refuses to run if `$settings['environment'] = 'production'`, `@prod` alias, or `APP_ENV=production` is detected
2. **Snapshot before everything** — DB dump + config export + git tag created before first modification
3. **CSS/JS aggregation disabled** during migration to surface errors that would be masked by cached assets
4. **Automatic rollback** on critical failure — no silent broken states
5. **Patch validation before update** — knows a patch will fail BEFORE running `composer update`
6. **Test content isolated** — all test nodes owned by `AgentIA` user, deletable with one command

---

## Supported migration paths

| Migration | Status |
|-----------|--------|
| D9 → D10 | ✅ Supported |
| D10 → D11 | ✅ Supported (battle-tested) |
| D11 → D12 | 🔄 Ready (deprecations DB placeholder) |
| D9 → D11 (multi-hop) | ✅ Automatic two-pass |
| D8 → D9 | ⚠️ Untested (D9→D10 patterns apply) |

---

## Contributing

### Adding a new deprecation pattern

Edit `config/known-deprecations.json` and add to the appropriate version key:

```json
"D10_to_D11": {
  "php_functions": [
    {
      "deprecated": "old_function\\(",
      "replacement": "\\Drupal::service('new.service')->method(",
      "type": "regex",
      "severity": "error"
    }
  ]
}
```

### Adding a known contrib module bug

```json
"known_module_bugs": {
  "bugs": [
    {
      "module": "drupal/module_name",
      "affected_versions": ["2.x"],
      "description": "What breaks and why",
      "file": "src/path/to/File.php",
      "fix_type": "signature|return_type|namespace",
      "fix": {
        "old": "broken code pattern",
        "new": "corrected code pattern"
      },
      "auto_fixable": true
    }
  ]
}
```

---

## License

MIT — see [LICENSE](LICENSE)

---

## Credits

Built and battle-tested during a real D10.5.3 → D11.3.8 migration in April 2026.

Issues and edge cases discovered during that migration are documented in `known-deprecations.json` and the agent files, so future users don't encounter the same silent failures.
