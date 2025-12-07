# Class Detail Page Redesign

**Date:** 2025-11-28
**Status:** Design Complete, Ready for Implementation
**Author:** Claude + Human collaboration

## Overview

### Problem Statement

The current class detail page presents all data uniformly in accordions, which:
- Buries important information
- Doesn't match how D&D players think about classes
- Treats all data types the same (features, proficiencies, equipment)
- Doesn't account for different user needs (learning vs. reference vs. gameplay)

### Solution

A **three-view approach** that serves different user needs:

| View | Purpose | Best For |
|------|---------|----------|
| **Overview** | Quick scanning, class identity | "What is this class?" |
| **Journey** | Level-by-level progression story | "What's my path 1→20?" |
| **Reference** | Complete data, full details | "Give me everything" |

Each view is a **separate route**, making them bookmarkable and shareable.

### Target Users

1. **Learning** - New players discovering classes
2. **Research** - Comparing options, theory-crafting builds
3. **Gameplay** - Referencing mechanics during play

---

## Route Structure

```
/classes/wizard              → Overview (default)
/classes/wizard/journey      → Journey timeline view
/classes/wizard/reference    → Full reference view

/classes/fighter-champion              → Subclass Overview
/classes/fighter-champion/journey      → Subclass Journey (interleaved with parent)
/classes/fighter-champion/reference    → Subclass Reference
```

### File Structure

```
app/pages/classes/
├── index.vue                    # List page (existing)
└── [slug]/
    ├── index.vue                # Overview view (default)
    ├── journey.vue              # Journey timeline view
    └── reference.vue            # Reference view
```

---

## Shared Components (All Views)

### Header Section

Every view shares the same header for consistent class identity:

```
┌─────────────────────────────────────────────────────────────┐
│ ← Back to Classes                                           │
│                                                             │
│ ┌───────────────────────────────────┬───────────────────┐  │
│ │  WIZARD                           │   ┌───────────┐   │  │
│ │                                   │   │           │   │  │
│ │  [Base Class]  [d6 HP]  [INT]     │   │  [IMAGE]  │   │  │
│ │                                   │   │           │   │  │
│ │  "Supreme magic-users, defined    │   └───────────┘   │  │
│ │   and united by the spells..."    │                   │  │
│ └───────────────────────────────────┴───────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │   📊 Overview    📜 Journey    📋 Reference         │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Header Data Mapping

| Element | Data Source | Condition |
|---------|-------------|-----------|
| Name | `entity.name` | Always |
| Base Class badge | — | `entity.is_base_class === true` |
| Hit Die badge | `entity.hit_die` | Format as "d6", "d10" etc |
| Spellcasting badge | `entity.spellcasting_ability?.code` | Only if present |
| Description | `entity.description` | Truncate to ~200 chars |
| Image | `getImagePath('classes', slug, 512)` | Always |

### Subclass Header Adaptations

For subclasses, the header adds:
- Breadcrumb: `Classes → Fighter → Champion`
- "Subclass of [Parent]" link badge
- Parent class thumbnail overlay on image

### View Navigation Component

Links to the three routes, highlights current:

```vue
<template>
  <nav class="flex gap-1 p-1 bg-gray-100 dark:bg-gray-800 rounded-lg">
    <NuxtLink
      v-for="view in views"
      :key="view.path"
      :to="view.path"
      :class="[
        'px-4 py-2 rounded-md transition-colors flex items-center gap-2',
        isActive(view.path)
          ? 'bg-white dark:bg-gray-700 shadow-sm'
          : 'hover:bg-gray-200 dark:hover:bg-gray-700'
      ]"
    >
      <UIcon :name="view.icon" />
      {{ view.label }}
    </NuxtLink>
  </nav>
