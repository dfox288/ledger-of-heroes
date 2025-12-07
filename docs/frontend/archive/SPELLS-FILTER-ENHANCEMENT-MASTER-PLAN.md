# Spells Page Filter Enhancement - Master Implementation Plan

**Date:** 2025-11-24
**Purpose:** Comprehensive roadmap for enhancing Spells list page with all available API filters
**Status:** 🎯 Planning Complete - Ready for Implementation
**Priority:** High - Template for all other entity pages

---

## Table of Contents

1. [Current State Analysis](#1-current-state-analysis)
2. [Spells Filter Implementation Roadmap](#2-spells-filter-implementation-roadmap)
3. [UI Component Requirements](#3-ui-component-requirements)
4. [Data Requirements](#4-data-requirements)
5. [Implementation Plan - Phase 1 (High Impact)](#5-implementation-plan---phase-1-high-impact)
6. [Implementation Plan - Phase 2 (Medium Impact)](#6-implementation-plan---phase-2-medium-impact)
7. [Implementation Plan - Phase 3 (Polish)](#7-implementation-plan---phase-3-polish)
8. [Testing Strategy](#8-testing-strategy)
9. [Success Metrics](#9-success-metrics)
10. [Reusability for Other Pages](#10-reusability-for-other-pages)

---

## 1. Current State Analysis

### What We Have Now

**Current Filters (5 of 29 parameters):**
- ✅ `level` (integer 0-9) - Dropdown selector with 11 options (Cantrip + 1st-9th)
- ✅ `school` (integer ID) - Dropdown selector from `/spell-schools` reference data
- ✅ `classes` (string slug) - Dropdown selector from `/classes` reference data (base classes only)
- ✅ `concentration` (string '0'/'1') - Tri-state toggle (All/Yes/No)
- ✅ `ritual` (string '0'/'1') - Tri-state toggle (All/Yes/No)

**Implementation Details:**
- **Component:** `/app/pages/spells/index.vue` (368 lines)
- **Pattern:** Uses `useEntityList` composable with custom `queryBuilder`
- **UI Components:** `USelectMenu` (dropdowns), `UiFilterToggle` (toggles), `UiFilterCollapse` (collapsible section)
- **Filter Chips:** Active filters shown as removable badges with entity-specific colors
- **URL Persistence:** All filters synced to query parameters (e.g., `?level=3&school=7&concentration=1`)

### Current API Utilization

**Spells Endpoint (`/api/v1/spells`):**
- **Total Parameters:** 29
- **Implemented:** 5
- **Utilization:** **17.2%** 🔴
- **Unused Opportunity:** 24 parameters

**Parameter Categories:**
1. **Search/Pagination:** `q`, `page`, `per_page`, `sort_by`, `sort_direction` (2/5 used)
2. **Basic Filters:** `level`, `school`, `classes` (3/3 used)
3. **Boolean Toggles:** `concentration`, `ritual`, `has_verbal`, `has_somatic`, `has_material`, `has_higher_levels` (2/6 used)
4. **Array Filters:** `filter[damage_type_codes][]`, `filter[saving_throw_codes][]`, `filter[tag_slugs][]` (0/3 used)
5. **Direct Field Filters:** `filter[casting_time]`, `filter[range]`, `filter[duration]` (0/3 used)
6. **Legacy Parameters:** Same as above without `filter[]` prefix (0/11 used - backward compatibility)

### Component Inventory

**Available Reusable Components:**
- ✅ `<UiFilterCollapse>` - Collapsible filter section with badge count
- ✅ `<UiFilterToggle>` - Tri-state toggle (All/Yes/No) with 23 tests
- ✅ `<UiFilterMultiSelect>` - Multi-select dropdown with search, checkboxes, count badge (30 tests)
- ✅ `<UiFilterRangeSlider>` - Dual-handle range slider (31 tests)
- ✅ `<USelectMenu>` - NuxtUI native dropdown (single-select)

**UI State Components:**
- ✅ `<UiListPageHeader>` - Title, count, loading indicator
- ✅ `<UiListSkeletonCards>` - Loading state (6 animated cards)
- ✅ `<UiListErrorState>` - Error handling with retry
- ✅ `<UiListEmptyState>` - Empty state with clear filters option
- ✅ `<UiListResultsCount>` - "1-24 of 150 spells" display
- ✅ `<UiListPagination>` - Pagination controls

---

## 2. Spells Filter Implementation Roadmap

### Phase 1: Multi-Select Filters (High Impact)
**Goal:** Add 3 new multi-select filters using `<UiFilterMultiSelect>` component
**API Utilization:** 17% → 28% (+11 percentage points)
**Estimated Time:** 3-4 hours
**Impact:** HIGH - Most requested features, immediate value

#### Filters to Add:
1. **Damage Types** (`filter[damage_type_codes][]`)
   - Component: `<UiFilterMultiSelect>`
   - Options: Fire, Cold, Lightning, Thunder, Acid, Poison, Necrotic, Radiant, Psychic, Force, Bludgeoning, Piercing, Slashing (13 types)
   - Data Source: `/api/v1/damage-types` endpoint
   - Use Case: "Show me all fire damage spells" / "Cold + Thunder damage for storm sorcerer"
   - Example: `?filter[damage_type_codes][]=F&filter[damage_type_codes][]=C`

2. **Saving Throws** (`filter[saving_throw_codes][]`)
   - Component: `<UiFilterMultiSelect>`
   - Options: STR, DEX, CON, INT, WIS, CHA (6 ability scores)
   - Data Source: `/api/v1/ability-scores` endpoint
   - Use Case: "Find spells targeting DEX saves" / "Spells bypassing high CON"
   - Example: `?filter[saving_throw_codes][]=DEX&filter[saving_throw_codes][]=WIS`

3. **Spell Tags** (`filter[tag_slugs][]`)
   - Component: `<UiFilterMultiSelect>`
   - Options: Dynamic from API (e.g., "healing", "summoning", "teleportation", "damage", "buff", "debuff")
   - Data Source: Extract from spell data or `/api/v1/tags` endpoint (if available)
   - Use Case: "Show healing spells" / "Find teleportation magic"
   - Example: `?filter[tag_slugs][]=healing&filter[tag_slugs][]=buff`

#### Success Criteria:
- ✅ All 3 filters render correctly
- ✅ Multi-selection works (checkboxes)
- ✅ Search within dropdown works
- ✅ Filter chips show selected items
- ✅ URL parameters update correctly (array format)
- ✅ Clear individual selections and "Clear All" work
- ✅ 30+ tests written FIRST (TDD) and passing

---

### Phase 2: Component Flags (Medium Impact)
**Goal:** Add 4 new toggle filters using existing `<UiFilterToggle>` component
**API Utilization:** 28% → 42% (+14 percentage points)
**Estimated Time:** 2-3 hours
**Impact:** MEDIUM - Useful for specific scenarios (silenced, restrained, etc.)

#### Filters to Add:
1. **Verbal Component** (`filter[has_verbal]`)
   - Component: `<UiFilterToggle>`
   - Options: All / Has Verbal / No Verbal
   - Use Case: "No verbal components (silenced condition)" / "Verbal-only for Subtle Spell metamagic"
   - Example: `?filter[has_verbal]=1` or `?filter[has_verbal]=0`

2. **Somatic Component** (`filter[has_somatic]`)
   - Component: `<UiFilterToggle>`
   - Options: All / Has Somatic / No Somatic
   - Use Case: "No somatic components (restrained/grappled)"
   - Example: `?filter[has_somatic]=0`

3. **Material Component** (`filter[has_material]`)
   - Component: `<UiFilterToggle>`
   - Options: All / Has Material / No Material
   - Example: `?filter[has_material]=1`

4. **Higher Level Scaling** (`filter[has_higher_levels]`)
   - Component: `<UiFilterToggle>`
   - Options: All / Has Scaling / No Scaling
   - Use Case: "Spells that scale with spell slot level"
   - Example: `?filter[has_higher_levels]=1`

#### UI Organization:
Group component filters together in a "Components" section:
```vue
<div class="space-y-2">
  <label class="text-sm font-medium">Spell Components</label>
  <div class="flex flex-wrap gap-4">
    <UiFilterToggle v-model="verbalFilter" label="Verbal (V)" />
    <UiFilterToggle v-model="somaticFilter" label="Somatic (S)" />
    <UiFilterToggle v-model="materialFilter" label="Material (M)" />
  </div>
</div>
```

#### Success Criteria:
- ✅ All 4 toggles work independently
- ✅ Tri-state logic correct (All/Yes/No)
- ✅ Filter chips show component requirements
- ✅ URL parameters update correctly
- ✅ 20+ tests written FIRST (TDD) and passing

---

### Phase 3: Advanced Filters (Polish)
**Goal:** Add direct field filters for specific use cases
**API Utilization:** 42% → 52% (+10 percentage points)
**Estimated Time:** 2-3 hours
**Impact:** LOW - Edge cases, advanced users

#### Filters to Add:
1. **Casting Time** (`filter[casting_time]`)
   - Component: `<USelectMenu>` or `<UInput>` with autocomplete
   - Options: "1 action", "1 bonus action", "1 reaction", "1 minute", "10 minutes", "1 hour", etc.
   - Data Source: Extract unique values from spell data
   - Use Case: "Only bonus action spells" / "Ritual spells (10+ minutes)"
   - Example: `?filter[casting_time]=1 bonus action`
   - **Note:** Exact string match, requires knowing exact format

2. **Range** (`filter[range]`)
   - Component: `<USelectMenu>` or `<UInput>` with autocomplete
   - Options: "Self", "Touch", "30 feet", "60 feet", "120 feet", "1 mile", "Sight", etc.
   - Use Case: "Long-range spells (120+ feet)" / "Touch spells only"
   - Example: `?filter[range]=120 feet`
   - **Note:** Exact string match

3. **Duration** (`filter[duration]`)
   - Component: `<USelectMenu>` or `<UInput>` with autocomplete
   - Options: "Instantaneous", "1 round", "1 minute", "10 minutes", "1 hour", "8 hours", "24 hours", etc.
   - Use Case: "Long-duration buffs (8+ hours)" / "Instantaneous damage"
   - Example: `?filter[duration]=8 hours`
   - **Note:** Exact string match

#### Implementation Considerations:
- **Challenge:** These are exact string matches, not flexible ranges
- **Solution 1:** Fetch unique values from API (e.g., `/api/v1/spells?per_page=9999` and extract distinct values)
- **Solution 2:** Hardcode common values (less flexible but faster)
- **Solution 3:** Defer to backend team for range-based filters (e.g., `min_range`, `max_range`)

#### Success Criteria:
- ✅ Dropdowns populated with actual values from API
- ✅ Exact match filtering works correctly
- ✅ Clear feedback when no results (suggest relaxing filters)
- ✅ 15+ tests written FIRST (TDD) and passing

---

### Phase 4: Sorting Enhancement (Quality of Life)
**Goal:** Add sorting options to list page header
**API Utilization:** 52% → 59% (+7 percentage points)
**Estimated Time:** 1-2 hours
**Impact:** MEDIUM - Improves browsing experience

#### Sorting Options:
1. **Sort By** (`sort_by`)
   - Options: `name`, `level`, `created_at`, `updated_at`
   - Default: `name` (alphabetical)
   - Component: `<USelectMenu>` in header
   - Example: `?sort_by=level`

2. **Sort Direction** (`sort_direction`)
   - Options: `asc`, `desc`
   - Default: `asc`
   - Component: Toggle button with icon (↑/↓) next to Sort By
   - Example: `?sort_by=level&sort_direction=desc`

#### UI Placement:
Add to `<UiListPageHeader>` component or as a separate sort bar above results:
```vue
<div class="flex items-center gap-2 mb-4">
  <span class="text-sm font-medium">Sort by:</span>
  <USelectMenu v-model="sortBy" :items="sortOptions" />
  <UButton
    :icon="sortDirection === 'asc' ? 'i-heroicons-arrow-up' : 'i-heroicons-arrow-down'"
    @click="toggleSortDirection"
  />
</div>
```

#### Success Criteria:
- ✅ Sort dropdown works
- ✅ Direction toggle works
- ✅ Results update correctly
- ✅ URL parameters update
- ✅ 10+ tests written FIRST (TDD) and passing

---

## 3. UI Component Requirements

### Existing Components (Ready to Use)

#### `<UiFilterMultiSelect>` ✅ NEW
**Location:** `/app/components/ui/filter/UiFilterMultiSelect.vue`
**Status:** Complete, 30 tests (60% passing - edge cases pending)

**Props API:**
```typescript
interface Props {
  modelValue: string[] | null | undefined  // Array of selected values
  label: string                            // Filter label
  options: Array<{ value: string, label: string }>  // Available options
  color?: string                           // Entity semantic color (default: 'primary')
  placeholder?: string                     // Placeholder text (default: 'Select...')
}
```

**Example Usage:**
```vue
<UiFilterMultiSelect
  v-model="selectedDamageTypes"
  label="Damage Types"
  :options="damageTypeOptions"
  color="spell"
  placeholder="Select damage types..."
/>
```

**Features:**
- Multi-select with checkboxes
- Searchable dropdown
- Count badge showing number of selections
- Clear button when items selected
- Accessible (ARIA labels, keyboard navigation)

---

#### `<UiFilterToggle>` ✅ EXISTING
**Location:** `/app/components/ui/filter/UiFilterToggle.vue`
**Status:** Complete, 23 tests (all passing)

**Props API:**
```typescript
interface Props {
  modelValue: string | null                // Current value ('1', '0', null)
  label: string                            // Filter label
  color?: string                           // Entity semantic color (default: 'primary')
  options: Array<{
    value: string | null,
    label: string
  }>                                       // Tri-state options
}
```

**Example Usage:**
```vue
<UiFilterToggle
  v-model="verbalFilter"
  label="Verbal"
  color="primary"
  :options="[
    { value: null, label: 'All' },
    { value: '1', label: 'Yes' },
    { value: '0', label: 'No' }
  ]"
/>
```

---

#### `<UiFilterCollapse>` ✅ EXISTING
**Location:** `/app/components/ui/filter/UiFilterCollapse.vue`
**Status:** Complete

**Props API:**
```typescript
interface Props {
  modelValue: boolean        // Open/closed state
  label: string              // "Filters"
  badgeCount?: number        // Active filter count
}
```

**Slots:**
- `#search` - Search input (before collapsible content)
- `#default` - Filter controls (inside collapsible)

**Example Usage:**
```vue
<UiFilterCollapse
  v-model="filtersOpen"
  label="Filters"
  :badge-count="activeFilterCount"
>
  <template #search>
    <UInput v-model="searchQuery" placeholder="Search spells..." />
  </template>
  <!-- Filters go here -->
</UiFilterCollapse>
```

---

### Components NOT Needed (Use NuxtUI Native)

#### `<USelectMenu>` - Single-Select Dropdown
**Already using for:** Level, School, Class filters
**Will use for:** Sort By, Casting Time, Range, Duration

#### `<UInput>` - Text Input
**Already using for:** Search query
**Could use for:** Advanced text-based filters (future)

---

## 4. Data Requirements

### Reference Data to Fetch

#### 1. Damage Types (`/api/v1/damage-types`)
**Endpoint:** `GET http://localhost:8080/api/v1/damage-types`
**Response Structure:**
```json
{
  "data": [
    { "id": 1, "code": "A", "name": "Acid" },
    { "id": 2, "code": "B", "name": "Bludgeoning" },
    { "id": 3, "code": "C", "name": "Cold" },
    { "id": 4, "code": "F", "name": "Fire" },
    { "id": 5, "code": "Fc", "name": "Force" },
    { "id": 6, "code": "L", "name": "Lightning" },
    { "id": 7, "code": "N", "name": "Necrotic" },
    { "id": 8, "code": "P", "name": "Piercing" },
    { "id": 9, "code": "Po", "name": "Poison" },
    { "id": 10, "code": "Ps", "name": "Psychic" },
    { "id": 11, "code": "R", "name": "Radiant" },
    { "id": 12, "code": "S", "name": "Slashing" },
    { "id": 13, "code": "T", "name": "Thunder" }
  ]
}
```

**Transform to Options:**
```typescript
const damageTypeOptions = computed(() => {
  if (!damageTypes.value) return []
  return damageTypes.value.map(type => ({
    value: type.code,  // Use code for API parameter
    label: type.name   // Display name
  }))
})
```

**Usage in Query:**
```typescript
// User selects: ["Fire", "Cold"]
// Model value: ["F", "C"]
// Query parameter: filter[damage_type_codes][]=F&filter[damage_type_codes][]=C
```

---

#### 2. Ability Scores (`/api/v1/ability-scores`)
**Endpoint:** `GET http://localhost:8080/api/v1/ability-scores`
**Response Structure:**
```json
{
  "data": [
    { "id": 1, "code": "STR", "name": "Strength" },
    { "id": 2, "code": "DEX", "name": "Dexterity" },
    { "id": 3, "code": "CON", "name": "Constitution" },
    { "id": 4, "code": "INT", "name": "Intelligence" },
    { "id": 5, "code": "WIS", "name": "Wisdom" },
    { "id": 6, "code": "CHA", "name": "Charisma" }
  ]
}
```

**Transform to Options:**
```typescript
const savingThrowOptions = computed(() => {
  if (!abilityScores.value) return []
  return abilityScores.value.map(ability => ({
    value: ability.code,  // Use code for API parameter
    label: ability.code   // Display code (STR, DEX, etc.)
  }))
})
```

**Usage in Query:**
```typescript
// User selects: ["DEX", "WIS"]
// Query parameter: filter[saving_throw_codes][]=DEX&filter[saving_throw_codes][]=WIS
```

---

#### 3. Spell Tags (To Be Determined)
**Potential Source 1:** Extract from existing spell data
```typescript
// Fetch all spells and extract unique tags
const allSpells = await apiFetch('/spells?per_page=9999')
const uniqueTags = [...new Set(allSpells.data.flatMap(spell => spell.tags || []))]
```

**Potential Source 2:** Dedicated endpoint (if available)
```typescript
const tags = await apiFetch('/tags?entity_type=spell')
```

**Expected Structure:**
```json
{
  "data": [
    { "id": 1, "slug": "healing", "name": "Healing" },
    { "id": 2, "slug": "damage", "name": "Damage" },
    { "id": 3, "slug": "buff", "name": "Buff" },
    { "id": 4, "slug": "debuff", "name": "Debuff" },
    { "id": 5, "slug": "summoning", "name": "Summoning" },
    { "id": 6, "slug": "teleportation", "name": "Teleportation" }
  ]
}
```

**Transform to Options:**
```typescript
const tagOptions = computed(() => {
  if (!tags.value) return []
  return tags.value.map(tag => ({
    value: tag.slug,   // Use slug for API parameter
    label: tag.name    // Display name
  }))
})
```

**Usage in Query:**
```typescript
// User selects: ["Healing", "Buff"]
// Model value: ["healing", "buff"]
// Query parameter: filter[tag_slugs][]=healing&filter[tag_slugs][]=buff
```

**⚠️ Action Required:** Verify tags endpoint exists or implement extraction logic

---

### Data Fetching Strategy

**Approach:** Use `useAsyncData` with caching (same pattern as existing School/Class filters)

```typescript
// Fetch damage types
const { data: damageTypes } = await useAsyncData<DamageType[]>(
  'damage-types',
  async () => {
    const response = await apiFetch<{ data: DamageType[] }>('/damage-types')
    return response.data
  }
)

// Fetch ability scores (for saving throws)
const { data: abilityScores } = await useAsyncData<AbilityScore[]>(
  'ability-scores',
  async () => {
    const response = await apiFetch<{ data: AbilityScore[] }>('/ability-scores')
    return response.data
  }
)

// Fetch tags (TBD - depends on endpoint availability)
const { data: tags } = await useAsyncData<Tag[]>(
  'spell-tags',
  async () => {
    const response = await apiFetch<{ data: Tag[] }>('/tags?entity_type=spell')
    return response.data
  }
)
```

**Benefits:**
- SSR-friendly (data fetched on server)
- Automatic caching (no re-fetch on navigation)
- Proper loading/error states

---

## 5. Implementation Plan - Phase 1 (High Impact)

### Step-by-Step: Damage Types + Saving Throws + Tags

**Estimated Time:** 3-4 hours
**Priority:** HIGH
**API Utilization Gain:** +11 percentage points (17% → 28%)

---

#### Step 1: Fetch Reference Data (30 minutes)

**Task:** Add data fetching for damage types, ability scores, and tags

**File:** `/app/pages/spells/index.vue`

**Changes:**
```typescript
// Add after existing useAsyncData calls (around line 24-28)

// Fetch damage types for filter options
const { data: damageTypes } = await useAsyncData<DamageType[]>('damage-types', async () => {
  const response = await apiFetch<{ data: DamageType[] }>('/damage-types')
  return response.data
})

// Fetch ability scores for saving throw filter
const { data: abilityScores } = await useAsyncData<AbilityScore[]>('ability-scores', async () => {
  const response = await apiFetch<{ data: AbilityScore[] }>('/ability-scores')
  return response.data
})

// TODO: Fetch spell tags (verify endpoint exists)
// const { data: spellTags } = await useAsyncData<Tag[]>('spell-tags', async () => {
//   const response = await apiFetch<{ data: Tag[] }>('/tags?entity_type=spell')
//   return response.data
// })
```

**Type Definitions:**
```typescript
// Add to /app/types/api/entities.ts or inline
interface DamageType {
  id: number
  code: string
  name: string
}

interface AbilityScore {
  id: number
  code: string
  name: string
}

interface Tag {
  id: number
  slug: string
  name: string
}
```

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 1 Reference Data', () => {
  it('fetches damage types on mount', async () => {
    // Test that useAsyncData is called with correct endpoint
  })

  it('fetches ability scores on mount', async () => {
    // Test that useAsyncData is called with correct endpoint
  })

  it('handles missing reference data gracefully', async () => {
    // Test fallback when API returns empty data
  })
})
```

---

#### Step 2: Create Filter State Refs (15 minutes)

**Task:** Add reactive refs for new filter selections

**File:** `/app/pages/spells/index.vue`

**Changes:**
```typescript
// Add after existing filter refs (around line 11-16)

// Phase 1 filters: Multi-select arrays
const selectedDamageTypes = ref<string[]>(
  route.query['filter[damage_type_codes][]']
    ? Array.isArray(route.query['filter[damage_type_codes][]'])
      ? route.query['filter[damage_type_codes][]'] as string[]
      : [route.query['filter[damage_type_codes][]'] as string]
    : []
)

const selectedSavingThrows = ref<string[]>(
  route.query['filter[saving_throw_codes][]']
    ? Array.isArray(route.query['filter[saving_throw_codes][]'])
      ? route.query['filter[saving_throw_codes][]'] as string[]
      : [route.query['filter[saving_throw_codes][]'] as string]
    : []
)

const selectedTags = ref<string[]>(
  route.query['filter[tag_slugs][]']
    ? Array.isArray(route.query['filter[tag_slugs][]'])
      ? route.query['filter[tag_slugs][]'] as string[]
      : [route.query['filter[tag_slugs][]'] as string]
    : []
)
```

**Note:** Array query parameters require careful parsing (single value vs array)

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 1 Filter State', () => {
  it('initializes damage types from URL array parameter', () => {
    // Test parsing ?filter[damage_type_codes][]=F&filter[damage_type_codes][]=C
  })

  it('initializes damage types from URL single parameter', () => {
    // Test parsing ?filter[damage_type_codes][]=F
  })

  it('defaults to empty array when no URL parameter', () => {
    // Test default state
  })
})
```

---

#### Step 3: Transform Reference Data to Options (15 minutes)

**Task:** Create computed properties for dropdown options

**File:** `/app/pages/spells/index.vue`

**Changes:**
```typescript
// Add after existing option transforms (around line 46-71)

// Damage type filter options
const damageTypeOptions = computed(() => {
  if (!damageTypes.value) return []
  return damageTypes.value.map(type => ({
    value: type.code,
    label: type.name
  }))
})

// Saving throw filter options (use codes as labels)
const savingThrowOptions = computed(() => {
  if (!abilityScores.value) return []
  return abilityScores.value.map(ability => ({
    value: ability.code,
    label: ability.code  // Display "STR", "DEX", etc.
  }))
})

// Tag filter options
const tagOptions = computed(() => {
  if (!spellTags.value) return []
  return spellTags.value.map(tag => ({
    value: tag.slug,
    label: tag.name
  }))
})
```

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 1 Filter Options', () => {
  it('transforms damage types to dropdown options', () => {
    // Test option structure { value: code, label: name }
  })

  it('returns empty array when damage types not loaded', () => {
    // Test fallback
  })

  it('transforms ability scores to saving throw options', () => {
    // Test option structure { value: code, label: code }
  })
})
```

---

#### Step 4: Add Filters to Query Builder (20 minutes)

**Task:** Include new filters in API query parameters

**File:** `/app/pages/spells/index.vue`

**Changes:**
```typescript
// Modify queryBuilder computed (around line 74-82)
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}

  // Existing filters
  if (selectedLevel.value !== null) params.level = selectedLevel.value
  if (selectedSchool.value !== null) params.school = selectedSchool.value
  if (selectedClass.value !== null) params.classes = selectedClass.value
  if (concentrationFilter.value !== null) params.concentration = concentrationFilter.value
  if (ritualFilter.value !== null) params.ritual = ritualFilter.value

  // Phase 1: Array filters (multi-select)
  if (selectedDamageTypes.value.length > 0) {
    params['filter[damage_type_codes][]'] = selectedDamageTypes.value
  }
  if (selectedSavingThrows.value.length > 0) {
    params['filter[saving_throw_codes][]'] = selectedSavingThrows.value
  }
  if (selectedTags.value.length > 0) {
    params['filter[tag_slugs][]'] = selectedTags.value
  }

  return params
})
```

**⚠️ Important:** Array parameters use `[]` suffix in key name

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 1 Query Builder', () => {
  it('includes damage type codes in query when selected', () => {
    // Test params['filter[damage_type_codes][]'] = ['F', 'C']
  })

  it('excludes damage types when none selected', () => {
    // Test empty array → no parameter
  })

  it('builds correct query with multiple array filters', () => {
    // Test all Phase 1 filters together
  })
})
```

---

#### Step 5: Add UI Components to Template (30 minutes)

**Task:** Integrate `<UiFilterMultiSelect>` components

**File:** `/app/pages/spells/index.vue`

**Changes:**
```vue
<!-- Add after existing filters (around line 224) -->
<template>
  <!-- ... existing filter section ... -->

  <!-- Phase 1: Multi-Select Filters -->
  <div class="space-y-4 mt-6">
    <h3 class="text-sm font-semibold text-gray-700 dark:text-gray-300">
      Advanced Filters
    </h3>

    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      <!-- Damage Types -->
      <UiFilterMultiSelect
        v-model="selectedDamageTypes"
        label="Damage Types"
        :options="damageTypeOptions"
        color="spell"
        placeholder="Select damage types..."
      />

      <!-- Saving Throws -->
      <UiFilterMultiSelect
        v-model="selectedSavingThrows"
        label="Saving Throws"
        :options="savingThrowOptions"
        color="spell"
        placeholder="Select saving throws..."
      />

      <!-- Spell Tags -->
      <UiFilterMultiSelect
        v-if="tagOptions.length > 0"
        v-model="selectedTags"
        label="Tags"
        :options="tagOptions"
        color="spell"
        placeholder="Select tags..."
      />
    </div>
  </div>
</template>
```

**Layout:** 3-column grid on desktop, stacks on mobile

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 1 UI Components', () => {
  it('renders damage types multi-select', () => {
    // Test component exists
  })

  it('passes correct props to multi-select', () => {
    // Test modelValue, label, options, color
  })

  it('hides tags filter when no options available', () => {
    // Test v-if conditional
  })
})
```

---

#### Step 6: Add Filter Chips (30 minutes)

**Task:** Show active selections as removable chips

**File:** `/app/pages/spells/index.vue`

**Changes:**
```vue
<!-- Add to filter chips section (around line 256-315) -->
<template>
  <!-- ... existing filter chips ... -->

  <!-- Phase 1: Damage Type Chips -->
  <template v-if="selectedDamageTypes.length > 0">
    <UButton
      v-for="code in selectedDamageTypes"
      :key="code"
      size="xs"
      color="spell"
      variant="soft"
      @click="removeDamageType(code)"
    >
      {{ getDamageTypeName(code) }} ✕
    </UButton>
  </template>

  <!-- Phase 1: Saving Throw Chips -->
  <template v-if="selectedSavingThrows.length > 0">
    <UButton
      v-for="code in selectedSavingThrows"
      :key="code"
      size="xs"
      color="info"
      variant="soft"
      @click="removeSavingThrow(code)"
    >
      {{ code }} Save ✕
    </UButton>
  </template>

  <!-- Phase 1: Tag Chips -->
  <template v-if="selectedTags.length > 0">
    <UButton
      v-for="slug in selectedTags"
      :key="slug"
      size="xs"
      color="primary"
      variant="soft"
      @click="removeTag(slug)"
    >
      {{ getTagName(slug) }} ✕
    </UButton>
  </template>
</template>
```

**Helper Functions:**
```typescript
// Add helper functions for chip labels
const getDamageTypeName = (code: string) => {
  return damageTypes.value?.find(t => t.code === code)?.name || code
}

const getTagName = (slug: string) => {
  return spellTags.value?.find(t => t.slug === slug)?.name || slug
}

// Remove individual selections
const removeDamageType = (code: string) => {
  selectedDamageTypes.value = selectedDamageTypes.value.filter(c => c !== code)
}

const removeSavingThrow = (code: string) => {
  selectedSavingThrows.value = selectedSavingThrows.value.filter(c => c !== code)
}

const removeTag = (slug: string) => {
  selectedTags.value = selectedTags.value.filter(s => s !== slug)
}
```

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 1 Filter Chips', () => {
  it('displays chip for each selected damage type', () => {
    // Test chip rendering
  })

  it('removes damage type when chip clicked', () => {
    // Test removeDamageType function
  })

  it('displays damage type name in chip', () => {
    // Test getDamageTypeName lookup
  })
})
```

---

#### Step 7: Update Clear Filters Function (10 minutes)

**Task:** Reset Phase 1 filters when "Clear All" clicked

**File:** `/app/pages/spells/index.vue`

**Changes:**
```typescript
// Modify clearFilters function (around line 109-116)
const clearFilters = () => {
  // Clear base filters (from composable)
  clearBaseFilters()

  // Clear existing filters
  selectedLevel.value = null
  selectedSchool.value = null
  selectedClass.value = null
  concentrationFilter.value = null
  ritualFilter.value = null

  // Clear Phase 1 filters
  selectedDamageTypes.value = []
  selectedSavingThrows.value = []
  selectedTags.value = []
}
```

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 1 Clear Filters', () => {
  it('resets all Phase 1 filters to empty arrays', () => {
    // Test clearFilters function
  })

  it('removes Phase 1 filter chips', () => {
    // Test chip removal
  })
})
```

---

#### Step 8: Update Active Filter Count (10 minutes)

**Task:** Include Phase 1 filters in collapse badge count

**File:** `/app/pages/spells/index.vue`

**Changes:**
```typescript
// Modify activeFilterCount computed (around line 132-140)
const activeFilterCount = computed(() => {
  let count = 0

  // Existing filters
  if (selectedLevel.value !== null) count++
  if (selectedSchool.value !== null) count++
  if (selectedClass.value !== null) count++
  if (concentrationFilter.value !== null) count++
  if (ritualFilter.value !== null) count++

  // Phase 1 filters (count number of selections, not just presence)
  count += selectedDamageTypes.value.length
  count += selectedSavingThrows.value.length
  count += selectedTags.value.length

  return count
})
```

**Note:** Badge shows total selections (e.g., 2 damage types + 1 saving throw = 3)

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 1 Filter Count', () => {
  it('includes damage type selections in count', () => {
    // Test count += selectedDamageTypes.length
  })

  it('includes all Phase 1 selections in count', () => {
    // Test total count with multiple filters
  })
})
```

