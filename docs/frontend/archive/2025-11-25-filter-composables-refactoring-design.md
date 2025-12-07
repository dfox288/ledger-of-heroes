# Filter Composables Refactoring - Design Document

**Date:** 2025-11-25
**Status:** 🎯 Design Complete - Ready for Implementation
**Approach:** Conservative Extraction (Approach 1)
**Estimated Effort:** 4-6 hours (3 composables + tests + 6 page migrations)

---

## Executive Summary

Following yesterday's successful Meilisearch filter migration across all 6 entity pages, we identified ~500 lines of duplicate filter-building logic. This design proposes extracting three focused composables to eliminate duplication while maintaining flexibility and testability.

**Goals:**
- ✅ DRY up queryBuilder logic (80 lines → 20 lines per page)
- ✅ Standardize reference data fetching (5 lines → 1 line per fetch)
- ✅ Simplify active filter counting (12 lines → 1 line per page)
- ✅ Maintain TDD discipline throughout
- ✅ Low-risk, incremental migration

---

## Problem Analysis

### Current State

After yesterday's Meilisearch migration (commit `2407f2d`), all 6 entity pages follow an identical filter pattern:

**Pattern Identified:**
```typescript
// Repeated in: spells, items, monsters, races, classes, feats
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // Pattern 1: Boolean conversion (15+ occurrences)
  if (boolFilter.value !== null) {
    const boolValue = boolFilter.value === '1' || boolFilter.value === 'true'
    meilisearchFilters.push(`field_name = ${boolValue}`)
  }

  // Pattern 2: Multi-select IN filters (8+ occurrences)
  if (arrayFilter.value.length > 0) {
    const values = arrayFilter.value.join(', ')
    meilisearchFilters.push(`field_name IN [${values}]`)
  }

  // Pattern 3: Range queries (3+ occurrences)
  if (rangeFilter.value === '5-10') {
    meilisearchFilters.push('field >= 5 AND field <= 10')
  }

  // Pattern 4: Combine with AND (6 occurrences)
  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

**Duplication Metrics:**
- **Total lines:** ~500 lines of queryBuilder logic across 6 pages
- **Boolean conversions:** 15+ identical lines
- **Filter combining:** 6 identical blocks
- **Reference data fetching:** 8+ identical patterns (useAsyncData + apiFetch)
- **Active filter counting:** 6 manual implementations (5-12 lines each)

**Pain Points:**
1. Bug fixes require changing 6 files
2. New filter types require copying patterns
3. Easy to introduce inconsistencies
4. Verbose, hard to scan

---

## Design Overview

### Approach: Conservative Extraction

Extract only the most obvious duplications without changing page architecture. Create three focused composables:

1. **`useMeilisearchFilters()`** - Declarative filter builder
2. **`useReferenceData<T>()`** - Type-safe reference fetching
3. **`useFilterCount()`** - Active filter counter

**Why Conservative?**
- ✅ Low risk - doesn't change page structure
- ✅ Easy to review - small, focused changes
- ✅ Incremental adoption - migrate one page at a time
- ✅ Easy to test - pure functions
- ✅ Preserves flexibility - pages still control filter logic

---

## Composable 1: `useMeilisearchFilters()`

### Purpose
Extract the repetitive "build filter array → join with AND" pattern into a reusable, declarative API.

### API Design

**Type Definitions:**
```typescript
export type FilterType =
  | 'equals'           // field = value
  | 'boolean'          // field = true/false (auto-converts strings)
  | 'in'              // field IN [value1, value2]
  | 'range'           // field >= min AND field <= max
  | 'isEmpty'         // field IS EMPTY / IS NOT EMPTY
  | 'greaterThan'     // field > value

export interface FilterConfig {
  /** Vue ref containing the filter value */
  ref: Ref<any>

  /** Meilisearch field name */
  field: string

  /** Filter type (default: 'equals') */
  type?: FilterType

  /** For 'range' type: min/max refs */
  min?: Ref<number | null>
  max?: Ref<number | null>

  /** For 'equals' with lookup: transform value before filtering */
  transform?: (value: any) => string | number | null
}