</template>
```

| View | Icon | Label | Path |
|------|------|-------|------|
| Overview | `i-heroicons-squares-2x2` | Overview | `/classes/{slug}` |
| Journey | `i-heroicons-map` | Journey | `/classes/{slug}/journey` |
| Reference | `i-heroicons-table-cells` | Reference | `/classes/{slug}/reference` |

---

## View 1: Overview

**Purpose:** Quick scanning, class identity at a glance

### Layout

```
┌─────────────────────────────────────────────────────────────┐
│ [Shared Header + Navigation]                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ┌─ COMBAT BASICS ───────────────────────────────────────┐  │
│ │  [HitPoints] [Saves] [Armor] [Weapons]                │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                             │
│ ┌─ SPELLCASTING ────────────────────────────────────────┐  │
│ │  (Only if entity.spellcasting_ability)                │  │
│ │  Ability, cantrips progression, spell slot summary    │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                             │
│ ┌─ CLASS RESOURCES ─────────────────────────────────────┐  │
│ │  (Only if entity.counters?.length > 0)                │  │
│ │  Ki, Rage, Sorcery Points, etc.                       │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                             │
│ ┌─ SUBCLASSES ──────────────────────────────────────────┐  │
│ │  (Only for base classes with subclasses)              │  │
│ │  Card gallery linking to each subclass                │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                             │
│ ┌─ KEY FEATURES ────────────────────────────────────────┐  │
│ │  Filtered list of core features (expandable)          │  │
│ │  [See All Features →] links to Reference              │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                             │
│ ┌─ COLLAPSED SECTIONS ──────────────────────────────────┐  │
│ │  [▶ Starting Equipment]                               │  │
│ │  [▶ Skill Choices]                                    │  │
│ │  [▶ Class Lore]                                       │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Components

| Component | Data Source | Notes |
|-----------|-------------|-------|
| `ClassCombatBasicsGrid` | Proficiencies, computed.hit_points | 2x2 grid on desktop |
| `ClassHitPointsCard` | `computed.hit_points` | Shows formula |
| `ClassSavingThrowsCard` | Proficiencies where `type === 'saving_throw'` | |
| `ClassArmorCard` | Proficiencies where `type === 'armor'` | |
| `ClassWeaponsCard` | Proficiencies where `type === 'weapon'` | |
| `ClassSpellcastingCard` | `spellcasting_ability`, `level_progression` | Conditional |
| `ClassResourcesCard` | `counters` | Conditional |
| `ClassSubclassGallery` | `subclasses` | Conditional, base class only |
| `ClassFeaturesPreview` | Filtered `features` | Core features only |

---

## View 2: Journey

**Purpose:** Level-by-level progression story, "what's my path?"

### Layout (Base Class)

```
┌─────────────────────────────────────────────────────────────┐
│ [Shared Header + Navigation]                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  YOUR PATH TO POWER                                         │
│                                                             │
│  ●━━━ LEVEL 1 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  │                                                          │
│  │  ┌────────────────────────────────────────────────────┐ │
│  │  │ ✦ Spellcasting                                     │ │
│  │  │   You learn 3 cantrips and 6 spells...            │ │
│  │  └────────────────────────────────────────────────────┘ │
│  │                                                          │
│  │  ┌─ SPELL SLOTS ─────────────────────────────────────┐  │
│  │  │  1st: ●● ○○                                       │  │
│  │  └───────────────────────────────────────────────────┘  │
│  │                                                          │
│  ●━━━ LEVEL 2 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ ★ MILESTONE   │
│  │                                                          │
│  │  ┌────────────────────────────────────────────────────┐ │
│  │  │ ★ Choose Your ARCANE TRADITION                     │ │
│  │  │   [Abjuration] [Conjuration] [Divination] [+more] │ │
│  │  └────────────────────────────────────────────────────┘ │
│  │                                                          │
│  ... continues through Level 20 ...                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Layout (Subclass - Interleaved)

```
┌─────────────────────────────────────────────────────────────┐
│ [Shared Header + Navigation]                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Show Wizard base features: [● ON] [○ OFF]           │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ●━━━ LEVEL 1 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  │                                                          │
│  │  ┌─ WIZARD ─────────────────────────────── [muted] ──┐  │
│  │  │ ✦ Spellcasting                                    │  │
│  │  └───────────────────────────────────────────────────┘  │
│  │                                                          │
│  ●━━━ LEVEL 2 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  │                                                          │
│  │  ┌─ WIZARD ─────────────────────────────── [muted] ──┐  │
│  │  │ ★ Arcane Tradition                                │  │
│  │  └───────────────────────────────────────────────────┘  │
│  │                                                          │
│  │  ┌─ SCHOOL OF ABJURATION ─────────── ★ YOUR SUBCLASS ┐  │
│  │  │ ✦ Abjuration Savant                               │  │
│  │  │ ✦ Arcane Ward                                     │  │
│  │  └───────────────────────────────────────────────────┘  │
│  │                                                          │
│  ... continues ...                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Milestone Detection

