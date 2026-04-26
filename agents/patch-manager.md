---
name: drupal-migration-patch-manager
description: Manages composer patches before and after migration. Pre-checks that each patch will apply to target module versions, archives obsolete patches, generates new patches from manual fixes applied during migration. Called before updater and again after.
---

# Drupal Migration Patch Manager

Composer patches are a silent failure point in migrations. A patch that "Skips" during `composer update` leaves the site broken without a clear error. This agent pre-validates patches, manages failures, and archives resolved patches.

## Prerequisites

Read `.drupal-migration/environment.json`. `CMD` = `environment.json > command_prefix`.
Read `composer.json` to extract `extra.patches`.

## Invocation modes

- `phase: pre-migration` — validate patches against target versions (called before updater)
- `phase: post-migration` — verify patches applied, archive obsolete ones, generate patches for manual fixes

---

## PHASE: pre-migration

### Step PM1 — Extract all patches from composer.json

Read `extra.patches` from `composer.json`. For each package→patch mapping:

```json
"patches": {
  "drupal/module_name": {
    "Description": "path/to/patch.patch"
  }
}
```

### Step PM2 — Check patch file existence

For each patch path:
```bash
ls -la PATCH_PATH 2>/dev/null || echo "MISSING_PATCH: PATCH_PATH"
```

Missing patches → 🔴 BLOCKING (will fail silently during composer update).

### Step PM3 — Validate each patch against target version

For local `.patch` files:
```bash
# Get the current module source
git -C vendor/drupal/MODULE diff HEAD 2>/dev/null || true

# Test if patch applies cleanly to the TARGET version
# First, check if patch was already applied (patch may be obsolete)
patch --dry-run -p1 -d web/modules/contrib/MODULE_NAME < patches/PATCH_FILE.patch 2>&1
```

Results:
- Exit 0 → ✅ patch applies cleanly
- Exit 1 with "already applied" → 🟡 OBSOLETE — patch may not be needed in target version
- Exit 1 with "failed" → 🔴 PATCH_WILL_FAIL — must resolve before migration

### Step PM4 — Report pre-migration patch status

For each patch:
```
drupal/xls_views_data_export:
  "Fix buildResponse signature": patches/fix.patch
  → STATUS: OBSOLETE (module 4.0.0 includes this fix natively)
  → ACTION: Remove from composer.json patches section

drupal/webform:
  "Custom submission patch": patches/webform-submission-patch-2.patch
  → STATUS: APPLIES CLEANLY on 6.3.0-beta8
  → ACTION: Keep
```

For 🔴 patches that will fail:
- Option 1: Remove the patch if the fix is included in the target version
- Option 2: Update the patch for the new version
- Option 3: Skip the patch (risky)

If autonomous mode: automatically remove OBSOLETE patches, keep VALID patches, flag FAILING patches for manual review.

### Step PM5 — Update composer.json

Remove obsolete patches from `extra.patches`. Run `composer validate` after changes.

---

## PHASE: post-migration

### Step PM-POST1 — Verify patches were applied

After `composer update`, check the `cweagans/composer-patches` log:
```bash
# composer update output contains "Applying patches for drupal/MODULE"
# or "Could not apply patch! Skipping."
# Check the module files for evidence of patch application
```

For each patch that should have applied:
```bash
# Check if the patch's key change exists in the installed file
grep -c "PATCH_SIGNATURE_STRING" web/modules/contrib/MODULE_NAME/file.php 2>/dev/null
```

### Step PM-POST2 — Generate patches for manual fixes

During the migration, we may have manually fixed files in contrib modules (like xls_views_data_export). These fixes should be formalized as patches so they survive future `composer update`:

```bash
# For each manually-modified contrib file:
cd web/modules/contrib/MODULE_NAME
git diff HEAD -- src/file.php > ../../../patches/MODULE_NAME-DESCRIPTION.patch

# Or if file was committed to migration branch:
git show HEAD:web/modules/contrib/MODULE_NAME/src/file.php > /tmp/after.php
git show pre-migration-TAG:web/modules/contrib/MODULE_NAME/src/file.php > /tmp/before.php
diff -u /tmp/before.php /tmp/after.php > patches/MODULE_NAME-DESCRIPTION.patch
```

Add the new patch to `composer.json > extra.patches`.

### Step PM-POST3 — Archive obsolete patches

Move patches that are no longer needed to `patches/archived/`:
```bash
mkdir -p patches/archived/
mv patches/OBSOLETE_PATCH.patch patches/archived/
# Add a README explaining why archived
```

Update `composer.json` to remove archived patches.

### Step PM-POST4 — Generate patch status report

Create `.drupal-migration/patch-report.md`:

```markdown
# Patch Manager Report — TIMESTAMP

## Pre-migration validation
| Package | Patch | Status | Action |
|---------|-------|--------|--------|
| drupal/module | patches/fix.patch | ✅ VALID | Kept |
| drupal/module2 | patches/old.patch | 🟡 OBSOLETE | Removed |
| drupal/module3 | patches/fail.patch | 🔴 FAILED | Manual fix required |

## Post-migration patches generated
| Package | Patch file | Reason |
|---------|-----------|--------|
| drupal/xls_views_data_export | patches/xls-signature-fix.patch | buildResponse signature auto-fix |

## Archived patches
| Patch | Reason |
|-------|--------|
| patches/old-fix.patch | Included in module 4.0.0 |
```

## Error handling

- Patch file missing → 🔴 abort, update composer.json to remove it
- All patches fail → still continue but flag as 🔴 REQUIRES_MANUAL_INTERVENTION
- `git diff` not available → note as "cannot generate patch from manual fix"