export interface UseMeilisearchFiltersReturn {
  /** Computed params object with 'filter' property */
  queryParams: ComputedRef<Record<string, unknown>>
}
```

**Implementation Signature:**
```typescript
export function useMeilisearchFilters(
  filters: FilterConfig[]
): UseMeilisearchFiltersReturn
```

### Implementation Logic

**Core Algorithm:**
1. Loop through filter configs
2. Skip null/undefined/empty values
3. Build filter string based on type
4. Collect all filter strings in array
5. Join with ' AND ' and set params.filter
6. Return computed params object

**Filter Type Handlers:**

**1. Equals (`type: 'equals'`):**
```typescript
// field = value
// Supports transform for ID→code lookups
if (transformedValue !== null) {
  meilisearchFilters.push(`${config.field} = ${transformedValue}`)
}
```

**2. Boolean (`type: 'boolean'`):**
```typescript
// field = true/false
// Auto-converts '1', 'true', true → true
// Auto-converts '0', 'false', false → false
const boolValue = value === '1' || value === 'true' || value === true
meilisearchFilters.push(`${config.field} = ${boolValue}`)
```

**3. IN (`type: 'in'`):**
```typescript
// field IN [value1, value2, ...]
// Handles arrays
const values = Array.isArray(value) ? value : [value]
const joined = values.join(', ')
meilisearchFilters.push(`${config.field} IN [${joined}]`)
```

**4. Range (`type: 'range'`):**
```typescript
// field >= min AND field <= max
// Uses config.min and config.max refs
const conditions: string[] = []
if (min !== null) conditions.push(`${config.field} >= ${min}`)
if (max !== null) conditions.push(`${config.field} <= ${max}`)
meilisearchFilters.push(conditions.join(' AND '))
```

**5. isEmpty (`type: 'isEmpty'`):**
```typescript
// field IS EMPTY or field IS NOT EMPTY
// Value = true → IS EMPTY
// Value = false → IS NOT EMPTY
meilisearchFilters.push(
  isEmpty ? `${config.field} IS EMPTY` : `${config.field} IS NOT EMPTY`
)
```

**6. greaterThan (`type: 'greaterThan'`):**
```typescript
// field > value
meilisearchFilters.push(`${config.field} > ${value}`)
```

### Usage Examples

**Spells Page (Before/After):**

**BEFORE (80 lines):**
```typescript
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  if (selectedLevel.value !== null) {
    meilisearchFilters.push(`level = ${selectedLevel.value}`)
  }

  if (selectedSchool.value !== null) {
    const schoolCode = spellSchools.value?.find(s => s.id === selectedSchool.value)?.code
    if (schoolCode) {
      meilisearchFilters.push(`school_code = ${schoolCode}`)
    }
  }

  if (selectedClass.value !== null) {
    meilisearchFilters.push(`class_slugs IN [${selectedClass.value}]`)
  }

  if (concentrationFilter.value !== null) {
    const boolValue = concentrationFilter.value === '1' || concentrationFilter.value === 'true'
    meilisearchFilters.push(`concentration = ${boolValue}`)
  }

  if (ritualFilter.value !== null) {
    const boolValue = ritualFilter.value === '1' || ritualFilter.value === 'true'
    meilisearchFilters.push(`ritual = ${boolValue}`)
  }

  if (selectedDamageTypes.value.length > 0) {
    const codes = selectedDamageTypes.value.join(', ')
    meilisearchFilters.push(`damage_types IN [${codes}]`)
  }

  if (selectedSavingThrows.value.length > 0) {
    const codes = selectedSavingThrows.value.join(', ')
    meilisearchFilters.push(`saving_throws IN [${codes}]`)
  }

  if (verbalFilter.value !== null) {
    const boolValue = verbalFilter.value === '1' || verbalFilter.value === 'true'
    meilisearchFilters.push(`requires_verbal = ${boolValue}`)
  }

  if (somaticFilter.value !== null) {
    const boolValue = somaticFilter.value === '1' || somaticFilter.value === 'true'
    meilisearchFilters.push(`requires_somatic = ${boolValue}`)
  }

  if (materialFilter.value !== null) {
    const boolValue = materialFilter.value === '1' || materialFilter.value === 'true'
    meilisearchFilters.push(`requires_material = ${boolValue}`)
  }

  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

**AFTER (22 lines - 73% reduction!):**
```typescript
const { queryParams } = useMeilisearchFilters([
  { ref: selectedLevel, field: 'level' },
  {
    ref: selectedSchool,
    field: 'school_code',
    transform: (id) => spellSchools.value?.find(s => s.id === id)?.code || null
  },
  { ref: selectedClass, field: 'class_slugs', type: 'in' },
  { ref: concentrationFilter, field: 'concentration', type: 'boolean' },
  { ref: ritualFilter, field: 'ritual', type: 'boolean' },
  { ref: selectedDamageTypes, field: 'damage_types', type: 'in' },
  { ref: selectedSavingThrows, field: 'saving_throws', type: 'in' },
  { ref: verbalFilter, field: 'requires_verbal', type: 'boolean' },
  { ref: somaticFilter, field: 'requires_somatic', type: 'boolean' },
  { ref: materialFilter, field: 'requires_material', type: 'boolean' }
])

// Pass directly to useEntityList
const { ... } = useEntityList({
  endpoint: '/spells',
  cacheKey: 'spells-list',
  queryBuilder: queryParams,  // ← Direct pass-through
  seo: { ... }
})
```

