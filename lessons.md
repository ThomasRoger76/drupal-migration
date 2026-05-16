---
name: drupal-migration lessons
description: Pièges et incidents découverts en projets de migration et upgrade Drupal réels.
---

# Leçons — drupal-migration

Incidents et pièges rencontrés en projets réels de migration de version et de données. Format : symptôme → cause → correction → prévention.

---

## Comment ajouter une leçon

Après chaque incident de migration :
1. Identifier si le skill aurait pu prévenir l'erreur
2. Ajouter une entrée avec date, symptôme, cause, correction, prévention
3. Mettre à jour CHANGELOG.md

---

### 2026-05-14 — `$config_directories` fatal D9 — bootstrap impossible

- **Symptôme :** Après bump vers D9, PHP Fatal à chaque requête avant même que Drupal charge
- **Cause :** `$config_directories['sync']` dans `settings.php` (syntaxe D8) supprimée en D9 — `\Drupal\Core\Site\Settings` exception au bootstrap
- **Correct :** Remplacer dans `settings.php` et `settings.local.php` : `$config_directories['sync']` → `$settings['config_sync_directory']`
- **Prévention :** `grep -r "config_directories" web/sites/` avant tout upgrade D8→D9. Agent compatibility-analyzer le détecte automatiquement.

### 2026-05-14 — Sauter D8→D10 directement — 47 hook_update_N manquants

- **Symptôme :** Base de données incohérente après upgrade direct D8→D10, rollback forcé
- **Cause :** Les scripts de mise à jour s'enchaînent séquentiellement — sauter D9 laisse des transformations de schéma non appliquées
- **Correct :** Rollback complet. Respecter D8.9.x → D9.5.x → D10.x
- **Prévention :** Règle absolue : jamais de saut de version majeure. Vérifier `drush core:requirements` après chaque étape.

### 2026-05-14 — CKEditor 4→5 — text formats custom cassés

- **Symptôme :** Après upgrade D9→D10, toolbar CKEditor vide pour les formats custom. `drush updb` avait semblé réussir.
- **Cause :** `hook_update_N` migrate automatiquement les formats core (`basic_html`, `full_html`) mais pas les formats personnalisés
- **Correct :** Vérifier `/admin/config/content/formats`. Inspecter `config/sync/editor.editor.*.yml` dans le diff git. Corriger les YAMLs manuellement ou via `drush config:edit`.
- **Prévention :** Après D9→D10, tester CHAQUE format de texte custom manuellement. Ajouter dans la checklist de recette.

### 2026-05-14 — migrate:import sans groupe — table de mapping orpheline

- **Symptôme :** Migration non visible dans `drush migrate:status`. Table `migrate_map_*` persistante avec entrées obsolètes après rollback.
- **Cause :** Sans `migration_group`, `migrate_tools` ne liste pas la migration. La table de mapping persiste si le rollback est interrompu.
- **Correct :** `drush migrate:reset-status {id}` puis `drush php:eval "\Drupal::database()->truncate('migrate_map_ID')->execute();"` puis réimporter depuis zéro.
- **Prévention :** Toujours définir `migration_group` dans les YAML. Vérifier le statut avant relance.

### 2026-05-16 — Modules contrib sans version D10 — composer.json bloqué

- **Symptôme :** `composer require drupal/core-recommended:^10` échoue avec "could not be resolved"
- **Cause :** Un module contrib installé n'a pas de release compatible D10. `composer why-not drupal/core:^10` révèle le coupable.
- **Correct :** `composer why-not drupal/core:^10` → identifier le module → chercher une version D10 sur drupal.org → si inexistant : désinstaller et trouver une alternative
- **Prévention :** `drush upgrade_status:analyze --all` avant tout upgrade. L'agent compatibility-analyzer le fait automatiquement.

### 2026-05-16 — PHP 8.3 requis D11 — modules custom avec typage insuffisant

- **Symptôme :** Fatal error sur des méthodes qui manquent de `: void` return type après upgrade D10→D11
- **Cause :** D11 enforce les return types sur les méthodes overridées (ex: `buildForm`, `submitForm`, `validateForm`)
- **Correct :** Ajouter `: void` aux méthodes `buildForm()`, `submitForm()`, `validateForm()` + corriger tous les overrides via Rector : `vendor/bin/rector process web/modules/custom`
- **Prévention :** L'agent compatibility-analyzer scan spécifiquement ces patterns (étape 5f). Lancer Rector --dry-run avant l'upgrade.

### 2026-05-16 — Backup DB manquant avant `drush updb` — perte irrécupérable

- **Symptôme :** `hook_update_N` échoue à mi-chemin, DB dans un état intermédiaire incohérent, pas de rollback possible
- **Cause :** Pas de dump DB avant `drush updb`. Certains updates sont partiellement appliqués.
- **Correct :** Restaurer depuis le backup précédent (si disponible). Sinon : intervention manuelle table par table.
- **Prévention :** Le pre-flight agent fait automatiquement `drush sql:dump --gzip`. Ne jamais lancer `drush updb` en production sans backup vérifié.

### 2026-05-16 — Annotations → Attributes D11 — module non découvert

- **Symptôme :** Block plugin ou Service custom non détecté après upgrade D10→D11
- **Cause :** D11 a déprécié les annotations `@Block(...)` — sans migration vers `#[Block(...)]`, le plugin est ignoré silencieusement dans certains contextes
- **Correct :** `vendor/bin/rector process web/modules/custom --dry-run` pour voir toutes les migrations d'annotations. Appliquer avec `vendor/bin/rector process web/modules/custom`
- **Prévention :** L'agent code-fixer gère automatiquement annotations→attributes. Vérifier dans l'UI Drupal que les blocs/plugins apparaissent après upgrade.

### 2026-05-16 — Migration D7 — paragraphes perdus après rollback partiel

- **Symptôme :** Après un `drush migrate:rollback --all`, des entités `paragraph` orphelines persistent en DB sans nœud parent
- **Cause :** Le rollback de `d7_node` supprime les nœuds mais pas automatiquement les paragraphes orphelins (table `paragraphs_item_revision`)
- **Correct :** `drush php:eval` pour trouver et supprimer les paragraphes orphelins + `drush php:eval "\Drupal::entityTypeManager()->getStorage('paragraph')->delete()"` avec requête filtrée
- **Prévention :** Toujours rollback dans l'ordre inverse de l'import : nœuds → paragraphes → taxonomies → users. Utiliser les groupes de migration avec `--execute-dependencies`.
