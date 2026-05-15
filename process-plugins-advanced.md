# Process Plugins Avancés — Migrate API

Référence complète des plugins de traitement (process) pour les migrations Drupal complexes. Chaque plugin est accompagné d'un exemple YAML complet et d'un contexte d'usage réel.

---

## 1. `sub_process` — Traitement de champs multi-valeurs

Permet d'appliquer un pipeline de process sur chaque élément d'un champ répétable (paragraphs, entity references multiples, etc.).

```yaml
# Migrer un champ Paragraphs (entity_reference_revisions) depuis D7
process:
  field_sections:
    plugin: sub_process
    source: field_sections
    process:
      target_id:
        plugin: migration_lookup
        migration: d7_paragraphs_hero
        source: value
      target_revision_id:
        plugin: migration_lookup
        migration: d7_paragraphs_hero
        source: revision_id
```

```yaml
# Migrer des images multiples avec alt et title pour chaque item
process:
  field_images:
    plugin: sub_process
    source: field_images
    process:
      target_id:
        plugin: migration_lookup
        migration: d7_file_public
        source: fid
      alt: alt
      title: title
      width: width
      height: height
```

> **Quand l'utiliser :** champ avec `cardinality > 1` ou `-1` (illimité) où chaque valeur doit être transformée via un lookup ou une règle.

---

## 2. `iterator` — Itérer sur un tableau source

Similaire à `sub_process` mais conçu pour les cas où la source est un tableau plat (pas un tableau de sous-objets). Utile pour migrer des champs multi-valeurs simples.

```yaml
# Pour chaque image dans un champ multi-image plat
process:
  field_gallery:
    plugin: iterator
    source: field_images
    process:
      target_id:
        plugin: migration_lookup
        migration: d7_file_public
        source: fid
      alt: alt
      title: title
```

```yaml
# Itérer sur un tableau de références taxonomy
process:
  field_categories:
    plugin: iterator
    source: field_category_tids
    process:
      target_id:
        plugin: migration_lookup
        migration: d7_taxonomy_term
        source: tid
```

> **`sub_process` vs `iterator` :** `sub_process` préserve les clés du tableau source et est recommandé pour les entités embarquées. `iterator` est plus simple pour les valeurs scalaires répétées.

---

## 3. `flatten` — Aplatir un tableau imbriqué

Convertit un tableau imbriqué `[[val1], [val2]]` en tableau plat `[val1, val2]`. Utile quand la source renvoie des structures imbriquées inattendues.

```yaml
# Exemple : la source D7 renvoie [[tid1], [tid2]] au lieu de [tid1, tid2]
process:
  field_tags:
    -
      plugin: flatten
      source: field_tags
    -
      plugin: migration_lookup
      migration: d7_taxonomy_term
```

```yaml
# Aplatir puis appliquer un callback de nettoyage
process:
  field_roles:
    -
      plugin: flatten
      source: roles
    -
      plugin: skip_on_empty
      method: process
```

---

## 4. `merge` — Fusionner plusieurs sources en un seul champ

Combine les valeurs de plusieurs champs source dans un seul champ destination (tableau).

```yaml
# Combiner deux champs de tags en une seule référence
process:
  field_all_tags:
    plugin: merge
    source:
      - field_tags
      - field_categories
```

```yaml
# Fusionner des IDs de fichiers de plusieurs champs image
process:
  field_medias:
    plugin: merge
    source:
      - field_image_principale
      - field_images_secondaires
```

---

## 5. `concat` — Concaténer des champs

Assemble plusieurs valeurs en une seule chaîne avec un délimiteur.

```yaml
# Concaténer prénom + nom
process:
  field_nom_complet:
    plugin: concat
    source:
      - field_prenom
      - field_nom
    delimiter: ' '
```

```yaml
# Construire une URI interne depuis un chemin D7
process:
  link/uri:
    plugin: concat
    source:
      - constants/prefix
      - link_path
    delimiter: ''
  constants:
    prefix: 'internal:/'
```

```yaml
# Construire un alias depuis le titre
process:
  path/alias:
    -
      plugin: concat
      source:
        - constants/prefix
        - title
      delimiter: ''
    -
      plugin: callback
      callable: strtolower
    -
      plugin: callback
      callable: trim
  constants:
    prefix: '/article/'
```

---

## 6. `callback` — Appliquer une fonction PHP native

Applique une fonction PHP standard (`strtolower`, `trim`, `strip_tags`, `nl2br`, etc.) à la valeur en cours.