---

#### Step 9: Write Tests (TDD - Write FIRST!) (60 minutes)

**Test File:** `/tests/pages/spells-phase1.test.ts`

**Test Categories:**
1. **Reference Data Fetching** (5 tests)
   - Fetch damage types on mount
   - Fetch ability scores on mount
   - Fetch spell tags on mount (if endpoint exists)
   - Handle API errors gracefully
   - Handle empty reference data

2. **Filter State Management** (8 tests)
   - Initialize from URL parameters (array format)
   - Initialize from URL parameters (single value)
   - Default to empty arrays
   - Update model values on selection
   - Update URL on value change
   - Clear individual selections
   - Clear all selections

3. **Query Builder** (6 tests)
   - Include damage types in query
   - Include saving throws in query
   - Include tags in query
   - Exclude filters when empty
   - Build correct array parameter format
   - Combine with existing filters

4. **UI Components** (8 tests)
   - Render multi-select components
   - Pass correct props
   - Hide tags when no options
   - Show filter chips
   - Remove chips on click
   - Display correct chip labels
   - Update badge count
   - Clear all filters

5. **Integration** (3 tests)
   - API receives correct query parameters
   - Results update when filters change
   - URL syncs with filter state

**Total Tests:** ~30 tests

**TDD Workflow:**
1. Write test → Run test (RED)
2. Write minimal code → Run test (GREEN)
3. Refactor → Run test (still GREEN)
4. Commit

