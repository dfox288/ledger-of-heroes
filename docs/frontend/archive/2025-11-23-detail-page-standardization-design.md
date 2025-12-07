# Detail Page Standardization - Design Document

**Date:** 2025-11-23
**Status:** 📋 Design Phase
**Objective:** Standardize all 7 detail pages with consistent structure, improved image integration, and complete API data display

---

## 🎯 Goals

1. **Structural Consistency** - All 7 detail pages follow identical patterns
2. **Complete Data Display** - Show ALL available API fields (currently missing ~10 fields)
3. **Improved Image Layout** - Integrate images into content (2/3-1/3 split) instead of separate column
4. **Component Reuse** - Leverage existing components, create minimal new ones
5. **Maintainability** - Future pages follow clear template pattern

---

## 📊 Current State Analysis

### Audit Results: What We Found

**7 Detail Pages Audited:**
- ✅ Spells, Items, Races, Classes, Backgrounds, Feats, Monsters

**Key Discrepancies:**

| Issue | Pages Affected | Impact |
|-------|---------------|--------|
| **Inconsistent always-visible sections** | Backgrounds, Feats (no accordion pattern) | UX inconsistency |
| **Monsters missing image integration** | Monsters | Visual inconsistency |
| **Description wrapper varies** | 4 pages use `<UCard v-if>`, 3 use different patterns | Code inconsistency |
| **Image feels disconnected** | All 6 pages with images | Poor visual integration |
| **Missing API data fields** | All pages | Incomplete feature set |

### Missing API Data (Critical Findings)

**HIGH Priority:**
- 🔴 **Monsters:** `spellcasting` field - Many monsters cast spells, not displayed!
- 🔴 **Monsters:** `conditions` field - Immunities/vulnerabilities not shown

**MEDIUM Priority:**
- 🟡 **Classes:** `counters` field - Ki points, Rage uses, etc. not shown
- 🟡 **Classes:** `level_progression` field - Level tables not shown
- 🟡 **Classes:** `traits` field - Base class traits not shown

**LOW Priority:**
- 🟢 **Feats:** `prerequisites_text`, `proficiencies`, `conditions` fields
- 🟢 **Items:** `detail`, `prerequisites` fields

---

## 🏗️ Design Solution: Approach 1 (Minimal Refactor)

### Core Principles

1. **Wrapper Component Pattern** - Create layout wrapper, not complete rewrite
2. **Slot-Based Flexibility** - Entity-specific content via slots
3. **Incremental Enhancement** - Add missing fields page-by-page
4. **TDD Required** - All new components must have tests first

### Architecture Overview

```
┌─────────────────────────────────────────────┐
│ Standard Detail Page Structure             │
├─────────────────────────────────────────────┤
│ 1. Loading State (UiDetailPageLoading)     │
│ 2. Error State (UiDetailPageError)         │
│ 3. Content Block:                           │
│    a. Breadcrumb (UiBackLink)              │
│    b. Header (UiDetailPageHeader)          │
│    c. Quick Stats (UiDetailQuickStatsCard) │
│    d. Description + Image (NEW!)           │
│    e. Always-Visible Sections (varies)     │
│    f. Accordion (UAccordion)               │
│    g. Debug Panel (JsonDebugPanel)         │
└─────────────────────────────────────────────┘
```

---

## 🎨 Image Layout Redesign

### Problem Statement

**Current:** Image in separate column via `UiEntityHeaderWithImage`
```vue
<!-- BEFORE: Image separate, feels disconnected -->
<UiEntityHeaderWithImage>
  <div class="flex flex-row gap-8">
    <div class="flex-1">Title + Badges</div>
    <div class="w-1/3">Image</div>
  </div>
</UiEntityHeaderWithImage>
```

**Issues:**
- ❌ Image dominates layout (full column height)
- ❌ Feels separate from content
- ❌ Wasted space when no image
- ❌ Not responsive to content length

### Solution: Integrated Image in Description Card

**New Pattern:** Image inside first content card (description)

```vue
<!-- AFTER: Image integrated with content -->
<UiDetailDescriptionWithImage
  :description="entity.description"
  :image-path="imagePath"
  :image-alt="`${entity.name} illustration`"
>
  <div class="flex flex-col lg:flex-row gap-6">
    <!-- Content: 2/3 width -->
    <div :class="imagePath ? 'lg:w-2/3' : 'w-full'">
      <h2>Description</h2>
      <p>{{ description }}</p>
    </div>

    <!-- Image: 1/3 width (conditional) -->
    <div v-if="imagePath" class="lg:w-1/3">
      <NuxtImg :src="imagePath" ... />
    </div>
  </div>
</UiDetailDescriptionWithImage>
```