```yaml
# Nettoyer le titre
process:
  title:
    -
      plugin: get
      source: title_raw
    -
      plugin: callback
      callable: trim
    -
      plugin: callback
      callable: strip_tags

# Convertir en minuscules pour un alias
process:
  field_slug:
    -
      plugin: get
      source: title
    -
      plugin: callback
      callable: strtolower
```

> **Limitation :** `callback` ne supporte que les fonctions à **un argument**. Pour plusieurs arguments, créer un process plugin custom.

---

## 7. `skip_on_empty` — Ignorer un champ ou une ligne si vide

Deux comportements selon `method` :

```yaml
# method: process → ignorer uniquement CE champ (la row continue)
process:
  field_image/target_id:
    plugin: skip_on_empty
    method: process
    source: field_image_fid

# method: row → ignorer toute la ligne si la valeur est vide
process:
  nid:
    plugin: skip_on_empty
    method: row
    message: 'NID manquant — ligne ignorée'
    source: nid
```

```yaml
# Combinaison : skip si vide, puis lookup
process:
  field_auteur/target_id:
    -
      plugin: skip_on_empty
      method: process
      source: field_auteur_uid
    -
      plugin: migration_lookup
      migration: d7_user
```

---

## 8. `skip_row_if_not_set` — Ignorer une ligne si la clé source est absente

Différent de `skip_on_empty` : vérifie l'**existence** de la clé dans la source (pas seulement si elle est vide).

```yaml
process:
  nid:
    plugin: skip_row_if_not_set
    source: nid

  # Ignorer si le bundle source n'est pas 'article'
  type:
    plugin: skip_row_if_not_set
    source: type
    message: 'Type manquant — ligne ignorée'
```

---

## 9. `static_map` — Mapper des valeurs avec un dictionnaire

Remplace une valeur source par sa correspondance dans un tableau de mapping.

```yaml
# Mapper les formats de texte D7 → D10
process:
  'body/format':
    plugin: static_map
    source: body_format
    map:
      filtered_html: basic_html
      full_html: full_html
      plain_text: plain_text
      php_code: basic_html       # Obsolète en D10
    default_value: basic_html    # Si valeur non trouvée dans le map

# Mapper les statuts de publication
process:
  status:
    plugin: static_map
    source: node_status
    map:
      '1': 1
      '0': 0
    default_value: 0

# Mapper les codes de langue D7 → BCP 47
process:
  langcode:
    plugin: static_map
    source: language
    map:
      fr: fr
      en: en
      und: fr        # 'undefined' → français
      zxx: fr        # 'non applicable' → français
    default_value: fr
```

---

## 10. `default_value` — Valeur par défaut inconditionnelle

Définit une valeur fixe, quelle que soit la source.

```yaml
process:
  langcode:
    plugin: default_value
    default_value: fr

  status:
    plugin: default_value
    default_value: 1

  # Valeur par défaut pour un champ booléen
  promote:
    plugin: default_value
    default_value: 0
```

---

## 11. `migration_lookup` — Résoudre les IDs entre migrations

Traduit un ID source (D7) en ID destination (D10) en consultant la map table d'une migration précédente.

```yaml
# Référence simple vers une entité migrée
process:
  uid:
    plugin: migration_lookup
    migration: d7_user
    source: uid
    no_stub: true    # true = ne pas créer de stub si non trouvé (recommandé)

# Référence vers une migration parmi plusieurs possibles (polymorphique)
process:
  field_media/target_id:
    plugin: migration_lookup
    migration:
      - d7_file_public
      - d7_file_private
    source: fid
    no_stub: true
```

> **`no_stub: true` vs `no_stub: false` :**
> - `false` (défaut) : crée un "stub" (entité vide temporaire) si la source n'est pas encore migrée. Nécessite un second passage avec `--update`.
> - `true` : laisse le champ vide si la dépendance n'existe pas encore. Recommandé quand on contrôle l'ordre d'import.

---

## 12. `get` — Lire une valeur source (explicite)

Récupère explicitement une valeur source. Utile dans les pipelines multi-étapes.

```yaml
process:
  # Équivalent direct : title: title
  title:
    plugin: get
    source: title

  # Lire une constante définie dans le YAML
  langcode:
    plugin: get
    source: constants/default_langcode
  constants:
    default_langcode: fr
```

---

## 13. Custom Process Plugin — Avec état et logging

Exemple de plugin process custom qui trace les items migrés avec logging intégré.

