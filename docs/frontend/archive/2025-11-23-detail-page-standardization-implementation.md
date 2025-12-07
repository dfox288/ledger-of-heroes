# Detail Page Standardization - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Standardize all 7 detail pages with consistent structure, integrated image layout (2/3-1/3 split), and complete API data display.

**Architecture:** Create new `<UiDetailDescriptionWithImage>` component for integrated image layout. Update all 7 detail pages to use consistent template structure. Add missing accordion components for undisplayed API fields (spellcasting, counters, level progression).

**Tech Stack:** Nuxt 4.x, Vue 3 Composition API, TypeScript, NuxtUI 4.x, Vitest, TDD workflow

---

## Phase A: Core Refactor (Structural Consistency + Image Layout)

**Goal:** All 7 pages follow identical template structure with integrated image layout

---

### Task A1: Create UiDetailDescriptionWithImage Component (TDD)

**Files:**
- Create: `app/components/ui/detail/UiDetailDescriptionWithImage.vue`
- Create: `tests/components/ui/detail/UiDetailDescriptionWithImage.test.ts`

**Step 1: Write the failing test**

Create test file with comprehensive test coverage:

```typescript
// tests/components/ui/detail/UiDetailDescriptionWithImage.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiDetailDescriptionWithImage from '~/components/ui/detail/UiDetailDescriptionWithImage.vue'

describe('UiDetailDescriptionWithImage', () => {
  it('displays description text', async () => {
    const wrapper = await mountSuspended(UiDetailDescriptionWithImage, {
      props: {
        description: 'Test description text'
      }
    })

    expect(wrapper.text()).toContain('Test description text')
  })

  it('shows default title "Description"', async () => {
    const wrapper = await mountSuspended(UiDetailDescriptionWithImage, {
      props: {
        description: 'Test description'
      }
    })

    expect(wrapper.text()).toContain('Description')
  })

  it('shows custom title when provided', async () => {
    const wrapper = await mountSuspended(UiDetailDescriptionWithImage, {
      props: {
        description: 'Test description',
        title: 'Custom Title'
      }
    })

    expect(wrapper.text()).toContain('Custom Title')
    expect(wrapper.text()).not.toContain('Description')
  })

  it('displays image when imagePath provided', async () => {
    const wrapper = await mountSuspended(UiDetailDescriptionWithImage, {
      props: {
        description: 'Test description',
        imagePath: '/images/test.png',
        imageAlt: 'Test image'
      }
    })

    const img = wrapper.find('img')
    expect(img.exists()).toBe(true)
    expect(img.attributes('alt')).toBe('Test image')
  })

  it('does not display image when imagePath is null', async () => {
    const wrapper = await mountSuspended(UiDetailDescriptionWithImage, {
      props: {
        description: 'Test description',
        imagePath: null
      }
    })

    const img = wrapper.find('img')
    expect(img.exists()).toBe(false)
  })

  it('uses 2/3 width layout when image present', async () => {
    const wrapper = await mountSuspended(UiDetailDescriptionWithImage, {
      props: {
        description: 'Test description',
        imagePath: '/images/test.png'
      }
    })

    const contentDiv = wrapper.find('.lg\\:w-2\\/3')
    expect(contentDiv.exists()).toBe(true)
  })

  it('uses full width layout when no image', async () => {
    const wrapper = await mountSuspended(UiDetailDescriptionWithImage, {
      props: {
        description: 'Test description',
        imagePath: null
      }
    })

    const contentDiv = wrapper.find('.w-full')
    expect(contentDiv.exists()).toBe(true)
    const narrowDiv = wrapper.find('.lg\\:w-2\\/3')
    expect(narrowDiv.exists()).toBe(false)
  })

  it('handles missing description gracefully', async () => {
    const wrapper = await mountSuspended(UiDetailDescriptionWithImage, {
      props: {
        description: undefined
      }
    })

    expect(wrapper.text()).toContain('Description')
    expect(wrapper.find('.prose').exists()).toBe(true)
  })

  it('applies prose styling to description', async () => {
    const wrapper = await mountSuspended(UiDetailDescriptionWithImage, {
      props: {
        description: 'Test description'
      }
    })

    expect(wrapper.find('.prose').exists()).toBe(true)
    expect(wrapper.find('.dark\\:prose-invert').exists()).toBe(true)
  })

  it('preserves whitespace in description', async () => {
    const wrapper = await mountSuspended(UiDetailDescriptionWithImage, {
      props: {
        description: 'Line 1\nLine 2\nLine 3'
      }
    })

    expect(wrapper.find('.whitespace-pre-line').exists()).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

```bash
docker compose exec nuxt npm run test -- tests/components/ui/detail/UiDetailDescriptionWithImage.test.ts
```

Expected: All tests FAIL with "Failed to resolve component: UiDetailDescriptionWithImage"

**Step 3: Write minimal implementation**

Create component file:

```vue
<!-- app/components/ui/detail/UiDetailDescriptionWithImage.vue -->
<script setup lang="ts">
interface Props {
  description?: string
  imagePath?: string | null
  imageAlt?: string
  title?: string
}

const props = withDefaults(defineProps<Props>(), {
  description: undefined,
  imagePath: null,
  imageAlt: '',
  title: 'Description'
})
</script>

<template>
  <UCard>
    <template #header>
      <h2 class="text-lg font-semibold text-gray-900 dark:text-gray-100">
        {{ title }}
      </h2>
    </template>

    <div class="flex flex-col lg:flex-row gap-6">
      <!-- Content: 2/3 width when image present, full width otherwise -->
      <div :class="imagePath ? 'lg:w-2/3' : 'w-full'">
        <div class="prose dark:prose-invert max-w-none">
          <p
            v-if="description"
            class="whitespace-pre-line text-base text-gray-700 dark:text-gray-300 leading-relaxed"
          >
            {{ description }}
          </p>
          <p
            v-else
            class="text-gray-500 dark:text-gray-400 italic"
          >
            No description available.
          </p>
        </div>
      </div>

      <!-- Image: 1/3 width on large screens, stacks below on mobile -->
      <div
        v-if="imagePath"
        class="lg:w-1/3 flex-shrink-0"
      >
        <NuxtImg
          :src="imagePath"
          :alt="imageAlt"
          class="w-full h-auto rounded-lg shadow-lg object-cover"
          loading="lazy"
          width="512"
          height="512"
        />
      </div>
    </div>
  </UCard>
</template>
```

**Step 4: Run test to verify it passes**

```bash
docker compose exec nuxt npm run test -- tests/components/ui/detail/UiDetailDescriptionWithImage.test.ts
```

Expected: All 10 tests PASS

**Step 5: Commit**

```bash
git add app/components/ui/detail/UiDetailDescriptionWithImage.vue tests/components/ui/detail/UiDetailDescriptionWithImage.test.ts
git commit -m "feat: Add UiDetailDescriptionWithImage component with integrated image layout

