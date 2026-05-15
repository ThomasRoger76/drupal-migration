# Migrate API — Patterns Avancés

## 1. Migration Paragraphs (D7 field_collection → D10/D11 Paragraphs)

### Prérequis

```bash
composer require drupal/paragraphs drupal/entity_reference_revisions
docker compose exec php drush en paragraphs entity_reference_revisions -y
```

### Structure : deux migrations liées

```yaml
# config/install/migrate_plus.migration.import_paragraphs_contenu.yml
id: import_paragraphs_contenu
label: 'Import field_collection → Paragraphs'

source:
  plugin: d7_field_collection
  field_collection_type: field_sections  # Le nom du field_collection D7

process:
  # Champs du Paragraph
  'field_titre/value': titre
  'field_texte/value': texte
  'field_texte/format':
    plugin: default_value
    default_value: full_html

destination:
  plugin: 'entity_reference_revisions:paragraph'
  default_bundle: section_texte

migration_dependencies: {}
```

```yaml
# config/install/migrate_plus.migration.import_articles.yml
id: import_articles
label: 'Import articles avec Paragraphs'

source:
  plugin: d7_node
  node_type: article

process:
  title: title
  uid:
    plugin: migration_lookup
    migration: import_users
    source: uid
  # Référencer les Paragraphs migrés
  field_sections:
    plugin: migration_lookup
    migration: import_paragraphs_contenu
    source: field_sections
    no_stub: true

destination:
  plugin: 'entity:node'
  default_bundle: article

migration_dependencies:
  required:
    - import_paragraphs_contenu
    - import_users
```

### Format de destination pour entity_reference_revisions

```yaml
process:
  field_sections:
    plugin: sub_process
    source: field_sections
    process:
      target_id:
        plugin: migration_lookup
        migration: import_paragraphs_contenu
        source: value
      target_revision_id:
        plugin: migration_lookup
        migration: import_paragraphs_contenu
        source: revision_id
```

---

## 2. high_water_property — Migration Incrémentale

Permet de n'importer que les **nouvelles lignes** ou les lignes **modifiées depuis le dernier import**.

### Via CSV avec timestamp

```yaml
source:
  plugin: csv
  path: '/tmp/exports/articles.csv'
  ids: [id]
  # Champ utilisé comme "borne haute" — seules les lignes > dernière valeur sont importées
  high_water_property:
    name: updated_at  # Nom de la colonne CSV
    alias: u          # Alias SQL (requis pour certains plugins)
```

### Via SQL source

```yaml
source:
  plugin: d7_node
  node_type: article
  high_water_property:
    name: changed    # Champ timestamp Drupal
    alias: n
```

### Vérifier et réinitialiser le high water mark

```bash
# Voir la valeur actuelle
docker compose exec php drush php-eval "
  \$migration = \Drupal::service('plugin.manager.migration')
    ->createInstance('import_articles');
  echo 'High water: ' . \$migration->getHighWater() . PHP_EOL;
"

# Réinitialiser pour forcer un re-import complet
docker compose exec php drush php-eval "
  \$migration = \Drupal::service('plugin.manager.migration')
    ->createInstance('import_articles');
  \$migration->saveHighWater(0);
"

# Puis relancer l'import
docker compose exec php drush migrate:import import_articles
```

---

## 3. MigrateEvents — Monitoring et Logging en Production

