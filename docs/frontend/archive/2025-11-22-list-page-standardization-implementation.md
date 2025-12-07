# List Page Standardization Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Standardize all 17 entity list pages to use consistent UI patterns and the `useEntityList` composable.

**Architecture:** Enhance `useEntityList` composable with `noPagination` flag, then incrementally refactor pages from simplest to most complex. Main entity pages get filter chips; reference pages migrate to composable.

**Tech Stack:** Nuxt 4, Vue 3 Composition API, TypeScript, Vitest, NuxtUI 4

---

## Phase A: Composable Enhancement

### Task 1: Write tests for noPagination flag

**Files:**
- Modify: `tests/composables/useEntityList.test.ts`

**Step 1: Add test for noPagination sets per_page to 9999**

```typescript
describe('useEntityList with noPagination', () => {
  it('sets per_page to 9999 when noPagination is true', async () => {
    const { result } = await mountComposable(() =>
      useEntityList({
        endpoint: '/test',
        cacheKey: 'test',
        queryBuilder: computed(() => ({})),
        noPagination: true,
        seo: { title: 'Test', description: 'Test' }
      })
    )

    // Assert query params include per_page: 9999
    expect(mockApiFetch).toHaveBeenCalledWith(
      '/test',
      expect.objectContaining({
        query: expect.objectContaining({ per_page: 9999 })
      })
    )
  })

  it('still applies search filter with noPagination', async () => {
    const { result } = await mountComposable(() =>
      useEntityList({
        endpoint: '/test',
        cacheKey: 'test',
        queryBuilder: computed(() => ({})),
        noPagination: true,
        seo: { title: 'Test', description: 'Test' }
      })
    )

    result.searchQuery.value = 'test query'
    await nextTick()

    expect(mockApiFetch).toHaveBeenCalledWith(
      '/test',
      expect.objectContaining({
        query: expect.objectContaining({
          q: 'test query',
          per_page: 9999
        })
      })
    )
  })

  it('hasActiveFilters works with noPagination', async () => {
    const { result } = await mountComposable(() =>
      useEntityList({
        endpoint: '/test',
        cacheKey: 'test',
        queryBuilder: computed(() => ({})),
        noPagination: true,
        seo: { title: 'Test', description: 'Test' }
      })
    )

    expect(result.hasActiveFilters.value).toBe(false)

    result.searchQuery.value = 'test'
    await nextTick()

    expect(result.hasActiveFilters.value).toBe(true)
  })
})
```

**Step 2: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test -- useEntityList.test.ts
```

Expected: FAIL - noPagination property doesn't exist

**Step 3: Commit failing tests**

```bash
git add tests/composables/useEntityList.test.ts
git commit -m "test: Add tests for useEntityList noPagination flag

- Test per_page set to 9999
- Test search still works
- Test hasActiveFilters works

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 2: Implement noPagination flag

**Files:**
- Modify: `app/composables/useEntityList.ts`

**Step 1: Add noPagination to interface**

In `app/composables/useEntityList.ts`, modify the interface:

```typescript
export interface UseEntityListConfig {
  endpoint: string
  cacheKey: string
  queryBuilder: ComputedRef<Record<string, unknown>>
  perPage?: number
  seo: {
    title: string
    description: string
  }
  initialRoute?: boolean

  /** Disable pagination for small datasets (default: false) */
  noPagination?: boolean
}
```

**Step 2: Update queryParams computation**

In `app/composables/useEntityList.ts`, modify the `queryParams` computed:

```typescript
const queryParams = computed(() => {
  const params: Record<string, unknown> = {
    per_page: config.noPagination ? 9999 : (config.perPage ?? 24),
    page: config.noPagination ? 1 : currentPage.value
  }

  // Add search if present
  const trimmedQuery = searchQuery.value.trim()
  if (trimmedQuery) {
    params.q = trimmedQuery
  }

  // Merge custom filters from page's queryBuilder
  Object.assign(params, config.queryBuilder.value)

  return params
})
```

**Step 3: Run tests to verify they pass**

```bash
docker compose exec nuxt npm run test -- useEntityList.test.ts
```

Expected: ALL PASS

**Step 4: Run full test suite**

```bash
docker compose exec nuxt npm run test
```

Expected: All existing tests still pass (700+ tests)

**Step 5: Commit implementation**