---

#### Step 10: Browser Verification (30 minutes)

**Manual Testing Checklist:**

1. **Basic Functionality**
   - [ ] Damage Types dropdown opens
   - [ ] Can select multiple damage types
   - [ ] Search within dropdown works
   - [ ] Saving Throws dropdown opens
   - [ ] Can select multiple saving throws
   - [ ] Tags dropdown opens (if data available)
   - [ ] Can select multiple tags

2. **Filter Behavior**
   - [ ] Results update when filters change
   - [ ] URL parameters update correctly
   - [ ] Filter chips appear
   - [ ] Can remove individual chips
   - [ ] "Clear All" resets Phase 1 filters
   - [ ] Badge count includes Phase 1 selections

3. **Edge Cases**
   - [ ] Works with no selections
   - [ ] Works with 1 selection
   - [ ] Works with all options selected
   - [ ] Handles missing reference data gracefully
   - [ ] Combines with existing filters correctly

4. **Cross-Browser**
   - [ ] Chrome (latest)
   - [ ] Firefox (latest)
   - [ ] Safari (latest)
   - [ ] Mobile Safari (iOS)
   - [ ] Chrome Mobile (Android)

5. **Accessibility**
   - [ ] Keyboard navigation works
   - [ ] Screen reader announces filters
   - [ ] Focus visible
   - [ ] ARIA labels correct

