# Detail Pages Refactoring Design
**Date:** 2025-11-24
**Status:** Approved for Implementation
**Approach:** Parallel Subagent Execution (4 work packages)

---

## Executive Summary

Refactor 7 entity detail pages to eliminate inconsistencies and improve code quality. All pages currently work well (8.5/10), this is **polish, not fixes**. Implementing **7 refactoring items** organized into **4 independent work packages** executed by parallel subagents.

**Total Estimated Time:** 1-2 hours (vs 3-5 hours sequential)
**Risk Level:** Low (cosmetic changes, existing patterns are good)
**Impact:** Improved consistency, reduced duplication, better UX

---

## Design Principles

1. **TDD Mandatory** - Tests written FIRST for all changes (per CLAUDE.md)
2. **Visual Parity** - No visual changes, only code standardization
3. **Independent Work Packages** - Minimize merge conflicts
4. **Existing Test Coverage** - 97.5% pass rate must be maintained
5. **Incremental Verification** - Each subagent verifies before completion

---

## Work Package 1: Image Standardization

### Refactoring Items
- #1: Standardize Image Components
- #4: Standardize Grid Pattern

### Objective
Replace `UiDetailStandaloneImage` with `UiDetailEntityImage` and unify grid layouts.

### Files to Modify
- `app/pages/items/[slug].vue`
- `app/pages/races/[slug].vue`
- `app/pages/classes/[slug].vue`
- `app/pages/backgrounds/[slug].vue`
- `app/pages/spells/[slug].vue` (grid pattern only)
- `app/pages/monsters/[slug].vue` (grid pattern + remove UCard wrapper)

### Changes

**Before (Items/Races/Classes/Backgrounds):**
```vue
<div class="flex flex-col lg:flex-row gap-6">
  <div class="lg:w-2/3">
    <UiDetailQuickStatsCard />
  </div>
  <div class="lg:w-1/3">
    <UiDetailStandaloneImage />
  </div>
</div>
```

**After (All pages):**
```vue
<div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
  <div class="lg:col-span-2">
    <UiDetailQuickStatsCard />
  </div>
  <div class="lg:col-span-1">
    <UiDetailEntityImage
      :image-path="imagePath"
      :image-alt="`${entity.name} illustration`"
    />
  </div>
</div>
```

**Monsters special case:**
- Remove `<UCard>` wrapper around image
- Apply same grid pattern

### TDD Approach
1. Write characterization tests for current image display
2. Test image component renders with correct props
3. Test grid layout responsive behavior (1 col mobile, 3 col desktop)
4. Test image path generation
5. Run tests (should FAIL initially)
6. Implement changes
7. Run tests (should PASS)
8. Visual verification in browser

### Success Criteria
- ✅ All 6 pages use `UiDetailEntityImage`
- ✅ All 6 pages use `grid grid-cols-1 lg:grid-cols-3 gap-6`
- ✅ Tests pass (no regressions)
- ✅ TypeScript compiles
- ✅ Visual parity maintained

---

## Work Package 2: Content Standardization

### Refactoring Items
- #2: Fix Accordion Slot Naming
- #3: Standardize Description Card

### Objective
Fix inconsistent kebab-case in Items accordion and standardize description rendering.

### Files to Modify
- `app/pages/items/[slug].vue` (slot naming)
- `app/pages/spells/[slug].vue` (description card)
- `app/pages/backgrounds/[slug].vue` (description card)
- `app/pages/feats/[slug].vue` (description card)
- `app/pages/monsters/[slug].vue` (description card)

### Changes

**1. Items Accordion Slot Naming**

Before:
```vue
<template #random_tables>...</template>
<template #saving_throws>...</template>
```

After:
```vue
<template #random-tables>...</template>
<template #saving-throws>...</template>
```

**2. Description Cards**

Before (manual UCard):
```vue
<UCard>
  <div class="prose dark:prose-invert max-w-none">
    <p class="whitespace-pre-line leading-relaxed">
      {{ spell.description }}
    </p>
  </div>
</UCard>
```

