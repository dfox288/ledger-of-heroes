# Fresh Nuxt 4 Installation Guide
**Date:** 2025-11-20
**Purpose:** Complete guide for fresh Nuxt 4.x installation with proper structure from the start

## 🎯 Critical Lessons Learned

### 1. Nuxt 4 Directory Structure (MANDATORY)
All application code MUST be inside `app/` directory:
```
frontend/
├── app/                    # ⚠️ REQUIRED
│   ├── components/
│   ├── composables/
│   ├── layouts/
│   ├── pages/
│   └── app.vue
├── public/
├── types/
├── tests/
├── nuxt.config.ts
└── package.json
```

### 2. @vueuse/core Version Conflict
**Problem:** NuxtUI 4.2.0 has version conflicts with @vueuse/core
**Solutions:**
- **Option A:** Don't install @vueuse/core directly (let NuxtUI manage it)
- **Option B:** Use package.json `overrides` to force single version
- **Option C:** Replace VueUse utilities with native implementations

### 3. SSR vs SPA Mode
Start with `ssr: false` for debugging, enable later once working.

## 📋 Fresh Installation Steps

### Step 1: Initialize Nuxt 4 Project
```bash
cd /Users/dfox/Development/dnd/frontend

# Remove old installation
rm -rf node_modules package-lock.json .nuxt .output app components composables layouts pages app.vue

# Initialize new Nuxt 4 project
npx nuxi@latest init . --packageManager npm
```

When prompted:
- Install dependencies? **Yes**
- Initialize git repository? **No** (already exists)

### Step 2: Install NuxtUI
```bash
npm install @nuxt/ui
```

### Step 3: Configure nuxt.config.ts
```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  compatibilityDate: '2025-07-15',

  future: {
    compatibilityVersion: 4
  },

  devtools: { enabled: true },

  modules: ['@nuxt/ui'],

  ssr: false, // Start with SSR disabled for debugging

  runtimeConfig: {
    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE || 'http://localhost:8080/api/v1',
      apiDocsUrl: process.env.NUXT_PUBLIC_API_DOCS_URL || 'http://localhost:8080/docs/api',
    }
  }
})
```

### Step 4: Create Environment File
```bash
# .env
NUXT_PUBLIC_API_BASE=http://localhost:8080/api/v1
NUXT_PUBLIC_API_DOCS_URL=http://localhost:8080/docs/api.json
```

### Step 5: Verify Structure
After `npx nuxi init`, you should have:
```
frontend/
├── app/              # ✅ Should exist
│   ├── app.vue       # ✅ Should exist
│   └── pages/        # May need to create
├── nuxt.config.ts
├── package.json
└── tsconfig.json
```

If `app/` doesn't exist, something went wrong!

### Step 6: Create Type Definitions Directory
```bash
mkdir -p types
```

## 🔧 Recreating Search Feature (Clean Version)

### 1. Create Types (`types/search.ts`)
```typescript
export type EntityType = 'spells' | 'items' | 'races' | 'classes' | 'backgrounds' | 'feats'

export interface Spell {
  id: number
  slug: string
  name: string
  level: number
  casting_time: string
  range: string
  components: string
  material_components: string | null
  duration: string
  needs_concentration: boolean
  is_ritual: boolean
  description: string
  higher_levels: string | null
}

export interface Item {
  id: number
  slug: string
  name: string
  rarity: string
  is_magic: boolean
  requires_attunement: boolean
  description: string
}

export interface Race {
  id: number
  slug: string
  name: string
}

export interface CharacterClass {
  id: number
  slug: string
  name: string
}

export interface Background {
  id: number
  slug: string
  name: string
}

export interface Feat {
  id: number
  slug: string
  name: string
}

export interface SearchResult {
  data: {
    spells?: Spell[]
    items?: Item[]
    races?: Race[]
    classes?: CharacterClass[]
    backgrounds?: Background[]
    feats?: Feat[]
  }
}
```