```bash
git add app/composables/useEntityList.ts
git commit -m "feat: Add noPagination flag to useEntityList composable

- Sets per_page to 9999 when noPagination is true
- Fixes page to 1 for non-paginated lists
- Enables reference pages to use composable
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase B: Main Entity Pages

### Task 3: Standardize backgrounds page

**Files:**
- Modify: `app/pages/backgrounds/index.vue`
- Modify: `tests/pages/backgrounds.test.ts` (if exists)

**Step 1: Add has-active-filters prop to header**

In `app/pages/backgrounds/index.vue`, update the `<UiListPageHeader>`:

```vue
<UiListPageHeader
  title="Backgrounds"
  :total="totalResults"
  description="Browse D&D 5e character backgrounds"
  :loading="loading"
  :has-active-filters="hasActiveFilters"  <!-- ADD THIS -->
/>
```

**Step 2: Verify all standard components present**

Check that the page uses:
- `<UiListSkeletonCards>`
- `<UiListErrorState>`
- `<UiListEmptyState>`
- `<UiListResultsCount>`
- `<UiListPagination>`
- `<UiBackLink />`

All should already be present. No changes needed.

**Step 3: Test in browser**

```bash
# Ensure Docker containers running
docker compose up -d

# Visit page
curl -I http://localhost:3000/backgrounds
```

Expected: HTTP 200

**Step 4: Run tests**

```bash
docker compose exec nuxt npm run test -- backgrounds
```

Expected: PASS

**Step 5: Commit**

```bash
git add app/pages/backgrounds/index.vue
git commit -m "refactor: Add has-active-filters to backgrounds page header

