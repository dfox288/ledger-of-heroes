# Reference Pages Batch 2 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add 6 new reference pages (Ability Scores, Spell Schools, Item Types, Proficiency Types, Skills, Conditions) following TDD and the established pattern from existing reference pages.

**Architecture:** Each reference type gets a card component (tested with TDD), list page with search, and API proxy. Components follow the neutral gray design system with appropriate badges and text truncation. Implementation is sequential from simplest to most complex.

**Tech Stack:** Nuxt 4.x, NuxtUI 4.x, TypeScript, Vitest, Docker

---

## Prerequisites

- Backend API running at `localhost:8080`
- Frontend Docker containers running (`docker compose up -d`)
- All dependencies installed (`docker compose exec nuxt npm install`)

---

## Task 1: Ability Scores - Card Component (TDD)

**Files:**
- Create: `app/components/ability-score/AbilityScoreCard.vue`
- Create: `tests/components/ability-score/AbilityScoreCard.test.ts`

### Step 1: Write the failing test

Create test file first:

```typescript
// tests/components/ability-score/AbilityScoreCard.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import AbilityScoreCard from '~/components/ability-score/AbilityScoreCard.vue'

describe('AbilityScoreCard', () => {
  const mockAbilityScore = {
    id: 1,
    code: 'STR',
    name: 'Strength'
  }

  it('displays ability score code as large badge', async () => {
    const wrapper = await mountSuspended(AbilityScoreCard, {
      props: { abilityScore: mockAbilityScore }
    })

    expect(wrapper.text()).toContain('STR')
  })

  it('displays ability score name as title', async () => {
    const wrapper = await mountSuspended(AbilityScoreCard, {
      props: { abilityScore: mockAbilityScore }
    })

    expect(wrapper.text()).toContain('Strength')
  })

  it('displays category badge', async () => {
    const wrapper = await mountSuspended(AbilityScoreCard, {
      props: { abilityScore: mockAbilityScore }
    })

    expect(wrapper.text()).toContain('Ability Score')
  })

  it('uses neutral color theme', async () => {
    const wrapper = await mountSuspended(AbilityScoreCard, {
      props: { abilityScore: mockAbilityScore }
    })

    const html = wrapper.html()
    // UCard with border styling indicates neutral theme
    expect(html).toContain('border-gray')
  })

  it('handles missing optional fields gracefully', async () => {
    const minimalData = {
      id: 2,
      code: 'DEX',
      name: 'Dexterity'
    }

    const wrapper = await mountSuspended(AbilityScoreCard, {
      props: { abilityScore: minimalData }
    })

    expect(wrapper.text()).toContain('DEX')
    expect(wrapper.text()).toContain('Dexterity')
  })
})
```

### Step 2: Run test to verify it fails

Run inside Docker container:

```bash
docker compose exec nuxt npm test -- AbilityScoreCard.test.ts
```

**Expected Output:** Test fails with "Cannot find module '~/components/ability-score/AbilityScoreCard.vue'"

### Step 3: Write minimal implementation

Create the component:

```vue
<!-- app/components/ability-score/AbilityScoreCard.vue -->
<script setup lang="ts">
interface AbilityScore {
  id: number
  code: string
  name: string
}

interface Props {
  abilityScore: AbilityScore
}

defineProps<Props>()
</script>

<template>
  <UCard class="hover:shadow-lg transition-shadow h-full border border-gray-200 dark:border-gray-700">
    <div class="space-y-3">
      <!-- Code Badge -->
      <div class="flex items-center gap-2">
        <UBadge color="neutral" variant="solid" size="lg">
          {{ abilityScore.code }}
        </UBadge>
      </div>

      <!-- Name -->
      <h3 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
        {{ abilityScore.name }}
      </h3>

      <!-- Category Badge -->
      <div class="flex items-center gap-2">
        <UBadge color="neutral" variant="soft" size="xs">
          Ability Score
        </UBadge>
      </div>
    </div>
  </UCard>
</template>
```

### Step 4: Run test to verify it passes

```bash
docker compose exec nuxt npm test -- AbilityScoreCard.test.ts
```

**Expected Output:** All 5 tests pass (GREEN)

### Step 5: Commit

```bash
git add app/components/ability-score/AbilityScoreCard.vue tests/components/ability-score/AbilityScoreCard.test.ts
git commit -m "test: Add AbilityScoreCard component with TDD (5 tests passing)

- Created AbilityScoreCard component following TDD
- Displays code badge, name, and category
- Neutral gray theme for reference material
- All 5 tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Ability Scores - Page and API Proxy

**Files:**
- Create: `app/pages/ability-scores/index.vue`
- Create: `server/api/ability-scores/index.get.ts`

### Step 1: Create API proxy

```typescript
// server/api/ability-scores/index.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const query = getQuery(event)
  const data = await $fetch(`${config.apiBaseServer}/ability-scores`, { query })
  return data
})
```

### Step 2: Create list page

```vue
<!-- app/pages/ability-scores/index.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

// API configuration
const { apiFetch } = useApi()

// Reactive filters
const searchQuery = ref('')

// Computed query params for API
const queryParams = computed(() => {
  const params: Record<string, any> = {}

  if (searchQuery.value.trim()) {
    params.q = searchQuery.value.trim()
  }

  return params
})

// Fetch ability scores with reactive filters (via Nitro proxy)
const { data: abilityScoresResponse, pending: loading, error, refresh } = await useAsyncData(
  'ability-scores-list',
  async () => {
    const response = await apiFetch('/ability-scores', {
      query: queryParams.value
    })
    return response
  },
  {
    watch: [queryParams]
  }
)

// Computed values
const abilityScores = computed(() => abilityScoresResponse.value?.data || [])
const totalResults = computed(() => abilityScores.value.length)

// SEO meta tags
useSeoMeta({
  title: 'Ability Scores - D&D 5e Compendium',
  description: 'Browse all D&D 5e ability scores: Strength, Dexterity, Constitution, Intelligence, Wisdom, and Charisma.',
})

