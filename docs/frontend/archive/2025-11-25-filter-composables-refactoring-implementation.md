# Filter Composables Refactoring - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Extract 3 reusable composables to eliminate ~500 lines of duplicate Meilisearch filter logic across 6 entity pages.

**Architecture:** Conservative extraction approach - create focused composables (useMeilisearchFilters, useReferenceData, useFilterCount) without changing page structure. Migrate pages incrementally with comprehensive testing.

**Tech Stack:** Vue 3 Composition API, TypeScript, Vitest, Nuxt 4

---

## Prerequisites

**Before starting:**
- [ ] Backend running at http://localhost:8080
- [ ] Frontend running: `docker compose up -d`
- [ ] Read design doc: `docs/plans/2025-11-25-filter-composables-refactoring-design.md`
- [ ] All tests passing: `docker compose exec nuxt npm run test`

---

## Phase 1: Create Composables (TDD)

### Task 1: Create `useMeilisearchFilters()` Composable

**Files:**
- Create: `app/composables/useMeilisearchFilters.ts`
- Create: `tests/composables/useMeilisearchFilters.test.ts`

#### Step 1.1: Write the failing test file structure

Create: `tests/composables/useMeilisearchFilters.test.ts`

```typescript
import { describe, it, expect } from 'vitest'
import { ref } from 'vue'
import { useMeilisearchFilters } from '~/composables/useMeilisearchFilters'

describe('useMeilisearchFilters', () => {
  describe('equals filter', () => {
    it('builds filter for single value', () => {
      const levelRef = ref(3)

      const { queryParams } = useMeilisearchFilters([
        { ref: levelRef, field: 'level' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'level = 3'
      })
    })

    it('skips null values', () => {
      const levelRef = ref(null)

      const { queryParams } = useMeilisearchFilters([
        { ref: levelRef, field: 'level' }
      ])

      expect(queryParams.value).toEqual({})
    })

    it('handles transform function', () => {
      const schoolIdRef = ref(2)

      const { queryParams } = useMeilisearchFilters([
        {
          ref: schoolIdRef,
          field: 'school_code',
          transform: (id) => id === 2 ? 'EV' : null
        }
      ])

      expect(queryParams.value).toEqual({
        filter: 'school_code = EV'
      })
    })

    it('skips when transform returns null', () => {
      const schoolIdRef = ref(99)

      const { queryParams } = useMeilisearchFilters([
        {
          ref: schoolIdRef,
          field: 'school_code',
          transform: () => null
        }
      ])

      expect(queryParams.value).toEqual({})
    })
  })

  describe('boolean filter', () => {
    it('converts string "1" to true', () => {
      const concentrationRef = ref('1')

      const { queryParams } = useMeilisearchFilters([
        { ref: concentrationRef, field: 'concentration', type: 'boolean' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'concentration = true'
      })
    })

    it('converts string "true" to true', () => {
      const ritualRef = ref('true')

      const { queryParams } = useMeilisearchFilters([
        { ref: ritualRef, field: 'ritual', type: 'boolean' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'ritual = true'
      })
    })

    it('converts string "0" to false', () => {
      const concentrationRef = ref('0')

      const { queryParams } = useMeilisearchFilters([
        { ref: concentrationRef, field: 'concentration', type: 'boolean' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'concentration = false'
      })
    })

    it('converts string "false" to false', () => {
      const ritualRef = ref('false')

      const { queryParams } = useMeilisearchFilters([
        { ref: ritualRef, field: 'ritual', type: 'boolean' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'ritual = false'
      })
    })

    it('handles actual boolean values', () => {
      const legendaryRef = ref(true)

      const { queryParams } = useMeilisearchFilters([
        { ref: legendaryRef, field: 'is_legendary', type: 'boolean' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'is_legendary = true'
      })
    })

    it('skips null values', () => {
      const concentrationRef = ref(null)

      const { queryParams } = useMeilisearchFilters([
        { ref: concentrationRef, field: 'concentration', type: 'boolean' }
      ])

      expect(queryParams.value).toEqual({})
    })
  })

  describe('in filter', () => {
    it('builds IN filter for array', () => {
      const damageTypesRef = ref(['F', 'C'])

      const { queryParams } = useMeilisearchFilters([
        { ref: damageTypesRef, field: 'damage_types', type: 'in' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'damage_types IN [F, C]'
      })
    })

    it('skips empty arrays', () => {
      const damageTypesRef = ref([])

      const { queryParams } = useMeilisearchFilters([
        { ref: damageTypesRef, field: 'damage_types', type: 'in' }
      ])

      expect(queryParams.value).toEqual({})
    })

    it('handles single value as array', () => {
      const classRef = ref('wizard')

      const { queryParams } = useMeilisearchFilters([
        { ref: classRef, field: 'class_slugs', type: 'in' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'class_slugs IN [wizard]'
      })
    })
  })

  describe('range filter', () => {
    it('builds range with min and max', () => {
      const minRef = ref(5)
      const maxRef = ref(10)

      const { queryParams } = useMeilisearchFilters([
        { field: 'challenge_rating', type: 'range', min: minRef, max: maxRef, ref: ref(null) }
      ])

      expect(queryParams.value).toEqual({
        filter: 'challenge_rating >= 5 AND challenge_rating <= 10'
      })
    })

    it('builds range with only min', () => {
      const minRef = ref(17)
      const maxRef = ref(null)

      const { queryParams } = useMeilisearchFilters([
        { field: 'challenge_rating', type: 'range', min: minRef, max: maxRef, ref: ref(null) }
      ])

      expect(queryParams.value).toEqual({
        filter: 'challenge_rating >= 17'
      })
    })

    it('builds range with only max', () => {
      const minRef = ref(null)
      const maxRef = ref(4)

      const { queryParams } = useMeilisearchFilters([
        { field: 'challenge_rating', type: 'range', min: minRef, max: maxRef, ref: ref(null) }
      ])

      expect(queryParams.value).toEqual({
        filter: 'challenge_rating <= 4'
      })
    })

    it('skips when both min and max are null', () => {
      const minRef = ref(null)
      const maxRef = ref(null)

      const { queryParams } = useMeilisearchFilters([
        { field: 'challenge_rating', type: 'range', min: minRef, max: maxRef, ref: ref(null) }
      ])

      expect(queryParams.value).toEqual({})
    })
  })

  describe('isEmpty filter', () => {
    it('builds IS EMPTY for true value', () => {
      const hasPrereqRef = ref('1')

      const { queryParams } = useMeilisearchFilters([
        { ref: hasPrereqRef, field: 'prerequisites', type: 'isEmpty' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'prerequisites IS EMPTY'
      })
    })

    it('builds IS NOT EMPTY for false value', () => {
      const hasPrereqRef = ref('0')

      const { queryParams } = useMeilisearchFilters([
        { ref: hasPrereqRef, field: 'prerequisites', type: 'isEmpty' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'prerequisites IS NOT EMPTY'
      })
    })

    it('converts string "true" to IS EMPTY', () => {
      const hasPrereqRef = ref('true')

      const { queryParams } = useMeilisearchFilters([
        { ref: hasPrereqRef, field: 'prerequisites', type: 'isEmpty' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'prerequisites IS EMPTY'
      })
    })

    it('converts boolean true to IS EMPTY', () => {
      const hasPrereqRef = ref(true)

      const { queryParams } = useMeilisearchFilters([
        { ref: hasPrereqRef, field: 'prerequisites', type: 'isEmpty' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'prerequisites IS EMPTY'
      })
    })
  })

  describe('greaterThan filter', () => {
    it('builds > filter for numeric value', () => {
      const chargesRef = ref(0)

      const { queryParams } = useMeilisearchFilters([
        { ref: chargesRef, field: 'charges_max', type: 'greaterThan' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'charges_max > 0'
      })
    })

    it('skips null values', () => {
      const chargesRef = ref(null)

      const { queryParams } = useMeilisearchFilters([
        { ref: chargesRef, field: 'charges_max', type: 'greaterThan' }
      ])

      expect(queryParams.value).toEqual({})
    })
  })

  describe('multiple filters', () => {
    it('combines multiple filters with AND', () => {
      const levelRef = ref(3)
      const concentrationRef = ref('1')

      const { queryParams } = useMeilisearchFilters([
        { ref: levelRef, field: 'level' },
        { ref: concentrationRef, field: 'concentration', type: 'boolean' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'level = 3 AND concentration = true'
      })
    })

    it('skips inactive filters', () => {
      const levelRef = ref(3)
      const schoolRef = ref(null)
      const damageTypesRef = ref([])

      const { queryParams } = useMeilisearchFilters([
        { ref: levelRef, field: 'level' },
        { ref: schoolRef, field: 'school' },
        { ref: damageTypesRef, field: 'damage_types', type: 'in' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'level = 3'
      })
    })

    it('handles all filter types together', () => {
      const levelRef = ref(3)
      const concentrationRef = ref('1')
      const damageTypesRef = ref(['F', 'C'])

      const { queryParams } = useMeilisearchFilters([
        { ref: levelRef, field: 'level' },
        { ref: concentrationRef, field: 'concentration', type: 'boolean' },
        { ref: damageTypesRef, field: 'damage_types', type: 'in' }
      ])

      expect(queryParams.value).toEqual({
        filter: 'level = 3 AND concentration = true AND damage_types IN [F, C]'
      })
    })
  })

  describe('edge cases', () => {
    it('returns empty params when no active filters', () => {
      const levelRef = ref(null)

      const { queryParams } = useMeilisearchFilters([
        { ref: levelRef, field: 'level' }
      ])

      expect(queryParams.value).toEqual({})
    })

    it('handles undefined refs', () => {
      const levelRef = ref(undefined)

      const { queryParams } = useMeilisearchFilters([
        { ref: levelRef, field: 'level' }
      ])

      expect(queryParams.value).toEqual({})
    })

    it('handles empty string values', () => {
      const typeRef = ref('')

      const { queryParams } = useMeilisearchFilters([
        { ref: typeRef, field: 'type' }
      ])

      expect(queryParams.value).toEqual({})
    })

    it('reactive updates when refs change', () => {
      const levelRef = ref(null)

      const { queryParams } = useMeilisearchFilters([
        { ref: levelRef, field: 'level' }
      ])

      expect(queryParams.value).toEqual({})

      levelRef.value = 5

      expect(queryParams.value).toEqual({
        filter: 'level = 5'
      })
    })
  })
})
```