- Standardizes backgrounds page with gold standard pattern
- All tests passing
- Page verified (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 4: Standardize feats page

**Files:**
- Modify: `app/pages/feats/index.vue`

**Step 1: Add has-active-filters prop to header**

In `app/pages/feats/index.vue`, update the `<UiListPageHeader>`:

```vue
<UiListPageHeader
  title="Feats"
  :total="totalResults"
  description="Browse D&D 5e feats"
  :loading="loading"
  :has-active-filters="hasActiveFilters"  <!-- ADD THIS -->
/>
```

**Step 2: Test in browser**

```bash
curl -I http://localhost:3000/feats
```

Expected: HTTP 200

**Step 3: Run tests**

```bash
docker compose exec nuxt npm run test -- feats
```

Expected: PASS

**Step 4: Commit**

```bash
git add app/pages/feats/index.vue
git commit -m "refactor: Add has-active-filters to feats page header

- Standardizes feats page with gold standard pattern
- All tests passing
- Page verified (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 5: Standardize classes page

**Files:**
- Modify: `app/pages/classes/index.vue`

**Step 1: Add has-active-filters prop to header**

In `app/pages/classes/index.vue`, update the `<UiListPageHeader>`:

```vue
<UiListPageHeader
  title="Classes"
  :total="totalResults"
  description="Browse D&D 5e classes and subclasses"
  :loading="loading"
  :has-active-filters="hasActiveFilters"  <!-- ADD THIS -->
/>
```

**Step 2: Remove conditional pagination**

Find the pagination section (around line 103) and remove the `v-if`:

BEFORE:
```vue
<UiListPagination
  v-if="totalResults > perPage"
  v-model="currentPage"
  :total="totalResults"
  :items-per-page="perPage"
/>
```

AFTER:
```vue
<UiListPagination
  v-model="currentPage"
  :total="totalResults"
  :items-per-page="perPage"
/>
```

**Step 3: Test in browser**

```bash
curl -I http://localhost:3000/classes
```

Expected: HTTP 200

**Step 4: Run tests**

```bash
docker compose exec nuxt npm run test -- classes
```

Expected: PASS

**Step 5: Commit**

```bash
git add app/pages/classes/index.vue
git commit -m "refactor: Standardize classes page

- Add has-active-filters to header
- Remove conditional pagination rendering
- All tests passing
- Page verified (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 6: Standardize races page with filter chips

**Files:**
- Modify: `app/pages/races/index.vue`

**Step 1: Add has-active-filters to header**

In `app/pages/races/index.vue`, update the `<UiListPageHeader>`:

```vue
<UiListPageHeader
  title="Races"
  :total="totalResults"
  description="Browse D&D 5e races and subraces"
  :loading="loading"
  :has-active-filters="hasActiveFilters"  <!-- ADD THIS -->
/>
```

**Step 2: Add clear filters button**

After the size filter button group (around line 114), add:

```vue
<!-- Clear filters button -->
<UButton
  v-if="hasActiveFilters"
  color="neutral"
  variant="soft"
  @click="clearFilters"
>
  Clear Filters
</UButton>
```

**Step 3: Add active filter chips section**

After the clear filters button, add:

```vue
<!-- Active Filter Chips -->
<div
  v-if="hasActiveFilters"
  class="flex flex-wrap items-center gap-2 pt-2"
>
  <span class="text-sm font-medium text-gray-600 dark:text-gray-400">Active:</span>
  <UButton
    v-if="selectedSize"
    size="xs"
    color="primary"
    variant="soft"
    @click="selectedSize = ''"
  >
    {{ getSizeName(selectedSize) }} ✕
  </UButton>
  <UButton
    v-if="searchQuery"
    size="xs"
    color="neutral"
    variant="soft"
    @click="searchQuery = ''"
  >
    "{{ searchQuery }}" ✕
  </UButton>
</div>
```

**Step 4: Add getSizeName helper in script**

In the `<script setup>` section, add:

```typescript
// Helper for filter chips
const getSizeName = (code: string) => {
  return sizes.value.find(s => s.code === code)?.name || code
}
```

**Step 5: Remove conditional pagination**

Find pagination (around line 158) and remove `v-if`:

BEFORE:
```vue
<UiListPagination
  v-if="totalResults > perPage"
  v-model="currentPage"
  :total="totalResults"
  :items-per-page="perPage"
/>
```

AFTER:
```vue
<UiListPagination
  v-model="currentPage"
  :total="totalResults"
  :items-per-page="perPage"
/>
```

**Step 6: Test in browser**

```bash
curl -I http://localhost:3000/races
```

Expected: HTTP 200

Test interactively:
1. Select a size filter
2. Verify filter chip appears
3. Click chip X to clear
4. Verify filter cleared

**Step 7: Run tests**

```bash
docker compose exec nuxt npm run test -- races
```

Expected: PASS

**Step 8: Commit**

```bash
git add app/pages/races/index.vue
git commit -m "refactor: Add filter chips and standardize races page

- Add has-active-filters to header
- Add clear filters button
- Add active filter chips section
- Add getSizeName helper
- Remove conditional pagination
- All tests passing
- Page verified (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 7: Complete monsters page overhaul

**Files:**
- Modify: `app/pages/monsters/index.vue`

**Step 1: Backup current implementation**

```bash
cp app/pages/monsters/index.vue app/pages/monsters/index.vue.backup
```

**Step 2: Rewrite template section**

Replace the entire `<template>` section with:

```vue
<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <!-- Header -->
    <UiListPageHeader
      title="Monsters"
      :total="totalResults"
      description="Browse D&D 5e monsters with stats, abilities, and lore"
      :loading="loading"
      :has-active-filters="hasActiveFilters"
    />

    <!-- Filters -->
    <div class="mb-6 space-y-4">
      <!-- Search input -->
      <UInput
        v-model="searchQuery"
        placeholder="Search monsters..."
      >
        <template
          v-if="searchQuery"
          #trailing
        >
          <UButton
            color="neutral"
            variant="link"
            :padded="false"
            @click="searchQuery = ''"
          />
        </template>
      </UInput>

      <!-- Filter dropdowns -->
      <div class="flex flex-wrap gap-2">
        <!-- CR Filter -->
        <USelectMenu
          v-model="selectedCR"
          :items="crOptions"
          value-key="value"
          text-key="label"
          placeholder="Challenge Rating"
          class="w-48"
        />

        <!-- Type Filter -->
        <USelectMenu
          v-model="selectedType"
          :items="typeOptions"
          value-key="value"
          text-key="label"
          placeholder="Type"
          class="w-48"
        />

        <!-- Clear filters button -->
        <UButton
          v-if="hasActiveFilters"
          color="neutral"
          variant="soft"
          @click="clearFilters"
        >
          Clear Filters
        </UButton>
      </div>

      <!-- Active Filter Chips -->
      <div
        v-if="hasActiveFilters"
        class="flex flex-wrap items-center gap-2 pt-2"
      >
        <span class="text-sm font-medium text-gray-600 dark:text-gray-400">Active:</span>
        <UButton
          v-if="selectedCR !== null"
          size="xs"
          color="primary"
          variant="soft"
          @click="selectedCR = null"
        >
          {{ getCRLabel(selectedCR) }} ✕
        </UButton>
        <UButton
          v-if="selectedType !== null"
          size="xs"
          color="info"
          variant="soft"
          @click="selectedType = null"
        >
          {{ getTypeLabel(selectedType) }} ✕
        </UButton>
        <UButton
          v-if="searchQuery"
          size="xs"
          color="neutral"
          variant="soft"
          @click="searchQuery = ''"
        >
          "{{ searchQuery }}" ✕
        </UButton>
      </div>
    </div>

    <!-- Loading State -->
    <UiListSkeletonCards v-if="loading" />

    <!-- Error State -->
    <UiListErrorState
      v-else-if="error"
      :error="error"
      entity-name="Monsters"
      @retry="refresh"
    />

    <!-- Empty State -->
    <UiListEmptyState
      v-else-if="monsters.length === 0"
      entity-name="monsters"
      :has-filters="hasActiveFilters"
      @clear-filters="clearFilters"
    />

    <!-- Results -->
    <div v-else>
      <!-- Results count -->
      <UiListResultsCount
        :from="meta?.from || 0"
        :to="meta?.to || 0"
        :total="totalResults"
        entity-name="monster"
      />

      <!-- Monsters Grid -->
      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 mb-8">
        <MonsterCard
          v-for="monster in monsters"
          :key="monster.id"
          :monster="monster"
        />
      </div>

      <!-- Pagination -->
      <UiListPagination
        v-model="currentPage"
        :total="totalResults"
        :items-per-page="perPage"
      />
    </div>

    <!-- Back to Home -->
    <UiBackLink />
  </div>
</template>
```

**Step 3: Update script section**

Replace script section with:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import type { Monster } from '~/types'

const route = useRoute()

// Custom filter state
const selectedCR = ref(route.query.cr ? String(route.query.cr) : null)
const selectedType = ref(route.query.type ? String(route.query.type) : null)

// CR range options
const crOptions = [
  { label: 'All CRs', value: null },
  { label: 'CR 0-4 (Easy)', value: '0-4' },
  { label: 'CR 5-10 (Medium)', value: '5-10' },
  { label: 'CR 11-16 (Hard)', value: '11-16' },
  { label: 'CR 17+ (Deadly)', value: '17+' }
]

// Type options
const typeOptions = [
  { label: 'All Types', value: null },
  { label: 'Aberration', value: 'aberration' },
  { label: 'Beast', value: 'beast' },
  { label: 'Celestial', value: 'celestial' },
  { label: 'Construct', value: 'construct' },
  { label: 'Dragon', value: 'dragon' },
  { label: 'Elemental', value: 'elemental' },
  { label: 'Fey', value: 'fey' },
  { label: 'Fiend', value: 'fiend' },
  { label: 'Giant', value: 'giant' },
  { label: 'Humanoid', value: 'humanoid' },
  { label: 'Monstrosity', value: 'monstrosity' },
  { label: 'Ooze', value: 'ooze' },
  { label: 'Plant', value: 'plant' },
  { label: 'Undead', value: 'undead' }
]

// Query builder
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  if (selectedCR.value) params.cr = selectedCR.value
  if (selectedType.value) params.type = selectedType.value
  return params
})

// Use entity list composable
const {
  searchQuery,
  currentPage,
  data,
  meta,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters: clearBaseFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/monsters',
  cacheKey: 'monsters-list',
  queryBuilder,
  seo: {
    title: 'Monsters - D&D 5e Compendium',
    description: 'Browse D&D 5e monsters with stats, abilities, and lore. Filter by Challenge Rating and creature type.'
  }
})

const monsters = computed(() => data.value as Monster[])

// Clear all filters
const clearFilters = () => {
  clearBaseFilters()
  selectedCR.value = null
  selectedType.value = null
}

// Helper functions for filter chips
const getCRLabel = (cr: string) => {
  return crOptions.find(o => o.value === cr)?.label || cr
}

const getTypeLabel = (type: string) => {
  return typeOptions.find(o => o.value === type)?.label || type
}

const perPage = 24
</script>
```

**Step 4: Test in browser**

```bash
curl -I http://localhost:3000/monsters
```

Expected: HTTP 200

Test interactively:
1. Apply CR filter
2. Apply type filter
3. Verify filter chips appear
4. Clear filters
5. Search for monster
6. Test pagination

**Step 5: Run tests**

```bash
docker compose exec nuxt npm run test -- monsters
```

Expected: May need test updates, but page should work

**Step 6: Delete backup file**

```bash
rm app/pages/monsters/index.vue.backup
```

**Step 7: Commit**

```bash
git add app/pages/monsters/index.vue
git commit -m "refactor: Complete monsters page overhaul

- Replace all raw components with standard Ui* components
- Add active filter chips
- Standardize USelectMenu props (:items, value-key, text-key)
- Use UiListPagination instead of raw UPagination
- Use UiBackLink instead of custom back button
- Add clear filters functionality
- Fix page header props (:total instead of :count)
- All 10 inconsistencies resolved
- Page verified (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase C: Reference Entity Pages

### Task 8: Migrate sizes page to useEntityList

**Files:**
- Modify: `app/pages/sizes/index.vue`

**Step 1: Replace manual useAsyncData with useEntityList**

In `app/pages/sizes/index.vue`, replace the entire `<script setup>` section:

BEFORE (lines 1-49):
```typescript
<script setup lang="ts">
import { ref, computed } from 'vue'
import type { Size } from '~/types'

const { apiFetch } = useApi()
const searchQuery = ref('')

const queryParams = computed(() => {
  const params: Record<string, string> = {}
  if (searchQuery.value.trim()) {
    params.q = searchQuery.value.trim()
  }
  return params
})

const { data: sizesResponse, pending: loading, error, refresh } = await useAsyncData<{ data: Size[] }>(
  'sizes-list',
  async () => {
    const response = await apiFetch<{ data: Size[] }>('/sizes', {
      query: queryParams.value
    })
    return response
  },
  { watch: [queryParams] }
)

const sizes = computed(() => sizesResponse.value?.data || [])
const totalResults = computed(() => sizes.value.length)

useSeoMeta({
  title: 'Creature Sizes - D&D 5e Compendium',
  description: 'Browse all D&D 5e creature size categories from Tiny to Gargantuan.'
})

useHead({
  title: 'Creature Sizes - D&D 5e Compendium'
})
</script>
```

AFTER:
```typescript
<script setup lang="ts">
import { computed } from 'vue'
import type { Size } from '~/types'

// Use entity list composable with noPagination
const {
  searchQuery,
  data,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/sizes',
  cacheKey: 'sizes-list',
  queryBuilder: computed(() => ({})), // No custom filters
  noPagination: true, // Small dataset, no pagination needed
  seo: {
    title: 'Creature Sizes - D&D 5e Compendium',
    description: 'Browse all D&D 5e creature size categories from Tiny to Gargantuan.'
  }
})

const sizes = computed(() => data.value as Size[])
</script>
```

**Step 2: Update template to use composable values**

In the template, update the empty state (around line 94):

BEFORE:
```vue
<UiListEmptyState
  v-else-if="sizes.length === 0"
  entity-name="sizes"
  :has-filters="!!searchQuery"
  @clear-filters="searchQuery = ''"
/>
```

AFTER:
```vue
<UiListEmptyState
  v-else-if="sizes.length === 0"
  entity-name="sizes"
  :has-filters="hasActiveFilters"
  @clear-filters="clearFilters"
/>
```

**Step 3: Add has-active-filters to header**

```vue
<UiListPageHeader
  title="Creature Sizes"
  :total="totalResults"
  description="Browse D&D 5e creature size categories"
  :loading="loading"
  :has-active-filters="hasActiveFilters"  <!-- ADD THIS -->
/>
```

**Step 4: Test URL sync**

```bash
# Visit page
curl -I http://localhost:3000/sizes
```

Expected: HTTP 200

Test in browser:
1. Type search query
2. Verify URL updates with ?q=query
3. Refresh page
4. Verify search persisted from URL

**Step 5: Run tests**

```bash
docker compose exec nuxt npm run test -- sizes
```

Expected: PASS

**Step 6: Commit**

```bash
git add app/pages/sizes/index.vue
git commit -m "refactor: Migrate sizes page to useEntityList composable

- Replace manual useAsyncData with useEntityList
- Add noPagination: true flag
- Gain automatic URL sync for search
- Add has-active-filters to header
- Reduce code by ~35 lines
- All tests passing
- Page verified (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 9: Migrate languages page

**Files:**
- Modify: `app/pages/languages/index.vue`

**Step 1: Replace script section**

Follow same pattern as sizes page:

```typescript
<script setup lang="ts">
import { computed } from 'vue'
import type { Language } from '~/types'

const {
  searchQuery,
  data,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/languages',
  cacheKey: 'languages-list',
  queryBuilder: computed(() => ({})),
  noPagination: true,
  seo: {
    title: 'Languages - D&D 5e Compendium',
    description: 'Browse all D&D 5e languages including Common, Elvish, Dwarvish, and more.'
  }
})

const languages = computed(() => data.value as Language[])
</script>
```

**Step 2: Update template**

Update header and empty state as in Task 8.

**Step 3: Test and commit**

```bash
curl -I http://localhost:3000/languages
docker compose exec nuxt npm run test -- languages
git add app/pages/languages/index.vue
git commit -m "refactor: Migrate languages page to useEntityList composable

- Replace manual useAsyncData with useEntityList
- Add noPagination: true flag
- Gain automatic URL sync
- Reduce code by ~35 lines
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 10: Migrate skills page

**Files:**
- Modify: `app/pages/skills/index.vue`

**Step 1: Replace script section**

```typescript
<script setup lang="ts">
import { computed } from 'vue'
import type { Skill } from '~/types'

const {
  searchQuery,
  data,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/skills',
  cacheKey: 'skills-list',
  queryBuilder: computed(() => ({})),
  noPagination: true,
  seo: {
    title: 'Skills - D&D 5e Compendium',
    description: 'Browse all D&D 5e skills including Acrobatics, Athletics, Stealth, Perception, and more.'
  }
})

const skills = computed(() => data.value as Skill[])
</script>
```

**Step 2: Update template**

Update header and empty state.

**Step 3: Test and commit**

```bash
curl -I http://localhost:3000/skills
docker compose exec nuxt npm run test -- skills
git add app/pages/skills/index.vue
git commit -m "refactor: Migrate skills page to useEntityList composable

- Replace manual useAsyncData with useEntityList
- Add noPagination: true flag
- Gain automatic URL sync
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 11: Migrate conditions page

**Files:**
- Modify: `app/pages/conditions/index.vue`

**Step 1: Replace script and update template**

Follow same pattern as previous reference pages.

```typescript
<script setup lang="ts">
import { computed } from 'vue'
import type { Condition } from '~/types'

const {
  searchQuery,
  data,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/conditions',
  cacheKey: 'conditions-list',
  queryBuilder: computed(() => ({})),
  noPagination: true,
  seo: {
    title: 'Conditions - D&D 5e Compendium',
    description: 'Browse all D&D 5e conditions and status effects.'
  }
})

const conditions = computed(() => data.value as Condition[])
</script>
```

**Step 2: Test and commit**

```bash
curl -I http://localhost:3000/conditions
docker compose exec nuxt npm run test -- conditions
git add app/pages/conditions/index.vue
git commit -m "refactor: Migrate conditions page to useEntityList composable

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 12: Migrate damage-types page

**Files:**
- Modify: `app/pages/damage-types/index.vue`

**Step 1: Migrate to composable**

```typescript
<script setup lang="ts">
import { computed } from 'vue'
import type { DamageType } from '~/types'

const {
  searchQuery,
  data,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/damage-types',
  cacheKey: 'damage-types-list',
  queryBuilder: computed(() => ({})),
  noPagination: true,
  seo: {
    title: 'Damage Types - D&D 5e Compendium',
    description: 'Browse all D&D 5e damage types.'
  }
})

