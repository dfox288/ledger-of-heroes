# useEntityList Composable - Design Document

**Date:** 2025-11-21
**Status:** Approved - Ready for Implementation
**Impact:** ~510 lines of code reduction across 6 list pages
**Pattern:** Strategy Pattern (callback-based query building)

---

## Problem Statement

All 6 list pages (`/spells`, `/items`, `/races`, `/classes`, `/backgrounds`, `/feats`) contain **85-95% duplicate code**:

- Identical data fetching with `useAsyncData`
- Identical pagination state management
- Identical URL query param sync logic
- Identical SEO setup patterns
- Identical computed values (`meta`, `totalResults`, etc.)

**Only difference:** Entity-specific filters (spell level, item rarity, etc.)

**Current State:** ~136 lines per page × 6 pages = **816 lines of duplicated logic**
**After Refactor:** ~50 lines per page × 6 pages = **300 lines** (510 lines saved)

---

## Solution Overview

Create a **generic `useEntityList` composable** that:

✅ Handles all shared list page concerns (pagination, search, URL sync, SEO)
✅ Accepts custom filter logic via callback pattern
✅ Returns reactive state + methods for pages to consume
✅ Maintains full type safety with TypeScript
✅ Preserves SSR support via `useAsyncData`

**Architecture:** Strategy Pattern - composable defines the algorithm, pages inject custom query building logic.

---

## Design Decisions

### 1. Callback-Based Query Building (Strategy Pattern)

**Approach:** Pages pass a `computed` callback that builds custom query parameters.

**Why:**
- Clean separation of concerns (composable = generic, page = specific)
- Fully reactive (computed callback tracks filter changes automatically)
- Testable in isolation
- No "magic" - explicit and predictable

**Alternative Rejected:** Config object with watchers - too implicit, harder to debug.

### 2. Composable Manages Base State Only

**Base State (Composable Owns):**
- `searchQuery` - All pages have search
- `currentPage` - All pages have pagination

**Custom State (Page Owns):**
- Entity-specific filters (level, school, rarity, etc.)
- Filter UI state (expanded/collapsed, etc.)

**Why:** Keeps composable generic and reusable. Pages maintain full control over their unique filters.

### 3. Automatic URL Sync

**Strategy:** Composable automatically syncs state ↔ URL query params.

**Bidirectional:**
- **Mount:** Read `route.query` → Initialize state
- **Change:** Watch state → Update URL via `navigateTo`

**Why:**
- All 6 pages do this identically
- Improves UX (shareable URLs, back button works)
- Eliminates 30+ lines of boilerplate per page

### 4. clearFilters is Split Responsibility

**Composable provides:** `clearFilters()` - clears base state (search, page)
**Page provides:** `clearAllFilters()` - clears base + custom filters

**Why:** Composable doesn't know about entity-specific filters, so it can't clear them.

**Pattern:**
```typescript
const clearAllFilters = () => {
  composable.clearFilters()      // Clear search + reset page
  selectedLevel.value = null      // Clear custom filters
  selectedSchool.value = null
}
```

---

## API Design

### Configuration Interface

```typescript
interface UseEntityListConfig {
  /** API endpoint (e.g., '/spells', '/items') */
  endpoint: string

  /** Cache key for useAsyncData (e.g., 'spells-list') */
  cacheKey: string

  /** Computed callback that builds custom query params */
  queryBuilder: ComputedRef<Record<string, any>>

  /** Items per page (default: 24) */
  perPage?: number

  /** SEO metadata */
  seo: {
    title: string          // 'Spells - D&D 5e Compendium'
    description: string    // SEO description
  }

  /** Parse URL params on mount (default: true) */
  initialRoute?: boolean
}
```

### Return Interface

```typescript
interface UseEntityListReturn {
  // Base State (composable-managed)
  searchQuery: Ref<string>
  currentPage: Ref<number>

  // Data (from API)
  data: ComputedRef<Array<any>>
  meta: ComputedRef<PaginationMeta | null>
  totalResults: ComputedRef<number>
  loading: Ref<boolean>
  error: Ref<any>

  // Methods
  refresh: () => Promise<void>
  clearFilters: () => void

  // Computed Helpers
  hasActiveFilters: ComputedRef<boolean>
}
```