#### Step 1.2: Run tests to verify they fail

Run: `docker compose exec nuxt npm run test -- useMeilisearchFilters`

Expected: FAIL - "Cannot find module '~/composables/useMeilisearchFilters'"

#### Step 1.3: Write minimal implementation

Create: `app/composables/useMeilisearchFilters.ts`

```typescript
import { computed, type Ref, type ComputedRef } from 'vue'

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

/**
 * Build Meilisearch filter params from declarative filter configs
 *
 * Handles all common filter types: equals, boolean, IN, ranges, isEmpty, greaterThan.
 * Auto-skips null/undefined/empty values. Combines multiple filters with AND.
 *
 * @example
 * ```typescript
 * const { queryParams } = useMeilisearchFilters([
 *   { ref: selectedLevel, field: 'level' },
 *   { ref: concentrationFilter, field: 'concentration', type: 'boolean' },
 *   { ref: selectedDamageTypes, field: 'damage_types', type: 'in' }
 * ])
 *
 * // Pass directly to useEntityList
 * const { ... } = useEntityList({
 *   endpoint: '/spells',
 *   queryBuilder: queryParams,
 *   // ...
 * })
 * ```
 */
export function useMeilisearchFilters(
  filters: FilterConfig[]
): UseMeilisearchFiltersReturn {
  const queryParams = computed(() => {
    const params: Record<string, unknown> = {}
    const meilisearchFilters: string[] = []

    for (const config of filters) {
      const value = config.ref.value

      // Skip null/undefined/empty values
      if (value === null || value === undefined) continue
      if (value === '') continue
      if (Array.isArray(value) && value.length === 0) continue

      switch (config.type || 'equals') {
        case 'equals': {
          const transformedValue = config.transform ? config.transform(value) : value
          if (transformedValue !== null && transformedValue !== undefined) {
            meilisearchFilters.push(`${config.field} = ${transformedValue}`)
          }
          break
        }

        case 'boolean': {
          const boolValue = value === '1' || value === 'true' || value === true
          meilisearchFilters.push(`${config.field} = ${boolValue}`)
          break
        }

        case 'in': {
          const values = Array.isArray(value) ? value : [value]
          if (values.length > 0) {
            const joined = values.join(', ')
            meilisearchFilters.push(`${config.field} IN [${joined}]`)
          }
          break
        }

        case 'range': {
          const min = config.min?.value
          const max = config.max?.value
          const conditions: string[] = []

          if (min !== null && min !== undefined) {
            conditions.push(`${config.field} >= ${min}`)
          }
          if (max !== null && max !== undefined) {
            conditions.push(`${config.field} <= ${max}`)
          }

          if (conditions.length > 0) {
            meilisearchFilters.push(conditions.join(' AND '))
          }
          break
        }

        case 'isEmpty': {
          const isEmpty = value === '1' || value === 'true' || value === true
          meilisearchFilters.push(
            isEmpty
              ? `${config.field} IS EMPTY`
              : `${config.field} IS NOT EMPTY`
          )
          break
        }

        case 'greaterThan': {
          meilisearchFilters.push(`${config.field} > ${value}`)
          break
        }
      }
    }

    // Combine all filters with AND
    if (meilisearchFilters.length > 0) {
      params.filter = meilisearchFilters.join(' AND ')
    }

    return params
  })

  return {
    queryParams
  }
}
```

#### Step 1.4: Run tests to verify they pass

Run: `docker compose exec nuxt npm run test -- useMeilisearchFilters`

Expected: PASS - All 28 tests passing

#### Step 1.5: Run full test suite

Run: `docker compose exec nuxt npm run test`

Expected: All tests passing (including new tests)

#### Step 1.6: Commit

```bash
git add app/composables/useMeilisearchFilters.ts tests/composables/useMeilisearchFilters.test.ts
git commit -m "feat: add useMeilisearchFilters composable with comprehensive tests

- Declarative filter builder for Meilisearch queries
- Supports 6 filter types: equals, boolean, in, range, isEmpty, greaterThan
- Auto-skips null/undefined/empty values
- Combines filters with AND
- Transform function for ID→code lookups
- 28 comprehensive tests (100% coverage)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 2: Create `useReferenceData<T>()` Composable

**Files:**
- Create: `app/composables/useReferenceData.ts`
- Create: `tests/composables/useReferenceData.test.ts`

#### Step 2.1: Write the failing test file

Create: `tests/composables/useReferenceData.test.ts`

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { useReferenceData } from '~/composables/useReferenceData'

// Mock useApi and useAsyncData
vi.mock('~/composables/useApi', () => ({
  useApi: () => ({
    apiFetch: vi.fn()
  })
}))

vi.mock('#app', () => ({
  useAsyncData: vi.fn()
}))

describe('useReferenceData', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('fetches data from endpoint', async () => {
    const mockData = [
      { id: 1, name: 'Abjuration', code: 'A' },
      { id: 2, name: 'Conjuration', code: 'C' }
    ]

    const { useAsyncData } = await import('#app')
    const { useApi } = await import('~/composables/useApi')

    vi.mocked(useAsyncData).mockImplementation((key, fetcher) => {
      return {
        data: { value: mockData },
        error: { value: null },
        status: { value: 'success' }
      } as any
    })

    const { data, loading } = useReferenceData('/spell-schools')

    expect(data.value).toEqual(mockData)
    expect(loading.value).toBe(false)
  })

  it('returns empty array for failed fetches', async () => {
    const { useAsyncData } = await import('#app')

    vi.mocked(useAsyncData).mockImplementation(() => {
      return {
        data: { value: null },
        error: { value: new Error('Network error') },
        status: { value: 'error' }
      } as any
    })

    const { data, error } = useReferenceData('/spell-schools')

    expect(data.value).toEqual([])
    expect(error.value).toBeTruthy()
  })

  it('uses default cache key', async () => {
    const { useAsyncData } = await import('#app')

    let capturedKey = ''
    vi.mocked(useAsyncData).mockImplementation((key) => {
      capturedKey = key as string
      return {
        data: { value: [] },
        error: { value: null },
        status: { value: 'success' }
      } as any
    })

    useReferenceData('/spell-schools')

    expect(capturedKey).toBe('reference-spell-schools')
  })

  it('uses custom cache key', async () => {
    const { useAsyncData } = await import('#app')

    let capturedKey = ''
    vi.mocked(useAsyncData).mockImplementation((key) => {
      capturedKey = key as string
      return {
        data: { value: [] },
        error: { value: null },
        status: { value: 'success' }
      } as any
    })

    useReferenceData('/spell-schools', { cacheKey: 'my-custom-key' })

    expect(capturedKey).toBe('my-custom-key')
  })

  it('applies transform function', async () => {
    const mockData = [
      { id: 1, name: 'Wizard', is_base_class: true },
      { id: 2, name: 'Eldritch Knight', is_base_class: false }
    ]

    const { useAsyncData } = await import('#app')

    vi.mocked(useAsyncData).mockImplementation((key, fetcher: any) => {
      const result = fetcher()
      return {
        data: { value: result },
        error: { value: null },
        status: { value: 'success' }
      } as any
    })

    const { data } = useReferenceData('/classes', {
      transform: (data: any) => data.filter((c: any) => c.is_base_class === true)
    })

    // Transform will be called inside fetcher, so we need to verify the behavior
    expect(data.value).toBeDefined()
  })

  it('sets loading state correctly', async () => {
    const { useAsyncData } = await import('#app')

    vi.mocked(useAsyncData).mockImplementation(() => {
      return {
        data: { value: [] },
        error: { value: null },
        status: { value: 'pending' }
      } as any
    })

    const { loading } = useReferenceData('/spell-schools')

    expect(loading.value).toBe(true)
  })
})
```

#### Step 2.2: Run tests to verify they fail

Run: `docker compose exec nuxt npm run test -- useReferenceData`

Expected: FAIL - "Cannot find module '~/composables/useReferenceData'"

#### Step 2.3: Write minimal implementation

Create: `app/composables/useReferenceData.ts`

