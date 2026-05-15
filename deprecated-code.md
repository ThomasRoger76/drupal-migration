---
name: drupal-migration — deprecated-code
description: Corriger le code déprécié Drupal avec Rector, drupal-check, PHPStan. Checklist D10/D11 compliance, anti-patterns fréquents annotations→attributes.
---

# Code Déprécié Drupal — Détection et Correction

## Vue d'ensemble des outils

| Outil | Usage | Vitesse | Auto-fix |
|-------|-------|---------|----------|
| `drupal-check` | Erreurs fatales + dépréciations (rapide) | Rapide | Non |
| `phpstan` + drupal ruleset | Analyse statique approfondie | Moyen | Non |
| `rector` | Correction automatique du code | Lent | **Oui** |

Ordre recommandé : `drupal-check` d'abord (vue rapide), puis `rector --dry-run` (aperçu corrections), puis `rector` (application), puis `phpstan` (vérification finale).

---

## 1. drupal-check

### Installation

```bash
composer require --dev mglaman/drupal-check
```

### Utilisation

```bash
# Analyser un module custom
./vendor/bin/drupal-check web/modules/custom/mon_module

# Analyser tous les modules custom
./vendor/bin/drupal-check web/modules/custom/

# Vérifier la compatibilité avec une version cible
./vendor/bin/drupal-check --drupal-root=web web/modules/custom/ -d

# Format JSON pour CI/CD
./vendor/bin/drupal-check web/modules/custom/ --format=json

# Inclure les dépréciations (pas seulement les erreurs fatales)
./vendor/bin/drupal-check web/modules/custom/ --deprecations
```

### Lire les résultats

```
 ------ --------------------------------------------------------- 
  Line   web/modules/custom/mon_module/src/Controller/MonCtrl.php  
 ------ --------------------------------------------------------- 
  42     Call to deprecated function drupal_set_message(): in      
         drupal:8.5.0 and is removed from drupal:9.0.0. Use the    
         messenger() service instead.                               
  67     Call to deprecated function \Drupal::url(): in            
         drupal:8.0.0 and is removed from drupal:9.0.0. Use        
         Url::fromRoute() instead.                                  
 ------ --------------------------------------------------------- 
```

---

## 2. Rector Drupal

### Installation

```bash
composer require --dev palantirnet/drupal-rector
```

### Configuration (`rector.php` à la racine du projet)

```php
<?php
// rector.php

use DrupalRector\Set\Drupal9SetList;
use DrupalRector\Set\Drupal10SetList;
use DrupalRector\Set\Drupal11SetList;
use Rector\Config\RectorConfig;

return RectorConfig::configure()
    ->withPaths([
        __DIR__ . '/web/modules/custom',
        __DIR__ . '/web/themes/custom',
    ])
    ->withSets([
        // Choisir le set correspondant à la cible
        Drupal9SetList::DRUPAL_9,   // Pour D8→D9
        Drupal10SetList::DRUPAL_10, // Pour D9→D10
        Drupal11SetList::DRUPAL_11, // Pour D10→D11
    ])
    ->withImportNames()
    ->withPhpSets(); // Appliquer aussi les recteurs PHP natifs
```

### Commandes Rector

```bash
# Aperçu sans modifier (obligatoire avant d'appliquer)
./vendor/bin/rector process --dry-run

# Appliquer les corrections
./vendor/bin/rector process

# Cibler un répertoire spécifique
./vendor/bin/rector process web/modules/custom/mon_module --dry-run

# Voir les détails de chaque transformation
./vendor/bin/rector process --dry-run --output-format=json

# Appliquer uniquement certaines règles
./vendor/bin/rector process --only="DrupalRector\Drupal9\Rector\Function_\DrupalSetMessageRector"
```

### Résultat typique d'un dry-run

```
 3 files with changes
 
 1) web/modules/custom/mon_module/src/Controller/MonController.php
 
    ---------- begin diff ----------
    -    drupal_set_message($message, 'status');
    +    \Drupal::messenger()->addStatus($message);
    ----------- end diff -----------
    
    Applied rules:
    * DrupalRector\Drupal9\Rector\Function_\DrupalSetMessageRector
```

---

## 3. PHPStan avec règles Drupal

### Installation

