# Nitro API Proxy Layer and Form Improvements

**Date:** 2025-11-20
**Status:** Approved
**Author:** Claude Code

## Overview

This design addresses four critical issues in the D&D 5e Compendium frontend:

1. **Search broken on homepage** - SSR/CSR API URL mismatch in Docker environment
2. **Homepage search box UX** - Not following NuxtUI form patterns
3. **Entity page search/filter UX** - Inconsistent form structure
4. **Empty filter dropdowns** - USelectMenu not receiving data due to SSR API failures

## Root Cause Analysis

### SSR/CSR API URL Problem

The current implementation uses `config.public.apiBase` for all API calls:

```typescript
// Current (BROKEN)
const data = await $fetch(`${config.public.apiBase}/search`, { query: params })
```

**Problem in Docker:**
- **Client-side (browser):** `http://localhost:8080/api/v1` ✅ Works (host machine)
- **Server-side (SSR):** `http://localhost:8080/api/v1` ❌ Fails (Nuxt container's localhost)

**Why filters are empty:**
- SSR fetches filter data (`/spell-schools`, `/item-types`) but fails silently
- Client-side hydration doesn't retry the fetch
- `USelectMenu` renders with empty `options` array

## Architecture Solution: Nitro API Proxy

### Design Principles

1. **Single URL for all environments:** Frontend uses `/api/*` (Nitro routes)
2. **Server-only backend URL:** Backend URL never exposed to client
3. **Future-proof:** Enables caching, rate limiting, response transformation
4. **Type-safe:** Nitro routes can define typed responses

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser / SSR                         │
└────────────────────────────┬────────────────────────────────┘
                             │
                             │ Calls /api/*
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    Nitro Server Routes                       │
│                   (server/api/*.get.ts)                      │
│                                                              │
│  • /api/search                                              │
│  • /api/spells                                              │
│  • /api/spell-schools                                       │
│  • /api/items                                               │
│  • /api/item-types                                          │
│  • /api/races                                               │
│  • /api/classes                                             │
│  • /api/backgrounds                                         │
│  • /api/feats                                               │
└────────────────────────────┬────────────────────────────────┘
                             │
                             │ Proxies to backend
                             │ using config.apiBaseServer
                             ▼
┌─────────────────────────────────────────────────────────────┐
│              Laravel Backend API (Docker)                    │
│              http://backend:8080/api/v1                     │
└─────────────────────────────────────────────────────────────┘
```

## Implementation Plan

### Phase 1: NuxtUI Documentation Check (5 min)

**Goal:** Verify correct USelectMenu v4 API syntax

**Action:**
- Fetch `https://ui.nuxt.com/llms.txt`
- Check USelectMenu component API
- Verify options format, value-attribute, option-attribute usage

**Success Criteria:**
- Documented correct API usage for USelectMenu v4

---

### Phase 2: Create Nitro Proxy Routes (45 min - TDD)

**Files to create:**
```
server/api/
├── search.get.ts                    # Global search
├── spells/
│   ├── index.get.ts                 # List spells
│   └── [slug].get.ts               # Single spell
├── items/
│   ├── index.get.ts                 # List items
│   └── [slug].get.ts               # Single item
├── races/
│   ├── index.get.ts                 # List races
│   └── [slug].get.ts               # Single race
├── classes/
│   ├── index.get.ts                 # List classes
│   └── [slug].get.ts               # Single class
├── backgrounds/
│   ├── index.get.ts                 # List backgrounds
│   └── [slug].get.ts               # Single background
├── feats/
│   ├── index.get.ts                 # List feats
│   └── [slug].get.ts               # Single feat
├── spell-schools/
│   └── index.get.ts                 # Lookup: spell schools
├── item-types/
│   └── index.get.ts                 # Lookup: item types
├── sources/
│   └── index.get.ts                 # Lookup: sourcebooks
├── damage-types/
│   └── index.get.ts                 # Lookup: damage types
└── conditions/
    └── index.get.ts                 # Lookup: conditions
```

**Test files to create:**
```
tests/server/api/
├── search.test.ts
├── spells/index.test.ts
├── spell-schools/index.test.ts
└── items/index.test.ts
```

#### Example Implementation

**Test First (RED):**
```typescript
// tests/server/api/search.test.ts
import { describe, it, expect, vi } from 'vitest'
import { $fetch } from 'ofetch'
import searchHandler from '~/server/api/search.get'

describe('GET /api/search', () => {
  it('proxies search query to backend and returns results', async () => {
    const mockEvent = {
      node: { req: {}, res: {} },
      context: {}
    }

    // Mock getQuery to return search params
    vi.mock('h3', () => ({
      getQuery: () => ({ q: 'fireball', limit: 5 })
    }))

    // Mock $fetch to return fake backend response
    vi.mock('ofetch', () => ({
      $fetch: vi.fn().mockResolvedValue({
        data: {
          spells: [{ id: 1, name: 'Fireball', slug: 'fireball' }]
        }
      })
    }))

    const result = await searchHandler(mockEvent)

    expect(result.data.spells).toHaveLength(1)
    expect(result.data.spells[0].name).toBe('Fireball')
  })

  it('forwards all query parameters to backend', async () => {
    const mockEvent = { /* ... */ }

    vi.mock('h3', () => ({
      getQuery: () => ({ q: 'dragon', types: ['spell', 'item'], limit: 10 })
    }))

    const fetchSpy = vi.spyOn($fetch, 'create')
    await searchHandler(mockEvent)

    expect(fetchSpy).toHaveBeenCalledWith(
      expect.stringContaining('/search'),
      expect.objectContaining({
        query: { q: 'dragon', types: ['spell', 'item'], limit: 10 }
      })
    )
  })
})
```

**Implementation (GREEN):**
```typescript
// server/api/search.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const query = getQuery(event)

  try {
    // Proxy to Laravel backend (server-side URL)
    const data = await $fetch(`${config.apiBaseServer}/search`, {
      query
    })

    return data
  } catch (error: any) {
    throw createError({
      statusCode: error.response?.status || 500,
      statusMessage: error.message || 'Search failed'
    })
  }
})
```

**Pattern for all routes:**
```typescript
// server/api/spells/index.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const query = getQuery(event)

  const data = await $fetch(`${config.apiBaseServer}/spells`, { query })
  return data
})