```php
// src/EventSubscriber/MigrationMonitor.php
namespace Drupal\mon_module\EventSubscriber;

use Drupal\migrate\Event\MigrateEvents;
use Drupal\migrate\Event\MigrateImportEvent;
use Drupal\migrate\Event\MigrateIdMapMessageEvent;
use Drupal\migrate\Event\MigratePostRowSaveEvent;
use Drupal\migrate\Plugin\MigrationInterface;
use Psr\Log\LoggerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

/**
 * Monitore les migrations et alerte sur les erreurs critiques.
 */
class MigrationMonitor implements EventSubscriberInterface {

  public function __construct(
    private readonly LoggerInterface $logger,
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [
      MigrateEvents::POST_ROW_SAVE   => ['onPostRowSave', 0],
      MigrateEvents::POST_IMPORT     => ['onPostImport', 0],
      MigrateEvents::IDMAP_MESSAGE   => ['onIdMapMessage', 0],
    ];
  }

  /**
   * Logguer chaque ligne importée avec succès (utile en debug, désactiver en prod).
   */
  public function onPostRowSave(MigratePostRowSaveEvent $event): void {
    // Éviter de loguer trop en production — utiliser uniquement en debug
    if (\Drupal::state()->get('mon_module.migration_debug', FALSE)) {
      $ids = implode(',', $event->getDestinationIdValues());
      $this->logger->debug('Migration @id — ligne importée : @dest_ids', [
        '@id'       => $event->getMigration()->id(),
        '@dest_ids' => $ids,
      ]);
    }
  }

  /**
   * Résumé de fin de migration.
   */
  public function onPostImport(MigrateImportEvent $event): void {
    $migration = $event->getMigration();
    $map       = $migration->getIdMap();

    $this->logger->info(
      'Migration @id terminée — importées : @imported, ignorées : @ignored, erreurs : @errors',
      [
        '@id'       => $migration->id(),
        '@imported' => $map->importedCount(),
        '@ignored'  => $map->ignoredCount(),
        '@errors'   => $map->errorCount(),
      ]
    );

    // Alerte si trop d'erreurs
    $error_rate = $map->errorCount() / max($map->processedCount(), 1);
    if ($error_rate > 0.05) { // Plus de 5% d'erreurs
      $this->logger->error(
        'Migration @id : taux d\'erreur élevé (@rate%), vérification requise.',
        ['@id' => $migration->id(), '@rate' => round($error_rate * 100, 1)]
      );
    }
  }

  /**
   * Logger les messages d'erreur de la table de mapping.
   */
  public function onIdMapMessage(MigrateIdMapMessageEvent $event): void {
    $level = match ($event->getLevel()) {
      MigrationInterface::MESSAGE_ERROR       => 'error',
      MigrationInterface::MESSAGE_WARNING     => 'warning',
      MigrationInterface::MESSAGE_NOTICE      => 'notice',
      MigrationInterface::MESSAGE_INFORMATIONAL => 'info',
      default                                 => 'debug',
    };

    $this->logger->log($level, 'Migration @id [@source_ids] : @message', [
      '@id'         => $event->getMigration()->id(),
      '@source_ids' => implode(',', $event->getSourceIdValues()),
      '@message'    => $event->getMessage(),
    ]);
  }

}
```

### Enregistrer le subscriber dans services.yml

```yaml
# mon_module.services.yml
services:
  mon_module.migration_monitor:
    class: Drupal\mon_module\EventSubscriber\MigrationMonitor
    arguments:
      - '@logger.channel.mon_module'
    tags:
      - { name: event_subscriber }
```

### Consulter les messages d'erreur via Drush

```bash
# Voir les erreurs de la migration
docker compose exec php drush migrate:messages import_articles

# Format tabulaire
docker compose exec php drush migrate:messages import_articles --format=table
```

---

## 4. Migration Groups et Dépendances

Les groupes permettent de lancer plusieurs migrations dans le bon ordre avec une seule commande.

### Définir un groupe

```yaml
# config/install/migrate_plus.migration_group.mon_import.yml
id: mon_import
label: 'Import complet Mon Projet'
description: 'Migrations groupées dans le bon ordre'
source_type: 'API externe v2'
shared_configuration:
  source:
    # Configuration partagée entre toutes les migrations du groupe
    key: 'default'
```

### Assigner des migrations à un groupe

```yaml
# Dans chaque fichier de migration
migration_group: mon_import
migration_dependencies:
  required:
    - import_taxonomies
    - import_medias
  optional:
    - import_users_secondaires
```

### Lancer toutes les migrations d'un groupe

```bash
# Import dans l'ordre des dépendances
docker compose exec php drush migrate:import --group=mon_import

# Status de toutes les migrations du groupe
docker compose exec php drush migrate:status --group=mon_import

# Rollback du groupe entier (ordre inverse)
docker compose exec php drush migrate:rollback --group=mon_import
```

### Ordre automatique via dépendances

```yaml
# import_taxonomies → import_medias → import_articles
# migrate:import résout l'ordre automatiquement

# config/install/migrate_plus.migration.import_medias.yml
id: import_medias
migration_group: mon_import
migration_dependencies:
  required:
    - import_taxonomies  # Les médias référencent des taxonomies

# config/install/migrate_plus.migration.import_articles.yml
id: import_articles
migration_group: mon_import
migration_dependencies:
  required:
    - import_taxonomies
    - import_medias  # Les articles incluent des médias et des tags
```

---

## 5. Rollback avec Nettoyage Fichiers

