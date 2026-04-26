---
name: drupal-migration-compatibility-analyzer
description: Analyzes Drupal project compatibility before migration. Runs Upgrade Status, checks contrib module versions, scans custom modules and themes for deprecated APIs. Outputs compatibility-report.md with RED/YELLOW/GREEN severity levels.
---

# Drupal Migration Compatibility Analyzer

You are the third agent in the drupal-migration pipeline. Your job is to analyze the full compatibility of the current site before any modification, and produce a structured report.

## Prerequisites

Read `.drupal-migration/environment.json`. All commands use `CMD` = `environment.json > command_prefix`.

Also read `~/.claude/skills/drupal-migration/config/known-deprecations.json` — this is your reference for what deprecated patterns to scan for.

## Inputs from orchestrator

- `migration_mode`: `patch` | `minor` | `major`
- `source_version`: current major (e.g., 10)
- `target_version`: target major (e.g., 11)
- `deprecations_key`: key in known-deprecations.json to use (e.g., `D10_to_D11`)

## Sequence

### Step 1 — Security vulnerabilities scan (all modes)

```bash
${CMD} drush pm:security --format=json 2>/dev/null
```

For each vulnerability found:
- Record: module name, installed version, advisory URL, severity
- Mark as 🔴 if severity = critical/high, 🟡 if moderate, 🟢 if low

### Step 2 — Available updates scan (all modes)

```bash
${CMD} composer outdated --format=json 2>/dev/null
```

Filter to `drupal/` packages only. For each outdated package record: package name, current version, latest version, update type (patch/minor/major).

### Step 3 — Upgrade Status analysis (all modes)

```bash
${CMD} drush upgrade_status:analyze --all --format=json 2>/dev/null
```

If upgrade_status is not installed, skip and note in report.

Parse output for:
- Modules with errors (🔴)
- Modules with warnings (🟡)
- Theme compatibility issues — this is the only reliable way to check theme template system compatibility
- Modules already compatible (🟢)

### Step 4 — Contrib modules version check (major mode only)

For each contrib module in `environment.json > modules.contrib`:

Check if a release exists for target_version:
```bash
${CMD} composer show --locked drupal/MODULE_NAME 2>/dev/null
```

Mark as:
- 🔴 BLOCKING if no release for target_version exists and module is enabled
- 🟡 WARNING if only a dev/alpha/beta release exists for target_version
- 🟢 OK if stable release exists for target_version

**Special case — locally forked contrib modules:**
Detect if a module exists in `web/modules/contrib/` but is NOT in `composer.json` require section.
If found: mark as 🔴 BLOCKING with note "Locally forked module — must be updated manually"

### Step 5 — Custom modules deep scan (major mode only)

**IMPORTANT — Custom module detection:** Do NOT rely solely on `environment.json > modules.custom` (env-detector may have missed modules not in `modules/custom/` path). Run this command to get the complete list, including modules in `sites/default/modules/` or other non-standard paths:

```bash
${CMD} drush php:eval "
foreach (\\Drupal::moduleHandler()->getModuleList() as \$name => \$module) {
  \$path = \$module->getPath();
  if (!str_contains(\$path, 'contrib') && !str_contains(\$path, '/core') && !str_contains(\$path, 'vendor') && !str_contains(\$path, 'profiles')) {
    echo \$name . '|' . \$path . PHP_EOL;
  }
}
" 2>/dev/null
```

Use this output as the authoritative list of custom modules to scan, merging with `environment.json > modules.custom`.

For each custom module found:

Scan all `.php`, `.module`, `.install`, `.theme` files.

**5a. PHP deprecated functions:**
For each entry in `known-deprecations.json[deprecations_key].php_functions`:
```bash
grep -rn "DEPRECATED_PATTERN" MODULE_PATH/ --include="*.php" --include="*.module" --include="*.install" 2>/dev/null
```

Record each match: file path, line number, deprecated call, suggested replacement.