const damageTypes = computed(() => data.value as DamageType[])
</script>
```

**Step 2: Test and commit**

```bash
curl -I http://localhost:3000/damage-types
docker compose exec nuxt npm run test -- damage-types
git add app/pages/damage-types/index.vue
git commit -m "refactor: Migrate damage-types page to useEntityList composable

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 13: Migrate item-types page

**Files:**
- Modify: `app/pages/item-types/index.vue`

**Step 1: Migrate to composable**

```typescript
<script setup lang="ts">
import { computed } from 'vue'
import type { ItemType } from '~/types'

const {
  searchQuery,
  data,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/item-types',
  cacheKey: 'item-types-list',
  queryBuilder: computed(() => ({})),
  noPagination: true,
  seo: {
    title: 'Item Types - D&D 5e Compendium',
    description: 'Browse all D&D 5e item categories.'
  }
})

const itemTypes = computed(() => data.value as ItemType[])
</script>
```

**Step 2: Test and commit**

```bash
curl -I http://localhost:3000/item-types
docker compose exec nuxt npm run test -- item-types
git add app/pages/item-types/index.vue
git commit -m "refactor: Migrate item-types page to useEntityList composable

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 14: Migrate proficiency-types page

**Files:**
- Modify: `app/pages/proficiency-types/index.vue`

**Step 1: Migrate to composable**

```typescript
<script setup lang="ts">
import { computed } from 'vue'
import type { ProficiencyType } from '~/types'