### Function Signature

```typescript
export function useEntityList(config: UseEntityListConfig): UseEntityListReturn
```

---

## Implementation Details

### 1. Query Params Builder

The composable merges **base params** + **custom params**:

```typescript
const queryParams = computed(() => {
  const params: Record<string, any> = {
    per_page: config.perPage ?? 24,
    page: currentPage.value,
  }

  // Add search if present
  if (searchQuery.value.trim()) {
    params.q = searchQuery.value.trim()
  }

  // Merge custom filters from page's queryBuilder
  Object.assign(params, config.queryBuilder.value)

  return params
})
```

### 2. Data Fetching with useAsyncData

```typescript
const { data: response, pending, error, refresh } = await useAsyncData(
  config.cacheKey,
  async () => {
    const result = await apiFetch(config.endpoint, {
      query: queryParams.value
    })
    return result
  },
  {
    watch: [queryParams]  // Auto-refetch when params change
  }
)
```

**Key:** `watch: [queryParams]` makes it reactive. When `queryBuilder` changes → `queryParams` changes → refetch happens automatically.

### 3. URL Sync Implementation

**On Mount (Route → State):**
```typescript
const route = useRoute()

if (config.initialRoute !== false) {
  searchQuery.value = (route.query.q as string) || ''
  currentPage.value = route.query.page ? Number(route.query.page) : 1
}
```

**On Change (State → Route):**
```typescript
watch([currentPage, searchQuery, config.queryBuilder], () => {
  const query: Record<string, any> = {}

  if (currentPage.value > 1) {
    query.page = currentPage.value.toString()
  }

  if (searchQuery.value) {
    query.q = searchQuery.value
  }

  // Merge custom filters
  Object.assign(query, config.queryBuilder.value)

  navigateTo({ query }, { replace: true })
})
```

**Note:** Pages must initialize their own filter state from `route.query`.

### 4. SEO Setup

```typescript
useSeoMeta({
  title: config.seo.title,
  description: config.seo.description,
})

useHead({
  title: config.seo.title,
})
```

### 5. Computed Helpers

```typescript
const data = computed(() => response.value?.data || [])
const meta = computed(() => response.value?.meta || null)
const totalResults = computed(() => meta.value?.total || 0)

const hasActiveFilters = computed(() => {
  return searchQuery.value.trim() !== '' ||
         Object.keys(config.queryBuilder.value).length > 0
})
```

---

## Usage Examples

### Example 1: Simple List (Classes - Search Only)

**Before (89 lines):**
```typescript
const searchQuery = ref('')
const currentPage = ref(1)
const perPage = 24

const queryParams = computed(() => {
  const params: Record<string, any> = {
    per_page: perPage,
    page: currentPage.value,
  }
  if (searchQuery.value.trim()) {
    params.q = searchQuery.value.trim()
  }
  return params
})

const { data: classesResponse, pending: loading, error, refresh } = await useAsyncData(
  'classes-list',
  async () => {
    const response = await apiFetch('/classes', {
      query: queryParams.value
    })
    return response
  },
  { watch: [queryParams] }
)

const classes = computed(() => classesResponse.value?.data || [])
const meta = computed(() => classesResponse.value?.meta || null)
const totalResults = computed(() => meta.value?.total || 0)

watch(searchQuery, () => { currentPage.value = 1 })

useSeoMeta({
  title: 'Classes - D&D 5e Compendium',
  description: 'Browse all D&D 5e classes...',
})

// ... more boilerplate
```

**After (30 lines):**
```typescript
const {
  searchQuery,
  currentPage,
  data: classes,
  meta,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/classes',
  cacheKey: 'classes-list',
  queryBuilder: computed(() => ({})),  // No custom filters
  seo: {
    title: 'Classes - D&D 5e Compendium',
    description: 'Browse all D&D 5e classes...'
  }
})
```

**Savings:** 59 lines

---

### Example 2: Complex List (Spells - Multiple Filters)

