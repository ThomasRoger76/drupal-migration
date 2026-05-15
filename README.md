# drupal-migration

Skill complet pour les deux types de "migration" en Drupal :

## Sous-fichiers

| Fichier | Sujet |
|---------|-------|
| [SKILL.md](SKILL.md) | Point d'entrée — Quick Decision Table, anti-patterns |
| [version-upgrade.md](version-upgrade.md) | Upgrade majeur D8→D9→D10→D11 (Composer, Drush, Rector) |
| [migrate-api.md](migrate-api.md) | Migrate API — importer des données (CSV, D7, XML) |
| [deprecated-code.md](deprecated-code.md) | Corriger le code déprécié (Rector, drupal-check, PHPStan) |
| [lessons.md](lessons.md) | Pièges réels découverts en projet |

## Distinction fondamentale

- **Version Upgrade** = faire évoluer le *code* du projet (D8→D10)
- **Migrate API** = importer des *données* depuis une source externe