const {
  searchQuery,
  data,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/proficiency-types',
  cacheKey: 'proficiency-types-list',
  queryBuilder: computed(() => ({})),
  noPagination: true,
  seo: {
    title: 'Proficiency Types - D&D 5e Compendium',
    description: 'Browse all D&D 5e proficiency categories.'
  }
})

const proficiencyTypes = computed(() => data.value as ProficiencyType[])
</script>
```

**Step 2: Test and commit**

```bash
curl -I http://localhost:3000/proficiency-types
docker compose exec nuxt npm run test
git add app/pages/proficiency-types/index.vue
git commit -m "refactor: Migrate proficiency-types page to useEntityList composable

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 15: Migrate sources page

**Files:**
- Modify: `app/pages/sources/index.vue`

**Step 1: Migrate to composable**

```typescript
<script setup lang="ts">
import { computed } from 'vue'
import type { Source } from '~/types'

const {
  searchQuery,
  data,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/sources',
  cacheKey: 'sources-list',
  queryBuilder: computed(() => ({})),
  noPagination: true,
  seo: {
    title: 'Sources - D&D 5e Compendium',
    description: 'Browse all D&D 5e source books and references.'
  }
})

const sources = computed(() => data.value as Source[])
</script>
```

**Step 2: Test and commit**