6. **Dark Mode**
   - [ ] All filters visible in dark mode
   - [ ] No contrast issues
   - [ ] Chips readable

---

#### Step 11: Commit Work (5 minutes)

**Commit Message:**
```
feat: Add damage type, saving throw, and tag filters to Spells page

- Add UiFilterMultiSelect components for 3 new filters
- Fetch damage types, ability scores, and spell tags reference data
- Implement array query parameter handling
- Add filter chips with individual removal
- Update badge count to include multi-select filters
- 30 tests (all passing)

Phase 1 of Spells filter enhancement roadmap.
API utilization: 17% → 28% (+11 percentage points)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
```

**Files Changed:**
- `app/pages/spells/index.vue`
- `tests/pages/spells-phase1.test.ts`
- `CHANGELOG.md` (add entry)

---

## 6. Implementation Plan - Phase 2 (Medium Impact)

### Step-by-Step: Component Flags

**Estimated Time:** 2-3 hours
**Priority:** MEDIUM
**API Utilization Gain:** +14 percentage points (28% → 42%)

---

#### Overview

Add 4 new toggle filters using existing `<UiFilterToggle>` component:
1. Verbal Component (`filter[has_verbal]`)
2. Somatic Component (`filter[has_somatic]`)
3. Material Component (`filter[has_material]`)
4. Higher Level Scaling (`filter[has_higher_levels]`)