**Before (136 lines):**
```typescript
// Route initialization
const route = useRoute()

// Filter state
const searchQuery = ref((route.query.q as string) || '')
const selectedLevel = ref(route.query.level ? Number(route.query.level) : null)
const selectedSchool = ref(route.query.school ? Number(route.query.school) : null)
const currentPage = ref(route.query.page ? Number(route.query.page) : 1)
const perPage = 24

// Fetch spell schools
const { data: spellSchools } = await useAsyncData('spell-schools', async () => {
  const response = await apiFetch('/spell-schools')
  return response.data
})

// Query params
const queryParams = computed(() => {
  const params: Record<string, any> = {
    per_page: perPage,
    page: currentPage.value,
  }
  if (searchQuery.value.trim()) {
    params.q = searchQuery.value.trim()
  }
  if (selectedLevel.value !== null) {
    params.level = selectedLevel.value
  }
  if (selectedSchool.value !== null) {
    params.school = selectedSchool.value
  }
  return params
})

// Fetch spells
const { data: spellsResponse, pending: loading, error, refresh } = await useAsyncData(
  'spells-list',
  async () => {
    const response = await apiFetch('/spells', {
      query: queryParams.value
    })
    return response
  },
  { watch: [currentPage, searchQuery, selectedLevel, selectedSchool] }
)

// Computed values
const spells = computed(() => spellsResponse.value?.data || [])
const meta = computed(() => spellsResponse.value?.meta || null)
const totalResults = computed(() => meta.value?.total || 0)

// Active filters check
const hasActiveFilters = computed(() =>
  searchQuery.value || selectedLevel.value !== null || selectedSchool.value !== null
)

// Clear filters
const clearFilters = () => {
  searchQuery.value = ''
  selectedLevel.value = null
  selectedSchool.value = null
  currentPage.value = 1
}

// Reset to page 1 when filters change
watch([searchQuery, selectedLevel, selectedSchool], () => {
  currentPage.value = 1
})

// URL sync
watch([currentPage, searchQuery, selectedLevel, selectedSchool], () => {
  const query: Record<string, any> = {}
  if (currentPage.value > 1) query.page = currentPage.value.toString()
  if (searchQuery.value) query.q = searchQuery.value
  if (selectedLevel.value !== null) query.level = selectedLevel.value.toString()
  if (selectedSchool.value !== null) query.school = selectedSchool.value.toString()
  navigateTo({ query }, { replace: true })
})

// SEO
useSeoMeta({
  title: 'Spells - D&D 5e Compendium',
  description: 'Browse all D&D 5e spells...',
})

useHead({
  title: 'Spells - D&D 5e Compendium',
})
```

**After (60 lines):**
```typescript
const route = useRoute()

// Custom filter state (entity-specific)
const selectedLevel = ref(route.query.level ? Number(route.query.level) : null)
const selectedSchool = ref(route.query.school ? Number(route.query.school) : null)

// Fetch spell schools (entity-specific data)
const { data: spellSchools } = await useAsyncData('spell-schools', async () => {
  const response = await apiFetch('/spell-schools')
  return response.data
})

// Query builder for custom filters
const queryBuilder = computed(() => {
  const params: Record<string, any> = {}
  if (selectedLevel.value !== null) params.level = selectedLevel.value
  if (selectedSchool.value !== null) params.school = selectedSchool.value
  return params
})

// Composable handles everything else
const {
  searchQuery,
  currentPage,
  data: spells,
  meta,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters: clearBaseFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/spells',
  cacheKey: 'spells-list',
  queryBuilder,
  seo: {
    title: 'Spells - D&D 5e Compendium',
    description: 'Browse all D&D 5e spells...'
  }
})

// Clear all filters (base + custom)
const clearAllFilters = () => {
  clearBaseFilters()
  selectedLevel.value = null
  selectedSchool.value = null
}

// Helper for filter chips
const getSchoolName = (schoolId: number) => {
  return spellSchools.value?.find((s: any) => s.id === schoolId)?.name || 'Unknown'
}

// Level options for dropdown
const levelOptions = [
  { label: 'All Levels', value: null },
  { label: 'Cantrip', value: 0 },
  // ... etc
]
```