**Benefits:**
- ✅ Image contextual to content (same card)
- ✅ Smaller image footprint (1/3 vs full column)
- ✅ Full-width content when no image
- ✅ Mobile-friendly (image stacks below)

**Responsive Behavior:**
- **Mobile (<1024px):** Image below description (full width)
- **Desktop (≥1024px):** Side-by-side 2/3-1/3 split
- **No Image:** Full-width description

---

## 📦 Component Architecture

### New Components

#### 1. `<UiDetailDescriptionWithImage>` (HIGH Priority)

**Purpose:** Display entity description with optional integrated image

**Props:**
```typescript
interface Props {
  description?: string        // Entity description text
  imagePath?: string | null   // Optional image path
  imageAlt?: string          // Image alt text
  title?: string             // Card title (default: "Description")
}
```

**Features:**
- Automatic 2/3-1/3 layout when image present
- Full-width when no image
- Graceful handling of missing description
- Prose styling for description text

**Tests Required:**
- Displays description text correctly
- Shows image when path provided
- Uses 2/3 layout with image
- Uses full-width without image
- Handles missing description gracefully
- Shows custom title if provided

---

#### 2. `<UiAccordionSpellcasting>` (HIGH Priority)

**Purpose:** Display monster spellcasting abilities

**Props:**
```typescript
interface Props {
  spellcasting: {
    ability?: string           // Spellcasting ability (INT, WIS, CHA)
    spell_save_dc?: number     // Spell save DC
    spell_attack_bonus?: number // Spell attack bonus
    slots?: {                  // Spell slots by level
      level_1?: number
      level_2?: number
      // ... up to level_9
    }
    spells?: Array<{           // Known spells
      level: number
      name: string
      uses?: number            // Uses per day (if applicable)
    }>
  }
}
```

**Features:**
- Display spellcasting ability and modifiers
- Show spell slots by level
- List known spells grouped by level
- Handle at-will spells (level 0)

**Tests Required:**
- Displays spellcasting ability
- Shows spell save DC and attack bonus
- Renders spell slots correctly
- Groups spells by level
- Handles missing slots gracefully
- Shows at-will spells separately

---

#### 3. `<UiAccordionCountersList>` (MEDIUM Priority)

**Purpose:** Display class counters (Ki, Rage, Sorcery Points, etc.)

**Props:**
```typescript
interface Props {
  counters: Array<{
    id: number
    name: string              // Counter name (e.g., "Ki Points")
    max_value?: number        // Maximum value
    recharge_timing?: string  // When it recharges (e.g., "Short Rest")
    description?: string      // Counter description
  }>
}
```

**Features:**
- Display counter name and max value
- Show recharge timing if present
- Optional description text

**Tests Required:**
- Displays counter name and max value
- Shows recharge timing
- Handles missing max value
- Displays description if present
- Handles empty counters array

---

#### 4. `<UiAccordionLevelProgression>` (MEDIUM Priority)

**Purpose:** Display class level progression table

**Props:**
```typescript
interface Props {
  levelProgression: Array<{
    level: number
    features?: string[]       // Features gained at this level
    proficiency_bonus?: number
    spell_slots?: Record<number, number> // Spell slots by level
  }>
}
```

**Features:**
- Table view of level progression
- Show features gained per level
- Display spell slots if applicable
- Proficiency bonus column

**Tests Required:**
- Renders table with correct columns
- Displays features for each level
- Shows spell slots if present
- Handles missing data gracefully

---

### Modified Components

#### `<UiDetailPageHeader>` (Update)

**Current State:** Simple title + badges, no image support

**Changes Needed:**
- ✅ No changes required! Keep simple.
- ✅ Image now in description card (not header)

#### `<UiEntityHeaderWithImage>` (Deprecate)

**Action:** Mark as deprecated, migrate all usages to:
- `<UiDetailPageHeader>` for title + badges
- `<UiDetailDescriptionWithImage>` for description + image

**Migration:** Update all 6 pages using this component (spells, items, races, classes, backgrounds, feats)

---

### Reused Components (No Changes)

- `<UiDetailPageLoading>` - Loading skeleton state
- `<UiDetailPageError>` - 404 error state
- `<UiDetailQuickStatsCard>` - Quick stats grid
- `<UiBackLink>` - Breadcrumb navigation
- `<JsonDebugPanel>` - Debug panel
- All existing accordion components (`UiAccordionTraitsList`, `UiAccordionBadgeList`, etc.)

---

## 📋 Standard Page Template

### Complete Template Structure

