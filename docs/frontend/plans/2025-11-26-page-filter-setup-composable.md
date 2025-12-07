# Page Filter Setup Composable Design

**Date:** 2025-11-26
**Status:** Approved
**Effort:** ~3 hours

---

## Problem

7 entity list pages have identical URL sync boilerplate (~20 lines each):
- `app/pages/spells/index.vue`
- `app/pages/items/index.vue`
- `app/pages/monsters/index.vue`
- `app/pages/classes/index.vue`
- `app/pages/races/index.vue`
- `app/pages/backgrounds/index.vue`
- `app/pages/feats/index.vue`

The duplicated code handles:
1. Reading URL params on mount → populating store
2. Watching store changes → syncing to URL (debounced)
3. Providing `clearFilters()` function

---

## Solution

Create `usePageFilterSetup(store)` composable with side-effect pattern (like `useHead()`).

### API

```typescript
// In any entity list page:
const store = useSpellFiltersStore()
const { clearFilters } = usePageFilterSetup(store)
```

### Implementation

```typescript
// app/composables/usePageFilterSetup.ts
import type { LocationQuery } from 'vue-router'

interface FilterStore {
  toUrlQuery: Record<string, string | string[]>
  setFromUrlQuery: (query: LocationQuery) => void
  clearAll: () => void
}

export function usePageFilterSetup(store: FilterStore) {
  const route = useRoute()
  const { hasUrlParams, syncToUrl, clearUrl } = useFilterUrlSync()

  // On mount: URL params override persisted state
  onMounted(() => {
    if (hasUrlParams.value) {
      store.setFromUrlQuery(route.query)
    }
  })

  // Sync store changes to URL (debounced 300ms)
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

  // Clear filters action
  const clearFilters = () => {
    store.clearAll()
    clearUrl()
  }

  return { clearFilters }
}
```

---

## Impact

### Before (per page): ~20 lines
```typescript
const { hasUrlParams, syncToUrl, clearUrl } = useFilterUrlSync()

onMounted(() => {
  if (hasUrlParams.value) {
    store.setFromUrlQuery(route.query)
  }
})

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

const clearFilters = () => {
  store.clearAll()
  clearUrl()
}
```

### After (per page): ~2 lines
```typescript
const { clearFilters } = usePageFilterSetup(store)
```

### Summary
- Lines removed from pages: ~140 (20 × 7)
- New composable: ~40 lines
- New tests: ~80 lines
- **Net reduction: ~100 lines**

---

## Implementation Tasks

1. Create `app/composables/usePageFilterSetup.ts`
2. Create `tests/composables/usePageFilterSetup.test.ts`
3. Refactor `app/pages/spells/index.vue`
4. Refactor `app/pages/items/index.vue`
5. Refactor `app/pages/monsters/index.vue`
6. Refactor `app/pages/classes/index.vue`
7. Refactor `app/pages/races/index.vue`
8. Refactor `app/pages/backgrounds/index.vue`
9. Refactor `app/pages/feats/index.vue`
10. Run full test suite
11. Update CHANGELOG.md

---

## Notes

- Monsters page currently uses `useDebounceFn` from @vueuse/core - will be standardized
- TypeScript interface `FilterStore` allows any store matching the contract
- Debounce timing kept at 300ms (existing behavior)
