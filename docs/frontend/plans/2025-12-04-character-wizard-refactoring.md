# Character Wizard Refactoring Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Reduce duplication in the character wizard by extracting shared patterns into reusable composables and components.

**Architecture:** The wizard currently has 5 picker card components with ~90% identical structure and 3+ step components with repeated fetch/search/modal patterns. We'll create: (1) a generic picker card component, (2) composables for entity fetching, search filtering, and modal state management, and (3) standardize naming conventions.

**Tech Stack:** Vue 3 Composition API | TypeScript | Nuxt 4.x | Pinia | NuxtUI 4.x

---

## Phase 1: Extract Shared Composables (Foundation)

### Task 1: Create useDetailModal Composable

**Files:**
- Create: `app/composables/useDetailModal.ts`
- Test: `tests/composables/useDetailModal.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/composables/useDetailModal.test.ts
import { describe, it, expect } from 'vitest'
import { useDetailModal } from '~/composables/useDetailModal'

interface TestItem {
  id: number
  name: string
}

describe('useDetailModal', () => {
  it('initializes with closed state and null item', () => {
    const { open, item } = useDetailModal<TestItem>()

    expect(open.value).toBe(false)
    expect(item.value).toBeNull()
  })

  it('shows modal with item when show() called', () => {
    const { open, item, show } = useDetailModal<TestItem>()
    const testItem: TestItem = { id: 1, name: 'Test' }

    show(testItem)

    expect(open.value).toBe(true)
    expect(item.value).toEqual(testItem)
  })

  it('closes modal and clears item when close() called', () => {
    const { open, item, show, close } = useDetailModal<TestItem>()
    const testItem: TestItem = { id: 1, name: 'Test' }

    show(testItem)
    close()

    expect(open.value).toBe(false)
    expect(item.value).toBeNull()
  })

  it('replaces item when show() called with different item', () => {
    const { item, show } = useDetailModal<TestItem>()
    const item1: TestItem = { id: 1, name: 'First' }
    const item2: TestItem = { id: 2, name: 'Second' }

    show(item1)
    show(item2)

    expect(item.value).toEqual(item2)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useDetailModal.test.ts --run`
Expected: FAIL with "Cannot find module"

**Step 3: Write minimal implementation**

```typescript
// app/composables/useDetailModal.ts
/**
 * Composable for managing detail modal state
 * Used across wizard steps to show entity details in a modal
 *
 * @example
 * const { open, item, show, close } = useDetailModal<Race>()
 * // In template: <RaceDetailModal :race="item" :open="open" @close="close" />
 * // On card click: @view-details="show(race)"
 */
export function useDetailModal<T>() {
  const open = ref(false)
  const item = ref<T | null>(null) as Ref<T | null>

  function show(newItem: T) {
    item.value = newItem
    open.value = true
  }

  function close() {
    open.value = false
    item.value = null
  }

  return {
    open: readonly(open),
    item: readonly(item),
    show,
    close
  }
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useDetailModal.test.ts --run`
Expected: PASS

**Step 5: Commit**

```bash
git add app/composables/useDetailModal.ts tests/composables/useDetailModal.test.ts
git commit -m "feat(composables): add useDetailModal for wizard step modals

Extracts common modal state management pattern used across
Race, Class, Background, and Subrace wizard steps."
```

---

### Task 2: Create useEntitySearch Composable

