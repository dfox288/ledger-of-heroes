# Entity Filter Meilisearch Migration Plan

**Date:** 2025-11-25
**Type:** Architecture - Filter Migration
**Status:** Ready for Implementation

---

## Overview

Migrate all 6 remaining entity filters (items, races, classes, backgrounds, feats, monsters) from deprecated MySQL params to Meilisearch syntax, following the proven pattern from the spell filter migration.

### Problem Statement

The spell filter migration (completed 2025-11-25) revealed a critical bug: 9/10 filters were silently broken, using deprecated MySQL params that the API ignored, returning ALL results instead of filtered results.

**Example from spells:**
```typescript
// BROKEN (silently ignored by API):
params.level = 3              // Returns ALL 477 spells ❌
params.concentration = '1'    // Returns ALL 477 spells ❌

// WORKING (Meilisearch syntax):
params.filter = 'level = 3'               // Returns 67 spells ✅
params.filter = 'concentration = true'    // Returns 218 spells ✅
```

**Current Risk:** All 6 remaining entities (items, races, classes, backgrounds, feats, monsters) use the same broken MySQL param pattern and are likely experiencing the same silent failures.

---

## Goals

1. **Fix broken filters** - Ensure all filters return correctly filtered results
2. **Standardize syntax** - Use Meilisearch `filter=` parameter consistently
3. **Maintain UX** - Preserve existing filter UI and user-facing query params
4. **Improve performance** - Leverage Meilisearch's <50ms response times
5. **Enable advanced queries** - Support AND/OR/IN operators, range queries

---

## Approach: Parallel Subagent Migration

### Why Parallel?

- ✅ **Fast execution** - 6x speedup (30-40 min vs 2-3 hours)
- ✅ **Proven pattern** - Spell migration provides clear template
- ✅ **Independent work** - Each entity has isolated file/logic
- ✅ **Consistent results** - All agents follow same instructions

### Migration Phases

```
Phase 1: Pre-Flight Testing (10 min)
  └─> Test API endpoints to identify filterable fields

Phase 2: Parallel Execution (20-30 min)
  ├─> Launch 6 subagents simultaneously
  ├─> items: Migrate 5 filters
  ├─> races: Migrate 2 filters
  ├─> classes: Migrate 2 filters
  ├─> backgrounds: Add filters if fields available
  ├─> feats: Migrate 1 filter
  └─> monsters: Migrate 3 filters

Phase 3: Validation & Integration (10 min)
  ├─> Collect subagent reports
  ├─> TypeScript compilation
  ├─> Test suite execution
  ├─> HTTP response tests
  └─> Filter accuracy tests

Phase 4: Documentation & Commit (10 min)
  ├─> Create handover document
  ├─> Update CHANGELOG.md
  ├─> Single atomic commit
  └─> Push to remote
```

---

## Entity Inventory

### Current State (Pre-Migration)

| Entity | File | Current Filters | Risk Level | Priority |
|--------|------|----------------|------------|----------|
| **items** | `app/pages/items/index.vue` | 5 filters | 🔴 HIGH | P1 |
| **monsters** | `app/pages/monsters/index.vue` | 3 filters | 🟡 MEDIUM | P1 |
| **races** | `app/pages/races/index.vue` | 2 filters | 🟡 MEDIUM | P2 |
| **classes** | `app/pages/classes/index.vue` | 2 filters | 🟡 MEDIUM | P2 |
| **feats** | `app/pages/feats/index.vue` | 1 filter | 🟢 LOW | P3 |
| **backgrounds** | `app/pages/backgrounds/index.vue` | 0 filters | 🟢 LOW | P3 |

### Detailed Filter Analysis

#### 1. Items (5 filters)
```typescript
// Current (potentially broken):
if (selectedType.value) params.type = selectedType.value           // ID-based
if (selectedRarity.value) params.rarity = selectedRarity.value     // String
if (selectedMagic.value) params.is_magic = selectedMagic.value     // Boolean string
if (hasCharges.value) params.has_charges = hasCharges.value        // Boolean string
if (hasPrerequisites.value) params.has_prerequisites = hasPrerequisites.value
```