### 2. Create Composable (`app/composables/useSearch.ts`)
```typescript
import { ref } from 'vue'
import type { SearchResult, EntityType } from '~/types/search'

export const useSearch = () => {
  const config = useRuntimeConfig()
  const results = ref<SearchResult | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  const search = async (
    query: string,
    options?: { types?: EntityType[]; limit?: number }
  ) => {
    const trimmedQuery = query.trim()
    if (!trimmedQuery) {
      results.value = null
      return
    }

    loading.value = true
    error.value = null

    try {
      const params: Record<string, any> = { q: trimmedQuery }
      if (options?.types) params.types = options.types
      if (options?.limit) params.limit = options.limit

      const data = await $fetch<SearchResult>(`${config.public.apiBase}/search`, {
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

### 3. Create SearchInput Component (`app/components/SearchInput.vue`)
**⚠️ DO NOT import @vueuse/core - use native debounce instead!**

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'
import type { SearchResult } from '~/types/search'

const router = useRouter()
const { search, results, loading, clearResults } = useSearch()

const query = ref('')
const showDropdown = ref(false)

// Native debounce (no @vueuse/core needed)
let debounceTimeout: ReturnType<typeof setTimeout> | null = null
const debouncedSearch = (searchQuery: string) => {
  if (debounceTimeout) {
    clearTimeout(debounceTimeout)
  }

  debounceTimeout = setTimeout(async () => {
    if (searchQuery.trim().length < 2) {
      clearResults()
      showDropdown.value = false
      return
    }

    await search(searchQuery, { limit: 5 })
    showDropdown.value = true
  }, 300)
}

watch(query, (newQuery) => {
  debouncedSearch(newQuery)
})

const handleSubmit = () => {
  if (query.value.trim()) {
    showDropdown.value = false
    router.push(`/search?q=${encodeURIComponent(query.value.trim())}`)
  }
}

const selectResult = (entityType: string, slug: string) => {
  showDropdown.value = false
  query.value = ''
  router.push(`/${entityType}/${slug}`)
}

const closeDropdown = () => {
  showDropdown.value = false
}

const getTotalResults = (searchResults: SearchResult | null): number => {
  if (!searchResults?.data) return 0
  return Object.values(searchResults.data).reduce((total, items) => {
    return total + (items?.length || 0)
  }, 0)
}

const formatEntityType = (type: string): string => {
  return type.charAt(0).toUpperCase() + type.slice(1)
}
</script>

<template>
  <div class="relative w-full">
    <form @submit.prevent="handleSubmit">
      <UInput
        v-model="query"
        type="search"
        placeholder="Search spells, items, races..."
        icon="i-heroicons-magnifying-glass"
        size="lg"
        :loading="loading"
        autocomplete="off"
        @blur="setTimeout(closeDropdown, 200)"
      />
    </form>

    <!-- Dropdown results -->
    <div
      v-if="showDropdown && results && getTotalResults(results) > 0"
      class="absolute z-50 mt-2 w-full rounded-lg border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-900 shadow-lg max-h-96 overflow-y-auto"
    >
      <div class="p-2">
        <!-- Spells -->
        <div v-if="results.data.spells && results.data.spells.length > 0" class="mb-3">
          <div class="px-3 py-1 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase">
            Spells
          </div>
          <button
            v-for="spell in results.data.spells"
            :key="spell.id"
            type="button"
            class="w-full text-left px-3 py-2 rounded hover:bg-gray-100 dark:hover:bg-gray-800 transition-colors"
            @click="selectResult('spells', spell.slug)"
          >
            <div class="font-medium text-gray-900 dark:text-gray-100">
              {{ spell.name }}
            </div>
            <div class="text-sm text-gray-500 dark:text-gray-400">
              Level {{ spell.level }} • {{ spell.casting_time }}
            </div>
          </button>
        </div>

        <!-- Items -->
        <div v-if="results.data.items && results.data.items.length > 0" class="mb-3">
          <div class="px-3 py-1 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase">
            Items
          </div>
          <button
            v-for="item in results.data.items"
            :key="item.id"
            type="button"
            class="w-full text-left px-3 py-2 rounded hover:bg-gray-100 dark:hover:bg-gray-800 transition-colors"
            @click="selectResult('items', item.slug)"
          >
            <div class="font-medium text-gray-900 dark:text-gray-100">
              {{ item.name }}
            </div>
            <div class="text-sm text-gray-500 dark:text-gray-400">
              {{ item.rarity }} {{ item.is_magic ? '• Magic' : '' }}
            </div>
          </button>
        </div>

        <!-- Races -->
        <div v-if="results.data.races && results.data.races.length > 0" class="mb-3">
          <div class="px-3 py-1 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase">
            Races
          </div>
          <button
            v-for="race in results.data.races"
            :key="race.id"
            type="button"
            class="w-full text-left px-3 py-2 rounded hover:bg-gray-100 dark:hover:bg-gray-800 transition-colors"
            @click="selectResult('races', race.slug)"
          >
            <div class="font-medium text-gray-900 dark:text-gray-100">
              {{ race.name }}
            </div>
          </button>
        </div>

        <!-- View all results footer -->
        <div class="border-t border-gray-200 dark:border-gray-700 pt-2 mt-2">
          <button
            type="button"
            class="w-full text-center px-3 py-2 text-sm text-primary-600 dark:text-primary-400 hover:bg-gray-100 dark:hover:bg-gray-800 rounded transition-colors"
            @click="handleSubmit"
          >
            View all results for "{{ query }}"
          </button>
        </div>
      </div>
    </div>

    <!-- No results message -->
    <div
      v-if="showDropdown && results && getTotalResults(results) === 0"
      class="absolute z-50 mt-2 w-full rounded-lg border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-900 shadow-lg p-4 text-center text-gray-500 dark:text-gray-400"
    >
      No results found for "{{ query }}"
    </div>
  </div>
</template>
```