| Condition | Milestone Type | Visual |
|-----------|---------------|--------|
| `level === subclass_level` | Subclass Choice | Gold star, picker |
| `level in [4,8,12,16,19]` | ASI | Up arrow |
| New spell tier unlocked | Spell Milestone | Sparkle (casters) |
| `level === 20` | Capstone | Crown |

### Components

| Component | Purpose |
|-----------|---------|
| `ClassJourneyTimeline` | Container for timeline |
| `ClassJourneyLevel` | Single level node |
| `ClassFeatureCard` | Feature display within level |
| `ClassSpellSlotIndicator` | Visual spell slot display |
| `ClassMilestoneBadge` | Milestone marker |
| `ClassParentToggle` | Show/hide parent features (subclass) |

### Subclass Visual Differentiation

| Element | Parent Class Features | Subclass Features |
|---------|----------------------|-------------------|
| Border | `border-gray-200` | `border-class-500` |
| Background | `bg-gray-50` | `bg-class-50` |
| Label | "WIZARD" (muted) | "SCHOOL OF ABJURATION ★" |
| Opacity | 75% when toggle allows | 100% |

---

## View 3: Reference

**Purpose:** Complete data, full details, rules lookup

### Layout

```
┌─────────────────────────────────────────────────────────────┐
│ [Shared Header + Navigation]                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ┌─ FULL PROGRESSION TABLE ──────────────────────────────┐  │
│ │  All columns from computed.progression_table          │  │
│ │  Horizontally scrollable, sticky header               │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                             │
│ ┌─ COMPLETE FEATURES ───────────────────────────────────┐  │
│ │  All features, fully expanded                         │  │
│ │  Grouped by level                                     │  │
│ │  Choice options shown inline                          │  │
│ │  Random tables rendered                               │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                             │
│ ┌─ ACCORDION SECTIONS ──────────────────────────────────┐  │
│ │  [▼ Proficiencies] - Full breakdown by type           │  │
│ │  [▼ Starting Equipment] - With item links             │  │
│ │  [▼ Class Lore] - All traits                          │  │
│ │  [▼ Multiclassing] - Requirements & rules             │  │
│ │  [▼ Source] - Book citations                          │  │
│ └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Progression Table

Uses `computed.progression_table` from API which has dynamic columns per class type:

- **Wizard:** level, prof, features, cantrips, 1st-9th slots
- **Fighter:** level, prof, features (no spell columns)
- **Warlock:** level, prof, features, cantrips, slots, slot_level, invocations
- **Monk:** level, prof, features, ki_points, martial_arts_die, unarmored_movement

### Components

| Component | Purpose |
|-----------|---------|
| `ClassProgressionTable` | Full dynamic table |
| `ClassFeaturesComplete` | All features expanded |
| `ClassFeatureExpanded` | Single feature with full description |
| `ClassChoiceOptions` | Nested options (Fighting Styles, etc.) |
| `ClassRandomTable` | Embedded tables from feature data |
| `ClassMulticlassSection` | Multiclass rules (separate from core) |

---

## Feature Filtering Logic

Different views show different subsets of features:

### Filter Criteria

```typescript
interface FeatureFilters {
  excludeMulticlass: boolean    // is_multiclass_only
  excludeChoiceOptions: boolean // is_choice_option (show nested instead)
  excludeStarting: boolean      // "Starting X" features
  excludeAsi: boolean           // "Ability Score Improvement"
  excludePlaceholders: boolean  // "X Feature" with no real content
}
```

### Per-View Filtering

| View | Multiclass | Choice Options | Starting | ASI | Placeholders |
|------|------------|----------------|----------|-----|--------------|
| **Overview** | Exclude | Exclude (nest) | Exclude | Exclude | Exclude |
| **Journey** | Exclude | Exclude (nest) | Exclude | Show (milestone) | Show (marker) |
| **Reference** | Show (separate) | Show inline | Show | Show | Show |

### Implementation

```typescript
// composables/useClassFeatures.ts