**Expected Meilisearch Fields:**
- `item_type_id` or `item_type_code` or `item_type_name` (needs API testing)
- `rarity` (string: common, uncommon, rare, very rare, legendary, artifact)
- `is_magic` (boolean)
- `charges_max` (number, filter with `> 0` for has_charges)
- `prerequisites` (string/array, filter with `IS NOT EMPTY` or length check)

#### 2. Monsters (3 filters)
```typescript
// Current (potentially broken):
if (selectedCR.value) params.cr = selectedCR.value         // Range string: "0-4", "5-10"
if (selectedType.value) params.type = selectedType.value   // String: "dragon", "beast"
if (isLegendary.value) params.is_legendary = isLegendary.value  // Boolean string
```

**Expected Meilisearch Fields:**
- `challenge_rating` (number/string, needs range support)
- `type` (string)
- `is_legendary` (boolean) - **may not exist, needs verification**

**Special Case:** CR ranges need translation:
- "0-4" → `challenge_rating >= 0 AND challenge_rating <= 4`
- "17+" → `challenge_rating >= 17`

#### 3. Races (2 filters)
```typescript
// Current (potentially broken):
if (selectedSize.value) params.size = selectedSize.value           // Code: M, S, L
if (hasDarkvision.value) params.has_darkvision = hasDarkvision.value  // Boolean string
```

**Expected Meilisearch Fields:**
- `size` or `size_code` (string code: T, S, M, L, H, G)
- `has_darkvision` (boolean) - **may not exist, needs verification**

#### 4. Classes (2 filters)
```typescript
// Current (potentially broken):
if (isBaseClass.value) params.is_base_class = isBaseClass.value      // Boolean string
if (isSpellcaster.value) params.is_spellcaster = isSpellcaster.value // Boolean string
```

**Expected Meilisearch Fields:**
- `is_base_class` (boolean)
- `is_spellcaster` (boolean)

#### 5. Feats (1 filter)
```typescript
// Current (potentially broken):
if (hasPrerequisites.value) params.has_prerequisites = hasPrerequisites.value
```

**Expected Meilisearch Fields:**
- `has_prerequisites` (boolean) or `prerequisites` (string, check for empty)

#### 6. Backgrounds (0 filters)
```typescript
// Current: Search only, no filters
queryBuilder: computed(() => ({}))
```

**Potential Meilisearch Fields to Add:**
- Language filters
- Proficiency type filters
- Source book filters

---

## Pre-Flight Testing Protocol

### Purpose
Verify which fields are actually indexed in Meilisearch before launching subagents.

### Test Script

```bash
#!/bin/bash
# Save as: test-entity-filters.sh

API="http://localhost:8080/api/v1"

echo "=== Entity Filter Pre-Flight Tests ==="
echo ""

# Items
echo "1. ITEMS"
echo "  - is_magic filter:"
curl -s "$API/items?filter=is_magic%20%3D%20true&per_page=1" | jq '.meta.total' || echo "FAILED"
echo "  - rarity filter:"
curl -s "$API/items?filter=rarity%20%3D%20rare&per_page=1" | jq '.meta.total' || echo "FAILED"
echo ""

# Races
echo "2. RACES"
echo "  - size filter:"
curl -s "$API/races?filter=size%20%3D%20M&per_page=1" | jq '.meta.total' || echo "FAILED"
echo ""

# Classes
echo "3. CLASSES"
echo "  - is_base_class filter:"
curl -s "$API/classes?filter=is_base_class%20%3D%20true&per_page=1" | jq '.meta.total' || echo "FAILED"
echo ""

# Feats
echo "4. FEATS"
echo "  - has_prerequisites filter:"
curl -s "$API/feats?filter=has_prerequisites%20%3D%20true&per_page=1" | jq '.meta.total' || echo "FAILED"
echo ""

# Monsters
echo "5. MONSTERS"
echo "  - type filter:"
curl -s "$API/monsters?filter=type%20%3D%20dragon&per_page=1" | jq '.meta.total' || echo "FAILED"
echo "  - challenge_rating filter:"
curl -s "$API/monsters?filter=challenge_rating%20%3E%3D%205%20AND%20challenge_rating%20%3C%3D%2010&per_page=1" | jq '.meta.total' || echo "FAILED"
echo ""

echo "=== Tests Complete ==="
```