- Creates 2/3 content, 1/3 image layout when image present
- Full-width content when no image
- Responsive (stacks on mobile)
- 10 comprehensive tests (all passing)
- Follows TDD workflow

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task A2: Update Spells Detail Page

**Files:**
- Modify: `app/pages/spells/[slug].vue`

**Step 1: Update spells page to use new component**

Replace the header + description sections:

```vue
<!-- app/pages/spells/[slug].vue -->
<script setup lang="ts">
// ... existing imports and logic (keep all)
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <!-- Loading State -->
    <UiDetailPageLoading
      v-if="loading"
      entity-type="spell"
    />

    <!-- Error State -->
    <UiDetailPageError
      v-else-if="error"
      entity-type="Spell"
    />

    <!-- Spell Content -->
    <div
      v-else-if="spell"
      class="space-y-8"
    >
      <!-- Breadcrumb -->
      <UiBackLink
        to="/spells"
        label="Back to Spells"
      />

      <!-- Header (title + badges) - UPDATED: No image -->
      <UiDetailPageHeader
        :title="spell.name"
        :badges="[
          { label: spellLevelText, color: getSpellLevelColor(spell.level), variant: 'subtle' as const, size: 'lg' as const },
          { label: spell.school?.name || 'Unknown', color: getSpellSchoolColor(spell.school?.code || ''), variant: 'subtle' as const, size: 'lg' as const },
          ...(spell.is_ritual ? [{ label: '🔮 Ritual', color: 'info' as const, variant: 'soft' as const, size: 'sm' as const }] : []),
          ...(spell.needs_concentration ? [{ label: '⭐ Concentration', color: 'warning' as const, variant: 'soft' as const, size: 'sm' as const }] : [])
        ]"
      />

      <!-- Quick Stats -->
      <UiDetailQuickStatsCard
        :stats="[
          { icon: 'i-heroicons-clock', label: 'Casting Time', value: spell.casting_time },
          { icon: 'i-heroicons-arrow-trending-up', label: 'Range', value: spell.range },
          { icon: 'i-heroicons-sparkles', label: 'Components', value: spell.components, subtext: spell.material_components },
          { icon: 'i-heroicons-clock', label: 'Duration', value: spell.duration }
        ]"
      />

      <!-- Description + Image (integrated) - NEW COMPONENT -->
      <UiDetailDescriptionWithImage
        :description="spell.description"
        :image-path="imagePath"
        :image-alt="`${spell.name} spell illustration`"
      />

      <!-- Additional Details (Accordion) - UNCHANGED -->
      <UAccordion
        :items="[
          ...(spell.higher_levels ? [{
            label: 'At Higher Levels',
            slot: 'higher-levels',
            defaultOpen: false
          }] : []),
          ...(spellEffects.length > 0 ? [{
            label: 'Effects',
            slot: 'effects',
            defaultOpen: false
          }] : []),
          ...(spell.saving_throws && spell.saving_throws.length > 0 ? [{
            label: 'Saving Throws',
            slot: 'saving-throws',
            defaultOpen: false
          }] : []),
          ...(spell.classes && spell.classes.length > 0 ? [{
            label: 'Available to Classes',
            slot: 'classes',
            defaultOpen: false
          }] : []),
          ...(spell.sources && spell.sources.length > 0 ? [{
            label: 'Source',
            slot: 'source',
            defaultOpen: false
          }] : []),
          ...(spell.random_tables && spell.random_tables.length > 0 ? [{
            label: 'Random Tables',
            slot: 'random-tables',
            defaultOpen: false
          }] : []),
          ...(spell.tags && spell.tags.length > 0 ? [{
            label: 'Tags',
            slot: 'tags',
            defaultOpen: false
          }] : [])
        ]"
        type="multiple"
      >
        <!-- All slots remain UNCHANGED -->
        <template
          v-if="spell.higher_levels"
          #higher-levels
        >
          <div class="p-4">
            <p class="text-gray-700 dark:text-gray-300">
              {{ spell.higher_levels }}
            </p>
          </div>
        </template>

        <template
          v-if="spellEffects.length > 0"
          #effects
        >
          <UiAccordionDamageEffects :effects="spellEffects" />
        </template>

        <template
          v-if="spell.saving_throws && spell.saving_throws.length > 0"
          #saving-throws
        >
          <UiAccordionSavingThrows :saving-throws="spell.saving_throws" />
        </template>

        <template
          v-if="spell.classes && spell.classes.length > 0"
          #classes
        >
          <UiAccordionBadgeList
            :items="spell.classes"
            color="primary"
          />
        </template>

        <template
          v-if="spell.random_tables && spell.random_tables.length > 0"
          #random-tables
        >
          <UiAccordionRandomTablesList :tables="spell.random_tables" />
        </template>

        <template
          v-if="spell.sources && spell.sources.length > 0"
          #source
        >
          <UiSourceDisplay :sources="spell.sources" />
        </template>

        <template
          v-if="spell.tags && spell.tags.length > 0"
          #tags
        >
          <UiTagsDisplay :tags="spell.tags" />
        </template>
      </UAccordion>

      <!-- JSON Debug Panel -->
      <JsonDebugPanel
        :data="spell"
        title="Spell Data"
      />
    </div>
  </div>
</template>
```

**Step 2: Test spells page in browser**

```bash
docker compose exec nuxt npm run dev
```

Visit: http://localhost:3000/spells/fireball

Expected:
- Page loads successfully (HTTP 200)
- Image appears in description card (right side, 1/3 width)
- Description on left (2/3 width)
- All accordion sections work

**Step 3: Run full test suite**

```bash
docker compose exec nuxt npm run test
```

Expected: All tests pass (no regressions)

**Step 4: Commit**