**Files:**
- Create: `app/composables/useEntitySearch.ts`
- Test: `tests/composables/useEntitySearch.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/composables/useEntitySearch.test.ts
import { describe, it, expect } from 'vitest'
import { ref } from 'vue'
import { useEntitySearch } from '~/composables/useEntitySearch'

interface TestItem {
  id: number
  name: string
  description?: string
}

describe('useEntitySearch', () => {
  const mockItems: TestItem[] = [
    { id: 1, name: 'Fireball', description: 'A bright streak of fire' },
    { id: 2, name: 'Ice Storm', description: 'Hail pounds an area' },
    { id: 3, name: 'Fire Bolt', description: 'A mote of fire' }
  ]

  it('returns all items when search query is empty', () => {
    const items = ref(mockItems)
    const { filtered, searchQuery } = useEntitySearch(items)

    expect(searchQuery.value).toBe('')
    expect(filtered.value).toEqual(mockItems)
  })

  it('filters items by name (case-insensitive)', () => {
    const items = ref(mockItems)
    const { filtered, searchQuery } = useEntitySearch(items)

    searchQuery.value = 'fire'

    expect(filtered.value).toHaveLength(2)
    expect(filtered.value.map(i => i.name)).toEqual(['Fireball', 'Fire Bolt'])
  })

  it('returns empty array when items is null', () => {
    const items = ref<TestItem[] | null>(null)
    const { filtered } = useEntitySearch(items)

    expect(filtered.value).toEqual([])
  })

  it('uses custom predicate when provided', () => {
    const items = ref(mockItems)
    const { filtered, searchQuery } = useEntitySearch(items, (item, query) => {
      return item.description?.toLowerCase().includes(query) ?? false
    })

    searchQuery.value = 'hail'

    expect(filtered.value).toHaveLength(1)
    expect(filtered.value[0].name).toBe('Ice Storm')
  })

  it('searches multiple fields with searchableFields option', () => {
    const items = ref(mockItems)
    const { filtered, searchQuery } = useEntitySearch(items, {
      searchableFields: ['name', 'description']
    })

    searchQuery.value = 'mote'

    expect(filtered.value).toHaveLength(1)
    expect(filtered.value[0].name).toBe('Fire Bolt')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useEntitySearch.test.ts --run`
Expected: FAIL with "Cannot find module"

**Step 3: Write minimal implementation**

```typescript
// app/composables/useEntitySearch.ts
type SearchPredicate<T> = (item: T, query: string) => boolean

interface SearchOptions<T> {
  /** Fields to search within (uses 'name' if not specified) */
  searchableFields?: Array<keyof T>
}

/**
 * Composable for filtering entities by search query
 * Used across wizard steps for searching races, classes, backgrounds, etc.
 *
 * @param items - Ref containing the items to search
 * @param predicateOrOptions - Custom predicate function OR options object
 *
 * @example
 * // Simple name search
 * const { searchQuery, filtered } = useEntitySearch(races)
 *
 * // Search multiple fields
 * const { searchQuery, filtered } = useEntitySearch(backgrounds, {
 *   searchableFields: ['name', 'feature_name']
 * })
 *
 * // Custom predicate
 * const { searchQuery, filtered } = useEntitySearch(items, (item, query) =>
 *   item.name.includes(query) || item.tags.some(t => t.includes(query))
 * )
 */
export function useEntitySearch<T extends { name: string }>(
  items: Ref<T[] | null>,
  predicateOrOptions?: SearchPredicate<T> | SearchOptions<T>
) {
  const searchQuery = ref('')

  const filtered = computed((): T[] => {
    if (!items.value) return []
    if (!searchQuery.value.trim()) return items.value

    const query = searchQuery.value.toLowerCase()

    // Custom predicate function
    if (typeof predicateOrOptions === 'function') {
      return items.value.filter(item => predicateOrOptions(item, query))
    }

    // Options object with searchableFields
    if (predicateOrOptions?.searchableFields) {
      const fields = predicateOrOptions.searchableFields
      return items.value.filter(item =>
        fields.some(field => {
          const value = item[field]
          return typeof value === 'string' && value.toLowerCase().includes(query)
        })
      )
    }

    // Default: search by name only
    return items.value.filter(item =>
      item.name.toLowerCase().includes(query)
    )
  })

  return {
    searchQuery,
    filtered: readonly(filtered)
  }
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useEntitySearch.test.ts --run`
Expected: PASS

**Step 5: Commit**

```bash
git add app/composables/useEntitySearch.ts tests/composables/useEntitySearch.test.ts
git commit -m "feat(composables): add useEntitySearch for wizard step filtering

Extracts common search/filter pattern used across Race, Class,
and Background wizard steps. Supports name search by default,
multiple fields, or custom predicates."
```