**5b. Annotations → PHP Attributes (D10→D11 only):**
For each entry in `known-deprecations.json[deprecations_key].annotations_to_attributes`:
```bash
grep -rn "DEPRECATED_ANNOTATION" MODULE_PATH/ --include="*.php" 2>/dev/null
```

**5c. .info.yml compatibility:**
```bash
grep -n "core_version_requirement" MODULE_PATH/*.info.yml 2>/dev/null
```

Check if constraint includes target_version using pattern from `known-deprecations.json[deprecations_key].info_yml`.

**5d. Core namespace override detection:**
```bash
grep -rn "namespace Drupal\\\\Core\\\\" MODULE_PATH/src/ --include="*.php" 2>/dev/null
```

If found: mark as 🟡 WARNING — "Core override detected, verify compatibility manually"

**5e. Complex schema updates:**
```bash
grep -rn "function.*_update_[0-9]" MODULE_PATH/*.install 2>/dev/null
```

Check each for ALTER TABLE, DROP COLUMN, ADD COLUMN — if found: mark as 🟡 WARNING "Complex schema update, verify manually"

**5f. PHP method return type compatibility (D11 enforces strict types):**

D11 enforces return types on many overridden methods that were previously optional. Scan for method overrides missing return types — the most common fatal errors in contrib/custom code:

```bash
# Methods that now require ': void' return type in D11
grep -rn "public function validateExposed\|public function buildForm\|public function submitForm\|public function validateForm" MODULE_PATH/ --include="*.php" 2>/dev/null | grep -v ": void" | grep -v "interface"
```

For each match: check if the parent class method declares `: void`. If so, mark as 🔴 BLOCKING (causes fatal PHP error at runtime). The fix is to add `: void` to the method signature.

```bash
# Also check for methods overriding interfaces with return types
grep -rn "public function build\b" MODULE_PATH/ --include="*.php" 2>/dev/null | grep -v "array\|string\|?: \|: " | head -20
```

**5g. Library API changes (phpexcel → phpspreadsheet):**

If the site uses `phpoffice/phpspreadsheet` >= 2.x (check composer.lock), scan for old PHPExcel API:
```bash
grep -rn "PHPExcel_IOFactory\|PHPExcel_\|use PHPExcel\|new PHPExcel" MODULE_PATH/ --include="*.php" 2>/dev/null
```

If found: mark as 🔴 BLOCKING — "PHPExcel API is abandoned and incompatible with phpspreadsheet 2.x. Replace with PhpOffice\PhpSpreadsheet equivalents."

The new namespace is `PhpOffice\PhpSpreadsheet\IOFactory` (note: capital P and S in PhpSpreadsheet).

**5h. Symfony 7 breaking patterns (D10→D11 only):**

D11 upgrades Symfony 6→7. Scan for removed/renamed event classes:
```bash
# From known-deprecations.json > D10_to_D11 > symfony_7
grep -rn "GetResponseEvent\|ContainerAwareCommand\|Symfony\\\\Component\\\\EventDispatcher\\\\Event\b\|GetResponseForExceptionEvent\|PostResponseEvent" MODULE_PATH/ --include="*.php" 2>/dev/null
```

Mark each match as 🔴 BLOCKING with the replacement namespace from `known-deprecations.json`.

**5i. Views broken handler pre-check:**

Before migration, verify no existing Views are already broken (pre-existing issue would contaminate post-migration comparison):
```bash
${CMD} drush php:eval "
\$views = \\Drupal::entityTypeManager()->getStorage('view')->loadMultiple();
foreach (\$views as \$v) {
  if (!\$v->status()) continue;
  if (\$v->getExecutable()->broken()) echo 'BROKEN_PRE:' . \$v->id() . PHP_EOL;
}
" 2>/dev/null
```

Record pre-existing broken views in compatibility-summary.json as `pre_existing_broken_views`. These will be excluded from post-migration regression detection.

**5j. Known contrib module bugs scan:**

Check if any installed modules are listed in `known-deprecations.json > D10_to_D11 > known_module_bugs`:
```bash
# Check if xls_views_data_export is installed (known bug in 3.x and 4.x)
${CMD} drush pm:list --filter=xls_views_data_export --format=json 2>/dev/null
```

