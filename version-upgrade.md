---
name: drupal-migration — version-upgrade
description: Guide complet pour upgrader Drupal d'une version majeure à la suivante (D8→D9, D9→D10, D10→D11). Commandes Composer, Drush, Rector, stratégie de rollback.
---

# Version Upgrade Drupal — Guide Complet

## Vue d'ensemble du processus

```
1. Analyse pré-upgrade  →  2. Backup  →  3. Mise à jour Composer
         ↓
4. composer install     →  5. drush updb  →  6. Fix code déprécié
         ↓
7. drush cim            →  8. Tests       →  9. Rollback si échec
```

---

## 1. Analyse pré-upgrade

### Vérifier l'état actuel

```bash
# Version Drupal + PHP actuel
drush core:status

# Modules en retard (dépendances directes seulement)
composer outdated --direct

# Modules sans version compatible avec la cible
composer why-not drupal/core:^11

# Vérifier les updates disponibles pour les modules
drush pm:list --status=enabled --type=module

# Modules contrib qui n'ont pas de version D10/D11
composer require drupal/upgrade_status --dev
drush en upgrade_status
drush upgrade_status:analyze --all
```

### Outils d'analyse de code

```bash
# drupal-check : dépréciations et erreurs fatales (rapide)
composer require --dev mglaman/drupal-check
./vendor/bin/drupal-check web/modules/custom

# PHPStan avec règles Drupal
composer require --dev phpstan/phpstan mglaman/phpstan-drupal
./vendor/bin/phpstan analyse web/modules/custom --level=2

# Rector : aperçu des corrections automatiques possibles
composer require --dev palantirnet/drupal-rector
./vendor/bin/rector process web/modules/custom --dry-run
```

---

## 2. Backup obligatoire

```bash
# Backup base de données
drush sql:dump --gzip --result-file=backup_$(date +%Y%m%d_%H%M%S).sql.gz

# Backup fichiers (si nécessaire)
tar -czf files_backup_$(date +%Y%m%d).tar.gz web/sites/default/files/

# Git : créer un tag de sauvegarde
git tag pre-upgrade-d$(drush php:eval "echo \Drupal::VERSION;")
git push origin --tags
```

---

## 3. Patterns d'upgrade par version

### D8.9 → D9 (le plus délicat)

**Contraintes :**
- PHP minimum : 7.3 (PHP 8.0 recommandé)
- Tous les modules contrib doivent avoir une version compatible D9
- Symfony passe de 3.x à 4.x : certaines dépendances tierces peuvent casser

**Étapes :**

```bash
# 1. Mettre à jour composer.json
composer require \
  drupal/core-recommended:^9 \
  drupal/core-composer-scaffold:^9 \
  drupal/core-project-message:^9 \
  --update-with-dependencies

# 2. Si vous utilisez drupal/core (sans recommended)
composer require drupal/core:^9 --update-with-dependencies

# 3. Mettre à jour Drush (D9 requiert Drush 10+)
composer require drush/drush:^10

# 4. Vérifier les modules contrib
# Chaque module doit avoir une version ^2.x ou indiquée comme compatible D9
# Exemple : pathauto ^1.8, metatag ^1.14, etc.

# 5. Appliquer les mises à jour
drush updb -y
drush cim -y
drush cr
```

**Pièges D8→D9 spécifiques :**
- `$config_directories['sync']` dans settings.php → remplacer par `$settings['config_sync_directory']`
- `Drupal\Component\Utility\SafeMarkup` supprimé → utiliser `Markup::create()` ou `Xss::filter()`
- `drupal_set_message()` supprimé → `\Drupal::messenger()->addStatus()`
- `db_query()`, `db_select()` supprimés → `\Drupal::database()->query()`

---

### D9 → D10 (ruptures majeures)

**Contraintes :**
- PHP minimum : 8.1 (obligatoire)
- CKEditor 4 complètement retiré → migration vers CKEditor 5 obligatoire
- jQuery et plusieurs dépendances JS retirées du core
- Symfony 6.x
- Drush minimum : 12

**Étapes :**