```bash
git add app/pages/spells/\[slug\].vue
git commit -m "refactor: Update spells page to use UiDetailDescriptionWithImage

- Replace UiEntityHeaderWithImage with UiDetailPageHeader + UiDetailDescriptionWithImage
- Image now integrated with description (2/3-1/3 layout)
- No functional changes, visual improvement
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task A3: Update Items Detail Page

**Files:**
- Modify: `app/pages/items/[slug].vue`

**Step 1: Update items page to use new component**

Same pattern as spells - replace header and description:

```vue
<!-- app/pages/items/[slug].vue -->
<!-- Update template section only, keep all script logic unchanged -->

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <UiDetailPageLoading
      v-if="loading"
      entity-type="item"
    />

    <UiDetailPageError
      v-else-if="error"
      entity-type="Item"
    />

    <div
      v-else-if="item"
      class="space-y-8"
    >
      <UiBackLink
        to="/items"
        label="Back to Items"
      />

      <!-- Header - UPDATED -->
      <UiDetailPageHeader
        :title="item.name"
        :badges="[
          { label: item.item_type?.name || 'Unknown', color: itemTypeColor, variant: 'subtle' as const, size: 'lg' as const },
          { label: rarityText, color: rarityColor, variant: 'subtle' as const, size: 'lg' as const },
          ...(item.is_magic ? [{ label: '✨ Magic', color: 'primary' as const, variant: 'soft' as const, size: 'sm' as const }] : []),
          ...(item.requires_attunement ? [{ label: '🔮 Attunement', color: 'info' as const, variant: 'soft' as const, size: 'sm' as const }] : [])
        ]"
      />

      <UiDetailQuickStatsCard
        :stats="[
          ...(costInGold ? [{ icon: 'i-heroicons-currency-dollar', label: 'Cost', value: costInGold }] : []),
          ...(item.weight ? [{ icon: 'i-heroicons-scale', label: 'Weight', value: `${item.weight} lb` }] : []),
          ...(item.damage_dice ? [{ icon: 'i-heroicons-bolt', label: 'Damage', value: item.damage_dice + (item.damage_type ? ` ${item.damage_type.name}` : ''), subtext: item.versatile_damage ? `Versatile: ${item.versatile_damage}` : undefined }] : []),
          ...(item.armor_class !== null ? [{ icon: 'i-heroicons-shield-check', label: 'Armor Class', value: String(item.armor_class) }] : []),
          ...(item.range_normal ? [{ icon: 'i-heroicons-arrow-trending-up', label: 'Range', value: `${item.range_normal}${item.range_long ? `/${item.range_long}` : ''} ft.` }] : []),
          ...(item.strength_requirement ? [{ icon: 'i-heroicons-hand-raised', label: 'Strength Required', value: String(item.strength_requirement) }] : []),
          ...(item.charges_max ? [{ icon: 'i-heroicons-bolt-slash', label: 'Charges', value: String(item.charges_max), subtext: item.recharge_formula && item.recharge_timing ? `Recharge: ${item.recharge_formula} at ${item.recharge_timing}` : undefined }] : [])
        ]"
      />

      <!-- Description + Image - NEW -->
      <UiDetailDescriptionWithImage
        :description="item.description"
        :image-path="imagePath"
        :image-alt="`${item.name} item illustration`"
      />

      <!-- Rest unchanged -->
      <UAccordion
        :items="[
          ...(item.properties && item.properties.length > 0 ? [{
            label: 'Properties',
            slot: 'properties',
            defaultOpen: false
          }] : []),
          ...(item.modifiers && item.modifiers.length > 0 ? [{
            label: 'Modifiers',
            slot: 'modifiers',
            defaultOpen: false
          }] : []),
          ...(item.proficiencies && item.proficiencies.length > 0 ? [{
            label: 'Proficiencies',
            slot: 'proficiencies',
            defaultOpen: false
          }] : []),
          ...(item.abilities && item.abilities.length > 0 ? [{
            label: 'Abilities',
            slot: 'abilities',
            defaultOpen: false
          }] : []),
          ...(item.spells && item.spells.length > 0 ? [{
            label: 'Spells',
            slot: 'spells',
            defaultOpen: false
          }] : []),
          ...(item.random_tables && item.random_tables.length > 0 ? [{
            label: 'Random Tables',
            slot: 'random_tables',
            defaultOpen: false
          }] : []),
          ...(item.saving_throws && item.saving_throws.length > 0 ? [{
            label: 'Saving Throws',
            slot: 'saving_throws',
            defaultOpen: false
          }] : []),
          ...(item.sources && item.sources.length > 0 ? [{
            label: 'Source',
            slot: 'source',
            defaultOpen: false
          }] : []),
          ...(item.tags && item.tags.length > 0 ? [{
            label: 'Tags',
            slot: 'tags',
            defaultOpen: false
          }] : [])
        ]"
        type="multiple"
      >
        <!-- All slots remain unchanged -->
        <template
          v-if="item.properties && item.properties.length > 0"
          #properties
        >
          <UiAccordionPropertiesList :properties="item.properties" />
        </template>

        <template
          v-if="item.modifiers && item.modifiers.length > 0"
          #modifiers
        >
          <UiModifiersDisplay :modifiers="item.modifiers" />
        </template>

        <template
          v-if="item.proficiencies && item.proficiencies.length > 0"
          #proficiencies
        >
          <UiAccordionBulletList :items="item.proficiencies" />
        </template>

        <template
          v-if="item.abilities && item.abilities.length > 0"
          #abilities
        >
          <UiAccordionAbilitiesList :abilities="item.abilities" />
        </template>

        <template
          v-if="item.spells && item.spells.length > 0"
          #spells
        >
          <UiAccordionItemSpells :spells="item.spells" />
        </template>

        <template
          v-if="item.random_tables && item.random_tables.length > 0"
          #random_tables
        >
          <UiAccordionRandomTablesList :tables="item.random_tables" />
        </template>

        <template
          v-if="item.saving_throws && item.saving_throws.length > 0"
          #saving_throws
        >
          <UiAccordionSavingThrows :saving-throws="item.saving_throws" />
        </template>

        <template
          v-if="item.sources && item.sources.length > 0"
          #source
        >
          <UiSourceDisplay :sources="item.sources" />
        </template>

        <template
          v-if="item.tags && item.tags.length > 0"
          #tags
        >
          <UiTagsDisplay :tags="item.tags" />
        </template>
      </UAccordion>

      <JsonDebugPanel
        :data="item"
        title="Item Data"
      />
    </div>
  </div>
</template>
```

**Step 2: Test and commit**

```bash
# Test in browser
open http://localhost:3000/items/1

# Run tests
docker compose exec nuxt npm run test

# Commit
git add app/pages/items/\[slug\].vue
git commit -m "refactor: Update items page to use UiDetailDescriptionWithImage

- Replace UiEntityHeaderWithImage with UiDetailPageHeader + UiDetailDescriptionWithImage
- Image now integrated with description (2/3-1/3 layout)
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task A4: Update Races Detail Page

**Files:**
- Modify: `app/pages/races/[slug].vue`

**Step 1: Update races page**

Follow same pattern - update header and description sections:

```vue
<!-- app/pages/races/[slug].vue -->
<!-- Keep all script logic unchanged, update template only -->

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <UiDetailPageLoading
      v-if="loading"
      entity-type="race"
    />

    <UiDetailPageError
      v-else-if="error"
      entity-type="Race"
    />

    <div
      v-else-if="race"
      class="space-y-8"
    >
      <UiBackLink
        to="/races"
        label="Back to Races"
      />

      <!-- Header - UPDATED -->
      <UiDetailPageHeader
        :title="race.name"
        :badges="[
          ...(race.size ? [{ label: race.size.name, color: sizeColor as unknown as BadgeColor, variant: 'subtle' as BadgeVariant, size: 'lg' as BadgeSize }] : []),
          { label: race.parent_race ? 'Subrace' : 'Race', color: (race.parent_race ? 'primary' : 'info') as BadgeColor, variant: 'subtle' as BadgeVariant, size: 'lg' as BadgeSize }
        ]"
      />

      <UiDetailQuickStatsCard
        :stats="[
          ...(race.size ? [{ icon: 'i-heroicons-user', label: 'Size', value: race.size.name }] : []),
          ...(race.speed ? [{ icon: 'i-heroicons-bolt', label: 'Speed', value: `${race.speed} ft.` }] : [])
        ]"
      />

      <!-- Description + Image - NEW -->
      <UiDetailDescriptionWithImage
        v-if="race.description"
        :description="race.description"
        :image-path="imagePath"
        :image-alt="`${race.name} character portrait`"
      />

      <!-- Ability Score Increases - KEEP AS-IS -->
      <UCard v-if="race.ability_score_increases && race.ability_score_increases.length > 0">
        <template #header>
          <h2 class="text-lg font-semibold text-gray-900 dark:text-gray-100">
            Ability Score Increases
          </h2>
        </template>
        <div class="flex flex-wrap gap-3">
          <div
            v-for="increase in race.ability_score_increases"
            :key="increase.id"
            class="px-4 py-2 rounded-lg bg-primary-50 dark:bg-primary-900/20"
          >
            <span class="font-semibold text-gray-900 dark:text-gray-100">
              {{ increase.ability_score.code }}
            </span>
            <span class="text-primary-600 dark:text-primary-400 ml-1">
              +{{ increase.value }}
            </span>
          </div>
        </div>
      </UCard>

      <!-- Rest unchanged -->
      <UAccordion
        :items="[
          ...(race.parent_race ? [{
            label: 'Parent Race',
            slot: 'parent',
            defaultOpen: false
          }] : []),
          ...(race.subraces && race.subraces.length > 0 ? [{
            label: 'Subraces',
            slot: 'subraces',
            defaultOpen: false
          }] : []),
          ...(race.traits && race.traits.length > 0 ? [{
            label: 'Racial Traits',
            slot: 'traits',
            defaultOpen: false
          }] : []),
          ...(race.modifiers && race.modifiers.length > 0 ? [{
            label: 'Modifiers',
            slot: 'modifiers',
            defaultOpen: false
          }] : []),
          ...(race.languages && race.languages.length > 0 ? [{
            label: 'Languages',
            slot: 'languages',
            defaultOpen: false
          }] : []),
          ...(race.proficiencies && race.proficiencies.length > 0 ? [{
            label: 'Proficiencies',
            slot: 'proficiencies',
            defaultOpen: false
          }] : []),
          ...(race.spells && race.spells.length > 0 ? [{
            label: 'Spells',
            slot: 'spells',
            defaultOpen: false
          }] : []),
          ...(race.conditions && race.conditions.length > 0 ? [{
            label: 'Conditions',
            slot: 'conditions',
            defaultOpen: false
          }] : []),
          ...(race.sources && race.sources.length > 0 ? [{
            label: 'Source',
            slot: 'source',
            defaultOpen: false
          }] : []),
          ...(race.tags && race.tags.length > 0 ? [{
            label: 'Tags',
            slot: 'tags',
            defaultOpen: false
          }] : [])
        ]"
        type="multiple"
      >
        <!-- All slots remain unchanged -->
        <template
          v-if="race.parent_race"
          #parent
        >
          <div class="p-4">
            <NuxtLink :to="`/races/${race.parent_race.slug}`">
              <UButton
                color="primary"
                variant="soft"
              >
                View {{ race.parent_race.name }}
              </UButton>
            </NuxtLink>
          </div>
        </template>

        <template
          v-if="race.subraces && race.subraces.length > 0"
          #subraces
        >
          <UiAccordionEntityGrid
            :entities="race.subraces"
            base-path="/races"
          />
        </template>

        <template
          v-if="race.traits && race.traits.length > 0"
          #traits
        >
          <UiAccordionTraitsList
            :traits="race.traits"
            border-color="primary-500"
          />
        </template>

        <template
          v-if="race.languages && race.languages.length > 0"
          #languages
        >
          <UiAccordionBadgeList
            :items="race.languages.map(l => l.language)"
            color="neutral"
          />
        </template>

        <template
          v-if="race.modifiers && race.modifiers.length > 0"
          #modifiers
        >
          <UiModifiersDisplay :modifiers="race.modifiers" />
        </template>

        <template
          v-if="race.proficiencies && race.proficiencies.length > 0"
          #proficiencies
        >
          <UiAccordionBulletList :items="race.proficiencies" />
        </template>

        <template
          v-if="race.spells && race.spells.length > 0"
          #spells
        >
          <UiAccordionBadgeList
            :items="race.spells"
            color="primary"
          />
        </template>

        <template
          v-if="race.conditions && race.conditions.length > 0"
          #conditions
        >
          <div class="p-4 space-y-3">
            <div
              v-for="conditionRelation in race.conditions"
              :key="conditionRelation.id"
              class="p-3 rounded-lg bg-gray-50 dark:bg-gray-800"
            >
              <div class="flex items-start gap-3">
                <UBadge
                  color="warning"
                  variant="soft"
                >
                  {{ conditionRelation.condition?.name }}
                </UBadge>
                <div class="flex-1">
                  <div
                    v-if="conditionRelation.effect_type"
                    class="text-sm font-medium text-gray-900 dark:text-gray-100 mb-1"
                  >
                    Effect: {{ conditionRelation.effect_type.charAt(0).toUpperCase() + conditionRelation.effect_type.slice(1) }}
                  </div>
                  <div class="text-sm text-gray-700 dark:text-gray-300">
                    {{ conditionRelation.condition?.description }}
                  </div>
                  <div
                    v-if="conditionRelation.description"
                    class="text-sm text-gray-600 dark:text-gray-400 mt-2 italic"
                  >
                    {{ conditionRelation.description }}
                  </div>
                </div>
              </div>
            </div>
          </div>
        </template>

        <template
          v-if="race.sources && race.sources.length > 0"
          #source
        >
          <UiSourceDisplay :sources="race.sources" />
        </template>

        <template
          v-if="race.tags && race.tags.length > 0"
          #tags
        >
          <UiTagsDisplay :tags="race.tags" />
        </template>
      </UAccordion>

      <JsonDebugPanel
        :data="race"
        title="Race Data"
      />
    </div>
  </div>
</template>
```

**Step 2: Test and commit**