---

### Task 3: Create useBuildStepEntityFetch Composable

**Files:**
- Create: `app/composables/useBuildStepEntityFetch.ts`
- Test: `tests/composables/useBuildStepEntityFetch.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/composables/useBuildStepEntityFetch.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'

// Mock useAsyncData at module level
const mockUseAsyncData = vi.fn()
vi.mock('#app', () => ({
  useAsyncData: (...args: unknown[]) => mockUseAsyncData(...args)
}))

// Mock useApi
const mockApiFetch = vi.fn()
vi.mock('~/composables/useApi', () => ({
  useApi: () => ({ apiFetch: mockApiFetch })
}))

describe('useBuildStepEntityFetch', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    vi.clearAllMocks()

    mockUseAsyncData.mockResolvedValue({
      data: ref([{ id: 1, name: 'Test' }]),
      pending: ref(false),
      refresh: vi.fn()
    })
  })

  it('constructs URL with source filter when present', async () => {
    // This test validates the URL construction logic
    // The actual fetch is mocked, so we test the composable's behavior
    const { useBuildStepEntityFetch } = await import('~/composables/useBuildStepEntityFetch')

    // Setup store with source filter
    const store = useCharacterBuilderStore()
    store.selectedSourcebooks = [{ id: 1, code: 'PHB', name: 'PHB', slug: 'phb' }]

    await useBuildStepEntityFetch('races')

    expect(mockUseAsyncData).toHaveBeenCalledWith(
      expect.stringContaining('builder-races'),
      expect.any(Function),
      expect.objectContaining({ watch: expect.any(Array) })
    )
  })

  it('constructs URL without filter when no sourcebooks selected', async () => {
    const { useBuildStepEntityFetch } = await import('~/composables/useBuildStepEntityFetch')

    await useBuildStepEntityFetch('classes')

    expect(mockUseAsyncData).toHaveBeenCalled()
  })

  it('allows custom per_page parameter', async () => {
    const { useBuildStepEntityFetch } = await import('~/composables/useBuildStepEntityFetch')

    await useBuildStepEntityFetch('backgrounds', { perPage: 50 })

    expect(mockUseAsyncData).toHaveBeenCalled()
  })

  it('allows additional filter conditions', async () => {
    const { useBuildStepEntityFetch } = await import('~/composables/useBuildStepEntityFetch')

    await useBuildStepEntityFetch('classes', {
      additionalFilter: 'is_base_class=true'
    })

    expect(mockUseAsyncData).toHaveBeenCalled()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useBuildStepEntityFetch.test.ts --run`
Expected: FAIL with "Cannot find module"

**Step 3: Write minimal implementation**

```typescript
// app/composables/useBuildStepEntityFetch.ts
import { storeToRefs } from 'pinia'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

interface FetchOptions {
  /** Number of items per page (default: 100) */
  perPage?: number
  /** Additional Meilisearch filter to combine with source filter */
  additionalFilter?: string
}

/**
 * Composable for fetching entities in character builder wizard steps
 * Automatically includes source filter based on selected sourcebooks
 *
 * @param entityEndpoint - API endpoint name (e.g., 'races', 'classes', 'backgrounds')
 * @param options - Optional fetch configuration
 *
 * @example
 * // Basic usage - fetch races filtered by selected sourcebooks
 * const { data: races, pending } = await useBuildStepEntityFetch<Race>('races')
 *
 * // With additional filter
 * const { data: classes, pending } = await useBuildStepEntityFetch<CharacterClass>('classes', {
 *   additionalFilter: 'is_base_class=true'
 * })
 */
export async function useBuildStepEntityFetch<T extends { id: number }>(
  entityEndpoint: string,
  options: FetchOptions = {}
) {
  const { perPage = 100, additionalFilter } = options

  const store = useCharacterBuilderStore()
  const { sourceFilterString } = storeToRefs(store)
  const { apiFetch } = useApi()

  const buildUrl = () => {
    // Combine additional filter with source filter
    let filter = ''
    if (additionalFilter && sourceFilterString.value) {
      filter = `${additionalFilter} AND ${sourceFilterString.value}`
    } else if (additionalFilter) {
      filter = additionalFilter
    } else if (sourceFilterString.value) {
      filter = sourceFilterString.value
    }

    const baseUrl = `/${entityEndpoint}?per_page=${perPage}`
    return filter
      ? `${baseUrl}&filter=${encodeURIComponent(filter)}`
      : baseUrl
  }

  const { data, pending, refresh } = await useAsyncData(
    `builder-${entityEndpoint}-${sourceFilterString.value}`,
    () => apiFetch<{ data: T[] }>(buildUrl()),
    {
      transform: (response: { data: T[] }) => response.data,
      watch: [sourceFilterString]
    }
  )

  return {
    data,
    pending,
    refresh
  }
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useBuildStepEntityFetch.test.ts --run`
Expected: PASS

