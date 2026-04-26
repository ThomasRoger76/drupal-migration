---
name: drupal-migration-test-runner
description: Runs pre and post-migration tests for Drupal. Creates content baseline via Drush/API before migration, then validates via Playwright browser tests after. Compares results and updates the migration report with pass/fail counts.
---

# Drupal Migration Test Runner

You are the sixth agent in the drupal-migration pipeline. You run in two phases:
- **PHASE: baseline** — called BEFORE the update to create test content and snapshot URLs
- **PHASE: validate** — called AFTER the update to verify everything still works

The orchestrator tells you which phase via the `phase` input.

## Prerequisites

Read `.drupal-migration/environment.json`. `CMD` = `environment.json > command_prefix`.

## Inputs

- `phase`: `baseline` | `validate`
- `report_file`: path to MIGRATION-REPORT.md (validate phase only)

---

## PHASE: baseline

### Step B1 — Discover content types

```bash
${CMD} drush php:eval "foreach (\Drupal::entityTypeManager()->getStorage('node_type')->loadMultiple() as \$type) { echo \$type->id() . '\n'; }" 2>/dev/null
```

Limit to max 20 content types.

### Step B2 — Create test nodes (one per content type)

For each content type:

```bash
${CMD} drush php:eval "
\$node = \Drupal\node\Entity\Node::create([
  'type' => 'CONTENT_TYPE',
  'title' => '[MIGRATION-TEST] CONTENT_TYPE test node',
  'status' => 1,
]);
\$node->save();
echo 'Created: ' . \$node->id() . ' - ' . \$node->toUrl()->toString() . '\n';
" 2>/dev/null
```

**If `environment.json > special_modules.paragraphs = true`:**

First, discover all paragraph types and which content types accept them:
```bash
${CMD} drush php:eval "
\$para_types = \\Drupal::entityTypeManager()->getStorage('paragraphs_type')->loadMultiple();
foreach (\$para_types as \$t) { echo \$t->id() . '|' . \$t->label() . PHP_EOL; }
" 2>/dev/null
```

For each content type that has `entity_reference_revisions` fields (paragraph fields), create one fully-filled paragraph of each available type and attach it to the test node.

**Goal: create a real reference page, not an empty skeleton.** Every field of every paragraph must be filled with realistic data — lorem ipsum text, a real image pulled from existing site files, real link URLs, etc.

**Two mandatory pre-conditions before creating any content:**

**A) Create or find the "AgentIA" test user** — all test content must be owned by this dedicated user, NOT anonymous:
```bash
${CMD} drush php:eval "
\$users = \\Drupal::entityQuery('user')->condition('name','AgentIA')->accessCheck(FALSE)->execute();
if (empty(\$users)) {
  \$u = \\Drupal\\user\\Entity\\User::create(['name'=>'AgentIA','mail'=>'agent-ia@migration.test','status'=>1,'pass'=>bin2hex(random_bytes(16))]);
  \$u->save();
  echo 'AgentIA created uid=' . \$u->id() . PHP_EOL;
} else {
  echo 'AgentIA exists uid=' . reset(\$users) . PHP_EOL;
}
" 2>/dev/null
```
Store the uid as `$AGENT_UID`. Pass `'uid' => $AGENT_UID` on every `Node::create()` and `Paragraph::create()` call.

**B) Get a usable image media** — ALWAYS create a dedicated GD test image to guarantee visibility. Local environments often have no physical files. A media entity referencing a missing file renders as a broken image.

**Critical:** Do NOT use existing media entities unless you verify their file exists on disk. Always create a fresh test image.

```bash
# Step 1 — Create the image file with GD (run from Drupal docroot)
${CMD} drush php:eval "
\$docroot = \\Drupal::root();
\$dir = \$docroot . '/sites/default/files/migration-test/';
if (!is_dir(\$dir)) mkdir(\$dir, 0755, true);
\$filepath = \$dir . 'migration-test-800x500.jpg';
if (!file_exists(\$filepath)) {
  \$img = imagecreatetruecolor(800, 500);
  \$bg  = imagecolorallocate(\$img, 14, 93, 163);
  \$fg  = imagecolorallocate(\$img, 255, 255, 255);
  \$acc = imagecolorallocate(\$img, 230, 100, 30);
  imagefill(\$img, 0, 0, \$bg);
  imagefilledrectangle(\$img, 0, 420, 800, 500, imagecolorallocate(\$img, 220, 230, 240));
  imagefilledrectangle(\$img, 0, 415, 800, 420, \$acc);
  \$txt = 'IMAGE DE TEST - MIGRATION DRUPAL';
  imagestring(\$img, 5, (800 - strlen(\$txt)*imagefontwidth(5))/2, 220, \$txt, \$fg);
  imagejpeg(\$img, \$filepath, 90);
  imagedestroy(\$img);
}
echo file_exists(\$filepath) ? 'OK:' . \$filepath : 'FAIL' . PHP_EOL;
" 2>/dev/null

# Step 2 — Create File + Media entities pointing to that real file
${CMD} drush php:eval "
\$uri = 'public://migration-test/migration-test-800x500.jpg';
// Check if media already exists
\$existing = \\Drupal::entityQuery('media')->condition('name','[MIGRATION-TEST] Image référence')->accessCheck(FALSE)->execute();
if (\$existing) { echo 'MID=' . reset(\$existing) . PHP_EOL; return; }
\$file = \\Drupal\\file\\Entity\\File::create(['uri'=>\$uri,'status'=>1,'filename'=>'migration-test-800x500.jpg','filemime'=>'image/jpeg']);
\$file->save();
\$media = \\Drupal\\media\\Entity\\Media::create([
  'bundle' => 'images',
  'name'   => '[MIGRATION-TEST] Image référence',
  'status' => 1,
  'field_media_image' => ['target_id'=>\$file->id(),'alt'=>'Image test migration','title'=>'Migration test','width'=>800,'height'=>500],
]);
\$media->save();
echo 'MID=' . \$media->id() . PHP_EOL;
" 2>/dev/null
```
Store the returned `MID` as `$TEST_MID`.