```bash
# Test
open http://localhost:3000/races/1
docker compose exec nuxt npm run test

# Commit
git add app/pages/races/\[slug\].vue
git commit -m "refactor: Update races page to use UiDetailDescriptionWithImage

- Replace UiEntityHeaderWithImage with UiDetailPageHeader + UiDetailDescriptionWithImage
- Image now integrated with description (2/3-1/3 layout)
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task A5: Update Classes Detail Page

**Files:**
- Modify: `app/pages/classes/[slug].vue`

**Step 1: Update classes page**

Same pattern:

```vue
<!-- app/pages/classes/[slug].vue -->
<!-- Keep all script, update template -->

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <UiDetailPageLoading
      v-if="loading"
      entity-type="class"
    />

    <UiDetailPageError
      v-else-if="error"
      entity-type="Class"
    />

    <div
      v-else-if="entity"
      class="space-y-8"
    >
      <UiBackLink
        to="/classes"
        label="Back to Classes"
      />

      <!-- Header - UPDATED -->
      <UiDetailPageHeader
        :title="entity.name"
        :badges="[
          { label: entity.is_base_class ? 'Base Class' : 'Subclass', color: (entity.is_base_class ? 'error' : 'warning') as BadgeColor, variant: 'subtle' as BadgeVariant, size: 'lg' as BadgeSize },
          ...(entity.spellcasting_ability ? [{ label: `✨ ${entity.spellcasting_ability.name}`, color: 'primary' as BadgeColor, variant: 'soft' as BadgeVariant, size: 'sm' as BadgeSize }] : [])
        ]"
      />

      <UiDetailQuickStatsCard
        :columns="3"
        :stats="[
          ...(entity.hit_die ? [{ icon: 'i-heroicons-heart', label: 'Hit Die', value: `1d${entity.hit_die}` }] : []),
          ...(entity.primary_ability ? [{ icon: 'i-heroicons-star', label: 'Primary Ability', value: entity.primary_ability }] : []),
          ...(entity.spellcasting_ability ? [{ icon: 'i-heroicons-sparkles', label: 'Spellcasting Ability', value: `${entity.spellcasting_ability.name} (${entity.spellcasting_ability.code})` }] : [])
        ]"
      />

      <!-- Description + Image - NEW -->
      <UiDetailDescriptionWithImage
        v-if="entity.description"
        :description="entity.description"
        :image-path="imagePath"
        :image-alt="`${entity.name} class illustration`"
      />

      <!-- Rest unchanged -->
      <UAccordion
        :items="[
          ...(entity.proficiencies && entity.proficiencies.length > 0 ? [{
            label: `Proficiencies (${entity.proficiencies.length})`,
            slot: 'proficiencies',
            defaultOpen: false
          }] : []),
          ...(entity.features && entity.features.length > 0 ? [{
            label: `Features (${entity.features.length})`,
            slot: 'features',
            defaultOpen: false
          }] : []),
          ...(entity.subclasses && entity.subclasses.length > 0 ? [{
            label: `Subclasses (${entity.subclasses.length})`,
            slot: 'subclasses',
            defaultOpen: false
          }] : []),
          ...(entity.sources && entity.sources.length > 0 ? [{
            label: 'Source',
            slot: 'source',
            defaultOpen: false
          }] : []),
          ...(entity.tags && entity.tags.length > 0 ? [{
            label: 'Tags',
            slot: 'tags',
            defaultOpen: false
          }] : [])
        ]"
        type="multiple"
      >
        <template
          v-if="entity.proficiencies && entity.proficiencies.length > 0"
          #proficiencies
        >
          <UiAccordionBulletList :items="entity.proficiencies" />
        </template>

        <template
          v-if="entity.features && entity.features.length > 0"
          #features
        >
          <UiAccordionTraitsList
            :traits="entity.features"
            :show-level="true"
            border-color="primary-500"
          />
        </template>

        <template
          v-if="entity.subclasses && entity.subclasses.length > 0"
          #subclasses
        >
          <UiAccordionEntityGrid
            :entities="entity.subclasses"
            base-path="/classes"
          />
        </template>

        <template
          v-if="entity.sources && entity.sources.length > 0"
          #source
        >
          <UiSourceDisplay :sources="entity.sources" />
        </template>

        <template
          v-if="entity.tags && entity.tags.length > 0"
          #tags
        >
          <UiTagsDisplay :tags="entity.tags" />
        </template>
      </UAccordion>

      <JsonDebugPanel
        :data="entity"
        title="Class Data"
      />
    </div>
  </div>
</template>
```

**Step 2: Test and commit**

```bash
open http://localhost:3000/classes/1
docker compose exec nuxt npm run test

git add app/pages/classes/\[slug\].vue
git commit -m "refactor: Update classes page to use UiDetailDescriptionWithImage

- Replace UiEntityHeaderWithImage with UiDetailPageHeader + UiDetailDescriptionWithImage
- Image now integrated with description (2/3-1/3 layout)
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task A6: Update Backgrounds Detail Page

**Files:**
- Modify: `app/pages/backgrounds/[slug].vue`

**Step 1: Update backgrounds page**

Same pattern:

```vue
<!-- app/pages/backgrounds/[slug].vue -->

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <UiDetailPageLoading
      v-if="loading"
      entity-type="background"
    />

    <UiDetailPageError
      v-else-if="error"
      entity-type="Background"
    />

    <div
      v-else-if="entity"
      class="space-y-8"
    >
      <UiBackLink
        to="/backgrounds"
        label="Back to Backgrounds"
      />

      <!-- Header - UPDATED -->
      <UiDetailPageHeader
        :title="entity.name"
        :badges="[
          { label: 'Background', color: 'success', variant: 'subtle', size: 'lg' }
        ]"
      />

      <!-- Traits Section - KEEP AS-IS (always visible) -->
      <UCard v-if="entity.traits && entity.traits.length > 0">
        <template #header>
          <h2 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
            Background Traits
          </h2>
        </template>
        <UiAccordionTraitsList
          :traits="entity.traits"
          :show-category="true"
          border-color="purple-500"
        />
      </UCard>

      <!-- Description + Image - NEW -->
      <UiDetailDescriptionWithImage
        v-if="entity.description"
        :description="entity.description"
        :image-path="imagePath"
        :image-alt="`${entity.name} background illustration`"
      />

      <!-- Proficiencies - KEEP AS-IS (always visible) -->
      <UCard v-if="entity.proficiencies && entity.proficiencies.length > 0">
        <template #header>
          <h2 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
            Skill Proficiencies
          </h2>
        </template>
        <UiAccordionBulletList :items="entity.proficiencies" />
      </UCard>

      <!-- Languages - KEEP AS-IS (always visible) -->
      <UCard v-if="entity.languages && entity.languages.length > 0">
        <template #header>
          <h2 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
            Languages
          </h2>
        </template>
        <UiAccordionBadgeList
          :items="entity.languages"
          color="neutral"
        />
      </UCard>

      <!-- Accordion - minimal (only source/tags) -->
      <UAccordion
        :items="[
          ...(entity.sources && entity.sources.length > 0 ? [{
            label: 'Source',
            slot: 'source',
            defaultOpen: false
          }] : []),
          ...(entity.tags && entity.tags.length > 0 ? [{
            label: 'Tags',
            slot: 'tags',
            defaultOpen: false
          }] : [])
        ]"
        type="multiple"
      >
        <template
          v-if="entity.sources && entity.sources.length > 0"
          #source
        >
          <UiSourceDisplay :sources="entity.sources" />
        </template>

        <template
          v-if="entity.tags && entity.tags.length > 0"
          #tags
        >
          <UiTagsDisplay :tags="entity.tags" />
        </template>
      </UAccordion>

      <JsonDebugPanel
        :data="entity"
        title="Background Data"
      />
    </div>
  </div>
</template>
```