// server/api/spells/[slug].get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const slug = getRouterParam(event, 'slug')

  const data = await $fetch(`${config.apiBaseServer}/spells/${slug}`)
  return data
})

// server/api/spell-schools/index.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()

  const data = await $fetch(`${config.apiBaseServer}/spell-schools`)
  return data
})
```

**Success Criteria:**
- ✅ All tests pass (RED → GREEN)
- ✅ Routes proxy correctly to backend
- ✅ Query parameters forwarded
- ✅ Error handling works

---

### Phase 3: Update Runtime Config (5 min)

**File:** `nuxt.config.ts`

**Changes:**
```typescript
export default defineNuxtConfig({
  // ... existing config

  runtimeConfig: {
    // Server-side only (NEVER exposed to client)
    apiBaseServer: process.env.NUXT_API_BASE_SERVER || 'http://localhost:8080/api/v1',

    public: {
      // Remove apiBase - no longer needed
      // apiDocsUrl can stay for documentation links
      apiDocsUrl: process.env.NUXT_PUBLIC_API_DOCS_URL || 'http://localhost:8080/docs/api',
    }
  }
})
```

**Environment Variables:**

**For Docker (production-like):**
```bash
# .env
NUXT_API_BASE_SERVER=http://dnd-backend-php:8080/api/v1
```

**For local development (non-Docker):**
```bash
# .env
NUXT_API_BASE_SERVER=http://localhost:8080/api/v1
```

**Success Criteria:**
- ✅ Backend URL only accessible server-side
- ✅ Environment variable configurable

---

### Phase 4: Update Composables (30 min - TDD)

**Files to update:**
- `composables/useSearch.ts`
- `composables/useApi.ts`

#### Update useApi composable

**Test First (RED):**
```typescript
// tests/composables/useApi.test.ts
import { describe, it, expect } from 'vitest'
import { useApi } from '~/composables/useApi'

