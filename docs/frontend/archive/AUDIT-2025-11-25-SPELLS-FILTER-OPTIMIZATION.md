# Spells Filter Optimization Audit

**Date:** 2025-11-25
**Auditor:** Claude Code
**Project:** D&D 5e Compendium Frontend
**Focus:** `/spells` API endpoint filter implementation

---

## Executive Summary

This audit compares the **backend API capabilities** (477 spells, Meilisearch v1.11) against our **frontend implementation** (`app/pages/spells/index.vue`). The goal is to identify missing filters, optimization opportunities, and provide actionable recommendations.

**Key Findings:**
- ✅ **10/18 filterable fields implemented** (56% coverage)
- ✅ **Core filters working correctly** (level, school, class, concentration, ritual, components)
- ⚠️ **8 missing filters** (tags, sources, effect types, casting time, range, duration, etc.)
- ⚠️ **No sorting UI** (API supports 4 sort fields)
- ⚠️ **No range operators** for level filtering (e.g., "1st-3rd level")
- ✅ **Composable architecture** is sound and extensible

---

## API Capabilities (Backend)

### Available Query Parameters

From `http://localhost:8080/docs/api.json` - `/v1/spells` GET endpoint:

```
?q=<search>              Full-text search (name, description)
?filter=<expression>     Meilisearch filter syntax
?sort_by=<field>         name, level, created_at, updated_at (default: name)
?sort_direction=<dir>    asc, desc (default: asc)
?per_page=<number>       1-100 (default: 15)
?page=<number>           Page number (default: 1)
```

### Filterable Fields (18 Total)

#### Integer Fields (2)
| Field | Type | Operators | Examples | Implemented |
|-------|------|-----------|----------|-------------|
| `id` | int | `=`, `!=`, `>`, `>=`, `<`, `<=`, `TO` | `id = 123` | ❌ No (not useful for UI) |
| `level` | int (0-9) | `=`, `!=`, `>`, `>=`, `<`, `<=`, `TO` | `level = 3`, `level 1 TO 3`, `level <= 3` | ✅ **Yes** (equals only) |

#### String Fields (5)
| Field | Type | Operators | Examples | Implemented |
|-------|------|-----------|----------|-------------|
| `school_code` | string | `=`, `!=` | `school_code = EV` (Evocation) | ✅ **Yes** (equals only) |
| `school_name` | string | `=`, `!=` | `school_name = Evocation` | ❌ No (using `school_code`) |
| `casting_time` | string | `=`, `!=` | `casting_time = "1 action"` | ❌ **No** |
| `range` | string | `=`, `!=` | `range = "60 feet"` | ❌ **No** |
| `duration` | string | `=`, `!=` | `duration = "Concentration, up to 1 minute"` | ❌ **No** |

#### Boolean Fields (5)
| Field | Type | Examples | Implemented |
|-------|------|----------|-------------|
| `concentration` | bool | `concentration = true` | ✅ **Yes** |
| `ritual` | bool | `ritual = true` | ✅ **Yes** |
| `requires_verbal` | bool | `requires_verbal = false` | ✅ **Yes** |
| `requires_somatic` | bool | `requires_somatic = false` | ✅ **Yes** |
| `requires_material` | bool | `requires_material = false` | ✅ **Yes** |

#### Array Fields (6)
| Field | Type | Operators | Examples | Implemented |
|-------|------|-----------|----------|-------------|
| `class_slugs` | array | `IN`, `NOT IN`, `IS EMPTY` | `class_slugs IN [wizard, bard]` | ✅ **Yes** (IN only) |
| `damage_types` | array | `IN`, `NOT IN`, `IS EMPTY` | `damage_types IN [F, C]` (Fire, Cold) | ✅ **Yes** (IN only) |
| `saving_throws` | array | `IN`, `NOT IN`, `IS EMPTY` | `saving_throws IN [DEX, CON]` | ✅ **Yes** (IN only) |
| `tag_slugs` | array | `IN`, `NOT IN`, `IS EMPTY` | `tag_slugs IN [fire]` | ❌ **No** |
| `source_codes` | array | `IN`, `NOT IN`, `IS EMPTY` | `source_codes IN [PHB, XGE]` | ❌ **No** |
| `effect_types` | array | `IN`, `NOT IN`, `IS EMPTY` | `effect_types IN [healing, buff]` | ❌ **No** |

