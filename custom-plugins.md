# Migrate API — Plugins Custom PHP

## Vue d'ensemble

Quand les plugins core (CSV, SQL, JSON) ne suffisent pas, créer des plugins custom pour :
- **Source** : lire depuis une API REST, un format exotique, une base legacy
- **Process** : transformer les données selon une logique métier complexe
- **Destination** : écrire vers un système autre que les entités Drupal

Structure namespace : `Drupal\mon_module\Plugin\migrate\{source|process|destination}`

---

## 1. Custom Source Plugin

### Cas d'usage : importer depuis une API REST paginée

```php
// src/Plugin/migrate/source/ApiSource.php
namespace Drupal\mon_module\Plugin\migrate\source;

use Drupal\migrate\Annotation\MigrateSource;
use Drupal\migrate\Plugin\migrate\source\SourcePluginBase;
use Drupal\migrate\Plugin\MigrationInterface;
use Drupal\migrate\Row;
use GuzzleHttp\ClientInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Source plugin qui lit depuis une API REST paginée.
 *
 * Exemple d'utilisation dans une migration YAML :
 * @code
 * source:
 *   plugin: mon_module_api
 *   endpoint: 'https://api.exemple.com/articles'
 *   page_size: 50
 * @endcode
 *
 * @MigrateSource(
 *   id = "mon_module_api",
 *   source_module = "mon_module",
 * )
 */
class ApiSource extends SourcePluginBase {

  protected ClientInterface $httpClient;

  /**
   * {@inheritdoc}
   */
  public static function create(
    ContainerInterface $container,
    array $configuration,
    string $plugin_id,
    mixed $plugin_definition,
    ?MigrationInterface $migration = NULL
  ): static {
    $instance = parent::create(
      $container, $configuration, $plugin_id, $plugin_definition, $migration
    );
    $instance->httpClient = $container->get('http_client');
    return $instance;
  }

  /**
   * {@inheritdoc}
   *
   * Décrire les champs disponibles dans la source.
   */
  public function fields(): array {
    return [
      'id'      => $this->t('Identifiant unique'),
      'title'   => $this->t('Titre de l\'article'),
      'body'    => $this->t('Corps HTML'),
      'created' => $this->t('Timestamp de création (Unix)'),
      'author'  => $this->t('Email de l\'auteur'),
      'tags'    => $this->t('Liste de tags (séparés par virgule)'),
    ];
  }

  /**
   * {@inheritdoc}
   *
   * Déclarer les champs qui forment l'identifiant unique de chaque ligne.
   */
  public function getIds(): array {
    return [
      'id' => [
        'type'  => 'integer',
        'alias' => 's',
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function __toString(): string {
    return $this->configuration['endpoint'] ?? 'API Source';
  }

  /**
   * {@inheritdoc}
   *
   * Initialise l'itérateur qui fournit les lignes à migrer.
   * Gère la pagination automatiquement.
   */
  protected function initializeIterator(): \Iterator {
    $endpoint  = $this->configuration['endpoint'] ?? '';
    $page_size = (int) ($this->configuration['page_size'] ?? 50);
    $page      = 0;

    do {
      try {
        $response = $this->httpClient->get($endpoint, [
          'query' => [
            'page'     => $page,
            'per_page' => $page_size,
          ],
          'headers' => [
            'Accept'        => 'application/json',
            'Authorization' => 'Bearer ' . $this->getApiToken(),
          ],
          'timeout' => 30,
        ]);

        $data  = json_decode($response->getBody()->getContents(), TRUE);
        $items = $data['items'] ?? [];
        $total = $data['total'] ?? 0;

        foreach ($items as $item) {
          yield $item;
        }

        $page++;
      }
      catch (\Exception $e) {
        $this->idMap->saveMessage(
          [],
          $e->getMessage(),
          MigrationInterface::MESSAGE_ERROR
        );
        break;
      }
    } while (count($items) === $page_size && ($page * $page_size) < $total);
  }

  /**
   * {@inheritdoc}
   *
   * Transformation et validation avant le pipeline process.
   * Retourner FALSE pour ignorer une ligne.
   */
  public function prepareRow(Row $row): bool {
    // Nettoyer le titre
    $title = trim($row->getSourceProperty('title') ?? '');
    $row->setSourceProperty('title', $title);

    // Ignorer les articles sans corps de texte
    if (empty(trim($row->getSourceProperty('body') ?? ''))) {
      $this->idMap->saveMessage(
        $row->getSourceIdValues(),
        'Article ignoré : corps vide',
        MigrationInterface::MESSAGE_INFORMATIONAL
      );
      return FALSE;
    }

    // Convertir la date ISO en timestamp Unix si nécessaire
    $created = $row->getSourceProperty('created');
    if (is_string($created) && !is_numeric($created)) {
      $row->setSourceProperty('created', strtotime($created));
    }

    // Exploser les tags en tableau
    $tags = $row->getSourceProperty('tags');
    if (is_string($tags)) {
      $row->setSourceProperty('tags', array_filter(
        array_map('trim', explode(',', $tags))
      ));
    }

    return parent::prepareRow($row);
  }

  /**
   * Récupère le token API depuis la configuration.
   */
  protected function getApiToken(): string {
    return $this->configuration['api_token']
      ?? \Drupal::state()->get('mon_module.api_token', '');
  }

}
```