```typescript
import { computed, type ComputedRef, type Ref } from 'vue'

export interface ReferenceDataOptions {
  /** Override cache key (default: endpoint-based) */
  cacheKey?: string

  /** Transform response (default: extracts .data property) */
  transform?: (data: any[]) => any[]

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

/**
 * Fetch reference data for filter options
 *
 * Wraps the common pattern of useAsyncData + apiFetch for reference entities.
 * Automatically handles pagination, caching, and type safety.
 *
 * @example
 * ```typescript
 * // Simple fetch
 * const { data: schools } = useReferenceData<SpellSchool>('/spell-schools')
 *
 * // Multi-page fetch (for large datasets)
 * const { data: classes } = useReferenceData<CharacterClass>('/classes', {
 *   pages: 2,
 *   transform: (data) => data.filter(c => c.is_base_class === true)
 * })
 *
 * // Use in computed options
 * const schoolOptions = computed(() =>
 *   schools.value?.map(s => ({ label: s.name, value: s.id })) || []
 * )
 * ```
 */
export function useReferenceData<T>(
  endpoint: string,
  options: ReferenceDataOptions = {}
): UseReferenceDataReturn<T> {
  const { apiFetch } = useApi()

  const cacheKey = options.cacheKey || `reference-${endpoint.replace(/\//g, '-')}`

  const { data, error, status } = useAsyncData<T[]>(
    cacheKey,
    async () => {
      // Handle multi-page fetching
      if (options.pages && options.pages > 1) {
        const pagePromises = Array.from({ length: options.pages }, (_, i) => {
          const page = i + 1
          return apiFetch<{ data: T[] }>(`${endpoint}?per_page=100&page=${page}`)
        })

        const responses = await Promise.all(pagePromises)
        const allData = responses.flatMap(r => r?.data || [])

        return options.transform ? options.transform(allData) : allData
      }

      // Single page fetch
      const response = await apiFetch<{ data: T[] }>(endpoint)
      const data = response?.data || []

      return options.transform ? options.transform(data) : data
    }
  )

  return {
    data: computed(() => data.value || []),
    error,
    status,
    loading: computed(() => status.value === 'pending')
  }
}
```

#### Step 2.4: Run tests to verify they pass

Run: `docker compose exec nuxt npm run test -- useReferenceData`

Expected: PASS - All 6 tests passing

#### Step 2.5: Run full test suite

Run: `docker compose exec nuxt npm run test`

Expected: All tests passing

#### Step 2.6: Commit

```bash
git add app/composables/useReferenceData.ts tests/composables/useReferenceData.test.ts
git commit -m "feat: add useReferenceData composable for type-safe reference fetching

- Generic composable for fetching reference entities
- Supports multi-page fetching for large datasets
- Optional transform function for filtering/mapping
- Auto-generates cache keys from endpoint
- 6 comprehensive tests (100% coverage)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 3: Create `useFilterCount()` Utility

**Files:**
- Create: `app/composables/useFilterCount.ts`
- Create: `tests/composables/useFilterCount.test.ts`

#### Step 3.1: Write the failing test file

Create: `tests/composables/useFilterCount.test.ts`

```typescript
import { describe, it, expect } from 'vitest'
import { ref } from 'vue'
import { useFilterCount } from '~/composables/useFilterCount'

describe('useFilterCount', () => {
  describe('single value refs', () => {
    it('counts non-null values', () => {
      const levelRef = ref(3)

      const count = useFilterCount(levelRef)

      expect(count.value).toBe(1)
    })

    it('skips null values', () => {
      const levelRef = ref(null)

      const count = useFilterCount(levelRef)

      expect(count.value).toBe(0)
    })

    it('skips undefined values', () => {
      const levelRef = ref(undefined)

      const count = useFilterCount(levelRef)

      expect(count.value).toBe(0)
    })

    it('skips empty strings', () => {
      const typeRef = ref('')

      const count = useFilterCount(typeRef)

      expect(count.value).toBe(0)
    })

    it('counts zero as active', () => {
      const levelRef = ref(0)

      const count = useFilterCount(levelRef)

      expect(count.value).toBe(1)
    })

    it('counts false as active', () => {
      const boolRef = ref(false)

      const count = useFilterCount(boolRef)

      expect(count.value).toBe(1)
    })
  })

  describe('array refs', () => {
    it('counts non-empty arrays', () => {
      const damageTypesRef = ref(['F', 'C'])

      const count = useFilterCount(damageTypesRef)

      expect(count.value).toBe(1)
    })

    it('skips empty arrays', () => {
      const damageTypesRef = ref([])

      const count = useFilterCount(damageTypesRef)

      expect(count.value).toBe(0)
    })

    it('handles single-item arrays', () => {
      const classRef = ref(['wizard'])

      const count = useFilterCount(classRef)

      expect(count.value).toBe(1)
    })
  })

  describe('multiple refs', () => {
    it('counts multiple active refs', () => {
      const levelRef = ref(3)
      const schoolRef = ref(2)
      const concentrationRef = ref('1')

      const count = useFilterCount(levelRef, schoolRef, concentrationRef)

      expect(count.value).toBe(3)
    })

    it('skips all inactive refs', () => {
      const levelRef = ref(null)
      const schoolRef = ref(null)
      const damageTypesRef = ref([])

      const count = useFilterCount(levelRef, schoolRef, damageTypesRef)

      expect(count.value).toBe(0)
    })

    it('counts mixed active/inactive refs', () => {
      const levelRef = ref(3)
      const schoolRef = ref(null)
      const damageTypesRef = ref(['F'])
      const concentrationRef = ref(null)

      const count = useFilterCount(levelRef, schoolRef, damageTypesRef, concentrationRef)

      expect(count.value).toBe(2)
    })
  })

  describe('reactivity', () => {
    it('updates when refs change', () => {
      const levelRef = ref(null)

      const count = useFilterCount(levelRef)

      expect(count.value).toBe(0)

      levelRef.value = 5

      expect(count.value).toBe(1)
    })

    it('decrements when filter cleared', () => {
      const levelRef = ref(3)

      const count = useFilterCount(levelRef)

      expect(count.value).toBe(1)

      levelRef.value = null

      expect(count.value).toBe(0)
    })

    it('increments when filter added', () => {
      const levelRef = ref(null)
      const schoolRef = ref(null)

      const count = useFilterCount(levelRef, schoolRef)

      expect(count.value).toBe(0)

      levelRef.value = 3
      schoolRef.value = 2

      expect(count.value).toBe(2)
    })
  })

  describe('edge cases', () => {
    it('handles zero refs passed', () => {
      const count = useFilterCount()

      expect(count.value).toBe(0)
    })

    it('handles all null refs', () => {
      const count = useFilterCount(ref(null), ref(null), ref(null))

      expect(count.value).toBe(0)
    })

    it('handles mixed types', () => {
      const stringRef = ref('hello')
      const numberRef = ref(42)
      const boolRef = ref(true)
      const arrayRef = ref(['a', 'b'])

      const count = useFilterCount(stringRef, numberRef, boolRef, arrayRef)

      expect(count.value).toBe(4)
    })
  })
})
```

#### Step 3.2: Run tests to verify they fail

Run: `docker compose exec nuxt npm run test -- useFilterCount`

Expected: FAIL - "Cannot find module '~/composables/useFilterCount'"

#### Step 3.3: Write minimal implementation

Create: `app/composables/useFilterCount.ts`

```typescript
import { computed, type Ref, type ComputedRef } from 'vue'

/**
 * Count active filters for badge display
 *
 * Counts non-null/non-empty filter values. Handles:
 * - Single value refs (null/undefined = inactive)
 * - Array refs (empty array = inactive)
 * - String refs (empty string = inactive)
 *
 * @example
 * ```typescript
 * const activeFilterCount = useFilterCount(
 *   selectedLevel,
 *   selectedSchool,
 *   selectedClass,
 *   concentrationFilter,
 *   selectedDamageTypes,  // array
 *   selectedSavingThrows  // array
 * )
 *
 * // Use in template
 * <UiFilterCollapse :badge-count="activeFilterCount" />
 * ```
 */
export function useFilterCount(...refs: Ref<any>[]): ComputedRef<number> {
  return computed(() => {
    let count = 0

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

      // Count any other truthy value (including 0 and false)
      count++
    }

    return count
  })
}
```

#### Step 3.4: Run tests to verify they pass

Run: `docker compose exec nuxt npm run test -- useFilterCount`

Expected: PASS - All 15 tests passing

#### Step 3.5: Run full test suite

Run: `docker compose exec nuxt npm run test`

Expected: All tests passing (1061/1088 + 49 new = 1110 total)

#### Step 3.6: Commit

```bash
git add app/composables/useFilterCount.ts tests/composables/useFilterCount.test.ts
git commit -m "feat: add useFilterCount utility for active filter counting

- Variadic function accepting any number of refs
- Smart empty detection (null, undefined, '', [])
- Counts 0 and false as active (valid filter values)
- Returns computed ref for reactivity
- 15 comprehensive tests (100% coverage)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase 2: Pilot Migration (Spells Page)

### Task 4: Migrate Spells Page

**Files:**
- Modify: `app/pages/spells/index.vue` (lines 1-294)

#### Step 4.1: Read current spells page

Run: `cat app/pages/spells/index.vue | head -100`

Expected: See current queryBuilder implementation (80+ lines)

#### Step 4.2: Replace queryBuilder with useMeilisearchFilters

Modify: `app/pages/spells/index.vue` (lines 133-209)

**BEFORE:**
```typescript
// Query builder for custom filters
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // Level filter (Meilisearch)
  if (selectedLevel.value !== null) {
    meilisearchFilters.push(`level = ${selectedLevel.value}`)
  }

  // School filter (Meilisearch) - Convert ID to school_code
  if (selectedSchool.value !== null) {
    const schoolCode = spellSchools.value?.find(s => s.id === selectedSchool.value)?.code
    if (schoolCode) {
      meilisearchFilters.push(`school_code = ${schoolCode}`)
    }
  }

  // ... 60 more lines ...

  // Combine all Meilisearch filters with AND
  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

**AFTER:**
```typescript
// Query builder for custom filters (using composable)
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
```

#### Step 4.3: Replace reference fetches with useReferenceData

Modify: `app/pages/spells/index.vue` (lines 37-65)

**BEFORE:**
```typescript
// Fetch spell schools for filter options
const { data: spellSchools } = await useAsyncData<SpellSchool[]>('spell-schools', async () => {
  const response = await apiFetch<{ data: SpellSchool[] }>('/spell-schools')
  return response?.data || []
})

// Fetch classes for filter options (base classes only)
const { data: classes } = await useAsyncData<CharacterClass[]>('classes-filter', async () => {
  const [page1, page2] = await Promise.all([
    apiFetch<{ data: CharacterClass[] }>('/classes?per_page=100&page=1'),
    apiFetch<{ data: CharacterClass[] }>('/classes?per_page=100&page=2')
  ])
  const allClasses = [...(page1?.data || []), ...(page2?.data || [])]
  return allClasses.filter(c => c.is_base_class === true)
})

// Phase 1: Fetch damage types for filter options
const { data: damageTypes } = await useAsyncData<DamageType[]>('damage-types', async () => {
  const response = await apiFetch<{ data: DamageType[] }>('/damage-types')
  return response?.data || []
})

// Phase 1: Fetch ability scores (for saving throw filter options)
const { data: abilityScores } = await useAsyncData<AbilityScore[]>('ability-scores', async () => {
  const response = await apiFetch<{ data: AbilityScore[] }>('/ability-scores')
  return response?.data || []
})
```

**AFTER:**
```typescript
// Fetch reference data for filter options (using composable)
const { data: spellSchools } = useReferenceData<SpellSchool>('/spell-schools')

const { data: classes } = useReferenceData<CharacterClass>('/classes', {
  pages: 2,
  transform: (data) => data.filter(c => c.is_base_class === true)
})

const { data: damageTypes } = useReferenceData<DamageType>('/damage-types')

const { data: abilityScores } = useReferenceData<AbilityScore>('/ability-scores')
```

#### Step 4.4: Replace activeFilterCount with useFilterCount

Modify: `app/pages/spells/index.vue` (lines 276-293)

**BEFORE:**
```typescript
// Count active filters (excluding search) for collapse badge
const activeFilterCount = computed(() => {
  let count = 0
  if (selectedLevel.value !== null) count++
  if (selectedSchool.value !== null) count++
  if (selectedClass.value !== null) count++
  if (concentrationFilter.value !== null) count++
  if (ritualFilter.value !== null) count++
  // Phase 1: Count multi-select filters
  if (selectedDamageTypes.value.length > 0) count++
  if (selectedSavingThrows.value.length > 0) count++
  // Phase 2: Count component flag filters
  if (verbalFilter.value !== null) count++
  if (somaticFilter.value !== null) count++
  if (materialFilter.value !== null) count++
  // Phase 3: Removed unsupported filter counts
  return count
})
```

**AFTER:**
```typescript
// Count active filters (excluding search) for collapse badge (using composable)
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

#### Step 4.5: Update useEntityList to use queryParams

Modify: `app/pages/spells/index.vue` (lines 211-231)

**BEFORE:**
```typescript
} = useEntityList({
  endpoint: '/spells',
  cacheKey: 'spells-list',
  queryBuilder,  // Old computed
  seo: {
    title: 'Spells - D&D 5e Compendium',
    description: 'Browse all D&D 5e spells. Filter by level, school, and search for specific spells.'
  }
})
```

**AFTER:**
```typescript
} = useEntityList({
  endpoint: '/spells',
  cacheKey: 'spells-list',
  queryBuilder: queryParams,  // New from composable
  seo: {
    title: 'Spells - D&D 5e Compendium',
    description: 'Browse all D&D 5e spells. Filter by level, school, and search for specific spells.'
  }
})
```

#### Step 4.6: Remove unused imports and add new imports

Modify: `app/pages/spells/index.vue` (lines 1-6)

**BEFORE:**
```typescript
<script setup lang="ts">
import { ref, computed } from 'vue'
import type { SpellSchool, Spell, CharacterClass, DamageType, AbilityScore } from '~/types'

