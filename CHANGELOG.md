# Changelog — drupal-migration

## v3.1 — 2026-06-09

### Cohérence interne + corrections ciblées

- `process-plugins-advanced.md` : exemples `CounterTracker` et `D7PathToUri` convertis des annotations `@MigrateProcess("id")` vers les attributs PHP `#[MigrateProcess(id: '…')]` (+ `use Drupal\migrate\Attribute\MigrateProcess;`). Aligne ces exemples sur `custom-plugins.md` et `migrate-api.md` (standard D11+). La nuance "annotations Migrate encore fonctionnelles en D11, obligatoires en attribut en D12" reste documentée.
- `agents/updater.md` : correction de l'extension du chemin de backup dans la section Notes du DEPLOYMENT-CHECKLIST (`db.sql.gz.gz` → `db.sql.gz`), désormais cohérent avec la procédure de rollback et le template `migration-success.md`.

---

## v3.0 — 2026-05-15 → 2026-05-16

### Pipeline multi-agents complet + corrections majeures

**Bug critique corrigé :**
- Description frontmatter : les triggers `/drupal-migrate`, `/drupal-update`, `/drupal-status`, `/drupal-rollback` et déclencheurs français ("migrer drupal", "mise à jour drupal", "upgrade drupal", "montée de version drupal") avaient été retirés lors de la refonte v2.0. Restaurés en v3.0 — sans eux, le skill ne se déclenche pas sur les commandes principales.
- Note : la v2.0 mentionnait à tort que ces commandes "n'existent pas" — elles sont les commandes principales du pipeline multi-agents.

**Pipeline multi-agents (10 agents — ajoutés le 2026-05-15) :**
- `agents/env-detector.md` — détecte PHP, DB, modules, thèmes, CI, DDEV/Lando/Docker
- `agents/pre-flight.md` — backup DB, export config, branch git, protection scaffold, disk check
- `agents/patch-manager.md` — valide les patches Composer avant/après migration
- `agents/compatibility-analyzer.md` — scan dépréciations (11 sous-étapes), modules contrib, thèmes custom, Symfony 7 patterns, PHP return types D11
- `agents/code-fixer.md` — auto-corrections annotations→attributes, API deprecated, Rector
- `agents/test-runner.md` — baseline HTML (avant) + validation (après) : diff HTML, API création/édition, crawl navigation (Views + menus), 900+ lignes
- `agents/updater.md` — composer update + drush updb + checklist
- `agents/config-doctor.md` — Views cassées, config_split, entity definitions
- `agents/db-health.md` — version MariaDB, JSON support, tables orphelines
- `agents/rollback-manager.md` — restauration DB + composer depuis snapshot, options R/F/M
- `config/known-deprecations.json` — base de données des patterns dépréciés par version

**Nouveaux fichiers de référence :**
- `d7-complete-migration.md` — pipeline D7→D10 complet (10 étapes YAML)
- `custom-plugins.md` — `SourcePluginBase`, `ProcessPluginBase` avec `#[MigrateSource/Process]`
- `multilingual-migration.md` — migrations multilingues, `translations: true`, plugin `language_map`
- `process-plugins-advanced.md` — 15 plugins avancés : `sub_process`, `iterator`, `flatten`, `merge`, `skip_on_empty`, `static_map`, `callback`, `migration_lookup` stubs
- `advanced-patterns.md` — migrations incrémentales, monitoring `MigrateEvents`, références circulaires

**SKILL.md — 2026-05-16 :**
- Section "Support Environnements Locaux" : tableau DDEV / Lando / Docker Compose / Local avec préfixes de commandes
- Section "Roadmap D12" : D12 prévu 2026, PHP 8.4+, Symfony 8.x, anticipation Rector

---

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

---

## v1.0 — (date inconnue)

- Placeholder initial — 20 lignes, aucun sous-fichier

---

## Compatibilité Drupal

| Skill version | Drupal source | Drupal cible | Notes |
|--------------|--------------|-------------|-------|
| v3.x | D8, D9, D10 | D9, D10, D11 | Pipeline multi-agents, Lando support, D12 roadmap |
| v2.x | D8, D9 | D9, D10 | Guides manuels, Migrate API |
| v1.x | — | — | Placeholder |