**Savings:** 76 lines

---

## Migration Strategy

### Phase 1: Create Composable + Tests (TDD)
1. Write tests first (following TDD mandate)
2. Implement composable
3. Ensure 100% test coverage

### Phase 2: Refactor One Page (Proof of Concept)
1. Choose simplest page: `classes/index.vue`
2. Refactor using composable
3. Verify all functionality works
4. Commit with passing tests

### Phase 3: Roll Out to Remaining Pages
1. Refactor in order: backgrounds → feats → races → items → spells
2. One commit per page
3. Verify tests pass after each page

### Phase 4: Cleanup
1. Remove deprecated code patterns
2. Update documentation
3. Create usage guide for future entities

---

## Test Requirements

Following the **TDD mandate** in CLAUDE.md, tests MUST be written FIRST.

### Test Coverage Required

**Unit Tests (useEntityList.test.ts):**
- ✅ Initializes with default values
- ✅ Initializes from route query params
- ✅ Updates query params when state changes
- ✅ Syncs state to URL correctly
- ✅ Merges custom filters from queryBuilder
- ✅ Clears base filters (search, page)
- ✅ Detects active filters correctly
- ✅ Refetches data when filters change
- ✅ Handles API errors gracefully
- ✅ Returns correct computed values (meta, totalResults)
- ✅ SEO meta is set correctly
- ✅ Pagination state updates correctly

**Integration Tests (per page):**
- ✅ Page renders with composable
- ✅ Custom filters work correctly
- ✅ clearAllFilters clears all state
- ✅ URL params sync bidirectionally
- ✅ Data fetching succeeds

**Minimum:** 15-20 tests for composable + 5 tests per refactored page

---

## Success Criteria

**Before marking complete:**
- [ ] ✅ Composable tests written FIRST (TDD)
- [ ] ✅ All composable tests pass
- [ ] ✅ At least one page refactored successfully
- [ ] ✅ All page functionality preserved
- [ ] ✅ No regressions in existing tests
- [ ] ✅ TypeScript compiles with no errors
- [ ] ✅ Manual browser verification (all filters work)
- [ ] ✅ URL sync works in both directions
- [ ] ✅ SSR works correctly (no hydration errors)
- [ ] ✅ Code committed with clear commit messages

---

## Benefits Summary

**Quantitative:**
- 510 lines of code removed
- 35% reduction in list page code
- 1 composable replaces 6 duplicated implementations
- Estimated 2-3 hours saved per new entity type

**Qualitative:**
- Enforces consistency across all list pages
- Single source of truth for list page logic
- Easier to maintain and extend
- Reduces cognitive load for developers
- Improves testability (test composable once, not 6 times)
- Future-proof (new entities just use the composable)

---

## Risks & Mitigations

**Risk 1: Breaking existing functionality**
**Mitigation:** Refactor one page at a time, extensive testing after each

**Risk 2: TypeScript type inference issues**
**Mitigation:** Explicit return type annotations, proper generic constraints

**Risk 3: SSR hydration issues**
**Mitigation:** Use `useAsyncData` correctly, test SSR thoroughly

**Risk 4: URL sync edge cases**
**Mitigation:** Comprehensive test coverage for URL sync scenarios

---

## Future Enhancements (Post-Implementation)

**Not in scope for initial implementation:**
- Debounced search (can add later if needed)
- Advanced filter operators (AND/OR logic)
- Filter presets/saved searches
- Export to URL/share functionality
- Analytics integration (track popular filters)

---

## References

- **Existing Pattern:** `useSearch()` composable (app/composables/useSearch.ts)
- **Framework Docs:** Nuxt 4.x useAsyncData - https://nuxt.com/docs/api/composables/use-async-data
- **Related Files:** All 6 list pages in `app/pages/*/index.vue`
- **Test Examples:** `tests/composables/useSearch.test.ts`

---

**Status:** Design approved - ready for TDD implementation
**Next Step:** Write tests first, then implement composable
**Estimated Effort:** 4-6 hours (including tests and refactoring all pages)
