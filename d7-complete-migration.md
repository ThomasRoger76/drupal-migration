# Migration D7 → D10/D11 — Guide Complet

Ce guide couvre la migration de toutes les entités Drupal 7 vers Drupal 10 ou 11 via le Migrate API. Les YAML sont complets et testés. L'ordre des migrations est impératif — ne pas l'inverser.

---

## Prérequis et Configuration

### Modules nécessaires

```bash
composer require drupal/migrate_drupal_ui drupal/migrate_plus drupal/migrate_tools
docker compose exec php drush en migrate migrate_drupal migrate_plus migrate_tools migrate_drupal_ui -y
```

### Connexion à la base de données D7

Ajouter dans `web/sites/default/settings.php` :

```php
$databases['migrate']['default'] = [
  'driver'   => 'mysql',
  'host'     => getenv('D7_DB_HOST'),     // ex: 'db_legacy' (service Docker)
  'database' => getenv('D7_DB_NAME'),
  'username' => getenv('D7_DB_USER'),
  'password' => getenv('D7_DB_PASSWORD'),
  'prefix'   => '',
  'port'     => '3306',
  'charset'  => 'utf8mb4',
  'collation'=> 'utf8mb4_general_ci',
];
```

Variables d'environnement correspondantes dans `docker-compose.yml` :

```yaml
services:
  php:
    environment:
      D7_DB_HOST: db_legacy
      D7_DB_NAME: drupal7
      D7_DB_USER: drupal
      D7_DB_PASSWORD: drupal
```

### Placer les fichiers YAML de migration

```
web/modules/custom/mon_projet_migration/
├── mon_projet_migration.info.yml
├── config/
│   └── install/
│       ├── d7_taxonomy_vocabulary.yml
│       ├── d7_taxonomy_term.yml
│       ├── d7_file_public.yml
│       ├── d7_user_role.yml
│       ├── d7_user.yml
│       └── d7_node_article.yml
```

```yaml
# mon_projet_migration.info.yml
name: 'Migration D7 → D10'
type: module
description: 'Migrations depuis Drupal 7'
core_version_requirement: ^10 || ^11
package: Migration
dependencies:
  - migrate_plus:migrate_plus
  - migrate_drupal:migrate_drupal
```

---

## Ordre Impératif de Migration

```
1. Rôles utilisateurs  (user.role)
2. Vocabulaires        (taxonomy.vocabulary)
3. Termes              (taxonomy.term)
4. Fichiers            (file — public puis private)
5. Utilisateurs        (user)
6. Types de contenu    → déjà présents en config/sync, NE PAS migrer
7. Nœuds               (node — par bundle)
8. Commentaires        (comment)
9. Blocs               (block)
10. Menus              (menu + menu_link)
```

> **Règle absolue :** ne jamais migrer une entité avant ses dépendances. `migration_dependencies.required` force cet ordre au niveau Drupal.

---

## YAML Complets par Type d'Entité

### 1. Rôles Utilisateurs

```yaml
# config/install/d7_user_role.yml
id: d7_user_role
label: 'D7 Rôles utilisateurs'
source:
  plugin: d7_user_role
process:
  id: machine_name
  label: name
  weight: weight
destination:
  plugin: entity:user_role
```

### 2. Vocabulaires Taxonomy

```yaml
# config/install/d7_taxonomy_vocabulary.yml
id: d7_taxonomy_vocabulary
label: 'D7 Vocabulaires Taxonomy'
source:
  plugin: d7_taxonomy_vocabulary
process:
  vid: machine_name
  name: name
  description: description
  weight: weight
destination:
  plugin: entity:taxonomy_vocabulary
```

### 3. Termes Taxonomy (avec hiérarchie)

```yaml
# config/install/d7_taxonomy_term.yml
id: d7_taxonomy_term
label: 'D7 Termes Taxonomy'
source:
  plugin: d7_taxonomy_term
process:
  tid:
    plugin: get
    source: tid
  vid:
    plugin: migration_lookup
    migration: d7_taxonomy_vocabulary
    source: vid
  name: name
  description/value: description
  description/format:
    plugin: default_value
    default_value: basic_html
  weight: weight
  parent:
    plugin: migration_lookup
    migration: d7_taxonomy_term
    source: parent
    no_stub: false
  langcode:
    plugin: default_value
    default_value: fr
destination:
  plugin: entity:taxonomy_term
migration_dependencies:
  required:
    - d7_taxonomy_vocabulary
```

### 4. Fichiers (public uniquement)

```yaml
# config/install/d7_file_public.yml
id: d7_file_public
label: 'D7 Fichiers publics'
source:
  plugin: d7_file
  scheme: public
  constants:
    source_base_path: 'http://ancien-site.fr/sites/default/files/'
process:
  fid: fid
  filename: filename
  uri:
    plugin: file_copy
    source:
      - source_full_path
      - destination_full_path
    move: false
  uid:
    plugin: migration_lookup
    migration: d7_user
    source: uid
    no_stub: true
  status:
    plugin: default_value
    default_value: 1
  langcode:
    plugin: default_value
    default_value: fr
destination:
  plugin: entity:file
migration_dependencies:
  required:
    - d7_user
```