**Key Difference from Phase 1:** These use existing component, faster implementation

---

#### Step 1: Create Filter State Refs (10 minutes)

**File:** `/app/pages/spells/index.vue`

**Changes:**
```typescript
// Add after Phase 1 filter refs
const verbalFilter = ref<string | null>((route.query['filter[has_verbal]'] as string) || null)
const somaticFilter = ref<string | null>((route.query['filter[has_somatic]'] as string) || null)
const materialFilter = ref<string | null>((route.query['filter[has_material]'] as string) || null)
const higherLevelsFilter = ref<string | null>((route.query['filter[has_higher_levels]'] as string) || null)
```

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 2 Filter State', () => {
  it('initializes component filters from URL', () => {})
  it('defaults to null when no URL parameter', () => {})
})
```

---

#### Step 2: Add to Query Builder (10 minutes)

**File:** `/app/pages/spells/index.vue`

**Changes:**
```typescript
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}

  // ... existing filters ...

  // Phase 2: Component flags
  if (verbalFilter.value !== null) params['filter[has_verbal]'] = verbalFilter.value
  if (somaticFilter.value !== null) params['filter[has_somatic]'] = somaticFilter.value
  if (materialFilter.value !== null) params['filter[has_material]'] = materialFilter.value
  if (higherLevelsFilter.value !== null) params['filter[has_higher_levels]'] = higherLevelsFilter.value

  return params
})
```

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 2 Query Builder', () => {
  it('includes component filters in query', () => {})
  it('excludes filters when null', () => {})
})
```

---

#### Step 3: Add UI Components (20 minutes)

**File:** `/app/pages/spells/index.vue`

**Changes:**
```vue
<template>
  <!-- Add after Phase 1 filters -->
  <div class="space-y-4 mt-6">
    <h3 class="text-sm font-semibold text-gray-700 dark:text-gray-300">
      Spell Components
    </h3>

    <div class="flex flex-wrap gap-4">
      <!-- Verbal Component -->
      <UiFilterToggle
        v-model="verbalFilter"
        label="Verbal (V)"
        color="primary"
        :options="[
          { value: null, label: 'All' },
          { value: '1', label: 'Yes' },
          { value: '0', label: 'No' }
        ]"
      />

      <!-- Somatic Component -->
      <UiFilterToggle
        v-model="somaticFilter"
        label="Somatic (S)"
        color="primary"
        :options="[
          { value: null, label: 'All' },
          { value: '1', label: 'Yes' },
          { value: '0', label: 'No' }
        ]"
      />

      <!-- Material Component -->
      <UiFilterToggle
        v-model="materialFilter"
        label="Material (M)"
        color="primary"
        :options="[
          { value: null, label: 'All' },
          { value: '1', label: 'Yes' },
          { value: '0', label: 'No' }
        ]"
      />
    </div>
  </div>

  <!-- Higher Levels Scaling (separate section) -->
  <div class="space-y-4 mt-6">
    <h3 class="text-sm font-semibold text-gray-700 dark:text-gray-300">
      Spell Scaling
    </h3>

    <UiFilterToggle
      v-model="higherLevelsFilter"
      label="At Higher Levels"
      color="primary"
      :options="[
        { value: null, label: 'All' },
        { value: '1', label: 'Has Scaling' },
        { value: '0', label: 'No Scaling' }
      ]"
    />
  </div>
</template>
```

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 2 UI', () => {
  it('renders component toggles', () => {})
  it('renders higher levels toggle', () => {})
  it('passes correct props', () => {})
})
```

---

#### Step 4: Add Filter Chips (15 minutes)

**File:** `/app/pages/spells/index.vue`

**Changes:**
```vue
<!-- Add to chips section -->
<UButton
  v-if="verbalFilter !== null"
  size="xs"
  color="primary"
  variant="soft"
  @click="verbalFilter = null"
>
  Verbal: {{ verbalFilter === '1' ? 'Yes' : 'No' }} ✕
</UButton>

<UButton
  v-if="somaticFilter !== null"
  size="xs"
  color="primary"
  variant="soft"
  @click="somaticFilter = null"
>
  Somatic: {{ somaticFilter === '1' ? 'Yes' : 'No' }} ✕
</UButton>

<UButton
  v-if="materialFilter !== null"
  size="xs"
  color="primary"
  variant="soft"
  @click="materialFilter = null"
>
  Material: {{ materialFilter === '1' ? 'Yes' : 'No' }} ✕
</UButton>

<UButton
  v-if="higherLevelsFilter !== null"
  size="xs"
  color="primary"
  variant="soft"
  @click="higherLevelsFilter = null"
>
  Scaling: {{ higherLevelsFilter === '1' ? 'Yes' : 'No' }} ✕
</UButton>
```

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 2 Chips', () => {
  it('displays component filter chips', () => {})
  it('removes filter when chip clicked', () => {})
})
```

---

#### Step 5: Update Clear Filters & Badge Count (10 minutes)

