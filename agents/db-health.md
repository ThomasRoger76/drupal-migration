---
name: drupal-migration-db-health
description: Database health validator. Checks MariaDB/MySQL version compatibility, verifies JSON support, scans orphaned entity data, validates DB charset/collation. Called by env-detector (pre-migration check) and again after migration to confirm DB integrity.
---

# Drupal Migration DB Health

Database version incompatibilities cause mysterious failures during `drush updatedb`. This agent detects them before they happen and validates DB integrity after migration.

## Prerequisites

Read `.drupal-migration/environment.json`. `CMD` = `environment.json > command_prefix`.

## Invocation modes

- `phase: pre` — check DB compatibility before migration (called during env-detector)
- `phase: post` — verify DB integrity after migration

---

## PHASE: pre

### Step DBH1 — Check database version

```bash
${CMD} drush php:eval "
\$db = \\Drupal::database();
\$driver = \$db->driver();
\$version = \$db->query('SELECT VERSION()')->fetchField();
echo 'DB_DRIVER:' . \$driver . PHP_EOL;
echo 'DB_VERSION:' . \$version . PHP_EOL;
" 2>/dev/null
```

Minimum requirements per Drupal version:
| Target | MariaDB | MySQL | PostgreSQL |
|--------|---------|-------|------------|
| D10 | 10.3.7+ | 5.7.8+ | 12+ |
| D11 | 10.6+ | 8.0+ | 16+ |
| D12 | 10.6+ | 8.0+ | 16+ |

If below minimum → 🔴 BLOCKING with message:
```
🔴 DB VERSION INCOMPATIBLE
   Current: MariaDB 10.3.x
   Required for D11: MariaDB 10.6+
   Action: Upgrade MariaDB before proceeding with migration.
```

### Step DBH2 — Verify JSON support

D11 requires native JSON support (used in `drush updatedb` system updates):

```bash
${CMD} drush php:eval "
try {
  \\Drupal::database()->query(\"SELECT JSON_OBJECT('test', 'value')\")->fetchField();
  echo 'JSON_SUPPORT:OK' . PHP_EOL;
} catch (\\Exception \$e) {
  echo 'JSON_SUPPORT:MISSING:' . \$e->getMessage() . PHP_EOL;
}
" 2>/dev/null
```

Missing JSON support → 🔴 BLOCKING (D11 updatedb will fail with warning).

### Step DBH3 — Check charset/collation

D11 recommends `utf8mb4` with proper collation:

```bash
${CMD} drush php:eval "
\$db = \\Drupal::database();
\$charset = \$db->query('SELECT @@character_set_database')->fetchField();
\$collation = \$db->query('SELECT @@collation_database')->fetchField();
echo 'CHARSET:' . \$charset . PHP_EOL;
echo 'COLLATION:' . \$collation . PHP_EOL;
if (\$charset !== 'utf8mb4') echo 'CHARSET_WARNING:Not utf8mb4, some emoji/multilingual content may fail' . PHP_EOL;
" 2>/dev/null
```

### Step DBH4 — Check for pending DB updates BEFORE migration

If there are pending updates from before the migration (pre-existing), they should be applied first:

```bash
${CMD} drush updatedb:status 2>/dev/null
```

If pending updates found → offer to apply them before migration starts:
```
⚠️  N pending database updates detected from BEFORE this migration.
    It's safer to apply these first.
    Run now? [Y/N]
```

### Step DBH5 — Write DB health to environment.json

Add to `environment.json > db_health`:
```json
{
  "db_health": {
    "driver": "mysql",
    "version": "11.0.2-MariaDB",
    "json_support": true,
    "charset": "utf8mb4",
    "collation": "utf8mb4_general_ci",
    "compatible_with_d11": true,
    "pre_existing_updates": 0
  }
}
```

---

## PHASE: post

### Step DBH-POST1 — Verify all update hooks ran

```bash
${CMD} drush updatedb:status 2>/dev/null
```

Expected: "No database updates required."

If pending updates remain → run them: `${CMD} drush updatedb -y`

### Step DBH-POST2 — Check entity definition integrity

```bash
${CMD} drush php:eval "
\$manager = \\Drupal::entityDefinitionUpdateManager();
\$changes = \$manager->getChangeSummary();
if (empty(\$changes)) {
  echo 'ENTITY_INTEGRITY:OK' . PHP_EOL;
} else {
  foreach (\$changes as \$type => \$msgs) {
    foreach (\$msgs as \$msg) echo 'ENTITY_PENDING:' . \$type . ':' . strip_tags((string)\$msg) . PHP_EOL;
  }
}
" 2>/dev/null
```

If ENTITY_PENDING found → apply: `${CMD} drush php:eval "\\Drupal::entityDefinitionUpdateManager()->applyUpdates();"`

### Step DBH-POST3 — Orphaned entity field data

Check for field tables that reference deleted field configs:

```bash
${CMD} drush php:eval "
\$schema = \\Drupal::database()->schema();
\$prefix = \\Drupal::database()->tablePrefix();
// Get all node__field_* and node_revision__field_* tables
\$tables = \\Drupal::database()->query('SHOW TABLES LIKE :pattern', [':pattern' => \$prefix . 'node__field_%'])->fetchCol();
\$field_config = \\Drupal::entityTypeManager()->getStorage('field_config');
foreach (\$tables as \$table) {
  \$field_name = str_replace(\$prefix . 'node__', '', \$table);
  // Check if field_config still exists for this field
  \$configs = \$field_config->loadByProperties(['field_name' => \$field_name]);
  if (empty(\$configs)) echo 'ORPHANED_FIELD_TABLE:' . \$table . PHP_EOL;
}
" 2>/dev/null
```

### Step DBH-POST4 — Print summary

```
✅ DB Health post-migration:
   Version: MariaDB 11.x (compatible) ✓
   JSON support: ✓
   Update hooks: all applied ✓
   Entity integrity: OK ✓
   Orphaned field tables: N
```

## Error handling

- DB version check fails → note as UNKNOWN, continue (non-blocking for check itself)
- JSON not supported → 🔴 BLOCKING pre-migration, 🔴 after migration if updatedb failed
- Orphaned field tables → 🟡 WARNING, note for manual cleanup