**Monsters Page (Range Example):**

**BEFORE:**
```typescript
if (selectedCR.value === '0-4') {
  meilisearchFilters.push('challenge_rating >= 0 AND challenge_rating <= 4')
} else if (selectedCR.value === '5-10') {
  meilisearchFilters.push('challenge_rating >= 5 AND challenge_rating <= 10')
}
// ... more if/else
```

**AFTER:**
```typescript
// Option 1: Manual range (current approach, still works)
const crMinRef = computed(() => {
  if (selectedCR.value === '0-4') return 0
  if (selectedCR.value === '5-10') return 5
  // ...
  return null
})

const crMaxRef = computed(() => {
  if (selectedCR.value === '0-4') return 4
  if (selectedCR.value === '5-10') return 10
  // ...
  return null
})

const { queryParams } = useMeilisearchFilters([
  { field: 'challenge_rating', type: 'range', min: crMinRef, max: crMaxRef }
])

// Option 2: Keep as-is (special case handling)
// Range UI values are acceptable to handle manually
// Composable focuses on most common patterns
```

**Items Page (isEmpty Example):**

**BEFORE:**
```typescript
if (hasPrerequisites.value !== null) {
  const hasPrereq = hasPrerequisites.value === '1' || hasPrerequisites.value === 'true'
  if (hasPrereq) {
    meilisearchFilters.push('prerequisites IS NOT EMPTY')
  } else {
    meilisearchFilters.push('prerequisites IS EMPTY')
  }
}
```

**AFTER:**
```typescript
const { queryParams } = useMeilisearchFilters([
  { ref: hasPrerequisites, field: 'prerequisites', type: 'isEmpty' }
])
```

### Testing Requirements

**Test File:** `tests/composables/useMeilisearchFilters.test.ts`

**Test Coverage (25+ tests):**

1. **Equals Filter:**
   - Builds filter for single value
   - Skips null values
   - Handles transform function (ID→code lookup)
   - Handles numeric values
   - Handles string values

2. **Boolean Filter:**
   - Converts string '1' to true
   - Converts string 'true' to true
   - Converts string '0' to false
   - Converts string 'false' to false
   - Handles actual boolean values
   - Skips null values

3. **IN Filter:**
   - Builds IN filter for array
   - Skips empty arrays
   - Handles single value as array
   - Joins multiple values with comma

4. **Range Filter:**
   - Builds range with min and max
   - Builds range with only min
   - Builds range with only max
   - Skips when both null

5. **isEmpty Filter:**
   - Builds IS EMPTY for true value
   - Builds IS NOT EMPTY for false value
   - Converts string '1' to true (IS EMPTY)
   - Converts string '0' to false (IS NOT EMPTY)

6. **greaterThan Filter:**
   - Builds > filter for numeric value
   - Skips null values

7. **Multiple Filters:**
   - Combines multiple filters with AND
   - Skips inactive filters (nulls, empty arrays)
   - Handles all filter types together
   - Preserves order

8. **Edge Cases:**
   - Returns empty params when no active filters
   - Handles undefined refs
   - Handles empty string values
   - Reactive updates when refs change

### Migration Checklist (Per Page)

For each entity page migration:

- [ ] Replace queryBuilder computed with useMeilisearchFilters call
- [ ] Convert each filter condition to FilterConfig object
- [ ] Add transform functions for ID→code lookups
- [ ] Update useEntityList to use queryParams
- [ ] Run existing page tests - ensure no regressions
- [ ] Browser test filters - verify identical behavior
- [ ] Commit with migration message

---

## Composable 2: `useReferenceData<T>()`

### Purpose
Extract the repetitive "fetch reference data for filter options" pattern. Standardize reference entity fetching with type safety and pagination support.

### API Design

**Type Definitions:**
```typescript
export interface ReferenceDataOptions {
  /** Override cache key (default: endpoint-based) */
  cacheKey?: string

  /** Transform response (default: extracts .data property) */
  transform?: (data: T[]) => T[]

  /** Fetch multiple pages (for large datasets like classes) */
  pages?: number
}

export interface UseReferenceDataReturn<T> {
  /** Reference data array */
  data: ComputedRef<T[]>

  /** Fetch error */
  error: Ref<unknown>

  /** Fetch status */
  status: Ref<'idle' | 'pending' | 'success' | 'error'>

  /** Loading state */
  loading: ComputedRef<boolean>
}
```

**Implementation Signature:**
```typescript
export function useReferenceData<T>(
  endpoint: string,
  options?: ReferenceDataOptions
): UseReferenceDataReturn<T>
```

### Implementation Logic