```bash
# 1. Vérifier la compatibilité PHP
php -v  # Doit être >= 8.1

# 2. Upgrade Composer
composer require \
  drupal/core-recommended:^10 \
  drupal/core-composer-scaffold:^10 \
  drupal/core-project-message:^10 \
  --update-with-dependencies

# 3. Drush 12
composer require drush/drush:^12

# 4. CKEditor 4 → 5 : migration de la configuration
# Le module ckeditor est retiré, ckeditor5 prend le relais
# La migration se fait via hook_update_N dans le core — mais les
# text formats personnalisés doivent être vérifiés manuellement
drush updb -y  # lance la migration CKEditor automatique

# 5. Vérifier les text formats après migration
drush cex -y
git diff config/sync/  # Inspecter les fichiers editor.editor.*.yml

# 6. Modules qui utilisaient jQuery directement
# Chercher les appels $.ajax(), $.fn.*, jQuery() dans les JS custom
grep -r "jQuery\|drupalSettings" web/modules/custom/ --include="*.js"
```

**CKEditor 4 → 5 — guide complet de migration :**

La migration automatique (`drush updb`) convertit les text formats **standard**. Les formats personnalisés nécessitent une intervention manuelle.

```bash
# 1. Identifier les text formats impactés
drush php:eval "
  foreach (\Drupal::entityTypeManager()->getStorage('editor')->loadMultiple() as \$editor) {
    echo \$editor->id() . ' → ' . \$editor->getEditor() . PHP_EOL;
  }
"
# Sortie : basic_html → ckeditor5, full_html → ckeditor5, custom_format → ckeditor (pas encore migré!)

# 2. Forcer la migration d'un format spécifique
drush php:eval "
  \$update = \Drupal::entityTypeManager()->getStorage('editor')->load('custom_format');
  if (\$update && \$update->getEditor() === 'ckeditor') {
    \$update->setEditor('ckeditor5');
    \$update->setSettings(['toolbar' => ['items' => ['bold', 'italic', 'link']]]);
    \$update->save();
    echo 'Migré.';
  }
"

# 3. Vérifier la config résultante
drush cex -y
git diff config/sync/editor.editor.custom_format.yml
```

```yaml
# Avant (D9) : editor.editor.full_html.yml
editor: ckeditor
settings:
  toolbar:
    rows:
      - - name: Formatting
          items:
            - Bold
            - Italic
            - Blockquote
            - Source

# Après (D10) : editor.editor.full_html.yml
editor: ckeditor5
settings:
  toolbar:
    items:
      - bold
      - italic
      - blockQuote
      - sourceEditing        # Remplace "Source"
  plugins:
    ckeditor5_heading:
      enabled_headings:
        - heading2
        - heading3
    ckeditor5_sourceEditing:
      allowed_tags: []       # Tags HTML autorisés en mode source
```

**Plugins CKEditor4 custom → CKEditor5 :**

Les plugins CKEditor 4 (`*.js` déclarés dans `ckeditor.yml`) n'ont **pas d'équivalent automatique** en CKEditor 5.

```bash
# Identifier les plugins custom
grep -r "ckeditor\[plugins\]" web/modules/custom/ --include="*.yml"

# Solution 1 : module contrib équivalent (ex: ckeditor5_font)
composer require drupal/ckeditor5_font

# Solution 2 : écrire un plugin CKEditor5 (format ES module, très différent de CKEditor4)
# Voir : https://www.drupal.org/docs/contributed-modules/ckeditor-5/creating-custom-ckeditor-5-plugins
```

**⚠️ Erreur fréquente : text format "Full HTML" casse après migration**

```bash
# Symptôme : "The CKEditor 5 configuration for filter format full_html
# is invalid: No plugin found for the following toolbar items: justifyLeft"

# Cause : plugins obsolètes dans la config CKEditor4

# Fix : supprimer les items non supportés de la toolbar CKEditor5
drush config:edit editor.editor.full_html
# Puis relancer : drush cr && tester l'éditeur en UI
```

**jQuery retiré — modules impactés fréquents :**
- `jquery_ui_*` → remplacer par équivalents Drupal natifs ou modules maintenus
- `select2` → vérifier la version compat D10
- JS custom avec `Drupal.behaviors` utilisant `$` → remplacer par `once()` + vanilla JS