```php
<?php
// web/modules/custom/mon_projet_migration/src/Plugin/migrate/process/CounterTracker.php

namespace Drupal\mon_projet_migration\Plugin\migrate\process;

use Drupal\migrate\MigrateExecutableInterface;
use Drupal\migrate\ProcessPluginBase;
use Drupal\migrate\Row;

/**
 * Plugin process qui loggue la progression tous les N items.
 *
 * @MigrateProcess("counter_tracker")
 */
class CounterTracker extends ProcessPluginBase {

  private static int $count = 0;

  /**
   * {@inheritdoc}
   */
  public function transform($value, MigrateExecutableInterface $migrate_executable, Row $row, string $destination_property): mixed {
    self::$count++;

    if (self::$count % 100 === 0) {
      \Drupal::logger('migration')->info(
        'Migration @id : @count items traités',
        ['@id' => $row->getSourceIdValues()['nid'] ?? '?', '@count' => self::$count]
      );
    }

    return $value;
  }

}
```

Usage dans le YAML :

```yaml
process:
  nid:
    -
      plugin: get
      source: nid
    -
      plugin: counter_tracker   # Log tous les 100 items
```

---

## 14. Custom Process Plugin — Transformation métier

Plugin pour transformer un chemin D7 en URI Drupal 10 valide.

```php
<?php
// web/modules/custom/mon_projet_migration/src/Plugin/migrate/process/D7PathToUri.php

namespace Drupal\mon_projet_migration\Plugin\migrate\process;

use Drupal\migrate\MigrateExecutableInterface;
use Drupal\migrate\ProcessPluginBase;
use Drupal\migrate\Row;

/**
 * Convertit un chemin D7 (ex: 'node/42') en URI interne D10 ('internal:/node/42').
 *
 * @MigrateProcess("d7_path_to_uri")
 */
class D7PathToUri extends ProcessPluginBase {

  public function transform($value, MigrateExecutableInterface $migrate_executable, Row $row, string $destination_property): mixed {
    if (empty($value)) {
      return 'internal:/';
    }

    // Chemin externe (commence par http:// ou https://)
    if (str_starts_with($value, 'http://') || str_starts_with($value, 'https://')) {
      return $value;
    }

    // Chemin d'ancre
    if (str_starts_with($value, '#')) {
      return 'internal:/' . $value;
    }

    // Chemin interne standard
    return 'internal:/' . ltrim($value, '/');
  }

}
```

Usage :

```yaml
process:
  link/uri:
    plugin: d7_path_to_uri
    source: link_path
```

---

## 15. Pipeline Complet — Exemple d'Article avec Tous les Plugins

```yaml
id: d7_node_article_complet
label: 'D7 Articles — Pipeline complet'
source:
  plugin: d7_node
  node_type: article

process:
  # --- Identifiants ---
  nid: nid
  vid: vid

  # --- Langue ---
  langcode:
    plugin: static_map
    source: language
    map:
      fr: fr
      en: en
      und: fr
    default_value: fr

  # --- Titre : nettoyer les espaces ---
  title:
    -
      plugin: get
      source: title
    -
      plugin: callback
      callable: trim

  # --- Auteur : skip si UID 0 (anonyme) ---
  uid:
    -
      plugin: skip_on_empty
      method: process
      source: uid
    -
      plugin: migration_lookup
      migration: d7_user
      no_stub: true

  # --- Statuts ---
  status: status
  created: created
  changed: changed
  promote: promote
  sticky: sticky

  # --- Corps avec mapping de format ---
  'body/value': body_value
  'body/summary': body_summary
  'body/format':
    plugin: static_map
    source: body_format
    map:
      filtered_html: basic_html
      full_html: full_html
      plain_text: plain_text
    default_value: basic_html

  # --- Image principale (skip si absente) ---
  'field_image/target_id':
    -
      plugin: skip_on_empty
      method: process
      source: field_image_fid
    -
      plugin: migration_lookup
      migration: d7_file_public
      no_stub: true
  'field_image/alt': field_image_alt
  'field_image/title': field_image_title

  # --- Tags (multi-valeurs avec sub_process) ---
  field_tags:
    plugin: sub_process
    source: field_tags
    process:
      target_id:
        plugin: migration_lookup
        migration: d7_taxonomy_term
        source: tid
        no_stub: true

  # --- Galerie d'images (iterator) ---
  field_galerie:
    plugin: iterator
    source: field_galerie
    process:
      target_id:
        plugin: migration_lookup
        migration: d7_file_public
        source: fid
      alt: alt

  # --- Compteur de progression ---
  nid_tracked:
    -
      plugin: get
      source: nid
    -
      plugin: counter_tracker

destination:
  plugin: entity:node
  default_bundle: article

migration_dependencies:
  required:
    - d7_user
    - d7_file_public
    - d7_taxonomy_term
```
