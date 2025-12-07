# Backgrounds Quick Stats Panel - Design Document

**Date**: 2025-11-24
**Status**: Approved for Implementation
**Pattern**: 2/3 Quick Stats + 1/3 Image Grid Layout

---

## Problem Statement

The backgrounds detail page currently displays a **full-width image** (lines 61-70 of `app/pages/backgrounds/[slug].vue`), which is too large and inconsistent with other entity detail pages (classes, races, feats).

**Current Issues**:
- ✗ Image is full-width (wasteful use of space)
- ✗ No quick stats summary at top of page
- ✗ Inconsistent with classes/races pages which use 2/3 stats + 1/3 image grid
- ✗ Duplicate logic between BackgroundCard and detail page

**User Request**: "The image is now full-width, which is waaay to large. We should adapt it to the way it is on the other detail pages. Image on the right (1/3) width, and a quick stats panel (2/3) to the left of it."

---

## Solution Overview

**Adopt the 2/3 quick stats + 1/3 image grid pattern** used by classes and races detail pages:

1. **Create reusable composable**: `useBackgroundStats` - Extract proficiency/language/equipment logic from BackgroundCard
2. **Add quick stats panel**: Display actual proficiency/language names (not just counts)
3. **Resize image**: Move to 1/3 width column (right side)
4. **Grid layout**: Side-by-side on desktop, stacked on mobile

**Visual Consistency**: Matches the exact pattern from:
- `app/pages/classes/[slug].vue` (lines 79-100)
- `app/pages/races/[slug].vue` (lines 97-100)

---

## Design Decisions

### Approach Selected: **Expand Detail** (Show Actual Content)

**What to display in quick stats**:
- **Skill Proficiencies**: Actual skill names (e.g., "Deception, Sleight of Hand")
- **Languages**: Actual language names (e.g., "Common, Thieves' Cant")
- **Tool Proficiencies**: Actual tool names (e.g., "Disguise Kit, Forgery Kit")
- **Starting Equipment**: Combined count + gold (e.g., "5 items + 15 gp")

**Why this approach**:
- Players want to know *which* proficiencies they get, not just counts
- Consistent with other pages (classes show "Primary Ability: Strength", not "1 Primary Ability")
- Backgrounds typically have small, fixed sets (2 skills, 1-2 languages) that fit in the space
- More useful at a glance than forcing users to expand accordions

**Alternatives Considered**:
1. **Mirror the Card** - Show counts like "2 Languages" (too vague)
2. **Feature-First** - Highlight background feature (redundant with page header badge)

---

## Architecture

### Components Involved

**New Component**:
- `app/composables/useBackgroundStats.ts` - Business logic composable

**Existing Components** (reused):
- `app/components/ui/detail/UiDetailQuickStatsCard.vue` - Stats display card
- `app/components/ui/detail/UiDetailStandaloneImage.vue` - Image wrapper

**Updated Components**:
- `app/pages/backgrounds/[slug].vue` - Add grid layout
- `app/components/background/BackgroundCard.vue` - Migrate to use composable (reduces duplication)

### Data Flow

```
Background Entity (from API)
        ↓
useBackgroundStats(entity)
        ↓
Computed Stats (skills, tools, languages, equipment, gold)
        ↓
UiDetailQuickStatsCard (displays stats with icons)
```

---

## Composable Design - `useBackgroundStats`

**Location**: `app/composables/useBackgroundStats.ts`

**Purpose**: Extract proficiency/language/equipment parsing logic from BackgroundCard into a reusable, testable composable.

### API Signature

```typescript
export function useBackgroundStats(background: Ref<Background | null>) {
  return {
    skillProficiencies: ComputedRef<string[]>,      // e.g., ["Deception", "Sleight of Hand"]
    toolProficiencies: ComputedRef<string[]>,       // e.g., ["Thieves' Tools", "Disguise Kit"]
    languages: ComputedRef<string[]>,               // e.g., ["Common", "Thieves' Cant"]
    equipmentCount: ComputedRef<number>,            // e.g., 5 (excluding gold)
    startingGold: ComputedRef<number | null>        // e.g., 15 (gold pieces)
  }
}
```

### Implementation Details

**Skill Proficiencies**:
```typescript
const skillProficiencies = computed(() => {
  if (!background.value?.proficiencies) return []

  return background.value.proficiencies
    .filter(p => p.proficiency_type === 'skill')
    .map(p => p.skill?.name)
    .filter(Boolean) as string[]
})
```

**Tool Proficiencies**:
```typescript
const toolProficiencies = computed(() => {
  if (!background.value?.proficiencies) return []

  return background.value.proficiencies
    .filter(p => p.proficiency_type === 'tool')
    .map(p => p.tool?.name)
    .filter(Boolean) as string[]
})
```

