# UiFilterLayout Component - Implementation Guide

**Created:** 2025-11-25
**Status:** ✅ Production-ready
**Component:** `app/components/ui/filter/UiFilterLayout.vue`
**Tests:** `tests/components/ui/filter/UiFilterLayout.test.ts` (16 tests, 100% passing)

---

## Overview

`UiFilterLayout` provides a consistent, responsive 3-tier layout structure for entity list filters. It eliminates layout code duplication across entity pages and ensures visual consistency.

**Based on:** Spell filter refactor (Nov 25, 2025)
**Applied to:** Spells page (✅), ready for 6 more entity pages

---

## Why This Component Exists

### Problem (Before)
Each entity page had to manually implement:
- Flex/grid layouts with specific responsive breakpoints
- Consistent spacing (space-y-4, gap-3)
- Proper alignment (primary flex, quick grid, advanced flex, actions right-aligned)
- ~50 lines of repetitive layout HTML per page

### Solution (After)
One reusable component with semantic slots:
- Encapsulates layout logic (spacing, responsiveness, alignment)
- Declarative API (slots describe purpose: primary, quick, advanced, actions)
- Single source of truth for filter layout
- ~30 lines of clean slot-based HTML per page

---

## API Reference

### Slots (4 total - all optional)

#### 1. `#primary` - Primary Filters
**Purpose:** Most frequently used filters (dropdowns, select menus)
**Layout:** `flex flex-wrap gap-3`
**Use for:** Level, School, Class, Rarity, Size, Type, etc.
**Width:** `w-full sm:w-48` (192px on desktop)

#### 2. `#quick` - Quick Toggles
**Purpose:** Binary toggle filters (All/Yes/No choices)
**Layout:** `grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-5 gap-3`
**Use for:** Concentration, Ritual, Legendary, Attunement, V/S/M components
**Responsive:** 2 cols mobile → 3 cols tablet → 5 cols desktop

#### 3. `#advanced` - Advanced Filters
**Purpose:** Less frequently used filters (multi-select, range sliders)
**Layout:** `flex flex-wrap gap-3`
**Use for:** Damage Types, Saving Throws, Skills, Languages, etc.
**Width:** `w-full sm:w-48` (192px on desktop)

#### 4. `#actions` - Action Buttons
**Purpose:** Filter action buttons (clear, apply, reset)
**Layout:** `flex justify-end`
**Use for:** Clear Filters, Apply, Reset buttons
**Alignment:** Right-aligned

### Props
None. This is a pure layout component controlled entirely by slots.

### Events
None. Child components within slots emit their own events.

---

## Usage Example (Spells Page)

```vue
<UiFilterLayout>
  <!-- Primary: Most used filters -->
  <template #primary>
    <USelectMenu
      v-model="selectedLevel"
      :items="levelOptions"
      placeholder="All Levels"
      class="w-full sm:w-48"
    />
    <USelectMenu
      v-model="selectedSchool"
      :items="schoolOptions"
      placeholder="All Schools"
      class="w-full sm:w-48"
    />
    <USelectMenu
      v-model="selectedClass"
      :items="classOptions"
      placeholder="All Classes"
      class="w-full sm:w-48"
    />
  </template>

  <!-- Quick: Binary toggles -->
  <template #quick>
    <UiFilterToggle v-model="concentration" label="Concentration" />
    <UiFilterToggle v-model="ritual" label="Ritual" />
    <UiFilterToggle v-model="verbal" label="Verbal" />
    <UiFilterToggle v-model="somatic" label="Somatic" />
    <UiFilterToggle v-model="material" label="Material" />
  </template>

  <!-- Advanced: Multi-select filters -->
  <template #advanced>
    <UiFilterMultiSelect
      v-model="damageTypes"
      :options="damageTypeOptions"
      label="Damage Types"
      class="w-full sm:w-48"
    />
    <UiFilterMultiSelect
      v-model="savingThrows"
      :options="savingThrowOptions"
      label="Saving Throws"
      class="w-full sm:w-48"
    />
  </template>

  <!-- Actions: Clear button -->
  <template #actions>
    <UButton
      v-if="hasActiveFilters"
      color="neutral"
      variant="soft"
      @click="clearFilters"
    >
      Clear Filters
    </UButton>
  </template>
</UiFilterLayout>
```

---

## Applying to Other Entity Pages

### Step-by-Step Migration

**1. Identify Existing Filters**

Look for filter sections in `app/pages/{entity}/index.vue`:
```vue
<!-- OLD: Manual layout -->
<div class="space-y-4">
  <div class="flex flex-wrap gap-2">
    <!-- Filters here -->
  </div>
</div>
```