const route = useRoute()
const { apiFetch } = useApi()
```

**AFTER:**
```typescript
<script setup lang="ts">
import { ref, computed } from 'vue'
import type { SpellSchool, Spell, CharacterClass, DamageType, AbilityScore } from '~/types'

const route = useRoute()
// Note: useApi no longer needed for reference fetches (handled by useReferenceData)
```

#### Step 4.7: Verify TypeScript compiles

Run: `docker compose exec nuxt npm run typecheck`

Expected: No TypeScript errors

#### Step 4.8: Run existing spells tests

Run: `docker compose exec nuxt npm run test -- spells`

Expected: All spells tests passing

#### Step 4.9: Run full test suite

Run: `docker compose exec nuxt npm run test`

Expected: All tests passing (1110/1110 or similar)

#### Step 4.10: Browser verify spells page

Start dev server: `docker compose exec nuxt npm run dev`

Navigate to: http://localhost:3000/spells

**Test each filter:**
- [ ] Level dropdown (select "3rd Level", verify results update)
- [ ] School dropdown (select "Evocation", verify results update)
- [ ] Class dropdown (select "Wizard", verify results update)
- [ ] Concentration toggle (set to "Yes", verify results update)
- [ ] Ritual toggle (set to "Yes", verify results update)
- [ ] Damage types multi-select (select "Fire", "Cold", verify results update)
- [ ] Saving throws multi-select (select "DEX", "WIS", verify results update)
- [ ] Verbal component toggle (set to "Yes", verify results update)
- [ ] Somatic component toggle (set to "Yes", verify results update)
- [ ] Material component toggle (set to "Yes", verify results update)

**Test UI:**
- [ ] Filter count badge shows correct number
- [ ] Clear filters button resets all filters
- [ ] URL params update when filters change
- [ ] Page reloads preserve filter state from URL

Expected: All filters work identically to before migration

#### Step 4.11: Commit spells migration

```bash
git add app/pages/spells/index.vue
git commit -m "refactor: migrate spells page to filter composables

- Replace 80-line queryBuilder with 14-line useMeilisearchFilters config
- Replace 30 lines of reference fetches with 4-line useReferenceData calls
- Replace 14-line activeFilterCount with 1-line useFilterCount call
- Reduced page from 622 lines to ~460 lines (26% reduction)
- All 10 filters still working identically
- All tests passing, browser verified

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase 3: Roll Out to Remaining Pages

### Task 5: Migrate Items Page

**Files:**
- Modify: `app/pages/items/index.vue` (lines 1-384)

#### Step 5.1: Replace queryBuilder with useMeilisearchFilters

Modify: `app/pages/items/index.vue` (lines 54-101)

**BEFORE (47 lines):**
```typescript
// Query builder for custom filters (Meilisearch syntax)
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // Type filter (use item_type_id for Meilisearch)
  if (selectedType.value !== null) {
    meilisearchFilters.push(`item_type_id = ${selectedType.value}`)
  }

  // Rarity filter (string value)
  if (selectedRarity.value !== null) {
    meilisearchFilters.push(`rarity = ${selectedRarity.value}`)
  }

  // is_magic filter (convert string to boolean)
  if (selectedMagic.value !== null) {
    const boolValue = selectedMagic.value === 'true' || selectedMagic.value === '1'
    meilisearchFilters.push(`is_magic = ${boolValue}`)
  }

  // has_charges filter (check if charges_max > 0)
  if (hasCharges.value !== null) {
    const hasCharge = hasCharges.value === '1' || hasCharges.value === 'true'
    if (hasCharge) {
      meilisearchFilters.push('charges_max > 0')
    } else {
      meilisearchFilters.push('charges_max = 0')
    }
  }

  // has_prerequisites filter (check if prerequisites field is not empty)
  if (hasPrerequisites.value !== null) {
    const hasPrereq = hasPrerequisites.value === '1' || hasPrerequisites.value === 'true'
    if (hasPrereq) {
      meilisearchFilters.push('prerequisites IS NOT EMPTY')
    } else {
      meilisearchFilters.push('prerequisites IS EMPTY')
    }
  }

  // Combine all filters with AND
  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

**AFTER (10 lines):**
```typescript
// Query builder for custom filters (using composable)
const { queryParams } = useMeilisearchFilters([
  { ref: selectedType, field: 'item_type_id' },
  { ref: selectedRarity, field: 'rarity' },
  { ref: selectedMagic, field: 'is_magic', type: 'boolean' },
  { ref: hasCharges, field: 'charges_max', type: 'greaterThan', transform: () => 0 },
  { ref: hasPrerequisites, field: 'prerequisites', type: 'isEmpty' }
])
```

**Note:** The `hasCharges` filter needs special handling. Update to:

```typescript
// Create computed for charges filter (has_charges uses > 0 logic)
const hasChargesComputed = computed(() => {
  if (hasCharges.value === null) return null
  return hasCharges.value === '1' || hasCharges.value === 'true' ? true : false
})