**Changes:**
```typescript
const clearFilters = () => {
  // ... existing ...

  // Phase 2
  verbalFilter.value = null
  somaticFilter.value = null
  materialFilter.value = null
  higherLevelsFilter.value = null
}

const activeFilterCount = computed(() => {
  let count = 0
  // ... existing ...

  // Phase 2
  if (verbalFilter.value !== null) count++
  if (somaticFilter.value !== null) count++
  if (materialFilter.value !== null) count++
  if (higherLevelsFilter.value !== null) count++

  return count
})
```

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 2 Helpers', () => {
  it('clears Phase 2 filters', () => {})
  it('includes Phase 2 in badge count', () => {})
})
```

---

#### Step 6: Write Tests (20 tests, 45 minutes)

**Test Categories:**
1. State Management (4 tests)
2. Query Builder (4 tests)
3. UI Components (6 tests)
4. Filter Chips (3 tests)
5. Integration (3 tests)

**Total Tests:** ~20 tests

---

#### Step 7: Browser Verification (20 minutes)

**Checklist:**
- [ ] All 4 toggles work
- [ ] Tri-state logic correct
- [ ] Results update
- [ ] URL syncs
- [ ] Chips display/remove
- [ ] Badge count updates
- [ ] Clear all works
- [ ] Dark mode OK

---

#### Step 8: Commit Work (5 minutes)

**Commit Message:**
```
feat: Add component flags to Spells page filters

- Add Verbal, Somatic, Material component toggles
- Add "At Higher Levels" scaling filter
- Group component filters in dedicated section
- Add filter chips for component filters
- 20 tests (all passing)

Phase 2 of Spells filter enhancement roadmap.
API utilization: 28% → 42% (+14 percentage points)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

## 7. Implementation Plan - Phase 3 (Polish)

### Step-by-Step: Direct Field Filters

**Estimated Time:** 2-3 hours
**Priority:** LOW
**API Utilization Gain:** +10 percentage points (42% → 52%)

---

#### Overview

Add 3 direct field filters:
1. Casting Time (`filter[casting_time]`)
2. Range (`filter[range]`)
3. Duration (`filter[duration]`)

**Challenge:** These require exact string matches, need to populate dropdowns with actual values from API

---

#### Step 1: Fetch Unique Values from API (45 minutes)

**Approach:** Fetch all spells and extract unique values

**File:** `/app/pages/spells/index.vue`

**Changes:**
```typescript
// Fetch all spells to extract unique filter values
const { data: allSpells } = await useAsyncData<Spell[]>('all-spells-filters', async () => {
  const response = await apiFetch<{ data: Spell[] }>('/spells?per_page=9999')
  return response.data
})

// Extract unique casting times
const castingTimeOptions = computed(() => {
  if (!allSpells.value) return []
  const unique = [...new Set(allSpells.value.map(s => s.casting_time))]
  return [
    { value: null, label: 'All Casting Times' },
    ...unique.sort().map(time => ({ value: time, label: time }))
  ]
})

// Extract unique ranges
const rangeOptions = computed(() => {
  if (!allSpells.value) return []
  const unique = [...new Set(allSpells.value.map(s => s.range))]
  return [
    { value: null, label: 'All Ranges' },
    ...unique.sort().map(range => ({ value: range, label: range }))
  ]
})

// Extract unique durations
const durationOptions = computed(() => {
  if (!allSpells.value) return []
  const unique = [...new Set(allSpells.value.map(s => s.duration))]
  return [
    { value: null, label: 'All Durations' },
    ...unique.sort().map(duration => ({ value: duration, label: duration }))
  ]
})
```

**⚠️ Performance Consideration:** Fetching 9999 spells on page load adds ~500KB payload

**Alternative:** Hardcode common values (faster but less flexible):
```typescript
const castingTimeOptions = [
  { value: null, label: 'All Casting Times' },
  { value: '1 action', label: '1 Action' },
  { value: '1 bonus action', label: '1 Bonus Action' },
  { value: '1 reaction', label: '1 Reaction' },
  { value: '1 minute', label: '1 Minute' },
  { value: '10 minutes', label: '10 Minutes' },
  { value: '1 hour', label: '1 Hour' },
  { value: '8 hours', label: '8 Hours' },
  { value: '12 hours', label: '12 Hours' },
  { value: '24 hours', label: '24 Hours' }
]
```

**Recommendation:** Start with dynamic extraction, optimize later if needed

**Tests to Write FIRST (TDD):**
```typescript
describe('Spells - Phase 3 Unique Values', () => {
  it('extracts unique casting times', () => {})
  it('extracts unique ranges', () => {})
  it('extracts unique durations', () => {})
  it('sorts options alphabetically', () => {})
})
```

---

#### Step 2: Create Filter State & Query Builder (15 minutes)

**Changes:**
```typescript
// Filter refs
const castingTimeFilter = ref<string | null>((route.query['filter[casting_time]'] as string) || null)
const rangeFilter = ref<string | null>((route.query['filter[range]'] as string) || null)
const durationFilter = ref<string | null>((route.query['filter[duration]'] as string) || null)

// Query builder
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  // ... existing ...

  // Phase 3
  if (castingTimeFilter.value !== null) params['filter[casting_time]'] = castingTimeFilter.value
  if (rangeFilter.value !== null) params['filter[range]'] = rangeFilter.value
  if (durationFilter.value !== null) params['filter[duration]'] = durationFilter.value

  return params
})
```

**Tests:** (10 tests)

---

#### Step 3: Add UI Components (20 minutes)

**Changes:**
```vue
<div class="grid grid-cols-1 md:grid-cols-3 gap-4 mt-6">
  <!-- Casting Time -->
  <USelectMenu
    v-model="castingTimeFilter"
    :items="castingTimeOptions"
    value-key="value"
    placeholder="All Casting Times"
    size="md"
  />

  <!-- Range -->
  <USelectMenu
    v-model="rangeFilter"
    :items="rangeOptions"
    value-key="value"
    placeholder="All Ranges"
    size="md"
  />

  <!-- Duration -->
  <USelectMenu
    v-model="durationFilter"
    :items="durationOptions"
    value-key="value"
    placeholder="All Durations"
    size="md"
  />
</div>
```

**Tests:** (6 tests)

---

#### Step 4: Add Chips, Clear, Badge (15 minutes)

**Chips:**
```vue
<UButton v-if="castingTimeFilter" @click="castingTimeFilter = null">
  {{ castingTimeFilter }} ✕
</UButton>
<!-- Same for range and duration -->
```

**Clear & Badge:**
```typescript
const clearFilters = () => {
  // ...
  castingTimeFilter.value = null
  rangeFilter.value = null
  durationFilter.value = null
}

const activeFilterCount = computed(() => {
  // ...
  if (castingTimeFilter.value !== null) count++
  if (rangeFilter.value !== null) count++
  if (durationFilter.value !== null) count++
  return count
})
```

**Tests:** (4 tests)

---

#### Step 5: Write Tests, Verify, Commit (60 minutes)

**Total Tests:** ~15 tests
**Browser Verification:** 15 minutes
**Commit:** 5 minutes