**2. Categorize Filters by Tier**

| Tier | Filter Type | Examples |
|------|-------------|----------|
| **Primary** | Dropdowns, select menus | Level, School, Class, Type, Rarity, Size, CR |
| **Quick** | Binary toggles (All/Yes/No) | Concentration, Ritual, Legendary, Attunement, Magic |
| **Advanced** | Multi-select, range sliders | Damage Types, Skills, Languages, Alignments |
| **Actions** | Buttons | Clear Filters, Apply, Reset |

**3. Replace with UiFilterLayout**

```vue
<!-- NEW: Semantic slots -->
<UiFilterLayout>
  <template #primary>
    <!-- Move dropdown filters here -->
  </template>
  <template #quick>
    <!-- Move toggle filters here -->
  </template>
  <template #advanced>
    <!-- Move multi-select filters here -->
  </template>
  <template #actions>
    <!-- Move action buttons here -->
  </template>
</UiFilterLayout>
```

**4. Ensure Consistent Widths**

- Primary dropdowns: `class="w-full sm:w-48"`
- Advanced filters: `class="w-full sm:w-48"`
- Quick toggles: No width needed (grid handles it)

**5. Test**

- ✅ All filters work correctly
- ✅ Page loads (HTTP 200)
- ✅ Layout is responsive (mobile, tablet, desktop)
- ✅ Dark mode looks good

---

## Entity-Specific Examples

### Items Page

```vue
<UiFilterLayout>
  <template #primary>
    <USelectMenu v-model="itemType" :items="itemTypes" class="w-full sm:w-48" />
    <USelectMenu v-model="rarity" :items="rarities" class="w-full sm:w-48" />
  </template>

  <template #quick>
    <UiFilterToggle v-model="requiresAttunement" label="Attunement" />
    <UiFilterToggle v-model="isMagic" label="Magic" />
  </template>

  <template #advanced>
    <UiFilterMultiSelect v-model="properties" label="Properties" class="w-full sm:w-48" />
  </template>

  <template #actions>
    <UButton @click="clearFilters">Clear Filters</UButton>
  </template>
</UiFilterLayout>
```

### Monsters Page

```vue
<UiFilterLayout>
  <template #primary>
    <USelectMenu v-model="size" :items="sizes" class="w-full sm:w-48" />
    <USelectMenu v-model="type" :items="types" class="w-full sm:w-48" />
    <USelectMenu v-model="cr" :items="challengeRatings" class="w-full sm:w-48" />
  </template>

  <template #quick>
    <UiFilterToggle v-model="legendary" label="Legendary" />
    <UiFilterToggle v-model="spellcaster" label="Spellcaster" />
  </template>

  <template #advanced>
    <UiFilterMultiSelect v-model="damageTypes" label="Damage Types" class="w-full sm:w-48" />
    <UiFilterMultiSelect v-model="environments" label="Environments" class="w-full sm:w-48" />
  </template>

  <template #actions>
    <UButton @click="clearFilters">Clear Filters</UButton>
  </template>
</UiFilterLayout>
```

### Races Page (Minimal Filters)

```vue
<UiFilterLayout>
  <template #primary>
    <USelectMenu v-model="size" :items="sizes" class="w-full sm:w-48" />
  </template>

  <template #quick>
    <UiFilterToggle v-model="hasSubraces" label="Has Subraces" />
  </template>

  <template #actions>
    <UButton @click="clearFilters">Clear Filters</UButton>
  </template>
</UiFilterLayout>
```

---

## Responsive Behavior

### Breakpoints

| Screen Size | Primary | Quick Toggles | Advanced |
|-------------|---------|---------------|----------|
| **Mobile (<640px)** | Stack vertically (full width) | 2 columns grid | Stack vertically (full width) |
| **Tablet (640-1024px)** | 3 dropdowns per row (192px each) | 3 columns grid | 2 filters per row (192px each) |
| **Desktop (>1024px)** | 3 dropdowns per row (192px each) | **5 columns grid (all in one row!)** | 2 filters per row (192px each) |

### Spacing

- **Between tiers:** `space-y-4` (16px)
- **Between filters:** `gap-3` (12px)

---

## Visual Layout Diagram

