---
name: drupal-migration — migrate-api
description: Migrate API Drupal — pipeline Source→Process→Destination, plugins CSV/XML/D7, drush migrate:import, migrate_plus, migrate_tools, debugging.
---

# Migrate API Drupal — Guide Complet

## Concepts fondamentaux

La Migrate API est un pipeline en trois étapes :

```
[Source]  →  [Process]  →  [Destination]
  Lire         Transformer     Créer/mettre à jour
  les data     les valeurs     les entités Drupal
```

Chaque migration est définie par un fichier de configuration YAML (ou un plugin PHP). Les modules core nécessaires :

```bash
drush en migrate migrate_drupal -y

# Modules contrib recommandés
composer require drupal/migrate_plus drupal/migrate_tools drupal/migrate_source_csv
drush en migrate_plus migrate_tools migrate_source_csv -y
```

---

## Structure d'une migration YAML

```yaml
# config/install/migrate_plus.migration.import_articles.yml

id: import_articles
label: 'Import des articles depuis CSV'
migration_group: mon_projet

# SOURCES DISPONIBLES : csv, url, d7_node, entity:node, xml...
source:
  plugin: csv
  path: 'public://imports/articles.csv'
  ids:
    - id
  delimiter: ','
  enclosure: '"'
  header_offset: 0  # 0 = première ligne est l'en-tête
  column_names:
    - id: Identifiant
    - title: Titre
    - body: Contenu
    - category: Catégorie
    - date: Date de publication
    - image_url: URL image

# PROCESS : transformation champ par champ
process:
  title: title

  body/value: body
  body/format:
    plugin: default_value
    default_value: basic_html

  field_categorie:
    plugin: migration_lookup
    migration: import_categories
    source: category

  created:
    plugin: callback
    callable: strtotime
    source: date

  uid:
    plugin: default_value
    default_value: 1

  status:
    plugin: default_value
    default_value: 1

# DESTINATION : entity:node, entity:taxonomy_term, entity:user...
destination:
  plugin: entity:node
  default_bundle: article
  # Pour les mises à jour (évite les doublons)
  overwrite_properties:
    - title
    - body
    - field_categorie

# Dépendances : cette migration a besoin que import_categories soit faite avant
migration_dependencies:
  required:
    - import_categories
  optional: []
```

---

## Plugins Source

### CSV (migrate_source_csv)

```yaml
source:
  plugin: csv
  path: 'public://imports/products.csv'
  ids:
    - sku
  delimiter: ';'          # Séparateur (défaut: virgule)
  enclosure: '"'          # Guillemets
  escape: '\\'            # Caractère d'échappement
  header_offset: 0        # Ligne d'en-tête (null si aucune)
  # Ou nommer les colonnes manuellement
  column_names:
    - sku: SKU
    - name: Nom du produit
    - price: Prix
```

### URL / JSON (migrate_plus)

```yaml
source:
  plugin: url
  data_fetcher_plugin: http
  data_parser_plugin: json
  urls:
    - 'https://api.exemple.com/produits'
  item_selector: /data     # Chemin JSONPath vers le tableau d'items
  ids:
    - id
  fields:
    - name: id
      label: ID
      selector: id
    - name: title
      label: Titre
      selector: attributes/title
    - name: body
      label: Corps
      selector: attributes/body
```

### XML (migrate_plus)

```yaml
source:
  plugin: url
  data_fetcher_plugin: http
  data_parser_plugin: xml
  urls:
    - 'https://flux.exemple.com/feed.xml'
  item_selector: /rss/channel/item
  ids:
    - guid
  fields:
    - name: guid
      label: GUID
      selector: guid
    - name: title
      label: Titre
      selector: title
    - name: description
      label: Description
      selector: description
    - name: pubDate
      label: Date
      selector: pubDate
```

### Drupal 7 (migrate_drupal)

```yaml
source:
  plugin: d7_node
  node_type: article
  # Connexion D7 définie dans settings.php :
  # $databases['migrate']['default'] = [...];
```

