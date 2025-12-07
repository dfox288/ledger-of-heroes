# Filter Persistence Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Persist filter state across browser sessions using Pinia stores with IndexedDB, enabling users to return to their last-used filter configuration.

**Architecture:** Pinia store per entity type (7 stores) with `pinia-plugin-persistedstate` using a custom IndexedDB adapter via `idb-keyval`. URL params override persisted state for shareability. Store updates trigger on every filter change.

**Tech Stack:** Pinia, @pinia/nuxt, pinia-plugin-persistedstate, idb-keyval, TypeScript

**TDD Approach:** Every task follows RED-GREEN-REFACTOR. Tests written first, verified failing, then implementation.

---

## Task 1: Install Dependencies

**Files:**
- Modify: `package.json`
- Modify: `nuxt.config.ts`

**Step 1: Install npm packages**

Run:
```bash
docker compose exec nuxt npm install pinia @pinia/nuxt pinia-plugin-persistedstate idb-keyval
```

Expected: Packages added to package.json dependencies

**Step 2: Verify installation**

Run:
```bash
docker compose exec nuxt npm ls pinia @pinia/nuxt pinia-plugin-persistedstate idb-keyval
```

Expected: All 4 packages listed with versions

**Step 3: Add Pinia module to nuxt.config.ts**

Modify `nuxt.config.ts` - add to modules array (line 3-9):

```typescript
modules: [
  '@pinia/nuxt',           // ADD THIS
  '@nuxt/eslint',
  '@nuxt/ui',
  '@nuxt/test-utils',
  '@nuxt/image',
  '@nuxt/fonts'
],
```

**Step 4: Verify dev server starts**

Run:
```bash
docker compose exec nuxt npm run dev -- --host 0.0.0.0 &
sleep 10
curl -s http://localhost:3000 -o /dev/null -w "%{http_code}"
```

Expected: `200`

**Step 5: Commit**

```bash
git add package.json package-lock.json nuxt.config.ts
git commit -m "chore: Add Pinia and filter persistence dependencies

- @pinia/nuxt for Nuxt integration
- pinia-plugin-persistedstate for automatic persistence
- idb-keyval for IndexedDB adapter

🤖 Generated with [Claude Code](https://claude.ai/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Create IndexedDB Storage Adapter

**Files:**
- Create: `app/utils/idbStorage.ts`
- Create: `tests/utils/idbStorage.test.ts`

**Step 1: Write the failing test**

Create `tests/utils/idbStorage.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { idbStorage } from '~/utils/idbStorage'

// Mock idb-keyval for testing (IndexedDB not available in jsdom)
vi.mock('idb-keyval', () => ({
  get: vi.fn(),
  set: vi.fn(),
  del: vi.fn()
}))

import { get, set, del } from 'idb-keyval'