```vue
<script setup lang="ts">
import type { EntityType } from '~/types/api/entities'

const route = useRoute()

// Fetch entity data
const { data: entity, loading, error } = useEntityDetail<EntityType>({
  slug: route.params.slug as string,
  endpoint: '/entities',
  cacheKey: 'entity',
  seo: {
    titleTemplate: name => `${name} - D&D 5e Entity`,
    descriptionExtractor: (e: unknown) => {
      const entity = e as { description?: string }
      return entity.description?.substring(0, 160) || ''
    },
    fallbackTitle: 'Entity - D&D 5e Compendium'
  }
})

// Get image path
const { getImagePath } = useEntityImage()
const imagePath = computed(() => {
  if (!entity.value) return null
  return getImagePath('entities', entity.value.slug, 512)
})

// Quick stats (entity-specific)
const quickStats = computed(() => {
  if (!entity.value) return []
  return [
    // ... entity-specific stats
  ]
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <!-- 1. Loading State -->
    <UiDetailPageLoading
      v-if="loading"
      entity-type="entity"
    />

    <!-- 2. Error State -->
    <UiDetailPageError
      v-else-if="error"
      entity-type="Entity"
    />

    <!-- 3. Content -->
    <div
      v-else-if="entity"
      class="space-y-8"
    >
      <!-- 3a. Breadcrumb -->
      <UiBackLink
        to="/entities"
        label="Back to Entities"
      />

      <!-- 3b. Header (title + badges) -->
      <UiDetailPageHeader
        :title="entity.name"
        :badges="[
          // Entity-specific badges
        ]"
      />

      <!-- 3c. Quick Stats (conditional) -->
      <UiDetailQuickStatsCard
        v-if="quickStats.length > 0"
        :stats="quickStats"
      />

      <!-- 3d. Description + Image (integrated) -->
      <UiDetailDescriptionWithImage
        :description="entity.description"
        :image-path="imagePath"
        :image-alt="`${entity.name} illustration`"
      />

      <!-- 3e. Always-Visible Sections (entity-specific) -->
      <!-- Examples:
           - Ability Score Increases (races)
           - Prerequisites (feats)
           - Background Traits (backgrounds)
      -->

      <!-- 3f. Accordion (secondary data) -->
      <UAccordion
        :items="[
          // Conditional accordion items based on available data
        ]"
        type="multiple"
      >
        <!-- Entity-specific slots for each accordion item -->
      </UAccordion>

      <!-- 3g. JSON Debug Panel -->
      <JsonDebugPanel
        :data="entity"
        title="Entity Data"
      />
    </div>
  </div>
</template>
```

---

## 🚀 Implementation Plan

### Phase A: Core Refactor (4-6 hours)

**Goal:** Standardize structure and image layout across all pages

**Tasks:**
1. ✅ Create `<UiDetailDescriptionWithImage>` component (TDD)
   - Write tests first (RED)
   - Implement component (GREEN)
   - Verify all tests pass
2. ✅ Update Spells page to new pattern
3. ✅ Update Items page to new pattern
4. ✅ Update Races page to new pattern
5. ✅ Update Classes page to new pattern
6. ✅ Update Backgrounds page to new pattern
7. ✅ Update Feats page to new pattern
8. ✅ Update Monsters page to new pattern (add image support)
9. ✅ Deprecate `<UiEntityHeaderWithImage>` component
10. ✅ Verify all pages work (HTTP 200)
11. ✅ Run full test suite
12. ✅ Commit changes

**Success Criteria:**
- All 7 pages follow identical structure
- Image layout consistent (2/3-1/3 split)
- All tests passing
- No regressions

---

### Phase B: Missing Data - High Priority (2-3 hours)

**Goal:** Add critical missing monster data

**Tasks:**
1. ✅ Create `<UiAccordionSpellcasting>` component (TDD)
   - Write tests first (RED)
   - Implement component (GREEN)
   - Verify all tests pass
2. ✅ Add spellcasting to monsters detail page
3. ✅ Add conditions to monsters detail page (reuse pattern from races)
4. ✅ Test with monsters that have spellcasting
5. ✅ Verify all pages work
6. ✅ Commit changes

**Success Criteria:**
- Monster spellcasting displayed correctly
- Monster conditions displayed
- All tests passing

---

### Phase C: Missing Data - Medium Priority (3-4 hours)

**Goal:** Add valuable class data

**Tasks:**
1. ✅ Create `<UiAccordionCountersList>` component (TDD)
2. ✅ Create `<UiAccordionLevelProgression>` component (TDD)
3. ✅ Add counters to class detail pages
4. ✅ Add level progression to class detail pages
5. ✅ Add traits to class detail pages (if different from features)
6. ✅ Test with various classes
7. ✅ Verify all pages work
8. ✅ Commit changes

**Success Criteria:**
- Class counters displayed
- Level progression table displayed
- All tests passing