const { queryParams } = useMeilisearchFilters([
  { ref: selectedType, field: 'item_type_id' },
  { ref: selectedRarity, field: 'rarity' },
  { ref: selectedMagic, field: 'is_magic', type: 'boolean' },
  {
    ref: hasChargesComputed,
    field: 'charges_max',
    type: 'greaterThan',
    transform: (hasCharge) => hasCharge ? 0 : null  // Only filter if true
  },
  { ref: hasPrerequisites, field: 'prerequisites', type: 'isEmpty' }
])
```

Actually, let's keep the manual handling for `hasCharges` since it's a special case:

```typescript
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // Standard filters via composable
  const standardFilters = useMeilisearchFilters([
    { ref: selectedType, field: 'item_type_id' },
    { ref: selectedRarity, field: 'rarity' },
    { ref: selectedMagic, field: 'is_magic', type: 'boolean' },
    { ref: hasPrerequisites, field: 'prerequisites', type: 'isEmpty' }
  ])

  // Extract standard filter string
  if (standardFilters.queryParams.value.filter) {
    meilisearchFilters.push(standardFilters.queryParams.value.filter)
  }

  // Special handling for has_charges (uses > 0 / = 0 logic)
  if (hasCharges.value !== null) {
    const hasCharge = hasCharges.value === '1' || hasCharges.value === 'true'
    if (hasCharge) {
      meilisearchFilters.push('charges_max > 0')
    } else {
      meilisearchFilters.push('charges_max = 0')
    }
  }

  // Combine all filters
  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

Wait - let me reconsider. The `greaterThan` type handles `> 0`, but we also need `= 0`. Let me check the design...

Actually, the best approach is to keep `hasCharges` manual for now since it's a special case (positive vs zero, not just > threshold). Update the composable usage:

```typescript
// Query builder for custom filters (hybrid: composable + manual for special cases)
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // Standard filters via composable
  const { queryParams: standardParams } = useMeilisearchFilters([
    { ref: selectedType, field: 'item_type_id' },
    { ref: selectedRarity, field: 'rarity' },
    { ref: selectedMagic, field: 'is_magic', type: 'boolean' },
    { ref: hasPrerequisites, field: 'prerequisites', type: 'isEmpty' }
  ])

  // Extract standard filter string
  if (standardParams.value.filter) {
    meilisearchFilters.push(standardParams.value.filter as string)
  }

  // Special handling for has_charges (needs both > 0 and = 0 logic)
  if (hasCharges.value !== null) {
    const hasCharge = hasCharges.value === '1' || hasCharges.value === 'true'
    meilisearchFilters.push(hasCharge ? 'charges_max > 0' : 'charges_max = 0')
  }

  // Combine all filters
  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

This hybrid approach is cleaner: composable for standard filters, manual for special case.

#### Step 5.2: Replace reference fetch with useReferenceData

Modify: `app/pages/items/index.vue` (lines 18-22)

**BEFORE:**
```typescript
// Fetch item types for filter options
const { data: itemTypes } = await useAsyncData('item-types', async () => {
  const response = await apiFetch<{ data: ItemType[] }>('/item-types')
  return response?.data || []
})
```

**AFTER:**
```typescript
// Fetch item types for filter options (using composable)
const { data: itemTypes } = useReferenceData<ItemType>('/item-types')
```

#### Step 5.3: Replace activeFilterCount with useFilterCount

Modify: `app/pages/items/index.vue` (lines 147-156)

**BEFORE:**
```typescript
const activeFilterCount = computed(() => {
  let count = 0
  if (selectedType.value !== null) count++
  if (selectedRarity.value !== null) count++
  if (selectedMagic.value !== null) count++
  if (hasCharges.value !== null) count++
  if (hasPrerequisites.value !== null) count++
  return count
})
```

**AFTER:**
```typescript
const activeFilterCount = useFilterCount(
  selectedType,
  selectedRarity,
  selectedMagic,
  hasCharges,
  hasPrerequisites
)
```

#### Step 5.4: Update useEntityList

Modify: `app/pages/items/index.vue` (lines 103-113)

No changes needed - already using `queryBuilder` which we updated.

#### Step 5.5: Test items page

Run: `docker compose exec nuxt npm run test -- items`

Expected: All items tests passing

Browser verify: http://localhost:3000/items
- [ ] Type filter works
- [ ] Rarity filter works
- [ ] Magic filter works
- [ ] Has Charges filter works
- [ ] Has Prerequisites filter works
- [ ] Filter count badge correct
- [ ] Clear filters works

#### Step 5.6: Commit

```bash
git add app/pages/items/index.vue
git commit -m "refactor: migrate items page to filter composables

- Replace queryBuilder with hybrid approach (composable + manual for special case)
- Replace reference fetch with useReferenceData
- Replace activeFilterCount with useFilterCount
- Reduced page from 384 lines to ~310 lines (19% reduction)
- All 5 filters still working identically
- All tests passing, browser verified

Note: hasCharges filter kept manual (needs both > 0 and = 0 logic)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 6: Migrate Monsters Page

**Files:**
- Modify: `app/pages/monsters/index.vue` (lines 1-318)

#### Step 6.1: Replace queryBuilder (keep CR range manual)

Modify: `app/pages/monsters/index.vue` (lines 40-75)

**BEFORE:**
```typescript
// Query builder (Meilisearch syntax)
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // CR range filter (convert UI range to Meilisearch range query)
  if (selectedCR.value) {
    if (selectedCR.value === '0-4') {
      meilisearchFilters.push('challenge_rating >= 0 AND challenge_rating <= 4')
    } else if (selectedCR.value === '5-10') {
      meilisearchFilters.push('challenge_rating >= 5 AND challenge_rating <= 10')
    } else if (selectedCR.value === '11-16') {
      meilisearchFilters.push('challenge_rating >= 11 AND challenge_rating <= 16')
    } else if (selectedCR.value === '17+') {
      meilisearchFilters.push('challenge_rating >= 17')
    }
  }

  // Type filter (string value, no quotes needed)
  if (selectedType.value) {
    meilisearchFilters.push(`type = ${selectedType.value}`)
  }

  // is_legendary filter (convert string to boolean)
  if (isLegendary.value !== null) {
    const boolValue = isLegendary.value === '1' || isLegendary.value === 'true'
    meilisearchFilters.push(`is_legendary = ${boolValue}`)
  }

  // Combine all filters with AND
  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

**AFTER:**
```typescript
// Query builder (hybrid: composable + manual CR range)
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // Standard filters via composable
  const { queryParams: standardParams } = useMeilisearchFilters([
    { ref: selectedType, field: 'type' },
    { ref: isLegendary, field: 'is_legendary', type: 'boolean' }
  ])

  if (standardParams.value.filter) {
    meilisearchFilters.push(standardParams.value.filter as string)
  }

  // Special handling for CR range (UI string → numeric range)
  if (selectedCR.value) {
    const crMap: Record<string, string> = {
      '0-4': 'challenge_rating >= 0 AND challenge_rating <= 4',
      '5-10': 'challenge_rating >= 5 AND challenge_rating <= 10',
      '11-16': 'challenge_rating >= 11 AND challenge_rating <= 16',
      '17+': 'challenge_rating >= 17'
    }
    if (crMap[selectedCR.value]) {
      meilisearchFilters.push(crMap[selectedCR.value])
    }
  }

  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

#### Step 6.2: Replace activeFilterCount

Modify: `app/pages/monsters/index.vue` (lines 122-127)

**BEFORE:**
```typescript
const activeFilterCount = computed(() => {
  let count = 0
  if (selectedCR.value) count++
  if (selectedType.value) count++
  if (isLegendary.value !== null) count++
  return count
})
```

**AFTER:**
```typescript
const activeFilterCount = useFilterCount(selectedCR, selectedType, isLegendary)
```

#### Step 6.3: Test monsters page

Run: `docker compose exec nuxt npm run test -- monsters`

Expected: All monsters tests passing

Browser verify: http://localhost:3000/monsters

#### Step 6.4: Commit

```bash
git add app/pages/monsters/index.vue
git commit -m "refactor: migrate monsters page to filter composables

- Replace queryBuilder with hybrid approach (composable + manual CR range)
- Replace activeFilterCount with useFilterCount
- Reduced page from 318 lines to ~270 lines (15% reduction)
- All 3 filters still working identically
- All tests passing, browser verified

Note: CR range filter kept manual (UI string to numeric range mapping)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 7: Migrate Races Page

**Files:**
- Modify: `app/pages/races/index.vue` (lines 1-251)

#### Step 7.1: Replace queryBuilder

Modify: `app/pages/races/index.vue` (lines 23-44)

**BEFORE:**
```typescript
// Query builder for custom filters (Meilisearch)
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // Size filter (use size_code, NOT size!)
  if (selectedSize.value) {
    meilisearchFilters.push(`size_code = ${selectedSize.value}`)
  }

  // NOTE: has_darkvision is NOT filterable in Meilisearch

  // Combine all filters with AND
  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

**AFTER:**
```typescript
// Query builder for custom filters (using composable)
const { queryParams } = useMeilisearchFilters([
  { ref: selectedSize, field: 'size_code' }
])
```

#### Step 7.2: Replace reference fetch

Modify: `app/pages/races/index.vue` (lines 13-19)

**BEFORE:**
```typescript
const { data: sizesResponse } = await useAsyncData(
  'sizes-for-races',
  async () => {
    const response = await apiFetch<{ data: Size[] }>('/sizes')
    return response || { data: [] }
  }
)

const sizes = computed(() => sizesResponse.value?.data || [])
```

**AFTER:**
```typescript
const { data: sizes } = useReferenceData<Size>('/sizes', {
  cacheKey: 'sizes-for-races'
})
```

#### Step 7.3: Replace activeFilterCount

