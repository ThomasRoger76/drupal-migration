# Changelog — drupal-migration

## v2.0 — 2026-05-14

### Refonte complète du skill (était quasi vide — 20 lignes)

**Ajouts :**
- `SKILL.md` : Quick Decision Table complète, anti-patterns, tableau d'évolution D8→D11
- `version-upgrade.md` : guide complet upgrade D8→D9→D10→D11, Composer, Drush, Rector, rollback
- `migrate-api.md` : Migrate API complète — Source/Process/Destination plugins, CSV, D7, debugging
- `deprecated-code.md` : Rector, drupal-check, PHPStan, checklists D10/D11 compliance
- `lessons.md` : pièges réels (upgrade D8.9 → D10+, bootstrap PHP fatal, etc.)

**Corrections :**
- Le skill confondait "version upgrade" et "Migrate API" — les deux sujets sont maintenant séparés
- Les commandes `/drupal-migrate` référencées dans l'ancienne version n'existent pas — remplacées par les vraies commandes drush/composer

## v1.0 — (date inconnue)

- Placeholder initial — 20 lignes, aucun sous-fichier