**Core Algorithm:**
1. Generate cache key from endpoint (or use custom)
2. Check if multi-page fetch needed
3. If multi-page: fetch all pages in parallel, flatten results
4. If single-page: fetch once
5. Apply optional transform function
6. Return reactive data with loading/error states

**Multi-Page Fetch:**
```typescript
if (options.pages && options.pages > 1) {
  const pagePromises = Array.from({ length: options.pages }, (_, i) => {
    const page = i + 1
    return apiFetch<{ data: T[] }>(`${endpoint}?per_page=100&page=${page}`)
  })

  const responses = await Promise.all(pagePromises)
  const allData = responses.flatMap(r => r?.data || [])

  return options.transform ? options.transform(allData) : allData
}
```

**Single-Page Fetch:**
```typescript
const response = await apiFetch<{ data: T[] }>(endpoint)
const data = response?.data || []

return options.transform ? options.transform(data) : data
```

### Usage Examples

**Simple Fetch:**

**BEFORE:**
```typescript
const { data: spellSchools } = await useAsyncData<SpellSchool[]>('spell-schools', async () => {
  const response = await apiFetch<{ data: SpellSchool[] }>('/spell-schools')
  return response?.data || []
})
```

**AFTER:**
```typescript
const { data: spellSchools } = useReferenceData<SpellSchool>('/spell-schools')
```

**Multi-Page Fetch with Transform:**

**BEFORE:**
```typescript
const { data: classes } = await useAsyncData<CharacterClass[]>('classes-filter', async () => {
  const [page1, page2] = await Promise.all([
    apiFetch<{ data: CharacterClass[] }>('/classes?per_page=100&page=1'),
    apiFetch<{ data: CharacterClass[] }>('/classes?per_page=100&page=2')
  ])
  const allClasses = [...(page1?.data || []), ...(page2?.data || [])]
  return allClasses.filter(c => c.is_base_class === true)
})
```

**AFTER:**
```typescript
const { data: classes } = useReferenceData<CharacterClass>('/classes', {
  pages: 2,
  transform: (data) => data.filter(c => c.is_base_class === true)
})
```

**All Spells Page Fetches (Before/After):**

**BEFORE (30 lines):**
```typescript
const { data: spellSchools } = await useAsyncData<SpellSchool[]>('spell-schools', async () => {
  const response = await apiFetch<{ data: SpellSchool[] }>('/spell-schools')
  return response?.data || []
})

const { data: classes } = await useAsyncData<CharacterClass[]>('classes-filter', async () => {
  const [page1, page2] = await Promise.all([
    apiFetch<{ data: CharacterClass[] }>('/classes?per_page=100&page=1'),
    apiFetch<{ data: CharacterClass[] }>('/classes?per_page=100&page=2')
  ])
  const allClasses = [...(page1?.data || []), ...(page2?.data || [])]
  return allClasses.filter(c => c.is_base_class === true)
})

const { data: damageTypes } = await useAsyncData<DamageType[]>('damage-types', async () => {
  const response = await apiFetch<{ data: DamageType[] }>('/damage-types')
  return response?.data || []
})

const { data: abilityScores } = await useAsyncData<AbilityScore[]>('ability-scores', async () => {
  const response = await apiFetch<{ data: AbilityScore[] }>('/ability-scores')
  return response?.data || []
})
```

**AFTER (7 lines - 77% reduction!):**
```typescript
const { data: spellSchools } = useReferenceData<SpellSchool>('/spell-schools')

const { data: classes } = useReferenceData<CharacterClass>('/classes', {
  pages: 2,
  transform: (data) => data.filter(c => c.is_base_class === true)
})

const { data: damageTypes } = useReferenceData<DamageType>('/damage-types')

const { data: abilityScores } = useReferenceData<AbilityScore>('/ability-scores')
```

### Testing Requirements

**Test File:** `tests/composables/useReferenceData.test.ts`

**Test Coverage (15+ tests):**

1. **Basic Fetching:**
   - Fetches data from endpoint
   - Returns data array
   - Handles API errors
   - Sets loading state correctly
   - Uses default cache key

2. **Multi-Page Fetching:**
   - Fetches multiple pages in parallel
   - Flattens results into single array
   - Passes correct page params
   - Handles partial failures

3. **Transform Function:**
   - Applies transform to single-page results
   - Applies transform to multi-page results
   - Receives correct data format

4. **Custom Options:**
   - Uses custom cache key
   - Handles empty responses gracefully
   - Returns empty array for failed fetches

5. **Type Safety:**
   - Generic type parameter enforced
   - Return type matches generic
   - data.value is typed correctly

6. **Reactivity:**
   - data is reactive
   - loading updates correctly
   - error updates on failure

### Migration Checklist (Per Page)

For each reference data fetch:

- [ ] Replace useAsyncData + apiFetch with useReferenceData
- [ ] Add pages option if multi-page (e.g., classes)
- [ ] Move filter/transform logic to options.transform
- [ ] Verify type parameter matches entity type
- [ ] Run tests - ensure data loads correctly
- [ ] Browser test - verify filter options populate

---

## Composable 3: `useFilterCount()`

### Purpose
Extract the manual "count each active filter" logic into a simple utility that handles all value types automatically.

### API Design

**Implementation Signature:**
```typescript
export function useFilterCount(...refs: Ref<any>[]): ComputedRef<number>
```

**Returns:** Computed number representing active filter count

### Implementation Logic

**Core Algorithm:**
1. Accept variadic refs
2. Loop through each ref
3. Check if value is "active" (non-null, non-empty)
4. Increment count for active values
5. Return computed count

**Active Value Detection:**
```typescript
for (const ref of refs) {
  const value = ref.value

  // Skip null/undefined
  if (value === null || value === undefined) continue

  // Skip empty strings
  if (value === '') continue

  // Count non-empty arrays
  if (Array.isArray(value)) {
    if (value.length > 0) count++
    continue
  }

  // Count any other truthy value
  count++
}
```

### Usage Examples

**Spells Page:**

**BEFORE (12 lines):**
```typescript
const activeFilterCount = computed(() => {
  let count = 0
  if (selectedLevel.value !== null) count++
  if (selectedSchool.value !== null) count++
  if (selectedClass.value !== null) count++
  if (concentrationFilter.value !== null) count++
  if (ritualFilter.value !== null) count++
  if (selectedDamageTypes.value.length > 0) count++
  if (selectedSavingThrows.value.length > 0) count++
  if (verbalFilter.value !== null) count++
  if (somaticFilter.value !== null) count++
  if (materialFilter.value !== null) count++
  return count
})
```

**AFTER (1 line!):**
```typescript
const activeFilterCount = useFilterCount(
  selectedLevel,
  selectedSchool,
  selectedClass,
  concentrationFilter,
  ritualFilter,
  selectedDamageTypes,
  selectedSavingThrows,
  verbalFilter,
  somaticFilter,
  materialFilter
)
```

**Monsters Page:**

**BEFORE (6 lines):**
```typescript
const activeFilterCount = computed(() => {
  let count = 0
  if (selectedCR.value) count++
  if (selectedType.value) count++
  if (isLegendary.value !== null) count++
  return count
})
```

**AFTER (1 line!):**
```typescript
const activeFilterCount = useFilterCount(selectedCR, selectedType, isLegendary)
```

### Testing Requirements

**Test File:** `tests/composables/useFilterCount.test.ts`

**Test Coverage (15+ tests):**

1. **Single Value Refs:**
   - Counts non-null values
   - Skips null values
   - Skips undefined values
   - Skips empty strings
   - Counts zero as active
   - Counts false as active

2. **Array Refs:**
   - Counts non-empty arrays
   - Skips empty arrays
   - Handles single-item arrays

3. **Multiple Refs:**
   - Counts multiple active refs
   - Skips all inactive refs
   - Counts mixed active/inactive refs

4. **Reactivity:**
   - Updates when refs change
   - Decrements when filter cleared
   - Increments when filter added

5. **Edge Cases:**
   - Handles zero refs passed
   - Handles all null refs
   - Handles mixed types (string, number, boolean, array)

### Migration Checklist (Per Page)

For each active filter count:

- [ ] Replace manual computed with useFilterCount
- [ ] Pass all filter refs as arguments
- [ ] Remove manual count logic
- [ ] Run tests - ensure count updates correctly
- [ ] Browser test - verify badge count matches

---

## Migration Strategy

### Phase 1: Create Composables (TDD) - 2-3 hours

**Task 1.1: Create `useMeilisearchFilters()` (TDD)**
- [ ] Write test file: `tests/composables/useMeilisearchFilters.test.ts`
- [ ] Write ~25 tests covering all filter types (RED)
- [ ] Create file: `app/composables/useMeilisearchFilters.ts`
- [ ] Implement composable until all tests pass (GREEN)
- [ ] Refactor for clarity (REFACTOR)
- [ ] Commit: "feat: add useMeilisearchFilters composable"

**Task 1.2: Create `useReferenceData()` (TDD)**
- [ ] Write test file: `tests/composables/useReferenceData.test.ts`
- [ ] Write ~15 tests covering fetching/pagination/transform (RED)
- [ ] Create file: `app/composables/useReferenceData.ts`
- [ ] Implement composable until all tests pass (GREEN)
- [ ] Refactor for clarity (REFACTOR)
- [ ] Commit: "feat: add useReferenceData composable"