**Step 5: Commit**

```bash
git add app/composables/useBuildStepEntityFetch.ts tests/composables/useBuildStepEntityFetch.test.ts
git commit -m "feat(composables): add useBuildStepEntityFetch for wizard data fetching

Extracts common entity fetch pattern from Race, Class, and Background
steps. Automatically combines source filter with optional additional
filters and handles watch/transform setup."
```

---

## Phase 2: Standardize Picker Cards

### Task 4: Standardize BackgroundPickerCard Naming Conventions

**Files:**
- Modify: `app/components/character/builder/BackgroundPickerCard.vue`

**Step 1: Document current inconsistencies**

Current issues in BackgroundPickerCard.vue:
- Uses `viewDetails` (camelCase) instead of `view-details` (kebab-case) in emit
- Uses `data-test` instead of `data-testid` for test attributes
- Missing file header comment

**Step 2: Fix emit naming**

Change lines 11-14 from:
```typescript
const emit = defineEmits<{
  select: [background: Background]
  viewDetails: []
}>()
```

To:
```typescript
const emit = defineEmits<{
  'select': [background: Background]
  'view-details': []
}>()
```

**Step 3: Fix emit call**

Change line 42 from:
```typescript
emit('viewDetails')
```

To:
```typescript
emit('view-details')
```

**Step 4: Fix test attributes**

Change `data-test` to `data-testid` on lines 48, 66, 129

**Step 5: Add file header comment**

Add at line 1:
```vue
<!-- app/components/character/builder/BackgroundPickerCard.vue -->
```

**Step 6: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

**Step 7: Commit**

```bash
git add app/components/character/builder/BackgroundPickerCard.vue
git commit -m "fix(wizard): standardize BackgroundPickerCard naming conventions

- Change viewDetails emit to view-details (kebab-case)
- Change data-test to data-testid for consistency
- Add file header comment"
```

---

### Task 5: Standardize SpellPickerCard Naming Conventions

**Files:**
- Modify: `app/components/character/builder/SpellPickerCard.vue`

**Step 1: Fix emit naming**

SpellPickerCard uses `viewDetails` (camelCase). Change lines 14-17 from:
```typescript
const emit = defineEmits<{
  toggle: [spell: Spell]
  viewDetails: []
}>()
```

To:
```typescript
const emit = defineEmits<{
  'toggle': [spell: Spell]
  'view-details': []
}>()
```

**Step 2: Fix emit call**

Change line 53 from:
```typescript
emit('viewDetails')
```

To:
```typescript
emit('view-details')
```

**Step 3: Fix test attributes**

Change `data-test` to `data-testid` on lines 59, 72, 124

**Step 4: Add file header comment**

Add at line 1:
```vue
<!-- app/components/character/builder/SpellPickerCard.vue -->
```

**Step 5: Update StepSpells.vue to use new emit name**

Search for `@viewDetails` or `@view-details` in StepSpells.vue and ensure it matches.