### 4. Create Homepage (`app/pages/index.vue`)
```vue
<script setup lang="ts">
useHead({
  title: 'D&D 5e Compendium',
  meta: [
    { name: 'description', content: 'Search and browse D&D 5th Edition spells, items, races, classes, backgrounds, and feats' },
  ],
})
</script>

<template>
  <div class="container mx-auto px-4 py-12 max-w-4xl">
    <!-- Hero Section -->
    <div class="text-center mb-12">
      <h1 class="text-5xl font-bold text-gray-900 dark:text-gray-100 mb-4">
        D&D 5e Compendium
      </h1>
      <p class="text-xl text-gray-600 dark:text-gray-400 mb-8">
        Search and browse thousands of spells, items, races, and more from D&D 5th Edition
      </p>

      <!-- Featured Search -->
      <div class="max-w-2xl mx-auto">
        <SearchInput />
      </div>
    </div>

    <!-- Quick Links -->
    <div class="grid grid-cols-2 md:grid-cols-3 gap-4 mt-12">
      <UCard to="/spells" class="hover:shadow-lg transition-shadow">
        <div class="text-center">
          <div class="text-3xl mb-2">✨</div>
          <h3 class="font-semibold text-gray-900 dark:text-gray-100">Spells</h3>
          <p class="text-sm text-gray-600 dark:text-gray-400">Browse magic spells</p>
        </div>
      </UCard>

      <UCard to="/items" class="hover:shadow-lg transition-shadow">
        <div class="text-center">
          <div class="text-3xl mb-2">⚔️</div>
          <h3 class="font-semibold text-gray-900 dark:text-gray-100">Items</h3>
          <p class="text-sm text-gray-600 dark:text-gray-400">Equipment & gear</p>
        </div>
      </UCard>

      <UCard to="/races" class="hover:shadow-lg transition-shadow">
        <div class="text-center">
          <div class="text-3xl mb-2">🧝</div>
          <h3 class="font-semibold text-gray-900 dark:text-gray-100">Races</h3>
          <p class="text-sm text-gray-600 dark:text-gray-400">Character races</p>
        </div>
      </UCard>

      <UCard to="/classes" class="hover:shadow-lg transition-shadow">
        <div class="text-center">
          <div class="text-3xl mb-2">🛡️</div>
          <h3 class="font-semibold text-gray-900 dark:text-gray-100">Classes</h3>
          <p class="text-sm text-gray-600 dark:text-gray-400">Character classes</p>
        </div>
      </UCard>

      <UCard to="/backgrounds" class="hover:shadow-lg transition-shadow">
        <div class="text-center">
          <div class="text-3xl mb-2">📜</div>
          <h3 class="font-semibold text-gray-900 dark:text-gray-100">Backgrounds</h3>
          <p class="text-sm text-gray-600 dark:text-gray-400">Character origins</p>
        </div>
      </UCard>

      <UCard to="/feats" class="hover:shadow-lg transition-shadow">
        <div class="text-center">
          <div class="text-3xl mb-2">💪</div>
          <h3 class="font-semibold text-gray-900 dark:text-gray-100">Feats</h3>
          <p class="text-sm text-gray-600 dark:text-gray-400">Special abilities</p>
        </div>
      </UCard>
    </div>

    <!-- Stats Section -->
    <div class="mt-16 text-center">
      <p class="text-gray-600 dark:text-gray-400">
        Powered by D&D 5e API • Over 3,000 entities indexed
      </p>
    </div>
  </div>
</template>
```

