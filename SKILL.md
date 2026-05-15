---
name: drupal-migration
description: Use when upgrading Drupal core versions (D8→D9→D10→D11), fixing deprecated APIs with Rector, planning data migrations with the Migrate API (source/process/destination plugins, migrate_plus, migrate_tools), migrating from Drupal 7/8, importing CSV/XML data, or running drush migrate commands in Drupal 8-11+
---

# Drupal Migration — Référence Complète

## Overview

Ce skill couvre **deux sujets distincts** qu'il ne faut pas confondre :

1. **Version Upgrade** — Mettre à jour Drupal d'une version majeure à une autre (D8→D9, D9→D10, D10→D11). Il s'agit de faire évoluer le *code et les dépendances* du projet.
2. **Migrate API** — Importer des *données* depuis une source externe (D7, CSV, XML, JSON) vers Drupal via le pipeline Source → Process → Destination.

Les deux sujets partagent le terme "migration" dans l'écosystème Drupal — ce skill les traite séparément et clairement.

---

## Quick Decision Table

| Situation | Sujet | Référence |
|-----------|-------|-----------|
| Passer de D8.9 à D9 ou D10 | Version Upgrade | [version-upgrade.md](version-upgrade.md) |
| Passer de D10 à D11 | Version Upgrade | [version-upgrade.md](version-upgrade.md) |
| Modules contrib cassés après upgrade | Version Upgrade | [version-upgrade.md](version-upgrade.md) |
| `@deprecated` dans le code custom | Code déprécié | [deprecated-code.md](deprecated-code.md) |
| Annotations `@Block` → attributes `#[Block]` | Code déprécié | [deprecated-code.md](deprecated-code.md) |
| Lancer `drupal-check` ou Rector | Code déprécié | [deprecated-code.md](deprecated-code.md) |
| Importer des nœuds depuis un CSV | Migrate API | [migrate-api.md](migrate-api.md) |
| Migrer depuis Drupal 7 | Migrate API | [migrate-api.md](migrate-api.md) |
| `drush migrate:import` ne fonctionne pas | Migrate API | [migrate-api.md](migrate-api.md) |
| Définir un plugin source/process/destination | Migrate API | [migrate-api.md](migrate-api.md) |
| CKEditor 4 → 5 (D9→D10) | Version Upgrade | [version-upgrade.md](version-upgrade.md) |
| PHP 8.0 → 8.3 requis (D11) | Version Upgrade | [version-upgrade.md](version-upgrade.md) |
| Créer un plugin source custom (API, CSV parsé) | `SourcePluginBase` + `#[MigrateSource]` | [custom-plugins.md](custom-plugins.md) |
| Créer un plugin process custom (transformation) | `ProcessPluginBase` + `#[MigrateProcess]` | [custom-plugins.md](custom-plugins.md) |
| Migration incrémentale (seulement les nouvelles lignes) | `high_water_property` | [advanced-patterns.md](advanced-patterns.md) |
| Monitorer une migration en production | `MigrateEvents` EventSubscriber | [advanced-patterns.md](advanced-patterns.md) |
| Grouper des migrations avec dépendances | `migrate_plus.migration_group` YAML | [advanced-patterns.md](advanced-patterns.md) |
| Migrer des Paragraphs | `entity_reference_revisions:paragraph` | [advanced-patterns.md](advanced-patterns.md) |
| Migrer du contenu multilingue | `langcode` + `translations: true/false` | [multilingual-migration.md](multilingual-migration.md) |
| Ajouter une traduction à une entité existante | `destination: translations: true` | [multilingual-migration.md](multilingual-migration.md) |
| Mapper les codes de langue source | Plugin `language_map` custom | [multilingual-migration.md](multilingual-migration.md) |
| Migration D7 multilingue native | `d7_node` + `langcodes:` | [multilingual-migration.md](multilingual-migration.md) |
| Références circulaires (stubs) | `migration_lookup` + `no_stub: false` + `--update` | [advanced-patterns.md](advanced-patterns.md) |
| Migration D7 → D10 multi-entités (ordre) | Groupe taxonomies → médias → users → paragraphs → nodes | [advanced-patterns.md](advanced-patterns.md) |
| Rollback + nettoyage fichiers orphelins | `migrate:rollback` + nettoyage `file.usage` | [advanced-patterns.md](advanced-patterns.md) |