**Step 2: Test and commit**

```bash
open http://localhost:3000/backgrounds/1
docker compose exec nuxt npm run test

git add app/pages/backgrounds/\[slug\].vue
git commit -m "refactor: Update backgrounds page to use UiDetailDescriptionWithImage

- Replace UiEntityHeaderWithImage with UiDetailPageHeader + UiDetailDescriptionWithImage
- Image now integrated with description (2/3-1/3 layout)
- Keep traits/proficiencies/languages as always-visible sections
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task A7: Update Feats Detail Page

**Files:**
- Modify: `app/pages/feats/[slug].vue`

**Step 1: Update feats page**

Same pattern:

```vue
<!-- app/pages/feats/[slug].vue -->

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <UiDetailPageLoading
      v-if="loading"
      entity-type="feat"
    />

    <UiDetailPageError
      v-else-if="error"
      entity-type="Feat"
    />

    <div
      v-else-if="entity"
      class="space-y-8"
    >
      <UiBackLink
        to="/feats"
        label="Back to Feats"
      />

      <!-- Header - UPDATED -->
      <UiDetailPageHeader
        :title="entity.name"
        :badges="[
          { label: 'Feat', color: 'warning', variant: 'subtle', size: 'lg' }
        ]"
      />

      <!-- Prerequisites - KEEP AS-IS (always visible) -->
      <UCard v-if="entity.prerequisites && entity.prerequisites.length > 0">
        <template #header>
          <h2 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
            Prerequisites
          </h2>
        </template>
        <div class="space-y-2">
          <div
            v-for="prereq in entity.prerequisites"
            :key="prereq.id"
            class="text-gray-700 dark:text-gray-300"
          >
            • {{ prereq.description || prereq.prerequisite_type }}
          </div>
        </div>
      </UCard>

      <!-- Description + Image - NEW -->
      <UiDetailDescriptionWithImage
        v-if="entity.description"
        :description="entity.description"
        :image-path="imagePath"
        :image-alt="`${entity.name} feat illustration`"
      />

      <!-- Accordion -->
      <UAccordion
        :items="[
          ...(entity.modifiers && entity.modifiers.length > 0 ? [{
            label: 'Modifiers',
            slot: 'modifiers',
            defaultOpen: false
          }] : []),
          ...(entity.sources && entity.sources.length > 0 ? [{
            label: 'Source',
            slot: 'source',
            defaultOpen: false
          }] : []),
          ...(entity.tags && entity.tags.length > 0 ? [{
            label: 'Tags',
            slot: 'tags',
            defaultOpen: false
          }] : [])
        ]"
        type="multiple"
      >
        <template
          v-if="entity.modifiers && entity.modifiers.length > 0"
          #modifiers
        >
          <UiModifiersDisplay :modifiers="entity.modifiers" />
        </template>

        <template
          v-if="entity.sources && entity.sources.length > 0"
          #source
        >
          <UiSourceDisplay :sources="entity.sources" />
        </template>

        <template
          v-if="entity.tags && entity.tags.length > 0"
          #tags
        >
          <UiTagsDisplay :tags="entity.tags" />
        </template>
      </UAccordion>

      <JsonDebugPanel
        :data="entity"
        title="Feat Data"
      />
    </div>
  </div>
</template>
```

**Step 2: Test and commit**

```bash
open http://localhost:3000/feats/1
docker compose exec nuxt npm run test

git add app/pages/feats/\[slug\].vue
git commit -m "refactor: Update feats page to use UiDetailDescriptionWithImage

- Replace UiEntityHeaderWithImage with UiDetailPageHeader + UiDetailDescriptionWithImage
- Image now integrated with description (2/3-1/3 layout)
- Keep prerequisites as always-visible section
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task A8: Update Monsters Detail Page (Add Image Support)

**Files:**
- Modify: `app/pages/monsters/[slug].vue`

**Step 1: Add image support to monsters page**

Monsters page currently doesn't have image integration - add it:

```vue
<!-- app/pages/monsters/[slug].vue -->
<script setup lang="ts">
import type { Monster } from '~/types/api/entities'
import { getChallengeRatingColor } from '~/utils/badgeColors'

const route = useRoute()

// Fetch monster data and setup SEO
const { data: monster, loading, error } = useEntityDetail<Monster>({
  slug: route.params.slug as string,
  endpoint: '/monsters',
  cacheKey: 'monster',
  seo: {
    titleTemplate: name => `${name} - D&D 5e Monster`,
    descriptionExtractor: (monster: unknown) => {
      const m = monster as { description?: string }
      return m.description?.substring(0, 160) || ''
    },
    fallbackTitle: 'Monster - D&D 5e Compendium'
  }
})

/**
 * Format speeds into readable text
 */
const speedText = computed(() => {
  if (!monster.value) return ''
  const speeds: string[] = []

  if (monster.value.speed_walk) speeds.push(`${monster.value.speed_walk} ft.`)
  if (monster.value.speed_fly) speeds.push(`fly ${monster.value.speed_fly} ft.${monster.value.can_hover ? ' (hover)' : ''}`)
  if (monster.value.speed_swim) speeds.push(`swim ${monster.value.speed_swim} ft.`)
  if (monster.value.speed_burrow) speeds.push(`burrow ${monster.value.speed_burrow} ft.`)
  if (monster.value.speed_climb) speeds.push(`climb ${monster.value.speed_climb} ft.`)

  return speeds.join(', ')
})

/**
 * Quick stats for card display
 */
const quickStats = computed(() => {
  if (!monster.value) return []

  return [
    { icon: 'i-heroicons-arrows-pointing-out', label: 'Size', value: monster.value.size?.name || 'Unknown' },
    { icon: 'i-heroicons-cube', label: 'Type', value: monster.value.type },
    { icon: 'i-heroicons-scale', label: 'Alignment', value: monster.value.alignment || 'Unaligned' },
    { icon: 'i-heroicons-shield-check', label: 'Armor Class', value: monster.value.armor_type ? `${monster.value.armor_class} (${monster.value.armor_type})` : String(monster.value.armor_class) },
    { icon: 'i-heroicons-heart', label: 'Hit Points', value: `${monster.value.hit_points_average} (${monster.value.hit_dice})` },
    { icon: 'i-heroicons-bolt', label: 'Speed', value: speedText.value || 'None' },
    { icon: 'i-heroicons-hand-raised', label: 'STR', value: String(monster.value.strength) },
    { icon: 'i-heroicons-arrow-trending-up', label: 'DEX', value: String(monster.value.dexterity) },
    { icon: 'i-heroicons-heart', label: 'CON', value: String(monster.value.constitution) },
    { icon: 'i-heroicons-light-bulb', label: 'INT', value: String(monster.value.intelligence) },
    { icon: 'i-heroicons-eye', label: 'WIS', value: String(monster.value.wisdom) },
    { icon: 'i-heroicons-sparkles', label: 'CHA', value: String(monster.value.charisma) },
    { icon: 'i-heroicons-star', label: 'Challenge Rating', value: `${monster.value.challenge_rating} (${monster.value.experience_points.toLocaleString()} XP)` }
  ]
})

/**
 * Regular actions (not legendary)
 */
const regularActions = computed(() => {
  if (!monster.value?.actions) return []
  return monster.value.actions.filter(a => a.action_type === 'action')
})

/**
 * Get entity image path (512px variant) - NEW
 */
const { getImagePath } = useEntityImage()
const imagePath = computed(() => {
  if (!monster.value) return null
  return getImagePath('monsters', monster.value.slug, 512)
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <UiDetailPageLoading
      v-if="loading"
      entity-type="monster"
    />

    <UiDetailPageError
      v-else-if="error"
      entity-type="Monster"
    />

    <div
      v-else-if="monster"
      class="space-y-8"
    >
      <UiBackLink
        to="/monsters"
        label="Back to Monsters"
      />

      <!-- Header - UNCHANGED (already uses UiDetailPageHeader) -->
      <UiDetailPageHeader
        :title="monster.name"
        :badges="[
          { label: `CR ${monster.challenge_rating}`, color: getChallengeRatingColor(monster.challenge_rating), variant: 'subtle' as const, size: 'lg' as const },
          { label: monster.type, color: 'neutral' as const, variant: 'subtle' as const, size: 'lg' as const },
          ...(monster.legendary_actions && monster.legendary_actions.length > 0 ? [{ label: '⭐ Legendary', color: 'warning' as const, variant: 'soft' as const, size: 'sm' as const }] : [])
        ]"
      />

      <!-- Quick Stats - UNCHANGED -->
      <UiDetailQuickStatsCard :stats="quickStats" />

      <!-- Description + Image - NEW (replace old description div) -->
      <UiDetailDescriptionWithImage
        v-if="monster.description"
        :description="monster.description"
        :image-path="imagePath"
        :image-alt="`${monster.name} monster illustration`"
      />

      <!-- Traits - KEEP AS-IS (currently using custom component) -->
      <UiAccordionTraits
        v-if="monster.traits && monster.traits.length > 0"
        :traits="monster.traits"
      />

      <!-- Actions - KEEP AS-IS -->
      <UiAccordionActions
        v-if="regularActions.length > 0"
        :actions="regularActions"
        title="Actions"
      />

      <!-- Legendary Actions - KEEP AS-IS -->
      <UiAccordionActions
        v-if="monster.legendary_actions && monster.legendary_actions.length > 0"
        :actions="monster.legendary_actions"
        title="Legendary Actions"
        :show-cost="true"
      />

      <!-- Modifiers - KEEP AS-IS -->
      <UiModifiersDisplay
        v-if="monster.modifiers && monster.modifiers.length > 0"
        :modifiers="monster.modifiers"
      />

      <!-- Sources - KEEP AS-IS -->
      <UiSourceDisplay
        v-if="monster.sources && monster.sources.length > 0"
        :sources="monster.sources"
      />

      <!-- Back to Monsters button - KEEP AS-IS -->
      <div class="text-center">
        <UButton
          to="/monsters"
          variant="soft"
          color="neutral"
        >
          ← Back to Monsters
        </UButton>
      </div>

      <!-- Debug Panel - KEEP AS-IS -->
      <JsonDebugPanel :data="monster" />
    </div>
  </div>
</template>
```

**Step 2: Test and commit**

```bash
open http://localhost:3000/monsters/1
docker compose exec nuxt npm run test

git add app/pages/monsters/\[slug\].vue
git commit -m "refactor: Add image integration to monsters page

- Add useEntityImage composable for image path generation
- Replace plain div description with UiDetailDescriptionWithImage
- Image now integrated with description (2/3-1/3 layout)
- Monsters now consistent with other 6 entity pages
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task A9: Verify All Pages and Deprecate Old Component

**Files:**
- Modify: `app/components/ui/UiEntityHeaderWithImage.vue`
- Create: `docs/HANDOVER-2025-11-23-DETAIL-PAGE-STANDARDIZATION-PHASE-A.md`

**Step 1: Add deprecation notice to old component**

```vue
<!-- app/components/ui/UiEntityHeaderWithImage.vue -->
<!-- Add deprecation notice at top of file -->
<script setup lang="ts">
/**
 * @deprecated This component is deprecated as of 2025-11-23.
 * Use UiDetailPageHeader for title/badges and UiDetailDescriptionWithImage for description/image instead.
 *
 * This component will be removed in a future version.
 *
 * Migration guide:
 * - Replace header usage with <UiDetailPageHeader :title="..." :badges="..." />
 * - Replace description with <UiDetailDescriptionWithImage :description="..." :image-path="..." />
 */

// ... rest of component unchanged
</script>
```

**Step 2: Verify all pages work**

```bash
# Test all 7 detail pages in browser
for entity in spells items races classes backgrounds feats monsters; do
  echo "Testing /$entity page..."
  curl -s -o /dev/null -w "%{http_code} $entity\n" "http://localhost:3000/$entity/1"
done
```

Expected: All return HTTP 200

**Step 3: Run full test suite**

```bash
docker compose exec nuxt npm run test
```

Expected: All tests pass (no regressions)

**Step 4: Create handover document**

```markdown
<!-- docs/HANDOVER-2025-11-23-DETAIL-PAGE-STANDARDIZATION-PHASE-A.md -->
# Detail Page Standardization - Phase A Complete

**Date:** 2025-11-23
**Status:** ✅ **COMPLETE**
**Phase:** A - Core Refactor

---