### Interpreting Results

**Success Response:**
```json
{
  "data": [...],
  "meta": { "total": 42 }
}
```
✅ Field is indexed and filterable

**Error Response:**
```json
{
  "message": "Attribute `field_name` is not filterable.",
  "filterable_attributes": ["other", "fields"]
}
```
❌ Field not indexed - filter must be removed or alternative found

**All Records Response:**
```json
{
  "meta": { "total": 999 }  // Matches total entity count
}
```
⚠️ Filter syntax wrong or field name mismatch

---

## Meilisearch Syntax Reference

### Core Patterns

```typescript
// Equality
filter = 'level = 3'
filter = 'school_code = EV'

// Boolean (MUST be true/false, not "1"/"0")
filter = 'concentration = true'
filter = 'is_magic = false'

// Comparison
filter = 'challenge_rating >= 5'
filter = 'level <= 3'

// Arrays (IN = OR logic)
filter = 'class_slugs IN [wizard, bard]'
filter = 'damage_types IN [F, C, L]'

// Combine with AND
filter = 'level = 3 AND class_slugs IN [wizard]'
filter = 'is_magic = true AND rarity = legendary'

// Empty checks
filter = 'prerequisites IS NOT EMPTY'
filter = 'prerequisites IS EMPTY'

// Range queries
filter = 'challenge_rating >= 5 AND challenge_rating <= 10'
```

### Type Conversions

```typescript
// Boolean filters: Frontend uses strings, convert to boolean
const concentrationFilter = ref<string | null>('1')  // From query params

// Convert for Meilisearch
if (concentrationFilter.value !== null) {
  const boolValue = concentrationFilter.value === '1' || concentrationFilter.value === 'true'
  meilisearchFilters.push(`concentration = ${boolValue}`)
}

// Multi-select: Join with commas, wrap in IN [...]
const selectedTypes = ref<string[]>(['F', 'C'])

if (selectedTypes.value.length > 0) {
  const codes = selectedTypes.value.join(', ')
  meilisearchFilters.push(`damage_types IN [${codes}]`)
}

// Range queries: Convert UI range to Meilisearch expression
const selectedCR = ref<string | null>('5-10')

if (selectedCR.value === '5-10') {
  meilisearchFilters.push('challenge_rating >= 5 AND challenge_rating <= 10')
} else if (selectedCR.value === '17+') {
  meilisearchFilters.push('challenge_rating >= 17')
}
```

---

## Subagent Task Template

**Use this template for each entity migration.**

```markdown
# Migrate [ENTITY] Filters to Meilisearch

## Context
You are migrating [ENTITY] entity filters from deprecated MySQL params to Meilisearch syntax. The spell filters were recently migrated, fixing a critical bug where 9/10 filters were silently broken (returning ALL results instead of filtered).

## Reference Implementation
- **Gold Standard:** `app/pages/spells/index.vue` (lines 134-209)
- **Handover:** `docs/HANDOVER-2025-11-25-COMPLETE-MEILISEARCH-MIGRATION.md`
- **Migration Plan:** `docs/plans/2025-11-25-entity-filter-meilisearch-migration.md`

## Your Entity: [ENTITY]
- **File:** `app/pages/[ENTITY]/index.vue`
- **Current Filters:** [LIST]
- **Filterable Fields:** [LIST from pre-flight testing]

## Step-by-Step Instructions

### 1. Document Current State (5 min)
```bash
# Test each current filter to see if it's broken
curl "http://localhost:8080/api/v1/[ENTITY]?[filter_param]=value&per_page=5"

# Compare result count to total entity count
# If they match → filter is broken (returns all records)
```

Create a "Before" section documenting:
- Filter name
- Current param syntax
- Expected result count
- Actual result count
- Status: ✅ Working or ❌ Broken

### 2. Update queryBuilder (10 min)

**Locate the queryBuilder in `app/pages/[ENTITY]/index.vue`:**

```typescript
// BEFORE (MySQL params):
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  if (selectedFilter.value) params.filter_name = selectedFilter.value
  return params
})
```

**Convert to Meilisearch syntax:**

```typescript
// AFTER (Meilisearch):
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // Convert each filter to Meilisearch syntax
  if (selectedFilter.value !== null) {
    // For boolean filters: convert string to boolean
    const boolValue = selectedFilter.value === '1' || selectedFilter.value === 'true'
    meilisearchFilters.push(`field_name = ${boolValue}`)
  }

  // Combine all filters with AND
  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