**Languages**:
```typescript
const languages = computed(() => {
  if (!background.value?.languages) return []

  return background.value.languages
    .map(l => l.name)
    .filter(Boolean) as string[]
})
```

**Equipment Count** (excluding gold):
```typescript
const equipmentCount = computed(() => {
  if (!background.value?.equipment) return 0

  // Exclude gold pieces from count
  return background.value.equipment.filter(
    eq => eq.item_id !== ITEM_ID_GOLD_PIECE
  ).length
})
```

**Starting Gold**:
```typescript
const startingGold = computed(() => {
  if (!background.value?.equipment) return null

  const goldItem = background.value.equipment.find(
    eq => eq.item_id === ITEM_ID_GOLD_PIECE
  )

  return goldItem?.quantity || null
})
```

---

## Page Layout Changes

### Current Layout (lines 61-70)

```vue
<!-- Entity Image (dedicated section, before description) -->
<div
  v-if="imagePath"
  class="rounded-lg overflow-hidden"
>
  <UiDetailEntityImage
    :image-path="imagePath"
    :image-alt="`${entity.name} background illustration`"
  />
</div>
```

### New Layout (2/3 + 1/3 Grid)

```vue
<!-- Quick Stats (2/3) + Image (1/3) Side-by-Side -->
<div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
  <!-- Quick Stats - 2/3 width on large screens -->
  <div class="lg:col-span-2">
    <UiDetailQuickStatsCard
      :columns="2"
      :stats="statsForDisplay"
    />
  </div>

  <!-- Standalone Image - 1/3 width on large screens -->
  <div class="lg:col-span-1">
    <UiDetailStandaloneImage
      v-if="imagePath"
      :image-path="imagePath"
      :image-alt="`${entity.name} background illustration`"
    />
  </div>
</div>
```

### Stats Computation

```typescript
const {
  skillProficiencies,
  toolProficiencies,
  languages,
  equipmentCount,
  startingGold
} = useBackgroundStats(entity)

const statsForDisplay = computed(() => {
  const stats = []

  // Skills (always show if present)
  if (skillProficiencies.value.length > 0) {
    stats.push({
      icon: 'i-heroicons-academic-cap',
      label: 'Skill Proficiencies',
      value: skillProficiencies.value.join(', ')
    })
  }

  // Languages (show actual names)
  if (languages.value.length > 0) {
    stats.push({
      icon: 'i-heroicons-language',
      label: 'Languages',
      value: languages.value.join(', ')
    })
  }

  // Tool Proficiencies (show actual names)
  if (toolProficiencies.value.length > 0) {
    stats.push({
      icon: 'i-heroicons-wrench',
      label: 'Tool Proficiencies',
      value: toolProficiencies.value.join(', ')
    })
  }

  // Equipment + Gold combined
  if (equipmentCount.value > 0 || startingGold.value) {
    const parts = []
    if (equipmentCount.value > 0) parts.push(`${equipmentCount.value} items`)
    if (startingGold.value) parts.push(`${startingGold.value} gp`)

    stats.push({
      icon: 'i-heroicons-shopping-bag',
      label: 'Starting Equipment',
      value: parts.join(' + ')
    })
  }

  return stats
})
```

---

## Example Output

### "Charlatan" Background

**Quick Stats Panel**:
- 📚 **Skill Proficiencies**: Deception, Sleight of Hand
- 🗣️ **Languages**: Common, Thieves' Cant
- 🔧 **Tool Proficiencies**: Disguise Kit, Forgery Kit
- 🛒 **Starting Equipment**: 5 items + 15 gp

**Image**: Charlatan illustration at 1/3 width (right side)

### "Acolyte" Background

**Quick Stats Panel**:
- 📚 **Skill Proficiencies**: Insight, Religion
- 🗣️ **Languages**: Common, Celestial
- 🛒 **Starting Equipment**: 3 items + 25 gp

*(No tools - section hidden)*

---

## Edge Cases & Data Handling

### Missing Data Scenarios

| Scenario | Behavior |
|----------|----------|
| No languages | Skip languages stat (some backgrounds give choice, not fixed) |
| No tool proficiencies | Skip tools stat (some backgrounds only give skills) |
| No equipment/gold | Skip equipment stat (unlikely but possible) |
| No image | `UiDetailStandaloneImage` has `v-if`, stats take full width |
| Empty proficiencies array | Return empty array, stat not displayed |

### Language Choices

Some backgrounds grant "Choose 1 language" (stored as text in description, not in `languages[]` array).

**Handling**: Only show languages that exist in the `languages[]` array. If empty, skip the languages stat entirely (better than showing "0 Languages").

### Gold Piece Detection

Use existing constant `ITEM_ID_GOLD_PIECE` from `~/constants/items` to identify gold in equipment array.

---

## Testing Strategy

### TDD Mandate