```bash
composer require --dev \
  phpstan/phpstan \
  mglaman/phpstan-drupal \
  phpstan/phpstan-deprecation-rules
```

### Configuration (`phpstan.neon` à la racine)

```neon
# phpstan.neon
includes:
  - vendor/mglaman/phpstan-drupal/extension.neon
  - vendor/phpstan/phpstan-deprecation-rules/rules.neon

parameters:
  level: 2  # 0-9, commencer bas et monter progressivement
  
  paths:
    - web/modules/custom
    - web/themes/custom
  
  # Ignorer les faux positifs connus
  ignoreErrors:
    - '#Drupal calls should be avoided in classes, use dependency injection instead#'
  
  drupal:
    drupal_root: web
```

### Utilisation

```bash
# Analyse complète
./vendor/bin/phpstan analyse

# Avec rapport HTML
./vendor/bin/phpstan analyse --error-format=html > phpstan-report.html

# Générer une baseline (ignorer les erreurs existantes, focus sur les nouvelles)
./vendor/bin/phpstan analyse --generate-baseline phpstan-baseline.neon
```

---

## 4. Anti-patterns fréquents et leurs corrections

### `drupal_set_message()` → Messenger service

```php
// ❌ Avant (supprimé en D9)
drupal_set_message($this->t('Opération réussie'), 'status');
drupal_set_message($this->t('Une erreur est survenue'), 'error');

// ✅ Après
\Drupal::messenger()->addStatus($this->t('Opération réussie'));
\Drupal::messenger()->addError($this->t('Une erreur est survenue'));
\Drupal::messenger()->addWarning($this->t('Attention'));

// Dans un service (avec injection de dépendance)
use Drupal\Core\Messenger\MessengerInterface;

class MonService {
  public function __construct(
    private readonly MessengerInterface $messenger,
  ) {}
  
  public function doSomething(): void {
    $this->messenger->addStatus($this->t('OK'));
  }
}
```

### `db_query()` / `db_select()` → Database API

```php
// ❌ Avant (supprimé en D8)
$result = db_query("SELECT * FROM {node} WHERE nid = :nid", [':nid' => 1]);
$query = db_select('node', 'n')->fields('n')->condition('type', 'article');

// ✅ Après — via service
$connection = \Drupal::database();
$result = $connection->query("SELECT * FROM {node} WHERE nid = :nid", [':nid' => 1]);
$query = $connection->select('node', 'n')->fields('n')->condition('type', 'article');

// ✅ Encore mieux — Entity Query
$nids = \Drupal::entityQuery('node')
  ->condition('type', 'article')
  ->condition('status', 1)
  ->execute();
```

### `\Drupal::url()` → `Url::fromRoute()`

```php
// ❌ Avant (supprimé en D9)
$url = \Drupal::url('node.view', ['node' => 1]);

// ✅ Après
use Drupal\Core\Url;
$url = Url::fromRoute('node.view', ['node' => 1])->toString();
```

### `SafeMarkup::set()` → `Markup::create()`

```php
// ❌ Avant (supprimé en D9)
use Drupal\Component\Utility\SafeMarkup;
$safe = SafeMarkup::set('<b>Texte</b>');

// ✅ Après
use Drupal\Core\Render\Markup;
$safe = Markup::create('<b>Texte</b>');
```

### Annotations `@Block` → Attributes `#[Block]` (D11)

```php
// ❌ Avant (D8-D10 — annotion docblock)
use Drupal\Core\Block\BlockBase;

/**
 * @Block(
 *   id = "mon_bloc_custom",
 *   admin_label = @Translation("Mon Bloc Custom"),
 *   category = @Translation("Custom Blocks"),
 * )
 */
class MonBlocCustom extends BlockBase {

// ✅ Après (D11 — PHP attribute)
use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[Block(
  id: 'mon_bloc_custom',
  admin_label: new TranslatableMarkup('Mon Bloc Custom'),
  category: new TranslatableMarkup('Custom Blocks'),
)]
class MonBlocCustom extends BlockBase {
```

```php
// ❌ @Plugin Field Formatter (D10)
/**
 * @FieldFormatter(
 *   id = "mon_formatter",
 *   label = @Translation("Mon Formatter"),
 *   field_types = {"string"},
 * )
 */

// ✅ D11
use Drupal\Core\Field\Attribute\FieldFormatter;

#[FieldFormatter(
  id: 'mon_formatter',
  label: new TranslatableMarkup('Mon Formatter'),
  field_types: ['string'],
)]
```

