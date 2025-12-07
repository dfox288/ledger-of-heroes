# Spell Pages Enhancement Design

**Date:** 2025-11-20
**Status:** Ready for Implementation
**Scope:** Spell list and detail pages only (prototype for other entities)

---

## Problem Statement

Current spell pages are functional but lack polish:
- Loading states are minimal (just text)
- Empty states are missing helpful guidance
- Cards on list page are generic (not spell-specific)
- Detail pages lack visual hierarchy and debugging tools
- No easy way to see raw JSON data for debugging

## Solution Overview

**Enhanced Visual Hierarchy Approach:**
- Spell-specific cards on list page
- Skeleton loading animations
- Helpful empty states
- JSON debug toggle on detail pages
- Improved typography and spacing

---

## List Page Improvements (`/spells/index.vue`)

### 1. Skeleton Loading State

**Replace:** Simple "Loading..." text
**With:** Animated skeleton cards

**Implementation:**
```vue
<div v-if="loading" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  <UCard v-for="i in 6" :key="i" class="animate-pulse">
    <div class="space-y-3">
      <div class="h-4 bg-gray-200 dark:bg-gray-700 rounded w-3/4"></div>
      <div class="h-4 bg-gray-200 dark:bg-gray-700 rounded w-1/2"></div>
      <div class="h-20 bg-gray-200 dark:bg-gray-700 rounded"></div>
    </div>
  </UCard>
</div>
```

**Benefits:**
- Visual feedback that content is loading
- Matches final card layout
- Professional appearance

### 2. Enhanced Empty State

**Appears when:** Filters return 0 results

**Design:**
```
┌───────────────────────────────────┐
│         🔍                        │
│    No spells found                │
│                                   │
│  Try adjusting your filters or    │
│  search for different keywords    │
│                                   │
│    [Clear All Filters]            │
└───────────────────────────────────┘
```

**Implementation:**
```vue
<UCard v-if="!loading && spells.length === 0">
  <div class="text-center py-12">
    <UIcon name="i-heroicons-magnifying-glass" class="w-16 h-16 mx-auto mb-4 text-gray-400" />
    <h3 class="text-xl font-semibold mb-2">No spells found</h3>
    <p class="text-gray-600 dark:text-gray-400 mb-4">
      Try adjusting your filters or search for different keywords
    </p>
    <UButton @click="clearFilters" color="primary">
      Clear All Filters
    </UButton>
  </div>
</UCard>
```

### 3. Spell-Specific Card Component

**Create:** `app/components/spell/SpellCard.vue`

**Card Layout:**
```
┌─────────────────────────────────────────┐
│ [3rd Level] [Evocation]                 │
│ Fireball                                │
│ ⏱️ 1 action • 📏 150 feet               │
│ [🔮 Ritual] [⭐ Concentration]          │
│                                         │
│ A bright streak flashes from your      │
│ pointing finger to a point you...      │
└─────────────────────────────────────────┘
```

**Key Features:**
- Level and school badges at top
- Spell name as heading
- Quick stats (casting time, range) with icons
- Conditional badges (ritual, concentration)
- Description preview (first 150 chars)
- Hover effect (shadow-lg)
- Clickable to detail page

### 4. Active Filter Chips

**Display:** Above results grid

**Design:**
```
Active Filters:
[Level: 3rd ✕] [School: Evocation ✕] [Clear All]
```

**Implementation:**
```vue
<div v-if="hasActiveFilters" class="flex flex-wrap gap-2 mb-4">
  <span class="text-sm text-gray-600 dark:text-gray-400">Active Filters:</span>
  <UButton v-if="selectedLevel !== null" size="xs" variant="soft" @click="selectedLevel = null">
    Level: {{ selectedLevel }} ✕
  </UButton>
  <UButton v-if="selectedSchool !== null" size="xs" variant="soft" @click="selectedSchool = null">
    School: {{ getSchoolName(selectedSchool) }} ✕
  </UButton>
  <UButton size="xs" color="gray" @click="clearAllFilters">
    Clear All
  </UButton>
</div>
```