Configuration de la connexion D7 dans `settings.php` :

```php
$databases['migrate']['default'] = [
  'database' => 'drupal7_db',
  'username' => 'dbuser',
  'password' => 'dbpass',
  'prefix' => '',
  'host' => 'localhost',
  'port' => '3306',
  'namespace' => 'Drupal\\Core\\Database\\Driver\\mysql',
  'driver' => 'mysql',
];
```

---

## Plugins Process

### `get` — Copie directe

```yaml
process:
  title: title
  # Équivalent long :
  title:
    plugin: get
    source: source_field_name
```

### `default_value` — Valeur par défaut

```yaml
process:
  status:
    plugin: default_value
    default_value: 1

  # Avec valeur source en priorité
  uid:
    plugin: default_value
    source: author_id
    default_value: 1
```

### `migration_lookup` — Clé étrangère vers une autre migration

```yaml
process:
  field_categorie:
    plugin: migration_lookup
    migration: import_categories
    source: category_id
    # no_stub: true  # Ne pas créer d'entité vide si non trouvé
```

### `explode` — String → array

```yaml
process:
  field_tags:
    plugin: explode
    source: tags_string    # "php, drupal, symfony"
    delimiter: ', '
```

### `callback` — Fonction PHP native

```yaml
process:
  created:
    plugin: callback
    callable: strtotime
    source: date_string

  title:
    plugin: callback
    callable: trim
    source: raw_title
```

### `static_map` — Correspondance de valeurs

```yaml
process:
  field_statut:
    plugin: static_map
    source: status_code
    map:
      A: publie
      B: brouillon
      C: archive
    default_value: brouillon
```

### `concat` — Concaténation

```yaml
process:
  title:
    plugin: concat
    source:
      - prenom
      - nom
    delimiter: ' '
```

### `skip_on_empty` — Ignorer si vide

```yaml
process:
  field_image:
    plugin: skip_on_empty
    method: process  # ou 'row' pour ignorer toute la ligne
    source: image_url
```

### Pipeline de plugins (chaîne)

```yaml
process:
  field_date_pub:
    - plugin: skip_on_empty
      method: process
      source: date_raw
    - plugin: callback
      callable: strtotime
    - plugin: format_date
      from_format: 'U'
      to_format: 'U'
      from_timezone: 'Europe/Paris'
      to_timezone: 'UTC'
```

---

## Plugins Destination

### `entity:node`

```yaml
destination:
  plugin: entity:node
  default_bundle: article
```

### `entity:taxonomy_term`

```yaml
destination:
  plugin: entity:taxonomy_term
  default_bundle: tags
```

### `entity:user`

```yaml
destination:
  plugin: entity:user
```

### `entity:file` (pour les images)

```yaml
# Migration en deux étapes : d'abord le fichier, puis le nœud
# 1. Migration des fichiers
destination:
  plugin: entity:file

# Dans la migration des nœuds, utiliser migration_lookup
process:
  field_image/target_id:
    plugin: migration_lookup
    migration: import_images
    source: image_filename
  field_image/alt: image_alt
```

---

## Exemple complet : CSV vers nœuds Drupal

### Le fichier CSV (`articles.csv`)

```csv
id,title,body,tags,status,date
1,"Premier article","<p>Contenu du premier article</p>","drupal,cms",1,"2024-01-15"
2,"Deuxième article","<p>Contenu du second article</p>","php,symfony",1,"2024-02-20"
```

### Migration des tags (d'abord)

```yaml
# config/install/migrate_plus.migration.import_tags.yml
id: import_tags
label: 'Import des tags'

source:
  plugin: embedded_data
  data_rows:
    - { tag: 'drupal' }
    - { tag: 'cms' }
    - { tag: 'php' }
    - { tag: 'symfony' }
  ids:
    - tag

process:
  name: tag

destination:
  plugin: entity:taxonomy_term
  default_bundle: tags
```