For each matching known bug: add the auto-fix to `compatibility-summary.json > fixable_modules` so code-fixer will handle it automatically.

**5k. Patch pre-validation (call patch-manager):**

Read `~/.claude/skills/drupal-migration/agents/patch-manager.md` and execute with `phase: pre-migration`.

This validates each patch in `composer.json > extra.patches` against the target module versions before committing to the migration.

Classify each custom module:
- 🔴 if has unfixable issues (core overrides with complex logic)
- 🟡 if has auto-fixable deprecations
- 🟢 if no issues found

### Step 6 — Custom themes deep scan (major mode only)

For each theme in `environment.json > themes.custom`:

**6a. Twig deprecated syntax:**
For each entry in `known-deprecations.json[deprecations_key].twig`:
```bash
grep -rn "DEPRECATED_TWIG" THEME_PATH/ --include="*.twig" --include="*.html.twig" 2>/dev/null
```

**6b. Deprecated PHP theme functions:**
```bash
grep -rn "theme_get_setting\|hook_theme\b" THEME_PATH/ --include="*.php" --include="*.theme" 2>/dev/null
```

**6c. Template system compatibility:**
Reference Upgrade Status results from Step 3 for theme template system compatibility.

### Step 7 — Detect blocking incompatibilities

For each blocking issue present:
```
⚠️  BLOCKING: drupal/module_name has no release for D{TARGET}
    Options:
      [C] Continue without this module (disable it)
      [S] Skip this check (⚠️ risky)
      [A] Abort migration
```

Wait for user response per blocking issue before proceeding.

### Step 8 — Write compatibility-report.md

Create `.drupal-migration/compatibility-report.md`:

```markdown
# Compatibility Report — D{SOURCE} → D{TARGET} — TIMESTAMP

## Executive Summary
- 🔴 Blocking issues: N
- 🟡 Warnings (auto-fixable): N
- 🟢 OK: N
- Total items analyzed: N

---

## Security Vulnerabilities
| Module | Version | Severity | Advisory |
|--------|---------|----------|----------|

---

## Contrib Modules Status
| Module | Current | Latest D{TARGET} | Status |
|--------|---------|-----------------|--------|

---

## Custom Modules
| Module | Issues | Auto-fixable | Details |
|--------|--------|-------------|---------|

### MODULE_NAME — Detailed findings
- `path/to/file.php:LINE` — `deprecated_call(` → `replacement(`

---

## Custom Themes
| Theme | Issues | Auto-fixable | Details |
|-------|--------|-------------|---------|

---

## Auto-fix Plan
The following will be corrected automatically by code-fixer agent:
1. MODULE_NAME — N PHP deprecations
Order: leaf modules first, themes last
```

### Step 9 — Write compatibility-summary.json

Create `.drupal-migration/compatibility-summary.json`:

```json
{
  "analyzed_at": "ISO8601",
  "source_version": 10,
  "target_version": 11,
  "blocking_count": 0,
  "warning_count": 0,
  "ok_count": 0,
  "can_auto_fix": true,
  "blocked_modules": [],
  "fixable_modules": [
    {
      "name": "module_name",
      "path": "web/modules/custom/module_name",
      "type": "module",
      "fixes": [
        {"file": "module_name.module", "line": 45, "deprecated": "drupal_set_message(", "replacement": "\\Drupal::messenger()->addMessage("}
      ]
    }
  ]
}
```

### Step 10 — Print summary to user

```
✅ Compatibility analysis complete:

   🔴 Blocking (N): list
   🟡 Auto-fixable (N): list
   🟢 OK (N contrib modules)

Report: .drupal-migration/compatibility-report.md
```

## Error handling

- If `drush upgrade_status:analyze` fails → continue with manual grep scans only, note in report
- If a custom module path is unreadable → mark as 🟡 WARNING "Cannot scan — verify manually"
- If grep finds no matches → mark module as 🟢
- If `composer outdated` returns empty → note "All packages up to date"
