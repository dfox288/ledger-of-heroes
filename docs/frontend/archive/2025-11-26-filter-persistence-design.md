# Filter Persistence with Pinia + IndexedDB

**Date:** 2025-11-26
**Status:** Design Complete
**Author:** Claude Code (Brainstorming Skill)

## Problem Statement

When users navigate from a list page (e.g., `/spells?level=3&school=evocation`) to a detail page (`/spells/fireball`) and press Back, or close and reopen the browser, their filter selections are lost. This creates friction for users who have carefully configured complex filter combinations.

## Requirements

| Requirement | Decision |
|-------------|----------|
| Tab behavior | Independent per tab (in-memory) |
| Persistence duration | Across browser sessions (IndexedDB) |
| URL sharing | URL params override persisted state |
| Restore behavior | Last-used filters restored when no URL params |

## Solution Architecture

### Technology Stack

```
Pinia (state management)
  └── pinia-plugin-persistedstate (automatic sync)
        └── idb-keyval (IndexedDB adapter)
```

**Packages to install:**
```bash
npm install pinia @pinia/nuxt pinia-plugin-persistedstate idb-keyval
```

## Store Design

### One Store Per Entity Type

7 stores, one for each filterable entity page:

| Store | File | Key |
|-------|------|-----|
| `useSpellFiltersStore` | `app/stores/spellFilters.ts` | `dnd-filters-spells` |
| `useItemFiltersStore` | `app/stores/itemFilters.ts` | `dnd-filters-items` |
| `useMonsterFiltersStore` | `app/stores/monsterFilters.ts` | `dnd-filters-monsters` |
| `useRaceFiltersStore` | `app/stores/raceFilters.ts` | `dnd-filters-races` |
| `useClassFiltersStore` | `app/stores/classFilters.ts` | `dnd-filters-classes` |
| `useBackgroundFiltersStore` | `app/stores/backgroundFilters.ts` | `dnd-filters-backgrounds` |
| `useFeatFiltersStore` | `app/stores/featFilters.ts` | `dnd-filters-feats` |

### Store Template

Each store follows this pattern (example: spells):

```typescript
// app/stores/spellFilters.ts
import { defineStore } from 'pinia'

export interface SpellFiltersState {
  // Search & Sort
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'

  // Entity-specific filters
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

  // Common filters
  selectedSources: string[]
}

export const useSpellFiltersStore = defineStore('spellFilters', {
  state: (): SpellFiltersState => ({
    // Search & Sort
    searchQuery: '',
    sortBy: 'name',
    sortDirection: 'asc',

    // Entity-specific filters (mirror pages/spells/index.vue)
    selectedLevels: [],
    selectedSchool: null,
    selectedClass: null,
    concentrationFilter: null,
    ritualFilter: null,
    selectedDamageTypes: [],
    selectedSavingThrows: [],
    selectedTags: [],
    verbalFilter: null,
    somaticFilter: null,
    materialFilter: null,

    // Common filters
    selectedSources: []
  }),

  getters: {
    hasActiveFilters: (state): boolean => {
      return (
        state.searchQuery !== '' ||
        state.selectedLevels.length > 0 ||
        state.selectedSchool !== null ||
        state.selectedClass !== null ||
        state.concentrationFilter !== null ||
        state.ritualFilter !== null ||
        state.selectedDamageTypes.length > 0 ||
        state.selectedSavingThrows.length > 0 ||
        state.selectedTags.length > 0 ||
        state.verbalFilter !== null ||
        state.somaticFilter !== null ||
        state.materialFilter !== null ||
        state.selectedSources.length > 0
      )
    }
  },

  actions: {
    clearAll() {
      this.$reset()
    }
  },

  persist: {
    key: 'dnd-filters-spells',
    storage: persistedState.localStorage // Will be replaced with IndexedDB adapter
  }
})
```

## IndexedDB Adapter

Custom adapter using `idb-keyval` for the persisted state plugin:

```typescript
// app/utils/idbStorage.ts
import { get, set, del } from 'idb-keyval'
import type { StorageLike } from 'pinia-plugin-persistedstate'

export const idbStorage: StorageLike = {
  getItem: async (key: string): Promise<string | null> => {
    try {
      const value = await get(key)
      return value ?? null
    } catch {
      return null
    }
  },

  setItem: async (key: string, value: string): Promise<void> => {
    try {
      await set(key, value)
    } catch (e) {
      console.warn('Failed to persist to IndexedDB:', e)
    }
  },

  removeItem: async (key: string): Promise<void> => {
    try {
      await del(key)
    } catch (e) {
      console.warn('Failed to remove from IndexedDB:', e)
    }
  }
}
```

## Nuxt Configuration

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: [
    '@pinia/nuxt',
    'pinia-plugin-persistedstate/nuxt'
  ]
})
```

## Page Integration Pattern

### Before (Current Implementation)

```typescript
// pages/spells/index.vue
const route = useRoute()