### `hook_*` dans `.module` → `#[Hook]` attribute (D11 optionnel)

```php
// ❌/✅ D8-D10 (fonctionne encore en D11 mais déprécié)
// fichier: mon_module.module
function mon_module_node_presave(NodeInterface $node): void {
  // logique
}

// ✅ D11 — classe avec attribute
// fichier: src/Hook/MonModuleHooks.php
namespace Drupal\mon_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;
use Drupal\node\NodeInterface;

class MonModuleHooks {

  #[Hook('node_presave')]
  public function onNodePresave(NodeInterface $node): void {
    // logique
  }
}
```

### `$config_directories` → `$settings['config_sync_directory']`

```php
// ❌ Avant (D8 — supprimé D9+)
$config_directories['sync'] = '../config/sync';

// ✅ Après (D9+)
$settings['config_sync_directory'] = '../config/sync';
```

### `once()` dans les behaviors JS (D10)

```javascript
// ❌ Avant (D9 — jQuery UI once)
Drupal.behaviors.monBehavior = {
  attach: function (context) {
    $(once('init', '.mon-element', context)).each(function () {
      // ...
    });
  }
};

// ✅ Après (D10+ — once() natif, pas de jQuery requis)
Drupal.behaviors.monBehavior = {
  attach: function (context) {
    once('init', '.mon-element', context).forEach(function (element) {
      // ...
    });
  }
};
```

---

## 5. Checklist D10 compliance

```
[ ] PHP >= 8.1 sur le serveur
[ ] drupal-check : 0 erreur fatale sur les modules custom
[ ] drupal_set_message() → messenger() partout
[ ] $config_directories → $settings['config_sync_directory']
[ ] db_query()/db_select() supprimés
[ ] SafeMarkup::set() → Markup::create()
[ ] \Drupal::url() → Url::fromRoute()
[ ] jQuery : aucun appel direct à $.ajax(), $(selector), etc.
[ ] jquery_ui_* : modules remplacés ou supprimés
[ ] CKEditor : éditeur text format fonctionne (interface vérifiée)
[ ] Drush >= 12 installé
[ ] Tous les modules contrib en version ^2.x ou stable D10
[ ] Rector appliqué, diff reviewé, tests passent
```

## 6. Checklist D11 compliance

```
[ ] PHP >= 8.3 sur le serveur
[ ] Toutes les annotations @Block/@Plugin → PHP attributes #[Block]
[ ] Tests : 0 PHPUnit failure après conversion
[ ] Rector drupal11 set appliqué
[ ] PHPStan level 2 : 0 erreur critique
[ ] Drush >= 13 installé
[ ] Symfony 7.x compatible (pas de calls aux APIs dépréciées Symfony 6→7)
[ ] hook_* dans .module : prioriser les classes avec #[Hook] pour le code custom
[ ] drupal-check --deprecations : 0 sur les modules custom
[ ] Tous les modules contrib en version stable D11
```

---

## Workflow recommandé sur un projet réel

```bash
# 1. Audit rapide
./vendor/bin/drupal-check web/modules/custom/ --deprecations 2>&1 | tee drupal-check-report.txt

# 2. Aperçu des corrections Rector
./vendor/bin/rector process web/modules/custom --dry-run 2>&1 | tee rector-dry-run.txt

# 3. Appliquer Rector
./vendor/bin/rector process web/modules/custom

# 4. Vérifier le diff
git diff web/modules/custom/

# 5. PHPStan pour les erreurs restantes
./vendor/bin/phpstan analyse web/modules/custom/

# 6. Tests unitaires/fonctionnels
./vendor/bin/phpunit web/modules/custom/

# 7. Commit propre
git add web/modules/custom/
git commit -m "fix: correct deprecated code for D10/D11 compliance (rector + manual)"
```

---

## Voir aussi

- [version-upgrade.md](version-upgrade.md) — Guide d'upgrade complet
- [migrate-api.md](migrate-api.md) — Migrate API pour les données
- Skill `rector` — Rector PHP en profondeur
- Skill `drupal-core` — Anti-patterns et évolution par version
