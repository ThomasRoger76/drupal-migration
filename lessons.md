---
name: drupal-migration lessons
description: Lessons learned from real migration projects
---

# Lessons — drupal-migration

---

## [2026-05-14] D8.9 → D10 : `$config_directories` est le premier bloquant

**Contexte :** Projet Drupal 8.9.x. Lors du premier `composer install` après bump vers D9, Drupal crashait avec une erreur fatale dès le bootstrap — avant même `drush updb`.

**Cause :** `settings.php` contenait `$config_directories['sync']` (syntaxe D8). Cette clé est supprimée en D9 et provoque une `\Drupal\Core\Site\Settings` exception avant que la page s'affiche.

**Règle :** Avant tout upgrade D8→D9, chercher et remplacer systématiquement dans `settings.php` et `settings.local.php` :
```bash
grep -r "config_directories" web/sites/
# Remplacer :
# $config_directories['sync'] = '../config/sync';
# Par :
# $settings['config_sync_directory'] = '../config/sync';
```

---

## [2026-05-14] Ne jamais sauter D8→D10 directement

**Contexte :** Tentative de mise à jour directe D8→D10 pour gagner du temps. Résultat : 47 `hook_update_N` manquants, base de données incohérente, rollback forcé.

**Cause :** Les scripts de mise à jour s'enchaînent séquentiellement (`hook_update_8001`, `hook_update_9001`, `hook_update_10001`). Sauter D9 laisse des transformations de schéma non appliquées. Certains modules contrib ne gèrent pas non plus les migrations "en saut".

**Règle :** Toujours respecter les étapes intermédiaires : D8.9.x → D9.5.x → D10.x. Vérifier avec `drush core:requirements` après chaque étape.

---

## [2026-05-14] CKEditor 4→5 : les text formats custom sont souvent cassés

**Contexte :** Après upgrade D9→D10, l'éditeur de texte n'affichait plus les boutons de toolbar dans les back-offices CCI. `drush updb` avait semblé réussir.

**Cause :** Le `hook_update_N` de migration CKEditor 4→5 gère les configurations core (`basic_html`, `full_html`). Mais les text formats *custom* (ex: `cci_editeur_avance`) n'avaient pas de mapping automatique pour leurs plugins CKEditor custom. Drupal les migraient partiellement, laissant des `editor.editor.cci_editeur_avance.yml` invalides.

**Règle :** Après upgrade D9→D10, vérifier chaque text format custom dans `/admin/config/content/formats`. Inspecter les fichiers `config/sync/editor.editor.*.yml` dans le diff git et tester l'éditeur manuellement pour chaque format.

---

## [2026-05-14] migrate:import sur une migration sans groupe — table de mapping orpheline

**Contexte :** Lors d'un import CSV, une migration lancée sans `migration_group` ne montrait pas dans `drush migrate:status` après un rollback partiel. La table `migrate_map_import_articles` restait en base avec des entrées obsolètes.

**Cause :** Sans groupe défini, `migrate_tools` ne liste que les migrations déclarées dans un groupe. La table de mapping persiste même après rollback si des erreurs interrompent le processus.

**Règle :**
1. Toujours définir `migration_group` dans les migrations YAML.
2. Après un rollback forcé, vérifier : `drush migrate:reset-status {migration_id}` avant de relancer.
3. En cas de table corrompue : `drush php:eval "\Drupal::database()->truncate('migrate_map_import_articles')->execute();"` puis relancer l'import depuis zéro.