**Key Rules:**
- Boolean values: Use `true`/`false` (NOT `'1'`/`'0'`)
- String values: No quotes needed for codes/slugs
- Arrays: Use `IN [value1, value2]` syntax
- Combine: Use `' AND '` to join multiple filters
- Spacing: `field = value` (spaces around `=`)

### 3. Test New Filters (5 min)

```bash
# Test each migrated filter
curl "http://localhost:8080/api/v1/[ENTITY]?filter=field_name%20%3D%20value&per_page=5"

# Test combined filters
curl "http://localhost:8080/api/v1/[ENTITY]?filter=field1%20%3D%20value1%20AND%20field2%20%3D%20value2&per_page=5"
```

Document "After" results:
- Filter name
- New Meilisearch syntax
- Result count
- Status: ✅ Working

### 4. Verify TypeScript (2 min)

```bash
npm run typecheck
```

Ensure no new TypeScript errors related to your changes.

### 5. Test Frontend (3 min)

```bash
# Start dev server if not running
docker compose exec nuxt npm run dev

# Visit page
open http://localhost:3000/[ENTITY]

# Test each filter in the UI
# Verify filter chips display correctly
# Verify clear filters works
```

### 6. Report Results

**DO NOT COMMIT.** Report back with:

```markdown
## [ENTITY] Migration Report

### Filters Migrated: [X]

#### Before (Broken Filters):
1. **[filter_name]**: `params.[param] = value`
   - Expected: ~[N] results
   - Actual: [TOTAL] results (ALL RECORDS) ❌

#### After (Working Filters):
1. **[filter_name]**: `filter=[field] = [value]`
   - Result: [N] results ✅

### Combined Filters:
- Test: `filter=[field1] = [value1] AND [field2] = [value2]`
- Result: [N] results ✅

### Issues Encountered:
- [List any problems, unsupported fields, etc.]

### TypeScript:
- ✅ No errors / ⚠️ [X] pre-existing errors (unrelated)

### Frontend Testing:
- ✅ Page loads (HTTP 200)
- ✅ All filters work in UI
- ✅ Filter chips display correctly
- ✅ Clear filters works

### File Changed:
- `app/pages/[ENTITY]/index.vue` (lines [X-Y])
```

## Success Criteria

- ✅ All filters return correct filtered results (not all records)
- ✅ Response times <100ms
- ✅ Combined filters work with AND operator
- ✅ No new TypeScript errors
- ✅ Frontend page loads and filters work in UI
- ✅ Filter chips and clear functionality intact

## If You Encounter Issues

1. **"Attribute not filterable" error:**
   - Document the error
   - List alternative fields from error message
   - Report back for guidance

2. **Filter returns all records:**
   - Check spelling of field name
   - Verify boolean values are `true`/`false` (not strings)
   - Check spacing: `field = value` not `field=value`

3. **TypeScript errors:**
   - Check filter value types match ref types
   - Ensure meilisearchFilters is `string[]`
   - Verify computed returns `Record<string, unknown>`

4. **Frontend breaks:**
   - Check browser console for errors
   - Verify filter refs are updated correctly
   - Check clearFilters function updates all refs

## Resources
- Meilisearch docs: https://docs.meilisearch.com/learn/filtering/filter_expression_reference.html
- Spell migration: `docs/HANDOVER-2025-11-25-COMPLETE-MEILISEARCH-MIGRATION.md`
- Gold standard: `app/pages/spells/index.vue`
```

---

## Post-Migration Validation

### Comprehensive Test Suite

```bash
#!/bin/bash
# Save as: validate-entity-migration.sh

echo "=== Entity Filter Migration Validation ==="
echo ""

# 1. TypeScript compilation
echo "1. TypeScript Compilation"
npm run typecheck
if [ $? -eq 0 ]; then
  echo "  ✅ TypeScript: PASS"
else
  echo "  ❌ TypeScript: FAIL"
fi
echo ""