describe('useApi', () => {
  it('provides fetch function that uses relative /api paths', () => {
    const { apiFetch } = useApi()

    expect(apiFetch).toBeDefined()
    expect(typeof apiFetch).toBe('function')
  })

  it('does not expose backend URL to client', () => {
    const config = useRuntimeConfig()

    // Should NOT have public.apiBase
    expect(config.public.apiBase).toBeUndefined()
  })
})
```

**Implementation (GREEN):**
```typescript
// composables/useApi.ts
export const useApi = () => {
  // Create fetch instance for Nitro API routes
  const apiFetch = $fetch.create({
    baseURL: '/api',  // Relative to current origin
    onRequest({ options }) {
      // Add any client-side headers if needed (auth, etc.)
    },
    onResponseError({ response }) {
      console.error('API Error:', response.status, response.statusText)
    }
  })

  return { apiFetch }
}
```

#### Update useSearch composable

**Test Updates (RED):**
```typescript
// tests/composables/useSearch.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { useSearch } from '~/composables/useSearch'

describe('useSearch', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('calls /api/search endpoint with query params', async () => {
    const { search } = useSearch()
    const fetchSpy = vi.spyOn($fetch, 'create')

    await search('fireball', { limit: 5 })

    expect(fetchSpy).toHaveBeenCalledWith(
      '/search',  // Relative to /api base
      expect.objectContaining({
        query: { q: 'fireball', limit: 5 }
      })
    )
  })
})
```

**Implementation (GREEN):**
```typescript
// composables/useSearch.ts
import { ref } from 'vue'
import type { SearchResult, SearchOptions } from '~/types/search'