```bash
curl -I http://localhost:3000/sources
docker compose exec nuxt npm run test
git add app/pages/sources/index.vue
git commit -m "refactor: Migrate sources page to useEntityList composable

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 16: Migrate spell-schools page

**Files:**
- Modify: `app/pages/spell-schools/index.vue`

**Step 1: Migrate to composable**

```typescript
<script setup lang="ts">
import { computed } from 'vue'
import type { SpellSchool } from '~/types'

const {
  searchQuery,
  data,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/spell-schools',
  cacheKey: 'spell-schools-list',
  queryBuilder: computed(() => ({})),
  noPagination: true,
  seo: {
    title: 'Spell Schools - D&D 5e Compendium',
    description: 'Browse all D&D 5e schools of magic.'
  }
})

const spellSchools = computed(() => data.value as SpellSchool[])
</script>
```

**Step 2: Test and commit**

```bash
curl -I http://localhost:3000/spell-schools
docker compose exec nuxt npm run test
git add app/pages/spell-schools/index.vue
git commit -m "refactor: Migrate spell-schools page to useEntityList composable

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 17: Migrate ability-scores page

**Files:**
- Modify: `app/pages/ability-scores/index.vue`

**Step 1: Migrate to composable**

```typescript
<script setup lang="ts">
import { computed } from 'vue'
import type { AbilityScore } from '~/types'

const {
  searchQuery,
  data,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/ability-scores',
  cacheKey: 'ability-scores-list',
  queryBuilder: computed(() => ({})),
  noPagination: true,
  seo: {
    title: 'Ability Scores - D&D 5e Compendium',
    description: 'Browse all D&D 5e ability scores (STR, DEX, CON, INT, WIS, CHA).'
  }
})

const abilityScores = computed(() => data.value as AbilityScore[])
</script>
```

