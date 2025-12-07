# Subclass Detail Page Enhancement Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Enhance subclass detail pages to display inherited parent class data (hit points, proficiencies, features, progression) alongside subclass-specific content.

**Architecture:** Modify the existing `classes/[slug].vue` page to detect subclasses and conditionally render inherited content from the `parent_class` object provided by the API. Create one new component for the parent class image overlay.

**Tech Stack:** Vue 3, TypeScript, NuxtUI 4, Vitest, TDD

---

## Task 1: Create UiClassParentImageOverlay Component (RED)

**Files:**
- Create: `tests/components/ui/class/UiClassParentImageOverlay.test.ts`

**Step 1: Write the failing tests**

```typescript
import { describe, it, expect } from 'vitest'
import { mount, RouterLinkStub } from '@vue/test-utils'
import UiClassParentImageOverlay from '~/components/ui/class/UiClassParentImageOverlay.vue'

describe('UiClassParentImageOverlay', () => {
  const mountOptions = {
    global: {
      stubs: {
        NuxtLink: RouterLinkStub,
        NuxtImg: {
          template: '<img :src="src" :alt="alt" class="nuxt-img" />',
          props: ['src', 'alt', 'loading']
        }
      }
    }
  }

  it('renders parent class image', () => {
    const wrapper = mount(UiClassParentImageOverlay, {
      props: {
        parentSlug: 'rogue',
        parentName: 'Rogue'
      },
      ...mountOptions
    })

    const img = wrapper.find('.nuxt-img')
    expect(img.exists()).toBe(true)
    expect(img.attributes('alt')).toContain('Rogue')
  })

  it('shows Base Class label', () => {
    const wrapper = mount(UiClassParentImageOverlay, {
      props: {
        parentSlug: 'rogue',
        parentName: 'Rogue'
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('Base Class')
  })

  it('links to parent class page', () => {
    const wrapper = mount(UiClassParentImageOverlay, {
      props: {
        parentSlug: 'rogue',
        parentName: 'Rogue'
      },
      ...mountOptions
    })

    const link = wrapper.findComponent(RouterLinkStub)
    expect(link.props('to')).toBe('/classes/rogue')
  })

  it('displays parent class name', () => {
    const wrapper = mount(UiClassParentImageOverlay, {
      props: {
        parentSlug: 'fighter',
        parentName: 'Fighter'
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('Fighter')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/ui/class/UiClassParentImageOverlay.test.ts`