### Utilisation dans la migration YAML

```yaml
# config/install/migrate_plus.migration.import_articles.yml
id: import_articles
label: 'Import articles depuis API'
source:
  plugin: mon_module_api
  endpoint: 'https://api.exemple.com/v2/articles'
  page_size: 100
  api_token: 'secret_token_ici'  # Mieux : via \Drupal::state()
process:
  title: title
  body/value: body
  created: created
destination:
  plugin: 'entity:node'
  default_bundle: article
```

---

## 2. Custom Process Plugin

### Cas d'usage : formater un numéro de téléphone

```php
// src/Plugin/migrate/process/FormatTelephone.php
namespace Drupal\mon_module\Plugin\migrate\process;

use Drupal\migrate\Annotation\MigrateProcess;
use Drupal\migrate\MigrateExecutableInterface;
use Drupal\migrate\ProcessPluginBase;
use Drupal\migrate\Row;

/**
 * Formate un numéro de téléphone au format E.164 ou FR.
 *
 * Exemple YAML :
 * @code
 * process:
 *   field_telephone:
 *     plugin: format_telephone
 *     source: phone_number
 *     format: fr  # ou 'e164'
 * @endcode
 *
 * @MigrateProcess(id = "format_telephone")
 */
class FormatTelephone extends ProcessPluginBase {

  /**
   * {@inheritdoc}
   */
  public function transform(
    mixed $value,
    MigrateExecutableInterface $migrate_executable,
    Row $row,
    string $destination_property
  ): string {
    if (empty($value)) {
      return '';
    }

    // Supprimer tout sauf chiffres et le + initial
    $clean = preg_replace('/[^0-9+]/', '', (string) $value);

    $format = $this->configuration['format'] ?? 'fr';

    if ($format === 'e164') {
      // Convertir 06XXXXXXXX → +336XXXXXXXX
      if (strlen($clean) === 10 && str_starts_with($clean, '0')) {
        return '+33' . substr($clean, 1);
      }
      return $clean;
    }

    // Format FR : 0X XX XX XX XX
    if (strlen($clean) === 10) {
      return implode(' ', str_split($clean, 2));
    }

    // Format international : retourner tel quel
    return $clean;
  }

}
```

### Process plugin avec gestion de plusieurs valeurs

```php
/**
 * @MigrateProcess(
 *   id = "extract_media_id",
 *   handle_multiples = TRUE,
 * )
 */
class ExtractMediaId extends ProcessPluginBase {

  /**
   * {@inheritdoc}
   *
   * Transforme un tableau d'URLs en IDs de Media Drupal.
   */
  public function transform(
    mixed $value,
    MigrateExecutableInterface $migrate_executable,
    Row $row,
    string $destination_property
  ): int|string|null {
    if (empty($value)) {
      return NULL;
    }

    // Chercher un Media existant par champ source
    $mids = \Drupal::entityQuery('media')
      ->condition('field_source_url', $value)
      ->accessCheck(FALSE)
      ->range(0, 1)
      ->execute();

    return $mids ? (int) reset($mids) : NULL;
  }

}
```

---

## 3. Custom Destination Plugin

### Cas d'usage : écrire dans une table custom ou un système externe