const isStartingFeature = (f: Feature) =>
  f.feature_name.startsWith('Starting ') ||
  f.feature_name === 'Multiclass Features'

const isAsiFeature = (f: Feature) =>
  f.feature_name === 'Ability Score Improvement'

const isPlaceholderFeature = (f: Feature) =>
  f.feature_name.endsWith(' Feature') &&
  f.description.length < 200

const overviewFeatures = computed(() =>
  features.value.filter(f =>
    !f.is_multiclass_only &&
    !f.is_choice_option &&
    !isStartingFeature(f) &&
    !isAsiFeature(f) &&
    !isPlaceholderFeature(f)
  )
)

const journeyFeaturesAtLevel = (level: number) =>
  features.value.filter(f =>
    f.level === level &&
    !f.is_multiclass_only &&
    !f.is_choice_option &&
    !isStartingFeature(f)
  )

const referenceFeatures = computed(() => ({
  core: features.value.filter(f => !f.is_multiclass_only && !f.is_choice_option),
  multiclass: features.value.filter(f => f.is_multiclass_only),
  choiceOptions: features.value.filter(f => f.is_choice_option),
}))
```

---

## Data Fetching Strategy

All three views share the same class data. Use a composable with Nuxt's caching:

```typescript
// composables/useClassDetail.ts

export function useClassDetail(slug: Ref<string>) {
  const nuxtApp = useNuxtApp()

  const { data: entity, pending, error } = useAsyncData(
    `class-${slug.value}`,
    () => apiFetch<{ data: CharacterClass }>(`/classes/${slug.value}`),
    {
      watch: [slug],
      getCachedData: (key) => nuxtApp.payload.data[key]
    }
  )

  // Computed helpers
  const isSubclass = computed(() => !entity.value?.data.is_base_class)
  const parentClass = computed(() => entity.value?.data.parent_class)
  const features = computed(() => entity.value?.data.features ?? [])
  const counters = computed(() => entity.value?.data.counters ?? [])
  const proficiencies = computed(() => entity.value?.data.proficiencies ?? [])
  const subclasses = computed(() => entity.value?.data.subclasses ?? [])
  const progressionTable = computed(() => entity.value?.data.computed?.progression_table)
  const hitPoints = computed(() => entity.value?.data.computed?.hit_points)

  // Proficiency helpers
  const savingThrows = computed(() =>
    proficiencies.value.filter(p => p.proficiency_type === 'saving_throw')
  )
  const armorProficiencies = computed(() =>
    proficiencies.value.filter(p => p.proficiency_type === 'armor')
  )
  const weaponProficiencies = computed(() =>
    proficiencies.value.filter(p => p.proficiency_type === 'weapon')
  )
  const skillChoices = computed(() =>
    proficiencies.value.filter(p => p.proficiency_type === 'skill')
  )

  // For subclasses: get inherited data
  const inheritedData = computed(() => {
    if (isSubclass.value) {
      return entity.value?.data.inherited_data
    }
    return null
  })

  return {
    entity,
    pending,
    error,
    isSubclass,
    parentClass,
    features,
    counters,
    proficiencies,
    subclasses,
    progressionTable,
    hitPoints,
    savingThrows,
    armorProficiencies,
    weaponProficiencies,
    skillChoices,
    inheritedData,
  }
}
```

---

## Component Hierarchy

```
app/pages/classes/[slug]/index.vue (Overview)
├── components/class/DetailHeader.vue
├── components/class/ViewNavigation.vue
└── components/class/overview/View.vue
    ├── overview/CombatBasicsGrid.vue
    │   ├── overview/HitPointsCard.vue
    │   ├── overview/SavingThrowsCard.vue
    │   ├── overview/ArmorCard.vue
    │   └── overview/WeaponsCard.vue
    ├── overview/SpellcastingCard.vue (v-if caster)
    ├── overview/ResourcesCard.vue (v-if counters)
    ├── overview/SubclassGallery.vue (v-if base class)
    ├── overview/FeaturesPreview.vue
    └── UAccordion (equipment, skills, lore)