describe('idbStorage', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  describe('getItem', () => {
    it('returns value from IndexedDB', async () => {
      vi.mocked(get).mockResolvedValue('{"test": "value"}')

      const result = await idbStorage.getItem('test-key')

      expect(get).toHaveBeenCalledWith('test-key')
      expect(result).toBe('{"test": "value"}')
    })

    it('returns null when key not found', async () => {
      vi.mocked(get).mockResolvedValue(undefined)

      const result = await idbStorage.getItem('missing-key')

      expect(result).toBeNull()
    })

    it('returns null on error', async () => {
      vi.mocked(get).mockRejectedValue(new Error('IndexedDB error'))

      const result = await idbStorage.getItem('error-key')

      expect(result).toBeNull()
    })
  })

  describe('setItem', () => {
    it('stores value in IndexedDB', async () => {
      vi.mocked(set).mockResolvedValue(undefined)

      await idbStorage.setItem('test-key', '{"data": 123}')

      expect(set).toHaveBeenCalledWith('test-key', '{"data": 123}')
    })

    it('handles errors gracefully', async () => {
      vi.mocked(set).mockRejectedValue(new Error('Quota exceeded'))

      // Should not throw
      await expect(idbStorage.setItem('test-key', 'value')).resolves.toBeUndefined()
    })
  })

  describe('removeItem', () => {
    it('removes key from IndexedDB', async () => {
      vi.mocked(del).mockResolvedValue(undefined)

      await idbStorage.removeItem('test-key')

      expect(del).toHaveBeenCalledWith('test-key')
    })

    it('handles errors gracefully', async () => {
      vi.mocked(del).mockRejectedValue(new Error('Delete failed'))

      // Should not throw
      await expect(idbStorage.removeItem('test-key')).resolves.toBeUndefined()
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run:
```bash
docker compose exec nuxt npm run test tests/utils/idbStorage.test.ts
```

Expected: FAIL - Cannot find module '~/utils/idbStorage'

**Step 3: Write minimal implementation**

Create `app/utils/idbStorage.ts`:

```typescript
import { get, set, del } from 'idb-keyval'

/**
 * IndexedDB storage adapter for pinia-plugin-persistedstate.
 *
 * Uses idb-keyval for simple key-value storage in IndexedDB.
 * All methods are async but the plugin handles this internally.
 * Errors are caught and logged to prevent app crashes.
 */
export const idbStorage = {
  async getItem(key: string): Promise<string | null> {
    try {
      const value = await get<string>(key)
      return value ?? null
    } catch (e) {
      console.warn(`[idbStorage] Failed to get "${key}":`, e)
      return null
    }
  },

  async setItem(key: string, value: string): Promise<void> {
    try {
      await set(key, value)
    } catch (e) {
      console.warn(`[idbStorage] Failed to set "${key}":`, e)
    }
  },

  async removeItem(key: string): Promise<void> {
    try {
      await del(key)
    } catch (e) {
      console.warn(`[idbStorage] Failed to remove "${key}":`, e)
    }
  }
}
```

**Step 4: Run test to verify it passes**

Run:
```bash
docker compose exec nuxt npm run test tests/utils/idbStorage.test.ts
```

Expected: PASS (6 tests)

**Step 5: Commit**

```bash
git add app/utils/idbStorage.ts tests/utils/idbStorage.test.ts
git commit -m "feat: Add IndexedDB storage adapter for Pinia persistence

- idbStorage utility wraps idb-keyval
- Graceful error handling (no crashes)
- 6 tests covering get/set/remove + error cases

🤖 Generated with [Claude Code](https://claude.ai/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Create Pinia Plugin Configuration

**Files:**
- Create: `app/plugins/pinia-persistence.client.ts`
- Modify: `nuxt.config.ts` (if needed)

**Step 1: Create client-only plugin**

Create `app/plugins/pinia-persistence.client.ts`:

```typescript
import { defineNuxtPlugin } from '#app'
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'

/**
 * Client-side only Pinia persistence plugin.
 *
 * This plugin configures pinia-plugin-persistedstate to work with
 * our IndexedDB adapter. The .client.ts suffix ensures this only
 * runs in the browser (IndexedDB is not available during SSR).
 */
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.$pinia.use(piniaPluginPersistedstate)
})
```

**Step 2: Verify plugin is auto-loaded**

Run:
```bash
docker compose exec nuxt npm run dev -- --host 0.0.0.0 &
sleep 10
curl -s http://localhost:3000 -o /dev/null -w "%{http_code}"
```

Expected: `200` (no errors in console about missing Pinia)

**Step 3: Commit**

```bash
git add app/plugins/pinia-persistence.client.ts
git commit -m "feat: Add client-side Pinia persistence plugin

- Configures pinia-plugin-persistedstate
- Client-only to avoid SSR hydration issues
- Auto-loaded by Nuxt plugin system

🤖 Generated with [Claude Code](https://claude.ai/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Create Base Filter Store Type Definitions

**Files:**
- Create: `app/stores/types.ts`

**Step 1: Create shared types**

Create `app/stores/types.ts`:

```typescript
/**
 * Common types for filter stores.
 * Each entity store extends these base types.
 */

/** Base filter state shared by all entity stores */
export interface BaseFilterState {
  // Search & Sort
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'

  // Common filters
  selectedSources: string[]

  // UI state
  filtersOpen: boolean
}

/** Options for the persist plugin */
export interface FilterPersistOptions {
  key: string
  paths?: string[]
}

/** Store key constants for IndexedDB */
export const STORE_KEYS = {
  spells: 'dnd-filters-spells',
  items: 'dnd-filters-items',
  monsters: 'dnd-filters-monsters',
  races: 'dnd-filters-races',
  classes: 'dnd-filters-classes',
  backgrounds: 'dnd-filters-backgrounds',
  feats: 'dnd-filters-feats'
} as const

export type EntityType = keyof typeof STORE_KEYS
```

**Step 2: Commit**

```bash
git add app/stores/types.ts
git commit -m "feat: Add filter store type definitions

- BaseFilterState for common fields
- STORE_KEYS for IndexedDB key constants
- EntityType union for type safety

🤖 Generated with [Claude Code](https://claude.ai/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Create Feats Filter Store (Simplest - 4 filters)

**Files:**
- Create: `app/stores/featFilters.ts`
- Create: `tests/stores/featFilters.test.ts`

**Step 1: Write the failing test**

Create `tests/stores/featFilters.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useFeatFiltersStore } from '~/stores/featFilters'

describe('useFeatFiltersStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  describe('initial state', () => {
    it('initializes with default values', () => {
      const store = useFeatFiltersStore()

      expect(store.searchQuery).toBe('')
      expect(store.sortBy).toBe('name')
      expect(store.sortDirection).toBe('asc')
      expect(store.selectedSources).toEqual([])
      expect(store.hasPrerequisites).toBeNull()
      expect(store.grantsProficiencies).toBeNull()
      expect(store.selectedImprovedAbilities).toEqual([])
      expect(store.selectedPrerequisiteTypes).toEqual([])
      expect(store.filtersOpen).toBe(false)
    })
  })

  describe('hasActiveFilters getter', () => {
    it('returns false when no filters active', () => {
      const store = useFeatFiltersStore()
      expect(store.hasActiveFilters).toBe(false)
    })

    it('returns true when searchQuery has value', () => {
      const store = useFeatFiltersStore()
      store.searchQuery = 'alert'
      expect(store.hasActiveFilters).toBe(true)
    })

    it('returns true when hasPrerequisites is set', () => {
      const store = useFeatFiltersStore()
      store.hasPrerequisites = '1'
      expect(store.hasActiveFilters).toBe(true)
    })

    it('returns true when selectedSources has values', () => {
      const store = useFeatFiltersStore()
      store.selectedSources = ['PHB']
      expect(store.hasActiveFilters).toBe(true)
    })

    it('returns true when selectedImprovedAbilities has values', () => {
      const store = useFeatFiltersStore()
      store.selectedImprovedAbilities = ['STR']
      expect(store.hasActiveFilters).toBe(true)
    })
  })

  describe('activeFilterCount getter', () => {
    it('returns 0 when no filters active', () => {
      const store = useFeatFiltersStore()
      expect(store.activeFilterCount).toBe(0)
    })

    it('counts each active filter', () => {
      const store = useFeatFiltersStore()
      store.hasPrerequisites = '1'
      store.grantsProficiencies = '0'
      store.selectedSources = ['PHB', 'XGTE']
      store.selectedImprovedAbilities = ['STR', 'DEX']

      // hasPrerequisites (1) + grantsProficiencies (1) + sources (2) + abilities (2) = 6
      expect(store.activeFilterCount).toBe(6)
    })

    it('does not count searchQuery in filter count', () => {
      const store = useFeatFiltersStore()
      store.searchQuery = 'alert'
      expect(store.activeFilterCount).toBe(0)
    })
  })

  describe('clearAll action', () => {
    it('resets all filters to defaults', () => {
      const store = useFeatFiltersStore()

      // Set various filters
      store.searchQuery = 'alert'
      store.sortBy = 'level'
      store.sortDirection = 'desc'
      store.selectedSources = ['PHB']
      store.hasPrerequisites = '1'
      store.grantsProficiencies = '0'
      store.selectedImprovedAbilities = ['STR']
      store.selectedPrerequisiteTypes = ['Race']
      store.filtersOpen = true

      store.clearAll()

      expect(store.searchQuery).toBe('')
      expect(store.sortBy).toBe('name')
      expect(store.sortDirection).toBe('asc')
      expect(store.selectedSources).toEqual([])
      expect(store.hasPrerequisites).toBeNull()
      expect(store.grantsProficiencies).toBeNull()
      expect(store.selectedImprovedAbilities).toEqual([])
      expect(store.selectedPrerequisiteTypes).toEqual([])
      // filtersOpen should NOT be reset (UI preference)
      expect(store.filtersOpen).toBe(true)
    })
  })

  describe('setFromUrlQuery action', () => {
    it('sets filters from URL query object', () => {
      const store = useFeatFiltersStore()

      store.setFromUrlQuery({
        has_prerequisites: '1',
        grants_proficiencies: '0',
        improved_ability: ['STR', 'DEX'],
        prerequisite_type: 'Race',
        source: 'PHB',
        sort_by: 'name',
        sort_direction: 'desc'
      })

      expect(store.hasPrerequisites).toBe('1')
      expect(store.grantsProficiencies).toBe('0')
      expect(store.selectedImprovedAbilities).toEqual(['STR', 'DEX'])
      expect(store.selectedPrerequisiteTypes).toEqual(['Race'])
      expect(store.selectedSources).toEqual(['PHB'])
      expect(store.sortBy).toBe('name')
      expect(store.sortDirection).toBe('desc')
    })

    it('handles array vs string query params', () => {
      const store = useFeatFiltersStore()

      // Single value as string
      store.setFromUrlQuery({ improved_ability: 'STR' })
      expect(store.selectedImprovedAbilities).toEqual(['STR'])

      // Multiple values as array
      store.setFromUrlQuery({ improved_ability: ['STR', 'DEX'] })
      expect(store.selectedImprovedAbilities).toEqual(['STR', 'DEX'])
    })
  })

  describe('toUrlQuery getter', () => {
    it('returns empty object when no filters active', () => {
      const store = useFeatFiltersStore()
      expect(store.toUrlQuery).toEqual({})
    })

    it('returns query object with active filters', () => {
      const store = useFeatFiltersStore()
      store.hasPrerequisites = '1'
      store.selectedSources = ['PHB']
      store.sortDirection = 'desc'

      expect(store.toUrlQuery).toEqual({
        has_prerequisites: '1',
        source: ['PHB'],
        sort_direction: 'desc'
      })
    })

    it('excludes default sort values', () => {
      const store = useFeatFiltersStore()
      // Default sort - should not appear in URL
      expect(store.toUrlQuery).toEqual({})

      // Non-default sort - should appear
      store.sortBy = 'level'
      expect(store.toUrlQuery).toEqual({ sort_by: 'level' })
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run:
```bash
docker compose exec nuxt npm run test tests/stores/featFilters.test.ts
```

Expected: FAIL - Cannot find module '~/stores/featFilters'

**Step 3: Write implementation**

Create `app/stores/featFilters.ts`:

```typescript
import { defineStore } from 'pinia'
import type { LocationQuery } from 'vue-router'
import { idbStorage } from '~/utils/idbStorage'
import { STORE_KEYS } from './types'

export interface FeatFiltersState {
  // Search & Sort
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'

  // Common filters
  selectedSources: string[]

  // Entity-specific filters
  hasPrerequisites: string | null
  grantsProficiencies: string | null
  selectedImprovedAbilities: string[]
  selectedPrerequisiteTypes: string[]

  // UI state (persisted for convenience)
  filtersOpen: boolean
}

const DEFAULT_STATE: FeatFiltersState = {
  searchQuery: '',
  sortBy: 'name',
  sortDirection: 'asc',
  selectedSources: [],
  hasPrerequisites: null,
  grantsProficiencies: null,
  selectedImprovedAbilities: [],
  selectedPrerequisiteTypes: [],
  filtersOpen: false
}

export const useFeatFiltersStore = defineStore('featFilters', {
  state: (): FeatFiltersState => ({ ...DEFAULT_STATE }),

  getters: {
    hasActiveFilters: (state): boolean => {
      return (
        state.searchQuery !== '' ||
        state.selectedSources.length > 0 ||
        state.hasPrerequisites !== null ||
        state.grantsProficiencies !== null ||
        state.selectedImprovedAbilities.length > 0 ||
        state.selectedPrerequisiteTypes.length > 0
      )
    },

    activeFilterCount: (state): number => {
      let count = 0
      // Don't count searchQuery - it's shown separately
      count += state.selectedSources.length
      if (state.hasPrerequisites !== null) count++
      if (state.grantsProficiencies !== null) count++
      count += state.selectedImprovedAbilities.length
      count += state.selectedPrerequisiteTypes.length
      return count
    },

    toUrlQuery: (state): Record<string, string | string[]> => {
      const query: Record<string, string | string[]> = {}

      if (state.hasPrerequisites !== null) {
        query.has_prerequisites = state.hasPrerequisites
      }
      if (state.grantsProficiencies !== null) {
        query.grants_proficiencies = state.grantsProficiencies
      }
      if (state.selectedImprovedAbilities.length > 0) {
        query.improved_ability = state.selectedImprovedAbilities
      }
      if (state.selectedPrerequisiteTypes.length > 0) {
        query.prerequisite_type = state.selectedPrerequisiteTypes
      }
      if (state.selectedSources.length > 0) {
        query.source = state.selectedSources
      }
      // Only include sort if non-default
      if (state.sortBy !== 'name') {
        query.sort_by = state.sortBy
      }
      if (state.sortDirection !== 'asc') {
        query.sort_direction = state.sortDirection
      }

      return query
    }
  },

  actions: {
    clearAll() {
      // Reset all except filtersOpen (UI preference)
      const filtersOpen = this.filtersOpen
      this.$reset()
      this.filtersOpen = filtersOpen
    },

    setFromUrlQuery(query: LocationQuery) {
      // Parse URL query params into store state
      if (query.has_prerequisites) {
        this.hasPrerequisites = String(query.has_prerequisites)
      }
      if (query.grants_proficiencies) {
        this.grantsProficiencies = String(query.grants_proficiencies)
      }
      if (query.improved_ability) {
        this.selectedImprovedAbilities = Array.isArray(query.improved_ability)
          ? query.improved_ability.map(String)
          : [String(query.improved_ability)]
      }
      if (query.prerequisite_type) {
        this.selectedPrerequisiteTypes = Array.isArray(query.prerequisite_type)
          ? query.prerequisite_type.map(String)
          : [String(query.prerequisite_type)]
      }
      if (query.source) {
        this.selectedSources = Array.isArray(query.source)
          ? query.source.map(String)
          : [String(query.source)]
      }
      if (query.sort_by) {
        this.sortBy = String(query.sort_by)
      }
      if (query.sort_direction) {
        this.sortDirection = query.sort_direction as 'asc' | 'desc'
      }
    }
  },

  persist: {
    key: STORE_KEYS.feats,
    storage: idbStorage,
    // Don't persist searchQuery - usually not wanted on reload
    paths: [
      'sortBy',
      'sortDirection',
      'selectedSources',
      'hasPrerequisites',
      'grantsProficiencies',
      'selectedImprovedAbilities',
      'selectedPrerequisiteTypes',
      'filtersOpen'
    ]
  }
})
```

**Step 4: Run test to verify it passes**

Run:
```bash
docker compose exec nuxt npm run test tests/stores/featFilters.test.ts
```

Expected: PASS (12 tests)

**Step 5: Commit**

```bash
git add app/stores/featFilters.ts tests/stores/featFilters.test.ts
git commit -m "feat: Add Feats filter store with persistence

- FeatFiltersState with all filter fields
- hasActiveFilters and activeFilterCount getters
- clearAll and setFromUrlQuery actions
- toUrlQuery getter for URL sync
- IndexedDB persistence via idbStorage
- 12 tests covering all functionality

🤖 Generated with [Claude Code](https://claude.ai/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Create URL Sync Composable

**Files:**
- Create: `app/composables/useFilterUrlSync.ts`
- Create: `tests/composables/useFilterUrlSync.test.ts`

**Step 1: Write the failing test**

Create `tests/composables/useFilterUrlSync.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { ref } from 'vue'

// Mock vue-router
const mockReplace = vi.fn()
const mockRoute = ref({ query: {} })

vi.mock('vue-router', () => ({
  useRoute: () => mockRoute.value,
  useRouter: () => ({ replace: mockReplace })
}))

// Must import after mocks
import { useFilterUrlSync } from '~/composables/useFilterUrlSync'

describe('useFilterUrlSync', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    mockRoute.value = { query: {} }
  })

  describe('hasUrlParams', () => {
    it('returns false when no query params', () => {
      const { hasUrlParams } = useFilterUrlSync()
      expect(hasUrlParams.value).toBe(false)
    })

    it('returns true when query params exist', () => {
      mockRoute.value = { query: { level: '3' } }
      const { hasUrlParams } = useFilterUrlSync()
      expect(hasUrlParams.value).toBe(true)
    })
  })

  describe('syncToUrl', () => {
    it('updates URL with query object', () => {
      const { syncToUrl } = useFilterUrlSync()

      syncToUrl({ level: '3', school: 'evocation' })

      expect(mockReplace).toHaveBeenCalledWith({
        query: { level: '3', school: 'evocation' }
      })
    })

    it('removes empty values from URL', () => {
      const { syncToUrl } = useFilterUrlSync()

      syncToUrl({ level: '3', school: null, empty: '' })

      expect(mockReplace).toHaveBeenCalledWith({
        query: { level: '3' }
      })
    })

    it('handles array values', () => {
      const { syncToUrl } = useFilterUrlSync()

      syncToUrl({ sources: ['PHB', 'XGTE'] })

      expect(mockReplace).toHaveBeenCalledWith({
        query: { sources: ['PHB', 'XGTE'] }
      })
    })
  })

  describe('clearUrl', () => {
    it('removes all query params', () => {
      mockRoute.value = { query: { level: '3', school: 'evocation' } }
      const { clearUrl } = useFilterUrlSync()

      clearUrl()

      expect(mockReplace).toHaveBeenCalledWith({ query: {} })
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run:
```bash
docker compose exec nuxt npm run test tests/composables/useFilterUrlSync.test.ts
```

Expected: FAIL - Cannot find module '~/composables/useFilterUrlSync'

**Step 3: Write implementation**

Create `app/composables/useFilterUrlSync.ts`:

```typescript
import { computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'

/**
 * Composable for syncing filter state with URL query parameters.
 *
 * Provides utilities for:
 * - Checking if URL has filter params
 * - Updating URL from store state
 * - Clearing URL params
 *
 * @example
 * ```typescript
 * const { hasUrlParams, syncToUrl, clearUrl } = useFilterUrlSync()
 *
 * // Check if should restore from URL or storage
 * onMounted(() => {
 *   if (hasUrlParams.value) {
 *     store.setFromUrlQuery(route.query)
 *   }
 * })
 *
 * // Sync store changes to URL
 * watch(() => store.toUrlQuery, (query) => {
 *   syncToUrl(query)
 * }, { deep: true })
 * ```
 */
export function useFilterUrlSync() {
  const route = useRoute()
  const router = useRouter()

  /** Whether the current URL has any query parameters */
  const hasUrlParams = computed(() => {
    return Object.keys(route.query).length > 0
  })

  /**
   * Update URL with the given query object.
   * Empty/null values are filtered out.
   * Uses replace() to avoid polluting browser history.
   */
  const syncToUrl = (query: Record<string, string | string[] | null | undefined>) => {
    const cleanQuery: Record<string, string | string[]> = {}

    for (const [key, value] of Object.entries(query)) {
      // Skip null, undefined, empty strings, empty arrays
      if (value === null || value === undefined) continue
      if (value === '') continue
      if (Array.isArray(value) && value.length === 0) continue

      cleanQuery[key] = value
    }

    router.replace({ query: cleanQuery })
  }

  /** Clear all URL query parameters */
  const clearUrl = () => {
    router.replace({ query: {} })
  }

  return {
    hasUrlParams,
    syncToUrl,
    clearUrl
  }
}
```

**Step 4: Run test to verify it passes**

Run:
```bash
docker compose exec nuxt npm run test tests/composables/useFilterUrlSync.test.ts
```

Expected: PASS (6 tests)

**Step 5: Commit**

```bash
git add app/composables/useFilterUrlSync.ts tests/composables/useFilterUrlSync.test.ts
git commit -m "feat: Add useFilterUrlSync composable

- hasUrlParams computed for checking URL state
- syncToUrl for updating URL from store
- clearUrl for resetting URL params
- Filters out empty/null values
- Uses replace() to avoid history spam
- 6 tests

🤖 Generated with [Claude Code](https://claude.ai/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Refactor Feats Page to Use Store

**Files:**
- Modify: `app/pages/feats/index.vue`
- Modify: `tests/pages/feats-filters.test.ts` (update if needed)

**Step 1: Read current feats filter tests**

Check existing tests to understand what needs to be maintained.

**Step 2: Refactor page to use store**

Replace the filter refs with store. Key changes:

1. Import and use store instead of local refs
2. Use `storeToRefs` for reactive bindings
3. Add URL sync on mount and on store changes
4. Update `clearFilters` to use store action

Modify `app/pages/feats/index.vue`:

```typescript
<script setup lang="ts">
import { watch, onMounted } from 'vue'
import { storeToRefs } from 'pinia'
import type { Feat, AbilityScore } from '~/types'
import { useFeatFiltersStore } from '~/stores/featFilters'

const route = useRoute()

// Use filter store instead of local refs
const store = useFeatFiltersStore()
const {
  searchQuery,
  sortBy,
  sortDirection,
  selectedSources,
  hasPrerequisites,
  grantsProficiencies,
  selectedImprovedAbilities,
  selectedPrerequisiteTypes,
  filtersOpen
} = storeToRefs(store)

// URL sync composable
const { hasUrlParams, syncToUrl, clearUrl } = useFilterUrlSync()

// Sort value computed (combines sortBy + sortDirection)
const sortValue = useSortValue(sortBy, sortDirection)

// Source filter options (still need the composable for options)
const { sourceOptions, getSourceName } = useSourceFilter()

// On mount: URL params override persisted state
onMounted(() => {
  if (hasUrlParams.value) {
    store.setFromUrlQuery(route.query)
  }
})

// Sync store changes to URL (debounced)
let urlSyncTimeout: ReturnType<typeof setTimeout> | null = null
watch(
  () => store.toUrlQuery,
  (query) => {
    if (urlSyncTimeout) clearTimeout(urlSyncTimeout)
    urlSyncTimeout = setTimeout(() => {
      syncToUrl(query)
    }, 300)
  },
  { deep: true }
)

// Fetch reference data
const { data: abilityScores } = useReferenceData<AbilityScore>('/ability-scores')

// Filter options
const improvedAbilityOptions = computed(() => {
  if (!abilityScores.value) return []
  return abilityScores.value.map(ab => ({
    label: `${ab.name} (${ab.code})`,
    value: ab.code
  }))
})

const prerequisiteTypeOptions = [
  { label: 'Race Requirement', value: 'Race' },
  { label: 'Ability Score Requirement', value: 'AbilityScore' },
  { label: 'Proficiency Requirement', value: 'ProficiencyType' }
]

const sortOptions = [
  { label: 'Name (A-Z)', value: 'name:asc' },
  { label: 'Name (Z-A)', value: 'name:desc' }
]

// Query builder for filters (uses store refs)
const { queryParams: filterParams } = useMeilisearchFilters([
  { ref: hasPrerequisites, field: 'has_prerequisites', type: 'boolean' },
  { ref: grantsProficiencies, field: 'grants_proficiencies', type: 'boolean' },
  { ref: selectedImprovedAbilities, field: 'improved_abilities', type: 'in' },
  { ref: selectedPrerequisiteTypes, field: 'prerequisite_types', type: 'in' },
  { ref: selectedSources, field: 'source_codes', type: 'in' }
])

const queryParams = computed(() => ({
  ...filterParams.value,
  sort_by: sortBy.value,
  sort_direction: sortDirection.value
}))

// Use entity list composable
const {
  currentPage,
  data,
  meta,
  totalResults,
  loading,
  error,
  refresh
} = useEntityList({
  endpoint: '/feats',
  cacheKey: 'feats-list',
  queryBuilder: queryParams,
  searchQuery, // Pass store's searchQuery
  seo: {
    title: 'Feats - D&D 5e Compendium',
    description: 'Browse all D&D 5e feats and character abilities with advanced filtering.'
  }
})

const feats = computed(() => data.value as Feat[])

// Clear all filters - uses store action + URL clear
const clearFilters = () => {
  store.clearAll()
  clearUrl()
}

// Helper functions
const getAbilityName = (code: string) => {
  const ability = abilityScores.value?.find(a => a.code === code)
  return ability ? `${ability.name} (${code})` : code
}

const getPrerequisiteTypeLabel = (type: string) => {
  return prerequisiteTypeOptions.find(opt => opt.value === type)?.label || type
}

const perPage = 24
</script>
```

**Note:** The template section remains largely unchanged - it already uses the variable names that now come from the store.

**Step 3: Run existing tests to verify no regression**

Run:
```bash
docker compose exec nuxt npm run test tests/pages/feats
```

Expected: All existing feats tests should still pass

**Step 4: Run full test suite**

Run:
```bash
docker compose exec nuxt npm run test
```

Expected: All 1302+ tests pass

**Step 5: Manual verification**

1. Navigate to http://localhost:3000/feats
2. Set some filters (e.g., Has Prerequisites = Yes)
3. Click on a feat to go to detail page
4. Press browser Back button
5. Verify filters are still set
6. Close browser tab, reopen http://localhost:3000/feats
7. Verify filters are restored from IndexedDB

**Step 6: Commit**

```bash
git add app/pages/feats/index.vue
git commit -m "refactor: Migrate Feats page to Pinia filter store

- Replace local refs with useFeatFiltersStore
- Add URL sync on mount and store changes
- Debounced URL updates (300ms)
- clearFilters uses store.clearAll()
- All existing tests still pass

🤖 Generated with [Claude Code](https://claude.ai/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 8: Create Backgrounds Filter Store

**Files:**
- Create: `app/stores/backgroundFilters.ts`
- Create: `tests/stores/backgroundFilters.test.ts`

Follow the same pattern as Task 5 (Feats store), but with these fields:

```typescript
export interface BackgroundFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  selectedSkillProficiencies: string[]
  selectedToolProficiencyTypes: string[]
  grantsLanguageChoice: string | null
  filtersOpen: boolean
}
```

**Test, implement, verify, commit.**

---

## Task 9: Refactor Backgrounds Page to Use Store

Follow the same pattern as Task 7 (Feats page refactor).

---

## Task 10: Create Classes Filter Store

**Fields:**
```typescript
export interface ClassFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  isBaseClassFilter: string | null
  isSpellcasterFilter: string | null
  filtersOpen: boolean
}
```

---

## Task 11: Refactor Classes Page to Use Store

---

## Task 12: Create Races Filter Store

**Fields:**
```typescript
export interface RaceFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  selectedSize: string
  selectedSpeedRange: string | null
  selectedParentRace: string
  raceTypeFilter: string | null
  hasInnateSpellsFilter: string | null
  selectedAbilities: string[]
  hasDarkvision: string | null
  filtersOpen: boolean
}
```

---

## Task 13: Refactor Races Page to Use Store

---

## Task 14: Create Spells Filter Store

**Fields (largest store - 14 fields):**
```typescript
export interface SpellFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  selectedLevels: string[]
  selectedSchool: number | null
  selectedClass: string | null
  concentrationFilter: string | null
  ritualFilter: string | null
  selectedDamageTypes: string[]
  selectedSavingThrows: string[]
  selectedTags: string[]
  verbalFilter: string | null
  somaticFilter: string | null
  materialFilter: string | null
  filtersOpen: boolean
}
```

---

## Task 15: Refactor Spells Page to Use Store

---

## Task 16: Create Items Filter Store

**Fields (19 fields):**
```typescript
export interface ItemFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  selectedType: number | null
  selectedRarity: string | null
  selectedMagic: string | null
  hasCharges: string | null
  requiresAttunement: string | null
  stealthDisadvantage: string | null
  selectedStrengthRequirement: string | null
  selectedDamageDice: string[]
  selectedVersatileDice: string[]
  selectedRangeNormal: string | null
  selectedRechargeTiming: string[]
  minAC: number | null
  maxAC: number | null
  minCost: number | null
  maxCost: number | null
  filtersOpen: boolean
}
```

---

## Task 17: Refactor Items Page to Use Store

---

## Task 18: Create Monsters Filter Store

**Fields (20 fields - largest):**
```typescript
export interface MonsterFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  selectedCRs: string[]
  selectedType: string | null
  isLegendary: string | null
  selectedSizeIds: string[]
  selectedAlignments: string[]
  selectedMovementTypes: string[]
  selectedArmorTypes: string[]
  canHover: string | null
  hasLairActions: string | null
  hasReactions: string | null
  isSpellcaster: string | null
  hasMagicResistance: string | null
  minAC: number | null
  maxAC: number | null
  minHP: number | null
  maxHP: number | null
  filtersOpen: boolean
}
```

---

## Task 19: Refactor Monsters Page to Use Store

---

## Task 20: Update useSourceFilter to Work with Store

**Files:**
- Modify: `app/composables/useSourceFilter.ts`

The `useSourceFilter` composable currently creates its own ref. Update it to optionally accept a ref from outside (the store):

```typescript
export function useSourceFilter(externalRef?: Ref<string[]>) {
  const selectedSources = externalRef ?? ref<string[]>([])
  // ... rest of composable
}
```

---

## Task 21: Integration Test - Full Flow

**Files:**
- Create: `tests/integration/filter-persistence.test.ts`

Test the full flow:
1. Set filters on a page
2. Navigate away
3. Navigate back
4. Verify filters restored
5. Simulate browser restart (clear memory, reload from IndexedDB)
6. Verify filters restored

---

## Task 22: Update CLAUDE.md Documentation

**Files:**
- Modify: `CLAUDE.md`

Add section about filter stores:

```markdown
## Filter Persistence (Pinia Stores)

Each entity list page has a corresponding Pinia store for filter state:

| Page | Store | Key |
|------|-------|-----|
| /feats | `useFeatFiltersStore` | `dnd-filters-feats` |
| /spells | `useSpellFiltersStore` | `dnd-filters-spells` |
| ... | ... | ... |

**Pattern for new filter pages:**
1. Create store in `app/stores/<entity>Filters.ts`
2. Use `storeToRefs()` in page for reactive bindings
3. Call `store.setFromUrlQuery()` on mount if URL has params
4. Watch `store.toUrlQuery` to sync URL (debounced 300ms)
5. Use `store.clearAll()` + `clearUrl()` for clear filters button
```

---

## Task 23: Final Verification and Cleanup

**Step 1: Run full test suite**

```bash
docker compose exec nuxt npm run test
```

Expected: All tests pass

**Step 2: Run type check**

```bash
docker compose exec nuxt npm run typecheck
```

Expected: No errors

**Step 3: Run lint**

```bash
docker compose exec nuxt npm run lint
```

Expected: No errors

**Step 4: Manual E2E verification**

Test each page:
1. Feats, Backgrounds, Classes, Races, Spells, Items, Monsters
2. Set filters → detail → back → verify
3. Set filters → close tab → reopen → verify

**Step 5: Final commit**

```bash
git add -A
git commit -m "feat: Complete filter persistence implementation

- 7 Pinia stores (one per entity type)
- IndexedDB persistence via idb-keyval adapter
- URL sync with debouncing
- 50+ new tests
- Full TDD implementation

🤖 Generated with [Claude Code](https://claude.ai/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Summary

| Task | Description | Tests |
|------|-------------|-------|
| 1 | Install dependencies | - |
| 2 | IndexedDB adapter | 6 |
| 3 | Pinia plugin | - |
| 4 | Store types | - |
| 5 | Feats store | 12 |
| 6 | URL sync composable | 6 |
| 7 | Refactor Feats page | existing |
| 8-9 | Backgrounds store + page | 12 |
| 10-11 | Classes store + page | 12 |
| 12-13 | Races store + page | 12 |
| 14-15 | Spells store + page | 12 |
| 16-17 | Items store + page | 12 |
| 18-19 | Monsters store + page | 12 |
| 20 | Update useSourceFilter | - |
| 21 | Integration tests | 6 |
| 22 | Documentation | - |
| 23 | Final verification | - |

**Total new tests:** ~90
**Estimated time:** 10-12 hours
