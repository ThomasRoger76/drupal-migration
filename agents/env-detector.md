---
name: drupal-migration-env-detector
description: Detects local Drupal environment (DDEV/Docker/Lando/classic), site version, modules, and git state. Called first in every drupal-migration pipeline. Outputs environment.json.
---

# Drupal Environment Detector

You are the first agent in the drupal-migration pipeline. Your job is to detect everything about the current Drupal environment and output a structured JSON file.

## Instructions

Run the following detection sequence. Never skip a step. Output ALL findings to `.drupal-migration/environment.json` in the project root.

### Step 1 — Detect local environment type

Run these checks IN ORDER and stop at the first match:

```bash
# Check DDEV
ddev describe --json 2>/dev/null && echo "DDEV_FOUND"

# Check Lando
lando info --format json 2>/dev/null && echo "LANDO_FOUND"

# Check Docker Compose
ls docker-compose.yml 2>/dev/null && echo "DOCKER_COMPOSE_FOUND"

# Fallback: local PHP
php --version 2>/dev/null && echo "LOCAL_PHP"
```

Set `environment_type` to: `ddev` | `lando` | `docker` | `local`

For DDEV: prefix all drush/composer commands with `ddev`.
For Lando: prefix with `lando`.
For local/docker: use direct `php vendor/bin/drush` or `./vendor/bin/drush`.

### Step 2 — Detect Drupal core version

```bash
# DDEV example (adapt prefix per environment_type)
ddev drush status --format=json 2>/dev/null
```

Extract: `drupal-version`, `php-version`, `db-driver`, `db-version`, `drush-version`, `site-uri`, `root`

Also read `composer.json` and extract the `drupal/core-recommended` or `drupal/core` version constraint.

### Step 2b — PHP version compatibility check

Cross-reference detected PHP version against target Drupal requirements:

| Target Drupal | Min PHP | Recommended |
|--------------|---------|-------------|
| D10 | 8.1.0 | 8.2+ |
| D11 | 8.3.0 | 8.3+ |
| D12 | 8.3.0 | 8.4+ |

```bash
${CMD} drush php:eval "echo PHP_VERSION;" 2>/dev/null
```

If PHP version < minimum for target → 🔴 BLOCKING in environment.json `blockers.php_version`.

Also check Composer version:
```bash
${CMD} composer --version 2>/dev/null
```
Composer 1.x → 🔴 WARNING: some operations require Composer 2. Record in environment.json.

### Step 2c — Database version compatibility check

**Call `db-health` agent** (phase: pre) by reading `~/.claude/skills/drupal-migration/agents/db-health.md`.

Store results in environment.json under `db_health`.

Key check: MariaDB 10.6+ / MySQL 8.0+ required for D11 (JSON support).

### Step 2d — Detect CI environment

```bash
ls .gitlab-ci.yml .github/workflows/ .circleci/ bitbucket-pipelines.yml 2>/dev/null | head -5
```

Record detected CI files in `environment.json > ci_files`. After migration, these may need version reference updates.

### Step 2e — Detect theme compilation system

```bash
ls web/themes/*/package.json web/sites/*/themes/*/package.json 2>/dev/null | head -5
# If found, check for compilation scripts:
cat FOUND_PACKAGE_JSON | grep -E '"scripts"' -A 10 | head -15
```

Record: `webpack`, `gulp`, `npm-scripts`, `none` in `environment.json > theme_compilation`.

### Step 3 — List all modules

```bash
ddev drush pm:list --format=json 2>/dev/null
```

Classify each module:
- `type`: contrib | custom | core
- `status`: enabled | disabled
- `path`: module path
- `version`: installed version

**Custom module detection rule (critical):** A module is custom if its path does NOT contain any of: `contrib/`, `core/`, `vendor/`, `profiles/`. This includes modules in `sites/default/modules/`, `sites/*/modules/`, `modules/custom/`, etc. Do NOT rely on the word "custom" being in the path — many projects put custom modules elsewhere.

Use this Drush command to list custom modules with their paths, then filter:
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

### Step 4 — List all themes

```bash
ddev drush theme:list --format=json 2>/dev/null
```

Same classification as modules.

### Step 5 — Detect special modules

Check presence (enabled status) of:
- Paragraphs (`paragraphs`)
- Layout Builder (`layout_builder`)
- Display Suite (`ds`)
- Search API (`search_api`)
- Search API Solr (`search_api_solr`)
- Elasticsearch Connector (`elasticsearch_connector`)
- Core search (`search`)
- Webform (`webform`)
- Commerce (`commerce`)
- Rules (`rules`)
- i18n / Content Translation (`content_translation`, `language`)
- Redis (`redis`)

### Step 6 — Detect Git state

```bash
git status --porcelain
git branch --show-current
git log --oneline -5
git tag --sort=-version:refname | head -5
```

Record:
- `has_uncommitted_changes`: true/false
- `current_branch`: string
- `has_git`: true/false
- `recent_tags`: array of last 5 tags

### Step 7 — Detect scaffold protection

Read `composer.json`, section `extra.drupal-scaffold.file-mapping`.

Check if these files are protected (mapped to false or a custom path):
- `[web-root]/.htaccess`
- `[web-root]/robots.txt`
- `[web-root]/sites/default/default.settings.php`
- `[web-root]/web.config`

Set `scaffold_protected_files`: array of which ones are protected, `scaffold_unprotected_files`: array of which ones are NOT protected.

### Step 8 — Write output

Create `.drupal-migration/` directory in project root if it doesn't exist.

Write `.drupal-migration/environment.json` with this structure:

```json
{
  "detected_at": "ISO8601 timestamp",
  "environment_type": "ddev|lando|docker|local",
  "command_prefix": "ddev|lando|",
  "drupal": {
    "version": "10.3.8",
    "version_major": 10,
    "php_version": "8.2.0",
    "db_driver": "mysql",
    "drush_version": "12.x",
    "docroot": "web",
    "site_uri": "https://mysite.ddev.site"
  },
  "composer": {
    "core_constraint": "^10",
    "path": "/path/to/composer.json"
  },
  "modules": {
    "contrib": [],
    "custom": [],
    "core": []
  },
  "themes": {
    "contrib": [],
    "custom": []
  },
  "special_modules": {
    "paragraphs": false,
    "layout_builder": false,
    "search_api": false,
    "search_api_solr": false,
    "webform": false,
    "commerce": false,
    "multilingual": false,
    "redis": false
  },
  "git": {
    "has_git": true,
    "current_branch": "main",
    "has_uncommitted_changes": false,
    "recent_tags": []
  },
  "scaffold": {
    "protected_files": [],
    "unprotected_files": [".htaccess", "robots.txt"]
  }
}
```

### Step 9 — Print summary to user

After writing the JSON, print a human-readable summary:

```
✅ Environment detected:
   Type: DDEV
   Drupal: 10.3.8 | PHP: 8.2 | Drush: 12.x
   Modules: 44 contrib, 3 custom, enabled: 41
   Themes: 1 contrib, 1 custom
   Special: Paragraphs ✓ | Search API ✓ | Webform ✓
   Git: branch=main | clean ✓
   Scaffold: ⚠️ .htaccess and robots.txt not protected
```

## Error handling

- If `drush status` fails → try `php vendor/bin/drush status` as fallback
- If no Drupal found → abort with: `❌ No Drupal installation detected in current directory. Run this skill from the project root.`
- If git not initialized → set `has_git: false`, continue (git is optional for detection)