// Local refs initialized from URL
const selectedLevels = ref<string[]>(
  route.query.level
    ? (Array.isArray(route.query.level) ? route.query.level.map(String) : [String(route.query.level)])
    : []
)
const selectedSchool = ref(route.query.school ? Number(route.query.school) : null)
// ... 10+ more refs

// Clear function resets each ref
const clearFilters = () => {
  selectedLevels.value = []
  selectedSchool.value = null
  // ... reset each ref
}
```

### After (With Pinia Store)

```typescript
// pages/spells/index.vue
const route = useRoute()
const router = useRouter()
const store = useSpellFiltersStore()

// Get reactive refs from store
const {
  searchQuery, sortBy, sortDirection,
  selectedLevels, selectedSchool, selectedClass,
  concentrationFilter, ritualFilter,
  selectedDamageTypes, selectedSavingThrows, selectedTags,
  verbalFilter, somaticFilter, materialFilter,
  selectedSources
} = storeToRefs(store)

// On mount: URL params override persisted state (for shareability)
onMounted(() => {
  if (Object.keys(route.query).length > 0) {
    syncFromUrl()
  }
})

// Sync URL → Store (when URL has params)
const syncFromUrl = () => {
  if (route.query.level) {
    store.selectedLevels = parseQueryArray(route.query.level)
  }
  if (route.query.school) {
    store.selectedSchool = Number(route.query.school)
  }
  // ... other params
}

// Watch store changes → Update URL (optional, for shareable URLs)
watch(
  () => store.$state,
  () => {
    updateUrlFromStore()
  },
  { deep: true }
)

// Clear function uses store action
const clearFilters = () => {
  store.clearAll()
  router.replace({ query: {} }) // Also clear URL
}
```

## Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Actions                              │
├─────────────────────────────────────────────────────────────────┤
│  1. Change filter    2. Clear filters    3. Navigate away/back  │
└──────────┬───────────────────┬────────────────────┬─────────────┘
           │                   │                    │
           ▼                   ▼                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Pinia Store                                 │
│  • Reactive state (in-memory)                                   │
│  • Watched by components                                        │
│  • Triggers URL update                                          │
└──────────┬───────────────────────────────────────────┬──────────┘
           │                                           │
           ▼                                           ▼
┌─────────────────────────┐              ┌─────────────────────────┐
│    URL Query Params     │              │       IndexedDB         │
│  • Shareable links      │              │  • Cross-session        │
│  • Browser history      │              │  • Automatic sync       │
│  • Override on mount    │              │  • 7 keys (per entity)  │
└─────────────────────────┘              └─────────────────────────┘
```

## State Synchronization Rules

| Scenario | Behavior |
|----------|----------|
| Page load with URL params | URL params → Store (override persisted) |
| Page load without URL params | IndexedDB → Store (restore last session) |
| Filter changed | Store → IndexedDB (auto-persist) |
| Filter changed | Store → URL (keep URL in sync) |
| Clear filters clicked | Store.$reset() → IndexedDB (cleared) → URL (cleared) |
| Navigate to detail, then back | Store state preserved (never cleared) |
| Close browser, reopen | IndexedDB → Store (restored) |
| Open new tab | Fresh store, IndexedDB → Store (same filters as other tab initially) |

## URL Sync Strategy

To maintain shareable URLs while using Pinia:

```typescript
// composables/useFilterUrlSync.ts
export function useFilterUrlSync<T extends object>(
  store: Store<string, T>,
  fieldToQueryMap: Record<keyof T, string>
) {
  const route = useRoute()
  const router = useRouter()

  // URL → Store (on mount, if URL has params)
  const syncFromUrl = () => {
    for (const [field, queryKey] of Object.entries(fieldToQueryMap)) {
      const queryValue = route.query[queryKey]
      if (queryValue !== undefined) {
        // Parse and set store value
        (store as any)[field] = parseQueryValue(queryValue)
      }
    }
  }

  // Store → URL (on change)
  const syncToUrl = () => {
    const query: Record<string, any> = {}
    for (const [field, queryKey] of Object.entries(fieldToQueryMap)) {
      const value = (store as any)[field]
      if (hasValue(value)) {
        query[queryKey] = serializeQueryValue(value)
      }
    }
    router.replace({ query })
  }

  return { syncFromUrl, syncToUrl }
}
```

## Migration Strategy

### Phase 1: Add Infrastructure (Non-Breaking)
1. Install packages
2. Configure Nuxt modules
3. Create IndexedDB adapter
4. Create all 7 stores with types

### Phase 2: Migrate Pages One at a Time
1. Start with simplest page (Feats - 1 filter)
2. Replace local refs with store refs
3. Add URL sync logic
4. Test: filter → detail → back flow
5. Test: close browser → reopen flow
6. Repeat for each page

### Phase 3: Cleanup
1. Remove unused local refs
2. Update tests to use stores
3. Document patterns in CLAUDE.md

## Testing Strategy