---

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Raison |
|---------------------|------------------|--------|
| Sauter D8→D10 directement | D8→D9 d'abord, puis D9→D10 | Les scripts de maj s'enchaînent par étapes, `hook_update_N` dans l'ordre |
| `composer update` sans contrainte de version | `composer require drupal/core-recommended:^10` | Évite des mises à jour de dépendances non désirées |
| Ignorer les erreurs Rector | Corriger avant de continuer | Le code déprécié devient fatal en D11 |
| Importer des migrations en production sans test | Toujours `--dry-run` et test en staging | Les rollbacks de migrations ne récupèrent pas les fichiers uploadés |
| Mélanger `hook_update_N` et logique de migration CSV | Garder les deux séparés | `hook_update_N` est pour le schéma, Migrate API pour les données |
| Lancer `drush updb` sans backup | Backup DB **avant** chaque `updb` | Irrécupérable sans backup si un update échoue |
| Activer `update` module en production en permanence | Le désactiver après la mise à jour | Surface d'attaque inutile |

---

## Évolution par Version Majeure — Tableau Récapitulatif

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| PHP minimum | 7.0 | 7.3 | 8.1 | 8.3 |
| Symfony | 3.x | 4.x | 6.x | 7.x |
| jQuery (core) | ✅ | ✅ | ❌ retiré | ❌ |
| CKEditor | v4 | v4 | v5 | v5 |
| Plugin annotations `@Block` | ✅ | ✅ | ✅ | ⚠️ déprécié |
| PHP attributes `#[Block]` | ❌ | ❌ | ✅ optionnel | ✅ **standard** |
| `drupal_set_message()` | ✅ | ❌ | ❌ | ❌ |
| Guzzle | 6 | 7 | 7 | 7 |
| Drush minimum | 8 | 10 | 12 | 13 |
| `config_sync_directory` (settings.php) | ancien format | ✅ | ✅ | ✅ |
| Migrate API (core) | ✅ | ✅ | ✅ | ✅ |

---

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Pièges découverts en projets réels (D8.9→D10, modules obsolètes, etc.)
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions du skill.

**Workflow :** problème découvert en projet → corriger le fichier source → ajouter entrée dans `lessons.md` → incrémenter CHANGELOG.

---

## Pipeline d'Automatisation Multi-Agents

Pour les upgrades de version majeure automatisés (D9→D10→D11→D12) :

```bash
/drupal-migrate    # Migration majeure complète (D10→D11, D11→D12)
/drupal-update     # Patches sécurité et mises à jour mineures
/drupal-status     # Analyse lecture seule — sans modification
/drupal-dry-run    # Simulation complète sans changement
/drupal-rollback   # Restauration depuis le dernier snapshot
```

**Pipeline 10 agents :**

| Agent | Rôle |
|-------|------|
| `env-detector` | Détecte PHP, DB, modules, thèmes, CI |
| `pre-flight` | Backup DB, export config, branche migration |
| `patch-manager` | Valide les patches Composer avant/après |
| `compatibility-analyzer` | Scanne dépréciations, Symfony 7, modules |
| `code-fixer` | Auto-corrige annotations→attributes, API deprecated |
| `test-runner` | Snapshots HTML, tests création/édition contenu |
| `updater` | composer update + drush updb + checklist |
| `config-doctor` | Views cassées, config_split, entity definitions |
| `db-health` | Version MariaDB, JSON support, tables orphelines |
| `rollback-manager` | Restauration automatique sur échec critique |

→ Documentation complète dans `agents/`

---

## See Also

- `drupal-core` — Architecture modules, hooks, Plugin system, EntityAPI
- `drupal-config` — Config Management, `drush cim`, UUID conflicts
- `drupal-testing` — PHPUnit, tests de régression post-upgrade
- `composer` — Patches composer, dépôts privés, optimisation autoloader
- `rector` — Rector PHP — refactoring automatique, dry-run, sets
- `drush` — Drush CLI — automatisation, aliases, déploiement