**Step 6: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

**Step 7: Commit**

```bash
git add app/components/character/builder/SpellPickerCard.vue
git commit -m "fix(wizard): standardize SpellPickerCard naming conventions

- Change viewDetails emit to view-details (kebab-case)
- Change data-test to data-testid for consistency
- Add file header comment"
```

---

## Phase 3: Apply Composables to Step Components

### Task 6: Refactor StepBackground to Use New Composables

**Files:**
- Modify: `app/components/character/builder/StepBackground.vue`

**Step 1: Replace modal state with useDetailModal**

Remove lines 33-34:
```typescript
const detailModalOpen = ref(false)
const detailBackground = ref<Background | null>(null)
```

Add import and usage:
```typescript
import { useDetailModal } from '~/composables/useDetailModal'

const { open: detailModalOpen, item: detailBackground, show: showDetails, close: closeDetails } = useDetailModal<Background>()
```

**Step 2: Replace modal handlers**

Remove lines 62-73:
```typescript
function handleViewDetails(background: Background) {
  detailBackground.value = background
  detailModalOpen.value = true
}

function handleCloseModal() {
  detailModalOpen.value = false
  detailBackground.value = null
}
```

**Step 3: Replace search logic with useEntitySearch**

Remove lines 31, 37-45:
```typescript
const searchQuery = ref('')
// ...
const filteredBackgrounds = computed((): Background[] => { ... })
```

Add:
```typescript
import { useEntitySearch } from '~/composables/useEntitySearch'

const { searchQuery, filtered: filteredBackgrounds } = useEntitySearch(backgrounds, {
  searchableFields: ['name', 'feature_name']
})
```

**Step 4: Update template event handlers**

Change `@view-details="handleViewDetails(background)"` to `@view-details="showDetails(background)"`
Change `@close="handleCloseModal"` to `@close="closeDetails"`

**Step 5: Run tests**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StepBackground.test.ts --run`
Expected: PASS (if tests exist) or no errors

**Step 6: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

**Step 7: Commit**

```bash
git add app/components/character/builder/StepBackground.vue
git commit -m "refactor(wizard): use composables in StepBackground

- Replace manual modal state with useDetailModal composable
- Replace search logic with useEntitySearch composable
- Reduces component complexity and improves consistency"
```

---

### Task 7: Refactor StepClass to Use New Composables

**Files:**
- Modify: `app/components/character/builder/StepClass.vue`

**Step 1: Replace fetch logic with useBuildStepEntityFetch**

Remove lines 15-31 (the useAsyncData block).

Add:
```typescript
import { useBuildStepEntityFetch } from '~/composables/useBuildStepEntityFetch'

const { data: classes, pending: loadingClasses } = await useBuildStepEntityFetch<CharacterClass>('classes', {
  additionalFilter: 'is_base_class=true',
  perPage: 50
})
```

**Step 2: Replace modal state with useDetailModal**

Remove lines 36-37:
```typescript
const detailModalOpen = ref(false)
const detailClass = ref<CharacterClass | null>(null)
```

Add:
```typescript
import { useDetailModal } from '~/composables/useDetailModal'

const { open: detailModalOpen, item: detailClass, show: showDetails, close: closeDetails } = useDetailModal<CharacterClass>()
```

**Step 3: Replace search logic with useEntitySearch**

Remove lines 34, 40-47.

Add:
```typescript
import { useEntitySearch } from '~/composables/useEntitySearch'

const { searchQuery, filtered: filteredClasses } = useEntitySearch(classes)
```

**Step 4: Remove modal handler functions**

Remove lines 64-75 (handleViewDetails, handleCloseModal).

**Step 5: Update template event handlers**

- `@view-details="handleViewDetails(cls)"` → `@view-details="showDetails(cls)"`
- `@close="handleCloseModal"` → `@close="closeDetails"`

**Step 6: Run typecheck and tests**

Run: `docker compose exec nuxt npm run typecheck && docker compose exec nuxt npm run test -- --run`
Expected: PASS

**Step 7: Commit**

```bash
git add app/components/character/builder/StepClass.vue
git commit -m "refactor(wizard): use composables in StepClass