useHead({
  title: 'Ability Scores - D&D 5e Compendium',
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <!-- Header -->
    <UiListPageHeader
      title="Ability Scores"
      :total="totalResults"
      description="Browse D&D 5e ability scores"
      :loading="loading"
    />

    <!-- Search -->
    <div class="mb-6">
      <UInput
        v-model="searchQuery"
        icon="i-heroicons-magnifying-glass"
        size="lg"
        placeholder="Search ability scores..."
        :ui="{ icon: { trailing: { pointer: '' } } }"
      >
        <template v-if="searchQuery" #trailing>
          <UButton
            color="gray"
            variant="link"
            icon="i-heroicons-x-mark-20-solid"
            :padded="false"
            @click="searchQuery = ''"
          />
        </template>
      </UInput>
    </div>

    <!-- Loading State -->
    <UiListSkeletonCards v-if="loading" />

    <!-- Error State -->
    <UiListErrorState
      v-else-if="error"
      :error="error"
      entity-name="Ability Scores"
      @retry="refresh"
    />

    <!-- Empty State -->
    <UiListEmptyState
      v-else-if="abilityScores.length === 0"
      entity-name="ability scores"
      :has-filters="!!searchQuery"
      @clear-filters="searchQuery = ''"
    />

    <!-- Results -->
    <div v-else>
      <!-- Results count -->
      <UiListResultsCount
        :from="1"
        :to="totalResults"
        :total="totalResults"
        entity-name="ability score"
      />

      <!-- Ability Scores Grid -->
      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 mb-8">
        <AbilityScoreCard
          v-for="abilityScore in abilityScores"
          :key="abilityScore.id"
          :ability-score="abilityScore"
        />
      </div>
    </div>

    <!-- Back to Home -->
    <UiBackLink />

    <!-- JSON Debug Panel -->
    <JsonDebugPanel :data="{ abilityScores, total: totalResults }" title="Ability Scores Data" />
  </div>
</template>
```

### Step 3: Test in browser

```bash
# Ensure dev server is running
docker compose exec nuxt npm run dev

# Test the page
curl -I http://localhost:3000/ability-scores
```

**Expected Output:** HTTP 200 OK

### Step 4: Manual verification

Open browser to `http://localhost:3000/ability-scores` and verify:
- ✅ Page loads without errors
- ✅ 6 ability score cards display (STR, DEX, CON, INT, WIS, CHA)
- ✅ Search filters results
- ✅ Light/dark mode works
- ✅ Responsive grid (1/2/3 columns)

### Step 5: Commit

```bash
git add app/pages/ability-scores/ server/api/ability-scores/
git commit -m "feat: Add Ability Scores reference page

- Created /ability-scores page with search
- Added API proxy endpoint
- Verified in browser (HTTP 200)
- Search filtering works correctly

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Spell Schools - Card Component (TDD)

**Files:**
- Create: `app/components/spell-school/SpellSchoolCard.vue`
- Create: `tests/components/spell-school/SpellSchoolCard.test.ts`

### Step 1: Write the failing test

```typescript
// tests/components/spell-school/SpellSchoolCard.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SpellSchoolCard from '~/components/spell-school/SpellSchoolCard.vue'

describe('SpellSchoolCard', () => {
  const mockSpellSchool = {
    id: 1,
    code: 'A',
    name: 'Abjuration',
    description: 'Abjuration spells are protective in nature.'
  }

  it('displays spell school code as large badge', async () => {
    const wrapper = await mountSuspended(SpellSchoolCard, {
      props: { spellSchool: mockSpellSchool }
    })

    expect(wrapper.text()).toContain('A')
  })

  it('displays spell school name as title', async () => {
    const wrapper = await mountSuspended(SpellSchoolCard, {
      props: { spellSchool: mockSpellSchool }
    })

    expect(wrapper.text()).toContain('Abjuration')
  })

  it('displays description when present', async () => {
    const wrapper = await mountSuspended(SpellSchoolCard, {
      props: { spellSchool: mockSpellSchool }
    })

    expect(wrapper.text()).toContain('protective in nature')
  })

  it('handles missing description gracefully', async () => {
    const noDescription = {
      id: 2,
      code: 'E',
      name: 'Evocation',
      description: null
    }

    const wrapper = await mountSuspended(SpellSchoolCard, {
      props: { spellSchool: noDescription }
    })

    expect(wrapper.text()).toContain('Evocation')
    expect(wrapper.text()).not.toContain('null')
  })

  it('truncates long descriptions', async () => {
    const longDescription = {
      id: 3,
      code: 'T',
      name: 'Transmutation',
      description: 'Transmutation spells change the properties of a creature, object, or environment. They can turn a creature into another creature, change the nature of an object, or alter the properties of an environment. These spells are among the most versatile and powerful in existence.'
    }

    const wrapper = await mountSuspended(SpellSchoolCard, {
      props: { spellSchool: longDescription }
    })

    const html = wrapper.html()
    // line-clamp-2 class should be present
    expect(html).toContain('line-clamp-2')
  })

  it('displays category badge', async () => {
    const wrapper = await mountSuspended(SpellSchoolCard, {
      props: { spellSchool: mockSpellSchool }
    })

    expect(wrapper.text()).toContain('Spell School')
  })
})
```

### Step 2: Run test to verify it fails

```bash
docker compose exec nuxt npm test -- SpellSchoolCard.test.ts
```

**Expected Output:** Test fails with module not found error

### Step 3: Write minimal implementation

```vue
<!-- app/components/spell-school/SpellSchoolCard.vue -->
<script setup lang="ts">
interface SpellSchool {
  id: number
  code: string
  name: string
  description?: string | null
}

interface Props {
  spellSchool: SpellSchool
}

defineProps<Props>()
</script>

<template>
  <UCard class="hover:shadow-lg transition-shadow h-full border border-gray-200 dark:border-gray-700">
    <div class="space-y-3">
      <!-- Code Badge -->
      <div class="flex items-center gap-2">
        <UBadge color="neutral" variant="solid" size="lg">
          {{ spellSchool.code }}
        </UBadge>
      </div>

      <!-- Name -->
      <h3 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
        {{ spellSchool.name }}
      </h3>

      <!-- Description (optional, truncated) -->
      <p
        v-if="spellSchool.description"
        class="text-sm text-gray-600 dark:text-gray-400 line-clamp-2"
      >
        {{ spellSchool.description }}
      </p>

      <!-- Category Badge -->
      <div class="flex items-center gap-2">
        <UBadge color="neutral" variant="soft" size="xs">
          Spell School
        </UBadge>
      </div>
    </div>
  </UCard>
</template>
```

### Step 4: Run test to verify it passes

```bash
docker compose exec nuxt npm test -- SpellSchoolCard.test.ts
```

**Expected Output:** All 6 tests pass (GREEN)

### Step 5: Commit

```bash
git add app/components/spell-school/ tests/components/spell-school/
git commit -m "test: Add SpellSchoolCard component with TDD (6 tests passing)

- Created SpellSchoolCard with code, name, optional description
- Truncates long descriptions with line-clamp-2
- Handles missing descriptions gracefully
- All 6 tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Spell Schools - Page and API Proxy

**Files:**
- Create: `app/pages/spell-schools/index.vue`
- Create: `server/api/spell-schools/index.get.ts`

### Step 1: Create API proxy

```typescript
// server/api/spell-schools/index.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const query = getQuery(event)
  const data = await $fetch(`${config.apiBaseServer}/spell-schools`, { query })
  return data
})
```

### Step 2: Create list page

```vue
<!-- app/pages/spell-schools/index.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const { apiFetch } = useApi()
const searchQuery = ref('')

const queryParams = computed(() => {
  const params: Record<string, any> = {}
  if (searchQuery.value.trim()) {
    params.q = searchQuery.value.trim()
  }
  return params
})

const { data: spellSchoolsResponse, pending: loading, error, refresh } = await useAsyncData(
  'spell-schools-list',
  async () => {
    const response = await apiFetch('/spell-schools', {
      query: queryParams.value
    })
    return response
  },
  { watch: [queryParams] }
)

const spellSchools = computed(() => spellSchoolsResponse.value?.data || [])
const totalResults = computed(() => spellSchools.value.length)

useSeoMeta({
  title: 'Spell Schools - D&D 5e Compendium',
  description: 'Browse all D&D 5e schools of magic: Abjuration, Conjuration, Divination, Enchantment, Evocation, Illusion, Necromancy, and Transmutation.',
})

useHead({
  title: 'Spell Schools - D&D 5e Compendium',
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <UiListPageHeader
      title="Spell Schools"
      :total="totalResults"
      description="Browse D&D 5e schools of magic"
      :loading="loading"
    />

    <div class="mb-6">
      <UInput
        v-model="searchQuery"
        icon="i-heroicons-magnifying-glass"
        size="lg"
        placeholder="Search spell schools..."
        :ui="{ icon: { trailing: { pointer: '' } } }"
      >
        <template v-if="searchQuery" #trailing>
          <UButton
            color="gray"
            variant="link"
            icon="i-heroicons-x-mark-20-solid"
            :padded="false"
            @click="searchQuery = ''"
          />
        </template>
      </UInput>
    </div>

    <UiListSkeletonCards v-if="loading" />

    <UiListErrorState
      v-else-if="error"
      :error="error"
      entity-name="Spell Schools"
      @retry="refresh"
    />

    <UiListEmptyState
      v-else-if="spellSchools.length === 0"
      entity-name="spell schools"
      :has-filters="!!searchQuery"
      @clear-filters="searchQuery = ''"
    />

    <div v-else>
      <UiListResultsCount
        :from="1"
        :to="totalResults"
        :total="totalResults"
        entity-name="spell school"
      />

      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 mb-8">
        <SpellSchoolCard
          v-for="spellSchool in spellSchools"
          :key="spellSchool.id"
          :spell-school="spellSchool"
        />
      </div>
    </div>

    <UiBackLink />
    <JsonDebugPanel :data="{ spellSchools, total: totalResults }" title="Spell Schools Data" />
  </div>
</template>
```

### Step 3: Test and verify

```bash
curl -I http://localhost:3000/spell-schools
```

**Expected:** HTTP 200, page displays 8 spell school cards with descriptions

### Step 4: Commit

```bash
git add app/pages/spell-schools/ server/api/spell-schools/
git commit -m "feat: Add Spell Schools reference page

- Created /spell-schools page with search
- Added API proxy endpoint
- Displays 8 schools with descriptions
- Verified in browser (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Item Types - Card Component (TDD)

**Files:**
- Create: `app/components/item-type/ItemTypeCard.vue`
- Create: `tests/components/item-type/ItemTypeCard.test.ts`

### Step 1: Write the failing test

```typescript
// tests/components/item-type/ItemTypeCard.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import ItemTypeCard from '~/components/item-type/ItemTypeCard.vue'

describe('ItemTypeCard', () => {
  const mockItemType = {
    id: 1,
    code: 'A',
    name: 'Ammunition',
    description: 'Arrows, bolts, sling bullets, and other projectiles'
  }

  it('displays item type code as large badge', async () => {
    const wrapper = await mountSuspended(ItemTypeCard, {
      props: { itemType: mockItemType }
    })

    expect(wrapper.text()).toContain('A')
  })

  it('displays item type name as title', async () => {
    const wrapper = await mountSuspended(ItemTypeCard, {
      props: { itemType: mockItemType }
    })

    expect(wrapper.text()).toContain('Ammunition')
  })

  it('displays description', async () => {
    const wrapper = await mountSuspended(ItemTypeCard, {
      props: { itemType: mockItemType }
    })

    expect(wrapper.text()).toContain('Arrows, bolts')
  })

  it('truncates long descriptions', async () => {
    const longDesc = {
      id: 2,
      code: 'W',
      name: 'Weapon',
      description: 'Weapons are used to attack enemies in combat. They come in many varieties including swords, axes, bows, and more exotic options. Each weapon has unique properties that affect how it can be used in battle. Some weapons are simple while others require special training.'
    }

    const wrapper = await mountSuspended(ItemTypeCard, {
      props: { itemType: longDesc }
    })

    const html = wrapper.html()
    expect(html).toContain('line-clamp-2')
  })

  it('displays category badge', async () => {
    const wrapper = await mountSuspended(ItemTypeCard, {
      props: { itemType: mockItemType }
    })

    expect(wrapper.text()).toContain('Item Type')
  })
})
```

### Step 2: Run test to verify it fails

```bash
docker compose exec nuxt npm test -- ItemTypeCard.test.ts
```

**Expected:** Test fails (module not found)

### Step 3: Write minimal implementation

```vue
<!-- app/components/item-type/ItemTypeCard.vue -->
<script setup lang="ts">
interface ItemType {
  id: number
  code: string
  name: string
  description: string
}

interface Props {
  itemType: ItemType
}

defineProps<Props>()
</script>

<template>
  <UCard class="hover:shadow-lg transition-shadow h-full border border-gray-200 dark:border-gray-700">
    <div class="space-y-3">
      <!-- Code Badge -->
      <div class="flex items-center gap-2">
        <UBadge color="neutral" variant="solid" size="lg">
          {{ itemType.code }}
        </UBadge>
      </div>

      <!-- Name -->
      <h3 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
        {{ itemType.name }}
      </h3>

      <!-- Description (truncated) -->
      <p class="text-sm text-gray-600 dark:text-gray-400 line-clamp-2">
        {{ itemType.description }}
      </p>

      <!-- Category Badge -->
      <div class="flex items-center gap-2">
        <UBadge color="neutral" variant="soft" size="xs">
          Item Type
        </UBadge>
      </div>
    </div>
  </UCard>
</template>
```

### Step 4: Run test to verify it passes

```bash
docker compose exec nuxt npm test -- ItemTypeCard.test.ts
```

**Expected:** All 5 tests pass (GREEN)

### Step 5: Commit

```bash
git add app/components/item-type/ tests/components/item-type/
git commit -m "test: Add ItemTypeCard component with TDD (5 tests passing)

- Created ItemTypeCard with code, name, description
- Truncates long descriptions with line-clamp-2
- All 5 tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Item Types - Page and API Proxy

**Files:**
- Create: `app/pages/item-types/index.vue`
- Create: `server/api/item-types/index.get.ts`

### Step 1: Create API proxy

```typescript
// server/api/item-types/index.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const query = getQuery(event)
  const data = await $fetch(`${config.apiBaseServer}/item-types`, { query })
  return data
})
```

### Step 2: Create list page

```vue
<!-- app/pages/item-types/index.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const { apiFetch } = useApi()
const searchQuery = ref('')

const queryParams = computed(() => {
  const params: Record<string, any> = {}
  if (searchQuery.value.trim()) {
    params.q = searchQuery.value.trim()
  }
  return params
})

const { data: itemTypesResponse, pending: loading, error, refresh } = await useAsyncData(
  'item-types-list',
  async () => {
    const response = await apiFetch('/item-types', {
      query: queryParams.value
    })
    return response
  },
  { watch: [queryParams] }
)

const itemTypes = computed(() => itemTypesResponse.value?.data || [])
const totalResults = computed(() => itemTypes.value.length)

useSeoMeta({
  title: 'Item Types - D&D 5e Compendium',
  description: 'Browse all D&D 5e item categories including weapons, armor, potions, tools, and more.',
})

useHead({
  title: 'Item Types - D&D 5e Compendium',
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <UiListPageHeader
      title="Item Types"
      :total="totalResults"
      description="Browse D&D 5e item categories"
      :loading="loading"
    />

    <div class="mb-6">
      <UInput
        v-model="searchQuery"
        icon="i-heroicons-magnifying-glass"
        size="lg"
        placeholder="Search item types..."
        :ui="{ icon: { trailing: { pointer: '' } } }"
      >
        <template v-if="searchQuery" #trailing>
          <UButton
            color="gray"
            variant="link"
            icon="i-heroicons-x-mark-20-solid"
            :padded="false"
            @click="searchQuery = ''"
          />
        </template>
      </UInput>
    </div>

    <UiListSkeletonCards v-if="loading" />

    <UiListErrorState
      v-else-if="error"
      :error="error"
      entity-name="Item Types"
      @retry="refresh"
    />

    <UiListEmptyState
      v-else-if="itemTypes.length === 0"
      entity-name="item types"
      :has-filters="!!searchQuery"
      @clear-filters="searchQuery = ''"
    />

    <div v-else>
      <UiListResultsCount
        :from="1"
        :to="totalResults"
        :total="totalResults"
        entity-name="item type"
      />

      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 mb-8">
        <ItemTypeCard
          v-for="itemType in itemTypes"
          :key="itemType.id"
          :item-type="itemType"
        />
      </div>
    </div>

    <UiBackLink />
    <JsonDebugPanel :data="{ itemTypes, total: totalResults }" title="Item Types Data" />
  </div>
</template>
```

### Step 3: Test and verify

```bash
curl -I http://localhost:3000/item-types
```

**Expected:** HTTP 200, displays item type cards with descriptions

### Step 4: Commit

```bash
git add app/pages/item-types/ server/api/item-types/
git commit -m "feat: Add Item Types reference page

- Created /item-types page with search
- Added API proxy endpoint
- Displays item categories with descriptions
- Verified in browser (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Proficiency Types - Card Component (TDD)

**Files:**
- Create: `app/components/proficiency-type/ProficiencyTypeCard.vue`
- Create: `tests/components/proficiency-type/ProficiencyTypeCard.test.ts`

### Step 1: Write the failing test

```typescript
// tests/components/proficiency-type/ProficiencyTypeCard.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import ProficiencyTypeCard from '~/components/proficiency-type/ProficiencyTypeCard.vue'

describe('ProficiencyTypeCard', () => {
  const mockProficiencyType = {
    id: 1,
    name: 'Light Armor',
    category: 'armor',
    subcategory: 'light'
  }

  it('displays proficiency type name as title', async () => {
    const wrapper = await mountSuspended(ProficiencyTypeCard, {
      props: { proficiencyType: mockProficiencyType }
    })

    expect(wrapper.text()).toContain('Light Armor')
  })

  it('displays category badge', async () => {
    const wrapper = await mountSuspended(ProficiencyTypeCard, {
      props: { proficiencyType: mockProficiencyType }
    })

    expect(wrapper.text()).toContain('armor')
  })

  it('displays subcategory badge when present', async () => {
    const wrapper = await mountSuspended(ProficiencyTypeCard, {
      props: { proficiencyType: mockProficiencyType }
    })

    expect(wrapper.text()).toContain('light')
  })

  it('handles missing subcategory gracefully', async () => {
    const noSubcategory = {
      id: 2,
      name: 'Simple Weapons',
      category: 'weapon',
      subcategory: null
    }

    const wrapper = await mountSuspended(ProficiencyTypeCard, {
      props: { proficiencyType: noSubcategory }
    })

    expect(wrapper.text()).toContain('Simple Weapons')
    expect(wrapper.text()).toContain('weapon')
    expect(wrapper.text()).not.toContain('null')
  })

  it('displays type category badge', async () => {
    const wrapper = await mountSuspended(ProficiencyTypeCard, {
      props: { proficiencyType: mockProficiencyType }
    })

    expect(wrapper.text()).toContain('Proficiency Type')
  })
})
```

### Step 2: Run test to verify it fails

```bash
docker compose exec nuxt npm test -- ProficiencyTypeCard.test.ts
```

**Expected:** Test fails (module not found)

### Step 3: Write minimal implementation

```vue
<!-- app/components/proficiency-type/ProficiencyTypeCard.vue -->
<script setup lang="ts">
interface ProficiencyType {
  id: number
  name: string
  category: string
  subcategory?: string | null
}

interface Props {
  proficiencyType: ProficiencyType
}

defineProps<Props>()
</script>

<template>
  <UCard class="hover:shadow-lg transition-shadow h-full border border-gray-200 dark:border-gray-700">
    <div class="space-y-3">
      <!-- Name -->
      <h3 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
        {{ proficiencyType.name }}
      </h3>

      <!-- Category and Subcategory Badges -->
      <div class="flex items-center gap-2 flex-wrap">
        <UBadge color="neutral" variant="solid" size="md">
          {{ proficiencyType.category }}
        </UBadge>
        <UBadge
          v-if="proficiencyType.subcategory"
          color="neutral"
          variant="soft"
          size="md"
        >
          {{ proficiencyType.subcategory }}
        </UBadge>
      </div>

      <!-- Type Badge -->
      <div class="flex items-center gap-2">
        <UBadge color="neutral" variant="soft" size="xs">
          Proficiency Type
        </UBadge>
      </div>
    </div>
  </UCard>
</template>
```

### Step 4: Run test to verify it passes

```bash
docker compose exec nuxt npm test -- ProficiencyTypeCard.test.ts
```

**Expected:** All 5 tests pass (GREEN)

### Step 5: Commit

```bash
git add app/components/proficiency-type/ tests/components/proficiency-type/
git commit -m "test: Add ProficiencyTypeCard component with TDD (5 tests passing)

- Created ProficiencyTypeCard with category/subcategory badges
- Handles optional subcategory gracefully
- All 5 tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 8: Proficiency Types - Page and API Proxy

**Files:**
- Create: `app/pages/proficiency-types/index.vue`
- Create: `server/api/proficiency-types/index.get.ts`

### Step 1: Create API proxy

```typescript
// server/api/proficiency-types/index.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const query = getQuery(event)
  const data = await $fetch(`${config.apiBaseServer}/proficiency-types`, { query })
  return data
})
```

### Step 2: Create list page

```vue
<!-- app/pages/proficiency-types/index.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const { apiFetch } = useApi()
const searchQuery = ref('')

const queryParams = computed(() => {
  const params: Record<string, any> = {}
  if (searchQuery.value.trim()) {
    params.q = searchQuery.value.trim()
  }
  return params
})

const { data: proficiencyTypesResponse, pending: loading, error, refresh } = await useAsyncData(
  'proficiency-types-list',
  async () => {
    const response = await apiFetch('/proficiency-types', {
      query: queryParams.value
    })
    return response
  },
  { watch: [queryParams] }
)

const proficiencyTypes = computed(() => proficiencyTypesResponse.value?.data || [])
const totalResults = computed(() => proficiencyTypes.value.length)

useSeoMeta({
  title: 'Proficiency Types - D&D 5e Compendium',
  description: 'Browse all D&D 5e proficiency categories including armor, weapons, tools, and skills.',
})

useHead({
  title: 'Proficiency Types - D&D 5e Compendium',
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <UiListPageHeader
      title="Proficiency Types"
      :total="totalResults"
      description="Browse D&D 5e proficiency categories"
      :loading="loading"
    />

    <div class="mb-6">
      <UInput
        v-model="searchQuery"
        icon="i-heroicons-magnifying-glass"
        size="lg"
        placeholder="Search proficiency types..."
        :ui="{ icon: { trailing: { pointer: '' } } }"
      >
        <template v-if="searchQuery" #trailing>
          <UButton
            color="gray"
            variant="link"
            icon="i-heroicons-x-mark-20-solid"
            :padded="false"
            @click="searchQuery = ''"
          />
        </template>
      </UInput>
    </div>

    <UiListSkeletonCards v-if="loading" />

    <UiListErrorState
      v-else-if="error"
      :error="error"
      entity-name="Proficiency Types"
      @retry="refresh"
    />

    <UiListEmptyState
      v-else-if="proficiencyTypes.length === 0"
      entity-name="proficiency types"
      :has-filters="!!searchQuery"
      @clear-filters="searchQuery = ''"
    />

    <div v-else>
      <UiListResultsCount
        :from="1"
        :to="totalResults"
        :total="totalResults"
        entity-name="proficiency type"
      />

      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 mb-8">
        <ProficiencyTypeCard
          v-for="proficiencyType in proficiencyTypes"
          :key="proficiencyType.id"
          :proficiency-type="proficiencyType"
        />
      </div>
    </div>

    <UiBackLink />
    <JsonDebugPanel :data="{ proficiencyTypes, total: totalResults }" title="Proficiency Types Data" />
  </div>
</template>
```

### Step 3: Test and verify

```bash
curl -I http://localhost:3000/proficiency-types
```

**Expected:** HTTP 200, displays proficiency types with category/subcategory badges

### Step 4: Commit

```bash
git add app/pages/proficiency-types/ server/api/proficiency-types/
git commit -m "feat: Add Proficiency Types reference page

- Created /proficiency-types page with search
- Added API proxy endpoint
- Displays categories and subcategories
- Verified in browser (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 9: Skills - Card Component (TDD)

**Files:**
- Create: `app/components/skill/SkillCard.vue`
- Create: `tests/components/skill/SkillCard.test.ts`

### Step 1: Write the failing test

```typescript
// tests/components/skill/SkillCard.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SkillCard from '~/components/skill/SkillCard.vue'

describe('SkillCard', () => {
  const mockSkill = {
    id: 1,
    name: 'Acrobatics',
    ability_score: {
      id: 2,
      code: 'DEX',
      name: 'Dexterity'
    }
  }

  it('displays skill name as title', async () => {
    const wrapper = await mountSuspended(SkillCard, {
      props: { skill: mockSkill }
    })

    expect(wrapper.text()).toContain('Acrobatics')
  })

  it('displays ability score code as badge', async () => {
    const wrapper = await mountSuspended(SkillCard, {
      props: { skill: mockSkill }
    })

    expect(wrapper.text()).toContain('DEX')
  })

  it('displays ability score full name', async () => {
    const wrapper = await mountSuspended(SkillCard, {
      props: { skill: mockSkill }
    })

    expect(wrapper.text()).toContain('Dexterity')
  })

  it('handles missing ability score gracefully', async () => {
    const noAbility = {
      id: 2,
      name: 'Perception',
      ability_score: null
    }

    const wrapper = await mountSuspended(SkillCard, {
      props: { skill: noAbility }
    })

    expect(wrapper.text()).toContain('Perception')
    expect(wrapper.text()).not.toContain('null')
  })

  it('displays category badge', async () => {
    const wrapper = await mountSuspended(SkillCard, {
      props: { skill: mockSkill }
    })

    expect(wrapper.text()).toContain('Skill')
  })

  it('uses info color for ability score badge', async () => {
    const wrapper = await mountSuspended(SkillCard, {
      props: { skill: mockSkill }
    })

    const html = wrapper.html()
    // Info color badge should be present
    expect(html).toMatch(/color.*info|info.*color/)
  })
})
```

### Step 2: Run test to verify it fails

```bash
docker compose exec nuxt npm test -- SkillCard.test.ts
```

**Expected:** Test fails (module not found)

### Step 3: Write minimal implementation

```vue
<!-- app/components/skill/SkillCard.vue -->
<script setup lang="ts">
interface Skill {
  id: number
  name: string
  ability_score?: {
    id: number
    code: string
    name: string
  } | null
}

interface Props {
  skill: Skill
}

defineProps<Props>()
</script>

<template>
  <UCard class="hover:shadow-lg transition-shadow h-full border border-gray-200 dark:border-gray-700">
    <div class="space-y-3">
      <!-- Skill Name -->
      <h3 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
        {{ skill.name }}
      </h3>

      <!-- Ability Score (if present) -->
      <div v-if="skill.ability_score" class="space-y-2">
        <div class="flex items-center gap-2">
          <UBadge color="info" variant="solid" size="md">
            {{ skill.ability_score.code }}
          </UBadge>
        </div>
        <p class="text-sm text-gray-600 dark:text-gray-400">
          {{ skill.ability_score.name }}
        </p>
      </div>

      <!-- Category Badge -->
      <div class="flex items-center gap-2">
        <UBadge color="neutral" variant="soft" size="xs">
          Skill
        </UBadge>
      </div>
    </div>
  </UCard>
</template>
```

### Step 4: Run test to verify it passes

```bash
docker compose exec nuxt npm test -- SkillCard.test.ts
```

**Expected:** All 6 tests pass (GREEN)

### Step 5: Commit

```bash
git add app/components/skill/ tests/components/skill/
git commit -m "test: Add SkillCard component with TDD (6 tests passing)

- Created SkillCard with nested ability_score display
- Info-colored badge for ability score code
- Handles missing ability_score gracefully
- All 6 tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 10: Skills - Page and API Proxy

**Files:**
- Create: `app/pages/skills/index.vue`
- Create: `server/api/skills/index.get.ts`

### Step 1: Create API proxy

```typescript
// server/api/skills/index.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const query = getQuery(event)
  const data = await $fetch(`${config.apiBaseServer}/skills`, { query })
  return data
})
```

### Step 2: Create list page

```vue
<!-- app/pages/skills/index.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const { apiFetch } = useApi()
const searchQuery = ref('')

const queryParams = computed(() => {
  const params: Record<string, any> = {}
  if (searchQuery.value.trim()) {
    params.q = searchQuery.value.trim()
  }
  return params
})

const { data: skillsResponse, pending: loading, error, refresh } = await useAsyncData(
  'skills-list',
  async () => {
    const response = await apiFetch('/skills', {
      query: queryParams.value
    })
    return response
  },
  { watch: [queryParams] }
)

const skills = computed(() => skillsResponse.value?.data || [])
const totalResults = computed(() => skills.value.length)

useSeoMeta({
  title: 'Skills - D&D 5e Compendium',
  description: 'Browse all D&D 5e skills including Acrobatics, Athletics, Stealth, Perception, and more.',
})

useHead({
  title: 'Skills - D&D 5e Compendium',
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <UiListPageHeader
      title="Skills"
      :total="totalResults"
      description="Browse D&D 5e skills"
      :loading="loading"
    />

    <div class="mb-6">
      <UInput
        v-model="searchQuery"
        icon="i-heroicons-magnifying-glass"
        size="lg"
        placeholder="Search skills..."
        :ui="{ icon: { trailing: { pointer: '' } } }"
      >
        <template v-if="searchQuery" #trailing>
          <UButton
            color="gray"
            variant="link"
            icon="i-heroicons-x-mark-20-solid"
            :padded="false"
            @click="searchQuery = ''"
          />
        </template>
      </UInput>
    </div>

    <UiListSkeletonCards v-if="loading" />

    <UiListErrorState
      v-else-if="error"
      :error="error"
      entity-name="Skills"
      @retry="refresh"
    />

    <UiListEmptyState
      v-else-if="skills.length === 0"
      entity-name="skills"
      :has-filters="!!searchQuery"
      @clear-filters="searchQuery = ''"
    />

    <div v-else>
      <UiListResultsCount
        :from="1"
        :to="totalResults"
        :total="totalResults"
        entity-name="skill"
      />

      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 mb-8">
        <SkillCard
          v-for="skill in skills"
          :key="skill.id"
          :skill="skill"
        />
      </div>
    </div>

    <UiBackLink />
    <JsonDebugPanel :data="{ skills, total: totalResults }" title="Skills Data" />
  </div>
</template>
```

### Step 3: Test and verify

```bash
curl -I http://localhost:3000/skills
```

**Expected:** HTTP 200, displays ~18 skill cards with ability score badges

### Step 4: Commit

```bash
git add app/pages/skills/ server/api/skills/
git commit -m "feat: Add Skills reference page

- Created /skills page with search
- Added API proxy endpoint
- Displays skills with linked ability scores
- Verified in browser (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 11: Conditions - Card Component (TDD)

**Files:**
- Create: `app/components/condition/ConditionCard.vue`
- Create: `tests/components/condition/ConditionCard.test.ts`

### Step 1: Write the failing test

```typescript
// tests/components/condition/ConditionCard.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import ConditionCard from '~/components/condition/ConditionCard.vue'

describe('ConditionCard', () => {
  const mockCondition = {
    id: 1,
    name: 'Blinded',
    slug: 'blinded',
    description: "A blinded creature can't see and automatically fails any ability check that requires sight."
  }

  it('displays condition name as title', async () => {
    const wrapper = await mountSuspended(ConditionCard, {
      props: { condition: mockCondition }
    })

    expect(wrapper.text()).toContain('Blinded')
  })

  it('displays description', async () => {
    const wrapper = await mountSuspended(ConditionCard, {
      props: { condition: mockCondition }
    })

    expect(wrapper.text()).toContain("can't see")
  })

  it('truncates long descriptions with line-clamp-3', async () => {
    const longDesc = {
      id: 2,
      name: 'Exhaustion',
      slug: 'exhaustion',
      description: 'Some special abilities and environmental hazards, such as starvation and the long-term effects of freezing or scorching temperatures, can lead to a special condition called exhaustion. Exhaustion is measured in six levels. An effect can give a creature one or more levels of exhaustion, as specified in the effect\'s description. Each level has cumulative effects.'
    }

    const wrapper = await mountSuspended(ConditionCard, {
      props: { condition: longDesc }
    })

    const html = wrapper.html()
    expect(html).toContain('line-clamp-3')
  })

  it('displays category badge', async () => {
    const wrapper = await mountSuspended(ConditionCard, {
      props: { condition: mockCondition }
    })

    expect(wrapper.text()).toContain('Condition')
  })

  it('handles empty description', async () => {
    const noDesc = {
      id: 3,
      name: 'Custom',
      slug: 'custom',
      description: ''
    }

    const wrapper = await mountSuspended(ConditionCard, {
      props: { condition: noDesc }
    })

    expect(wrapper.text()).toContain('Custom')
  })
})
```

### Step 2: Run test to verify it fails

```bash
docker compose exec nuxt npm test -- ConditionCard.test.ts
```

**Expected:** Test fails (module not found)

### Step 3: Write minimal implementation

```vue
<!-- app/components/condition/ConditionCard.vue -->
<script setup lang="ts">
interface Condition {
  id: number
  name: string
  slug: string
  description: string
}

interface Props {
  condition: Condition
}

defineProps<Props>()
</script>

<template>
  <UCard class="hover:shadow-lg transition-shadow h-full border border-gray-200 dark:border-gray-700">
    <div class="space-y-3">
      <!-- Condition Name -->
      <h3 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
        {{ condition.name }}
      </h3>

      <!-- Description (truncated to 3 lines) -->
      <p
        v-if="condition.description"
        class="text-sm text-gray-600 dark:text-gray-400 line-clamp-3"
      >
        {{ condition.description }}
      </p>

      <!-- Category Badge -->
      <div class="flex items-center gap-2">
        <UBadge color="neutral" variant="soft" size="xs">
          Condition
        </UBadge>
      </div>
    </div>
  </UCard>
</template>
```

### Step 4: Run test to verify it passes

```bash
docker compose exec nuxt npm test -- ConditionCard.test.ts
```

**Expected:** All 5 tests pass (GREEN)

### Step 5: Commit

```bash
git add app/components/condition/ tests/components/condition/
git commit -m "test: Add ConditionCard component with TDD (5 tests passing)

- Created ConditionCard with name and description
- Truncates descriptions with line-clamp-3
- Handles empty descriptions gracefully
- All 5 tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 12: Conditions - Page and API Proxy

**Files:**
- Create: `app/pages/conditions/index.vue`
- Create: `server/api/conditions/index.get.ts`

### Step 1: Create API proxy

```typescript
// server/api/conditions/index.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const query = getQuery(event)
  const data = await $fetch(`${config.apiBaseServer}/conditions`, { query })
  return data
})
```

### Step 2: Create list page

```vue
<!-- app/pages/conditions/index.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const { apiFetch } = useApi()
const searchQuery = ref('')

const queryParams = computed(() => {
  const params: Record<string, any> = {}
  if (searchQuery.value.trim()) {
    params.q = searchQuery.value.trim()
  }
  return params
})

const { data: conditionsResponse, pending: loading, error, refresh } = await useAsyncData(
  'conditions-list',
  async () => {
    const response = await apiFetch('/conditions', {
      query: queryParams.value
    })
    return response
  },
  { watch: [queryParams] }
)

const conditions = computed(() => conditionsResponse.value?.data || [])
const totalResults = computed(() => conditions.value.length)

useSeoMeta({
  title: 'Conditions - D&D 5e Compendium',
  description: 'Browse all D&D 5e conditions including Blinded, Charmed, Frightened, Paralyzed, and more.',
})

useHead({
  title: 'Conditions - D&D 5e Compendium',
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <UiListPageHeader
      title="Conditions"
      :total="totalResults"
      description="Browse D&D 5e conditions"
      :loading="loading"
    />

    <div class="mb-6">
      <UInput
        v-model="searchQuery"
        icon="i-heroicons-magnifying-glass"
        size="lg"
        placeholder="Search conditions..."
        :ui="{ icon: { trailing: { pointer: '' } } }"
      >
        <template v-if="searchQuery" #trailing>
          <UButton
            color="gray"
            variant="link"
            icon="i-heroicons-x-mark-20-solid"
            :padded="false"
            @click="searchQuery = ''"
          />
        </template>
      </UInput>
    </div>

    <UiListSkeletonCards v-if="loading" />

    <UiListErrorState
      v-else-if="error"
      :error="error"
      entity-name="Conditions"
      @retry="refresh"
    />

    <UiListEmptyState
      v-else-if="conditions.length === 0"
      entity-name="conditions"
      :has-filters="!!searchQuery"
      @clear-filters="searchQuery = ''"
    />

    <div v-else>
      <UiListResultsCount
        :from="1"
        :to="totalResults"
        :total="totalResults"
        entity-name="condition"
      />

      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 mb-8">
        <ConditionCard
          v-for="condition in conditions"
          :key="condition.id"
          :condition="condition"
        />
      </div>
    </div>

    <UiBackLink />
    <JsonDebugPanel :data="{ conditions, total: totalResults }" title="Conditions Data" />
  </div>
</template>
```

### Step 3: Test and verify

```bash
curl -I http://localhost:3000/conditions
```

**Expected:** HTTP 200, displays ~15 condition cards with descriptions

### Step 4: Commit

```bash
git add app/pages/conditions/ server/api/conditions/
git commit -m "feat: Add Conditions reference page

- Created /conditions page with search
- Added API proxy endpoint
- Displays game conditions with descriptions
- Verified in browser (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 13: Update Navigation with All 6 New Pages

**Files:**
- Modify: `app/layouts/default.vue` or navigation component (exact path TBD)

### Step 1: Find navigation component

```bash
# Search for navigation dropdown
docker compose exec nuxt sh -c 'grep -r "Reference" app/layouts/ app/components/ | grep -i dropdown'
```

### Step 2: Update navigation to add new items alphabetically

Find the navigation items array and update it to include:

```typescript
// Add these items to the Reference dropdown
const referenceItems = [
  { label: 'Ability Scores', to: '/ability-scores' },      // NEW
  { label: 'Conditions', to: '/conditions' },              // NEW
  { label: 'Damage Types', to: '/damage-types' },
  { label: 'Item Types', to: '/item-types' },              // NEW
  { label: 'Languages', to: '/languages' },
  { label: 'Proficiency Types', to: '/proficiency-types' }, // NEW
  { label: 'Sizes', to: '/sizes' },
  { label: 'Skills', to: '/skills' },                      // NEW
  { label: 'Spell Schools', to: '/spell-schools' },        // NEW
  { label: 'Sources', to: '/sources' }
]
```

### Step 3: Verify all pages load

```bash
# Test all 6 new pages
for path in ability-scores spell-schools item-types proficiency-types skills conditions; do
  echo -n "$path: "
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/$path
done
```

**Expected:** All return HTTP 200

### Step 4: Manual browser check

Open navigation dropdown and verify:
- ✅ All 10 reference items visible
- ✅ Alphabetically sorted
- ✅ All links work
- ✅ Active state highlights correct page

### Step 5: Commit

```bash
git add app/layouts/ app/components/
git commit -m "feat: Add 6 new reference pages to navigation

- Added Ability Scores, Conditions, Item Types, Proficiency Types, Skills, Spell Schools
- Navigation dropdown now has 10 reference items
- All items alphabetically sorted
- Verified all links work (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 14: Run Full Test Suite

**Files:**
- No new files, just verification

### Step 1: Run all tests

```bash
docker compose exec nuxt npm test
```

**Expected Output:**
- All tests pass
- New tests: 32 added (5-6 per component × 6)
- Total: 276+ tests passing

### Step 2: Check for test failures

If any tests fail, fix them before proceeding.

### Step 3: Verify test count

```bash
docker compose exec nuxt npm test -- --reporter=verbose | grep -i "test files"
```

**Expected:** Should show increased test count

---

## Task 15: Update Documentation

**Files:**
- Modify: `docs/CURRENT_STATUS.md`
- Create: `docs/HANDOVER-2025-11-21-REFERENCE-PAGES-BATCH-2.md`

### Step 1: Update CURRENT_STATUS.md

Update these sections:

```markdown
## 🎯 Project Overview
- **10 Reference Pages:** (update from 4 to 10)

## ✅ What's Complete and Working

### Reference Pages (10/10) ✅
**✅ Sources, ✅ Languages, ✅ Sizes, ✅ Damage Types, ✅ Ability Scores, ✅ Spell Schools, ✅ Item Types, ✅ Proficiency Types, ✅ Skills, ✅ Conditions**

**Reference Page Details:**
- **Ability Scores** (6 items) - D&D ability scores with codes
- **Spell Schools** (8 items) - Schools of magic with descriptions
- **Item Types** (~20 items) - Item categories with descriptions
- **Proficiency Types** (~40 items) - Proficiency categories/subcategories
- **Skills** (18 items) - Skills with linked ability scores
- **Conditions** (~15 items) - Game conditions with descriptions

## 📊 Current Stats

**Test Coverage:**
- ✅ **276+ tests total** (ALL PASSING ✅)
- ✅ **32 new tests** for reference card components
- Updated reference component count
```

### Step 2: Create handover document

```markdown
<!-- docs/HANDOVER-2025-11-21-REFERENCE-PAGES-BATCH-2.md -->
# Handover: Reference Pages Batch 2 (6 New Reference Types)

**Date:** 2025-11-21
**Session Duration:** ~2.5 hours
**Status:** ✅ COMPLETE

## Summary

Added 6 new reference pages following TDD and the established pattern from batch 1.

## What Was Built

### New Reference Pages (6)
1. **Ability Scores** (`/ability-scores`) - 6 items
2. **Spell Schools** (`/spell-schools`) - 8 items
3. **Item Types** (`/item-types`) - ~20 items
4. **Proficiency Types** (`/proficiency-types`) - ~40 items
5. **Skills** (`/skills`) - 18 items
6. **Conditions** (`/conditions`) - ~15 items

### Components Created (6 card components)
- `AbilityScoreCard.vue` (5 tests)
- `SpellSchoolCard.vue` (6 tests)
- `ItemTypeCard.vue` (5 tests)
- `ProficiencyTypeCard.vue` (5 tests)
- `SkillCard.vue` (6 tests)
- `ConditionCard.vue` (5 tests)

### Total Impact
- **32 new tests** (all passing)
- **12 new files** (6 components + 6 pages)
- **6 new API proxies**
- **10 reference pages total** (4 from batch 1 + 6 from batch 2)
- **~110 reference items** in compendium

## TDD Process Followed

Every component was built using RED-GREEN-REFACTOR:
1. Write failing tests first
2. Verify tests fail
3. Write minimal implementation
4. Verify tests pass
5. Commit

## Git Commits

- 12 commits (2 per entity)
- 1 navigation update commit
- Clean, descriptive commit messages
- All work properly attributed

## Testing

All pages verified:
- ✅ HTTP 200 status
- ✅ Search functionality
- ✅ Light/dark mode
- ✅ Responsive grid
- ✅ All tests passing

## Next Steps

- Consider adding detail pages for reference items
- Consider cross-references between entities
- Performance optimization if needed
```

### Step 3: Commit documentation

```bash
git add docs/
git commit -m "docs: Update status and create handover for reference pages batch 2

- Updated CURRENT_STATUS.md (4 → 10 reference pages)
- Created HANDOVER-2025-11-21-REFERENCE-PAGES-BATCH-2.md
- Documented all 6 new reference types
- Updated test counts (244 → 276+ tests)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Completion Checklist

- [ ] Task 1: Ability Scores card (TDD)
- [ ] Task 2: Ability Scores page + proxy
- [ ] Task 3: Spell Schools card (TDD)
- [ ] Task 4: Spell Schools page + proxy
- [ ] Task 5: Item Types card (TDD)
- [ ] Task 6: Item Types page + proxy
- [ ] Task 7: Proficiency Types card (TDD)
- [ ] Task 8: Proficiency Types page + proxy
- [ ] Task 9: Skills card (TDD)
- [ ] Task 10: Skills page + proxy
- [ ] Task 11: Conditions card (TDD)
- [ ] Task 12: Conditions page + proxy
- [ ] Task 13: Update navigation
- [ ] Task 14: Run full test suite (all passing)
- [ ] Task 15: Update documentation

**Total Commits:** 13 (12 entity commits + 1 nav/docs commit)

---

## Success Metrics

**Functional:**
- ✅ All 6 pages return HTTP 200
- ✅ Search works on all pages
- ✅ All API data displayed correctly
- ✅ Navigation includes all new items

**Quality:**
- ✅ 32 new tests (5-6 per component)
- ✅ 100% test pass rate
- ✅ TDD followed for every component
- ✅ All components use TypeScript

**User Experience:**
- ✅ Consistent with existing reference pages
- ✅ Light/dark mode works
- ✅ Responsive (375px to 1440px+)
- ✅ Loading/error/empty states

---

## Design Patterns Applied

- **DRY:** Reused existing page template and card patterns
- **YAGNI:** No unnecessary features, just core display
- **TDD:** Tests written FIRST for every component
- **Frequent commits:** 13 commits (one per task completion)
- **Neutral theme:** All reference material uses gray
- **Text truncation:** Appropriate line-clamp for descriptions
- **Optional chaining:** Handles missing nested data gracefully

---

**Plan Status:** ✅ COMPLETE AND READY FOR EXECUTION

**Estimated Time:** 2-3 hours total

**Next Step:** Execute tasks 1-15 sequentially using @superpowers:executing-plans