**Commit Message:**
```
feat: Add casting time, range, and duration filters to Spells page

- Extract unique values from all spells
- Add dropdown selectors for direct field filters
- Add filter chips for field filters
- 15 tests (all passing)

Phase 3 of Spells filter enhancement roadmap.
API utilization: 42% → 52% (+10 percentage points)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

## 8. Testing Strategy

### Test Coverage Goals

**Target:** 80+ new tests across all phases

**Breakdown:**
- Phase 1 (Multi-Select): ~30 tests
- Phase 2 (Component Flags): ~20 tests
- Phase 3 (Direct Fields): ~15 tests
- Phase 4 (Sorting): ~10 tests
- **Total:** ~75 tests

---

### Test Categories

#### 1. Unit Tests (Component Behavior)

**Test Files:**
- `/tests/pages/spells-phase1.test.ts`
- `/tests/pages/spells-phase2.test.ts`
- `/tests/pages/spells-phase3.test.ts`

**What to Test:**
- Filter state initialization from URL
- Filter state updates on user interaction
- Query builder includes correct parameters
- Clear filters resets all values
- Badge count calculation
- Helper functions (label lookups, removals)

**Example:**
```typescript
describe('Spells - Damage Type Filter', () => {
  it('initializes from URL array parameter', () => {
    const route = {
      query: { 'filter[damage_type_codes][]': ['F', 'C'] }
    }
    // Test that selectedDamageTypes = ['F', 'C']
  })

  it('updates query builder when damage types selected', () => {
    selectedDamageTypes.value = ['F', 'C']
    const query = queryBuilder.value
    expect(query['filter[damage_type_codes][]']).toEqual(['F', 'C'])
  })

  it('removes individual damage type', () => {
    selectedDamageTypes.value = ['F', 'C', 'L']
    removeDamageType('C')
    expect(selectedDamageTypes.value).toEqual(['F', 'L'])
  })
})
```

---

#### 2. Integration Tests (Full Page Flow)

**Test Files:**
- `/tests/pages/spells-integration.test.ts`

**What to Test:**
- Page mounts successfully with new filters
- Reference data fetches on mount
- Filter interactions trigger API calls
- Multiple filters work together
- URL syncs with filter state
- Results update when filters change

**Example:**
```typescript
describe('Spells - Integration', () => {
  it('fetches reference data on mount', async () => {
    const wrapper = await mountSuspended(SpellsPage)
    await flushPromises()

    // Verify API calls
    expect(apiFetch).toHaveBeenCalledWith('/damage-types')
    expect(apiFetch).toHaveBeenCalledWith('/ability-scores')
  })

  it('updates results when damage type filter changes', async () => {
    const wrapper = await mountSuspended(SpellsPage)

    // Select fire damage
    const multiSelect = wrapper.findComponent(UiFilterMultiSelect)
    await multiSelect.vm.$emit('update:modelValue', ['F'])

    // Verify API called with filter
    expect(apiFetch).toHaveBeenCalledWith('/spells', {
      query: expect.objectContaining({
        'filter[damage_type_codes][]': ['F']
      })
    })
  })
})
```

---

#### 3. Component Tests (UI Rendering)

**Test Files:**
- `/tests/components/ui/filter/UiFilterMultiSelect.test.ts` (already exists - 30 tests)

**What to Test:**
- Component renders with props
- Multi-select interactions work
- Search within dropdown works
- Clear button appears/works
- Badge count updates
- Emits correct events

**Note:** Component tests already written in Phase 1C & 2, no new tests needed unless bugs found

---

### Browser Testing Checklist

**Manual Verification (30 minutes per phase):**

#### Functional Testing
1. **Filter Interactions**
   - [ ] All filters render correctly
   - [ ] Can select/deselect options
   - [ ] Search within multi-select works
   - [ ] Toggles cycle through states correctly
   - [ ] Results update when filters change
   - [ ] Loading state shows during fetch

2. **URL Synchronization**
   - [ ] URL updates when filters change
   - [ ] Back/forward browser buttons work
   - [ ] Can share URL with filters
   - [ ] Page loads with URL filters applied

3. **Filter Management**
   - [ ] Filter chips appear
   - [ ] Can remove individual chips
   - [ ] "Clear All" resets all filters
   - [ ] Badge count is accurate
   - [ ] Collapsible section works

#### Visual Testing
4. **Responsiveness**
   - [ ] Mobile (375px): Filters stack vertically
   - [ ] Tablet (768px): 2-column grid
   - [ ] Desktop (1440px): 3-column grid
   - [ ] No horizontal scroll on mobile
   - [ ] Touch targets large enough (44x44px minimum)

5. **Dark Mode**
   - [ ] All filters visible in dark mode
   - [ ] Dropdown menus readable
   - [ ] Chips have good contrast
   - [ ] No white backgrounds on dark theme

6. **Cross-Browser**
   - [ ] Chrome (latest)
   - [ ] Firefox (latest)
   - [ ] Safari (latest)
   - [ ] Mobile Safari
   - [ ] Chrome Mobile

#### Accessibility Testing
7. **Keyboard Navigation**
   - [ ] Tab through all filters
   - [ ] Enter/Space to open dropdowns
   - [ ] Arrow keys to navigate options
   - [ ] Escape to close dropdowns
   - [ ] Focus visible on all interactive elements

8. **Screen Reader**
   - [ ] Filter labels announced
   - [ ] Selection counts announced
   - [ ] Chip removal announced
   - [ ] ARIA labels correct

#### Edge Cases
9. **Error Handling**
   - [ ] Handles missing reference data
   - [ ] Handles API errors gracefully
   - [ ] Shows fallback when no results
   - [ ] Doesn't break with invalid URL params

10. **Performance**
    - [ ] No lag when typing in search
    - [ ] Fast filter updates (<100ms)
    - [ ] No memory leaks on repeated use
    - [ ] Large datasets don't freeze UI

---

### Automated Testing Commands

```bash
# Run all tests
docker compose exec nuxt npm run test

# Run tests in watch mode
docker compose exec nuxt npm run test:watch

# Run specific test file
docker compose exec nuxt npm run test tests/pages/spells-phase1.test.ts

# Run tests with coverage
docker compose exec nuxt npm run test -- --coverage

# Type checking
docker compose exec nuxt npm run typecheck

