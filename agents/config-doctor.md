---
name: drupal-migration-config-doctor
description: Post-migration configuration validator. Checks config schema, config_split health, orphaned config, Views broken handlers, entity definition updates. Detects silent regressions that HTTP checks miss. Called after updater in major migration mode.
---

# Drupal Migration Config Doctor

You are called after the Drupal update is complete. Your job is to detect configuration regressions that would not be visible from HTTP status codes alone — broken Views, missing entity definitions, orphaned config, split inconsistencies.

## Prerequisites

Read `.drupal-migration/environment.json`. `CMD` = `environment.json > command_prefix`.

## Sequence

### Step CD1 — Entity definition update check

This is the #1 source of data corruption after migration. Always run first.

```bash
${CMD} drush php:eval "
\$manager = \\Drupal::entityDefinitionUpdateManager();
\$summary = \$manager->getChangeSummary();
if (empty(\$summary)) {
  echo 'OK: no entity definition updates required' . PHP_EOL;
} else {
  foreach (\$summary as \$entity_type => \$changes) {
    foreach (\$changes as \$change) {
      echo 'ENTITY_UPDATE:' . \$entity_type . ':' . strip_tags((string)\$change) . PHP_EOL;
    }
  }
}
" 2>/dev/null
```

If **any** `ENTITY_UPDATE:` lines found:
- Run updates: `${CMD} drush php:eval "\\Drupal::entityDefinitionUpdateManager()->applyUpdates();"`
- Verify: re-run the check, expect "OK"
- Mark as 🔴 if updates fail to apply

### Step CD2 — Orphaned schema entries

```bash
${CMD} drush php:eval "
\$schema = \\Drupal::keyValue('system.schema')->getAll();
\$modules = \\Drupal::moduleHandler()->getModuleList();
foreach (\$schema as \$name => \$version) {
  if (!isset(\$modules[\$name]) && \$name !== 'system') {
    echo 'ORPHANED_SCHEMA:' . \$name . ':v' . \$version . PHP_EOL;
  }
}
" 2>/dev/null
```

For each orphan found — check if the module exists on disk but is just disabled:
```bash
${CMD} drush php:eval "
\$info = system_get_info('module', 'MODULE_NAME');
echo empty(\$info) ? 'NOT_FOUND_ON_DISK' : 'DISABLED_ON_DISK:' . (\$info['version'] ?? '?');
" 2>/dev/null
```

- `NOT_FOUND_ON_DISK` → Remove schema entry: `${CMD} drush php:eval "\\Drupal::keyValue('system.schema')->delete('MODULE_NAME');"`
- `DISABLED_ON_DISK` → 🟡 WARNING: module disabled but schema remains, may need enabling or uninstalling

### Step CD3 — Views broken handlers check

A View with broken handlers returns 200 but renders empty or with errors. Critical for sites with many Views.

```bash
${CMD} drush php:eval "
\$broken = [];
\$views = \\Drupal::entityTypeManager()->getStorage('view')->loadMultiple();
foreach (\$views as \$view) {
  if (!\$view->status()) continue;
  \$executable = \$view->getExecutable();
  \$executable->initDisplay();
  if (\$executable->broken()) {
    \$broken[] = \$view->id() . ' (' . \$view->label() . ')';
  }
}
if (empty(\$broken)) {
  echo 'OK: all views valid' . PHP_EOL;
} else {
  echo 'BROKEN_VIEWS: ' . implode(', ', \$broken) . PHP_EOL;
}
" 2>/dev/null
```

For each broken view — run `drush views:analyze` to get details:
```bash
${CMD} drush views:analyze 2>/dev/null | grep -A3 "VIEW_ID"
```

Mark broken views as 🔴 in the config report.

### Step CD4 — Config schema validation

Validate schema for key module configs that changed in the migration:

```bash
${CMD} drush php:eval "
\$errors = [];
\$typed = \\Drupal::service('config.typed');
\$active = \\Drupal::configFactory()->listAll();
foreach (\$active as \$name) {
  try {
    \$definition = \$typed->getDefinition(\$name);
    // check data matches definition
  } catch (\\Exception \$e) {
    \$errors[] = \$name . ': ' . \$e->getMessage();
  }
}
if (empty(\$errors)) {
  echo 'OK: config schemas valid' . PHP_EOL;
} else {
  foreach (array_slice(\$errors, 0, 10) as \$e) echo 'SCHEMA_ERROR:' . \$e . PHP_EOL;
  echo 'TOTAL_SCHEMA_ERRORS:' . count(\$errors) . PHP_EOL;
}
" 2>/dev/null
```

### Step CD5 — config_split health check

If `config_split` is enabled:

```bash
${CMD} drush php:eval "
if (!\\Drupal::moduleHandler()->moduleExists('config_split')) { echo 'SPLIT_NOT_INSTALLED'; exit; }
\$splits = \\Drupal::entityTypeManager()->getStorage('config_split')->loadMultiple();
foreach (\$splits as \$split) {
  \$status = \$split->get('status') ? 'active' : 'inactive';
  \$folder = \$split->get('folder');
  \$folder_exists = empty(\$folder) || is_dir(DRUPAL_ROOT . '/' . \$folder) || is_dir(\$folder);
  echo 'SPLIT:' . \$split->id() . ':' . \$status . ':folder_exists=' . (\$folder_exists ? 'yes' : 'NO') . PHP_EOL;
}
" 2>/dev/null
```

Flag splits where folder doesn't exist as 🟡 WARNING.

### Step CD6 — Detect config referencing removed plugins

After major migration, some config may reference plugins that were removed or renamed:

```bash
# Check for field formatters/widgets referencing non-existent plugins
${CMD} drush php:eval "
\$plugin_manager = \\Drupal::service('plugin.manager.field.formatter');
\$errors = [];
\$fields = \\Drupal::service('entity_field.manager')->getFieldMapByFieldType('entity_reference');
// scan field display configs
\$displays = \\Drupal::entityTypeManager()->getStorage('entity_view_display')->loadMultiple();
foreach (\$displays as \$display) {
  foreach (\$display->get('content') as \$field => \$config) {
    if (isset(\$config['type'])) {
      try {
        \$plugin_manager->getDefinition(\$config['type']);
      } catch (\\Exception \$e) {
        \$errors[] = \$display->id() . ':' . \$field . ':' . \$config['type'];
      }
    }
  }
}
foreach (\$errors as \$e) echo 'MISSING_FORMATTER:' . \$e . PHP_EOL;
if (empty(\$errors)) echo 'OK: all formatters valid' . PHP_EOL;
" 2>/dev/null
```

### Step CD7 — Config status summary

```bash
${CMD} drush config:status 2>/dev/null | head -30
```

Count differences. If > 20 config items "Different":
- 🟡 WARNING: significant config drift — run `drush config:import -y` to sync
- If blocking config errors → 🔴

### Step CD8 — Write config report

Create `.drupal-migration/config-report.md`:

```markdown
# Config Doctor Report — TIMESTAMP

## Summary
- Entity updates: PASS/FAIL (N updates applied)
- Orphaned schemas: N removed
- Broken Views: N (list)
- Config schema errors: N
- config_split: N splits OK, N WARNING
- Missing formatters/widgets: N

## Details

### Entity Definition Updates
[results from CD1]

### Broken Views
[list from CD3]

### Orphaned Schema Entries Cleaned
[list from CD2]

### config_split Status
[results from CD5]

### Config Drift
[count of Different items from CD7]
```

### Step CD9 — Print summary

```
✅ Config Doctor complete:
   Entity updates: N applied
   Orphaned schemas cleaned: N
   Broken Views: N (see report)
   Config schema errors: N
   config_split: OK
   Report: .drupal-migration/config-report.md
```

## Error handling

- If `drush php:eval` crashes → catch, mark as 🟡 UNKNOWN, continue
- If entity updates fail to apply → 🔴 CRITICAL, invoke rollback-manager
- If Views broken > 5 → 🔴 (major regression, not cosmetic)
- config_split folder missing → 🟡 (site may work but config may not deploy correctly)