```javascript
// Avant (D9)
Drupal.behaviors.monBehavior = {
  attach: function(context, settings) {
    $(context).find('.mon-element').once('mon-behavior').each(function() {
      // ...
    });
  }
};

// Après (D10)
Drupal.behaviors.monBehavior = {
  attach: function(context, settings) {
    once('mon-behavior', '.mon-element', context).forEach(function(element) {
      // ...
    });
  }
};
```

---

### D10 → D11 (annotations → attributes)

**Contraintes :**
- PHP minimum : 8.3 (obligatoire)
- Annotations de plugins dépréciées → PHP Attributes obligatoires
- Symfony 7.x
- Drush minimum : 13

**Étapes :**

```bash
# 1. Vérifier PHP 8.3
php -v  # Doit être >= 8.3

# 2. Upgrade Composer
composer require \
  drupal/core-recommended:^11 \
  drupal/core-composer-scaffold:^11 \
  drupal/core-project-message:^11 \
  --update-with-dependencies

# 3. Drush 13
composer require drush/drush:^13

# 4. Rector pour migrer les annotations → attributes automatiquement
./vendor/bin/rector process web/modules/custom

# 5. Exécuter les updates
drush updb -y
drush cim -y
drush cr
```

**Annotations → PHP Attributes (D10→D11) :**

```php
// Avant (D10 et antérieur) — annotation dans docblock
/**
 * @Block(
 *   id = "mon_bloc",
 *   admin_label = @Translation("Mon Bloc"),
 *   category = @Translation("Custom")
 * )
 */
class MonBloc extends BlockBase {

// Après (D11) — PHP 8 attribute
#[Block(
  id: 'mon_bloc',
  admin_label: new TranslatableMarkup('Mon Bloc'),
  category: new TranslatableMarkup('Custom'),
)]
class MonBloc extends BlockBase {
```

```php
// Hook via attribute (D11+)
// Avant : dans un fichier .module
function mon_module_entity_presave(EntityInterface $entity): void {
  // ...
}

// Après (D11 optionnel) : dans une classe
use Drupal\Core\Hook\Attribute\Hook;

class MonModuleHooks {
  #[Hook('entity_presave')]
  public function onEntityPresave(EntityInterface $entity): void {
    // ...
  }
}
```

---

## 4. Commandes de déploiement standard

```bash
# Séquence complète post-upgrade
drush updb -y          # Exécute tous les hook_update_N en attente
drush cim -y           # Importe la configuration depuis sync/
drush cr               # Vide tous les caches

# Ou en une seule commande (recommandé)
drush deploy           # updb + cim + cr dans l'ordre correct

# Vérification finale
drush core:status
drush pm:list --status=enabled | grep -i "not installed\|missing"
```

---

## 5. Stratégie de rollback

```bash
# Si l'upgrade a échoué après drush updb
# Restaurer la DB depuis le backup
drush sql:drop -y
drush sql:cli < backup_20260514_120000.sql

# Ou avec le fichier gzippé
gunzip -c backup_20260514_120000.sql.gz | drush sql:cli

# Revenir au tag git
git checkout pre-upgrade-d10

# Réinstaller les dépendances de l'ancienne version
composer install

# Vider les caches
drush cr
```

---

## 6. Checklist pré-mise en production

```
[ ] Backup DB vérifié (restauration testée)
[ ] Tests automatisés (PHPUnit + Behat) passent
[ ] Tous les modules contrib ont une version stable compatible
[ ] drupal-check : 0 erreur fatale dans les modules custom
[ ] Rector : toutes les corrections appliquées et testées
[ ] CKEditor vérifié (si D9→D10) : éditeurs fonctionnels
[ ] Performance : caches Varnish/Redis vidés
[ ] Monitoring : erreurs PHP dans les logs après déploiement
[ ] Rollback testé en staging
```

---

## Voir aussi

- [deprecated-code.md](deprecated-code.md) — Rector, drupal-check, PHPStan en détail
- [migrate-api.md](migrate-api.md) — Importer des données depuis D7 ou sources externes
- Skill `rector` — Rector PHP en détail
- Skill `composer` — Gestion avancée des dépendances