**Step 2: Test and commit**

```bash
curl -I http://localhost:3000/ability-scores
docker compose exec nuxt npm run test
git add app/pages/ability-scores/index.vue
git commit -m "refactor: Migrate ability-scores page to useEntityList composable

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase D: Final Verification

### Task 18: Run full test suite

**Step 1: Run all tests**

```bash
docker compose exec nuxt npm run test
```

Expected: All tests pass (720+ tests)

**Step 2: Run TypeScript check**

```bash
docker compose exec nuxt npm run typecheck
```

Expected: No errors

**Step 3: If any failures, fix them**

Address any test failures or TypeScript errors before proceeding.

**Step 4: Commit any fixes**

```bash
git add .
git commit -m "test: Fix test failures after list page standardization

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 19: Browser verification of all pages

**Step 1: Verify all main entity pages**

```bash
for entity in spells items races classes backgrounds feats monsters; do
  echo "=== $entity ==="
  curl -I "http://localhost:3000/$entity" 2>&1 | grep "HTTP"
done
```

Expected: All HTTP 200

**Step 2: Verify all reference entity pages**

```bash
for entity in sizes languages skills conditions damage-types item-types proficiency-types sources spell-schools ability-scores; do
  echo "=== $entity ==="
  curl -I "http://localhost:3000/$entity" 2>&1 | grep "HTTP"
done
```

Expected: All HTTP 200

**Step 3: Manual spot checks**

Test in browser:
1. Visit /monsters - test all filters and chips
2. Visit /races - test size filter and chips
3. Visit /sizes - test search, verify URL updates
4. Visit /spells - verify nothing broke
5. Check dark mode on 2-3 pages
6. Check mobile view on 2-3 pages

---

### Task 20: Update documentation

**Files:**
- Modify: `CHANGELOG.md`
- Modify: `CLAUDE.md`

**Step 1: Update CHANGELOG.md**

Add to the `[Unreleased]` section:

```markdown
### Changed
- Standardized all 17 entity list pages for consistency (2025-11-22)
- Reference entity pages now use `useEntityList` composable with URL sync (2025-11-22)
- Enhanced `useEntityList` composable with `noPagination` option (2025-11-22)

### Fixed
- Monsters page now uses standard UI components (2025-11-22)
- Active filter chips now display on all filterable pages (2025-11-22)
- Pagination rendering is now consistent across all pages (2025-11-22)

### Added
- URL sync for search queries on reference entity pages (2025-11-22)
```