**Important — paragraphs do NOT have a `uid` field.** Only `Node::create()` takes `'uid' => $AGENT_UID`. Never pass `uid` to `Paragraph::create()`.

**Important — `field_image` on paragraphs targets `media` entities (not files directly).** Always pass `['target_id' => $TEST_MID]`.

Use this comprehensive approach, filling ALL fields of every paragraph:

```bash
${CMD} drush php:eval "
// ---- helpers ----
function migtest_lorem(\$words = 10) {
  \$pool = 'Lorem ipsum dolor sit amet consectetur adipiscing elit sed do eiusmod tempor incididunt ut labore et dolore magna aliqua Ut enim ad minim veniam quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur';
  \$pool_arr = explode(' ', \$pool);
  \$out = [];
  for (\$i = 0; \$i < \$words; \$i++) \$out[] = \$pool_arr[\$i % count(\$pool_arr)];
  return implode(' ', \$out);
}

function migtest_get_image_fid() {
  // Reuse any existing managed image from the site
  \$fids = \\Drupal::entityQuery('file')
    ->condition('filemime', 'image/%', 'LIKE')
    ->condition('status', 1)
    ->range(0, 1)
    ->accessCheck(FALSE)
    ->execute();
  return \$fids ? reset(\$fids) : NULL;
}

function migtest_fill_entity(\$entity) {
  \$fid = migtest_get_image_fid();
  foreach (\$entity->getFieldDefinitions() as \$fname => \$fdef) {
    if (\$fdef->isReadOnly() || \$fdef->isComputed()) continue;
    if (in_array(\$fname, ['id','uuid','vid','revision_id','type','created','changed','langcode',
        'default_langcode','revision_translation_affected','behavior_settings',
        'parent_id','parent_type','parent_field_name'])) continue;
    \$type = \$fdef->getType();
    try {
      switch (\$type) {
        case 'string':
        case 'string_long':
          \$entity->set(\$fname, migtest_lorem(8)); break;
        case 'text':
        case 'text_long':
        case 'text_with_summary':
          \$entity->set(\$fname, ['value' => '<p>' . migtest_lorem(25) . '</p>', 'format' => 'basic_html']); break;
        case 'integer':
        case 'float':
        case 'decimal':
          \$entity->set(\$fname, 42); break;
        case 'boolean':
          \$entity->set(\$fname, 1); break;
        case 'datetime':
          \$entity->set(\$fname, date('Y-m-d\TH:i:s')); break;
        case 'daterange':
          \$entity->set(\$fname, ['value' => date('Y-m-d\TH:i:s'), 'end_value' => date('Y-m-d\TH:i:s', strtotime('+7 days'))]); break;
        case 'link':
          \$entity->set(\$fname, ['uri' => 'https://example.com', 'title' => migtest_lorem(3)]); break;
        case 'email':
          \$entity->set(\$fname, 'test@migration.example.com'); break;
        case 'telephone':
          \$entity->set(\$fname, '+33 1 23 45 67 89'); break;
        case 'image':
          if (\$fid) \$entity->set(\$fname, ['target_id' => \$fid, 'alt' => migtest_lorem(4), 'title' => migtest_lorem(3)]);
          break;
        case 'file':
          if (\$fid) \$entity->set(\$fname, ['target_id' => \$fid, 'display' => 1, 'description' => migtest_lorem(4)]);
          break;
        case 'color_field_type':
          \$entity->set(\$fname, ['color' => '#C0392B', 'opacity' => 1]); break;
        case 'entity_reference':
          // skip — cross-entity refs are too risky to auto-fill
          break;
        case 'svg_image_field':
          if (\$fid) \$entity->set(\$fname, ['target_id' => \$fid]);
          break;
      }
    } catch (\\Exception \$e) {
      // best-effort: skip fields that reject test data
    }
  }
}

// ---- Create and fill paragraph ----
\$para = \\Drupal\\paragraphs\\Entity\\Paragraph::create(['type' => 'PARA_TYPE']);
migtest_fill_entity(\$para);
\$para->save();
echo 'pid=' . \$para->id() . ' type=PARA_TYPE fields_filled=' . count(\$para->getFieldDefinitions()) . PHP_EOL;
" 2>/dev/null
```

