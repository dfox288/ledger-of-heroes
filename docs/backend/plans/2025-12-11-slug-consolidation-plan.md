# Slug Consolidation Implementation Plan

**Date**: 2025-12-11
**Branch**: `feature/issue-506-slug-consolidation`
**Issue**: #506
**Status**: In Progress
**Breaking Change**: Yes (API response format)

## Overview

Eliminate the redundant `full_slug` column across all entities. The `slug` column will contain source-prefixed values (e.g., `phb:acid-splash`). Data will be re-imported fresh.

### Goals
- Single `slug` column containing source-prefixed values
- Eliminate confusion about which slug to use
- Simplify codebase and API surface

### Non-Goals
- API versioning (updating v1 in place)
- Frontend changes (separate team responsibility)
- Data migration (re-importing instead)

---

## Progress

| Phase | Description | Status |
|-------|-------------|--------|
| 1. Schema | Drop `full_slug` from 13 tables | ✅ Complete |
| 2. Importers | Refactor `GeneratesSlugs` + 9 importers | ✅ Complete |
| 3. Models | Remove from fillable + update 12 relationships | ✅ Complete |
| 4. Resources | Remove from 12 resources | ✅ Complete |
| 5. Factories | Update 13 factories | ✅ Complete |
| 6. Seeders | Update 12 seeders | ✅ Complete |
| 7. Requests/Controllers | Update validation rules + queries | ✅ Complete |
| 8. Tests | Update ~1,120 references | ✅ Complete |
| 9. Data | Re-import prod + test data | ✅ Complete |
| 10. Quality | Pint, tests, chaos testing | ✅ Complete |
| 11. Docs | CHANGELOG, commit, PR | ✅ Complete |

**Commit**: `3aa02d8` (WIP checkpoint)

### Phase 7 Summary (Completed 2025-12-11)

Updated 126 files in app/ with zero remaining `full_slug` references:
- AppServiceProvider route bindings (7 entities)
- Form Requests (2 files)
- Controllers (5 files + PHPDocs)
- Services (~15 files)
- Choice Handlers (12 files)
- WizardFlowTesting (3 files)

---

## Phase 7: Requests, Controllers & Services (NEW)

**Discovered during implementation** - these files still reference `full_slug`:

### Task 7.1: Update AppServiceProvider route bindings

**File**: `app/Providers/AppServiceProvider.php`

Change all route model bindings from `full_slug` to `slug`:
```php
// BEFORE
return Spell::where('full_slug', $value)->firstOrFail();

// AFTER
return Spell::where('slug', $value)->firstOrFail();
```

**Entities to update**: Spell, Race, Background, CharacterClass, Item, Feat, Monster

### Task 7.2: Update Form Requests

**Files**:
- `app/Http/Requests/Character/Choice/StoreFeatureSelectionRequest.php`
- `app/Http/Requests/Character/Condition/CharacterConditionStoreRequest.php`

**Changes**:
- Update `Rule::exists()` from `full_slug` to `slug`
- Update query conditions from `full_slug` to `slug`

### Task 7.3: Update Controllers

**Files**:
- `app/Http/Controllers/Api/FeatureSelectionController.php`
- `app/Http/Controllers/Api/CharacterSpellController.php`

**Changes**:
- Update all `where('full_slug', ...)` to `where('slug', ...)`
- Remove fallback `orWhere('slug', ...)` since slug is now the canonical column

### Task 7.4: Update Controller PHPDocs

**Files**:
- `app/Http/Controllers/Api/SpellController.php`
- `app/Http/Controllers/Api/RaceController.php`
- `app/Http/Controllers/Api/MonsterController.php`

**Changes**:
- Remove `full_slug` from `@response` schema annotations

### Task 7.5: Comprehensive full_slug sweep

**IMPORTANT**: Before proceeding to tests, run:
```bash
grep -r "full_slug" app/ --include="*.php"
```

Verify **zero results** before continuing.

---

## Phase 8: Test Updates

