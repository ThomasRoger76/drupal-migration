# Migration Multilingue — Drupal 10/11

## 1. Migrer du contenu multilingue avec `langcode`

```yaml
# config/install/migrate_plus.migration.import_articles_fr.yml
id: import_articles_fr
label: 'Import articles français'
source:
  plugin: csv
  path: /path/to/articles_fr.csv
  ids: [id]
  constants:
    langcode: fr
process:
  langcode:
    plugin: default_value
    default_value: fr
  title: titre
  'body/value': corps
  'body/format':
    plugin: default_value
    default_value: basic_html
destination:
  plugin: entity:node
  default_bundle: article
  translations: false  # false = créer une entité, true = ajouter une traduction
```

---

## 2. Migrer les traductions d'entités existantes

```yaml
# config/install/migrate_plus.migration.import_articles_de.yml
id: import_articles_de
label: 'Traductions allemandes des articles'
source:
  plugin: csv
  path: /path/to/articles_de.csv
  ids: [source_id]
process:
  langcode:
    plugin: default_value
    default_value: de
  nid:
    # Trouver l'entité source (version française)
    plugin: migration_lookup
    migration: import_articles_fr
    source: source_id
  title: titre_de
  'body/value': corps_de
destination:
  plugin: entity:node
  default_bundle: article
  translations: true   # true = ajouter traduction à l'entité existante
migration_dependencies:
  required:
    - import_articles_fr  # La version source doit exister d'abord
```

> **Règle critique :** `translations: true` sans `migration_dependencies` crée la traduction avant que l'entité source existe → erreur silencieuse ou entité orpheline.

---

## 3. Plugin process `language_map` — mapper les codes source

Quand la source utilise des codes différents des codes Drupal (ISO 639-2, noms complets, etc.) :

```php
<?php
// src/Plugin/migrate/process/LanguageMap.php
namespace Drupal\mon_module\Plugin\migrate\process;

use Drupal\migrate\Attribute\MigrateProcess;
use Drupal\migrate\MigrateExecutableInterface;
use Drupal\migrate\ProcessPluginBase;
use Drupal\migrate\Row;

/**
 * Mappe les codes de langue source vers les codes Drupal (ISO 639-1).
 *
 * Exemple d'usage :
 * @code
 * process:
 *   langcode:
 *     plugin: language_map
 *     source: langue_source
 * @endcode
 */
#[MigrateProcess(id: 'language_map')]
class LanguageMap extends ProcessPluginBase {

  protected array $map = [
    'fre'     => 'fr',   // ISO 639-2 → ISO 639-1
    'ger'     => 'de',
    'spa'     => 'es',
    'eng'     => 'en',
    'french'  => 'fr',   // Noms complets (ex : base Access/legacy)
    'german'  => 'de',
    'spanish' => 'es',
    'english' => 'en',
  ];

  public function transform(
    $value,
    MigrateExecutableInterface $migrate_executable,
    Row $row,
    string $destination_property,
  ): string {
    return $this->map[strtolower((string) $value)] ?? $value;
  }

}
```

Usage dans la migration :

```yaml
process:
  langcode:
    plugin: language_map
    source: langue_source  # colonne CSV avec "fre", "french", etc.
```

---

## 4. Migrer les traductions de Taxonomy Terms

```yaml
# config/install/migrate_plus.migration.import_tags_fr.yml
id: import_tags_fr
label: 'Import tags français (entités sources)'
source:
  plugin: csv
  path: /path/to/tags_fr.csv
  ids: [id]
process:
  langcode:
    plugin: default_value
    default_value: fr
  vid:
    plugin: default_value
    default_value: tags
  name: nom_fr
destination:
  plugin: entity:taxonomy_term
  default_bundle: tags
  translations: false
```

```yaml
# config/install/migrate_plus.migration.import_tags_de.yml
id: import_tags_de
label: 'Traductions allemandes des tags'
source:
  plugin: embedded_data
  data_rows:
    - { id: 1, lang: de, name: 'Technologie' }
    - { id: 2, lang: de, name: 'Gesundheit' }
  ids: [id, lang]
process:
  langcode: lang
  tid:
    plugin: migration_lookup
    migration: import_tags_fr
    source: id
  name: name
destination:
  plugin: entity:taxonomy_term
  default_bundle: tags
  translations: true
migration_dependencies:
  required:
    - import_tags_fr
```

---

## 5. D7 → D10/11 — Migration multilingue native

Le module `migrate_drupal` fournit des plugins source spécialisés pour D7 :

```yaml
# config/install/migrate_plus.migration.d7_node_article_fr.yml
id: d7_node_article_fr
label: 'Articles D7 → D10 (toutes langues)'
source:
  plugin: d7_node
  node_type: article
  langcodes: [fr, de, en]  # Migrer toutes les langues en une passe
process:
  langcode: language
  title: title
  'body/value': body_value
  'body/format': body_format
  status: status
  created: created
  changed: changed
destination:
  plugin: entity:node
  default_bundle: article
```

> **Astuce :** sans `langcodes:`, le plugin `d7_node` ne migre que `und` (undefined). Toujours lister les langues explicitement.

---

## 6. Ordre des migrations multilingues

```
1. Langues disponibles (activer fr, de, en via drush config:set)
2. Taxonomies sources (import_tags_fr)
3. Traductions taxonomies (import_tags_de, import_tags_en)
4. Médias sources (import_medias_fr)
5. Traductions médias (import_medias_de)
6. Nœuds sources (import_articles_fr)
7. Traductions nœuds (import_articles_de, import_articles_en)
```

---

## 7. Debugging migrations multilingues

```bash
# Vérifier les langues disponibles dans Drupal
docker compose exec php drush php:eval "
  foreach (\Drupal::languageManager()->getLanguages() as \$lang) {
    echo \$lang->getId() . ': ' . \$lang->getName() . PHP_EOL;
  }
"

# Statut des migrations par langcode
docker compose exec php drush migrate:status import_articles_fr
docker compose exec php drush migrate:status import_articles_de

# Vérifier qu'une traduction existe sur un nœud
docker compose exec php drush php:eval "
  \$node = \Drupal\node\Entity\Node::load(1);
  foreach (\$node->getTranslationLanguages() as \$langcode => \$lang) {
    echo \$langcode . ': ' . \$node->getTranslation(\$langcode)->getTitle() . PHP_EOL;
  }
"

# Voir les erreurs de mapping pour une migration
docker compose exec php drush migrate:messages import_articles_de

# Forcer le re-import d'une traduction spécifique
docker compose exec php drush migrate:import import_articles_de --idlist=42 --update
```

---

## Anti-patterns migration i18n

| ❌ À éviter | ✅ Bonne pratique | Raison |
|------------|------------------|--------|
| `translations: false` pour une migration de traduction | `translations: true` | Crée des nœuds dupliqués au lieu d'ajouter la traduction |
| Absence de `migration_dependencies` sur la traduction | Déclarer `required: [migration_source]` | La traduction est créée avant l'entité source → erreur FK |
| `langcode` codé en dur sans vérification | Plugin `language_map` + test `drush php:eval` | Codes invalides acceptés silencieusement, langcode `und` en production |
| `langcodes:` absent sur `d7_node` | Lister toutes les langues explicitement | Seuls les nœuds `und` sont migrés |
| Migrer traductions et entités sources en parallèle | Groupes de migrations avec `migration_dependencies` | Race condition → traductions sans entité source |

---

## See Also

- `migrate-api.md` — Migrations YAML standard
- `custom-plugins.md` — Créer des plugins source/process
- `advanced-patterns.md` — Stubs, rollback, groupes