Record each node: `{nid, content_type, url, title, paragraphs: [{field, type, pid, fields_filled}]}`.

### Step B3 — Create taxonomy terms

```bash
${CMD} drush php:eval "
\$vocabs = \Drupal::entityTypeManager()->getStorage('taxonomy_vocabulary')->loadMultiple();
foreach (\$vocabs as \$vocab) {
  \$term = \Drupal\taxonomy\Entity\Term::create(['vid' => \$vocab->id(), 'name' => '[MIGRATION-TEST] ' . \$vocab->id()]);
  \$term->save();
  echo 'Term: ' . \$term->id() . ' in ' . \$vocab->id() . '\n';
}
" 2>/dev/null
```

### Step B4 — Test Search API (if active)

If `environment.json > special_modules.search_api = true`:
```bash
${CMD} drush search-api:index 2>/dev/null
```

### Step B5 — Capture HTML snapshots (AVANT migration)

This is the critical step that enables real before/after comparison.

Get a one-time admin login URL:
```bash
${CMD} drush user:login --no-browser 2>/dev/null
# Follow the redirect to get the session cookie
```

For **each test node** and for **each content type's creation+edit forms**, capture and save the rendered HTML.

Use `curl` with the admin session cookie to capture authenticated pages:

```bash
mkdir -p .drupal-migration/baseline/rendered
mkdir -p .drupal-migration/baseline/create-forms
mkdir -p .drupal-migration/baseline/edit-forms

# Get admin session
ULI=$(${CMD} drush user:login --no-browser 2>/dev/null | tr -d '\n')
SITE_URL=$(${CMD} drush php:eval "echo \\Drupal::request()->getSchemeAndHttpHost();" 2>/dev/null | tr -d '\n')

# Capture session cookie by following the one-time login
COOKIE_JAR=$(mktemp)
curl -s -L -c "$COOKIE_JAR" -o /dev/null "$ULI"

# For each test node — capture rendered HTML
for NID in LIST_OF_NIDS; do
  curl -s -b "$COOKIE_JAR" "${SITE_URL}/node/${NID}" \
    | sed 's/form_token.*value="[^"]*"/form_token value="REDACTED"/g' \
    | sed 's/data-drupal-link-query[^>]*>//g' \
    > .drupal-migration/baseline/rendered/node-${NID}.html
  echo "Captured rendered: /node/${NID}"
done

# For each content type — capture the creation form HTML
for TYPE in LIST_OF_CONTENT_TYPES; do
  curl -s -b "$COOKIE_JAR" "${SITE_URL}/node/add/${TYPE}" \
    | sed 's/form_token.*value="[^"]*"/form_token value="REDACTED"/g' \
    > .drupal-migration/baseline/create-forms/${TYPE}.html
  echo "Captured create form: /node/add/${TYPE}"
done

# For each test node — capture the edit form HTML
for NID in LIST_OF_NIDS; do
  curl -s -b "$COOKIE_JAR" "${SITE_URL}/node/${NID}/edit" \
    | sed 's/form_token.*value="[^"]*"/form_token value="REDACTED"/g' \
    > .drupal-migration/baseline/edit-forms/node-${NID}.html
  echo "Captured edit form: /node/${NID}/edit"
done

rm -f "$COOKIE_JAR"
```

**HTTP status check** — record the HTTP status code of each URL:
```bash
for NID in LIST_OF_NIDS; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" -b "$COOKIE_JAR" "${SITE_URL}/node/${NID}")
  echo "node-${NID}: ${STATUS}"
done
```

Store all status codes in baseline JSON.

**Test creation via Drush API (baseline)** — verify each content type can be saved without errors:
```bash
${CMD} drush php:eval "
\$errors = [];
foreach (['CONTENT_TYPE_1', 'CONTENT_TYPE_2'] as \$type) {
  try {
    \$node = \\Drupal\\node\\Entity\\Node::create([
      'type'  => \$type,
      'title' => '[MIGRATION-TEST-CREATE] ' . \$type,
      'uid'   => \$AGENT_UID,
      'status'=> 0,
    ]);
    \$violations = \$node->validate();
    if (\$violations->count() === 0) {
      \$node->save();
      \$node->delete(); // cleanup immediately
      echo 'CREATE_OK:' . \$type . PHP_EOL;
    } else {
      echo 'CREATE_VIOLATION:' . \$type . ':' . (string)\$violations->get(0)->getMessage() . PHP_EOL;
    }
  } catch (\\Exception \$e) {
    echo 'CREATE_ERROR:' . \$type . ':' . \$e->getMessage() . PHP_EOL;
  }
}
" 2>/dev/null
```

