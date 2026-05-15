# drupal-migration — Claude Code Skill

Skill complet pour tout ce qui touche à la "migration" Drupal — deux approches complémentaires :

## 1. Pipeline d'automatisation multi-agents (`/drupal-migrate`)

Pipeline automatisé de 10 agents pour les upgrades de version majeure (D9→D10→D11→D12) :

| Agent | Rôle |
|-------|------|
| `env-detector` | Détecte l'environnement, versions PHP/DB, modules |
| `pre-flight` | Backup DB, export config, protection scaffold |
| `patch-manager` | Valide les patches Composer avant migration |
| `compatibility-analyzer` | Scanne le code custom (dépréciations, Symfony 7) |
| `code-fixer` | Auto-corrige les APIs dépréciées |
| `test-runner` | Snapshots HTML, tests création/édition contenu |
| `updater` | composer update + drush updb + checklist déploiement |
| `config-doctor` | Santé des Views, config_split, entity definitions |
| `db-health` | Version MariaDB/MySQL, support JSON, tables orphelines |
| `rollback-manager` | Restauration automatique sur échec critique |

→ Voir `agents/` pour la documentation de chaque agent.

## 2. Référence technique Migrate API

Documentation de référence pour les migrations de données (CSV, JSON, D7→D10, multilingue) :

| Fichier | Contenu |
|---------|---------|
| [SKILL.md](SKILL.md) | Quick Decision Table (49 entrées), anti-patterns |
| [version-upgrade.md](version-upgrade.md) | Guide D8→D9→D10→D11, Rector, CKEditor 4→5 |
| [migrate-api.md](migrate-api.md) | Pipeline Source→Process→Destination, YAML, drush |
| [custom-plugins.md](custom-plugins.md) | Plugins Source/Process/Destination en PHP |
| [advanced-patterns.md](advanced-patterns.md) | Paragraphs, incrémental, MigrateEvents, groupes |
| [multilingual-migration.md](multilingual-migration.md) | Migration multilingue, langcode, translations |
| [deprecated-code.md](deprecated-code.md) | Rector, drupal-check, PHPStan, checklists D10/D11 |