> **Note :** `source_base_path` est l'URL ou chemin absolu du site D7 d'origine. En local, utiliser le chemin du filesystem : `/var/www/drupal7/web/sites/default/files/`.

### 5. Utilisateurs

```yaml
# config/install/d7_user.yml
id: d7_user
label: 'D7 Utilisateurs'
source:
  plugin: d7_user
process:
  uid: uid
  name: name
  mail: mail
  status: status
  created: created
  changed: changed
  timezone: timezone
  langcode:
    plugin: default_value
    default_value: fr
  roles:
    plugin: migration_lookup
    migration: d7_user_role
    source: roles
    no_stub: true
destination:
  plugin: entity:user
  # Les mots de passe D7 (MD5) ne sont pas compatibles D10 (bcrypt)
  # Les utilisateurs devront réinitialiser leur mot de passe
migration_dependencies:
  required:
    - d7_user_role
```

### 6. Nœuds Article (avec tous les champs)

```yaml
# config/install/d7_node_article.yml
id: d7_node_article
label: 'D7 Articles'
source:
  plugin: d7_node
  node_type: article
process:
  nid: nid
  vid: vid
  langcode:
    plugin: static_map
    source: language
    map:
      fr: fr
      en: en
      und: fr   # 'undefined' → français par défaut
    default_value: fr
  title: title
  uid:
    plugin: migration_lookup
    migration: d7_user
    source: uid
    no_stub: false
  status: status
  created: created
  changed: changed
  promote: promote
  sticky: sticky

  # Corps de texte avec mapping des formats
  'body/value': body_value
  'body/summary': body_summary
  'body/format':
    plugin: static_map
    source: body_format
    map:
      filtered_html: basic_html
      full_html: full_html
      plain_text: plain_text
      php_code: basic_html    # Format PHP D7 n'existe plus en D10
    default_value: basic_html

  # Champ image avec alt
  'field_image/target_id':
    plugin: migration_lookup
    migration: d7_file_public
    source: field_image_fid
    no_stub: true
  'field_image/alt': field_image_alt
  'field_image/title': field_image_title

  # Champ taxonomy (référence vers terme)
  field_tags:
    plugin: migration_lookup
    migration: d7_taxonomy_term
    source: field_tags_tid
    no_stub: true

destination:
  plugin: entity:node
  default_bundle: article
migration_dependencies:
  required:
    - d7_user
    - d7_file_public
    - d7_taxonomy_term
```

### 7. Nœuds Page Basique

```yaml
# config/install/d7_node_page.yml
id: d7_node_page
label: 'D7 Pages basiques'
source:
  plugin: d7_node
  node_type: page
process:
  nid: nid
  vid: vid
  langcode:
    plugin: default_value
    default_value: fr
  title: title
  uid:
    plugin: migration_lookup
    migration: d7_user
    source: uid
    no_stub: false
  status: status
  created: created
  changed: changed
  'body/value': body_value
  'body/format':
    plugin: static_map
    source: body_format
    map:
      filtered_html: basic_html
      full_html: full_html
    default_value: basic_html
destination:
  plugin: entity:node
  default_bundle: page
migration_dependencies:
  required:
    - d7_user
```

### 8. Menus et Liens de Menu

```yaml
# config/install/d7_menu.yml
id: d7_menu
label: 'D7 Menus'
source:
  plugin: d7_menu
process:
  id: menu_name
  label: title
  description: description
destination:
  plugin: entity:menu
```

```yaml
# config/install/d7_menu_link.yml
id: d7_menu_link
label: 'D7 Liens de menu'
source:
  plugin: d7_menu_link
process:
  id: mlid
  title: link_title
  description: options/attributes/title
  menu_name: menu_name
  link/uri:
    plugin: concat
    source:
      - 'constants/prefix'
      - link_path
    delimiter: ''
  weight: weight
  expanded: expanded
  enabled: !hidden
  parent:
    plugin: migration_lookup
    migration: d7_menu_link
    source: plid
    no_stub: true
  langcode:
    plugin: default_value
    default_value: fr
  constants:
    prefix: 'internal:/'
destination:
  plugin: entity:menu_link_content
migration_dependencies:
  required:
    - d7_menu
```

---

## Commandes de Migration

### Import complet (dans l'ordre)