### Migration des articles

```yaml
# config/install/migrate_plus.migration.import_articles.yml
id: import_articles
label: 'Import des articles'
migration_group: mon_projet

source:
  plugin: csv
  path: 'public://imports/articles.csv'
  ids:
    - id

process:
  title: title

  body/value: body
  body/format:
    plugin: default_value
    default_value: basic_html

  field_tags:
    plugin: migration_lookup
    migration: import_tags
    source: "@tags"  # Utiliser le champ transformé

  created:
    plugin: callback
    callable: strtotime
    source: date

  status: status
  uid:
    plugin: default_value
    default_value: 1

destination:
  plugin: entity:node
  default_bundle: article

migration_dependencies:
  required:
    - import_tags
```

### Groupe de migration (migrate_plus)

```yaml
# config/install/migrate_plus.migration_group.mon_projet.yml
id: mon_projet
label: 'Import Mon Projet'
description: 'Migrations du projet X'
source_type: CSV
shared_configuration:
  source:
    key: default
```

---

## Commandes Drush

```bash
# Lister les migrations disponibles
drush migrate:status

# Lancer une migration spécifique
drush migrate:import import_articles

# Lancer toutes les migrations d'un groupe
drush migrate:import --group=mon_projet

# Lancer avec limit (pour tester)
drush migrate:import import_articles --limit=10

# Lancer dans l'ordre des dépendances (--execute-dependencies)
drush migrate:import import_articles --execute-dependencies

# Rollback (supprime les entités créées, garde la table de mapping)
drush migrate:rollback import_articles

# Reset si une migration est coincée en "Importing"
drush migrate:reset-status import_articles

# Voir les messages d'erreur d'une migration
drush migrate:messages import_articles

# Relancer seulement les items en erreur
drush migrate:import import_articles --status=failed

# Mettre à jour les items déjà migrés
drush migrate:import import_articles --update

# Sync : importer + rollback des supprimés
drush migrate:import import_articles --sync
```

---

## Debugging de migrations

### Activer les logs verbeux

```bash
drush migrate:import import_articles --feedback=100 -v
```

### Inspecter la table de mapping

```sql
-- La table s'appelle migrate_map_{migration_id}
SELECT * FROM migrate_map_import_articles LIMIT 10;

-- sourceid1 = ID source, destid1 = ID Drupal créé
-- source_row_status : 0=importé, 1=erreur, 2=ignoré, 3=à supprimer
```

### Messages d'erreur détaillés

```bash
drush migrate:messages import_articles
# Ou en JSON pour traitement
drush migrate:messages import_articles --format=json
```

### Plugin source custom (PHP)

```php
<?php
// web/modules/custom/mon_module/src/Plugin/migrate/source/MonSource.php

namespace Drupal\mon_module\Plugin\migrate\source;

use Drupal\migrate\Plugin\migrate\source\SourcePluginBase;
use Drupal\migrate\Row;

/**
 * @MigrateSource(
 *   id = "mon_source_plugin"
 * )
 */
class MonSource extends SourcePluginBase {

  public function fields(): array {
    return [
      'id' => $this->t('Identifiant'),
      'title' => $this->t('Titre'),
      'body' => $this->t('Corps'),
    ];
  }

  public function getIds(): array {
    return [
      'id' => ['type' => 'integer'],
    ];
  }

  protected function initializeIterator(): \Iterator {
    // Retourner un itérateur sur les données source
    $data = $this->getData();
    foreach ($data as $row) {
      yield $row;
    }
  }

  private function getData(): array {
    // Votre logique de récupération des données
    return [];
  }

  public function __toString(): string {
    return 'Mon plugin source custom';
  }
}
```

---

## Voir aussi

- [version-upgrade.md](version-upgrade.md) — Mettre à jour la version de Drupal
- [deprecated-code.md](deprecated-code.md) — Corriger le code déprécié
- Skill `drush` — Drush CLI complet
- Skill `drupal-core` — Entity API, Database API