- Replace useAsyncData with useBuildStepEntityFetch
- Replace manual modal state with useDetailModal
- Replace search logic with useEntitySearch"
```

---

### Task 8: Refactor StepRace to Use New Composables

**Files:**
- Modify: `app/components/character/builder/StepRace.vue`

**Step 1: Replace fetch logic**

Remove lines 16-29 and replace with:
```typescript
import { useBuildStepEntityFetch } from '~/composables/useBuildStepEntityFetch'

const { data: races, pending: loadingRaces } = await useBuildStepEntityFetch<Race>('races')
```

**Step 2: Replace modal state**

Remove lines 34-35 and add:
```typescript
import { useDetailModal } from '~/composables/useDetailModal'

const { open: detailModalOpen, item: detailRace, show: showDetails, close: closeDetails } = useDetailModal<Race>()
```

**Step 3: Replace search logic**

Remove line 32 and lines 48-54. Add:
```typescript
import { useEntitySearch } from '~/composables/useEntitySearch'

// Keep baseRaces filter (race-specific logic)
const baseRaces = computed((): Race[] => {
  if (!races.value) return []
  return races.value.filter((race: Race) => !race.parent_race)
})

const { searchQuery, filtered: filteredRaces } = useEntitySearch(baseRaces)
```

**Step 4: Remove modal handlers (lines 96-107)**

**Step 5: Update template**

- `@view-details="handleViewDetails(race)"` → `@view-details="showDetails(race)"`
- `@close="handleCloseModal"` → `@close="closeDetails"`

**Step 6: Run typecheck and tests**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

**Step 7: Commit**

```bash
git add app/components/character/builder/StepRace.vue
git commit -m "refactor(wizard): use composables in StepRace

- Replace useAsyncData with useBuildStepEntityFetch
- Replace manual modal state with useDetailModal
- Replace search logic with useEntitySearch
- Keep race-specific baseRaces filter for parent_race logic"
```

---

## Phase 4: Final Verification

### Task 9: Run Full Test Suite and Verify

**Step 1: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

**Step 2: Run lint**

Run: `docker compose exec nuxt npm run lint`
Expected: No errors (or run lint:fix if needed)

**Step 3: Run full test suite**

Run: `docker compose exec nuxt npm run test -- --run`
Expected: All tests pass

**Step 4: Manual browser verification**

1. Navigate to character creator
2. Test Race step: search, select, view details modal, continue
3. Test Class step: search, select, view details modal, continue
4. Test Background step: search, select, view details modal, continue
5. Verify sourcebook filtering still works across all steps

**Step 5: Create summary commit**

If any fixes were needed during verification:
```bash
git add -A
git commit -m "fix(wizard): address issues found during refactoring verification"
```

---

## Summary

**Files Created:**
- `app/composables/useDetailModal.ts`
- `app/composables/useEntitySearch.ts`
- `app/composables/useBuildStepEntityFetch.ts`
- `tests/composables/useDetailModal.test.ts`
- `tests/composables/useEntitySearch.test.ts`
- `tests/composables/useBuildStepEntityFetch.test.ts`

**Files Modified:**
- `app/components/character/builder/BackgroundPickerCard.vue` (standardize naming)
- `app/components/character/builder/SpellPickerCard.vue` (standardize naming)
- `app/components/character/builder/StepBackground.vue` (use composables)
- `app/components/character/builder/StepClass.vue` (use composables)
- `app/components/character/builder/StepRace.vue` (use composables)

**Lines of Code Impact:**
- ~120 lines added (3 composables + tests)
- ~150 lines removed (duplicated logic across steps)
- Net reduction: ~30 lines + improved maintainability

**Deferred for Future:**
- Generic PickerCard component (larger refactor, needs design work)
- Dynamic `[step].vue` routing (lower priority)
- Store splitting (only if more features added)