**Write tests FIRST (RED phase)**, then implement (GREEN phase), following project's TDD mandate.

### Composable Tests

**File**: `tests/composables/useBackgroundStats.test.ts`

**Test Cases**:
1. ✅ Extracts skill proficiency names correctly
2. ✅ Extracts tool proficiency names correctly
3. ✅ Extracts language names correctly
4. ✅ Counts equipment items (excluding gold)
5. ✅ Finds starting gold from equipment array
6. ✅ Handles missing data gracefully (empty arrays, null values)
7. ✅ Handles backgrounds with no tools
8. ✅ Handles backgrounds with no languages
9. ✅ Handles backgrounds with only gold (no equipment items)
10. ✅ Returns empty arrays when background is null

### Page Tests

**File**: `tests/pages/backgrounds/slug.test.ts`

**Test Cases**:
1. ✅ Renders quick stats panel with correct grid layout
2. ✅ Renders image at 1/3 width (lg:col-span-1)
3. ✅ Shows skill proficiencies in stats panel
4. ✅ Shows languages in stats panel
5. ✅ Shows tool proficiencies when present
6. ✅ Shows equipment + gold combined
7. ✅ Hides stats sections when data missing
8. ✅ Grid uses lg:col-span-2 for stats
9. ✅ Image is hidden when imagePath is null
10. ✅ Stats take full width when no image

### BackgroundCard Tests

**File**: `tests/components/background/BackgroundCard.test.ts`

**Updates Required**:
- Update tests to verify component uses `useBackgroundStats` composable
- Ensure card behavior unchanged after migration
- All existing tests should continue passing

---

## Migration & Refactoring Plan

### Phase 1: Create Composable (TDD)
1. Write failing tests for `useBackgroundStats`
2. Implement composable to pass tests
3. Verify all composable tests pass

### Phase 2: Update BackgroundCard
1. Import `useBackgroundStats` in BackgroundCard
2. Replace inline computed properties with composable
3. Remove duplicate logic (lines 14-89)
4. Verify all BackgroundCard tests still pass
5. Visual test: Card appearance unchanged

### Phase 3: Update Detail Page
1. Write failing tests for quick stats panel
2. Add grid layout to backgrounds/[slug].vue
3. Use `useBackgroundStats` composable
4. Add `statsForDisplay` computed property
5. Replace full-width image with grid layout
6. Verify all page tests pass

### Phase 4: Visual Regression Testing
- Test with multiple backgrounds: Charlatan, Acolyte, Sage, Criminal
- Test mobile responsive (grid stacks vertically)
- Test with/without images
- Test with missing data (no tools, no languages, etc.)

---

## Breaking Changes

**None** - This is purely additive:
- BackgroundCard behavior unchanged (same logic, different structure)
- Detail page adds new section, doesn't remove existing content
- All existing functionality preserved

---

## Success Criteria

### Before (Current State)
- ❌ Full-width image (too large)
- ❌ No quick stats summary
- ❌ Duplicate logic in BackgroundCard and (future) detail page
- ❌ Inconsistent with classes/races pages

### After (Target State)
- ✅ Image at 1/3 width (consistent with other pages)
- ✅ Quick stats panel at 2/3 width showing actual proficiency/language names
- ✅ Reusable composable (no duplicate logic)
- ✅ Grid layout responsive (stacks on mobile)
- ✅ All tests passing (composable + page + card)
- ✅ Visual consistency across all backgrounds
- ✅ Matches classes/races detail page pattern exactly

---

## Files to Create/Modify

### New Files
- `app/composables/useBackgroundStats.ts` - Business logic composable
- `tests/composables/useBackgroundStats.test.ts` - Composable tests

### Modified Files
- `app/pages/backgrounds/[slug].vue` - Add grid layout + quick stats
- `app/components/background/BackgroundCard.vue` - Migrate to composable
- `tests/pages/backgrounds/slug.test.ts` - Add quick stats panel tests
- `tests/components/background/BackgroundCard.test.ts` - Update for composable usage
- `CHANGELOG.md` - Document changes

---

## Related Work

**Previous Session** (2025-11-23):
- Backgrounds detail page redesign (accordion standardization)
- All sections moved into single accordion
- Random tables border fix

**Pattern Reference**:
- `app/pages/classes/[slug].vue` (lines 79-100) - Gold standard for grid layout
- `app/pages/races/[slug].vue` (lines 97-100) - Same pattern

---

## Next Steps

1. ✅ Design approved (this document)
2. 🔄 Create implementation plan (use `superpowers:writing-plans`)
3. ⏳ Execute with TDD (follow migration plan)
4. ⏳ Test on multiple backgrounds
5. ⏳ Commit and update CHANGELOG

---

**Design Status**: ✅ Approved
**Next Agent**: Use `superpowers:writing-plans` to create detailed implementation plan, then execute with TDD mandate.