**Test edit via Drush API (baseline)** — verify each test node can be loaded and resaved:
```bash
${CMD} drush php:eval "
foreach ([NID1, NID2] as \$nid) {
  try {
    \$node = \\Drupal\\node\\Entity\\Node::load(\$nid);
    \$original = \$node->getTitle();
    \$node->setTitle(\$original . ' [edit-test]');
    \$violations = \$node->validate();
    if (\$violations->count() === 0) {
      \$node->save();
      \$node->setTitle(\$original);
      \$node->save();
      echo 'EDIT_OK:' . \$nid . PHP_EOL;
    } else {
      echo 'EDIT_VIOLATION:' . \$nid . ':' . (string)\$violations->get(0)->getMessage() . PHP_EOL;
    }
  } catch (\\Exception \$e) {
    echo 'EDIT_ERROR:' . \$nid . ':' . \$e->getMessage() . PHP_EOL;
  }
}
" 2>/dev/null
```

Store all results in baseline JSON under `pre_migration_checks`.

### Step B-NAV — Navigation walkthrough (Views + menus + crawl)

**Goal:** Capture the real front-end state of every accessible page — Views lists, menu pages, taxonomy pages — so after migration we can detect any rendering regression even on pages that have nothing to do with the test nodes.

#### B-NAV.1 — Discover all Views page displays

```bash
${CMD} drush php:eval "
\$views = \\Drupal::entityTypeManager()->getStorage('view')->loadMultiple();
foreach (\$views as \$view) {
  if (\$view->status() === false) continue;
  foreach (\$view->get('display') as \$did => \$display) {
    if (\$display['display_plugin'] !== 'page') continue;
    \$path = \$display['display_options']['path'] ?? null;
    if (!\$path) continue;
    \$access = \$display['display_options']['access']['type'] ?? 'none';
    echo \$view->id() . '|' . \$did . '|/' . ltrim(\$path,'/') . '|' . \$access . PHP_EOL;
  }
}
" 2>/dev/null
```

This gives the complete list of Views page URLs. Store them as `navigation.views`.

#### B-NAV.2 — Discover menu links (public navigation)

```bash
${CMD} drush php:eval "
\$tree = \\Drupal::menuTree()->load('main', new \\Drupal\\Core\\Menu\\MenuTreeParameters());
function flatten_menu(\$items, \$depth = 0) {
  foreach (\$items as \$item) {
    \$link = \$item->link;
    if (!\$link->isEnabled()) continue;
    \$url = \$link->getUrlObject();
    if (!\$url->isExternal() && \$url->isRouted()) {
      try { echo str_repeat(' ', \$depth*2) . \$link->getTitle() . '|' . '/' . ltrim(\$url->toString(), '/') . PHP_EOL; }
      catch (\\Exception \$e) {}
    }
    if (\$item->subtree) flatten_menu(\$item->subtree, \$depth + 1);
  }
}
flatten_menu(\$tree);
" 2>/dev/null
```

Also capture `footer` and `account` menus with the same approach. Store as `navigation.menus`.

#### B-NAV.3 — Build the navigation URL list

Combine and deduplicate:
1. Homepage `/`
2. All Views page URLs from B-NAV.1
3. All menu link URLs from B-NAV.2
4. Key admin pages: `/admin/content`, `/admin/structure/types`, `/admin/config`, `/admin/appearance`
5. Key taxonomy listing pages (one per vocab): `/taxonomy/term/FIRST_TID`

Cap at **50 URLs** to keep execution time reasonable. Prioritize: homepage → menus → views → admin → taxonomy.

Store as `navigation.urls` in the baseline JSON.

#### B-NAV.4 — Capture HTML of each navigation URL

```bash
mkdir -p .drupal-migration/baseline/navigation

COOKIE_JAR=$(mktemp)
ULI=$(${CMD} drush user:login --no-browser 2>/dev/null | tr -d '\n')
SITE_URL=$(${CMD} drush php:eval "echo \\Drupal::request()->getSchemeAndHttpHost();" 2>/dev/null | tr -d '\n')
curl -s -L -c "$COOKIE_JAR" -o /dev/null "$ULI"

while IFS= read -r url; do
  SLUG=$(echo "$url" | sed 's|/|_|g' | sed 's|^_||')
  OUTFILE=".drupal-migration/baseline/navigation/${SLUG}.html"
  STATUS=$(curl -s -b "$COOKIE_JAR" \
    -o "$OUTFILE" \
    -w "%{http_code}" \
    "${SITE_URL}${url}")
  # Normalize dynamic tokens
  sed -i 's/form_token.*value="[^"]*"/form_token value="REDACTED"/g' "$OUTFILE"
  sed -i 's/data-history-node-id="[0-9]*"/data-history-node-id="REDACTED"/g' "$OUTFILE"
  sed -i 's/"csrfToken":"[^"]*"/"csrfToken":"REDACTED"/g' "$OUTFILE"
  # Count rendered rows/items in Views (the key metric)
  ROW_COUNT=$(grep -c 'class="views-row\|class="view-content\|<tr\b' "$OUTFILE" 2>/dev/null || echo 0)
  echo "${STATUS}|${ROW_COUNT}|${url}" >> .drupal-migration/baseline/navigation/index.tsv
  echo "Captured: ${url} HTTP=${STATUS} rows=${ROW_COUNT}"
done < <(cat navigation_urls.txt)   # built from step B-NAV.3

rm -f "$COOKIE_JAR"
```

