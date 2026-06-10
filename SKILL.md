---
name: drupal-migration
description: Use when upgrading Drupal core versions (D8→D9→D10→D11), fixing deprecated APIs with Rector, planning or executing data migrations with the Migrate API (source/process/destination plugins, migrate_plus, migrate_tools), migrating from Drupal 7/8, importing CSV/XML/JSON data, or running drush migrate commands. Also triggers on /drupal-migrate, /drupal-update, /drupal-status, /drupal-rollback, "migrer drupal", "mise à jour drupal", "upgrade drupal", "montée de version drupal". Multi-agent pipeline with backup, compatibility analysis, auto-fix, testing, and rollback. Docker Compose (docker compose exec php) is the reference environment; also supports classic local installs, DDEV, and Lando in Drupal 8-11+
---

# Drupal Migration — Référence Complète

## Overview

Ce skill couvre **deux sujets distincts** qu'il ne faut pas confondre :

1. **Version Upgrade** — Mettre à jour Drupal d'une version majeure à une autre (D8→D9, D9→D10, D10→D11). Il s'agit de faire évoluer le *code et les dépendances* du projet.
2. **Migrate API** — Importer des *données* depuis une source externe (D7, CSV, XML, JSON) vers Drupal via le pipeline Source → Process → Destination.

> ⚠️ **Drupal 7 est EOL depuis le 5 janvier 2025** (fin du support commercial étendu progressif ensuite) : plus de correctifs de sécurité officiels. Toute migration D7 restante est **urgente** — l'argument décisif côté client. HTTPS obligatoire sur la source D7 pendant la migration (le site est vulnérable).

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
| Migration D7→D10 complète (rôles, vocabs, users, fichiers, nœuds) | Pipeline 10 étapes avec YAML complets | [d7-complete-migration.md](d7-complete-migration.md) |
| Configurer la connexion DB D7 dans settings.php | `$databases['migrate']['default']` | [d7-complete-migration.md](d7-complete-migration.md) |
| Ordre impératif des migrations D7 | Rôles → Vocabs → Terms → Fichiers → Users → Nœuds | [d7-complete-migration.md](d7-complete-migration.md) |
| Groupe de migrations (`migrate_plus`) exécuté en une commande | `migrate_plus.migration_group` + `--execute-dependencies` | [d7-complete-migration.md](d7-complete-migration.md) |
| Tester la connexion DB D7 depuis Drush | `drush php:eval` + `Database::getConnection('default', 'migrate')` | [d7-complete-migration.md](d7-complete-migration.md) |
| Compter les items importés / en erreur | `$map->importedCount()` + `$map->errorCount()` | [d7-complete-migration.md](d7-complete-migration.md) |
| Migrer un champ multi-valeurs (gallery, paragraphs) | `sub_process` plugin | [process-plugins-advanced.md](process-plugins-advanced.md) |
| Itérer sur un tableau de valeurs simples | `iterator` plugin | [process-plugins-advanced.md](process-plugins-advanced.md) |
| Aplatir un tableau imbriqué `[[v1],[v2]]` → `[v1,v2]` | `flatten` plugin | [process-plugins-advanced.md](process-plugins-advanced.md) |
| Combiner plusieurs sources en un seul champ | `merge` + `concat` plugins | [process-plugins-advanced.md](process-plugins-advanced.md) |
| Sauter un champ vide sans ignorer toute la row | `skip_on_empty` avec `method: process` | [process-plugins-advanced.md](process-plugins-advanced.md) |
| Sauter toute la row si une clé source est absente | `skip_row_if_not_set` | [process-plugins-advanced.md](process-plugins-advanced.md) |
| Mapper les formats de texte D7 (filtered_html → basic_html) | `static_map` plugin | [process-plugins-advanced.md](process-plugins-advanced.md) |
| Appliquer `trim`, `strtolower`, `strip_tags` dans un YAML | `callback` plugin | [process-plugins-advanced.md](process-plugins-advanced.md) |
| Références circulaires avec stubs entre migrations | `migration_lookup` + `no_stub: false` + relancer `--update` | [process-plugins-advanced.md](process-plugins-advanced.md) |
| Custom process plugin avec logging tous les 100 items | `ProcessPluginBase` + `\Drupal::logger()` | [process-plugins-advanced.md](process-plugins-advanced.md) |
| Convertir un chemin D7 en URI interne D10 | Custom plugin `D7PathToUri` | [process-plugins-advanced.md](process-plugins-advanced.md) |

---

## Support Environnements Locaux

Le pipeline multi-agents détecte automatiquement l'environnement et **paramètre toutes les commandes** via `command_prefix` (`${CMD}`) — il n'y a aucun préfixe codé en dur. **L'environnement de référence de ce skill est Docker Compose natif** (`docker compose exec php …`), conformément au workflow standard. DDEV et Lando restent supportés comme autres environnements.

| Environnement | Détection | Préfixe de commande (`${CMD}`) |
|---------------|-----------|---------------------|
| **Docker Compose** (référence) | `docker-compose.yml` / `compose.yaml` présent | `docker compose exec php` |
| **Local classique** | Aucun gestionnaire détecté | `` (vide) — `./vendor/bin/drush` direct |
| DDEV (autre env) | `ddev describe` réussit | `ddev` |
| Lando (autre env) | `.lando.yml` présent à la racine | `lando` |

L'agent `env-detector` détecte automatiquement l'environnement au démarrage et écrit `command_prefix` dans `environment.json`. Tous les autres agents lisent cette valeur — aucun ne suppose `ddev`.

**Docker Compose — commandes clés (env de référence) :**
```bash
docker compose exec php drush updb -y
docker compose exec php drush cim -y
docker compose exec php composer require drupal/core-recommended:^11 --update-with-dependencies
docker compose exec php composer install
```

---

## Roadmap D12

Drupal 12 est en cours de développement (sortie prévue 2026). Préparer dès maintenant :
- PHP 8.4+ requis (Drupal 12 suivra le cycle PHP)
- Symfony 8.x (migration depuis 7.x)
- Suppression des API dépréciées en D11
- Commencer par appliquer Rector avec le set D11 pour garantir la compatibilité D12

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
- `drupal-composer` — Patches Composer, version constraints, dépendances lors de l'upgrade
- `drupal-deployment` — drush deploy, drush aliases, déploiement post-migration