```php
// src/Plugin/migrate/destination/CustomStorage.php
namespace Drupal\mon_module\Plugin\migrate\destination;

use Drupal\migrate\Annotation\MigrateDestination;
use Drupal\migrate\Plugin\migrate\destination\DestinationBase;
use Drupal\migrate\Plugin\MigrationInterface;
use Drupal\migrate\Row;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Destination plugin vers une table custom.
 *
 * @MigrateDestination(id = "custom_storage")
 */
class CustomStorage extends DestinationBase {

  protected \Drupal\Core\Database\Connection $database;

  /**
   * {@inheritdoc}
   */
  public static function create(
    ContainerInterface $container,
    array $configuration,
    string $plugin_id,
    mixed $plugin_definition,
    MigrationInterface $migration
  ): static {
    $instance = parent::create(
      $container, $configuration, $plugin_id, $plugin_definition, $migration
    );
    $instance->database = $container->get('database');
    return $instance;
  }

  /**
   * {@inheritdoc}
   *
   * Retourner les IDs de destination pour la table de mapping.
   */
  public function import(Row $row, array $old_destination_id_values = []): array {
    $data = $row->getDestination();

    // Upsert dans la table custom
    $this->database->upsert('mon_module_imported_data')
      ->fields(array_keys($data))
      ->values($data)
      ->key('source_id')
      ->execute();

    // Retourner les valeurs correspondant aux getIds()
    return [$data['source_id']];
  }

  /**
   * {@inheritdoc}
   *
   * Rollback : supprimer les lignes importées.
   */
  public function rollback(array $destination_identifier): void {
    $this->database->delete('mon_module_imported_data')
      ->condition('source_id', $destination_identifier['source_id'])
      ->execute();
  }

  /**
   * {@inheritdoc}
   *
   * Déclarer les champs qui forment l'ID unique de destination.
   */
  public function getIds(): array {
    return [
      'source_id' => [
        'type'        => 'integer',
        'description' => 'ID source de l\'enregistrement importé',
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function fields(MigrationInterface $migration = NULL): array {
    return [
      'source_id'  => 'ID source',
      'title'      => 'Titre',
      'content'    => 'Contenu',
      'imported_at'=> 'Date d\'import (timestamp)',
    ];
  }

}
```

---

## 4. Utilisation combinée dans une migration YAML

```yaml
# config/install/migrate_plus.migration.import_contacts.yml
id: import_contacts
label: 'Import contacts depuis API externe'

source:
  plugin: mon_module_api
  endpoint: 'https://crm.exemple.com/api/contacts'
  page_size: 200

process:
  # Copie directe
  nid: id

  # Champ simple
  title: name

  # Process plugin custom
  field_telephone:
    plugin: format_telephone
    source: phone
    format: fr

  # Chaîne de plugins
  field_email:
    - plugin: skip_on_empty
      method: row
      source: email
    - plugin: callback
      callable: strtolower

  # Référence à un terme de taxonomie existant
  field_categorie:
    plugin: entity_lookup
    source: category_name
    entity_type: taxonomy_term
    bundle: categories
    value_key: name

destination:
  plugin: custom_storage

# Dépendances de migration
migration_dependencies:
  required:
    - import_categories
```

---

## 5. Tester un plugin custom

```php
// tests/Kernel/Plugin/migrate/process/FormatTelephoneTest.php
namespace Drupal\Tests\mon_module\Kernel\Plugin\migrate\process;

use Drupal\KernelTests\KernelTestBase;
use Drupal\mon_module\Plugin\migrate\process\FormatTelephone;
use Drupal\migrate\MigrateExecutable;
use Drupal\migrate\Row;

class FormatTelephoneTest extends KernelTestBase {

  public function testFormatFr(): void {
    $plugin = new FormatTelephone(['format' => 'fr'], 'format_telephone', []);
    $result = $plugin->transform('0612345678', $this->createMock(MigrateExecutable::class), new Row(), 'field_tel');
    $this->assertEquals('06 12 34 56 78', $result);
  }

  public function testFormatE164(): void {
    $plugin = new FormatTelephone(['format' => 'e164'], 'format_telephone', []);
    $result = $plugin->transform('0612345678', $this->createMock(MigrateExecutable::class), new Row(), 'field_tel');
    $this->assertEquals('+33612345678', $result);
  }

  public function testEmpty(): void {
    $plugin = new FormatTelephone([], 'format_telephone', []);
    $result = $plugin->transform('', $this->createMock(MigrateExecutable::class), new Row(), 'field_tel');
    $this->assertEmpty($result);
  }

}
```

---

## Anti-patterns plugins custom

| ❌ À éviter | ✅ Bonne pratique | Raison |
|------------|------------------|--------|
| `\Drupal::service()` dans `initializeIterator()` | Injecter via `create()` | Non testable, couplage fort |
| Charger toutes les données en mémoire dans `initializeIterator()` | Utiliser `yield` (Generator) | OOM sur 100k+ lignes |
| Ignorer les exceptions HTTP dans la source | Logguer via `idMap->saveMessage()` | Les erreurs silencieuses causent des migrations incomplètes non détectées |
| Retourner `[]` dans `import()` au lieu des IDs | Toujours retourner les IDs de destination | Sans IDs, le rollback et le suivi sont impossibles |
| Logique métier complexe dans `prepareRow()` | Déplacer dans un Process plugin | `prepareRow()` est pour le nettoyage simple |
| Annotations `@MigrateSource` sur D11+ | `#[MigrateSource]` attribute PHP | Annotations dépréciées D11 |

---

## See Also

- `migrate-api.md` — Migrations YAML standard (CSV, SQL, D7)
- `advanced-patterns.md` — high_water, events, groupes, Paragraphs
- `drupal-core` — Plugin system, DI, services
- `drupal-testing` — KernelTests pour plugins Migrate