### Unit Tests (per store)
```typescript
describe('useSpellFiltersStore', () => {
  it('initializes with default values', () => {
    const store = useSpellFiltersStore()
    expect(store.selectedLevels).toEqual([])
    expect(store.selectedSchool).toBeNull()
  })

  it('clears all filters', () => {
    const store = useSpellFiltersStore()
    store.selectedLevels = ['1', '2']
    store.selectedSchool = 5

    store.clearAll()

    expect(store.selectedLevels).toEqual([])
    expect(store.selectedSchool).toBeNull()
  })

  it('computes hasActiveFilters correctly', () => {
    const store = useSpellFiltersStore()
    expect(store.hasActiveFilters).toBe(false)

    store.selectedLevels = ['3']
    expect(store.hasActiveFilters).toBe(true)
  })
})
```

### Integration Tests (per page)
```typescript
describe('Spells page filter persistence', () => {
  it('restores filters from IndexedDB on mount', async () => {
    // Pre-populate IndexedDB
    await set('dnd-filters-spells', JSON.stringify({ selectedLevels: ['3'] }))

    const wrapper = await mountSuspended(SpellsPage)

    // Filters should be restored
    expect(wrapper.find('[data-testid="level-filter"]').text()).toContain('3rd')
  })

  it('persists filter changes to IndexedDB', async () => {
    const wrapper = await mountSuspended(SpellsPage)

    // Change a filter
    await wrapper.find('[data-testid="level-3"]').trigger('click')

    // Wait for persistence
    await new Promise(r => setTimeout(r, 100))

    // Check IndexedDB
    const stored = await get('dnd-filters-spells')
    expect(JSON.parse(stored).selectedLevels).toContain('3')
  })
})
```

## Estimated Effort

| Task | Time | Notes |
|------|------|-------|
| Package installation | 10 min | npm install |
| Nuxt configuration | 15 min | Module setup |
| IndexedDB adapter | 30 min | idb-keyval wrapper |
| Store definitions (7) | 2 hours | One per entity |
| URL sync composable | 45 min | Reusable helper |
| Page migrations (7) | 4 hours | ~35 min each |
| Tests (stores + pages) | 2 hours | Unit + integration |
| **Total** | **~10 hours** | |

## Files to Create/Modify

### New Files
- `app/stores/spellFilters.ts`
- `app/stores/itemFilters.ts`
- `app/stores/monsterFilters.ts`
- `app/stores/raceFilters.ts`
- `app/stores/classFilters.ts`
- `app/stores/backgroundFilters.ts`
- `app/stores/featFilters.ts`
- `app/utils/idbStorage.ts`
- `app/composables/useFilterUrlSync.ts`
- `tests/stores/*.test.ts` (7 files)

### Modified Files
- `nuxt.config.ts` (add modules)
- `package.json` (add dependencies)
- `app/pages/spells/index.vue`
- `app/pages/items/index.vue`
- `app/pages/monsters/index.vue`
- `app/pages/races/index.vue`
- `app/pages/classes/index.vue`
- `app/pages/backgrounds/index.vue`
- `app/pages/feats/index.vue`

## Open Questions

1. **URL sync frequency** - Should we debounce URL updates to avoid history spam?
2. **Pagination persistence** - Should current page be persisted too?
3. **Filter collapse state** - Should `filtersOpen` be persisted?

## Appendix: Filter Fields by Entity

### Spells (14 fields)
searchQuery, sortBy, sortDirection, selectedLevels, selectedSchool, selectedClass, concentrationFilter, ritualFilter, selectedDamageTypes, selectedSavingThrows, selectedTags, verbalFilter, somaticFilter, materialFilter, selectedSources

### Items (19 fields)
searchQuery, sortBy, sortDirection, selectedType, selectedRarity, selectedMagic, hasCharges, requiresAttunement, stealthDisadvantage, selectedStrengthRequirement, selectedDamageDice, selectedVersatileDice, selectedRangeNormal, selectedRechargeTiming, minAC, maxAC, minCost, maxCost, selectedSources

### Monsters (20 fields)
searchQuery, sortBy, sortDirection, selectedCRs, selectedType, isLegendary, selectedSizeIds, selectedAlignments, selectedMovementTypes, selectedArmorTypes, canHover, hasLairActions, hasReactions, isSpellcaster, hasMagicResistance, minAC, maxAC, minHP, maxHP, selectedSources

### Races (11 fields)
searchQuery, sortBy, sortDirection, selectedSize, selectedSpeedRange, selectedParentRace, raceTypeFilter, hasInnateSpellsFilter, selectedAbilities, hasDarkvision, selectedSources

### Classes (6 fields)
searchQuery, sortBy, sortDirection, isBaseClassFilter, isSpellcasterFilter, selectedSources

### Backgrounds (6 fields)
searchQuery, sortBy, sortDirection, selectedSkillProficiencies, selectedToolProficiencyTypes, grantsLanguageChoice, selectedSources

### Feats (4 fields)
searchQuery, sortBy, sortDirection, hasPrerequisitesFilter, selectedSources