# 2. Test suite
echo "2. Test Suite"
npm run test
if [ $? -eq 0 ]; then
  echo "  ✅ Tests: PASS"
else
  echo "  ⚠️  Tests: FAIL (check for pre-existing failures)"
fi
echo ""

# 3. HTTP responses
echo "3. HTTP Response Tests"
for entity in items races classes backgrounds feats monsters; do
  status=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:3000/$entity")
  if [ "$status" = "200" ]; then
    echo "  ✅ /$entity: HTTP $status"
  else
    echo "  ❌ /$entity: HTTP $status"
  fi
done
echo ""

# 4. API filter tests (sample from each entity)
echo "4. API Filter Tests"

# Items
items_magic=$(curl -s "http://localhost:8080/api/v1/items?filter=is_magic%20%3D%20true&per_page=1" | jq '.meta.total')
echo "  items (is_magic=true): $items_magic results"

# Races
races_medium=$(curl -s "http://localhost:8080/api/v1/races?filter=size%20%3D%20M&per_page=1" | jq '.meta.total')
echo "  races (size=M): $races_medium results"

# Classes
classes_base=$(curl -s "http://localhost:8080/api/v1/classes?filter=is_base_class%20%3D%20true&per_page=1" | jq '.meta.total')
echo "  classes (is_base_class=true): $classes_base results"

# Feats
feats_prereq=$(curl -s "http://localhost:8080/api/v1/feats?filter=has_prerequisites%20%3D%20true&per_page=1" | jq '.meta.total')
echo "  feats (has_prerequisites=true): $feats_prereq results"

# Monsters
monsters_dragon=$(curl -s "http://localhost:8080/api/v1/monsters?filter=type%20%3D%20dragon&per_page=1" | jq '.meta.total')
echo "  monsters (type=dragon): $monsters_dragon results"

echo ""
echo "=== Validation Complete ==="
```

### Success Metrics

**All checks must pass:**
- ✅ TypeScript compiles with no new errors
- ✅ Test suite passes (or only pre-existing failures)
- ✅ All 6 entity pages return HTTP 200
- ✅ All filter tests return filtered results (not total count)
- ✅ Combined filters work for each entity
- ✅ Frontend UI functional (manual spot check)

**If any check fails:**
1. Review subagent report for that entity
2. Check error messages
3. Manually test the failing filter
4. Fix or rollback that entity's changes
5. Re-run validation

---

## Documentation Plan

### 1. Handover Document
**File:** `docs/HANDOVER-2025-11-25-ENTITY-FILTER-MIGRATION.md`

**Contents:**
- Session overview
- Entities migrated (6 total)
- Before/after filter counts per entity
- Critical discoveries (broken filters documented)
- Migration pattern used
- Issues encountered and resolutions
- Testing results
- Files changed
- Commit summary
- Next steps

### 2. CHANGELOG Update
**File:** `CHANGELOG.md`

```markdown
### Changed
- Migrated all entity filters to Meilisearch syntax (2025-11-25)
  - Items: 5 filters (type, rarity, is_magic, has_charges, has_prerequisites)
  - Monsters: 3 filters (cr, type, is_legendary)
  - Races: 2 filters (size, has_darkvision)
  - Classes: 2 filters (is_base_class, is_spellcaster)
  - Feats: 1 filter (has_prerequisites)
  - Backgrounds: 0 filters (search only)

### Fixed
- Critical bug where entity filters were silently broken, returning ALL results instead of filtered results (2025-11-25)
- Items filters now correctly filter results
- Monster filters now correctly filter results
- [... etc for each entity]

### Performance
- Entity filter response times <50ms (was variable, often >200ms)
- Combined filters now supported (e.g., `filter=type = dragon AND cr >= 10`)
```

### 3. Filter Reference Document
**File:** `docs/MEILISEARCH-FILTER-REFERENCE.md`

**Contents:**
- Complete list of filterable fields per entity
- Syntax examples for each filter type
- Common patterns (boolean, array, range)
- Combining filters
- Troubleshooting guide

---

## Commit Strategy

### Single Atomic Commit (Recommended)

```bash
# Stage all changes
git add app/pages/items/index.vue
git add app/pages/races/index.vue
git add app/pages/classes/index.vue
git add app/pages/backgrounds/index.vue
git add app/pages/feats/index.vue
git add app/pages/monsters/index.vue
git add docs/
git add CHANGELOG.md