**Task 1.3: Create `useFilterCount()` (TDD)**
- [ ] Write test file: `tests/composables/useFilterCount.test.ts`
- [ ] Write ~15 tests covering all value types (RED)
- [ ] Create file: `app/composables/useFilterCount.ts`
- [ ] Implement composable until all tests pass (GREEN)
- [ ] Refactor for clarity (REFACTOR)
- [ ] Commit: "feat: add useFilterCount composable"

**Checkpoint:**
- ✅ 3 new composables created
- ✅ ~55 new tests passing
- ✅ No existing tests broken
- ✅ Ready for pilot migration

---

### Phase 2: Pilot Migration (Spells Page) - 1 hour

**Why Spells First?**
- Largest page (622 lines)
- Most complex filters (10 filters, multiple types)
- Best validation of composable design
- If this works, others will be easier

**Task 2.1: Migrate Spells Page**
- [ ] Read current spells/index.vue (622 lines)
- [ ] Replace queryBuilder with useMeilisearchFilters
- [ ] Replace reference fetches with useReferenceData
- [ ] Replace activeFilterCount with useFilterCount
- [ ] Update useEntityList to use new queryParams
- [ ] Verify code compiles (TypeScript check)

**Task 2.2: Test Spells Migration**
- [ ] Run existing spells tests: `npm run test -- spells`
- [ ] Verify all tests still pass
- [ ] Fix any test failures
- [ ] Run full test suite: `npm run test`
- [ ] Verify no regressions

**Task 2.3: Browser Verify Spells**
- [ ] Start dev server: `docker compose exec nuxt npm run dev`
- [ ] Navigate to http://localhost:3000/spells
- [ ] Test each filter type:
  - Level dropdown
  - School dropdown
  - Class dropdown
  - Concentration toggle
  - Ritual toggle
  - Damage types multi-select
  - Saving throws multi-select
  - Verbal component toggle
  - Somatic component toggle
  - Material component toggle
- [ ] Verify filter count badge
- [ ] Verify clear filters button
- [ ] Verify URL params update
- [ ] Verify results update correctly

**Task 2.4: Commit Spells Migration**
- [ ] Review changes: `git diff app/pages/spells/index.vue`
- [ ] Stage: `git add app/pages/spells/index.vue`
- [ ] Commit: "refactor: migrate spells page to filter composables"

**Checkpoint:**
- ✅ Spells page migrated
- ✅ All tests passing
- ✅ Browser verified
- ✅ Ready for rollout

**Decision Point:** If issues found, fix composables before proceeding to Phase 3

---

### Phase 3: Roll Out to Remaining Pages - 2-3 hours

**Migration Order:**
1. ✅ Spells (completed in Phase 2)
2. Items (384 lines, 5 filters)
3. Monsters (318 lines, 3 filters)
4. Races (251 lines, 1 filter)
5. Classes (248 lines, 2 filters)
6. Feats (215 lines, 1 filter)

**Why This Order?**
- Descending complexity (largest → smallest)
- More validation with complex pages first
- Builds confidence for simpler pages

**Task 3.1: Migrate Items Page**
- [ ] Replace queryBuilder with useMeilisearchFilters
- [ ] Replace reference fetch (item types) with useReferenceData
- [ ] Replace activeFilterCount with useFilterCount
- [ ] Run tests: `npm run test -- items`
- [ ] Browser verify filters work
- [ ] Commit: "refactor: migrate items page to filter composables"

**Task 3.2: Migrate Monsters Page**
- [ ] Replace queryBuilder with useMeilisearchFilters
- [ ] Note: CR range handling (keep manual or refactor?)
- [ ] Replace activeFilterCount with useFilterCount
- [ ] Run tests: `npm run test -- monsters`
- [ ] Browser verify filters work
- [ ] Commit: "refactor: migrate monsters page to filter composables"

**Task 3.3: Migrate Races Page**
- [ ] Replace queryBuilder with useMeilisearchFilters
- [ ] Replace reference fetch (sizes) with useReferenceData
- [ ] Replace activeFilterCount with useFilterCount
- [ ] Run tests: `npm run test -- races`
- [ ] Browser verify filters work
- [ ] Commit: "refactor: migrate races page to filter composables"

**Task 3.4: Migrate Classes Page**
- [ ] Replace queryBuilder with useMeilisearchFilters
- [ ] Replace activeFilterCount with useFilterCount
- [ ] Run tests: `npm run test -- classes`
- [ ] Browser verify filters work
- [ ] Commit: "refactor: migrate classes page to filter composables"

**Task 3.5: Migrate Feats Page**
- [ ] Replace queryBuilder with useMeilisearchFilters
- [ ] Replace activeFilterCount with useFilterCount
- [ ] Run tests: `npm run test -- feats`
- [ ] Browser verify filters work
- [ ] Commit: "refactor: migrate feats page to filter composables"