---

## Current Implementation Analysis

### ✅ What We Did Right

#### 1. Core Filter Coverage (10/18 fields)
```typescript
// app/pages/spells/index.vue:116-131
const { queryParams } = useMeilisearchFilters([
  { ref: selectedLevel, field: 'level' },                           // ✅ Integer
  { ref: selectedSchool, field: 'school_code', transform: ... },     // ✅ String (with transform)
  { ref: selectedClass, field: 'class_slugs', type: 'in' },          // ✅ Array
  { ref: concentrationFilter, field: 'concentration', type: 'boolean' }, // ✅ Boolean
  { ref: ritualFilter, field: 'ritual', type: 'boolean' },           // ✅ Boolean
  { ref: selectedDamageTypes, field: 'damage_types', type: 'in' },   // ✅ Array
  { ref: selectedSavingThrows, field: 'saving_throws', type: 'in' }, // ✅ Array
  { ref: verbalFilter, field: 'requires_verbal', type: 'boolean' },  // ✅ Boolean
  { ref: somaticFilter, field: 'requires_somatic', type: 'boolean' },// ✅ Boolean
  { ref: materialFilter, field: 'requires_material', type: 'boolean' }// ✅ Boolean
])
```

**Strengths:**
- All 5 boolean fields implemented (concentration, ritual, components)
- Multi-select arrays working correctly (damage types, saving throws)
- Transform function used correctly for `school_code` lookup
- Filter chips display active filters clearly
- Filter count badge works (`useFilterCount` composable)

#### 2. Composable Architecture
```typescript
// app/composables/useMeilisearchFilters.ts
export type FilterType = 'equals' | 'boolean' | 'in' | 'range' | 'isEmpty' | 'greaterThan'
```

**Strengths:**
- ✅ Declarative filter configs (readable, maintainable)
- ✅ Type-safe with TypeScript
- ✅ Auto-skips null/undefined/empty values
- ✅ Supports 6 filter types (equals, boolean, in, range, isEmpty, greaterThan)
- ✅ Combines filters with `AND` automatically

**Supported Types:**
| Type | Implemented | Used | Missing Operators |
|------|-------------|------|-------------------|
| `equals` | ✅ Yes | ✅ Yes (level, school) | `!=` (not equal) |
| `boolean` | ✅ Yes | ✅ Yes (5 fields) | - |
| `in` | ✅ Yes | ✅ Yes (3 arrays) | `NOT IN`, `IS EMPTY` |
| `range` | ✅ Yes | ❌ **Not used** | - |
| `isEmpty` | ✅ Yes | ❌ **Not used** | - |
| `greaterThan` | ✅ Yes | ❌ **Not used** | `<`, `<=`, `>=`, `TO` |

#### 3. UI Organization
```typescript
// app/pages/spells/index.vue:252-373
<UiFilterLayout>
  <template #primary>   <!-- Level, School, Class -->
  <template #quick>     <!-- Concentration, Ritual, Components -->
  <template #advanced>  <!-- Damage Types, Saving Throws -->
  <template #actions>   <!-- Clear Filters -->
</UiFilterLayout>
```

**Strengths:**
- ✅ Logical grouping (primary, quick, advanced)
- ✅ Most-used filters in primary slot
- ✅ Binary toggles use `UiFilterToggle`
- ✅ Multi-selects use `UiFilterMultiSelect`
- ✅ Filter chips for all active filters

---

## ⚠️ Missing Features (Optimization Opportunities)

### 1. Level Range Filtering ⭐ HIGH PRIORITY

