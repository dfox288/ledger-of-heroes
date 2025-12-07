# Feat Detail Page Redesign

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Issue:** [#57](https://github.com/dfox288/dnd-rulebook-project/issues/57)
**Date:** 2025-11-30
**Status:** Ready for Implementation

**Goal:** Redesign feat detail page to surface benefits prominently and enable variant discovery.

**Architecture:** Dedicated `useFeatDetail()` composable extracts all computed values. Two new components (`FeatBenefitsGrid`, `FeatVariantsSection`) handle the new UI sections. Page component orchestrates layout using existing shared UI components.

**Tech Stack:** Vue 3, Nuxt 4, NuxtUI 4, TypeScript, Vitest

## Overview

Redesign the feat detail page to surface valuable data that's currently hidden, leverage new API fields (`is_half_feat`, `parent_feat_slug`), and follow established patterns from background and class pages.

## Current Problems

1. **Generic stats** - Shows "Type: Feat" instead of meaningful information
2. **Hidden benefits** - Modifiers, proficiencies, conditions buried in accordions
3. **Missing new fields** - `is_half_feat` and `parent_feat_slug` not displayed
4. **No variant discovery** - Can't find related feats (e.g., all Resilient variants)

## API Data Available

| Field | Coverage | Display Value |
|-------|----------|---------------|
| `is_half_feat` | 49% of feats | High - optimizer favorite |
| `parent_feat_slug` | 39% of feats | High - variant grouping |
| `modifiers[]` | 6 feats | Medium - +1 ASI display |
| `proficiencies[]` | 6 feats | Medium - armor/weapon grants |
| `conditions[]` | 17 feats | High - advantages/special abilities |
| `prerequisites[]` | 30% of feats | High - who can take it |

## Design

### Page Layout

```
┌─────────────────────────────────────────────────────────────┐
│ Feats > War Caster                                          │
├─────────────────────────────────────────────────────────────┤
│ WAR CASTER                    [FEAT] [HALF-FEAT] [+1 CHA]   │
├─────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────┐ ┌─────────────────────────┐ │
│ │ PREREQUISITES               │ │                         │ │
│ │ • Ability to cast a spell   │ │     [FEAT IMAGE]        │ │
│ │                             │ │                         │ │
│ └─────────────────────────────┘ └─────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ WHAT YOU GET                                                │
│ ┌───────────────┐ ┌───────────────┐ ┌───────────────────┐   │
│ │ 📈 Ability    │ │ 🛡️ Proficiency │ │ ⚡ Advantage      │   │
│ │ +1 Charisma   │ │ Heavy Armor   │ │ CON concentration │   │
│ └───────────────┘ └───────────────┘ └───────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│ DESCRIPTION                                                 │
│ You have practiced casting spells in the midst of combat... │
├─────────────────────────────────────────────────────────────┤
│ RELATED VARIANTS (only if parent_feat_slug exists)          │
│ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐            │
│ │Resilient│ │Resilient│ │Resilient│ │Resilient│ ...        │
│ │  (STR)  │ │  (DEX)  │ │  (CON)  │ │  (WIS)  │            │
│ └─────────┘ └─────────┘ └─────────┘ └─────────┘            │
├─────────────────────────────────────────────────────────────┤
│ ▶ Source                                                    │
│ ▶ Tags                                                      │
├─────────────────────────────────────────────────────────────┤
│                    ← Back to Feats                          │
└─────────────────────────────────────────────────────────────┘
```

### Section Details

#### 1. Header
- Feat name as title
- Badges: `[FEAT]` (always), `[HALF-FEAT]` (if `is_half_feat`), `[+1 CHA]` (for each modifier)

#### 2. Prerequisites + Image (2/3 + 1/3 grid)
- Keep current pattern
- Show structured prerequisites or `prerequisites_text`
- Feat image on right

#### 3. "What You Get" Card Grid
Surface benefits prominently in a responsive card grid:
- **Ability Score card** - From `modifiers[]` where `modifier_category === 'ability_score'`
- **Proficiency card** - From `proficiencies[]`
- **Advantage card** - From `conditions[]` where `effect_type === 'advantage'`

Grid behavior:
- 1 card: full width mobile, centered desktop
- 2 cards: 2-column grid
- 3 cards: 3-column grid on large screens
- 0 cards: section hidden

#### 4. Description
- Render as markdown (preserve bullet formatting)
- Use existing `UiDetailDescriptionCard` component

#### 5. Related Variants
- Only shown if `parent_feat_slug` exists
- Fetch via API: `filter=parent_feat_slug = {slug}`
- Display as linked cards in 4-column grid
- Highlight current feat with ring
- Exclude current feat from "other variants" but show in grid for context

#### 6. Accordions
Minimal - only for:
- Source (book references)
- Tags (if any)

## Component Architecture

```
app/
├── composables/
│   └── useFeatDetail.ts          # NEW
├── components/
│   └── feat/
│       ├── BenefitsGrid.vue      # NEW
│       └── VariantsSection.vue   # NEW
└── pages/
    └── feats/
        └── [slug].vue            # REWRITE
```

### useFeatDetail Composable

```typescript
// app/composables/useFeatDetail.ts
export function useFeatDetail(slug: MaybeRef<string>) {
  // Fetch feat data
  const { data: entity, pending, error } = useEntityDetail<Feat>({
    slug: toValue(slug),
    endpoint: '/feats',
    cacheKey: 'feat',
    seo: { /* ... */ }
  })

  // Computed: is half feat
  const isHalfFeat = computed(() => entity.value?.is_half_feat ?? false)

  // Computed: ability modifiers
  const abilityModifiers = computed(() => {
    if (!entity.value?.modifiers) return []
    return entity.value.modifiers
      .filter(m => m.modifier_category === 'ability_score')
      .map(m => ({
        ability: m.ability_score?.name ?? 'Unknown',
        code: m.ability_score?.code ?? '?',
        value: parseInt(m.value) || 0
      }))
  })

  // Computed: granted proficiencies
  const grantedProficiencies = computed(() => {
    if (!entity.value?.proficiencies) return []
    return entity.value.proficiencies
      .filter(p => p.grants)
      .map(p => ({
        name: p.proficiency_name,
        type: p.proficiency_type
      }))
  })

  // Computed: advantages/conditions
  const advantages = computed(() => {
    if (!entity.value?.conditions) return []
    return entity.value.conditions.map(c => ({
      effectType: c.effect_type,
      description: c.description
    }))
  })

  // Computed: has any benefits to show
  const hasBenefits = computed(() =>
    abilityModifiers.value.length > 0 ||
    grantedProficiencies.value.length > 0 ||
    advantages.value.length > 0
  )

  // Computed: prerequisites
  const hasPrerequisites = computed(() =>
    (entity.value?.prerequisites?.length ?? 0) > 0 ||
    !!entity.value?.prerequisites_text
  )

  const prerequisitesList = computed(() => {
    if (!entity.value) return []
    const list: string[] = []

    // Add text prerequisite if exists
    if (entity.value.prerequisites_text) {
      list.push(entity.value.prerequisites_text)
    }

    // Add structured prerequisites
    entity.value.prerequisites?.forEach(p => {
      if (p.description) {
        list.push(p.description)
      } else if (p.prerequisite?.name) {
        list.push(p.prerequisite.name)
      }
    })

    return list
  })

  // Parent feat slug for variants
  const parentFeatSlug = computed(() => entity.value?.parent_feat_slug ?? null)

  // Fetch related variants (separate API call)
  const { data: variantsData } = useAsyncData(
    `feat-variants-${toValue(slug)}`,
    async () => {
      const parent = parentFeatSlug.value
      if (!parent) return []

      const response = await apiFetch<{ data: Feat[] }>('/feats', {
        query: { filter: `parent_feat_slug = ${parent}` }
      })
      return response.data ?? []
    },
    { watch: [parentFeatSlug] }
  )

  const relatedVariants = computed(() => variantsData.value ?? [])

  // Sources and tags
  const sources = computed(() => entity.value?.sources ?? [])
  const tags = computed(() => entity.value?.tags ?? [])

  return {
    entity,
    pending,
    error,
    isHalfFeat,
    abilityModifiers,
    grantedProficiencies,
    advantages,
    hasBenefits,
    hasPrerequisites,
    prerequisitesList,
    parentFeatSlug,
    relatedVariants,
    sources,
    tags
  }
}
```

### FeatBenefitsGrid Component

```vue
<!-- app/components/feat/BenefitsGrid.vue -->
<script setup lang="ts">
interface AbilityModifier {
  ability: string
  code: string
  value: number
}

interface Proficiency {
  name: string
  type: string
}

interface Advantage {
  effectType: string
  description: string
}

defineProps<{
  abilityModifiers: AbilityModifier[]
  grantedProficiencies: Proficiency[]
  advantages: Advantage[]
}>()
</script>

<template>
  <section>
    <h2 class="text-xl font-bold text-gray-900 dark:text-gray-100 mb-4">
      What You Get
    </h2>
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      <!-- Ability Score Card -->
      <UCard v-if="abilityModifiers.length > 0">
        <template #header>
          <div class="flex items-center gap-2 text-primary">
            <UIcon name="i-heroicons-arrow-trending-up" class="w-5 h-5" />
            <span class="font-semibold">Ability Score</span>
          </div>
        </template>
        <ul class="space-y-1">
          <li v-for="mod in abilityModifiers" :key="mod.code" class="flex items-center gap-2">
            <UBadge :label="mod.code" color="primary" variant="subtle" size="md" />
            <span>+{{ mod.value }} {{ mod.ability }}</span>
          </li>
        </ul>
      </UCard>

      <!-- Proficiency Card -->
      <UCard v-if="grantedProficiencies.length > 0">
        <template #header>
          <div class="flex items-center gap-2 text-success">
            <UIcon name="i-heroicons-shield-check" class="w-5 h-5" />
            <span class="font-semibold">Proficiency</span>
          </div>
        </template>
        <ul class="space-y-1">
          <li v-for="prof in grantedProficiencies" :key="prof.name" class="flex items-center gap-2">
            <span class="capitalize">{{ prof.name }}</span>
            <UBadge :label="prof.type" color="neutral" variant="subtle" size="sm" />
          </li>
        </ul>
      </UCard>

      <!-- Advantage Card -->
      <UCard v-if="advantages.length > 0">
        <template #header>
          <div class="flex items-center gap-2 text-warning">
            <UIcon name="i-heroicons-bolt" class="w-5 h-5" />
            <span class="font-semibold">Special Abilities</span>
          </div>
        </template>
        <ul class="space-y-2">
          <li v-for="(adv, idx) in advantages" :key="idx">
            <UBadge
              v-if="adv.effectType"
              :label="adv.effectType"
              color="warning"
              variant="subtle"
              size="sm"
              class="mr-2"
            />
            <span class="text-sm">{{ adv.description }}</span>
          </li>
        </ul>
      </UCard>
    </div>
  </section>
</template>
```

### FeatVariantsSection Component

```vue
<!-- app/components/feat/VariantsSection.vue -->
<script setup lang="ts">
import type { Feat } from '~/types/api/entities'

const props = defineProps<{
  variants: Feat[]
  currentSlug: string
}>()

// Filter out current feat for "other variants" label, but show all in grid
const otherVariantsCount = computed(() =>
  props.variants.filter(v => v.slug !== props.currentSlug).length
)
</script>

<template>
  <section v-if="variants.length > 1">
    <h2 class="text-xl font-bold text-gray-900 dark:text-gray-100 mb-4">
      Related Variants
      <span class="text-base font-normal text-gray-500 dark:text-gray-400">
        ({{ otherVariantsCount }} other{{ otherVariantsCount === 1 ? '' : 's' }})
      </span>
    </h2>
    <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 xl:grid-cols-6 gap-3">
      <NuxtLink
        v-for="variant in variants"
        :key="variant.slug"
        :to="`/feats/${variant.slug}`"
        class="block"
      >
        <UCard
          :class="[
            'h-full transition-all',
            variant.slug === currentSlug
              ? 'ring-2 ring-primary bg-primary/5'
              : 'hover:bg-gray-50 dark:hover:bg-gray-800'
          ]"
        >
          <div class="text-center py-2">
            <div class="font-medium text-sm">
              {{ variant.name }}
            </div>
            <div class="mt-1">
              <UBadge
                v-if="variant.is_half_feat"
                label="Half-Feat"
                color="warning"
                variant="subtle"
                size="sm"
              />
            </div>
            <div
              v-if="variant.slug === currentSlug"
              class="text-xs text-primary mt-1"
            >
              (current)
            </div>
          </div>
        </UCard>
      </NuxtLink>
    </div>
  </section>
</template>
```

### Updated Page Component

```vue
<!-- app/pages/feats/[slug].vue -->
<script setup lang="ts">
const route = useRoute()
const slug = computed(() => route.params.slug as string)

const {
  entity,
  pending,
  error,
  isHalfFeat,
  abilityModifiers,
  grantedProficiencies,
  advantages,
  hasBenefits,
  hasPrerequisites,
  prerequisitesList,
  relatedVariants,
  sources,
  tags
} = useFeatDetail(slug)

// Image path
const { getImagePath } = useEntityImage()
const imagePath = computed(() => {
  if (!entity.value) return null
  return getImagePath('feats', entity.value.slug, 512)
})

// Header badges
const headerBadges = computed(() => {
  const badges = [
    { label: 'Feat', color: 'warning' as const, variant: 'subtle' as const, size: 'lg' as const }
  ]

  if (isHalfFeat.value) {
    badges.push({ label: 'Half-Feat', color: 'success' as const, variant: 'subtle' as const, size: 'lg' as const })
  }

  // Add ability modifier badges
  abilityModifiers.value.forEach(mod => {
    badges.push({
      label: `+${mod.value} ${mod.code}`,
      color: 'primary' as const,
      variant: 'subtle' as const,
      size: 'lg' as const
    })
  })

  return badges
})

// Accordion items
const accordionItems = computed(() => {
  const items = []

  if (sources.value.length > 0) {
    items.push({
      label: 'Source',
      icon: 'i-heroicons-book-open',
      defaultOpen: false,
      slot: 'source'
    })
  }

  if (tags.value.length > 0) {
    items.push({
      label: 'Tags',
      icon: 'i-heroicons-tag',
      defaultOpen: false,
      slot: 'tags'
    })
  }

  return items
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <!-- Loading -->
    <UiDetailPageLoading v-if="pending" entity-type="feat" />

    <!-- Error -->
    <UiDetailPageError v-else-if="error" entity-type="Feat" />

    <!-- Content -->
    <div v-else-if="entity" class="space-y-8">
      <!-- Breadcrumb -->
      <UiDetailBreadcrumb
        list-path="/feats"
        list-label="Feats"
        :current-label="entity.name"
      />

      <!-- Header -->
      <UiDetailPageHeader
        :title="entity.name"
        :badges="headerBadges"
      />

      <!-- Prerequisites + Image Grid -->
      <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <!-- Prerequisites -->
        <div class="lg:col-span-2">
          <UCard v-if="hasPrerequisites">
            <template #header>
              <h2 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
                Prerequisites
              </h2>
            </template>
            <ul class="space-y-2">
              <li
                v-for="(prereq, idx) in prerequisitesList"
                :key="idx"
                class="text-gray-700 dark:text-gray-300"
              >
                • {{ prereq }}
              </li>
            </ul>
          </UCard>
          <div v-else class="h-full" />
        </div>

        <!-- Image -->
        <div class="lg:col-span-1">
          <UiDetailEntityImage
            :image-path="imagePath"
            :image-alt="`${entity.name} feat illustration`"
          />
        </div>
      </div>

      <!-- What You Get (Benefits Grid) -->
      <FeatBenefitsGrid
        v-if="hasBenefits"
        :ability-modifiers="abilityModifiers"
        :granted-proficiencies="grantedProficiencies"
        :advantages="advantages"
      />

      <!-- Description -->
      <UiDetailDescriptionCard
        v-if="entity.description"
        :description="entity.description"
      />

      <!-- Related Variants -->
      <FeatVariantsSection
        :variants="relatedVariants"
        :current-slug="entity.slug"
      />

      <!-- Accordions -->
      <UAccordion
        v-if="accordionItems.length > 0"
        :items="accordionItems"
        type="multiple"
      >
        <template #source>
          <UiSourceDisplay :sources="sources" />
        </template>
        <template #tags>
          <UiTagsDisplay :tags="tags" />
        </template>
      </UAccordion>

      <!-- Bottom Nav -->
      <UiDetailPageBottomNav to="/feats" label="Back to Feats" />

      <!-- Debug -->
      <JsonDebugPanel :data="entity" title="Feat Data" />
    </div>
  </div>
</template>
```

## Testing Strategy

### Unit Tests
- `useFeatDetail` composable - test all computed properties
- `FeatBenefitsGrid` - test conditional rendering of cards
- `FeatVariantsSection` - test variant filtering and current highlight

### Integration Tests
- Feat page renders with all sections
- Related variants load for feats with `parent_feat_slug`
- Empty states handled gracefully

### Manual Testing
- Actor (half-feat with modifiers and conditions)
- War Caster (prerequisites, conditions)
- Resilient (Constitution) (variant with parent_feat_slug)
- Lucky (no prerequisites, no modifiers - simple feat)
- Heavily Armored (prerequisites, modifiers, proficiencies)

## Implementation Plan

### Task 1: Create useFeatDetail composable with tests

**Files:**
- Create: `app/composables/useFeatDetail.ts`
- Create: `tests/composables/useFeatDetail.test.ts`

**Step 1: Write the failing test for basic feat loading**

```typescript
// tests/composables/useFeatDetail.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { useFeatDetail } from '~/composables/useFeatDetail'

// Mock the composables
vi.mock('~/composables/useEntityDetail', () => ({
  useEntityDetail: vi.fn()
}))

vi.mock('#app', () => ({
  useAsyncData: vi.fn(() => ({ data: ref([]) })),
  computed: (fn: () => any) => ({ value: fn() }),
  ref: (val: any) => ({ value: val }),
  toValue: (val: any) => val
}))

describe('useFeatDetail', () => {
  it('returns isHalfFeat as true when feat has is_half_feat', () => {
    const mockFeat = {
      id: 1,
      slug: 'actor',
      name: 'Actor',
      is_half_feat: true,
      modifiers: [],
      proficiencies: [],
      conditions: [],
      prerequisites: [],
      sources: []
    }

    // Setup mock return
    vi.mocked(useEntityDetail).mockReturnValue({
      data: ref({ value: mockFeat }),
      pending: ref(false),
      error: ref(null)
    } as any)

    const { isHalfFeat } = useFeatDetail('actor')
    expect(isHalfFeat.value).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useFeatDetail.test.ts`
Expected: FAIL - composable doesn't exist

**Step 3: Create minimal composable**

```typescript
// app/composables/useFeatDetail.ts
import type { Feat } from '~/types/api/entities'
import type { MaybeRef } from 'vue'

export function useFeatDetail(slug: MaybeRef<string>) {
  const { data: entity, pending, error } = useEntityDetail<Feat>({
    slug: toValue(slug),
    endpoint: '/feats',
    cacheKey: 'feat',
    seo: {
      titleTemplate: (name: string) => `${name} - D&D 5e Feat`,
      descriptionExtractor: (feat: unknown) => {
        const f = feat as { description?: string }
        return f.description?.substring(0, 160) || ''
      },
      fallbackTitle: 'Feat - D&D 5e Compendium'
    }
  })

  const isHalfFeat = computed(() => entity.value?.is_half_feat ?? false)

  return {
    entity,
    pending,
    error,
    isHalfFeat
  }
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useFeatDetail.test.ts`
Expected: PASS

**Step 5: Add remaining computed properties and tests**

Add tests for: `abilityModifiers`, `grantedProficiencies`, `advantages`, `hasBenefits`, `hasPrerequisites`, `prerequisitesList`, `sources`, `tags`

Then implement each computed property as shown in the Design section above.

**Step 6: Add related variants fetching**

```typescript
// Add to useFeatDetail.ts
const parentFeatSlug = computed(() => entity.value?.parent_feat_slug ?? null)

const { data: variantsData } = useAsyncData(
  `feat-variants-${toValue(slug)}`,
  async () => {
    const parent = parentFeatSlug.value
    if (!parent) return []

    const response = await apiFetch<{ data: Feat[] }>('/feats', {
      query: { filter: `parent_feat_slug = ${parent}` }
    })
    return response.data ?? []
  },
  { watch: [parentFeatSlug], immediate: false }
)

const relatedVariants = computed(() => variantsData.value ?? [])
```

**Step 7: Run full test suite**

Run: `docker compose exec nuxt npm run test:feats`
Expected: All pass

**Step 8: Commit**

```bash
git add app/composables/useFeatDetail.ts tests/composables/useFeatDetail.test.ts
git commit -m "feat(feats): Add useFeatDetail composable with tests (Issue #57)"
```

---

### Task 2: Create FeatBenefitsGrid component with tests

**Files:**
- Create: `app/components/feat/BenefitsGrid.vue`
- Create: `tests/components/feat/BenefitsGrid.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/feat/BenefitsGrid.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import FeatBenefitsGrid from '~/components/feat/BenefitsGrid.vue'

describe('FeatBenefitsGrid', () => {
  it('renders ability score card when modifiers provided', async () => {
    const wrapper = await mountSuspended(FeatBenefitsGrid, {
      props: {
        abilityModifiers: [{ ability: 'Charisma', code: 'CHA', value: 1 }],
        grantedProficiencies: [],
        advantages: []
      }
    })

    expect(wrapper.text()).toContain('Ability Score')
    expect(wrapper.text()).toContain('+1 Charisma')
    expect(wrapper.text()).toContain('CHA')
  })

  it('hides ability score card when no modifiers', async () => {
    const wrapper = await mountSuspended(FeatBenefitsGrid, {
      props: {
        abilityModifiers: [],
        grantedProficiencies: [],
        advantages: []
      }
    })

    expect(wrapper.text()).not.toContain('Ability Score')
  })

  it('renders proficiency card when proficiencies provided', async () => {
    const wrapper = await mountSuspended(FeatBenefitsGrid, {
      props: {
        abilityModifiers: [],
        grantedProficiencies: [{ name: 'heavy armor', type: 'armor' }],
        advantages: []
      }
    })

    expect(wrapper.text()).toContain('Proficiency')
    expect(wrapper.text()).toContain('heavy armor')
  })

  it('renders advantage card when conditions provided', async () => {
    const wrapper = await mountSuspended(FeatBenefitsGrid, {
      props: {
        abilityModifiers: [],
        grantedProficiencies: [],
        advantages: [{ effectType: 'advantage', description: 'CON saves for concentration' }]
      }
    })

    expect(wrapper.text()).toContain('Special Abilities')
    expect(wrapper.text()).toContain('CON saves for concentration')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/feat/BenefitsGrid.test.ts`
Expected: FAIL - component doesn't exist

**Step 3: Create the component**

Create `app/components/feat/BenefitsGrid.vue` with the code from the Design section above.

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/feat/BenefitsGrid.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/feat/BenefitsGrid.vue tests/components/feat/BenefitsGrid.test.ts
git commit -m "feat(feats): Add FeatBenefitsGrid component with tests (Issue #57)"
```

---

### Task 3: Create FeatVariantsSection component with tests

**Files:**
- Create: `app/components/feat/VariantsSection.vue`
- Create: `tests/components/feat/VariantsSection.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/feat/VariantsSection.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import FeatVariantsSection from '~/components/feat/VariantsSection.vue'

const mockVariants = [
  { id: 1, slug: 'resilient-strength', name: 'Resilient (Strength)', is_half_feat: true },
  { id: 2, slug: 'resilient-dexterity', name: 'Resilient (Dexterity)', is_half_feat: true },
  { id: 3, slug: 'resilient-constitution', name: 'Resilient (Constitution)', is_half_feat: true }
]

describe('FeatVariantsSection', () => {
  it('renders all variant cards', async () => {
    const wrapper = await mountSuspended(FeatVariantsSection, {
      props: {
        variants: mockVariants,
        currentSlug: 'resilient-constitution'
      }
    })

    expect(wrapper.text()).toContain('Resilient (Strength)')
    expect(wrapper.text()).toContain('Resilient (Dexterity)')
    expect(wrapper.text()).toContain('Resilient (Constitution)')
  })

  it('highlights current variant', async () => {
    const wrapper = await mountSuspended(FeatVariantsSection, {
      props: {
        variants: mockVariants,
        currentSlug: 'resilient-constitution'
      }
    })

    expect(wrapper.text()).toContain('(current)')
  })

  it('shows correct other variants count', async () => {
    const wrapper = await mountSuspended(FeatVariantsSection, {
      props: {
        variants: mockVariants,
        currentSlug: 'resilient-constitution'
      }
    })

    expect(wrapper.text()).toContain('2 others')
  })

  it('hides section when only one variant (current)', async () => {
    const wrapper = await mountSuspended(FeatVariantsSection, {
      props: {
        variants: [mockVariants[0]],
        currentSlug: 'resilient-strength'
      }
    })

    expect(wrapper.text()).not.toContain('Related Variants')
  })

  it('hides section when no variants', async () => {
    const wrapper = await mountSuspended(FeatVariantsSection, {
      props: {
        variants: [],
        currentSlug: 'lucky'
      }
    })

    expect(wrapper.text()).not.toContain('Related Variants')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/feat/VariantsSection.test.ts`
Expected: FAIL - component doesn't exist

**Step 3: Create the component**

Create `app/components/feat/VariantsSection.vue` with the code from the Design section above.

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/feat/VariantsSection.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/feat/VariantsSection.vue tests/components/feat/VariantsSection.test.ts
git commit -m "feat(feats): Add FeatVariantsSection component with tests (Issue #57)"
```

---

### Task 4: Rewrite feat detail page

**Files:**
- Modify: `app/pages/feats/[slug].vue`
- Modify: `tests/components/feat/FeatDetailPage.test.ts` (if exists, or create)

**Step 1: Backup current implementation**

Read current file to understand what tests might break.

**Step 2: Rewrite page component**

Replace entire contents of `app/pages/feats/[slug].vue` with the code from the Design section above.

**Step 3: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: PASS

**Step 4: Run feat test suite**

Run: `docker compose exec nuxt npm run test:feats`
Expected: All pass (may need to update existing tests)

**Step 5: Commit**

```bash
git add app/pages/feats/[slug].vue
git commit -m "feat(feats): Redesign feat detail page with benefits grid and variants (Issue #57)"
```

---

### Task 5: Manual QA and polish

**Test Cases:**

| Feat | What to Check |
|------|---------------|
| Actor | Half-feat badge, +1 CHA modifier, advantage condition |
| War Caster | Prerequisites card, advantage condition, no modifiers |
| Resilient (Constitution) | Half-feat badge, +1 CON, 5 other variants shown |
| Lucky | No prerequisites, no benefits grid, simple description |
| Heavily Armored | Prerequisites, +1 STR modifier, heavy armor proficiency |
| Sentinel | No prerequisites, no benefits, pure description feat |

**Step 1: Start dev server**

Run: `docker compose exec nuxt npm run dev`

**Step 2: Test each feat manually**

Navigate to each feat and verify:
- [ ] Header badges render correctly
- [ ] Prerequisites card shows/hides appropriately
- [ ] Benefits grid shows correct cards
- [ ] Description renders as markdown
- [ ] Related variants section shows for variant feats
- [ ] Current variant is highlighted
- [ ] Navigation between variants works
- [ ] Dark mode looks good
- [ ] Mobile responsive

**Step 3: Fix any issues found**

**Step 4: Final commit**

```bash
git add -A
git commit -m "fix(feats): Polish feat detail page based on QA (Issue #57)"
```

---

### Task 6: Update issue and close

**Step 1: Run full test suite**

Run: `docker compose exec nuxt npm run test`
Expected: All pass

**Step 2: Run lint**

Run: `docker compose exec nuxt npm run lint:fix`

**Step 3: Push changes**

```bash
git push
```

**Step 4: Close related issues**

```bash
gh issue close 55 --repo dfox288/dnd-rulebook-project --comment "Addressed by feat detail page redesign in Issue #57"
gh issue close 57 --repo dfox288/dnd-rulebook-project --comment "Feat detail page redesign complete. See commits on main branch."
```

---

## Related Issues

- #55 - Display feat half-feat status (addressed by this redesign)
- #57 - This issue