**Step 2: Update CLAUDE.md**

Add a new section after the "Project Structure" section:

```markdown
## List Page Patterns

**All 17 entity list pages follow standardized patterns using the `useEntityList` composable.**

### Paginated List Pages (Main Entities)

For entities with many records (spells, items, races, classes, backgrounds, feats, monsters):

```typescript
const {
  searchQuery,
  currentPage,
  data,
  meta,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/entities',
  cacheKey: 'entities-list',
  queryBuilder: computed(() => ({
    // Custom filters here
    level: selectedLevel.value
  })),
  seo: {
    title: 'Entities - D&D 5e Compendium',
    description: 'Browse D&D 5e entities'
  }
})
```

### Non-Paginated List Pages (Reference Entities)

For small datasets (sizes, languages, skills, etc.):

```typescript
const {
  searchQuery,
  data,
  totalResults,
  loading,
  error,
  refresh,
  clearFilters,
  hasActiveFilters
} = useEntityList({
  endpoint: '/entities',
  cacheKey: 'entities-list',
  queryBuilder: computed(() => ({})),
  noPagination: true,  // <-- Key difference
  seo: {
    title: 'Entities - D&D 5e Compendium',
    description: 'Browse D&D 5e entities'
  }
})
```

### Required Template Components

All list pages MUST include:
- `<UiListPageHeader>` with `:has-active-filters` prop
- `<UiListSkeletonCards>` for loading state
- `<UiListErrorState>` for error state
- `<UiListEmptyState>` for empty state
- `<UiListResultsCount>` for result count
- `<UiListPagination>` for paginated pages
- `<UiBackLink />` for navigation
- `<JsonDebugPanel>` for development (optional, but keep on reference pages)

### Filter Chips Pattern

Pages with filters should include active filter chips:

```vue
<div v-if="hasActiveFilters" class="flex flex-wrap items-center gap-2 pt-2">
  <span class="text-sm font-medium text-gray-600 dark:text-gray-400">Active:</span>
  <UButton
    v-if="selectedFilter !== null"
    size="xs"
    color="primary"
    variant="soft"
    @click="selectedFilter = null"
  >
    {{ getFilterLabel(selectedFilter) }} ✕
  </UButton>
  <UButton
    v-if="searchQuery"
    size="xs"
    color="neutral"
    variant="soft"
    @click="searchQuery = ''"
  >
    "{{ searchQuery }}" ✕
  </UButton>
</div>
```

**See:** `app/pages/spells/index.vue` or `app/pages/items/index.vue` for complete examples.
```

**Step 3: Commit documentation**

```bash
git add CHANGELOG.md CLAUDE.md
git commit -m "docs: Update documentation for list page standardization

- Document noPagination flag in CLAUDE.md
- Add list page patterns section
- Update CHANGELOG with standardization changes
- Include examples for both paginated and non-paginated patterns

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Completion Checklist

**Before marking complete, verify ALL items:**

### Functional Criteria
- [ ] All 17 entity list pages use consistent UI patterns
- [ ] All reference pages use `useEntityList` composable
- [ ] All main entity pages have active filter chips (if filterable)
- [ ] Monsters page uses standard components (10 issues fixed)
- [ ] All pages use `<UiListPagination>` (where applicable)
- [ ] All pages use same empty/error/loading states
- [ ] All pages have `<UiBackLink />`
- [ ] Reference pages have URL sync for search queries

### Quality Criteria
- [ ] All tests pass (target: 720+ tests)
- [ ] TypeScript compiles with no errors
- [ ] All 17 pages verified in browser (HTTP 200)
- [ ] Light and dark mode work on all pages
- [ ] Mobile responsive on all pages
- [ ] No console errors or warnings

### Documentation Criteria
- [ ] CLAUDE.md updated with patterns
- [ ] CHANGELOG.md updated
- [ ] All commits have descriptive messages

### Code Quality Criteria
- [ ] No duplicate code between pages
- [ ] Consistent component prop usage
- [ ] Consistent filter pattern usage
- [ ] Debug panels preserved on reference pages

---

## Summary

**Total Tasks:** 20
**Estimated Time:** 9-11 hours
**Files Modified:** ~35 files
**Code Reduction:** ~400-500 lines
**Tests Added/Updated:** ~90-100 tests

**Key Wins:**
1. ✅ Single pattern for all list pages
2. ✅ Massive code reduction
3. ✅ Reference pages gain URL sync
4. ✅ Consistent UX across all entities
5. ✅ Future-proof foundation

---

**Plan Status:** ✅ Complete and ready for execution
**Next Step:** Choose execution approach (Subagent-Driven or Parallel Session)
