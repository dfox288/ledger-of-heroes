# Classes Detail Page: Frontend Improvements Proposal

**Date**: 2025-11-26
**Author**: Claude (API Verification Audit)
**Status**: Proposed
**Related**: [CLASSES-DETAIL-PAGE-BACKEND-FIXES.md](./CLASSES-DETAIL-PAGE-BACKEND-FIXES.md)

---

## Executive Summary

This proposal outlines frontend improvements for the Classes detail page based on a comprehensive audit. Changes are organized into three categories:

1. **Immediate Fixes** - Can implement now without backend changes
2. **Layout Improvements** - UX enhancements for better information hierarchy
3. **Backend-Dependent Changes** - Require backend updates first

### Impact Summary

| Category | Items | Effort |
|----------|-------|--------|
| Immediate Fixes | 6 | Low |
| Layout Improvements | 8 | Medium |
| Backend-Dependent | 5 | Low (after backend) |

---

## Table of Contents

1. [Immediate Fixes](#1-immediate-fixes)
2. [Layout Improvements](#2-layout-improvements)
3. [Component Enhancements](#3-component-enhancements)
4. [Backend-Dependent Changes](#4-backend-dependent-changes)
5. [Implementation Plan](#5-implementation-plan)
6. [Mockups](#6-mockups)

---

## 1. Immediate Fixes

These can be implemented now using existing API data.

### 1.1 Handle `hit_die: 0` Gracefully

**File**: `app/pages/classes/[slug].vue`

**Problem**: Some subclasses return `hit_die: 0`, causing Hit Points Card to not render or show invalid data.

**Fix**: Fall back to parent class hit_die when current is 0.

```typescript
// Current
const hitPointsData = computed(() => {
  return entity.value?.computed?.hit_points ?? null
})

// Fixed
const hitPointsData = computed(() => {
  if (!entity.value) return null

  // Use computed hit_points if available
  if (entity.value.computed?.hit_points) {
    return entity.value.computed.hit_points
  }

  // Fallback: construct from parent class for subclasses with hit_die: 0
  if (!entity.value.is_base_class && entity.value.hit_die === 0) {
    const parentHitDie = entity.value.parent_class?.hit_die
    if (parentHitDie) {
      return {
        hit_die: `d${parentHitDie}`,
        hit_die_numeric: parentHitDie,
        first_level: {
          value: parentHitDie,
          description: `${parentHitDie} + your Constitution modifier`
        },
        higher_levels: {
          roll: `1d${parentHitDie}`,
          average: Math.floor(parentHitDie / 2) + 1,
          description: `1d${parentHitDie} (or ${Math.floor(parentHitDie / 2) + 1}) + your Constitution modifier per ${entity.value.parent_class?.name?.toLowerCase()} level after 1st`
        }
      }
    }
  }

  return null
})
```

**Acceptance Criteria**:
- [ ] Death Domain Cleric shows d8 hit points
- [ ] Oathbreaker Paladin shows d10 hit points
- [ ] Hit Points Card renders for all subclasses

---

### 1.2 Filter Multiclass Features from Progression Table Display

**File**: `app/components/ui/class/UiClassProgressionTable.vue`

**Problem**: Progression table shows "Multiclass Wizard, Multiclass Features" cluttering Level 1.

**Fix**: Filter out multiclass-related features from display.

```typescript
// Add computed to filter features
const filterProgressionFeatures = (featuresString: string): string => {
  if (!featuresString || featuresString === '—') return featuresString

  return featuresString
    .split(', ')
    .filter(f => !f.toLowerCase().includes('multiclass'))
    .join(', ') || '—'
}
```

```vue
<!-- In template -->
<td v-for="col in progressionTable.columns" :key="col.key">
  <template v-if="col.key === 'features'">
    {{ filterProgressionFeatures(row[col.key]) }}
  </template>
  <template v-else>
    {{ row[col.key] }}
  </template>
</td>
```

**Acceptance Criteria**:
- [ ] Level 1 Wizard shows "Spellcasting" not "Starting Wizard, Multiclass Wizard..."
- [ ] Multiclass features still visible in Features accordion

---

### 1.3 Collapse Fighting Style Options in Progression Table

**File**: `app/components/ui/class/UiClassProgressionTable.vue`

**Problem**: Level 1 Fighter shows all 6 fighting style options as separate features.

**Fix**: Detect and collapse choice patterns.

```typescript
const filterProgressionFeatures = (featuresString: string): string => {
  if (!featuresString || featuresString === '—') return featuresString

  let features = featuresString.split(', ')

  // Filter multiclass
  features = features.filter(f => !f.toLowerCase().includes('multiclass'))

  // Collapse Fighting Style options
  const fightingStyleOptions = features.filter(f => f.startsWith('Fighting Style:'))
  if (fightingStyleOptions.length > 1) {
    features = features.filter(f => !f.startsWith('Fighting Style:'))
    // Keep the parent "Fighting Style" if present, otherwise add indicator
    if (!features.includes('Fighting Style')) {
      features.push('Fighting Style')
    }
  }

  // Collapse Totem options (Bear, Eagle, Wolf at same level)
  const totemOptions = ['Bear', 'Eagle', 'Wolf', 'Elk', 'Tiger']
  const hasTotemOptions = totemOptions.some(t =>
    features.some(f => f.includes(`${t} (`))
  )
  if (hasTotemOptions) {
    features = features.filter(f =>
      !totemOptions.some(t => f.includes(`${t} (`))
    )
  }

  return features.join(', ') || '—'
}
```

**Acceptance Criteria**:
- [ ] Level 1 Fighter shows "Second Wind, Fighting Style"
- [ ] Level 3 Totem Warrior shows "Primal Path, Spirit Seeker, Totem Spirit"
- [ ] Individual options (Archery, Defense, Bear, Eagle) not listed

---

### 1.4 Use First Feature Description for Subclass Overview

**File**: `app/pages/classes/[slug].vue`

**Problem**: Subclass descriptions show "Subclass of Wizard" placeholder.

**Fix**: Extract meaningful description from first feature.

```typescript
/**
 * Get effective description for display
 * For subclasses with placeholder descriptions, use first feature's intro
 */
const effectiveDescription = computed(() => {
  if (!entity.value) return ''

  const desc = entity.value.description

  // Check if it's a placeholder
  const isPlaceholder = desc?.startsWith('Subclass of ')

  if (isPlaceholder && entity.value.features?.length > 0) {
    const firstFeature = entity.value.features[0]
    if (firstFeature?.description) {
      // Extract first 1-2 paragraphs (before "Source:" reference)
      const fullDesc = firstFeature.description
      const sourceIndex = fullDesc.indexOf('\n\nSource:')
      const cleanDesc = sourceIndex > 0
        ? fullDesc.substring(0, sourceIndex)
        : fullDesc

      // Take first 2 paragraphs max
      const paragraphs = cleanDesc.split('\n\t').slice(0, 2)
      return paragraphs.join('\n\n')
    }
  }

  return desc || ''
})
```

```vue
<!-- Use effectiveDescription instead of entity.description -->
<UiDetailDescriptionCard
  v-if="effectiveDescription"
  :description="effectiveDescription"
/>
```

**Acceptance Criteria**:
- [ ] School of Abjuration shows "The School of Abjuration emphasizes magic that blocks..."
- [ ] Base classes still show their full description
- [ ] No "Source:" references in displayed description

---

### 1.5 Show Accurate Feature Count

**File**: `app/components/ui/class/UiClassSubclassCards.vue`

**Problem**: Feature count includes choice options, showing "15 features" instead of "7 features".

**Fix**: Filter choice options from count.

```typescript
/**
 * Get meaningful feature count (excludes choice options)
 */
const getFeatureCount = (subclass: Subclass): number => {
  if (!subclass.features) return 0

  // Patterns that indicate choice options (not primary features)
  const choicePatterns = [
    /^Fighting Style: /,
    /^Bear \(/,
    /^Eagle \(/,
    /^Wolf \(/,
    /^Elk \(/,
    /^Tiger \(/,
    /^Aspect of the Bear/,
    /^Aspect of the Eagle/,
    /^Aspect of the Wolf/,
  ]

  return subclass.features.filter(f => {
    const name = f.feature_name || ''
    return !choicePatterns.some(pattern => pattern.test(name))
  }).length
}

const getFeatureCountText = (subclass: Subclass): string => {
  const count = getFeatureCount(subclass)
  return count === 1 ? '1 feature' : `${count} features`
}
```

**Acceptance Criteria**:
- [ ] Champion shows "6 features" not "13 features"
- [ ] Totem Warrior shows "7 features" not "15 features"
- [ ] Base feature counts unchanged

---

### 1.6 Fix Hit Points Description for Subclasses

**File**: `app/components/ui/class/UiClassHitPointsCard.vue`

**Problem**: Shows "per school of abjuration level" instead of "per wizard level".

**Fix**: Accept parent class name as prop and use it in description.

```typescript
// Update props
interface Props {
  hitPoints: ClassHitPoints
  parentClassName?: string  // NEW: for subclasses
}

const props = defineProps<Props>()

// Computed to fix description
const higherLevelsDescription = computed(() => {
  const desc = props.hitPoints.higher_levels.description

  if (props.parentClassName) {
    // Replace subclass name pattern with parent class name
    // "per school of abjuration level" -> "per wizard level"
    return desc.replace(
      /per .+ level after 1st$/,
      `per ${props.parentClassName.toLowerCase()} level after 1st`
    )
  }

  return desc
})
```

```vue
<!-- In [slug].vue -->
<UiClassHitPointsCard
  v-if="hitPointsData"
  :hit-points="hitPointsData"
  :parent-class-name="isSubclass ? parentClass?.name : undefined"
/>
```

**Acceptance Criteria**:
- [ ] School of Abjuration shows "per wizard level after 1st"
- [ ] Base classes unchanged

---

## 2. Layout Improvements

### 2.1 Restructure Quick Stats Card

**Current Issues**:
- Only 1-3 stats shown (sparse)
- Missing useful information like saving throws, armor proficiencies

**Proposed Enhancement**:

```typescript
// Enhanced stats for base classes
const quickStats = computed(() => {
  if (!entity.value) return []

  const stats = []

  // Hit Die (always present)
  if (entity.value.hit_die) {
    stats.push({
      icon: 'i-heroicons-heart',
      label: 'Hit Die',
      value: `d${entity.value.hit_die}`
    })
  }

  // Spellcasting Ability
  const spellAbility = entity.value.spellcasting_ability
    || (isSubclass.value && parentClass.value?.spellcasting_ability)
  if (spellAbility) {
    stats.push({
      icon: 'i-heroicons-sparkles',
      label: 'Spellcasting',
      value: spellAbility.name
    })
  }

  // Subclass Level (base classes only)
  if (entity.value.is_base_class && entity.value.subclasses?.length) {
    const subclassFeature = entity.value.features?.find(f =>
      f.feature_name?.includes('Arcane Tradition') ||
      f.feature_name?.includes('Martial Archetype') ||
      f.feature_name?.includes('Divine Domain') ||
      f.feature_name?.includes('Sacred Oath') ||
      f.feature_name?.includes('Primal Path')
    )
    const subclassLevel = subclassFeature?.level ||
      (entity.value.features?.find(f => !f.feature_name?.includes('Starting'))?.level)

    if (subclassLevel) {
      stats.push({
        icon: 'i-heroicons-users',
        label: 'Subclass at',
        value: `Level ${subclassLevel}`
      })
    }
  }

  // Saving Throws (extract from proficiencies)
  const savingThrows = getSavingThrowProficiencies()
  if (savingThrows.length) {
    stats.push({
      icon: 'i-heroicons-shield-check',
      label: 'Saving Throws',
      value: savingThrows.join(', ')
    })
  }

  // Feature Count
  const featureCount = getFilteredFeatureCount()
  stats.push({
    icon: 'i-heroicons-star',
    label: 'Features',
    value: `${featureCount} features`
  })

  return stats
})
```

**Acceptance Criteria**:
- [ ] Quick Stats shows 4-6 items for base classes
- [ ] Subclass level shown for base classes
- [ ] Saving throws extracted from proficiencies

---

### 2.2 Group Features by Level

**File**: New component `app/components/ui/class/UiClassFeaturesByLevel.vue`

**Current Issues**:
- Features shown as flat list
- Hard to find features at specific levels
- Choice options interleaved with primary features

**Proposed Component**:

```vue
<script setup lang="ts">
interface Feature {
  id: number
  level: number
  feature_name: string
  description: string
  is_optional?: boolean
}

interface Props {
  features: Feature[]
  showChoiceOptions?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  showChoiceOptions: false
})

// Group features by level
const featuresByLevel = computed(() => {
  const grouped = new Map<number, Feature[]>()

  props.features
    .filter(f => props.showChoiceOptions || !isChoiceOption(f))
    .forEach(feature => {
      const level = feature.level
      if (!grouped.has(level)) {
        grouped.set(level, [])
      }
      grouped.get(level)!.push(feature)
    })

  return Array.from(grouped.entries())
    .sort(([a], [b]) => a - b)
    .map(([level, features]) => ({ level, features }))
})

const isChoiceOption = (feature: Feature): boolean => {
  const name = feature.feature_name || ''
  const patterns = [
    /^Fighting Style: /,
    /^Bear \(/,
    /^Eagle \(/,
    /^Wolf \(/,
  ]
  return patterns.some(p => p.test(name))
}
</script>

<template>
  <div class="space-y-6">
    <div
      v-for="group in featuresByLevel"
      :key="group.level"
      class="space-y-3"
    >
      <!-- Level Header -->
      <div class="flex items-center gap-3">
        <div class="flex items-center justify-center w-10 h-10 rounded-full bg-class-100 dark:bg-class-900/30">
          <span class="text-sm font-bold text-class-700 dark:text-class-300">
            {{ group.level }}
          </span>
        </div>
        <h4 class="text-sm font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wide">
          Level {{ group.level }}
        </h4>
        <div class="flex-1 h-px bg-gray-200 dark:bg-gray-700" />
      </div>

      <!-- Features at this level -->
      <div class="ml-5 pl-8 border-l-2 border-class-200 dark:border-class-800 space-y-4">
        <UiAccordion
          :items="group.features.map(f => ({
            label: f.feature_name,
            slot: `feature-${f.id}`,
            defaultOpen: false
          }))"
          type="multiple"
        >
          <template
            v-for="feature in group.features"
            :key="feature.id"
            #[`feature-${feature.id}`]
          >
            <div class="p-4 text-sm text-gray-700 dark:text-gray-300 whitespace-pre-line">
              {{ feature.description }}
            </div>
          </template>
        </UiAccordion>
      </div>
    </div>
  </div>
</template>
```

**Acceptance Criteria**:
- [ ] Features grouped with visual level indicators
- [ ] Collapsible feature descriptions
- [ ] Choice options hidden by default

---

### 2.3 Improve Subclass Cards

**File**: `app/components/ui/class/UiClassSubclassCards.vue`

**Current Issues**:
- No subclass entry level shown
- Generic "View Details" link
- No visual distinction between PHB vs expansion subclasses

**Proposed Enhancements**:

```vue
<!-- Enhanced card content -->
<div class="relative z-10 space-y-3">
  <!-- Subclass Name -->
  <h4 class="font-semibold text-gray-900 dark:text-gray-100">
    {{ subclass.name }}
  </h4>

  <!-- Meta Info Row -->
  <div class="flex items-center gap-2 flex-wrap text-sm">
    <!-- Source Badge -->
    <UBadge
      v-if="getSourceAbbreviation(subclass)"
      color="class"
      variant="subtle"
      size="xs"
    >
      {{ getSourceAbbreviation(subclass) }}
    </UBadge>

    <!-- Entry Level Badge (NEW) -->
    <UBadge
      v-if="getEntryLevel(subclass)"
      color="info"
      variant="soft"
      size="xs"
    >
      Level {{ getEntryLevel(subclass) }}
    </UBadge>

    <!-- Feature Count -->
    <span class="text-gray-500 dark:text-gray-400">
      {{ getFeatureCountText(subclass) }}
    </span>
  </div>

  <!-- Brief Description Preview (NEW) -->
  <p
    v-if="getDescriptionPreview(subclass)"
    class="text-xs text-gray-500 dark:text-gray-400 line-clamp-2"
  >
    {{ getDescriptionPreview(subclass) }}
  </p>
</div>
```

```typescript
/**
 * Get subclass entry level (first feature's level)
 */
const getEntryLevel = (subclass: Subclass): number | null => {
  if (!subclass.features?.length) return null
  return Math.min(...subclass.features.map(f => f.level))
}

/**
 * Get brief description preview
 */
const getDescriptionPreview = (subclass: Subclass): string | null => {
  // Try to extract from first feature if description is placeholder
  if (subclass.description?.startsWith('Subclass of ')) {
    const firstFeature = subclass.features?.[0]
    if (firstFeature?.description) {
      // Take first sentence
      const firstSentence = firstFeature.description.split('.')[0]
      return firstSentence.length < 100 ? firstSentence + '.' : null
    }
  }
  return null
}
```

**Acceptance Criteria**:
- [ ] Entry level badge shown (e.g., "Level 2" for Wizard subclasses)
- [ ] Brief description preview for subclasses
- [ ] Improved visual hierarchy

---

### 2.4 Collapse Inherited Content on Subclass Pages

**File**: `app/pages/classes/[slug].vue`

**Current Issues**:
- Full 20-level progression table shown for subclasses
- Inherited proficiencies/equipment take up space
- Subclass-specific features buried below inherited content

**Proposed Changes**:

```typescript
// Reorder accordion items for subclasses
const accordionItems = computed(() => {
  if (!entity.value) return []

  const items = []
  const isBase = entity.value.is_base_class

  // For subclasses, prioritize subclass-specific content
  if (!isBase) {
    // Subclass counters first (if any)
    if (entity.value.counters?.length) {
      items.push({
        label: `${entity.value.name} Resources`,
        slot: 'subclass-counters',
        defaultOpen: true,  // Open by default
        icon: 'i-heroicons-calculator'
      })
    }
  }

  // ... existing items ...

  // For subclasses, group inherited content
  if (!isBase && parentClass.value) {
    items.push({
      label: `Inherited from ${parentClass.value.name}`,
      slot: 'inherited',
      defaultOpen: false,  // Collapsed by default
      icon: 'i-heroicons-arrow-down-tray'
    })
  }

  return items
})
```

**Acceptance Criteria**:
- [ ] Subclass features shown prominently (already done)
- [ ] Progression table collapsed by default on subclass pages
- [ ] Inherited content grouped in single collapsible section

---

### 2.5 Add Source Display to Subclass Cards

**File**: `app/components/ui/class/UiClassSubclassCards.vue`

**Current Issues**:
- Source abbreviation shown but not always meaningful
- No way to distinguish PHB subclasses from expansion content

**Proposed Enhancement**:

Add visual indicator for source book category:

```typescript
const getSourceCategory = (subclass: Subclass): 'core' | 'expansion' | 'setting' => {
  const source = subclass.sources?.[0]
  if (!source) return 'core'

  const coreBooks = ['PHB', 'DMG', 'MM']
  const expansionBooks = ['XGE', 'TCE', 'FTD', 'SCC']

  const abbr = source.abbreviation || ''

  if (coreBooks.includes(abbr)) return 'core'
  if (expansionBooks.includes(abbr)) return 'expansion'
  return 'setting'
}
```

```vue
<!-- Color-coded source badge -->
<UBadge
  v-if="getSourceAbbreviation(subclass)"
  :color="getSourceCategory(subclass) === 'core' ? 'success' :
          getSourceCategory(subclass) === 'expansion' ? 'info' : 'warning'"
  variant="subtle"
  size="xs"
>
  {{ getSourceAbbreviation(subclass) }}
</UBadge>
```

**Acceptance Criteria**:
- [ ] Core (PHB/DMG) subclasses have green badges
- [ ] Expansion (XGE/TCE) subclasses have blue badges
- [ ] Setting-specific subclasses have amber badges

---

### 2.6 Consolidate Counter Display

**File**: `app/components/ui/accordion/UiAccordionClassCounters.vue`

**Current Display**:
```
| Level | Counter | Value | Reset |
|-------|---------|-------|-------|
| 3     | Superiority Die | 4 | Short Rest |
| 7     | Superiority Die | 5 | Short Rest |
| 15    | Superiority Die | 6 | Short Rest |
```

**Proposed Display**:
```
┌─────────────────────────────────────────────────┐
│ Superiority Die                    Short Rest   │
│ ───────────────────────────────────────────────│
│  L3: 4  →  L7: 5  →  L15: 6                    │
└─────────────────────────────────────────────────┘
```

```vue
<script setup lang="ts">
// Group counters by name
const consolidatedCounters = computed(() => {
  const grouped = new Map<string, typeof props.counters>()

  props.counters.forEach(counter => {
    if (!grouped.has(counter.counter_name)) {
      grouped.set(counter.counter_name, [])
    }
    grouped.get(counter.counter_name)!.push(counter)
  })

  return Array.from(grouped.entries()).map(([name, counters]) => ({
    name,
    reset_timing: counters[0].reset_timing,
    progression: counters
      .sort((a, b) => a.level - b.level)
      .map(c => ({ level: c.level, value: c.counter_value }))
  }))
})
</script>

<template>
  <div class="p-4 space-y-4">
    <div
      v-for="counter in consolidatedCounters"
      :key="counter.name"
      class="p-4 rounded-lg bg-gray-50 dark:bg-gray-800/50"
    >
      <div class="flex items-center justify-between mb-2">
        <h4 class="font-semibold text-gray-900 dark:text-gray-100">
          {{ counter.name }}
        </h4>
        <UBadge :color="getResetTimingColor(counter.reset_timing)" variant="soft" size="sm">
          {{ counter.reset_timing }}
        </UBadge>
      </div>

      <div class="flex items-center gap-2 text-sm">
        <template v-for="(prog, idx) in counter.progression" :key="prog.level">
          <span v-if="idx > 0" class="text-gray-400">→</span>
          <span class="px-2 py-1 rounded bg-gray-200 dark:bg-gray-700">
            <span class="text-gray-500 dark:text-gray-400">L{{ prog.level }}:</span>
            <span class="font-medium text-gray-900 dark:text-gray-100 ml-1">{{ prog.value }}</span>
          </span>
        </template>
      </div>
    </div>
  </div>
</template>
```

**Acceptance Criteria**:
- [ ] Same-named counters consolidated into single display
- [ ] Level progression shown horizontally with arrows
- [ ] Reset timing shown once per counter type

---

### 2.7 Add Progression Table Level Highlighting

**File**: `app/components/ui/class/UiClassProgressionTable.vue`

**Enhancement**: Highlight rows where significant features are gained.

```vue
<tr
  v-for="(row, index) in progressionTable.rows"
  :key="row.level"
  :class="{
    'bg-gray-50/50 dark:bg-gray-800/50': index % 2 === 1,
    'bg-class-50 dark:bg-class-900/20 border-l-4 border-l-class-500': hasSignificantFeatures(row)
  }"
>
```

```typescript
const hasSignificantFeatures = (row: ProgressionRow): boolean => {
  const features = row.features || ''

  // Highlight ASI levels
  if (features.includes('Ability Score Improvement')) return true

  // Highlight subclass feature levels
  if (features.includes('Archetype Feature') ||
      features.includes('Tradition Feature') ||
      features.includes('Domain Feature') ||
      features.includes('Oath Feature') ||
      features.includes('Path Feature')) return true

  // Highlight Extra Attack
  if (features.includes('Extra Attack')) return true

  return false
}
```

**Acceptance Criteria**:
- [ ] ASI levels (4, 8, 12, 16, 19) highlighted
- [ ] Subclass feature levels highlighted
- [ ] Visual distinction without being overwhelming

---

### 2.8 Mobile-Responsive Progression Table

**File**: `app/components/ui/class/UiClassProgressionTable.vue`

**Current Issues**:
- Table requires horizontal scroll on mobile
- Many columns for spellcasters (9+ spell slot columns)

**Proposed Enhancement**: Collapsible columns or card view on mobile.

```vue
<template>
  <!-- Desktop: Full table -->
  <div class="hidden md:block overflow-x-auto">
    <!-- existing table -->
  </div>

  <!-- Mobile: Card view -->
  <div class="md:hidden space-y-2">
    <details
      v-for="row in progressionTable.rows"
      :key="row.level"
      class="border rounded-lg dark:border-gray-700"
    >
      <summary class="p-3 cursor-pointer flex items-center justify-between">
        <span class="font-medium">Level {{ row.level }}</span>
        <span class="text-sm text-gray-500">{{ row.proficiency_bonus }}</span>
      </summary>
      <div class="p-3 pt-0 space-y-2 text-sm">
        <div v-if="row.features && row.features !== '—'">
          <span class="text-gray-500">Features:</span>
          <span class="ml-1">{{ filterProgressionFeatures(row.features) }}</span>
        </div>
        <!-- Spell slots if applicable -->
        <div v-if="hasSpellSlots(row)" class="flex flex-wrap gap-2">
          <span
            v-for="slot in getSpellSlots(row)"
            :key="slot.level"
            class="px-2 py-1 rounded bg-gray-100 dark:bg-gray-800"
          >
            {{ slot.level }}: {{ slot.count }}
          </span>
        </div>
      </div>
    </details>
  </div>
</template>
```

**Acceptance Criteria**:
- [ ] Mobile users see collapsible level cards
- [ ] Desktop users see full table
- [ ] No horizontal scroll required on mobile

---

## 3. Component Enhancements

### 3.1 New Component: `UiClassSpellSlotsCompact.vue`

Display spell slots in a compact, visual format instead of 9 table columns.

```vue
<template>
  <div class="flex items-center gap-1">
    <template v-for="level in 9" :key="level">
      <div
        v-if="slots[level] > 0"
        class="w-6 h-6 rounded text-xs font-medium flex items-center justify-center"
        :class="level <= 5 ? 'bg-primary-100 text-primary-700' : 'bg-warning-100 text-warning-700'"
        :title="`${level}${getOrdinal(level)} level: ${slots[level]} slot${slots[level] > 1 ? 's' : ''}`"
      >
        {{ slots[level] }}
      </div>
    </template>
  </div>
</template>
```

### 3.2 New Component: `UiClassFeaturePreview.vue`

Compact feature preview for progression table cells.

```vue
<template>
  <div class="flex flex-wrap gap-1">
    <UBadge
      v-for="feature in limitedFeatures"
      :key="feature"
      color="neutral"
      variant="subtle"
      size="xs"
    >
      {{ feature }}
    </UBadge>
    <UBadge
      v-if="remainingCount > 0"
      color="info"
      variant="soft"
      size="xs"
    >
      +{{ remainingCount }}
    </UBadge>
  </div>
</template>
```

---

## 4. Backend-Dependent Changes

These require backend updates from [CLASSES-DETAIL-PAGE-BACKEND-FIXES.md](./CLASSES-DETAIL-PAGE-BACKEND-FIXES.md).

### 4.1 Use `is_choice_option` Flag

**Backend Requirement**: Add `is_choice_option` field to features

**Frontend Change**: Filter features by flag instead of regex patterns.

```typescript
// After backend update
const primaryFeatures = computed(() =>
  entity.value?.features?.filter(f => !f.is_choice_option) || []
)
```

### 4.2 Use `is_optional_rule` Flag

**Backend Requirement**: Add `is_optional_rule` field to features

**Frontend Change**: Separate optional rules into dedicated section.

```typescript
const coreFeatures = computed(() =>
  entity.value?.features?.filter(f => !f.is_optional_rule) || []
)

const optionalRuleFeatures = computed(() =>
  entity.value?.features?.filter(f => f.is_optional_rule) || []
)
```

### 4.3 Use `subclass_level` and `subclass_name`

**Backend Requirement**: Add fields to base classes

**Frontend Change**: Display in Quick Stats and subclass cards.

```typescript
// Quick Stats
if (entity.value.subclass_level) {
  stats.push({
    icon: 'i-heroicons-users',
    label: entity.value.subclass_name || 'Subclass',
    value: `Level ${entity.value.subclass_level}`
  })
}
```

### 4.4 Use `consolidated_counters`

**Backend Requirement**: Add consolidated counter format

**Frontend Change**: Simplify counter display component.

### 4.5 Use Nested `choice_options`

**Backend Requirement**: Nest choice options under parent feature

**Frontend Change**: Render as expandable sub-list.

```vue
<div v-for="feature in features" :key="feature.id">
  <FeatureHeader :feature="feature" />

  <div v-if="feature.choice_options?.length" class="ml-4 mt-2 space-y-1">
    <div v-for="option in feature.choice_options" :key="option.id" class="text-sm">
      • {{ option.feature_name }}
    </div>
  </div>
</div>
```

---

## 5. Implementation Plan

### Phase 1: Immediate Fixes (1-2 days)

| Task | File | Priority |
|------|------|----------|
| Handle hit_die: 0 | `[slug].vue` | Critical |
| Filter multiclass from progression | `UiClassProgressionTable.vue` | High |
| Collapse fighting style options | `UiClassProgressionTable.vue` | High |
| Use first feature for description | `[slug].vue` | High |
| Accurate feature count | `UiClassSubclassCards.vue` | Medium |
| Fix HP description for subclasses | `UiClassHitPointsCard.vue` | Medium |

### Phase 2: Layout Improvements (3-5 days)

| Task | File | Priority |
|------|------|----------|
| Enhanced Quick Stats | `[slug].vue` | Medium |
| Features grouped by level | New component | Medium |
| Improved subclass cards | `UiClassSubclassCards.vue` | Medium |
| Consolidated counters | `UiAccordionClassCounters.vue` | Low |
| Progression table highlighting | `UiClassProgressionTable.vue` | Low |
| Mobile-responsive table | `UiClassProgressionTable.vue` | Low |

### Phase 3: Post-Backend Updates (1-2 days)

| Task | Dependency | Priority |
|------|------------|----------|
| Use is_choice_option flag | Backend Phase 2 | High |
| Use is_optional_rule flag | Backend Phase 2 | High |
| Use subclass_level/name | Backend Phase 2 | Medium |
| Use consolidated_counters | Backend Phase 3 | Low |

---

## 6. Mockups

### Base Class Page (Proposed Layout)

```
┌─────────────────────────────────────────────────────────┐
│ ← Back to Classes                                       │
├─────────────────────────────────────────────────────────┤
│ WIZARD                                                  │
│ [Base Class] [✨ Intelligence]                          │
├─────────────────────────────────────────────────────────┤
│ ┌────────────────────┐  ┌──────────────────────────────┐│
│ │                    │  │ ❤️  Hit Die      d6          ││
│ │   [Wizard Image]   │  │ ✨ Spellcasting  Intelligence ││
│ │                    │  │ 🛡️  Saves        INT, WIS    ││
│ │                    │  │ 👥 Subclass at   Level 2     ││
│ │                    │  │ ⭐ Features      18 features ││
│ └────────────────────┘  └──────────────────────────────┘│
├─────────────────────────────────────────────────────────┤
│ DESCRIPTION                                             │
│ Wizards are supreme magic-users, defined and united...  │
│ [Read more...]                                          │
├─────────────────────────────────────────────────────────┤
│ ARCANE TRADITIONS (12)                                  │
│ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐        │
│ │Abjurat. │ │Conjurat.│ │Divinat. │ │Enchant. │        │
│ │PHB  L2  │ │PHB  L2  │ │PHB  L2  │ │PHB  L2  │        │
│ │6 feat.  │ │6 feat.  │ │6 feat.  │ │6 feat.  │        │
│ └─────────┘ └─────────┘ └─────────┘ └─────────┘        │
├─────────────────────────────────────────────────────────┤
│ CLASS FEATURES                                          │
│                                                         │
│ ● Level 1 ─────────────────────────────────────────    │
│   │ ▸ Spellcasting                                     │
│   │ ▸ Arcane Recovery                                  │
│                                                         │
│ ● Level 2 ─────────────────────────────────────────    │
│   │ ▸ Arcane Tradition (choose subclass)               │
│                                                         │
│ ● Level 4, 8, 12, 16, 19 ──────────────────────────    │
│   │ ▸ Ability Score Improvement                        │
│                                                         │
│ ● Level 18 ────────────────────────────────────────    │
│   │ ▸ Spell Mastery                                    │
│                                                         │
│ ● Level 20 ────────────────────────────────────────    │
│   │ ▸ Signature Spells                                 │
├─────────────────────────────────────────────────────────┤
│ ▸ Class Progression Table                               │
│ ▸ Hit Points                                            │
│ ▸ Starting Equipment                                    │
│ ▸ Proficiencies (12)                                    │
│ ▸ Multiclass Rules                                      │
│ ▸ Source                                                │
└─────────────────────────────────────────────────────────┘
```

### Subclass Page (Proposed Layout)

```
┌─────────────────────────────────────────────────────────┐
│ Classes > Wizard > School of Abjuration                 │
├─────────────────────────────────────────────────────────┤
│ SCHOOL OF ABJURATION                                    │
│ [Subclass of Wizard →]                                  │
├─────────────────────────────────────────────────────────┤
│ ┌────────────────────┐  ┌──────────────────────────────┐│
│ │                    │  │ 📖 Parent       Wizard       ││
│ │ [Subclass Image]   │  │ 🎯 Starts at    Level 2      ││
│ │      [Wizard]      │  │ ⭐ Features     6 features   ││
│ │                    │  │ 📚 Source       PHB p.115    ││
│ └────────────────────┘  └──────────────────────────────┘│
├─────────────────────────────────────────────────────────┤
│ The School of Abjuration emphasizes magic that blocks,  │
│ banishes, or protects. Detractors of this school say    │
│ that its tradition is about denial, negation rather     │
│ than positive assertion...                              │
├─────────────────────────────────────────────────────────┤
│ SCHOOL OF ABJURATION FEATURES                           │
│                                                         │
│ ● Level 2 ─────────────────────────────────────────    │
│   │ ▸ Abjuration Savant                                │
│   │ ▸ Arcane Ward                                      │
│                                                         │
│ ● Level 6 ─────────────────────────────────────────    │
│   │ ▸ Projected Ward                                   │
│                                                         │
│ ● Level 10 ────────────────────────────────────────    │
│   │ ▸ Improved Abjuration                              │
│                                                         │
│ ● Level 14 ────────────────────────────────────────    │
│   │ ▸ Spell Resistance                                 │
├─────────────────────────────────────────────────────────┤
│ ▸ Arcane Ward (1/Long Rest)           [Resource Card]  │
├─────────────────────────────────────────────────────────┤
│ ▸ Inherited from Wizard                                 │
│   • Class Progression Table                             │
│   • Hit Points                                          │
│   • Proficiencies                                       │
│   • Starting Equipment                                  │
│   • Base Class Features                                 │
├─────────────────────────────────────────────────────────┤
│ ▸ Source                                                │
└─────────────────────────────────────────────────────────┘
```

---

## Changelog

| Date | Author | Changes |
|------|--------|---------|
| 2025-11-26 | Claude | Initial proposal based on API verification audit |