# Commit with detailed message
git commit -m "$(cat <<'EOF'
feat: Migrate all entity filters to Meilisearch syntax

🚨 BREAKING CHANGE: Complete migration from MySQL params to Meilisearch-only filtering for all entities

**Critical Bug Fix:**
- Fixed broken filters across 6 entities using deprecated MySQL params
- Filters were silently returning ALL results instead of filtered results
- All entities now use unified `?filter=` parameter with Meilisearch syntax

**Entities Migrated (6 total):**

1. **Items (5 filters):**
   - type: `filter=item_type_id = X`
   - rarity: `filter=rarity = legendary`
   - is_magic: `filter=is_magic = true`
   - has_charges: `filter=charges_max > 0`
   - has_prerequisites: `filter=prerequisites IS NOT EMPTY`

2. **Monsters (3 filters):**
   - cr: `filter=challenge_rating >= 5 AND challenge_rating <= 10`
   - type: `filter=type = dragon`
   - is_legendary: `filter=is_legendary = true`

3. **Races (2 filters):**
   - size: `filter=size = M`
   - has_darkvision: `filter=has_darkvision = true`

4. **Classes (2 filters):**
   - is_base_class: `filter=is_base_class = true`
   - is_spellcaster: `filter=is_spellcaster = true`

5. **Feats (1 filter):**
   - has_prerequisites: `filter=has_prerequisites = true`

6. **Backgrounds:**
   - No filters (search only)

**Benefits:**
- All filters now work correctly (many were 100% broken before)
- Combined filters: `filter=type = dragon AND challenge_rating >= 10`
- Performance: <50ms response times (Meilisearch)
- Advanced queries: AND/OR/IN operators, range queries
- Consistent pattern across all 7 entities (including spells)

**Migration Pattern:**
- Followed proven spell migration pattern
- Parallel subagent execution (6 entities simultaneously)
- Comprehensive testing and validation
- Atomic commit for rollback safety

**Files Changed:**
- app/pages/items/index.vue (queryBuilder migration)
- app/pages/races/index.vue (queryBuilder migration)
- app/pages/classes/index.vue (queryBuilder migration)
- app/pages/backgrounds/index.vue (no changes, documented)
- app/pages/feats/index.vue (queryBuilder migration)
- app/pages/monsters/index.vue (queryBuilder migration)
- docs/HANDOVER-2025-11-25-ENTITY-FILTER-MIGRATION.md (new)
- docs/MEILISEARCH-FILTER-REFERENCE.md (new)
- CHANGELOG.md (updated)

**Testing:**
- All TypeScript compilation clean
- All entity pages HTTP 200
- All filters return correct filtered results
- Combined filters verified working
- Response times <50ms confirmed

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

**Rationale for Single Commit:**
- ✅ Atomic change - single revert point if needed
- ✅ Follows spell migration pattern (single commit)
- ✅ All entities use consistent approach
- ✅ Clear before/after state
- ✅ Easier to cherry-pick to other branches

---

## Rollback Plan

### If Migration Fails

```bash
# Rollback all changes
git checkout HEAD -- app/pages/items/index.vue
git checkout HEAD -- app/pages/races/index.vue
git checkout HEAD -- app/pages/classes/index.vue
git checkout HEAD -- app/pages/backgrounds/index.vue
git checkout HEAD -- app/pages/feats/index.vue
git checkout HEAD -- app/pages/monsters/index.vue

# Clean up docs
rm docs/HANDOVER-2025-11-25-ENTITY-FILTER-MIGRATION.md
rm docs/MEILISEARCH-FILTER-REFERENCE.md
git checkout HEAD -- CHANGELOG.md

# Investigate issues manually
npm run typecheck
npm run test
```

### If Specific Entity Fails

```bash
# Rollback just one entity
git checkout HEAD -- app/pages/[ENTITY]/index.vue

# Other entities remain migrated
# Document the failure for manual investigation
```

---

## Timeline Estimate