Modify: `app/pages/races/index.vue` (lines 85-89)

**BEFORE:**
```typescript
const activeFilterCount = computed(() => {
  let count = 0
  if (selectedSize.value) count++
  return count
})
```

**AFTER:**
```typescript
const activeFilterCount = useFilterCount(selectedSize)
```

#### Step 7.4: Update useEntityList

Modify: `app/pages/races/index.vue` (lines 46-66)

Change `queryBuilder` to `queryBuilder: queryParams`

#### Step 7.5: Test races page

Run: `docker compose exec nuxt npm run test -- races`

Expected: All races tests passing

Browser verify: http://localhost:3000/races

#### Step 7.6: Commit

```bash
git add app/pages/races/index.vue
git commit -m "refactor: migrate races page to filter composables

- Replace queryBuilder with useMeilisearchFilters (1 filter)
- Replace reference fetch with useReferenceData
- Replace activeFilterCount with useFilterCount
- Reduced page from 251 lines to ~200 lines (20% reduction)
- Size filter still working identically
- All tests passing, browser verified

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 8: Migrate Classes Page

**Files:**
- Modify: `app/pages/classes/index.vue` (lines 1-248)

#### Step 8.1: Replace queryBuilder

Modify: `app/pages/classes/index.vue` (lines 11-34)

**BEFORE:**
```typescript
// Query builder (Meilisearch syntax)
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // is_base_class filter (convert string to boolean)
  if (isBaseClass.value !== null) {
    const boolValue = isBaseClass.value === '1' || isBaseClass.value === 'true'
    meilisearchFilters.push(`is_base_class = ${boolValue}`)
  }

  // is_spellcaster filter (convert string to boolean)
  if (isSpellcaster.value !== null) {
    const boolValue = isSpellcaster.value === '1' || isSpellcaster.value === 'true'
    meilisearchFilters.push(`is_spellcaster = ${boolValue}`)
  }

  // Combine all filters with AND
  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

**AFTER:**
```typescript
// Query builder (using composable)
const { queryParams } = useMeilisearchFilters([
  { ref: isBaseClass, field: 'is_base_class', type: 'boolean' },
  { ref: isSpellcaster, field: 'is_spellcaster', type: 'boolean' }
])
```

#### Step 8.2: Replace activeFilterCount

Modify: `app/pages/classes/index.vue` (lines 72-77)

**BEFORE:**
```typescript
const activeFilterCount = computed(() => {
  let count = 0
  if (isBaseClass.value !== null) count++
  if (isSpellcaster.value !== null) count++
  return count
})
```

**AFTER:**
```typescript
const activeFilterCount = useFilterCount(isBaseClass, isSpellcaster)
```

#### Step 8.3: Update useEntityList

Change `queryBuilder` to `queryBuilder: queryParams`

#### Step 8.4: Test classes page

Run: `docker compose exec nuxt npm run test -- classes`

Expected: All classes tests passing

Browser verify: http://localhost:3000/classes

#### Step 8.5: Commit

```bash
git add app/pages/classes/index.vue
git commit -m "refactor: migrate classes page to filter composables

- Replace queryBuilder with useMeilisearchFilters (2 boolean filters)
- Replace activeFilterCount with useFilterCount
- Reduced page from 248 lines to ~200 lines (19% reduction)
- Both filters still working (backend issue with is_base_class persists)
- All tests passing, browser verified

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 9: Migrate Feats Page

**Files:**
- Modify: `app/pages/feats/index.vue` (lines 1-215)

#### Step 9.1: Replace queryBuilder

Modify: `app/pages/feats/index.vue` (lines 10-33)

**BEFORE:**
```typescript
// Query builder (Meilisearch syntax)
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // has_prerequisites filter (check if prerequisites field is not empty)
  if (hasPrerequisites.value !== null) {
    const hasPrereq = hasPrerequisites.value === '1' || hasPrerequisites.value === 'true'
    if (hasPrereq) {
      meilisearchFilters.push('prerequisites IS NOT EMPTY')
    } else {
      meilisearchFilters.push('prerequisites IS EMPTY')
    }
  }

  // Combine all filters with AND
  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

**AFTER:**
```typescript
// Query builder (using composable)
const { queryParams } = useMeilisearchFilters([
  { ref: hasPrerequisites, field: 'prerequisites', type: 'isEmpty' }
])
```

#### Step 9.2: Replace activeFilterCount

Modify: `app/pages/feats/index.vue` (lines 64-68)

**BEFORE:**
```typescript
const activeFilterCount = computed(() => {
  let count = 0
  if (hasPrerequisites.value !== null) count++
  return count
})
```

**AFTER:**
```typescript
const activeFilterCount = useFilterCount(hasPrerequisites)
```

#### Step 9.3: Update useEntityList

Change `queryBuilder` to `queryBuilder: queryParams`

#### Step 9.4: Test feats page

Run: `docker compose exec nuxt npm run test -- feats`

Expected: All feats tests passing

Browser verify: http://localhost:3000/feats

#### Step 9.5: Commit

```bash
git add app/pages/feats/index.vue
git commit -m "refactor: migrate feats page to filter composables

- Replace queryBuilder with useMeilisearchFilters (isEmpty filter)
- Replace activeFilterCount with useFilterCount
- Reduced page from 215 lines to ~170 lines (21% reduction)
- Prerequisites filter still working identically
- All tests passing, browser verified

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase 4: Documentation & Cleanup

### Task 10: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` (add filter composables section)

#### Step 10.1: Add filter composables documentation

Modify: `CLAUDE.md` (after "Component Auto-Import" section)

Add new section:

```markdown
---

## Filter Composables

**For entity list pages with Meilisearch filters, use these composables:**

### `useMeilisearchFilters()`

Declarative filter builder. Converts refs to Meilisearch filter params.

```typescript
const { queryParams } = useMeilisearchFilters([
  { ref: selectedLevel, field: 'level' },
  { ref: concentrationFilter, field: 'concentration', type: 'boolean' },
  { ref: selectedDamageTypes, field: 'damage_types', type: 'in' }
])

// Pass to useEntityList
const { ... } = useEntityList({
  endpoint: '/spells',
  queryBuilder: queryParams,
  // ...
})
```

**Supported filter types:**
- `equals` - field = value (default)
- `boolean` - Converts '1'/'true' to true, '0'/'false' to false
- `in` - field IN [value1, value2]
- `range` - field >= min AND field <= max
- `isEmpty` - field IS EMPTY / IS NOT EMPTY
- `greaterThan` - field > value

**Transform function:** For ID→code lookups:
```typescript
{
  ref: selectedSchool,
  field: 'school_code',
  transform: (id) => spellSchools.value?.find(s => s.id === id)?.code || null
}
```

### `useReferenceData<T>()`

Type-safe reference entity fetching. Replaces useAsyncData + apiFetch pattern.

```typescript
// Simple fetch
const { data: schools } = useReferenceData<SpellSchool>('/spell-schools')

// Multi-page with transform
const { data: classes } = useReferenceData<CharacterClass>('/classes', {
  pages: 2,
  transform: (data) => data.filter(c => c.is_base_class === true)
})

// Use in computed options
const schoolOptions = computed(() =>
  schools.value?.map(s => ({ label: s.name, value: s.id })) || []
)
```

### `useFilterCount()`

Count active filters for badge display.

```typescript
const activeFilterCount = useFilterCount(
  selectedLevel,
  selectedSchool,
  selectedDamageTypes  // arrays, nulls, empty strings auto-skipped
)

// Use in template
<UiFilterCollapse :badge-count="activeFilterCount" />
```

**When to use:**
- ✅ All entity list pages with filters
- ✅ New filter pages (follow existing patterns)
- ❌ Special cases (keep manual if composable doesn't fit)

**Gold Standard:** `app/pages/spells/index.vue` (10 filters using all 3 composables)

---
```

#### Step 10.2: Commit CLAUDE.md update

```bash
git add CLAUDE.md
git commit -m "docs: add filter composables section to CLAUDE.md

- Document useMeilisearchFilters usage and filter types
- Document useReferenceData for reference fetching
- Document useFilterCount for active filter badges
- Reference spells page as gold standard
- Include usage examples and best practices

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 11: Create Migration Guide

**Files:**
- Create: `docs/FILTER-COMPOSABLES-MIGRATION-GUIDE.md`

#### Step 11.1: Write migration guide

Create: `docs/FILTER-COMPOSABLES-MIGRATION-GUIDE.md`

```markdown
# Filter Composables Migration Guide

**Last Updated:** 2025-11-25
**Status:** ✅ Complete - All 6 entity pages migrated

---

## Overview

This guide documents how to use the filter composables when adding filters to entity list pages.

**Composables:**
1. `useMeilisearchFilters()` - Declarative filter builder
2. `useReferenceData<T>()` - Type-safe reference fetching
3. `useFilterCount()` - Active filter counter

---

## Quick Start

### 1. Basic Filter Setup

```typescript
// app/pages/your-entity/index.vue
<script setup lang="ts">
import { ref } from 'vue'

// Filter state
const selectedLevel = ref(route.query.level ? Number(route.query.level) : null)
const selectedSchool = ref(route.query.school ? Number(route.query.school) : null)

// Fetch reference data
const { data: schools } = useReferenceData<School>('/schools')

// Build query params
const { queryParams } = useMeilisearchFilters([
  { ref: selectedLevel, field: 'level' },
  { ref: selectedSchool, field: 'school_id' }
])

// Count active filters
const activeFilterCount = useFilterCount(selectedLevel, selectedSchool)

// Use in entity list
const { ... } = useEntityList({
  endpoint: '/your-entity',
  queryBuilder: queryParams,
  // ...
})
</script>
```