After (component):
```vue
<UiDetailDescriptionCard
  v-if="spell.description"
  :description="spell.description"
/>
```

**Note:** Classes page has special logic (first trait fallback) - keep that logic intact.

### TDD Approach
1. Test accordion slots render with kebab-case names
2. Test description card component displays content
3. Test prose styling applied correctly
4. Test whitespace-pre-line formatting preserved
5. Test missing description handled gracefully
6. Run tests (FAIL → implement → PASS)

### Success Criteria
- ✅ Items page uses kebab-case for all slot names
- ✅ 4 pages use `UiDetailDescriptionCard`
- ✅ Classes page special logic preserved
- ✅ Tests pass, visual parity maintained

---

## Work Package 3: Feats Quick Stats Enhancement

### Refactoring Item
- #5: Add Quick Stats to Feats

### Objective
Add quick stats card to Feats page for visual consistency.

### Files to Modify
- `app/pages/feats/[slug].vue`

### Changes

**Add Quick Stats Section**

Insert after image, before description:

```vue
<script setup>
const quickStatsForDisplay = computed(() => {
  if (!feat.value) return []

  return [
    {
      icon: 'i-heroicons-bolt',
      label: 'Type',
      value: 'Feat'
    },
    {
      icon: 'i-heroicons-check-badge',
      label: 'Prerequisites',
      value: feat.value.prerequisites?.length > 0 ? 'Yes' : 'None'
    }
  ]
})
</script>

<template>
  <!-- After image section -->
  <UiDetailQuickStatsCard
    :columns="2"
    :stats="quickStatsForDisplay"
  />
</template>
```

### TDD Approach
1. Write test for `quickStatsForDisplay` computed property
2. Test returns correct structure (icon, label, value)
3. Test "Type" always shows "Feat"
4. Test prerequisites shows "Yes" when array has items
5. Test prerequisites shows "None" when empty/undefined
6. Test card renders with 2 columns
7. Run tests (FAIL → implement → PASS)

### Success Criteria
- ✅ Feats page has quick stats card
- ✅ Shows "Type: Feat" and prerequisite status
- ✅ Visual consistency with other entity pages
- ✅ Tests pass

---

## Work Package 4: Monsters Accordion & Conditions

### Refactoring Items
- #6: Refactor Monsters to Use Accordion
- #7: Consolidate Condition Display Components

### Objective
Move Monsters standalone cards into accordion and consolidate conditions component.

### Files to Modify
- `app/pages/monsters/[slug].vue` (accordion refactor)
- `app/pages/races/[slug].vue` (conditions component)

### Changes

**1. Monsters Accordion Structure**

Before (standalone cards):
```vue
<UiAccordionTraits /> <!-- Standalone -->
<UiAccordionActions /> <!-- Standalone -->
<UCard><!-- Legendary Actions --></UCard>
<UCard><!-- Spellcasting --></UCard>
<UiModifiersDisplay />
<UCard><!-- Conditions --></UCard>
<UiSourceDisplay />
```

After (accordion):
```vue
<UAccordion type="multiple" :items="accordionItems">
  <template #traits>
    <UiAccordionTraitsList :traits="monster.traits" />
  </template>

  <template #actions>
    <UiAccordionActions
      :actions="regularActions"
      title="Actions"
    />
  </template>

  <template #legendary>
    <UiAccordionActions
      v-if="monster.legendary_actions?.length"
      :actions="monster.legendary_actions"
      title="Legendary Actions"
    />
  </template>

  <template #spellcasting>
    <div v-if="monster.spellcasting_description" class="prose dark:prose-invert">
      <p class="whitespace-pre-line">{{ monster.spellcasting_description }}</p>
    </div>
  </template>

  <template #modifiers>
    <UiModifiersDisplay :modifiers="monster.modifiers" />
  </template>

  <template #conditions>
    <UiAccordionConditions :conditions="monster.conditions" />
  </template>

  <template #source>
    <UiSourceDisplay :source="monster.source" />
  </template>
</UAccordion>
```

**2. Races Conditions Component**

