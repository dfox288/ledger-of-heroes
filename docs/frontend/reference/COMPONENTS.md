# Component Library Documentation

**Last Updated:** 2025-11-23

This document provides an overview of the D&D 5e Compendium component library and how to use Storybook for interactive component exploration.

---

## 📚 What is This?

The D&D 5e Compendium has **33+ reusable Vue 3 components** built with:
- **Nuxt 4** - Modern Vue framework with SSR
- **NuxtUI 4** - Component library with Tailwind CSS
- **TypeScript** - Full type safety
- **Vitest** - 731 passing tests (100%)

This component library powers all 7 entity types (Spells, Items, Races, Classes, Backgrounds, Feats, Monsters) and 10 reference pages.

---

## 🎨 Viewing Components in Storybook

**Storybook** is an interactive playground where you can:
- ✅ Browse all components visually
- ✅ Test different props and states
- ✅ View auto-generated documentation
- ✅ Copy code examples
- ✅ Toggle dark mode

### Running Storybook

```bash
# Start Storybook
npm run storybook

# Opens at http://localhost:6006
```

**Inside Docker:**
```bash
docker compose exec nuxt npm run storybook
```

---

## 📦 Component Categories

### 1. List Page Components (`app/components/ui/list/`)

Used on all entity list pages (spells, items, etc.):

| Component | Purpose | Story Count |
|-----------|---------|-------------|
| `PageHeader.vue` | Page title with count and description | 6 variants |
| `SkeletonCards.vue` | Animated loading placeholders | 5 variants |
| `EmptyState.vue` | No results message with filter clearing | 5 variants |
| `ResultsCount.vue` | Pagination info (1-24 of 150) | 7 variants |
| `Pagination.vue` | Page navigation controls | - |
| `ErrorState.vue` | Error handling with retry | - |

**Pattern:** All list pages follow the same structure using `useEntityList` composable.

### 2. Navigation Components

| Component | Purpose | Story Count |
|-----------|---------|-------------|
| `BackLink.vue` | Breadcrumb navigation | 4 variants |

### 3. Card Components (`app/components/*/`)

Display entities in list views:

| Component | Entity | Features |
|-----------|--------|----------|
| `SpellCard.vue` | Spells | Level, school, ritual/concentration badges |
| `ItemCard.vue` | Items | Rarity colors, magic/attunement |
| `RaceCard.vue` | Races | Size, speed, ability modifiers |
| `ClassCard.vue` | Classes | Hit die, spellcasting ability |
| `BackgroundCard.vue` | Backgrounds | Skills, languages, tools |
| `FeatCard.vue` | Feats | Prerequisites, modifiers |
| `MonsterCard.vue` | Monsters | CR badges, type |
| *Reference cards* | 10 types | Sizes, languages, schools, etc. |

**All card components:**
- Semantic entity-specific colors
- Background images (10% opacity)
- Source citations
- Hover effects
- Dark mode support

### 4. Detail Page Components (`app/components/ui/detail/`)

Used on entity detail pages:

| Component | Purpose |
|-----------|---------|
| `UiDetailPageHeader.vue` | Title with dynamic badges |
| `UiDetailQuickStatsCard.vue` | Grid of key stats |
| `UiDetailPageLoading.vue` | Full-page skeleton |
| `UiDetailPageError.vue` | 404 error state |
| `UiDetailEntityImage.vue` | CV-style hero images |
| `UiDetailDescriptionCard.vue` | Description section |

### 5. Accordion Components (`app/components/ui/accordion/`)

Organize secondary data on detail pages:

| Component | Purpose |
|-----------|---------|
| `UiAccordionBadgeList.vue` | Badge collections |
| `UiAccordionBulletList.vue` | Bullet-point lists |
| `UiAccordionTraitsList.vue` | Traits/features with levels |
| `UiAccordionDamageEffects.vue` | Spell effects with scaling |
| `UiAccordionSavingThrows.vue` | Save requirements |
| `UiAccordionRandomTablesList.vue` | Dice roll tables |
| `UiAccordionItemSpells.vue` | Item-granted spells |
| `UiAccordionPropertiesList.vue` | Item properties |
| `UiAccordionAbilitiesList.vue` | Item abilities |
| `UiAccordionEntityGrid.vue` | Related entities |
| `UiAccordionSubclassesList.vue` | Subclass features |
| `UiAccordionActions.vue` | Monster actions |
| `UiAccordionTraits.vue` | Monster traits |

### 6. Display Components

Reusable data display patterns:

| Component | Purpose |
|-----------|---------|
| `SourceDisplay.vue` | Source book citations |
| `ModifiersDisplay.vue` | Character modifiers (advantage/disadvantage) |
| `TagsDisplay.vue` | Tag pill display |
| `JsonDebugPanel.vue` | Developer debug toggle |

---

## 🎯 Component Usage Patterns

### List Page Pattern

All 17 list pages follow this structure:

```vue
<script setup lang="ts">
const queryBuilder = computed(() => ({
  level: selectedLevel.value,
  school: selectedSchool.value
}))

const {
  searchQuery,
  currentPage,
  data,
  loading,
  error,
  totalResults,
  hasActiveFilters,
  clearFilters
} = useEntityList({
  endpoint: '/spells',
  cacheKey: 'spells-list',
  queryBuilder,
  seo: { title: 'Spells', description: '...' }
})
</script>

<template>
  <div>
    <UiListPageHeader :title="Spells" :total="totalResults" :loading="loading" />
    
    <!-- Filters -->
    <div class="mb-6">
      <UInput v-model="searchQuery" placeholder="Search..." />
    </div>
    
    <!-- Loading State -->
    <UiListSkeletonCards v-if="loading" />
    
    <!-- Error State -->
    <UiListErrorState v-else-if="error" @retry="refresh" />
    
    <!-- Empty State -->
    <UiListEmptyState
      v-else-if="data.length === 0"
      entity-name="spells"
      :has-filters="hasActiveFilters"
      @clear-filters="clearFilters"
    />
    
    <!-- Results -->
    <template v-else>
      <UiListResultsCount :from="meta.from" :to="meta.to" :total="totalResults" entity-name="spell" />
      
      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        <SpellCard v-for="spell in data" :key="spell.id" :spell="spell" />
      </div>
      
      <UiListPagination v-model:page="currentPage" :items-per-page="24" :total="totalResults" />
    </template>
  </div>
</template>
```

**Components used:** 6-7 standard components per list page

### Detail Page Pattern

All detail pages follow this structure:

```vue
<template>
  <div>
    <!-- Loading State -->
    <UiDetailPageLoading v-if="loading" />
    
    <!-- Error State -->
    <UiDetailPageError v-else-if="error" />
    
    <!-- Content -->
    <template v-else>
      <UiDetailPageHeader :title="spell.name" :badges="badges" />
      
      <div class="grid md:grid-cols-3 gap-6">
        <!-- Main Content (2/3 width) -->
        <div class="md:col-span-2 space-y-6">
          <UiDetailQuickStatsCard :stats="quickStats" />
          
          <UAccordion :items="accordionItems">
            <!-- Use specific accordion slot components -->
          </UAccordion>
        </div>
        
        <!-- Sidebar (1/3 width) -->
        <div>
          <UiDetailEntityImage :src="imageSrc" :alt="spell.name" />
        </div>
      </div>
    </template>
  </div>
</template>
```

---

## 🎨 Design System

### Colors

**Entity-Specific Colors:**
- **Spells:** Purple (arcane)
- **Items:** Gold (treasure)
- **Races:** Green (glory)
- **Classes:** Red (error)
- **Backgrounds:** Amber (lore)
- **Feats:** Blue (info)
- **Monsters:** Orange (danger)

**Reference Entities:**
- Ability Scores: Indigo
- Conditions: Pink
- Damage Types: Slate
- Item Types: Teal
- Languages: Cyan
- Proficiencies: Lime
- Sizes: Zinc
- Skills: Yellow
- Spell Schools: Fuchsia
- Sources: Neutral

### Typography

- **Main headings:** `text-4xl` to `text-5xl font-bold`
- **Section headers:** `text-xl` to `text-2xl font-semibold`
- **Card titles:** `text-xl font-semibold`
- **Body text:** `text-base` to `text-lg leading-relaxed`
- **Stats labels:** `text-sm uppercase font-semibold`

### Spacing

- **Page sections:** `space-y-8`
- **Card content:** `space-y-3`
- **Grid gaps:** `gap-4` (list pages), `gap-6` (stats grids)

---

## 🧪 Testing

All components have comprehensive test coverage:

```bash
# Run tests
npm run test

# Watch mode
npm run test:watch

# With UI
npm run test:ui
```

**Current stats:** 731/731 tests passing (100%)

---

## 🚀 Creating New Components

### 1. Create Component File

```vue
<!-- app/components/ui/MyComponent.vue -->
<script setup lang="ts">
interface Props {
  /** Component prop description */
  title: string
  count?: number
}

defineProps<Props>()
</script>

<template>
  <div>
    <h2>{{ title }}</h2>
    <span v-if="count">{{ count }}</span>
  </div>
</template>
```

### 2. Create Story File

```typescript
// app/components/ui/MyComponent.stories.ts
import type { Meta, StoryObj } from '@storybook/vue3'
import MyComponent from './MyComponent.vue'

const meta: Meta<typeof MyComponent> = {
  title: 'UI/MyComponent',
  component: MyComponent,
  tags: ['autodocs']
}

export default meta
type Story = StoryObj<typeof MyComponent>

export const Default: Story = {
  args: {
    title: 'Hello World',
    count: 42
  }
}
```

### 3. Create Tests (TDD!)

```typescript
// tests/components/ui/MyComponent.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import MyComponent from '~/components/ui/MyComponent.vue'

describe('MyComponent', () => {
  it('renders title', async () => {
    const wrapper = await mountSuspended(MyComponent, {
      props: { title: 'Test' }
    })
    expect(wrapper.text()).toContain('Test')
  })
})
```

### 4. Add JSDoc Comments

Document all props with JSDoc for IDE autocomplete and Storybook auto-docs.

---

## 📚 Additional Resources

- **Storybook:** http://localhost:6006
- **Nuxt Docs:** https://nuxt.com/docs/4.x
- **NuxtUI Docs:** https://ui.nuxt.com/docs
- **Design Document:** `docs/plans/2025-11-23-storybook-setup-design.md`

---

## ✨ What's Next?

### Short Term
- Add stories for remaining components (Tier 2-4)
- Create MDX documentation for patterns
- Add Storybook addons (a11y, interactions)

### Long Term
- Visual regression testing with Chromatic
- Deploy Storybook to GitHub Pages
- Component templates/generators
- Figma integration

---

**Status:** 5 of 33 components documented in Storybook (Tier 1 complete)
**Last Updated:** 2025-11-23
**Maintainer:** Claude Code + dfox