```
┌─────────────────────────────────────────────────────────┐
│ PRIMARY (flex flex-wrap gap-3)                          │
│ [Level ▼] [School ▼] [Class ▼]                         │
├─────────────────────────────────────────────────────────┤
│ QUICK (grid 2→3→5 cols gap-3)                           │
│ [Concentration] [Ritual] [Verbal] [Somatic] [Material]  │
├─────────────────────────────────────────────────────────┤
│ ADVANCED (flex flex-wrap gap-3)                         │
│ [Damage Types ▼] [Saving Throws ▼]                     │
├─────────────────────────────────────────────────────────┤
│ ACTIONS (flex justify-end)                              │
│                                     [Clear Filters →]   │
└─────────────────────────────────────────────────────────┘
```

---

## Benefits

### Consistency
- All entity pages have identical layout structure
- Same spacing, responsiveness, alignment
- Single source of truth for filter layout

### Maintainability
- One place to tweak layout (e.g., change spacing from gap-3 to gap-4)
- Changes automatically apply to all pages using the component
- Easier to onboard new developers (semantic slots vs manual divs)

### Cleaner Code
- Reduces layout HTML by ~20 lines per page
- More declarative (slots describe purpose)
- Easier to read and understand

### Performance
- No performance difference (pure template component)
- Same number of DOM nodes as manual layout

---

## Migration Checklist

Use this checklist when applying `UiFilterLayout` to a new entity page:

- [ ] **Backup:** Copy current filter section to clipboard
- [ ] **Categorize:** Identify which filters belong in which tier
- [ ] **Replace:** Wrap filters in `<UiFilterLayout>` with appropriate slots
- [ ] **Widths:** Ensure `w-full sm:w-48` on primary/advanced filters
- [ ] **Test filters:** Verify all filters still work correctly
- [ ] **Test layout:** Check mobile, tablet, desktop layouts
- [ ] **Test dark mode:** Verify dark mode styling
- [ ] **Commit:** Commit changes with descriptive message

---

## Testing

### Component Tests
Location: `tests/components/ui/filter/UiFilterLayout.test.ts`
Coverage: 16 tests
- Slot rendering (primary, quick, advanced, actions)
- Layout structure (flex, grid, spacing)
- Empty slots (conditional rendering)
- Dark mode support

### Integration Tests
- Spells page: ✅ All filters working
- Other pages: Ready to test after migration

### Manual Testing
1. Visit entity page (e.g., `/spells`)
2. Open filters panel
3. Verify layout looks correct on mobile, tablet, desktop
4. Verify all filters work (dropdowns, toggles, multi-select)
5. Verify dark mode styling

---

## Related Components

- **`UiFilterCollapse`** - Search input + collapsible filter panel
- **`UiFilterToggle`** - Binary toggle buttons (All/Yes/No)
- **`UiFilterMultiSelect`** - Multi-select dropdown with count badge
- **`UiFilterRangeSlider`** - Range slider (future use)

---

## Next Steps

Apply `UiFilterLayout` to these 6 entity pages:

1. **Items** (`app/pages/items/index.vue`)
2. **Monsters** (`app/pages/monsters/index.vue`)
3. **Races** (`app/pages/races/index.vue`)
4. **Classes** (`app/pages/classes/index.vue`)
5. **Backgrounds** (`app/pages/backgrounds/index.vue`)
6. **Feats** (`app/pages/feats/index.vue`)

**Estimated time per page:** 15-30 minutes
**Total time:** 2-3 hours for all 6 pages

---

## Questions & Answers

**Q: Can I skip a tier?**
A: Yes! All slots are optional. If you don't have advanced filters, just don't include the `#advanced` slot.

**Q: Can I add custom content between tiers?**
A: Not with this component. If you need custom layout, consider using manual layout or extending the component.

**Q: Can I use different widths for filters?**
A: Yes, but stick to `w-full sm:w-48` for consistency. If you need different widths, add them inline.

**Q: What if I have more than 5 quick toggles?**
A: They'll wrap to a second row automatically. The grid handles this gracefully.

**Q: Can I use this outside of filter contexts?**
A: Technically yes, but it's optimized for filters. For other layouts, create a different component.

---

## Changelog

**2025-11-25:**
- ✅ Created `UiFilterLayout` component (TDD)
- ✅ Refactored spells page to use it
- ✅ Created comprehensive documentation
- ✅ Ready for migration to 6 other entity pages

---

## Support

If you encounter issues or have questions:
1. Check this guide first
2. Look at the spells page implementation (`app/pages/spells/index.vue`)
3. Review component tests (`tests/components/ui/filter/UiFilterLayout.test.ts`)
4. Check component JSDoc (`app/components/ui/filter/UiFilterLayout.vue`)

---

**Status:** ✅ Production-ready, documented, tested
**Next Agent:** Read this guide, then apply to items/monsters/races/classes/backgrounds/feats pages