Before (manual template):
```vue
<template #conditions>
  <div class="space-y-3">
    <div v-for="condition in race.conditions">
      <div class="flex items-start gap-3">
        <UBadge>{{ condition.name }}</UBadge>
        <p>{{ condition.description }}</p>
      </div>
    </div>
  </div>
</template>
```

After (component):
```vue
<template #conditions>
  <UiAccordionConditions :conditions="race.conditions" />
</template>
```

### TDD Approach
1. Test accordion renders all slots
2. Test each slot displays correct data
3. Test v-if conditions work (legendary actions, spellcasting)
4. Test Races conditions component integration
5. Test progressive disclosure (collapsed by default)
6. Run tests (FAIL → implement → PASS)

### Success Criteria
- ✅ Monsters uses accordion pattern
- ✅ All data sections accessible in accordion
- ✅ Races uses `UiAccordionConditions` component
- ✅ Tests pass, data display complete
- ✅ UX improved (progressive disclosure)

---

## Coordination Plan

### Parallel Execution
All 4 work packages execute simultaneously:
- **Package 1:** Image Standardization (6 files)
- **Package 2:** Content Standardization (5 files)
- **Package 3:** Feats Enhancement (1 file)
- **Package 4:** Monsters & Conditions (2 files)

**Total:** 14 file modifications across 4 subagents

### Merge Strategy
- Each package operates on mostly different files
- Only potential overlap: Monsters page (Packages 1, 2, 4)
- Resolution: Coordinate changes, combine in single file edit

### Verification Steps
After all subagents complete:

1. **Full Test Suite**
   ```bash
   docker compose exec nuxt npm run test
   ```
   Target: 97.5%+ pass rate maintained

2. **TypeScript Check**
   ```bash
   docker compose exec nuxt npm run typecheck
   ```
   Target: 0 errors

3. **Browser Verification**
   - Test all 7 detail pages (Spells, Items, Races, Classes, Backgrounds, Feats, Monsters)
   - Verify HTTP 200 responses
   - Verify visual parity (no layout changes)
   - Test dark mode
   - Test mobile responsive (375px)

4. **Git Commits**
   - Option A: 4 separate commits (one per package)
   - Option B: 1 combined commit with detailed message

---

## Risk Assessment

### Low Risk Items
- Image component swap (well-tested component)
- Grid pattern change (CSS only)
- Accordion slot naming (simple find-replace)
- Description card standardization (existing component)

### Medium Risk Items
- Monsters accordion refactor (most complex change)
- Feats quick stats (new computed property)

### Mitigation
- TDD ensures tests catch regressions
- Existing 97.5% test coverage provides safety net
- Visual verification catches UI issues
- Independent work packages minimize blast radius

---

## Success Metrics

### Code Quality
- ✅ Single image component (`UiDetailEntityImage`)
- ✅ Consistent grid pattern across all pages
- ✅ Kebab-case accordion slot naming
- ✅ Single description card component
- ✅ Feats has quick stats
- ✅ Monsters uses accordion
- ✅ Races uses conditions component

### Testing
- ✅ All existing tests pass
- ✅ New tests added for new behavior
- ✅ TypeScript compiles with 0 errors
- ✅ No ESLint warnings

### Visual
- ✅ All 7 detail pages maintain visual parity
- ✅ Responsive behavior preserved
- ✅ Dark mode works
- ✅ Images display correctly

---

## Implementation Timeline

**Estimated Duration:** 1-2 hours total

- **Package 1:** 20-30 minutes (6 files, straightforward swaps)
- **Package 2:** 15-20 minutes (5 files, simple changes)
- **Package 3:** 10-15 minutes (1 file, small addition)
- **Package 4:** 30-40 minutes (2 files, most complex)
- **Verification:** 10-15 minutes (tests, browser, commit)

---

## Next Steps

1. ✅ Design approved
2. ⏭️ Spawn 4 parallel subagents
3. ⏭️ Execute refactorings with TDD
4. ⏭️ Coordinate and merge results
5. ⏭️ Verification and commit
6. ⏭️ Update CHANGELOG.md
7. ⏭️ Document in handover

---

**End of Design Document**