export const useSearch = () => {
  const { apiFetch } = useApi()  // Use new relative API
  const results = ref<SearchResult | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  const search = async (query: string, options?: SearchOptions): Promise<void> => {
    const trimmedQuery = query.trim()

    if (!trimmedQuery) {
      results.value = null
      return
    }

    loading.value = true
    error.value = null

    try {
      const params: Record<string, any> = { q: trimmedQuery }

      if (options?.types) {
        params.types = options.types
      }

      if (options?.limit) {
        params.limit = options.limit
      }

      // Call Nitro route instead of Laravel directly
      const data = await apiFetch<SearchResult>('/search', {
        query: params,
      })

      results.value = data
    } catch (err: any) {
      error.value = err.message || 'Search failed'
      results.value = null
    } finally {
      loading.value = false
    }
  }

  const clearResults = () => {
    results.value = null
    error.value = null
  }

  return {
    results,
    loading,
    error,
    search,
    clearResults,
  }
}
```

**Success Criteria:**
- ✅ All tests pass
- ✅ Composables use `/api/*` routes
- ✅ No direct backend URL usage in client code

---

### Phase 5: Update Entity List Pages (30 min - TDD)

**Files to update:**
- `app/pages/spells/index.vue`
- `app/pages/items/index.vue`
- `app/pages/races/index.vue`
- `app/pages/classes/index.vue`
- `app/pages/backgrounds/index.vue`
- `app/pages/feats/index.vue`

**Pattern for all pages:**

**Before:**
```typescript
const { apiBase } = useApi()

const { data: spellSchools } = await useAsyncData('spell-schools', async () => {
  const response = await $fetch(`${apiBase}/spell-schools`)
  return response.data
})

const { data: spellsResponse } = await useAsyncData('spells-list', async () => {
  const response = await $fetch(`${apiBase}/spells`, {
    query: queryParams.value
  })
  return response
})
```

**After:**
```typescript
const { apiFetch } = useApi()

// Now works in both SSR and CSR!
const { data: spellSchools } = await useAsyncData('spell-schools', async () => {
  const response = await apiFetch('/spell-schools')
  return response.data
})

const { data: spellsResponse } = await useAsyncData('spells-list', async () => {
  const response = await apiFetch('/spells', {
    query: queryParams.value
  })
  return response
})
```

**Test for each page:**
```typescript
// tests/pages/spells/index.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SpellsIndex from '~/pages/spells/index.vue'

describe('Spells Index Page', () => {
  it('fetches spell schools for filters on mount', async () => {
    const wrapper = await mountSuspended(SpellsIndex)

    // Should have called /api/spell-schools
    expect(wrapper.vm.spellSchools).toBeDefined()
  })

  it('renders USelectMenu with spell school options', async () => {
    const wrapper = await mountSuspended(SpellsIndex)

    const selectMenu = wrapper.findComponent({ name: 'USelectMenu' })
    expect(selectMenu.exists()).toBe(true)
    expect(selectMenu.props('options')).toHaveLength(9) // 8 schools + "All Schools"
  })
})
```

**Success Criteria:**
- ✅ All pages updated to use Nitro routes
- ✅ Filter data loads correctly in SSR
- ✅ USelectMenu receives populated options array
- ✅ All tests pass

---

### Phase 6: Add UFormGroup Wrappers (30 min - TDD)

**Goal:** Improve form UX and accessibility

**Files to update:**
- `app/components/SearchInput.vue` (homepage search)
- `app/pages/spells/index.vue` (search input)
- `app/pages/items/index.vue` (search input)
- All other entity list pages

#### Homepage SearchInput Component

**Test First (RED):**
```typescript
// tests/components/SearchInput.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SearchInput from '~/components/SearchInput.vue'

describe('SearchInput', () => {
  it('renders UFormGroup with proper label', async () => {
    const wrapper = await mountSuspended(SearchInput)

    const formGroup = wrapper.findComponent({ name: 'UFormGroup' })
    expect(formGroup.exists()).toBe(true)
    expect(formGroup.props('label')).toBe('Search')
  })

  it('has accessible label for screen readers', async () => {
    const wrapper = await mountSuspended(SearchInput)

    const label = wrapper.find('label')
    expect(label.exists()).toBe(true)
    expect(label.attributes('for')).toBeTruthy()
  })
})
```

**Implementation (GREEN):**
```vue
<!-- app/components/SearchInput.vue -->
<template>
  <div class="relative w-full">
    <UForm @submit="handleSubmit">
      <UFormGroup
        label="Search"
        name="search"
        :ui="{ label: 'sr-only' }"
        help="Search for spells, items, races, classes, and more"
      >
        <UInput
          v-model="query"
          type="search"
          placeholder="Search spells, items, races..."
          icon="i-heroicons-magnifying-glass"
          size="xl"
          :loading="loading"
          autocomplete="off"
          variant="outline"
          :ui="{
            size: { xl: 'text-lg' },
            padding: { xl: 'px-4 py-3' }
          }"
          @blur="closeDropdown"
          @keydown="handleKeydown"
        />
      </UFormGroup>
    </UForm>

    <!-- Dropdown remains the same -->
    <UCard v-if="showDropdown && results && getTotalResults(results) > 0" ...>
      <!-- ... -->
    </UCard>
  </div>
</template>
```

#### Entity Page Search Inputs

**Pattern for all entity pages:**
```vue
<!-- Before -->
<UInput
  v-model="searchQuery"
  icon="i-heroicons-magnifying-glass"
  size="lg"
  placeholder="Search spells..."
>
  <!-- ... -->
</UInput>

<!-- After -->
<UFormGroup
  label="Search"
  name="search"
  :ui="{ label: 'sr-only' }"
>
  <UInput
    v-model="searchQuery"
    icon="i-heroicons-magnifying-glass"
    size="lg"
    placeholder="Search spells..."
  >
    <!-- ... -->
  </UInput>
</UFormGroup>
```

**Success Criteria:**
- ✅ All search inputs wrapped in UFormGroup
- ✅ Accessible labels (sr-only for visual consistency)
- ✅ Tests verify form structure
- ✅ No visual regressions

---

### Phase 7: Fix USelectMenu (if needed) (15 min)

**Goal:** Ensure USelectMenu uses correct NuxtUI v4 API

**Actions:**
1. Check NuxtUI v4 documentation (from Phase 1)
2. Verify current implementation matches documented API
3. Update if API changed from v3 to v4
4. Test that options populate correctly

**Current implementation (check if correct):**
```vue
<USelectMenu
  v-model="selectedLevel"
  :options="levelOptions"
  value-attribute="value"
  option-attribute="label"
  placeholder="Select level"
>
  <template #label>
    <span v-if="selectedLevel === null">All Levels</span>
    <span v-else>Level {{ selectedLevel }}</span>
  </template>
</USelectMenu>
```

**Potential fixes (based on v4 docs):**
- Verify `value-attribute` and `option-attribute` are correct props
- Check if options format needs to be different
- Ensure v-model binding is correct

**Success Criteria:**
- ✅ USelectMenu displays options correctly
- ✅ Selection updates v-model
- ✅ Custom label template works

---

### Phase 8: Manual Verification (15 min)

**Checklist:**
- [ ] Homepage loads without errors
- [ ] Homepage search box works (SSR + CSR)
- [ ] Homepage search dropdown shows results
- [ ] Entity pages load (spells, items, races, classes, backgrounds, feats)
- [ ] Entity page search works
- [ ] Filter dropdowns populate with options
- [ ] Filters actually filter results
- [ ] Pagination works
- [ ] Light mode looks good
- [ ] Dark mode looks good
- [ ] Mobile responsive (375px, 768px, 1024px)
- [ ] Keyboard navigation works
- [ ] Screen reader announcements (test with VoiceOver/NVDA)

**Browser DevTools Checks:**
- [ ] No console errors
- [ ] Network tab: /api/* routes return 200
- [ ] No hydration mismatches
- [ ] Performance acceptable (<500ms page load)

---

## Testing Strategy

### Unit Tests
- ✅ Nitro route handlers (proxy logic)
- ✅ Composables (useSearch, useApi)
- ✅ Utility functions (if any)

### Component Tests
- ✅ SearchInput (form structure, events)
- ✅ Entity list pages (data loading, filtering)
- ✅ USelectMenu (options rendering, selection)

### Integration Tests
- ✅ End-to-end search flow (query → results)
- ✅ Filter flow (select → API call → results update)
- ✅ SSR hydration (no mismatches)

### E2E Tests (Future)
- Search on homepage
- Navigate to spell detail
- Filter spells by level + school
- Verify results match filters

---

## Rollout Plan

### Step 1: Development (TDD)
Follow phases 1-7 above with test-first approach

### Step 2: Local Testing
Manual verification checklist (Phase 8)

### Step 3: Docker Testing
```bash
docker compose down
docker compose up -d
# Test SSR works correctly with backend container name
```

### Step 4: Commit & Document
```bash
git add .
git commit -m "feat: Add Nitro API proxy layer and improve form UX

- Create Nitro server routes to proxy Laravel backend
- Fix SSR/CSR API URL mismatch in Docker
- Add UFormGroup wrappers for better form accessibility
- Fix empty filter dropdowns by ensuring SSR data loads
- Update composables to use relative /api/* routes
- Add comprehensive tests for routes and components

Resolves homepage search, filter population, and form UX issues.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Success Metrics

### Functional
- ✅ Homepage search returns results (SSR + CSR)
- ✅ Filter dropdowns populate on all entity pages
- ✅ Filters actually filter results
- ✅ No SSR hydration errors

### Performance
- ✅ Search responds in <500ms
- ✅ Filter data loads in <200ms
- ✅ No unnecessary API calls

### Code Quality
- ✅ 100% of new code has tests (TDD)
- ✅ All tests pass
- ✅ TypeScript compiles with no errors
- ✅ ESLint passes

### UX
- ✅ Forms follow NuxtUI patterns
- ✅ Accessible (keyboard nav, screen readers)
- ✅ Responsive on mobile
- ✅ Dark mode works correctly

---

## Future Enhancements

### Phase 2 (Optional)
1. **Caching in Nitro layer** - Add Redis/memory cache for lookup data
2. **Rate limiting** - Protect backend from abuse
3. **Response transformation** - Normalize data shapes
4. **Error monitoring** - Add Sentry/logging to Nitro routes
5. **Request batching** - Combine multiple lookups into single request

### Phase 3 (Advanced)
1. **GraphQL layer** - Replace REST with GraphQL in Nitro
2. **Realtime updates** - WebSocket support for live search
3. **Offline support** - Service worker + IndexedDB caching

---

## Risk Mitigation

### Risk: Backend container name unknown
**Mitigation:** Check Docker Compose service name, update env var

### Risk: NuxtUI v4 API changed
**Mitigation:** Fetch docs first (Phase 1), verify before implementing

### Risk: Tests slow down development
**Mitigation:** TDD actually speeds up development by catching issues early

### Risk: SSR performance degradation
**Mitigation:** Add caching in Nitro layer (future enhancement)

---

## Conclusion

This design solves all four reported issues by:
1. **Nitro API proxy** - Eliminates SSR/CSR URL confusion
2. **UFormGroup wrappers** - Improves form UX and accessibility
3. **Fixed data fetching** - Ensures filters populate correctly
4. **Comprehensive tests** - Prevents regressions

The architecture is future-proof, enabling caching, rate limiting, and transformation layers as the application scales.