| Phase | Duration | Notes |
|-------|----------|-------|
| Pre-flight testing | 10 min | Run test script, analyze results |
| Launch subagents | 5 min | 6 parallel task invocations |
| Subagent execution | 20-30 min | Agents work in parallel |
| Collect reports | 5 min | Review all 6 reports |
| Validation | 10 min | Run test suite, HTTP checks |
| Documentation | 10 min | Handover, CHANGELOG, reference |
| Commit & push | 5 min | Single atomic commit |
| **TOTAL** | **65-75 min** | **~1 hour** |

Compare to sequential: 6 entities × 20 min = 120 min (2 hours)
**Time saved: ~45 minutes (40% faster)**

---

## Risk Assessment

### High Risk
- ❌ **Multiple entities fail** - If API doesn't support Meilisearch for most fields
  - **Mitigation:** Pre-flight testing catches this early
  - **Fallback:** Document unsupported filters, remove from UI (like spell migration)

### Medium Risk
- ⚠️ **Subagent conflicts** - Agents modify same shared files
  - **Mitigation:** Each entity has isolated page file, no overlap
  - **Fallback:** Agents report back before committing, manual conflict resolution

### Low Risk
- 🟢 **Filter syntax errors** - Agents use wrong Meilisearch syntax
  - **Mitigation:** Clear syntax reference in task template, spell migration examples
  - **Fallback:** Validation suite catches syntax errors before commit

### Negligible Risk
- ✅ **Breaking changes** - Users depend on old MySQL params in bookmarks
  - **Impact:** Minimal - most users use UI, not direct param manipulation
  - **Mitigation:** Document breaking change in CHANGELOG
  - **Future:** Could add backward compat layer if needed

---

## Success Criteria

### Must Have (Blocking)
- ✅ All filters return correctly filtered results
- ✅ TypeScript compiles with no new errors
- ✅ All entity pages return HTTP 200
- ✅ Combined filters work (AND operator)
- ✅ Changes committed and documented

### Should Have (Non-Blocking)
- ✅ Test suite passes (or only pre-existing failures)
- ✅ Response times <100ms
- ✅ Filter UI/UX unchanged
- ✅ Comprehensive filter reference doc

### Nice to Have (Future)
- 🔮 Additional filters based on available Meilisearch fields
- 🔮 Advanced filter UI (presets, saved queries)
- 🔮 Range filter UI components
- 🔮 Backward compatibility layer for old params

---

## Next Steps (After Migration)

1. **Monitor production** (if deployed)
   - Check error logs for filter-related issues
   - Monitor response times
   - Gather user feedback

2. **Enhance filters** (future sprint)
   - Add `tag_slugs` and `source_codes` filters (spells pattern)
   - Explore advanced filter UI patterns
   - Create filter presets for common queries

3. **Performance optimization** (future sprint)
   - Implement filter result caching
   - Add filter result count previews
   - Optimize combined filter queries

4. **Documentation** (continuous)
   - Update user guides with new filter capabilities
   - Create filter usage analytics
   - Document common filter patterns

---

## Appendix

### A. Meilisearch Resources
- [Filter Reference](https://docs.meilisearch.com/learn/filtering/filter_expression_reference.html)
- [Boolean Filters](https://docs.meilisearch.com/learn/filtering/filter_expression_reference.html#boolean)
- [Array Filters](https://docs.meilisearch.com/learn/filtering/filter_expression_reference.html#array)
- [Comparison Operators](https://docs.meilisearch.com/learn/filtering/filter_expression_reference.html#comparison-operators)

### B. Project Resources
- Spell migration: `docs/HANDOVER-2025-11-25-COMPLETE-MEILISEARCH-MIGRATION.md`
- API docs: `http://localhost:8080/docs/api`
- Gold standard: `app/pages/spells/index.vue`

### C. Common Pitfalls
1. **Boolean strings** - Must convert `'1'` to `true`, not pass string
2. **Missing spaces** - `field=value` fails, use `field = value`
3. **Quotes** - Don't add quotes around values (Meilisearch adds them)
4. **Field names** - Use exact field name from API response
5. **Empty checks** - Use `IS EMPTY` or `IS NOT EMPTY`, not `= ""`

---

**Plan Status:** ✅ Ready for Implementation
**Estimated Completion:** 2025-11-25
**Owner:** Claude (with parallel subagents)