```bash
# Rollback partiel par IDs
docker compose exec php drush migrate:rollback import_articles --idlist=1,2,3

# Rollback complet
docker compose exec php drush migrate:rollback import_articles

# ATTENTION : les fichiers Media uploadés NE sont PAS supprimés automatiquement
# Nettoyage manuel nécessaire :
docker compose exec php drush php-eval "
  // Supprimer les fichiers Media importés par la migration
  \$mids = \\Drupal::entityQuery('media')
    ->condition('field_source_migration', 'import_medias')
    ->accessCheck(FALSE)
    ->execute();
  foreach (\\Drupal\\media\\Entity\\Media::loadMultiple(\$mids) as \$media) {
    \$file = \$media->field_media_image->entity;
    \$media->delete();
    if (\$file && empty(\\Drupal::service('file.usage')->listUsage(\$file))) {
      \$file->delete();
    }
  }
  echo 'Nettoyage terminé.';
"
```

---

## 6. Performance — Batch size et mémoire

### Configuration via YAML

```yaml
source:
  plugin: csv
  path: '/tmp/exports/gros-fichier.csv'
  ids: [id]
  # Traiter par lots de 100 lignes
  batch_size: 100
```

### Lancer avec une limite mémoire augmentée

```bash
# Via Docker Compose
docker compose exec php php -d memory_limit=512M \
  vendor/bin/drush migrate:import import_articles --batch-size=500

# Ou configurer dans php.ini du container
docker compose exec php drush migrate:import import_articles \
  --batch-size=200 \
  --feedback=50  # Afficher le statut toutes les 50 lignes
```

### Optimisations SQL pour grandes migrations

```yaml
source:
  plugin: sqlsf  # Source SQL avec support des jointures
  query:
    table: node
    fields:
      - nid
      - title
      - body_value
    condition:
      status: 1
    # Index hint pour guider l'optimiseur MySQL
    join:
      table: field_data_body
      alias: b
      condition: 'b.entity_id = node.nid AND b.entity_type = :type'
      placeholders:
        ':type': node
```

```bash
# Vider le cache Drupal entre les batches (libère la mémoire statique)
docker compose exec php drush migrate:import import_articles \
  --batch-size=100 \
  && docker compose exec php drush cr
```

---

## 7. Stub Handling — Migrations circulaires

Quand A référence B et B référence A, `migration_lookup` crée des stubs temporaires.

```yaml
process:
  field_auteur:
    plugin: migration_lookup
    migration: import_users
    source: author_id
    # Créer un nœud stub si l'utilisateur n'est pas encore migré
    stub_id: import_users
    no_stub: false  # true = ignorer si non trouvé (pas de stub)
```

```bash
# Après migration initiale, synchroniser les stubs
docker compose exec php drush migrate:import import_articles --update
docker compose exec php drush migrate:import import_users --update
```

---

## 8. Debug avancé

```bash
# Voir le statut détaillé
docker compose exec php drush migrate:status import_articles --format=table

# Mode verbose — afficher chaque ligne traitée
docker compose exec php drush migrate:import import_articles \
  --batch-size=10 \
  2>&1 | head -100

# Forcer le re-import d'une ligne spécifique
docker compose exec php drush migrate:import import_articles \
  --idlist=42 \
  --update

# Réinitialiser une migration bloquée en "Importing"
docker compose exec php drush migrate:reset-status import_articles
```

---

## Anti-patterns patterns avancés

| ❌ À éviter | ✅ Bonne pratique | Raison |
|------------|------------------|--------|
| Lancer `migrate:import` sans `--group` en production | Utiliser les groupes + dépendances | Risque de migrer dans le mauvais ordre |
| `high_water` sur un champ non trié en DB | Vérifier l'index sur le champ | Résultats non déterministes |
| EventSubscriber qui envoie des emails sur `POST_ROW_SAVE` | Buffer + envoyer en `POST_IMPORT` | 10k emails envoyés lors de l'import |
| Rollback sans backup DB préalable | Toujours backup avant rollback | Le rollback des entités supprime les révisions |
| `batch_size` trop élevé sans augmenter la mémoire PHP | Calibrer les deux ensemble | OOM fatal = migration corrompue |
| Ignorer les stubs après migration | Toujours `--update` pour les migrations circulaires | Les stubs laissent des nœuds vides en production |

---

---

## 9. D7 → D10/11 — Migration Complète Multi-Entités

### Ordre de migration obligatoire

Les dépendances entre entités imposent un ordre strict. Tout import lancé dans le mauvais ordre crée des stubs ou des erreurs FK silencieuses.

```yaml
# Ordre de migration D7→D10 — les dépendances DOIVENT être respectées
# Groupe 1 : Config (bundles, champs) déjà déployée via drush cim
# Groupe 2 : Taxonomies (référencées par les nodes)
# Groupe 3 : Fichiers et Media (uploadés, référencés)
# Groupe 4 : Utilisateurs (propriétaires des nodes)
# Groupe 5 : Paragraphs
# Groupe 6 : Nodes (dépend de tout le reste)
```