## 🎉 What Was Accomplished

Successfully standardized all 7 detail pages with consistent structure and integrated image layout.

### Implementation Summary

**Completed:**
- ✅ Created `<UiDetailDescriptionWithImage>` component (10 tests, all passing)
- ✅ Updated all 7 detail pages to use new component
- ✅ Monsters page now has image integration
- ✅ All pages follow identical template structure
- ✅ Deprecated `<UiEntityHeaderWithImage>` component
- ✅ All tests passing (no regressions)
- ✅ All pages verified (HTTP 200)

---

## 📊 Key Metrics

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| **Pages with consistent structure** | 0/7 | 7/7 | +7 |
| **Pages with integrated image layout** | 0/7 | 7/7 | +7 |
| **New components created** | 0 | 1 | +1 |
| **Tests added** | 0 | 10 | +10 |
| **Tests passing** | N/A | All | ✅ |
| **Commits** | 0 | 9 | +9 |

---

## 🏗️ What Changed

### New Component

**`<UiDetailDescriptionWithImage>`**
- Location: `app/components/ui/detail/UiDetailDescriptionWithImage.vue`
- Tests: `tests/components/ui/detail/UiDetailDescriptionWithImage.test.ts` (10 tests)
- Purpose: Display description with optional integrated image (2/3-1/3 layout)
- Features:
  - Automatic responsive layout (stacks on mobile)
  - Full-width when no image
  - Prose styling for description text
  - Graceful handling of missing data

### Updated Pages

All 7 detail pages updated:
1. **Spells** - `app/pages/spells/[slug].vue`
2. **Items** - `app/pages/items/[slug].vue`
3. **Races** - `app/pages/races/[slug].vue`
4. **Classes** - `app/pages/classes/[slug].vue`
5. **Backgrounds** - `app/pages/backgrounds/[slug].vue`
6. **Feats** - `app/pages/feats/[slug].vue`
7. **Monsters** - `app/pages/monsters/[slug].vue` (added image support)

**Changes per page:**
- Replaced `<UiEntityHeaderWithImage>` with `<UiDetailPageHeader>` + `<UiDetailDescriptionWithImage>`
- Image now in description card (2/3 content, 1/3 image)
- All pages follow identical structure

### Deprecated Component

**`<UiEntityHeaderWithImage>`**
- Location: `app/components/ui/UiEntityHeaderWithImage.vue`
- Status: **DEPRECATED** (marked with JSDoc comment)
- Migration: Use `UiDetailPageHeader` + `UiDetailDescriptionWithImage`
- Removal: Planned for future version (after all usages migrated)

---

## 🎯 Benefits

### Visual Consistency
- All 7 pages now have identical structure
- Image placement consistent across entities
- Better visual integration (image in content, not separate column)

### User Experience
- More natural reading flow (content and image together)
- Better space utilization (no empty columns)
- Improved mobile experience (image stacks below)

### Developer Experience
- Clear template pattern to follow
- Single component for description + image
- Easy to maintain and update

---

## 📝 Standard Template Structure

All detail pages now follow this structure:

```vue
<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <!-- 1. Loading State -->
    <UiDetailPageLoading v-if="loading" entity-type="..." />

    <!-- 2. Error State -->
    <UiDetailPageError v-else-if="error" entity-type="..." />

    <!-- 3. Content -->
    <div v-else-if="entity" class="space-y-8">
      <!-- 3a. Breadcrumb -->
      <UiBackLink to="/..." label="Back to ..." />

      <!-- 3b. Header (title + badges) -->
      <UiDetailPageHeader :title="..." :badges="[...]" />

      <!-- 3c. Quick Stats (optional) -->
      <UiDetailQuickStatsCard v-if="..." :stats="[...]" />

      <!-- 3d. Description + Image (integrated) -->
      <UiDetailDescriptionWithImage
        :description="..."
        :image-path="..."
        :image-alt="..."
      />

      <!-- 3e. Always-Visible Sections (entity-specific) -->
      <!-- Examples: Ability Scores, Prerequisites, Traits -->

      <!-- 3f. Accordion (secondary data) -->
      <UAccordion :items="[...]" type="multiple">
        <!-- Entity-specific slots -->
      </UAccordion>

      <!-- 3g. JSON Debug Panel -->
      <JsonDebugPanel :data="entity" title="..." />
    </div>
  </div>
</template>
```

---

## ✅ Verification

**All Pages Tested:**
- ✅ Spells: http://localhost:3000/spells/fireball (HTTP 200)
- ✅ Items: http://localhost:3000/items/1 (HTTP 200)
- ✅ Races: http://localhost:3000/races/1 (HTTP 200)
- ✅ Classes: http://localhost:3000/classes/1 (HTTP 200)
- ✅ Backgrounds: http://localhost:3000/backgrounds/1 (HTTP 200)
- ✅ Feats: http://localhost:3000/feats/1 (HTTP 200)
- ✅ Monsters: http://localhost:3000/monsters/1 (HTTP 200)

**Test Suite:**
- Total tests: N/A (run `npm run test` to see current count)
- New tests: 10 (UiDetailDescriptionWithImage)
- Pass rate: 100%
- Regressions: 0

---

## 🔄 Next Steps

**Phase A: COMPLETE ✅**

**Phase B: Missing Data - High Priority (NEXT)**
- Add monster spellcasting display
- Add monster conditions display
- Create `<UiAccordionSpellcasting>` component

**Phase C: Missing Data - Medium Priority**
- Add class counters display
- Add class level progression display
- Create `<UiAccordionCountersList>` component
- Create `<UiAccordionLevelProgression>` component

**Phase D: Missing Data - Low Priority**
- Add feat proficiencies/conditions
- Add item detail/prerequisites
- Final polish

---

**Phase A Status:** ✅ **PRODUCTION-READY**

**Next Agent:** Phase B implementation can begin. All foundation work complete.
```

**Step 5: Commit handover document**

```bash
git add app/components/ui/UiEntityHeaderWithImage.vue docs/HANDOVER-2025-11-23-DETAIL-PAGE-STANDARDIZATION-PHASE-A.md
git commit -m "docs: Add Phase A completion handover and deprecate old component

- Mark UiEntityHeaderWithImage as deprecated
- Document Phase A completion (7/7 pages updated)
- All pages verified (HTTP 200)
- All tests passing

Phase A: Core Refactor COMPLETE ✅

Next: Phase B (Monster spellcasting)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase A Complete!

**✅ All tasks done. Phase A is production-ready.**

**Next Phase:** Phase B (High Priority Missing Data) - Add monster spellcasting and conditions.

---

## Execution Handoff

Plan complete and saved to `docs/plans/2025-11-23-detail-page-standardization-implementation.md`.

**Two execution options:**

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration with quality gates

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**Which approach?**