---

### Phase D: Missing Data - Low Priority (2-3 hours)

**Goal:** Complete all remaining missing fields

**Tasks:**
1. ✅ Add `prerequisites_text` to feats (show alongside structured prerequisites)
2. ✅ Add `proficiencies` to feats accordion
3. ✅ Add `conditions` to feats accordion
4. ✅ Add `detail` to items (append after description)
5. ✅ Add `prerequisites` to items accordion (if present)
6. ✅ Final polish and consistency check
7. ✅ Update documentation
8. ✅ Commit changes

**Success Criteria:**
- ALL API fields displayed across all entities
- Documentation updated
- All tests passing

---

## 📊 Success Metrics

**Before:**
- 7 pages with inconsistent structures
- ~10 missing API data fields
- Disconnected image layout
- No standard template

**After:**
- 7 pages following identical template
- 100% API field coverage
- Integrated image layout (2/3-1/3)
- Clear template for future pages
- 4 new tested components
- Full test coverage maintained

---

## 🎯 Testing Strategy

### Component Tests (TDD Required)

**For Each New Component:**
1. ✅ Write tests FIRST (RED phase)
2. ✅ Implement component (GREEN phase)
3. ✅ Refactor if needed (keep tests GREEN)

**Test Coverage Requirements:**
- Props render correctly
- Conditional rendering works (v-if logic)
- Missing/optional data handled gracefully
- Layout classes applied correctly
- Slot content displays
- Edge cases (empty arrays, null values)

### Integration Tests

**For Each Updated Page:**
- Loading state displays
- Error state displays
- Entity data renders
- All accordion sections work
- Image displays when present
- Layout correct with/without image
- Mobile responsive

### Browser Verification

**Manual Testing Required:**
- All 7 pages load (HTTP 200)
- Images display correctly
- 2/3-1/3 layout works on desktop
- Stacked layout works on mobile
- All accordion items expand/collapse
- Debug panel works
- Dark mode works

---

## 🔄 Migration Path

### Existing Components to Update

1. **Deprecate `<UiEntityHeaderWithImage>`**
   - Add deprecation notice in component
   - Update all 6 usages
   - Remove component after migration complete

2. **Update Monsters Page**
   - Currently uses `<UiDetailPageHeader>` (no image)
   - Add image support via new pattern
   - Maintain existing accordion structure

3. **Standardize Always-Visible Sections**
   - Backgrounds: Keep traits/proficiencies/languages visible
   - Feats: Keep prerequisites visible
   - Others: Keep current pattern

---

## 📝 Documentation Updates

**Files to Update:**
1. `CLAUDE.md` - Add detail page template pattern
2. `docs/CURRENT_STATUS.md` - Update completion status
3. Create handover document: `docs/HANDOVER-2025-11-23-DETAIL-PAGE-STANDARDIZATION.md`
4. Update `CHANGELOG.md` with new features

---

## ⚠️ Risks and Mitigation

| Risk | Mitigation |
|------|-----------|
| **Breaking existing pages** | TDD + incremental page-by-page migration |
| **Image layout not responsive** | Test on multiple screen sizes before committing |
| **Missing API data structure** | Check API docs + sample data for each field |
| **Test failures** | Write tests first, ensure all pass before merge |
| **Scope creep** | Focus on Phase A first, B/C/D optional |

---

## 🎓 Design Decisions

### Why integrate image into description card?

**Rationale:**
- Image contextually relates to description content
- Reduces visual separation
- Better utilizes space (no empty column when no image)
- More natural reading flow

### Why not create a full wrapper component?

**Rationale:**
- Each entity has unique always-visible sections
- Accordion items differ significantly per entity
- Slot-based approach would be overly complex
- Template pattern is clearer and more maintainable

### Why prioritize monster spellcasting?

**Rationale:**
- Many iconic monsters cast spells (liches, dragons, etc.)
- Critical gameplay mechanic
- Most impactful missing feature
- User-visible value

---

## ✅ Acceptance Criteria

**Phase A Complete When:**
- [ ] All 7 pages use identical structural template
- [ ] All 7 pages use `<UiDetailDescriptionWithImage>` for description + image
- [ ] Monsters page has image integration
- [ ] `<UiEntityHeaderWithImage>` deprecated
- [ ] All tests passing (0 regressions)
- [ ] All pages verified in browser (HTTP 200)
- [ ] Code committed

**Phases B/C/D Complete When:**
- [ ] All missing API fields identified and displayed
- [ ] New components have full test coverage
- [ ] All tests passing
- [ ] Documentation updated
- [ ] Code committed

---

**Next Steps:** Proceed to Phase 5 (Worktree Setup) → Phase 6 (Implementation Plan)