### 5. Enhanced Results Header

**Show:** Total count with filter context

**Before:** "Spells"
**After:** "Spells (477 total)" or "Spells (23 of 477)"

```vue
<h1 class="text-3xl font-bold mb-4">
  Spells
  <span class="text-xl text-gray-500 dark:text-gray-400">
    ({{ hasActiveFilters ? `${totalResults} of ${totalSpells}` : `${totalResults} total` }})
  </span>
</h1>
```

---

## Detail Page Improvements (`/spells/[slug].vue`)

### 1. Breadcrumb Navigation

**Add:** Back link at top of page

```vue
<div class="mb-4">
  <UButton to="/spells" variant="ghost" color="gray" icon="i-heroicons-arrow-left">
    Back to Spells
  </UButton>
</div>
```

### 2. Enhanced Header

**Layout:**
```
[← Back to Spells]

[3rd Level] [Evocation] [🔮 Ritual] [⭐ Concentration]
Fireball                                    [View JSON]
```

**Implementation:**
- Spell name: text-5xl font-bold
- JSON toggle button in top-right corner
- Badges with better spacing and icons

```vue
<div class="flex justify-between items-start mb-4">
  <div>
    <div class="flex items-center gap-2 mb-3">
      <UBadge color="purple" size="lg">{{ spellLevelText }} {{ spell.school.name }}</UBadge>
      <UBadge v-if="spell.is_ritual" color="blue" size="sm">🔮 Ritual</UBadge>
      <UBadge v-if="spell.needs_concentration" color="orange" size="sm">⭐ Concentration</UBadge>
    </div>
    <h1 class="text-5xl font-bold text-gray-900 dark:text-gray-100">
      {{ spell.name }}
    </h1>
  </div>
  <UButton @click="showJson = !showJson" variant="ghost" color="gray" icon="i-heroicons-code-bracket">
    {{ showJson ? 'Hide' : 'View' }} JSON
  </UButton>
</div>
```

### 3. Improved Stats Grid

**Add icons to each stat:**

```vue
<UCard>
  <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
    <div class="flex items-start gap-3">
      <UIcon name="i-heroicons-clock" class="w-5 h-5 text-gray-400 mt-1" />
      <div>
        <div class="text-sm font-semibold text-gray-500 dark:text-gray-400 uppercase mb-1">
          Casting Time
        </div>
        <div class="text-lg text-gray-900 dark:text-gray-100">
          {{ spell.casting_time }}
        </div>
      </div>
    </div>

    <div class="flex items-start gap-3">
      <UIcon name="i-heroicons-arrow-trending-up" class="w-5 h-5 text-gray-400 mt-1" />
      <div>
        <div class="text-sm font-semibold text-gray-500 dark:text-gray-400 uppercase mb-1">
          Range
        </div>
        <div class="text-lg text-gray-900 dark:text-gray-100">
          {{ spell.range }}
        </div>
      </div>
    </div>

    <!-- Similar for Components and Duration -->
  </div>
</UCard>
```

### 4. JSON Debug Panel

**Collapsible card at bottom of page:**

```vue
<script setup>
const showJson = ref(false)

const copyJson = () => {
  navigator.clipboard.writeText(JSON.stringify(spell.value, null, 2))
  // Show toast notification
}
</script>

<template>
  <UCard v-if="showJson" class="mt-8">
    <template #header>
      <div class="flex justify-between items-center">
        <h2 class="text-xl font-semibold">Raw JSON Data</h2>
        <div class="flex gap-2">
          <UButton @click="copyJson" size="xs" variant="ghost" icon="i-heroicons-clipboard">
            Copy
          </UButton>
          <UButton @click="showJson = false" size="xs" variant="ghost" icon="i-heroicons-x-mark">
            Close
          </UButton>
        </div>
      </div>
    </template>
    <pre class="bg-gray-900 text-gray-100 p-4 rounded-lg overflow-x-auto text-sm"><code>{{ JSON.stringify(spell, null, 2) }}</code></pre>
  </UCard>
</template>
```