**index.tsv format:** `HTTP_STATUS | VIEWS_ROW_COUNT | URL`

The `VIEWS_ROW_COUNT` is critical: after migration, if a View that had 42 rows now returns 0 rows, that's a regression even if HTTP is 200.

Also record for each URL:
- Presence of PHP errors: `grep -c "Fatal error\|Symfony.*Exception\|Drupal.*Error" OUTFILE`
- Presence of `<main` content block (page rendered correctly)
- Presence of navigation elements (header/nav didn't break)

Store all metrics in `navigation/index.tsv`.

### Step B6 — Write baseline

Create `.drupal-migration/test-baseline.json`:

```json
{
  "created_at": "ISO8601",
  "site_url": "https://site.ddev.site",
  "content_types_tested": ["article", "page"],
  "test_nodes": [
    {"nid": 100, "type": "article", "url": "/node/100", "title": "[MIGRATION-TEST] article test node",
     "paragraphs": [], "http_status_pre": 200}
  ],
  "taxonomy_terms": [{"tid": 50, "vocab": "tags"}],
  "search_api_indexed": false,
  "snapshots": {
    "rendered": [".drupal-migration/baseline/rendered/node-100.html"],
    "create_forms": [".drupal-migration/baseline/create-forms/article.html"],
    "edit_forms": [".drupal-migration/baseline/edit-forms/node-100.html"]
  },
  "pre_migration_checks": {
    "create": {"article": "OK", "page": "OK"},
    "edit": {"100": "OK", "101": "OK"}
  },
  "navigation": {
    "views_discovered": [
      {"view_id": "frontpage", "display": "page_1", "url": "/", "access": "none"},
      {"view_id": "gestion_des_marches", "display": "page_1", "url": "/marches", "access": "role"}
    ],
    "menus_discovered": [
      {"title": "Accueil", "url": "/"},
      {"title": "Marchés", "url": "/marches"}
    ],
    "urls_crawled": 48,
    "index_file": ".drupal-migration/baseline/navigation/index.tsv"
  },
  "urls_to_test": ["/", "/admin/content", "/node/100"]
}
```

Print:
```
✅ Baseline created:
   Content types: N nodes created + paragraphs filled
   Taxonomy: N terms created
   HTML snapshots: N rendered pages + N create forms + N edit forms
   Create API test: N/N OK
   Edit API test: N/N OK
   Navigation crawl: N Views pages + N menu pages = N URLs captured
   Baseline: .drupal-migration/test-baseline.json
```

---

## PHASE: validate

**Philosophy:** The same test nodes created in baseline are still in the database after migration (same nids). We do NOT recreate them — we re-render, re-edit, and compare the before/after state to detect any regression.

Three axes of comparison:
1. **Rendu** — HTML du contenu rendu avant vs après
2. **Création** — les formulaires de création acceptent-ils toujours les soumissions ?
3. **Édition** — les formulaires d'édition des nœuds de test fonctionnent-ils encore ?

### Step V1 — Read baseline

Read `.drupal-migration/test-baseline.json`. Get `test_nodes`, `content_types_tested`, `pre_migration_checks`, `snapshots`.

### Step V2 — Drush health checks

```bash
${CMD} drush status --format=json 2>/dev/null
${CMD} drush updatedb:status 2>/dev/null
${CMD} drush config:status 2>/dev/null | head -30
```

Record: Drupal version matches target, no pending updates, config sync state.

### Step V3 — Verify test nodes still exist

For each node in baseline:
```bash
${CMD} drush php:eval "
foreach ([NID1, NID2, ...] as \$nid) {
  \$node = \\Drupal\\node\\Entity\\Node::load(\$nid);
  echo \$node ? 'OK:' . \$nid . ':' . \$node->bundle() : 'MISSING:' . \$nid;
  echo PHP_EOL;
}
" 2>/dev/null
```

### Step V4 — Capture HTML post-migration (same nodes, same forms)

```bash
mkdir -p .drupal-migration/validate/rendered
mkdir -p .drupal-migration/validate/create-forms
mkdir -p .drupal-migration/validate/edit-forms

ULI=$(${CMD} drush user:login --no-browser 2>/dev/null | tr -d '\n')
SITE_URL=$(${CMD} drush php:eval "echo \\Drupal::request()->getSchemeAndHttpHost();" 2>/dev/null | tr -d '\n')
COOKIE_JAR=$(mktemp)
curl -s -L -c "$COOKIE_JAR" -o /dev/null "$ULI"

# Rendered pages — same nids as baseline
for NID in LIST_OF_NIDS; do
  STATUS=$(curl -s -o .drupal-migration/validate/rendered/node-${NID}.html \
    -w "%{http_code}" -b "$COOKIE_JAR" "${SITE_URL}/node/${NID}")
  # Strip dynamic tokens before diff
  sed -i 's/form_token.*value="[^"]*"/form_token value="REDACTED"/g' \
    .drupal-migration/validate/rendered/node-${NID}.html
  echo "POST node-${NID}: HTTP ${STATUS}"
done

# Create forms — same content types
for TYPE in LIST_OF_CONTENT_TYPES; do
  STATUS=$(curl -s -o .drupal-migration/validate/create-forms/${TYPE}.html \
    -w "%{http_code}" -b "$COOKIE_JAR" "${SITE_URL}/node/add/${TYPE}")
  sed -i 's/form_token.*value="[^"]*"/form_token value="REDACTED"/g' \
    .drupal-migration/validate/create-forms/${TYPE}.html
  echo "POST create form ${TYPE}: HTTP ${STATUS}"
done

# Edit forms — same nids
for NID in LIST_OF_NIDS; do
  STATUS=$(curl -s -o .drupal-migration/validate/edit-forms/node-${NID}.html \
    -w "%{http_code}" -b "$COOKIE_JAR" "${SITE_URL}/node/${NID}/edit")
  sed -i 's/form_token.*value="[^"]*"/form_token value="REDACTED"/g' \
    .drupal-migration/validate/edit-forms/node-${NID}.html
  echo "POST edit form ${NID}: HTTP ${STATUS}"
done

rm -f "$COOKIE_JAR"
```

### Step V5 — HTML diff comparison (before vs after)

**For each rendered node:**
```bash
diff -u \
  .drupal-migration/baseline/rendered/node-${NID}.html \
  .drupal-migration/validate/rendered/node-${NID}.html \
  > .drupal-migration/validate/diff-rendered-node-${NID}.diff 2>&1
LINES=$(wc -l < .drupal-migration/validate/diff-rendered-node-${NID}.diff)
echo "node-${NID} rendered diff: ${LINES} lines changed"
```

**For each create form:**
```bash
diff -u \
  .drupal-migration/baseline/create-forms/${TYPE}.html \
  .drupal-migration/validate/create-forms/${TYPE}.html \
  > .drupal-migration/validate/diff-create-${TYPE}.diff 2>&1
```

**For each edit form:**
```bash
diff -u \
  .drupal-migration/baseline/edit-forms/node-${NID}.html \
  .drupal-migration/validate/edit-forms/node-${NID}.html \
  > .drupal-migration/validate/diff-edit-node-${NID}.diff 2>&1
```

**Classify each diff:**
- 0 lines changed → ✅ IDENTICAL
- < 20 lines changed → 🟡 MINOR (likely cache tokens, Drupal version string — inspect manually)
- > 20 lines changed → 🔴 REGRESSION — show the first 40 lines of diff in the report

**Critical patterns to flag as 🔴 even in small diffs:**
```bash
grep -l "Fatal error\|Symfony\\Component\\Debug\\|Whoops\|The website encountered\|Error:" \
  .drupal-migration/validate/rendered/*.html \
  .drupal-migration/validate/create-forms/*.html \
  .drupal-migration/validate/edit-forms/*.html
```

If any file contains PHP/Drupal errors → 🔴 CRITICAL regardless of diff size.

Also check for missing fields in edit forms (a field lost after migration):
```bash
# For each content type, count the number of form fields in baseline vs validate
grep -c '<div class="form-item' .drupal-migration/baseline/create-forms/${TYPE}.html
grep -c '<div class="form-item' .drupal-migration/validate/create-forms/${TYPE}.html
# Difference → missing/added fields
```

### Step V6 — Test creation API post-migration

**Same test as baseline B5, but now AFTER the migration.** Compare results with `pre_migration_checks.create`:

```bash
${CMD} drush php:eval "
foreach (['CONTENT_TYPE_1', 'CONTENT_TYPE_2'] as \$type) {
  try {
    \$node = \\Drupal\\node\\Entity\\Node::create([
      'type'  => \$type,
      'title' => '[MIGRATION-TEST-POST] ' . \$type . ' creation test',
      'uid'   => \$AGENT_UID,
      'status'=> 0,
    ]);
    \$violations = \$node->validate();
    if (\$violations->count() === 0) {
      \$node->save();
      \$created_nid = \$node->id();
      \$node->delete();
      echo 'CREATE_OK:' . \$type . ':nid=' . \$created_nid . PHP_EOL;
    } else {
      echo 'CREATE_VIOLATION:' . \$type . ':' . (string)\$violations->get(0)->getMessage() . PHP_EOL;
    }
  } catch (\\Exception \$e) {
    echo 'CREATE_ERROR:' . \$type . ':' . \$e->getMessage() . PHP_EOL;
  }
}
" 2>/dev/null
```

Compare line by line with `pre_migration_checks.create`:
- Pre=OK, Post=OK → ✅
- Pre=OK, Post=ERROR → 🔴 REGRESSION — creation broke after migration
- Pre=VIOLATION, Post=VIOLATION (same message) → 🟡 pre-existing issue

### Step V7 — Test edit API post-migration

**Same test as baseline B5, but now AFTER the migration:**

```bash
${CMD} drush php:eval "
foreach ([NID1, NID2, ...] as \$nid) {
  try {
    \$node = \\Drupal\\node\\Entity\\Node::load(\$nid);
    \$original_title = \$node->getTitle();
    \$node->setTitle(\$original_title . ' [post-migration-edit-test]');
    \$violations = \$node->validate();
    if (\$violations->count() === 0) {
      \$node->save();
      // Revert
      \$node->setTitle(\$original_title);
      \$node->save();
      echo 'EDIT_OK:' . \$nid . PHP_EOL;
    } else {
      echo 'EDIT_VIOLATION:' . \$nid . ':' . (string)\$violations->get(0)->getMessage() . PHP_EOL;
    }
  } catch (\\Exception \$e) {
    echo 'EDIT_ERROR:' . \$nid . ':' . \$e->getMessage() . PHP_EOL;
  }
}
" 2>/dev/null
```

Compare with `pre_migration_checks.edit`.

### Step V-NAV — Re-play navigation walkthrough (post-migration)

Re-crawl the **exact same URLs** recorded in `baseline.navigation.index_file`. Do not re-discover — use the same list to ensure a strict before/after comparison.

```bash
mkdir -p .drupal-migration/validate/navigation

COOKIE_JAR=$(mktemp)
ULI=$(${CMD} drush user:login --no-browser 2>/dev/null | tr -d '\n')
SITE_URL=$(${CMD} drush php:eval "echo \\Drupal::request()->getSchemeAndHttpHost();" 2>/dev/null | tr -d '\n')
curl -s -L -c "$COOKIE_JAR" -o /dev/null "$ULI"

# Re-crawl every URL from baseline index
while IFS='|' read -r pre_status pre_rows url; do
  SLUG=$(echo "$url" | sed 's|/|_|g' | sed 's|^_||')
  OUTFILE=".drupal-migration/validate/navigation/${SLUG}.html"

  POST_STATUS=$(curl -s -b "$COOKIE_JAR" \
    -o "$OUTFILE" \
    -w "%{http_code}" \
    "${SITE_URL}${url}")

  # Normalize
  sed -i 's/form_token.*value="[^"]*"/form_token value="REDACTED"/g' "$OUTFILE"
  sed -i 's/data-history-node-id="[0-9]*"/data-history-node-id="REDACTED"/g' "$OUTFILE"
  sed -i 's/"csrfToken":"[^"]*"/"csrfToken":"REDACTED"/g' "$OUTFILE"

  POST_ROWS=$(grep -c 'class="views-row\|class="view-content\|<tr\b' "$OUTFILE" 2>/dev/null || echo 0)
  PHP_ERRORS=$(grep -c "Fatal error\|Symfony.*Exception\|Drupal.*Error\|The website encountered" "$OUTFILE" 2>/dev/null || echo 0)

  # Diff HTML
  BASELINE_FILE=".drupal-migration/baseline/navigation/${SLUG}.html"
  DIFF_LINES=0
  if [ -f "$BASELINE_FILE" ]; then
    diff -u "$BASELINE_FILE" "$OUTFILE" > ".drupal-migration/validate/navigation/diff-${SLUG}.diff" 2>&1
    DIFF_LINES=$(wc -l < ".drupal-migration/validate/navigation/diff-${SLUG}.diff")
  fi

  # Classify
  STATUS_CLASS="OK"
  if [ "$PHP_ERRORS" -gt 0 ]; then STATUS_CLASS="ERROR_PHP"; fi
  if [ "$POST_STATUS" != "$pre_status" ]; then STATUS_CLASS="HTTP_CHANGED"; fi
  if [ "$POST_ROWS" -lt "$((pre_rows / 2))" ] && [ "$pre_rows" -gt 5 ]; then STATUS_CLASS="VIEWS_EMPTY"; fi
  if [ "$DIFF_LINES" -gt 50 ]; then STATUS_CLASS="${STATUS_CLASS}_BIGDIFF"; fi

  echo "${pre_status}→${POST_STATUS}|${pre_rows}→${POST_ROWS}|${DIFF_LINES}|${PHP_ERRORS}|${STATUS_CLASS}|${url}" \
    >> .drupal-migration/validate/navigation/index.tsv

  echo "[${STATUS_CLASS}] ${url} HTTP=${pre_status}→${POST_STATUS} rows=${pre_rows}→${POST_ROWS} diff=${DIFF_LINES}lines"

done < .drupal-migration/baseline/navigation/index.tsv

rm -f "$COOKIE_JAR"
```

**Status classes explained:**
- `OK` — HTTP same, rows same, no PHP errors, diff < 50 lines
- `HTTP_CHANGED` — 200→403 (access broken), 200→500 (crash), etc.
- `VIEWS_EMPTY` — View had N rows before, < N/2 after (data loss or query broken)
- `ERROR_PHP` — PHP Fatal/Exception visible in page HTML
- `*_BIGDIFF` — More than 50 lines changed in the HTML (layout regression)

**For each `ERROR_PHP` or `HTTP_CHANGED` URL** — show the first 20 lines of diff immediately in the console and in the final report.

**For each `VIEWS_EMPTY` URL** — run a Drush check to confirm the View still executes:
```bash
${CMD} drush php:eval "
\$view = \\Drupal\\views\\Views::getView('VIEW_ID');
\$view->setDisplay('DISPLAY_ID');
\$view->execute();
echo 'rows=' . count(\$view->result) . PHP_EOL;
" 2>/dev/null
```

### Step V8 — Tally results

Build a results matrix:

| Check | Pre-migration | Post-migration | Status |
|-------|--------------|----------------|--------|
| node-NID rendered (HTTP) | 200 | 200 | ✅ |
| node-NID rendered (HTML diff) | — | 0 lines | ✅ |
| node-NID edit form (HTTP) | 200 | 200 | ✅ |
| node-NID edit form (field count) | 42 fields | 42 fields | ✅ |
| TYPE create form (HTTP) | 200 | 200 | ✅ |
| TYPE creation API | OK | OK | ✅ |
| TYPE edit API | OK | OK | ✅ |
| PHP errors in HTML | none | none | ✅ |
| Navigation: / (HTTP) | 200 | 200 | ✅ |
| Navigation: /marches (HTTP + rows) | 200 \| 42 rows | 200 \| 42 rows | ✅ |
| Navigation: /admin/content (HTTP) | 200 | 200 | ✅ |
| Navigation: View gestion_marches (rows) | 42 | 42 | ✅ |

Count: `total_passed` / `total_checks` / `total_regressions`.

### Step V9 — Generate comparison report

Create `.drupal-migration/comparison-report.md`:

```markdown
# Comparison Report — D{SOURCE} → D{TARGET} — TIMESTAMP

## Summary
- ✅ Identical: N
- 🟡 Minor diff (< 20 lines): N  
- 🔴 Regressions: N
- 🔴 PHP errors: N

---

## Rendu des nœuds de test

| Node | Type | HTTP pre | HTTP post | HTML diff | Status |
|------|------|---------|----------|-----------|--------|
| /node/NID | article | 200 | 200 | 0 lines | ✅ |

---

## Formulaires de création

| Content type | HTTP pre | HTTP post | Fields pre | Fields post | API create pre | API create post | Status |
|---|---|---|---|---|---|---|---|
| article | 200 | 200 | 12 | 12 | OK | OK | ✅ |

---

## Formulaires d'édition

| Node | Type | HTTP pre | HTTP post | Fields pre | Fields post | API edit pre | API edit post | Status |
|------|------|---------|----------|-----------|------------|-------------|--------------|--------|
| /node/NID | article | 200 | 200 | 18 | 18 | OK | OK | ✅ |

---

## Regressions détectées

[For each 🔴 diff — show the relevant diff excerpt]

\`\`\`diff
[first 40 lines of the diff]
\`\`\`

---

## Fichiers de diff complets

- `.drupal-migration/validate/diff-rendered-node-NID.diff`
- `.drupal-migration/validate/diff-create-TYPE.diff`
- `.drupal-migration/validate/diff-edit-node-NID.diff`
```

### Step V10 — Update migration report and final commit

Read `report_file`. Find `TESTS_PLACEHOLDER` and replace with comparison report summary.

```bash
git add MIGRATION-REPORT-YYYY-MM-DD.md .drupal-migration/comparison-report.md
git add .drupal-migration/baseline/ .drupal-migration/validate/
git commit -m "chore: migration report D{SOURCE}→D{TARGET} — N/N tests, N regressions"
```

### Step V11 — Print final result

```
✅ Migration D{SOURCE}→D{TARGET} COMPLETE

   Drupal version: TARGET_VERSION ✓
   Tests: N/N passed (N regressions)

   Rendu:     N/N identique (N diff mineur, N régressions)
   Création:  N/N OK par type de contenu
   Édition:   N/N OK par nœud de test

   Rapport complet: MIGRATION-REPORT-YYYY-MM-DD.md
   Diffs HTML:      .drupal-migration/validate/diff-*.diff

🎉 Your site is now running Drupal TARGET_VERSION
```

If 🔴 regressions: display each one immediately with the diff excerpt before the final summary.

If site down / admin 500 → invoke rollback-manager immediately.

## Error handling

- curl unavailable → use `wget` as fallback
- Playwright not available → curl-only HTML capture (already used above)
- Test node missing after migration → 🔴 CRITICAL, check if content migration ran
- Admin password unknown → use `drush uli` for one-time login URL
- Search API not indexed → re-index once, retry
- diff not available → compare file sizes + grep for error patterns

## Validation

```bash
wc -l /home/thomasroger/.claude/skills/drupal-migration/agents/test-runner.md
head -3 /home/thomasroger/.claude/skills/drupal-migration/agents/test-runner.md
```

## Commit

```bash
git -C /home/thomasroger/.claude add skills/drupal-migration/agents/test-runner.md
git -C /home/thomasroger/.claude commit -m "feat: add test-runner agent for drupal-migration skill"
```