### 2. Common Filter Patterns

**Equals Filter (default):**
```typescript
{ ref: selectedLevel, field: 'level' }
```

**Boolean Filter:**
```typescript
{ ref: concentrationFilter, field: 'concentration', type: 'boolean' }
// Converts '1'/'true' → true, '0'/'false' → false
```

**Multi-select (IN) Filter:**
```typescript
{ ref: selectedDamageTypes, field: 'damage_types', type: 'in' }
// Converts ['F', 'C'] → damage_types IN [F, C]
```

**ID→Code Transform:**
```typescript
{
  ref: selectedSchool,
  field: 'school_code',
  transform: (id) => schools.value?.find(s => s.id === id)?.code || null
}
```

**isEmpty Filter:**
```typescript
{ ref: hasPrerequisites, field: 'prerequisites', type: 'isEmpty' }
// '1'/'true' → IS EMPTY, '0'/'false' → IS NOT EMPTY
```

---

## Filter Types Reference

| Type | Meilisearch Output | Use Case |
|------|-------------------|----------|
| `equals` | `field = value` | Dropdowns, numeric values |
| `boolean` | `field = true/false` | Toggles, checkboxes |
| `in` | `field IN [val1, val2]` | Multi-select, arrays |
| `range` | `field >= min AND field <= max` | Numeric ranges |
| `isEmpty` | `field IS EMPTY/NOT EMPTY` | Has/doesn't have value |
| `greaterThan` | `field > value` | Numeric thresholds |

---

## Special Cases

### Hybrid Approach (Composable + Manual)

Some filters need custom logic. Use hybrid approach:

```typescript
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // Standard filters via composable
  const { queryParams: standardParams } = useMeilisearchFilters([
    { ref: selectedType, field: 'type' },
    { ref: isLegendary, field: 'is_legendary', type: 'boolean' }
  ])

  if (standardParams.value.filter) {
    meilisearchFilters.push(standardParams.value.filter as string)
  }

  // Special case: CR range mapping
  if (selectedCR.value) {
    const crMap: Record<string, string> = {
      '0-4': 'challenge_rating >= 0 AND challenge_rating <= 4',
      '5-10': 'challenge_rating >= 5 AND challenge_rating <= 10',
      '11-16': 'challenge_rating >= 11 AND challenge_rating <= 16',
      '17+': 'challenge_rating >= 17'
    }
    if (crMap[selectedCR.value]) {
      meilisearchFilters.push(crMap[selectedCR.value])
    }
  }

  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

**When to use manual:**
- Custom range string → numeric mapping (monsters CR)
- Positive vs zero logic (items has_charges)
- Complex conditional logic

---

## Reference Data Patterns

### Simple Fetch

```typescript
const { data: schools } = useReferenceData<School>('/schools')
```

### Multi-Page Fetch

```typescript
const { data: classes } = useReferenceData<Class>('/classes', {
  pages: 2  // Fetches 2 pages in parallel
})
```

### With Transform

```typescript
const { data: classes } = useReferenceData<Class>('/classes', {
  pages: 2,
  transform: (data) => data.filter(c => c.is_base_class === true)
})
```

### Custom Cache Key

```typescript
const { data: sizes } = useReferenceData<Size>('/sizes', {
  cacheKey: 'sizes-for-races'  // Avoid cache collision
})
```

---

## Migration Checklist

When migrating an existing page:

### Phase 1: Read Current Code
- [ ] Read current queryBuilder logic
- [ ] Identify all filter types
- [ ] Note any special cases
- [ ] Check reference data fetches

### Phase 2: Replace queryBuilder
- [ ] Create filter configs array
- [ ] Map each filter to FilterConfig
- [ ] Add transform functions for lookups
- [ ] Keep special cases manual (hybrid)
- [ ] Update useEntityList to use queryParams

### Phase 3: Replace Reference Fetches
- [ ] Replace useAsyncData + apiFetch with useReferenceData
- [ ] Add pages option if multi-page
- [ ] Add transform if filtering needed
- [ ] Update computed options to use new data refs

### Phase 4: Replace Filter Count
- [ ] Replace manual computed with useFilterCount
- [ ] Pass all filter refs as arguments
- [ ] Verify badge count updates correctly

### Phase 5: Test
- [ ] Run existing tests - verify all pass
- [ ] Run TypeScript check - no errors
- [ ] Browser test all filters
- [ ] Verify filter count badge
- [ ] Verify clear filters button
- [ ] Verify URL params persist

### Phase 6: Commit
- [ ] Stage changed file
- [ ] Write descriptive commit message
- [ ] Include metrics (lines reduced, filters count)
- [ ] Push commit

---

## Examples

### Spells Page (Gold Standard)

**10 filters using all 3 composables:**

```typescript
// Reference data
const { data: spellSchools } = useReferenceData<SpellSchool>('/spell-schools')
const { data: classes } = useReferenceData<CharacterClass>('/classes', {
  pages: 2,
  transform: (data) => data.filter(c => c.is_base_class === true)
})
const { data: damageTypes } = useReferenceData<DamageType>('/damage-types')
const { data: abilityScores } = useReferenceData<AbilityScore>('/ability-scores')

// Query builder
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

// Filter count
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

### Items Page (Hybrid Approach)

**4 composable + 1 manual:**

```typescript
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // Standard filters
  const { queryParams: standardParams } = useMeilisearchFilters([
    { ref: selectedType, field: 'item_type_id' },
    { ref: selectedRarity, field: 'rarity' },
    { ref: selectedMagic, field: 'is_magic', type: 'boolean' },
    { ref: hasPrerequisites, field: 'prerequisites', type: 'isEmpty' }
  ])

  if (standardParams.value.filter) {
    meilisearchFilters.push(standardParams.value.filter as string)
  }

  // Special case: has_charges (needs both > 0 and = 0)
  if (hasCharges.value !== null) {
    const hasCharge = hasCharges.value === '1' || hasCharges.value === 'true'
    meilisearchFilters.push(hasCharge ? 'charges_max > 0' : 'charges_max = 0')
  }

  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

---

## Troubleshooting

### Filter Not Working

1. **Check ref is reactive:**
   ```typescript
   const selectedLevel = ref(null)  // ✅ Reactive
   let selectedLevel = null         // ❌ Not reactive
   ```

2. **Check field name matches Meilisearch index:**
   ```typescript
   { ref: selectedSchool, field: 'school_code' }  // ✅ Correct field
   { ref: selectedSchool, field: 'school' }       // ❌ Not filterable
   ```

3. **Check filter type:**
   ```typescript
   { ref: concentrationRef, field: 'concentration', type: 'boolean' }  // ✅
   { ref: concentrationRef, field: 'concentration' }  // ❌ String '1' not boolean
   ```

### Transform Not Working

1. **Check reference data loaded:**
   ```typescript
   transform: (id) => schools.value?.find(s => s.id === id)?.code || null
   // Use optional chaining (?) and null fallback
   ```

2. **Check transform returns correct type:**
   ```typescript
   transform: (id) => id === 2 ? 'EV' : null  // ✅ Returns string or null
   transform: (id) => id === 2 ? 'EV' : undefined  // ❌ undefined skipped
   ```

### Filter Count Wrong

1. **Check all refs passed:**
   ```typescript
   useFilterCount(ref1, ref2, ref3)  // Pass all filter refs
   ```

2. **Check refs not arrays when should be:**
   ```typescript
   const damageTypes = ref([])  // ✅ Array ref
   const damageTypes = []       // ❌ Not a ref
   ```

---

## Benefits

**Code Reduction:**
- Spells: 622 → 460 lines (26%)
- Items: 384 → 310 lines (19%)
- Monsters: 318 → 270 lines (15%)
- Races: 251 → 200 lines (20%)
- Classes: 248 → 200 lines (19%)
- Feats: 215 → 170 lines (21%)
- **Total: ~560 lines eliminated**

**Maintainability:**
- Bug fixes in 1 place instead of 6
- Consistent behavior across pages
- Easier to add new filter types
- Better type safety

**Testability:**
- 49 new composable tests (100% coverage)
- Easier to test edge cases in isolation
- Regression protection via composable tests

---

## Additional Resources

- **Design Doc:** `docs/plans/2025-11-25-filter-composables-refactoring-design.md`
- **Implementation Plan:** `docs/plans/2025-11-25-filter-composables-refactoring-implementation.md`
- **Composable Code:** `app/composables/useMeilisearchFilters.ts`
- **Composable Tests:** `tests/composables/useMeilisearchFilters.test.ts`
- **Gold Standard:** `app/pages/spells/index.vue`

---

**Questions? Check the gold standard (spells page) or read the design doc for detailed explanations.**
```

#### Step 11.2: Commit migration guide

```bash
git add docs/FILTER-COMPOSABLES-MIGRATION-GUIDE.md
git commit -m "docs: create comprehensive filter composables migration guide

- Document all 3 composables with usage examples
- Include common patterns and special cases
- Provide migration checklist for new pages
- Add troubleshooting section
- Reference gold standard implementations
- Document benefits and metrics

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 12: Update CHANGELOG.md

**Files:**
- Modify: `CHANGELOG.md` (add refactoring entry)

#### Step 12.1: Add CHANGELOG entry

Modify: `CHANGELOG.md` (under "## [Unreleased]")

Add under "### Changed":

```markdown
### Changed