**Checkpoint:**
- ✅ All 6 entity pages migrated
- ✅ All tests passing
- ✅ All pages browser verified
- ✅ 6 clean commits
- ✅ Ready for documentation

---

### Phase 4: Documentation & Cleanup - 30 minutes

**Task 4.1: Update CLAUDE.md**
- [ ] Add filter composables section
- [ ] Document useMeilisearchFilters usage
- [ ] Document useReferenceData usage
- [ ] Document useFilterCount usage
- [ ] Add to "Best Practices" section
- [ ] Update patterns guide

**Task 4.2: Create Migration Guide**
- [ ] Create: `docs/FILTER-COMPOSABLES-MIGRATION-GUIDE.md`
- [ ] Document how to add filters to new entity pages
- [ ] Include code examples
- [ ] List common patterns
- [ ] Troubleshooting tips

**Task 4.3: Update CHANGELOG.md**
- [ ] Add refactoring entry under "Changed"
- [ ] List benefits (code reduction, consistency, testability)
- [ ] Note non-breaking change

**Task 4.4: Final Commit**
- [ ] Stage all documentation: `git add docs/ CLAUDE.md CHANGELOG.md`
- [ ] Commit with comprehensive message:

```
docs: Add filter composables documentation and migration guide

**Refactoring Summary:**
- Created 3 new composables for filter management
- Migrated all 6 entity pages (spells, items, monsters, races, classes, feats)
- Reduced duplicate code by ~500 lines (27% reduction in filter logic)
- Added 55 comprehensive tests (100% coverage)

**Composables Created:**
1. useMeilisearchFilters() - Declarative filter builder
2. useReferenceData<T>() - Type-safe reference fetching
3. useFilterCount() - Active filter counter

**Benefits:**
- Bug fixes in 1 place instead of 6
- Consistent filter behavior across all pages
- Easier to add new filter types
- Improved type safety
- Better testability

**Documentation:**
- Updated CLAUDE.md with new patterns
- Created comprehensive migration guide
- Updated CHANGELOG.md

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

## Expected Outcomes

### Code Reduction Metrics

**Per Page:**
- **Spells:** 622 → ~450 lines (27% reduction, 172 lines saved)
- **Items:** 384 → ~280 lines (27% reduction, 104 lines saved)
- **Monsters:** 318 → ~230 lines (28% reduction, 88 lines saved)
- **Races:** 251 → ~180 lines (28% reduction, 71 lines saved)
- **Classes:** 248 → ~180 lines (27% reduction, 68 lines saved)
- **Feats:** 215 → ~155 lines (28% reduction, 60 lines saved)

**Total Reduction:**
- **Before:** ~2,038 lines across 6 pages
- **After:** ~1,475 lines across 6 pages
- **Saved:** ~563 lines (27.6% reduction)
- **Plus:** 3 new composables (~500 lines of reusable code)

### Quality Improvements

**Testing:**
- ✅ +55 new composable tests (100% coverage)
- ✅ All existing page tests still passing
- ✅ Easier to test edge cases in isolation
- ✅ Regression protection via composable tests

**Type Safety:**
- ✅ Generic types for reference data
- ✅ FilterConfig enforces correct structure
- ✅ Compile-time errors for misconfigurations
- ✅ Better IDE autocomplete

**Maintainability:**
- ✅ Bug fixes in 1 place instead of 6
- ✅ New filter types added centrally
- ✅ Consistent behavior across pages
- ✅ Easier onboarding for new devs

**Readability:**
- ✅ Declarative filter configs vs imperative logic
- ✅ Less nesting, easier to scan
- ✅ Self-documenting filter types
- ✅ Clear separation of concerns

### Performance Impact

**No Performance Degradation:**
- Composables return computed refs (same as before)
- No additional watchers or reactive overhead
- Same number of API calls
- Same filter query generation
- Virtual DOM rendering identical

**Potential Improvements:**
- Shared reference data cache (useReferenceData caching)
- Fewer reactivity updates (computed optimization)

---

## Risk Assessment & Mitigation

### Risks

**1. Breaking Existing Functionality** (Medium Risk)
- **Mitigation:** TDD approach, comprehensive tests before migration
- **Mitigation:** Pilot migration on spells page first
- **Mitigation:** Browser verify each page after migration
- **Mitigation:** Each page is its own commit (easy to revert)

**2. Type Safety Issues** (Low Risk)
- **Mitigation:** Generic types enforce correctness
- **Mitigation:** TypeScript compilation before commit
- **Mitigation:** Test coverage includes type assertions

**3. Edge Case Handling** (Low Risk)
- **Mitigation:** Comprehensive test coverage (55+ tests)
- **Mitigation:** Test all filter types in isolation
- **Mitigation:** Real-world validation with spells page (most complex)

**4. Developer Confusion** (Low Risk)
- **Mitigation:** Comprehensive documentation
- **Mitigation:** Migration guide with examples
- **Mitigation:** Updated CLAUDE.md
- **Mitigation:** Clear commit messages

### Rollback Plan

**Per-Page Rollback:**
- Each migration is its own commit
- Can revert individual pages: `git revert <commit-hash>`
- Composables are additive (don't break existing code)

**Complete Rollback:**
- Revert commits in reverse order (Phase 4 → Phase 3 → Phase 2 → Phase 1)
- All original code preserved in git history
- No data loss or breaking changes

**Partial Adoption:**
- Can keep some pages migrated, others not
- Composables are optional, not required
- Pages can coexist with different approaches

---

## Future Enhancements (Out of Scope)

**Post-Refactoring Opportunities:**

1. **Auto-generating Filter UI** (Phase 2 of Hybrid Approach)
   - Create `<UiEntityFilterPanel>` component
   - Render filters from config
   - Further reduce boilerplate

2. **Filter Presets**
   - "High-level wizard spells"
   - "Magic weapons"
   - "Legendary monsters"

3. **Saved Filters**
   - localStorage persistence
   - User-defined filter combos
   - Quick apply

4. **Filter Analytics**
   - Track most-used filters
   - Identify popular combinations
   - Inform future filter additions

5. **Advanced Query Builder**
   - OR logic support
   - Nested conditions
   - Visual query builder UI

**These are intentionally excluded from current scope to minimize risk and complexity.**

---

## Success Criteria

**Phase 1 Success:**
- [ ] 3 composables created
- [ ] 55+ tests written and passing
- [ ] 100% test coverage on composables
- [ ] TypeScript compilation clean
- [ ] No existing tests broken

**Phase 2 Success:**
- [ ] Spells page migrated
- [ ] All spells tests passing
- [ ] Browser verification complete
- [ ] Filter behavior identical to before
- [ ] Commit pushed

**Phase 3 Success:**
- [ ] All 6 pages migrated
- [ ] All tests passing (1061/1088 or better)
- [ ] All pages browser verified
- [ ] 6 clean commits
- [ ] No regressions

**Phase 4 Success:**
- [ ] Documentation updated (CLAUDE.md, CHANGELOG.md)
- [ ] Migration guide created
- [ ] Final commit with comprehensive message
- [ ] Ready for production

**Overall Success:**
- [ ] ~500 lines of duplicate code eliminated
- [ ] Consistent filter patterns across all pages
- [ ] Improved testability and type safety
- [ ] Zero breaking changes
- [ ] Zero functionality regressions

---

## Questions for Review

### Design Questions

1. **FilterConfig API:** Is the declarative config approach clear enough? Any missing filter types?

2. **Transform Function:** Is inline transform (for ID→code lookups) acceptable, or should we extract those to helper functions?

3. **Range Filters:** Should monsters CR range handling stay manual, or should we create a dedicated range config helper?

4. **Error Handling:** Should composables throw errors or return error states? Current design returns error refs.

5. **Async/Await:** Should useReferenceData be async like useAsyncData, or return reactive refs?

### Implementation Questions

1. **Migration Order:** Agree with spells → items → monsters → races → classes → feats?

2. **Commit Granularity:** One commit per page migration, or batch smaller pages together?

3. **Browser Testing:** Test all filters on all pages, or spot-check after spells validation?

4. **Documentation Timing:** Update docs at end (Phase 4) or as we go?

---

## Timeline Estimate

**Total: 4-6 hours**

| Phase | Tasks | Time Estimate |
|-------|-------|---------------|
| Phase 1: Create Composables | 3 composables + 55 tests (TDD) | 2-3 hours |
| Phase 2: Pilot Migration | Spells page + testing | 1 hour |
| Phase 3: Rollout | 5 remaining pages + testing | 2-3 hours |
| Phase 4: Documentation | Docs + CHANGELOG + commit | 30 minutes |

**Confidence:** Medium-High
- TDD approach reduces risk
- Pilot migration validates design
- Incremental rollout allows course correction

---

## Appendix: Code Examples

### Full Composable Implementations

**(See Phase 6: Implementation Plan for complete code)**

### Full Test Suites

**(Test implementations will be written in Phase 1 following TDD)**

### Full Page Migrations

**(Before/after diffs will be generated during Phase 2-3)**

---

**End of Design Document**

**Status:** 🎯 Ready for Implementation
**Next Step:** Phase 5 (Worktree Setup) + Phase 6 (Implementation Planning)
**Approval Required:** Yes - Please review design before proceeding