app/pages/classes/[slug]/journey.vue
├── components/class/DetailHeader.vue
├── components/class/ViewNavigation.vue
└── components/class/journey/View.vue
    ├── journey/ParentToggle.vue (subclass only)
    └── journey/Timeline.vue
        └── journey/LevelNode.vue (per level)
            ├── journey/FeatureCard.vue
            ├── journey/SpellSlotIndicator.vue
            └── journey/MilestoneBadge.vue

app/pages/classes/[slug]/reference.vue
├── components/class/DetailHeader.vue
├── components/class/ViewNavigation.vue
└── components/class/reference/View.vue
    ├── reference/ProgressionTable.vue
    ├── reference/FeaturesComplete.vue
    │   └── reference/FeatureExpanded.vue (per feature)
    │       ├── reference/ChoiceOptions.vue
    │       └── reference/RandomTable.vue
    ├── reference/MulticlassSection.vue
    └── UAccordion (proficiencies, equipment, lore, source)
```

---

## Mobile Considerations

| Component | Desktop | Mobile |
|-----------|---------|--------|
| View Navigation | Horizontal tabs | Horizontal scroll or dropdown |
| Combat Basics Grid | 2x2 or 4-col | Stacked cards |
| Progression Table | Full width | Horizontal scroll, sticky first col |
| Journey Timeline | Full width | Works well (vertical scroll) |
| Subclass Gallery | 4 columns | 2 columns |
| Feature Cards | Side-by-side | Full width stacked |

The Journey view is **inherently mobile-friendly** - vertical timelines are natural scroll patterns.

---

## Implementation Order (Recommended)

### Phase 1: Foundation
1. Create `useClassDetail` composable
2. Create route structure (`[slug]/index.vue`, `journey.vue`, `reference.vue`)
3. Create shared `DetailHeader` and `ViewNavigation` components

### Phase 2: Overview View
4. Create `CombatBasicsGrid` with sub-cards
5. Create `SpellcastingCard` (conditional)
6. Create `ResourcesCard` (conditional)
7. Create `SubclassGallery` (reuse existing cards)
8. Create `FeaturesPreview`
9. Add collapsed accordions

### Phase 3: Journey View
10. Create `Timeline` component
11. Create `LevelNode` with feature cards
12. Add milestone detection and badges
13. Add spell slot indicators
14. Implement subclass interleaving with toggle

### Phase 4: Reference View
15. Create `ProgressionTable` (dynamic columns)
16. Create `FeaturesComplete` with expanded features
17. Add choice options and random tables
18. Add multiclass section
19. Add accordion sections

### Phase 5: Polish
20. Mobile responsiveness pass
21. Dark mode verification
22. Loading and error states
23. Tests

---

## Success Criteria

- [ ] All three views render correctly for base classes
- [ ] All three views render correctly for subclasses
- [ ] Subclass journey shows interleaved features with toggle
- [ ] Feature filtering works correctly per view
- [ ] Progression table shows correct columns per class type
- [ ] Mobile responsive on all views
- [ ] Dark mode works correctly
- [ ] Data is cached across view navigation
- [ ] Routes are bookmarkable and shareable
- [ ] Tests cover key functionality

---

## Open Questions / Future Enhancements

1. **Spell list integration** - Could we show "Wizard Spell List" link in Spellcasting card?
2. **Subclass comparison** - Side-by-side subclass comparison view?
3. **Character builder integration** - Deep link to character builder with class pre-selected?
4. **Print stylesheet** - Reference view optimized for printing?