### 5. Create Default Layout (`app/layouts/default.vue`)
```vue
<template>
  <div class="min-h-screen bg-gray-50 dark:bg-gray-950">
    <header class="sticky top-0 z-40 bg-white dark:bg-gray-900 border-b border-gray-200 dark:border-gray-800">
      <div class="container mx-auto px-4 py-4 flex items-center justify-between">
        <NuxtLink to="/" class="text-2xl font-bold text-gray-900 dark:text-gray-100">
          D&D 5e
        </NuxtLink>

        <div class="flex-1 max-w-2xl mx-8">
          <SearchInput />
        </div>

        <UButton
          icon="i-heroicons-sun"
          color="gray"
          variant="ghost"
          @click="$colorMode.preference = $colorMode.value === 'dark' ? 'light' : 'dark'"
        />
      </div>
    </header>

    <main>
      <slot />
    </main>

    <footer class="mt-16 py-8 border-t border-gray-200 dark:border-gray-800">
      <div class="container mx-auto px-4 text-center text-gray-600 dark:text-gray-400">
        <p>D&D 5e Compendium • Open Source</p>
      </div>
    </footer>
  </div>
</template>
```

## 🚀 Starting Fresh Server

```bash
# In frontend directory
docker compose up -d
docker compose exec nuxt npm run dev
```

Access at: `http://localhost:3000`

## ✅ Success Checklist

After fresh install, verify:
- [ ] `app/` directory exists with all files inside it
- [ ] `npm run dev` starts without errors
- [ ] Browser shows homepage (not default Nuxt welcome)
- [ ] No console errors about `toValue` or `@vueuse/core`
- [ ] Search input appears
- [ ] Dark mode toggle works

## 🐛 If Issues Persist After Fresh Install

1. **Check Node version:** `node --version` (should be 18+ or 20+)
2. **Check npm version:** `npm --version` (should be 9+ or 10+)
3. **Verify Docker:** Ensure Docker container is using correct Node version
4. **Check browser console:** Look for specific error messages
5. **Check network tab:** See which files are failing to load

---

Good luck with the fresh installation! This should give you a clean, working Nuxt 4 setup from scratch.