**Features:**
- Toggle with button in header
- Dark background (code-like appearance)
- Copy to clipboard button
- Close button
- Properly formatted JSON with 2-space indent
- Horizontal scroll for long lines

### 5. Typography & Spacing Improvements

**Changes:**
- Section headings: text-xl → text-2xl
- Main spacing: space-y-6 → space-y-8
- Description text: base → text-lg
- Better line height for readability

---

## Implementation Plan

### Step 1: Create SpellCard Component
1. Create `app/components/spell/SpellCard.vue`
2. Accept spell prop
3. Implement card layout with badges, stats, description preview
4. Add hover effects
5. Make clickable (NuxtLink to detail page)

### Step 2: Update List Page
1. Add skeleton loading state (6 cards)
2. Add empty state with clear filters button
3. Replace SearchResultCard with SpellCard
4. Add active filter chips above results
5. Update results header with count context
6. Implement clearAllFilters() method

### Step 3: Update Detail Page
1. Add breadcrumb "Back to Spells" button
2. Enlarge spell name to text-5xl
3. Add JSON toggle button to header
4. Implement showJson reactive ref
5. Add icons to stats grid
6. Create collapsible JSON debug card
7. Implement copyJson() function
8. Increase spacing and typography sizes

### Step 4: Testing
1. Test list page with 0 results
2. Test list page loading state
3. Test filter chips remove correctly
4. Test spell card click navigation
5. Test detail page JSON toggle
6. Test JSON copy button
7. Test breadcrumb navigation
8. Verify mobile responsive layout

---

## Component Structure

```
app/
├── components/
│   └── spell/
│       └── SpellCard.vue          [NEW] Spell-specific card
├── pages/
│   └── spells/
│       ├── index.vue              [MODIFY] Enhanced list page
│       └── [slug].vue             [MODIFY] Enhanced detail page
```

---

## Visual Design Tokens

### Colors
- Spell level badge: purple (magical theme)
- Ritual badge: blue
- Concentration badge: orange
- Empty state icon: gray-400
- JSON panel background: gray-900
- JSON panel text: gray-100

### Icons (Heroicons)
- Back arrow: arrow-left
- Loading: arrow-path (spinning)
- Empty state: magnifying-glass
- Stats - Casting time: clock
- Stats - Range: arrow-trending-up
- Stats - Components: sparkles
- Stats - Duration: clock
- JSON toggle: code-bracket
- Copy: clipboard
- Close: x-mark

### Spacing
- Card gap: gap-4
- Section spacing: space-y-8
- Grid gaps: gap-6
- Badge gaps: gap-2

### Typography
- Main heading: text-5xl font-bold
- Section heading: text-2xl font-semibold
- Card title: text-xl font-semibold
- Description: text-lg
- Stats value: text-lg
- Stats label: text-sm uppercase

---

## Expected Outcomes

**List Page:**
- ✅ Professional loading animation
- ✅ Helpful empty state with guidance
- ✅ Beautiful spell-specific cards
- ✅ Clear visual feedback for active filters
- ✅ Easy filter removal

**Detail Page:**
- ✅ Clear navigation back to list
- ✅ Prominent spell name and badges
- ✅ Easy-to-read stats with icons
- ✅ JSON debugging tool for developers
- ✅ Better typography and spacing
- ✅ Copy JSON functionality

**Developer Experience:**
- ✅ Can easily inspect API responses
- ✅ Can copy JSON for testing
- ✅ Debug panel doesn't interfere with main content

---

## Future Enhancements (Not in Scope)

- Syntax highlighting for JSON (would require library)
- Expandable/collapsible sections on detail page
- "Share" button to copy URL
- Print stylesheet for detail pages
- Favorites/bookmark system

---

**Status:** Design approved, ready for implementation

**Next Step:** Implement improvements following the plan above