**API Capability:**
```bash
# Current: Only equality
GET /api/v1/spells?filter=level = 3

# Missing: Range queries
GET /api/v1/spells?filter=level 1 TO 3         # Inclusive range (TO operator)
GET /api/v1/spells?filter=level >= 5           # 5th level or higher
GET /api/v1/spells?filter=level <= 3           # Up to 3rd level
```

**Why It Matters:**
- Users often search by **spell slot availability** ("I have 1st-3rd level slots")
- Common D&D gameplay pattern ("show me low-level spells")
- API supports `TO`, `>=`, `<=` operators

**Recommended Implementation:**
```typescript
// Option A: Add "Range" toggle to existing level dropdown
const levelFilterMode = ref<'exact' | 'range'>('exact')
const levelMin = ref<number | null>(null)
const levelMax = ref<number | null>(null)

// Use 'range' type in useMeilisearchFilters
{ ref: selectedLevel, field: 'level', type: 'range', min: levelMin, max: levelMax }

// Option B: Add separate "Up to Nth level" quick filter
const maxLevel = ref<number | null>(null)
{ ref: maxLevel, field: 'level', type: 'greaterThan' } // Would need 'lessThanOrEqual' type
```

**Effort:** Medium (2-3 hours with tests)

---

### 2. Sorting UI ⭐ HIGH PRIORITY

**API Capability:**
```bash
GET /api/v1/spells?sort_by=level&sort_direction=asc
GET /api/v1/spells?sort_by=name&sort_direction=desc
```

**Available Sort Fields:**
- `name` (default)
- `level`
- `created_at`
- `updated_at`

**Why It Matters:**
- Users may want to browse **alphabetically** vs **by level** vs **newest first**
- API supports 4 sort fields + 2 directions (8 combinations)
- Currently hardcoded to `name` ASC

**Recommended Implementation:**
```typescript
// Add sort state
const sortBy = ref<'name' | 'level' | 'created_at' | 'updated_at'>('name')
const sortDirection = ref<'asc' | 'desc'>('asc')

// Pass to useEntityList
const { ... } = useEntityList({
  endpoint: '/spells',
  queryBuilder: computed(() => ({
    ...queryParams.value,
    sort_by: sortBy.value,
    sort_direction: sortDirection.value
  })),
  // ...
})

// UI: Add USelectMenu in header area
<USelectMenu
  v-model="sortBy"
  :items="[
    { label: 'Name', value: 'name' },
    { label: 'Level', value: 'level' },
    { label: 'Newest', value: 'created_at' },
    { label: 'Recently Updated', value: 'updated_at' }
  ]"
  label="Sort by"
/>
```

**Effort:** Low (1-2 hours with tests)

---

### 3. Tag Filtering (tag_slugs) ⭐ MEDIUM PRIORITY

**API Capability:**
```bash
GET /api/v1/spells?filter=tag_slugs IN [fire, cold]
GET /api/v1/spells?filter=tag_slugs IS EMPTY
```

**Why It Matters:**
- API documentation states: "Only 22% of spells have tags"
- Tags could include: `fire`, `cold`, `teleportation`, `healing`, etc.
- Could help users discover thematic spell groups

**Challenge:**
- No `/tags` reference endpoint exists (need to inspect data)
- Low data coverage (78% of spells have no tags)

**Recommended Implementation:**
```typescript
// 1. Fetch tags from existing spells (no dedicated endpoint)
const { data: allSpells } = await useAsyncData('all-spells', () =>
  $fetch('/spells', { query: { per_page: 100 } })
)
const uniqueTags = [...new Set(allSpells.value.flatMap(s => s.tags || []))]

// 2. Add multi-select filter
const selectedTags = ref<string[]>([])
{ ref: selectedTags, field: 'tag_slugs', type: 'in' }

// 3. Add to advanced filters section
<UiFilterMultiSelect
  v-model="selectedTags"
  :options="tagOptions"
  label="Tags"
  placeholder="All Tags"
/>
```

**Effort:** Medium (3-4 hours - requires data inspection, testing)

---

### 4. Source Book Filtering (source_codes) ⭐ MEDIUM PRIORITY