### Task 8.1: Search and categorize test references

```bash
grep -r "full_slug" tests/ --include="*.php" | wc -l
# Current: ~1,120 references
```

### Task 8.2: Update test assertions

**Pattern changes**:
```php
// BEFORE
->assertJsonFragment(['full_slug' => 'phb:fireball']);

// AFTER
->assertJsonFragment(['slug' => 'phb:fireball']);
```

### Task 8.3: Update test data references

**Pattern changes**:
```php
// BEFORE
$spell = Spell::where('full_slug', 'phb:fireball')->first();

// AFTER
$spell = Spell::where('slug', 'phb:fireball')->first();
```

### Task 8.4: Run test suites incrementally

```bash
sail artisan test --testsuite=Unit-Pure
sail artisan test --testsuite=Unit-DB
sail artisan test --testsuite=Feature-DB
# Feature-Search requires re-index (Phase 9)
```

---

## Phase 9: Data Re-import

### Task 9.1: Re-import production data

```bash
sail artisan migrate:fresh --seed
sail artisan import:all
```

### Task 9.2: Re-import test fixtures

```bash
SCOUT_PREFIX=test_ sail artisan import:all --env=testing
```

### Task 9.3: Run Feature-Search tests

```bash
sail artisan test --testsuite=Feature-Search
```

---

## Phase 10: Quality Gates

### Task 10.1: Run Pint formatter

```bash
sail pint
```

### Task 10.2: Run full test suite

```bash
sail artisan test --testsuite=Unit-Pure
sail artisan test --testsuite=Unit-DB
sail artisan test --testsuite=Feature-DB
sail artisan test --testsuite=Feature-Search
sail artisan test --testsuite=Importers
```

### Task 10.3: Run wizard flow chaos testing

```bash
sail artisan test:wizard-flow --count=10 --chaos
```

### Task 10.4: Manual API verification

```bash
curl -s http://localhost:8080/api/v1/spells/1 | jq '.data | {slug, name}'
# Expected: {"slug": "phb:acid-splash", "name": "Acid Splash"}
```

---

## Phase 11: Documentation & Commit

### Task 11.1: Update CHANGELOG.md

```markdown
## [Unreleased]

### Changed
- **BREAKING**: Removed `full_slug` column from all entity tables
  - The `slug` column now contains source-prefixed values (e.g., `phb:acid-splash`)
  - Removed `full_slug` from API responses
  - Frontend must update all slug references to use prefixed format
```

### Task 11.2: Final commit and PR

```bash
git add -A
git commit -m "feat: Consolidate to single source-prefixed slug column

BREAKING CHANGE: API responses no longer include full_slug field.
The slug field now contains source-prefixed values (e.g., phb:acid-splash).

Closes #506"

gh pr create --title "feat: Consolidate to single source-prefixed slug" --body "..."
```

---

## Summary

| Phase | Tasks | Status |
|-------|-------|--------|
| 1. Schema | 1 | ✅ Done |
| 2. Importers | 2 | ✅ Done |
| 3. Models | 2 | ✅ Done |
| 4. Resources | 1 | ✅ Done |
| 5. Factories | 1 | ✅ Done |
| 6. Seeders | 1 | ✅ Done |
| 7. Requests/Controllers | 5 | ✅ Done |
| 8. Tests | 4 | ✅ Done |
| 9. Data | 3 | ✅ Done |
| 10. Quality | 4 | ✅ Done |
| 11. Docs | 2 | ✅ Done |

**Total**: 26 tasks across 11 phases

---

## Frontend Coordination Checklist

Provide to frontend team:

- [ ] `full_slug` field removed from all API responses
- [ ] `slug` field now contains prefixed values (e.g., `phb:acid-splash` instead of `acid-splash`)
- [ ] Meilisearch filters must use prefixed slug values
- [ ] Character reference fields (`race_slug`, `background_slug`, etc.) unchanged - already stored prefixed values
- [ ] Update any hardcoded slug values in frontend code