Expected: FAIL with "Cannot find module" or similar (component doesn't exist)

---

## Task 2: Create UiClassParentImageOverlay Component (GREEN)

**Files:**
- Create: `app/components/ui/class/UiClassParentImageOverlay.vue`

**Step 3: Write minimal implementation**

```vue
<script setup lang="ts">
/**
 * UiClassParentImageOverlay
 *
 * Displays a small parent class image thumbnail as an overlay.
 * Used on subclass detail pages to show class heritage.
 *
 * @example
 * <UiClassParentImageOverlay
 *   parent-slug="rogue"
 *   parent-name="Rogue"
 * />
 */

interface Props {
  parentSlug: string
  parentName: string
}

defineProps<Props>()

const { getImagePath } = useEntityImage()
</script>

<template>
  <NuxtLink
    :to="`/classes/${parentSlug}`"
    class="block group"
    :title="`View ${parentName} base class`"
  >
    <div class="flex flex-col items-center gap-1">
      <div class="relative w-20 h-20 rounded-lg overflow-hidden border-2 border-class-300 dark:border-class-600 shadow-md transition-all group-hover:border-class-500 group-hover:shadow-lg group-hover:scale-105">
        <NuxtImg
          :src="getImagePath('classes', parentSlug, 128)"
          :alt="`${parentName} class illustration`"
          class="w-full h-full object-cover"
          loading="lazy"
        />
      </div>
      <div class="text-center">
        <div class="text-xs font-medium text-gray-600 dark:text-gray-400">
          Base Class
        </div>
        <div class="text-sm font-semibold text-class-600 dark:text-class-400 group-hover:underline">
          {{ parentName }}
        </div>
      </div>
    </div>
  </NuxtLink>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/ui/class/UiClassParentImageOverlay.test.ts`

Expected: PASS (4 tests)

**Step 5: Commit**

```bash
git add tests/components/ui/class/UiClassParentImageOverlay.test.ts app/components/ui/class/UiClassParentImageOverlay.vue
git commit -m "feat: Add UiClassParentImageOverlay component (TDD)"
```

---

## Task 3: Add Subclass Detection Logic to Page

**Files:**
- Modify: `app/pages/classes/[slug].vue` (lines 1-40, script section)

**Step 6: Add computed properties for subclass detection**

Add these computed properties after the existing `baseClassFeatures` computed (around line 38):

```typescript
/**
 * Determine if viewing a subclass (not a base class)
 */
const isSubclass = computed(() => entity.value && !entity.value.is_base_class)

/**
 * Get parent class data (only available for subclasses)
 */
const parentClass = computed(() => entity.value?.parent_class)

/**
 * Get parent class image path for overlay
 */
const parentClassImagePath = computed(() => {
  if (!parentClass.value?.slug) return null
  return getImagePath('classes', parentClass.value.slug, 128)
})

/**
 * Get base class features for progression table (from parent for subclasses)
 */
const progressionFeatures = computed(() => {
  if (isSubclass.value && parentClass.value?.features) {
    return parentClass.value.features.filter((f: { is_optional?: boolean }) => !f.is_optional)
  }
  return baseClassFeatures.value
})

/**
 * Get counters for progression table (from parent for subclasses)
 */
const progressionCounters = computed(() => {
  if (isSubclass.value && parentClass.value?.counters) {
    return parentClass.value.counters
  }
  return entity.value?.counters || []
})
```

**Step 7: Verify no TypeScript errors**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No errors related to new computed properties

**Step 8: Commit**

```bash
git add app/pages/classes/[slug].vue
git commit -m "feat: Add subclass detection computed properties"
```

---

## Task 4: Update Breadcrumb for Subclass Hierarchy

**Files:**
- Modify: `app/pages/classes/[slug].vue` (template section, around line 131-134)

**Step 9: Replace simple breadcrumb with hierarchical one**

Replace:
```vue
<!-- Breadcrumb Navigation -->
<UiBackLink
  to="/classes"
  label="Back to Classes"
/>
```

With:
```vue
<!-- Breadcrumb Navigation -->
<nav
  v-if="isSubclass && parentClass"
  class="flex items-center gap-2 text-sm text-gray-500 dark:text-gray-400"
>
  <NuxtLink
    to="/classes"
    class="hover:text-gray-700 dark:hover:text-gray-200"
  >
    Classes
  </NuxtLink>
  <UIcon
    name="i-heroicons-chevron-right"
    class="w-4 h-4"
  />
  <NuxtLink
    :to="`/classes/${parentClass.slug}`"
    class="hover:text-class-600 dark:hover:text-class-400"
  >
    {{ parentClass.name }}
  </NuxtLink>
  <UIcon
    name="i-heroicons-chevron-right"
    class="w-4 h-4"
  />
  <span class="text-gray-900 dark:text-gray-100 font-medium">
    {{ entity.name }}
  </span>
</nav>
<UiBackLink
  v-else
  to="/classes"
  label="Back to Classes"
/>
```

**Step 10: Verify page renders**

Run: `curl -s http://localhost:3000/classes/rogue-assassin -o /dev/null -w "%{http_code}"`

Expected: 200

**Step 11: Commit**

```bash
git add app/pages/classes/[slug].vue
git commit -m "feat: Add hierarchical breadcrumb for subclass pages"
```

---

## Task 5: Update Header Badge for Subclasses

**Files:**
- Modify: `app/pages/classes/[slug].vue` (template section, around line 137-143)

**Step 12: Replace Subclass badge with interactive parent link badge**

Replace the badges array in `UiDetailPageHeader`:
```vue
:badges="[
  { label: entity.is_base_class ? 'Base Class' : 'Subclass', color: (entity.is_base_class ? 'error' : 'warning') as BadgeColor, variant: 'subtle' as BadgeVariant, size: 'lg' as BadgeSize },
  ...(entity.spellcasting_ability ? [{ label: `✨ ${entity.spellcasting_ability.name}`, color: 'primary' as BadgeColor, variant: 'soft' as BadgeVariant, size: 'sm' as BadgeSize }] : [])
]"
```

With a conditional template approach. First, update the header section:

```vue
<!-- Header -->
<div class="space-y-2">
  <UiDetailPageHeader
    :title="entity.name"
    :badges="[
      ...(entity.is_base_class ? [{ label: 'Base Class', color: 'error' as BadgeColor, variant: 'subtle' as BadgeVariant, size: 'lg' as BadgeSize }] : []),
      ...(entity.spellcasting_ability ? [{ label: `✨ ${entity.spellcasting_ability.name}`, color: 'primary' as BadgeColor, variant: 'soft' as BadgeVariant, size: 'sm' as BadgeSize }] : []),
      ...(!entity.is_base_class && parentClass?.spellcasting_ability && !entity.spellcasting_ability ? [{ label: `✨ ${parentClass.spellcasting_ability.name}`, color: 'primary' as BadgeColor, variant: 'soft' as BadgeVariant, size: 'sm' as BadgeSize }] : [])
    ]"
  />

  <!-- Subclass Parent Link Badge -->
  <div
    v-if="isSubclass && parentClass"
    class="flex items-center gap-2"
  >
    <NuxtLink
      :to="`/classes/${parentClass.slug}`"
      class="inline-flex items-center gap-1.5 px-3 py-1.5 rounded-full bg-warning-100 dark:bg-warning-900/30 text-warning-700 dark:text-warning-300 hover:bg-warning-200 dark:hover:bg-warning-800/50 transition-colors text-sm font-medium"
    >
      <span>Subclass of</span>
      <span class="font-semibold">{{ parentClass.name }}</span>
      <UIcon
        name="i-heroicons-arrow-right"
        class="w-4 h-4"
      />
    </NuxtLink>
  </div>
</div>
```

**Step 13: Verify page renders correctly**

Run: `curl -s http://localhost:3000/classes/rogue-assassin -o /dev/null -w "%{http_code}"`

Expected: 200

**Step 14: Commit**

```bash
git add app/pages/classes/[slug].vue
git commit -m "feat: Add interactive 'Subclass of X' badge with link"
```

---

## Task 6: Add Dual Image Display for Subclasses

**Files:**
- Modify: `app/pages/classes/[slug].vue` (template section, around line 159-167)

**Step 15: Update image section for dual display**

Replace the image section:
```vue
<!-- Standalone Image - 1/3 width on large screens -->
<div class="lg:col-span-1">
  <UiDetailEntityImage
    v-if="imagePath"
    :image-path="imagePath"
    :image-alt="`${entity.name} class illustration`"
  />
</div>
```

With:
```vue
<!-- Image Section - 1/3 width on large screens -->
<div class="lg:col-span-1">
  <!-- Subclass: Dual image display with parent overlay -->
  <div
    v-if="isSubclass && parentClass"
    class="relative"
  >
    <UiDetailEntityImage
      v-if="imagePath"
      :image-path="imagePath"
      :image-alt="`${entity.name} subclass illustration`"
    />
    <!-- Parent Class Overlay -->
    <div class="absolute bottom-2 right-2">
      <UiClassParentImageOverlay
        :parent-slug="parentClass.slug"
        :parent-name="parentClass.name"
      />
    </div>
  </div>

  <!-- Base Class: Single image -->
  <UiDetailEntityImage
    v-else-if="imagePath"
    :image-path="imagePath"
    :image-alt="`${entity.name} class illustration`"
  />
</div>
```

**Step 16: Verify dual image renders**

Open browser: `http://localhost:3000/classes/rogue-assassin`

Expected: Assassin image with Rogue thumbnail overlay in bottom-right

**Step 17: Commit**

```bash
git add app/pages/classes/[slug].vue
git commit -m "feat: Add dual image display with parent class overlay"
```

---

## Task 7: Show Inherited Hit Points Card

**Files:**
- Modify: `app/pages/classes/[slug].vue` (template section, around line 182-187)

**Step 18: Update Hit Points Card to show for subclasses too**

Replace:
```vue
<!-- Hit Points Card -->
<UiClassHitPointsCard
  v-if="entity.hit_die && entity.is_base_class"
  :hit-die="Number(entity.hit_die)"
  :class-name="entity.name"
/>
```

With:
```vue
<!-- Hit Points Card (for base class or inherited) -->
<div
  v-if="entity.is_base_class && entity.hit_die"
>
  <UiClassHitPointsCard
    :hit-die="Number(entity.hit_die)"
    :class-name="entity.name"
  />
</div>

<!-- Inherited Hit Points Card (for subclasses) -->
<div
  v-else-if="isSubclass && parentClass?.hit_die"
  class="space-y-2"
>
  <div class="flex items-center gap-2">
    <h3 class="text-lg font-semibold text-gray-900 dark:text-gray-100">
      Hit Points
    </h3>
    <UBadge
      color="neutral"
      variant="subtle"
      size="xs"
    >
      Inherited from {{ parentClass.name }}
    </UBadge>
  </div>
  <UiClassHitPointsCard
    :hit-die="Number(parentClass.hit_die)"
    :class-name="parentClass.name"
  />
</div>
```

**Step 19: Verify Hit Points Card renders on subclass page**

Open browser: `http://localhost:3000/classes/rogue-assassin`

Expected: Hit Points Card showing "1d8 per rogue level" with "Inherited from Rogue" badge

**Step 20: Commit**

```bash
git add app/pages/classes/[slug].vue
git commit -m "feat: Show inherited Hit Points Card on subclass pages"
```

---

## Task 8: Add Subclass Features Section

**Files:**
- Modify: `app/pages/classes/[slug].vue` (template section, add after Hit Points Card)

**Step 21: Add prominent subclass features section**

Add after the Hit Points Card section:

```vue
<!-- Subclass Features (Primary Content for Subclasses) -->
<div
  v-if="isSubclass && entity.features && entity.features.length > 0"
  class="space-y-4"
>
  <h3 class="text-lg font-semibold text-gray-900 dark:text-gray-100 flex items-center gap-2">
    <UIcon
      name="i-heroicons-sparkles"
      class="w-5 h-5 text-class-500"
    />
    {{ entity.name }} Features ({{ entity.features.length }})
  </h3>
  <UiAccordionTraitsList
    :traits="entity.features"
    :show-level="true"
    border-color="class-500"
  />
</div>
```

**Step 22: Verify subclass features section renders**

Open browser: `http://localhost:3000/classes/rogue-assassin`

Expected: "Assassin Features (4)" section with Assassinate, Infiltration Expertise, etc.

**Step 23: Commit**

```bash
git add app/pages/classes/[slug].vue
git commit -m "feat: Add prominent subclass features section"
```

---

## Task 9: Show Inherited Progression Table

**Files:**
- Modify: `app/pages/classes/[slug].vue` (template section, around line 169-174)

**Step 24: Update Progression Table to use computed values**

Replace:
```vue
<!-- Class Progression Table -->
<UiClassProgressionTable
  v-if="entity.is_base_class && baseClassFeatures.length > 0"
  :features="baseClassFeatures"
  :counters="entity.counters || []"
/>
```

With:
```vue
<!-- Class Progression Table (base class or inherited) -->
<div
  v-if="progressionFeatures.length > 0"
  class="space-y-2"
>
  <div
    v-if="isSubclass && parentClass"
    class="flex items-center gap-2"
  >
    <h3 class="text-lg font-semibold text-gray-900 dark:text-gray-100 flex items-center gap-2">
      <UIcon
        name="i-heroicons-table-cells"
        class="w-5 h-5 text-gray-400"
      />
      Class Progression
    </h3>
    <UBadge
      color="neutral"
      variant="subtle"
      size="xs"
    >
      Inherited from {{ parentClass.name }}
    </UBadge>
  </div>
  <UiClassProgressionTable
    :features="progressionFeatures"
    :counters="progressionCounters"
  />
</div>
```

**Step 25: Verify progression table renders on subclass page**

Open browser: `http://localhost:3000/classes/rogue-assassin`

Expected: Full 20-level progression table showing Sneak Attack, Cunning Action, etc. with "Inherited from Rogue" header

**Step 26: Commit**

```bash
git add app/pages/classes/[slug].vue
git commit -m "feat: Show inherited progression table on subclass pages"
```

---

## Task 10: Add Inherited Accordion Sections

**Files:**
- Modify: `app/pages/classes/[slug].vue` (script and template sections)

**Step 27: Update accordionItems computed to include inherited sections for subclasses**

Replace the `accordionItems` computed with an expanded version that handles subclass inheritance:

```typescript
/**
 * Build accordion items with icons
 * For subclasses, includes inherited content from parent class
 */
const accordionItems = computed(() => {
  if (!entity.value) return []

  const items = []

  // For base classes: show own content
  // For subclasses: show inherited content from parent

  const counters = entity.value.is_base_class
    ? entity.value.counters
    : parentClass.value?.counters

  const traits = entity.value.is_base_class
    ? entity.value.traits
    : parentClass.value?.traits

  const levelProgression = entity.value.is_base_class
    ? entity.value.level_progression
    : parentClass.value?.level_progression

  const equipment = entity.value.is_base_class
    ? entity.value.equipment
    : parentClass.value?.equipment

  const proficiencies = entity.value.is_base_class
    ? entity.value.proficiencies
    : parentClass.value?.proficiencies

  // Base class features (for subclasses, show parent's features here)
  const baseFeatures = entity.value.is_base_class
    ? entity.value.features
    : parentClass.value?.features

  const inheritedLabel = isSubclass.value && parentClass.value
    ? ` (Inherited from ${parentClass.value.name})`
    : ''

  if (counters && counters.length > 0) {
    items.push({
      label: `Class Counters${inheritedLabel}`,
      slot: 'counters',
      defaultOpen: false,
      icon: 'i-heroicons-calculator'
    })
  }

  if (traits && traits.length > 0) {
    items.push({
      label: `Class Traits (${traits.length})${inheritedLabel}`,
      slot: 'traits',
      defaultOpen: false,
      icon: 'i-heroicons-shield-check'
    })
  }

  if (levelProgression && levelProgression.length > 0) {
    items.push({
      label: `Spell Slot Progression${inheritedLabel}`,
      slot: 'level-progression',
      defaultOpen: false,
      icon: 'i-heroicons-sparkles'
    })
  }

  if (equipment && equipment.length > 0) {
    items.push({
      label: `Starting Equipment${inheritedLabel}`,
      slot: 'equipment',
      defaultOpen: false,
      icon: 'i-heroicons-shopping-bag'
    })
  }

  if (proficiencies && proficiencies.length > 0) {
    items.push({
      label: `Proficiencies (${proficiencies.length})${inheritedLabel}`,
      slot: 'proficiencies',
      defaultOpen: false,
      icon: 'i-heroicons-academic-cap'
    })
  }

  // For base classes: show Features section
  // For subclasses: show Base Class Features (parent's features)
  if (baseFeatures && baseFeatures.length > 0) {
    items.push({
      label: isSubclass.value
        ? `Base Class Features (${baseFeatures.length})${inheritedLabel}`
        : `Features (${baseFeatures.length})`,
      slot: 'features',
      defaultOpen: false,
      icon: 'i-heroicons-star'
    })
  }

  // Source is always from the current entity
  if (entity.value.sources && entity.value.sources.length > 0) {
    items.push({
      label: 'Source',
      slot: 'source',
      defaultOpen: false,
      icon: 'i-heroicons-book-open'
    })
  }

  return items
})

/**
 * Get data for accordion slots (handles inheritance)
 */
const accordionData = computed(() => {
  if (!entity.value) return {}

  const isBase = entity.value.is_base_class
  const parent = parentClass.value

  return {
    counters: isBase ? entity.value.counters : parent?.counters,
    traits: isBase ? entity.value.traits : parent?.traits,
    levelProgression: isBase ? entity.value.level_progression : parent?.level_progression,
    equipment: isBase ? entity.value.equipment : parent?.equipment,
    proficiencies: isBase ? entity.value.proficiencies : parent?.proficiencies,
    features: isBase ? entity.value.features : parent?.features
  }
})
```

**Step 28: Update accordion template slots to use accordionData**

Replace the accordion template slots with:

```vue
<!-- Additional Details (Accordion) -->
<UAccordion
  :items="accordionItems"
  type="multiple"
>
  <!-- Counters Slot -->
  <template
    v-if="accordionData.counters && accordionData.counters.length > 0"
    #counters
  >
    <UiAccordionClassCounters :counters="accordionData.counters" />
  </template>

  <!-- Traits Slot -->
  <template
    v-if="accordionData.traits && accordionData.traits.length > 0"
    #traits
  >
    <UiAccordionTraitsList
      :traits="accordionData.traits"
      border-color="primary-500"
    />
  </template>

  <!-- Level Progression Slot -->
  <template
    v-if="accordionData.levelProgression && accordionData.levelProgression.length > 0"
    #level-progression
  >
    <UiAccordionLevelProgression :level-progression="accordionData.levelProgression" />
  </template>

  <!-- Equipment Slot -->
  <template
    v-if="accordionData.equipment && accordionData.equipment.length > 0"
    #equipment
  >
    <UiAccordionEquipmentList
      :equipment="accordionData.equipment"
      type="class"
    />
  </template>

  <!-- Proficiencies Slot -->
  <template
    v-if="accordionData.proficiencies && accordionData.proficiencies.length > 0"
    #proficiencies
  >
    <UiAccordionBulletList :items="accordionData.proficiencies" />
  </template>

  <!-- Features Slot -->
  <template
    v-if="accordionData.features && accordionData.features.length > 0"
    #features
  >
    <UiAccordionTraitsList
      :traits="accordionData.features"
      :show-level="true"
    />
  </template>

  <!-- Source Slot (always from current entity) -->
  <template
    v-if="entity.sources && entity.sources.length > 0"
    #source
  >
    <UiSourceDisplay :sources="entity.sources" />
  </template>
</UAccordion>
```

**Step 29: Verify accordion renders with inherited content**

Open browser: `http://localhost:3000/classes/rogue-assassin`

Expected: Accordion sections showing "(Inherited from Rogue)" in labels, with parent class data

**Step 30: Commit**

```bash
git add app/pages/classes/[slug].vue
git commit -m "feat: Show inherited accordion sections on subclass pages"
```

---

## Task 11: Hide Subclasses Section on Subclass Pages

**Files:**
- Modify: `app/pages/classes/[slug].vue` (template section)

**Step 31: Verify subclasses section is already hidden**

The existing code already has:
```vue
v-if="entity.is_base_class && entity.subclasses && entity.subclasses.length > 0"
```

This correctly hides the subclasses section on subclass pages. No changes needed.

**Step 32: Commit (if any cleanup was needed)**

Skip commit if no changes made.

---

## Task 12: Verify All Subclass Pages Work

**Files:**
- None (verification only)

**Step 33: Test multiple subclass pages**

Run these commands:

```bash
# Test Rogue subclasses
curl -s http://localhost:3000/classes/rogue-assassin -o /dev/null -w "rogue-assassin: %{http_code}\n"
curl -s http://localhost:3000/classes/rogue-arcane-trickster -o /dev/null -w "rogue-arcane-trickster: %{http_code}\n"
curl -s http://localhost:3000/classes/rogue-thief -o /dev/null -w "rogue-thief: %{http_code}\n"

# Test Fighter subclasses
curl -s http://localhost:3000/classes/fighter-champion -o /dev/null -w "fighter-champion: %{http_code}\n"
curl -s http://localhost:3000/classes/fighter-battle-master -o /dev/null -w "fighter-battle-master: %{http_code}\n"

# Test Wizard subclasses
curl -s http://localhost:3000/classes/wizard-school-of-evocation -o /dev/null -w "wizard-school-of-evocation: %{http_code}\n"

# Test base classes still work
curl -s http://localhost:3000/classes/rogue -o /dev/null -w "rogue (base): %{http_code}\n"
curl -s http://localhost:3000/classes/fighter -o /dev/null -w "fighter (base): %{http_code}\n"
```

Expected: All return 200

**Step 34: Run full test suite**

Run: `docker compose exec nuxt npm run test`

Expected: All tests pass

**Step 35: Run TypeScript check**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No errors

---

## Task 13: Update CHANGELOG and Final Commit

**Files:**
- Modify: `CHANGELOG.md`

**Step 36: Add CHANGELOG entry**

Add under "### Added":

```markdown
### Added
- Subclass detail pages now show inherited parent class data (2025-11-26)
  - Hierarchical breadcrumb: Classes > Rogue > Assassin
  - Interactive "Subclass of X" badge linking to parent class
  - Dual image display with parent class thumbnail overlay
  - Inherited Hit Points Card with "Inherited from X" label
  - Prominent subclass-specific features section
  - Inherited Class Progression Table
  - Inherited accordion sections (proficiencies, equipment, traits, features)
- New `UiClassParentImageOverlay` component for subclass pages
```

**Step 37: Final commit**

```bash
git add CHANGELOG.md
git commit -m "docs: Update CHANGELOG with subclass detail page enhancements"
```

---

## Summary

**Total Tasks:** 13
**New Files:** 2 (component + test)
**Modified Files:** 2 (page + CHANGELOG)
**Expected Commits:** 10-12

**Components Created:**
- `UiClassParentImageOverlay` - Parent class image thumbnail with link

**Features Added:**
1. Hierarchical breadcrumb for subclasses
2. Interactive "Subclass of X" badge
3. Dual image display with parent overlay
4. Inherited Hit Points Card
5. Subclass-specific features section
6. Inherited Class Progression Table
7. Inherited accordion sections

---

**End of Implementation Plan**