### Exemple complet : migration taxonomy_term D7 → D10

```yaml
# config/install/migrate_plus.migration.d7_taxonomy_term_tags.yml
id: d7_taxonomy_term_tags
label: 'Tags D7 → D10'
source:
  plugin: d7_taxonomy_term
  bundle: tags
process:
  tid:
    plugin: ignore_stub  # Ignorer les stubs créés par migration_lookup
    source: tid
  vid:
    plugin: default_value
    default_value: tags
  name: name
  description/value: description
  langcode:
    plugin: default_value
    default_value: fr
destination:
  plugin: entity:taxonomy_term
  default_bundle: tags
migration_dependencies:
  required: []
```

```bash
# Lancer dans l'ordre avec le groupe
docker compose exec php drush migrate:import --group=d7_content

# Vérifier les stubs non résolus après migration
docker compose exec php drush php:eval "
  \$counts = \Drupal::database()->query(
    'SELECT migrate_map_d7_taxonomy_term_tags.source_row_status, COUNT(*) as n
     FROM migrate_map_d7_taxonomy_term_tags
     GROUP BY source_row_status'
  )->fetchAll();
  foreach (\$counts as \$row) { echo \$row->source_row_status . ': ' . \$row->n . PHP_EOL; }
"
```

---

## 10. Stub Handling — Références Circulaires

Les stubs sont des entités temporaires créées automatiquement par `migration_lookup` quand une entité référencée n'existe pas encore au moment de la migration.

```yaml
# Exemple : article référence un utilisateur pas encore migré
process:
  uid:
    plugin: migration_lookup
    migration: import_users
    source: author_id
    no_stub: false  # false = créer un stub user temporaire (défaut)
    # no_stub: true = ignorer la référence si user non trouvé (pas de stub)
```

### Résoudre les stubs après migration complète

```bash
# Étape 1 : migrer les entités sources (users)
docker compose exec php drush migrate:import import_users

# Étape 2 : re-passer les migrations avec stubs pour les résoudre
docker compose exec php drush migrate:import import_articles --update

# Vérifier qu'il ne reste pas de stubs non résolus
docker compose exec php drush php:eval "
  \$stubs = \Drupal::database()->query(
    'SELECT COUNT(*) FROM migrate_map_import_articles WHERE source_row_status = 2'
  )->fetchField();
  echo 'Stubs restants : ' . \$stubs . PHP_EOL;
"
# source_row_status = 2 → STUB (non résolu)
# source_row_status = 0 → IMPORTED (succès)
# source_row_status = 1 → NEEDS_UPDATE
```

---

## 11. Rollback avec Media/Files

```bash
# Rollback standard — supprime les entités Drupal mais PAS les fichiers disque
docker compose exec php drush migrate:rollback import_articles

# Les fichiers restent sur le disque — nettoyage des médias orphelins
docker compose exec php drush php:eval "
  // Supprimer les médias Media orphelins après rollback (status=0 ou sans usage)
  \$mids = \Drupal::entityQuery('media')
    ->accessCheck(FALSE)
    ->condition('status', 0)
    ->execute();
  foreach (\Drupal\media\Entity\Media::loadMultiple(\$mids) as \$media) {
    \$media->delete();
  }
  echo count(\$mids) . ' médias supprimés.' . PHP_EOL;
"

# Nettoyage plus agressif : fichiers non référencés par aucune entité
docker compose exec php drush php:eval "
  // Fichiers managed sans usage (file_usage vide)
  \$fids = \Drupal::entityQuery('file')
    ->accessCheck(FALSE)
    ->condition('status', 1)
    ->execute();
  \$file_usage = \Drupal::service('file.usage');
  \$deleted = 0;
  foreach (\Drupal\file\Entity\File::loadMultiple(\$fids) as \$file) {
    if (empty(\$file_usage->listUsage(\$file))) {
      \$file->delete();
      \$deleted++;
    }
  }
  echo \$deleted . ' fichiers orphelins supprimés.' . PHP_EOL;
"
```

> **Anti-pattern :** rollback sans backup DB préalable. Les révisions supprimées lors du rollback sont **irrécupérables** sans backup.

---

## See Also

- `migrate-api.md` — Migrations YAML standard (CSV, SQL, D7)
- `custom-plugins.md` — Source, Process, Destination plugins custom
- `multilingual-migration.md` — Migrations multilingues, langcode, translations
- `drupal-core` — Entity API, EntityQuery, Paragraphs
- `drupal-docker` — Environnement de migration, mémoire PHP