**API Capability:**
```bash
GET /api/v1/spells?filter=source_codes IN [PHB, XGE, TCoE]
GET /api/v1/spells?filter=source_codes NOT IN [UA]  # Exclude Unearthed Arcana
```

**Why It Matters:**
- **Official books:** PHB (Player's Handbook), XGE (Xanathar's Guide), TCoE (Tasha's Cauldron)
- **Playtest content:** UA (Unearthed Arcana) - many DMs disallow
- Users may want to filter by **campaign-legal sources**

**Recommended Implementation:**
```typescript
// Fetch reference data (if endpoint exists)
const { data: sources } = useReferenceData<Source>('/sources')

// Or hardcode common sources
const sourceOptions = [
  { label: 'Player\'s Handbook', value: 'PHB' },
  { label: 'Xanathar\'s Guide', value: 'XGE' },
  { label: 'Tasha\'s Cauldron', value: 'TCoE' },
  { label: 'Sword Coast Guide', value: 'SCAG' },
  { label: 'Elemental Evil', value: 'EE' }
]

const selectedSources = ref<string[]>([])
{ ref: selectedSources, field: 'source_codes', type: 'in' }
```

**Effort:** Low-Medium (2-3 hours - depends on reference endpoint availability)

---

### 5. Effect Type Filtering (effect_types) ⭐ LOW PRIORITY

**API Capability:**
```bash
GET /api/v1/spells?filter=effect_types IN [healing, buff, debuff, control]
```

**Why It Matters:**
- Helps users find spells by **mechanical effect** (not just damage)
- Examples: healing, buffing, debuffing, crowd control, utility

**Challenge:**
- Similar to `tag_slugs`, likely low data coverage
- No dedicated reference endpoint

**Effort:** Medium (3-4 hours - requires data inspection)

---

### 6. Casting Time / Range / Duration Filters ⭐ LOW PRIORITY

**API Capability:**
```bash
GET /api/v1/spells?filter=casting_time = "1 action"
GET /api/v1/spells?filter=range = "60 feet"
GET /api/v1/spells?filter=duration = "Instantaneous"
```

**Why We Skipped Them:**
```typescript
// app/pages/spells/index.vue:31-35
// Phase 3: Removed unsupported filters (not indexed in Meilisearch):
// - higherLevelsFilter (has_higher_levels not filterable)
// - castingTimeFilter (casting_time not filterable)
// - rangeFilter (range not filterable)
// - durationFilter (duration not filterable)
```

**⚠️ CORRECTION:** These fields **ARE filterable** according to API docs!

**Challenge:**
- **Free-text values** make dropdowns impractical
- Examples:
  - `casting_time`: "1 action", "1 bonus action", "1 reaction", "1 minute", "10 minutes", "1 hour", "8 hours", "12 hours"
  - `range`: "Self", "Touch", "30 feet", "60 feet", "120 feet", "1 mile", "Special"
  - `duration`: "Instantaneous", "1 round", "1 minute", "10 minutes", "1 hour", "8 hours", "Until dispelled", "Concentration, up to X"

**Recommended Approach:**
- Use **full-text search** (`?q=1 action`) instead of filters
- Or create **normalized categories** in backend:
  - `casting_time_category`: `action`, `bonus_action`, `reaction`, `ritual`
  - `range_category`: `self`, `touch`, `short` (<30ft), `medium` (30-120ft), `long` (120ft+)
  - `duration_category`: `instantaneous`, `concentration`, `permanent`, `timed`

**Effort:** High (requires backend changes to add categorical fields)

---

## 🎯 Recommendations

### Phase 1: Quick Wins (1-2 days)

1. **Add Sorting UI** ⭐ HIGH PRIORITY
   - Add `USelectMenu` for `sort_by` and `sort_direction`
   - 4 sort fields × 2 directions = 8 options
   - Effort: 1-2 hours + tests

2. **Add Level Range Filtering** ⭐ HIGH PRIORITY
   - Use existing `range` type in `useMeilisearchFilters`
   - Add "Min Level" / "Max Level" dropdowns OR "Up to Nth level" toggle
   - Common use case: "Show me spells I can cast at level 5"
   - Effort: 2-3 hours + tests

### Phase 2: Medium Priority (2-3 days)

3. **Add Source Book Filtering** ⭐ MEDIUM PRIORITY
   - Multi-select for PHB, XGE, TCoE, SCAG, etc.
   - Allows filtering out UA (playtest) content
   - Effort: 2-3 hours + tests

4. **Add Tag Filtering** ⭐ MEDIUM PRIORITY
   - Multi-select for thematic tags (fire, cold, teleportation)
   - Requires data inspection (no `/tags` endpoint)
   - Effort: 3-4 hours + tests

### Phase 3: Advanced Features (Optional)

5. **Add Effect Type Filtering** ⭐ LOW PRIORITY
   - Multi-select for healing, buff, debuff, control, utility
   - Effort: 3-4 hours + tests

6. **Add "Empty Array" Filters** ⭐ LOW PRIORITY
   - Extend `useMeilisearchFilters` to support `IS EMPTY` / `IS NOT EMPTY`
   - Use cases:
     - "Spells with no damage" (`damage_types IS EMPTY`)
     - "Spells with no saving throw" (`saving_throws IS EMPTY`)
   - Effort: 2-3 hours + tests

7. **Add Negative Filters** ⭐ LOW PRIORITY
   - Extend composable to support `NOT IN` operator
   - Use cases:
     - "Not available to Wizard" (`class_slugs NOT IN [wizard]`)
     - "Exclude fire damage" (`damage_types NOT IN [F]`)
   - Effort: 2-3 hours + tests

---

## 🛠️ Composable Enhancement Opportunities

### 1. Add Missing Operator Types

**Current Coverage:**
```typescript
export type FilterType =
  | 'equals'           // ✅ field = value
  | 'boolean'          // ✅ field = true/false
  | 'in'              // ✅ field IN [value1, value2]
  | 'range'           // ✅ field >= min AND field <= max
  | 'isEmpty'         // ✅ field IS EMPTY
  | 'greaterThan'     // ✅ field > value
```

**Recommendations:**
```typescript
export type FilterType =
  | 'equals'           // field = value
  | 'notEquals'        // ⚠️ MISSING: field != value
  | 'boolean'          // field = true/false
  | 'in'              // field IN [value1, value2]
  | 'notIn'           // ⚠️ MISSING: field NOT IN [value1, value2]
  | 'range'           // field >= min AND field <= max
  | 'isEmpty'         // field IS EMPTY
  | 'isNotEmpty'      // ⚠️ MISSING: field IS NOT EMPTY (currently using boolean for this)
  | 'greaterThan'     // field > value
  | 'greaterThanOrEqual' // ⚠️ MISSING: field >= value
  | 'lessThan'        // ⚠️ MISSING: field < value
  | 'lessThanOrEqual' // ⚠️ MISSING: field <= value
  | 'to'              // ⚠️ MISSING: field value1 TO value2 (inclusive range)
```

**Effort:** 4-6 hours (add types + tests for each operator)

---

## 📊 Implementation Scorecard

| Category | Implemented | Total | % Coverage |
|----------|-------------|-------|-----------|
| **Filterable Fields** | 10 | 18 | 56% |
| **Integer Filters** | 1 | 2 | 50% |
| **String Filters** | 1 | 5 | 20% |
| **Boolean Filters** | 5 | 5 | 100% ✅ |
| **Array Filters** | 3 | 6 | 50% |
| **Operators** | 6 | 13 | 46% |
| **Sorting** | 0 | 4 | 0% |

**Overall Grade: B (56% coverage, excellent architecture)**

---

## 🎯 Priority Matrix

| Priority | Feature | Impact | Effort | ROI |
|----------|---------|--------|--------|-----|
| ⭐⭐⭐ | Sorting UI | High | Low | ⭐⭐⭐⭐⭐ |
| ⭐⭐⭐ | Level Range Filtering | High | Medium | ⭐⭐⭐⭐ |
| ⭐⭐ | Source Book Filtering | Medium | Medium | ⭐⭐⭐ |
| ⭐⭐ | Tag Filtering | Medium | Medium | ⭐⭐ |
| ⭐ | Effect Type Filtering | Low | Medium | ⭐ |
| ⭐ | Empty Array Filters | Low | Low | ⭐⭐ |
| ⭐ | Negative Filters | Low | Low | ⭐⭐ |

---

## 🔥 Example API Queries We're Missing

### Powerful Combinations
```bash
# Fire damage spells, 3rd level or lower, Wizard can learn
GET /api/v1/spells?filter=damage_types IN [F] AND level <= 3 AND class_slugs IN [wizard]

# Subtle spell candidates (no verbal + no somatic)
GET /api/v1/spells?filter=requires_verbal = false AND requires_somatic = false

# High-level ritual spells (5+) without concentration
GET /api/v1/spells?filter=level >= 5 AND ritual = true AND concentration = false

# Utility spells (no damage, no saving throw)
GET /api/v1/spells?filter=damage_types IS EMPTY AND saving_throws IS EMPTY

# Bonus action spells from PHB
GET /api/v1/spells?filter=casting_time = "1 bonus action" AND source_codes IN [PHB]
```

**Current Status:**
- ✅ Can do: First query (except `level <=` part)
- ❌ Cannot do: Bonus action filter, utility spell filter (IS EMPTY)

---

## 📋 Action Items

### Immediate (This Sprint)
- [ ] Add sorting UI (`sort_by`, `sort_direction`)
- [ ] Add level range filtering (min/max)
- [ ] Update `CLAUDE.md` to remove incorrect "not filterable" comments about `casting_time`, `range`, `duration`

### Next Sprint
- [ ] Add source book filtering (`source_codes IN [...]`)
- [ ] Add tag filtering (`tag_slugs IN [...]`)
- [ ] Extend `useMeilisearchFilters` with missing operators (`notIn`, `lessThanOrEqual`, `to`)

### Backlog
- [ ] Investigate effect type data coverage
- [ ] Consider backend changes for categorical casting_time/range/duration fields
- [ ] Add empty array filters (`IS EMPTY`, `IS NOT EMPTY`)
- [ ] Add negative filters (`NOT IN`)

---

## 🧪 Testing Requirements

For each new filter:

1. **Unit Test (Vitest)**
   - Filter generates correct Meilisearch syntax
   - Filter skips null/empty values
   - Filter chips display/remove correctly

2. **Integration Test (Vitest)**
   - Filter updates query parameters in URL
   - Multiple filters combine with `AND`
   - Clear filters resets all state

3. **E2E Test (Playwright)**
   - User can select filter option
   - Results update correctly
   - Filter persists on page refresh
   - Filter chip removes filter

---

## 📚 Documentation Updates Needed

1. **`CLAUDE.md`** (Lines 31-35)
   - Remove incorrect comment about `casting_time`, `range`, `duration` being "not filterable"
   - Add note about string field challenges (free-text values)

2. **`docs/CURRENT_STATUS.md`**
   - Update filter count (currently says "10 filters", should clarify "10/18 fields")

3. **`README.md`** (if exists)
   - Add "Sorting" to feature list

---

## 🎓 Insights

`★ Insight ─────────────────────────────────────`
**Why we prioritize sorting and level ranges:**

1. **Sorting is universally useful** - Every user benefits from browsing by level vs. name
2. **Level ranges match gameplay** - "Show me spells I can cast" is a core D&D question
3. **Low implementation cost** - Both features use existing infrastructure (API already supports them)

**Why we skip casting_time/range/duration filters:**
- **Free-text values** make dropdowns impractical (100+ unique values)
- **Better served by search** - Users can type "1 action" in search box
- **Low ROI** - Significant effort for niche use case
`─────────────────────────────────────────────────`

---

**Last Updated:** 2025-11-25
**Next Review:** After Phase 1 implementation (sorting + level ranges)