# Linting
docker compose exec nuxt npm run lint
```

---

## 9. Success Metrics

### API Utilization Tracking

**Current State (Before Enhancement):**
```
Spells Endpoint: /api/v1/spells
Total Parameters: 29
Implemented: 5 (level, school, classes, concentration, ritual)
Utilization: 17.2% 🔴
```

**Phase 1 Target (Multi-Select Filters):**
```
Added: 3 (damage_type_codes[], saving_throw_codes[], tag_slugs[])
Total Implemented: 8
Utilization: 27.6% 🟡 (+10.4 percentage points)
```

**Phase 2 Target (Component Flags):**
```
Added: 4 (has_verbal, has_somatic, has_material, has_higher_levels)
Total Implemented: 12
Utilization: 41.4% 🟢 (+13.8 percentage points)
```

**Phase 3 Target (Direct Fields):**
```
Added: 3 (casting_time, range, duration)
Total Implemented: 15
Utilization: 51.7% 🟢 (+10.3 percentage points)
```

**Phase 4 Target (Sorting):**
```
Added: 2 (sort_by, sort_direction)
Total Implemented: 17
Utilization: 58.6% 🟢 (+6.9 percentage points)
```

**Full Implementation (All Phases):**
```
Added: 12 new filters
Total Implemented: 17 of 29
Utilization: 58.6% 🟢 (+41.4 percentage points from baseline)
```

**⚠️ Note:** Remaining 12 parameters are mostly legacy/duplicate parameters (same as modern params without `filter[]` prefix). Actual modern API coverage: **~85%**

---

### User Experience Metrics

**Goals:**
1. **Faster Results** - Users find spells 30% faster with advanced filters
2. **Higher Engagement** - 50% increase in filter usage (track via analytics)
3. **Lower Bounce Rate** - Users stay on page longer exploring results
4. **Better Conversions** - More clicks to detail pages (relevant results)

**How to Measure:**
- Add analytics events for filter interactions
- Track time to first spell click
- Monitor filter usage patterns (which filters most used)
- Survey users: "Did you find what you were looking for?"

---

### Code Quality Metrics

**Targets:**
- ✅ 100% TypeScript compilation (0 errors)
- ✅ 100% ESLint passing (0 errors)
- ✅ 80% test coverage for new code
- ✅ 100% test pass rate
- ✅ <500ms page load time
- ✅ <100ms filter update latency
- ✅ Accessibility: WCAG 2.1 AA compliance

---

### Comparison Table: Before vs After

| Metric | Before | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Improvement |
|--------|--------|---------|---------|---------|---------|-------------|
| **API Parameters Used** | 5 | 8 | 12 | 15 | 17 | +240% |
| **Utilization %** | 17% | 28% | 41% | 52% | 59% | +42 pts |
| **Filter Components** | 3 types | 4 types | 4 types | 4 types | 4 types | +1 type |
| **Total Tests** | ~30 | ~60 | ~80 | ~95 | ~105 | +75 tests |
| **Lines of Code** | 368 | ~480 | ~560 | ~620 | ~650 | +77% |
| **User Value** | Basic | High | High | Medium | Medium | 🚀 |

---

## 10. Reusability for Other Pages

### Pattern Replication Guide

**This Spells implementation serves as the master template for:**
1. Items page (similar multi-select needs)
2. Monsters page (CR range, alignment, size)
3. Races page (ability bonuses, languages, proficiencies)
4. Classes page (hit die, spellcasting, proficiencies)
5. Backgrounds page (skills, languages, proficiencies)
6. Feats page (prerequisites, granted bonuses)

---

### Items Page: Filter Mapping

**Phase 1 Equivalent (Multi-Select):**
- ✅ Item Spells (`filter[spells][]`) → Multi-select with spell search
- ✅ Granted Proficiencies (`filter[proficiencies][]`) → Multi-select
- ✅ Item Tags (`filter[tag_slugs][]`) → Multi-select

**Phase 2 Equivalent (Toggles):**
- ✅ Has Charges (`filter[has_charges]`) → Already implemented!
- ✅ Has Prerequisites (`filter[has_prerequisites]`) → Already implemented!
- ⚠️ Has Attunement (`filter[requires_attunement]`) → Add toggle

**Phase 3 Equivalent (Direct Fields):**
- Minimum Strength (`filter[min_strength]`) → Number input or slider
- Armor Class (`filter[armor_class]`) → Number input
- Damage Dice (`filter[damage_dice]`) → Dropdown with common dice

**Estimated Time:** 2-3 hours (most components already built)

---

### Monsters Page: Filter Mapping

**Phase 1 Equivalent (Multi-Select):**
- ✅ Size (`filter[size_codes][]`) → Multi-select (Tiny, Small, Medium, Large, Huge, Gargantuan)
- ✅ Alignment (`filter[alignment_codes][]`) → Multi-select or alignment grid
- ✅ Languages (`filter[languages][]`) → Multi-select
- ✅ Damage Immunities/Resistances/Vulnerabilities → Multi-select

**Phase 2 Equivalent (Range Slider):**
- ⚠️ **CR Range** (`min_cr`, `max_cr`) → **Use `<UiFilterRangeSlider>`!** 🎯
- ⚠️ **HP Range** (`min_hp`, `max_hp`) → Range slider
- ⚠️ **Speed Range** → Range slider

**Phase 3 Equivalent (Toggles):**
- ✅ Is Legendary (`filter[is_legendary]`) → Already implemented!
- Has Lair Actions (`filter[has_lair_actions]`) → Toggle
- Has Legendary Resistance (`filter[has_legendary_resistance]`) → Toggle

**Key Insight:** Monsters is the PERFECT use case for `<UiFilterRangeSlider>` (CR 0-30 range)

**Estimated Time:** 3-4 hours

---

### Races Page: Filter Mapping

**Phase 1 Equivalent (Multi-Select):**
- ⚠️ **Ability Score Bonuses** (`filter[ability_bonuses][]`) → Multi-select (STR, DEX, CON, INT, WIS, CHA)
- ⚠️ **Languages** (`filter[languages][]`) → Multi-select
- ⚠️ **Granted Skills** (`filter[skills][]`) → Multi-select
- ⚠️ **Granted Proficiencies** (`filter[proficiencies][]`) → Multi-select

**Phase 2 Equivalent (Toggles):**
- ✅ Has Darkvision (`filter[has_darkvision]`) → Already implemented!
- Has Innate Spells (`filter[has_innate_spells]`) → Toggle
- Grants Languages (`filter[grants_languages]`) → Toggle

**Phase 3 Equivalent (Range/Dropdown):**
- ✅ Size (`filter[size]`) → Already implemented (but backend doesn't support yet per CLAUDE.md)
- Minimum Speed (`filter[min_speed]`) → Slider (25-40 ft)
- Language Choice Count (`filter[language_choice_count]`) → Dropdown (0, 1, 2, 3+)

**Estimated Time:** 3-4 hours

---

### Classes Page: Filter Mapping

**Phase 1 Equivalent (Multi-Select):**
- ⚠️ **Hit Die** (`filter[hit_die][]`) → Multi-select (d6, d8, d10, d12)
- ⚠️ **Saving Throw Proficiencies** (`filter[saving_throws][]`) → Multi-select (STR, DEX, CON, INT, WIS, CHA)
- ⚠️ **Granted Skills** (`filter[skills][]`) → Multi-select
- ⚠️ **Granted Proficiencies** (`filter[proficiencies][]`) → Multi-select

**Phase 2 Equivalent (Toggles):**
- ✅ Is Base Class (`filter[is_base_class]`) → Already implemented!
- ✅ Is Spellcaster (`filter[is_spellcaster]`) → Already implemented!

**Phase 3 Equivalent (Dropdown):**
- Max Spell Level (`filter[max_spell_level]`) → Dropdown (0, 1-9)
- Spellcasting Ability (`filter[spellcasting_ability]`) → Dropdown (INT, WIS, CHA)

**Estimated Time:** 2-3 hours

---

### Backgrounds Page: Filter Mapping

**Phase 1 Equivalent (Multi-Select):**
- ⚠️ **Granted Skills** (`filter[skills][]`) → Multi-select
- ⚠️ **Languages** (`filter[languages][]`) → Multi-select
- ⚠️ **Granted Proficiencies** (`filter[proficiencies][]`) → Multi-select

**Phase 2 Equivalent (Toggles):**
- Grants Languages (`filter[grants_languages]`) → Toggle

**Phase 3 Equivalent (Dropdown):**
- Language Choice Count (`filter[language_choice_count]`) → Dropdown (0, 1, 2, 3+)

**Estimated Time:** 2-3 hours

---

### Feats Page: Filter Mapping

**Phase 1 Equivalent (Multi-Select):**
- ⚠️ **Prerequisite Ability** (`filter[prerequisite_abilities][]`) → Multi-select (STR, DEX, CON, INT, WIS, CHA)
- ⚠️ **Prerequisite Race** (`filter[prerequisite_races][]`) → Multi-select
- ⚠️ **Granted Proficiencies** (`filter[proficiencies][]`) → Multi-select
- ⚠️ **Granted Skills** (`filter[skills][]`) → Multi-select

**Phase 2 Equivalent (Toggles):**
- ✅ Has Prerequisites (`filter[has_prerequisites]`) → Already implemented!

**Phase 3 Equivalent (Dropdown):**
- Minimum Ability Value (`filter[min_ability_value]`) → Dropdown (13, 14, 15, 16, etc.)

**Estimated Time:** 2-3 hours

---

### Reusable Component Summary

**Components Built (Spells Implementation):**
- ✅ `<UiFilterMultiSelect>` - Ready for all multi-select filters
- ✅ `<UiFilterToggle>` - Ready for all toggle filters
- ✅ `<UiFilterRangeSlider>` - Ready for CR, HP, speed ranges
- ✅ `<UiFilterCollapse>` - Ready for all filter sections
- ✅ `<USelectMenu>` - Native NuxtUI component

**No New Components Needed!** 🎉

**Total Remaining Work:**
- 6 entity pages × 2-4 hours each = **12-24 hours total**
- With this roadmap: **Minimal learning curve, maximum reuse**

---

### Cross-Entity Filter Patterns

**Pattern 1: Multi-Select with Reference Data**
```vue
<!-- Reusable for: damage types, ability scores, sizes, alignments, languages, etc. -->
<UiFilterMultiSelect
  v-model="selectedItems"
  :label="filterLabel"
  :options="referenceDataOptions"
  :color="entityColor"
  :placeholder="placeholderText"
/>
```

**Pattern 2: Tri-State Toggle**
```vue
<!-- Reusable for: all boolean filters (has_X, is_X, requires_X) -->
<UiFilterToggle
  v-model="filterValue"
  :label="filterLabel"
  :color="entityColor"
  :options="[
    { value: null, label: 'All' },
    { value: '1', label: 'Yes' },
    { value: '0', label: 'No' }
  ]"
/>
```

**Pattern 3: Range Slider**
```vue
<!-- Reusable for: CR, HP, speed, level, etc. -->
<UiFilterRangeSlider
  v-model="rangeValue"
  :label="filterLabel"
  :min="minValue"
  :max="maxValue"
  :step="stepValue"
  :color="entityColor"
  :format-label="customFormatter"
/>
```

**Pattern 4: Query Builder with Arrays**
```typescript
// Reusable for all entity pages
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}

  // Single-value filters
  if (singleFilter.value !== null) params.filterName = singleFilter.value

  // Array filters (multi-select)
  if (arrayFilter.value.length > 0) params['filter[array_name][]'] = arrayFilter.value

  return params
})
```

---

## Summary

This master plan provides:
1. ✅ **Complete API analysis** - All 29 parameters documented
2. ✅ **Phased implementation** - 4 phases from high to low priority
3. ✅ **Detailed steps** - Step-by-step for each phase
4. ✅ **TDD requirements** - Test-first approach enforced
5. ✅ **Reusable components** - All components already built
6. ✅ **Cross-entity template** - Guide for 6 other entity pages
7. ✅ **Success metrics** - Clear targets for utilization and quality
8. ✅ **Realistic estimates** - 8-12 hours total for Spells page

**Next Steps:**
1. Review this plan with team/stakeholders
2. Prioritize phases based on user feedback
3. Execute Phase 1 first (highest impact)
4. Use lessons learned to optimize Phases 2-4
5. Replicate pattern to other entity pages
6. Achieve **60%+ API utilization across all entities!**

**Impact:**
- Spells page: 17% → 59% API utilization (+42 points)
- Template for 6 other pages: ~120 new filters total
- Overall project: 23% → 60%+ API utilization

🎯 **This is the roadmap to a best-in-class D&D filtering experience!**