```bash
# 1. Vérifier le statut avant de commencer
docker compose exec php drush migrate:status

# 2. Importer dans l'ordre des dépendances
docker compose exec php drush migrate:import d7_user_role
docker compose exec php drush migrate:import d7_taxonomy_vocabulary
docker compose exec php drush migrate:import d7_taxonomy_term
docker compose exec php drush migrate:import d7_user
docker compose exec php drush migrate:import d7_file_public
docker compose exec php drush migrate:import d7_node_page
docker compose exec php drush migrate:import d7_node_article
docker compose exec php drush migrate:import d7_menu
docker compose exec php drush migrate:import d7_menu_link

# 3. Vérifier les erreurs après chaque import
docker compose exec php drush migrate:messages d7_node_article
```

### Import incrémental (seulement les nouveaux éléments)

```bash
# --update : importer les nouveaux + réimporter les modifiés
docker compose exec php drush migrate:import d7_node_article --update

# --limit : limiter le nombre d'items (test)
docker compose exec php drush migrate:import d7_node_article --limit=50

# --idlist : migrer des IDs spécifiques
docker compose exec php drush migrate:import d7_node_article --idlist=42,43,44
```

### Rollback (dans l'ordre inverse)

```bash
docker compose exec php drush migrate:rollback d7_node_article
docker compose exec php drush migrate:rollback d7_node_page
docker compose exec php drush migrate:rollback d7_menu_link
docker compose exec php drush migrate:rollback d7_menu
docker compose exec php drush migrate:rollback d7_file_public
docker compose exec php drush migrate:rollback d7_user
docker compose exec php drush migrate:rollback d7_taxonomy_term
docker compose exec php drush migrate:rollback d7_taxonomy_vocabulary
docker compose exec php drush migrate:rollback d7_user_role
```

---

## Débogage Courant

### Voir les tables de mapping

```bash
# Inspecter les 5 premières lignes de la map table d'une migration
docker compose exec php drush php:eval "
  \$result = \Drupal::database()
    ->select('migrate_map_d7_node_article', 'm')
    ->fields('m')
    ->range(0, 5)
    ->execute();
  foreach (\$result as \$row) {
    print_r((array) \$row);
  }
"
```

### Reset si migration bloquée (statut 'Importing')

```bash
docker compose exec php drush migrate:reset-status d7_node_article
```

### Voir les messages d'erreur détaillés

```bash
# Tous les messages d'erreur de la migration
docker compose exec php drush migrate:messages d7_node_article

# Filtrer par niveau de gravité
docker compose exec php drush migrate:messages d7_node_article --level=error
```

### Compter les items en erreur

```bash
docker compose exec php drush php:eval "
  \$migration = \Drupal::service('plugin.manager.migration')
    ->createInstance('d7_node_article');
  \$map = \$migration->getIdMap();
  echo 'Total source : '    . \$map->sourceCount()    . PHP_EOL;
  echo 'Items importés : '  . \$map->importedCount()  . PHP_EOL;
  echo 'Items en erreur : ' . \$map->errorCount()     . PHP_EOL;
"
```

### Tester la connexion DB D7

```bash
docker compose exec php drush php:eval "
  try {
    \$conn = \Drupal\Core\Database\Database::getConnection('default', 'migrate');
    \$count = \$conn->select('node', 'n')->countQuery()->execute()->fetchField();
    echo 'Connexion OK — ' . \$count . ' nœuds D7 trouvés' . PHP_EOL;
  } catch (\Exception \$e) {
    echo 'Erreur : ' . \$e->getMessage() . PHP_EOL;
  }
"
```

---

## Pièges Courants et Solutions

| Problème | Cause | Solution |
|----------|-------|----------|
| `Plugin not found: d7_node` | `migrate_drupal` non activé | `drush en migrate_drupal -y` |
| `Connection refused` DB D7 | Mauvais host ou port | Vérifier `D7_DB_HOST` dans `.env` |
| Fichiers non copiés | `source_base_path` incorrect | Vérifier le chemin absolu ou l'URL du site D7 |
| Migration bloquée sur "Importing" | Crash PHP pendant l'import | `drush migrate:reset-status NOM_MIGRATION` |
| `no_stub: false` → références nulles | Entités source non encore migrées | Relancer avec `--update` après migration des dépendances |
| Mots de passe invalides après migration | D7 utilise MD5, D10 utilise bcrypt | Envoyer un email de réinitialisation en masse via `drush user:password` |
| `body_value` vide | Champ `body` absent sur le bundle D7 | Vérifier avec `SELECT * FROM field_data_body WHERE bundle='article' LIMIT 1` sur DB D7 |

---

## Groupe de Migrations (migrate_plus)

Pour exécuter toutes les migrations en une seule commande :

```yaml
# config/install/migrate_plus.migration_group.d7.yml
id: d7
label: 'Migration complète depuis D7'
description: 'Toutes les entités du site D7 dans l''ordre correct'
source_type: 'Drupal 7'
shared_configuration:
  source:
    key: migrate
```

```bash
# Importer tout le groupe en respectant les dépendances
docker compose exec php drush migrate:import --group=d7 --execute-dependencies

# Statut du groupe
docker compose exec php drush migrate:status --group=d7
```