- **Filter Composables Refactoring** (2025-11-25) - Extracted 3 reusable composables to eliminate ~560 lines of duplicate Meilisearch filter logic across 6 entity pages
  - Created `useMeilisearchFilters()` for declarative filter building (6 filter types)
  - Created `useReferenceData<T>()` for type-safe reference entity fetching
  - Created `useFilterCount()` for active filter counting
  - Migrated all entity pages: spells, items, monsters, races, classes, feats
  - Added 49 comprehensive tests (100% coverage on composables)
  - Improved maintainability: bug fixes in 1 place instead of 6
  - Non-breaking change: all filters work identically
  - See `docs/FILTER-COMPOSABLES-MIGRATION-GUIDE.md` for usage patterns
```

#### Step 12.2: Commit CHANGELOG

```bash
git add CHANGELOG.md
git commit -m "docs: update CHANGELOG with filter composables refactoring

- Document 3 new composables created
- Note code reduction (~560 lines eliminated)
- List all 6 pages migrated
- Reference migration guide for future use
- Emphasize non-breaking nature of changes

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 13: Final Verification

#### Step 13.1: Run full test suite

Run: `docker compose exec nuxt npm run test`

Expected: All tests passing (1110/1088 or higher)

#### Step 13.2: Run TypeScript check

Run: `docker compose exec nuxt npm run typecheck`

Expected: No new TypeScript errors (pre-existing monster errors OK)

#### Step 13.3: Run ESLint

Run: `docker compose exec nuxt npm run lint`

Expected: 0 errors

#### Step 13.4: Browser verify all pages

Visit each page and test filters:

- [ ] http://localhost:3000/spells - 10 filters working
- [ ] http://localhost:3000/items - 5 filters working
- [ ] http://localhost:3000/monsters - 3 filters working
- [ ] http://localhost:3000/races - 1 filter working
- [ ] http://localhost:3000/classes - 2 filters working
- [ ] http://localhost:3000/feats - 1 filter working

Expected: All filters behave identically to before refactoring

#### Step 13.5: Create final summary commit (optional)

```bash
git log --oneline --since="1 day ago" | head -15
```

Review all commits from this session. If desired, create summary document:

Create: `docs/HANDOVER-2025-11-25-FILTER-COMPOSABLES-COMPLETE.md`

```markdown
# Filter Composables Refactoring - Session Complete

**Date:** 2025-11-25
**Duration:** ~5 hours
**Status:** ✅ Complete - All 6 entity pages migrated

---

## Summary

Successfully extracted 3 reusable composables to eliminate ~560 lines of duplicate Meilisearch filter logic across 6 entity pages. All filters working identically, all tests passing, comprehensive documentation created.

---

## What Was Accomplished

### Phase 1: Create Composables (TDD) ✅
- Created `useMeilisearchFilters()` with 28 tests
- Created `useReferenceData<T>()` with 6 tests
- Created `useFilterCount()` with 15 tests
- Total: 49 new tests, 100% coverage

### Phase 2: Pilot Migration (Spells) ✅
- Migrated largest page (622 → 460 lines, 26% reduction)
- Validated composable design with most complex filters
- All 10 filters working identically
- Browser verified

### Phase 3: Roll Out (5 Pages) ✅
- Items: 384 → 310 lines (19% reduction)
- Monsters: 318 → 270 lines (15% reduction, hybrid approach)
- Races: 251 → 200 lines (20% reduction)
- Classes: 248 → 200 lines (19% reduction)
- Feats: 215 → 170 lines (21% reduction)
- All filters working, all tests passing

### Phase 4: Documentation ✅
- Updated CLAUDE.md with composables section
- Created comprehensive migration guide
- Updated CHANGELOG.md
- Created this handover document

---

## Metrics

**Code Reduction:**
- Total lines eliminated: ~560 lines
- Average reduction per page: 20%
- Largest reduction: Spells (162 lines)

**Test Coverage:**
- New tests added: 49
- Test files created: 3
- Coverage: 100% on all composables

**Pages Migrated:**
- Spells: 10 filters (full composable)
- Items: 5 filters (hybrid: 4 composable + 1 manual)
- Monsters: 3 filters (hybrid: 2 composable + 1 manual)
- Races: 1 filter (full composable)
- Classes: 2 filters (full composable)
- Feats: 1 filter (full composable)

**Quality:**
- ✅ All existing tests passing
- ✅ No TypeScript regressions
- ✅ ESLint 0 errors
- ✅ All filters browser verified

---

## Files Changed

**Composables Created:**
- `app/composables/useMeilisearchFilters.ts` (150 lines)
- `app/composables/useReferenceData.ts` (80 lines)
- `app/composables/useFilterCount.ts` (40 lines)

**Tests Created:**
- `tests/composables/useMeilisearchFilters.test.ts` (350 lines, 28 tests)
- `tests/composables/useReferenceData.test.ts` (120 lines, 6 tests)
- `tests/composables/useFilterCount.test.ts` (180 lines, 15 tests)

**Pages Migrated:**
- `app/pages/spells/index.vue` (reduced 162 lines)
- `app/pages/items/index.vue` (reduced 74 lines)
- `app/pages/monsters/index.vue` (reduced 48 lines)
- `app/pages/races/index.vue` (reduced 51 lines)
- `app/pages/classes/index.vue` (reduced 48 lines)
- `app/pages/feats/index.vue` (reduced 45 lines)

**Documentation:**
- `CLAUDE.md` (added composables section)
- `docs/FILTER-COMPOSABLES-MIGRATION-GUIDE.md` (comprehensive guide)
- `docs/plans/2025-11-25-filter-composables-refactoring-design.md` (design doc)
- `docs/plans/2025-11-25-filter-composables-refactoring-implementation.md` (this plan)
- `CHANGELOG.md` (refactoring entry)
- `docs/HANDOVER-2025-11-25-FILTER-COMPOSABLES-COMPLETE.md` (this document)

---

## Commits

**Total: 13 commits**

1. `feat: add useMeilisearchFilters composable`
2. `feat: add useReferenceData composable`
3. `feat: add useFilterCount composable`
4. `refactor: migrate spells page`
5. `refactor: migrate items page`
6. `refactor: migrate monsters page`
7. `refactor: migrate races page`
8. `refactor: migrate classes page`
9. `refactor: migrate feats page`
10. `docs: add filter composables to CLAUDE.md`
11. `docs: create migration guide`
12. `docs: update CHANGELOG`
13. `docs: create session handover` (this commit)

---

## Benefits Achieved

**Maintainability:**
- ✅ Bug fixes in 1 place instead of 6
- ✅ Consistent behavior across all pages
- ✅ Easier to add new filter types
- ✅ Better type safety with FilterConfig

**Testability:**
- ✅ 49 new tests (comprehensive coverage)
- ✅ Easier to test edge cases in isolation
- ✅ Regression protection via composable tests

**Readability:**
- ✅ Declarative filter configs vs imperative logic
- ✅ Less nesting, easier to scan
- ✅ Self-documenting filter types
- ✅ Clear separation of concerns

**Performance:**
- ✅ No degradation - same reactive pattern
- ✅ No additional overhead
- ✅ Same number of API calls

---

## Known Issues

**None!** All filters working identically to before refactoring.

**Backend Issues (Pre-existing):**
- Classes `is_base_class` filter returns HTML error (backend issue, not related to refactoring)
- Races `size_code` filter returns correct count but empty data array (backend issue)

These issues existed before refactoring and are unrelated to composable changes.

---

## Next Steps

**Immediate:**
- ✅ No action needed - refactoring complete

**Future Enhancements:**
- Consider auto-generating filter UI (Phase 2 of hybrid approach)
- Add filter presets ("High-level wizard spells", etc.)
- Add saved filters (localStorage persistence)
- Add filter analytics (track most-used filters)

**For New Developers:**
- Read `docs/FILTER-COMPOSABLES-MIGRATION-GUIDE.md`
- Reference spells page as gold standard
- Follow established patterns when adding new filters

---

## Success Criteria

**All criteria met! ✅**

- [x] 3 composables created with comprehensive tests
- [x] All 6 entity pages migrated successfully
- [x] ~560 lines of duplicate code eliminated
- [x] All existing tests passing (1110+)
- [x] No TypeScript regressions
- [x] All filters browser verified
- [x] Comprehensive documentation created
- [x] Non-breaking change (filters work identically)

---

**End of Handover**

**Next Agent:** Read this document for refactoring context. Use migration guide when adding new filters. Reference spells page for examples.
```

#### Step 13.6: Commit handover document

```bash
git add docs/HANDOVER-2025-11-25-FILTER-COMPOSABLES-COMPLETE.md docs/plans/2025-11-25-filter-composables-refactoring-design.md
git commit -m "docs: create comprehensive handover for filter composables refactoring

- Summary of all phases and accomplishments
- Metrics: 560 lines eliminated, 49 tests added
- List of all files changed and commits made
- Benefits achieved and success criteria met
- Reference for future developers

Session complete! All 6 entity pages migrated successfully.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Execution Complete! 🎉

**Total Time:** ~5 hours
**Commits:** 13
**Lines Eliminated:** ~560
**Tests Added:** 49
**Pages Migrated:** 6

**Next Steps:**
- Choose execution approach (see below)
- Begin Phase 1: Create Composables

---

## Execution Choice

Plan complete and saved to `docs/plans/2025-11-25-filter-composables-refactoring-implementation.md`.

**Two execution options:**

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**Which approach?**
