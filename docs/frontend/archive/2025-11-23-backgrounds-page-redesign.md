# Backgrounds Detail Page Redesign - Design Document

**Date**: 2025-11-23
**Status**: Approved - Ready for Implementation
**Related**: docs/HANDOVER-2025-11-23-BACKGROUNDS-DETAIL-PAGE-REDESIGN.md

---

## Problem Statement

The backgrounds detail page (`app/pages/backgrounds/[slug].vue`) is structurally inconsistent with other entity detail pages (classes, feats, races). This creates:

- **Poor UX**: Users encounter different interaction patterns across entity types
- **Maintenance burden**: Unique layouts require special handling
- **Visual bugs**: Nested random tables have double `border-l-4` styling
- **Scrollbar issues**: Tables not full-width cause horizontal scrolling

### Current Issues

1. **Non-accordion layout for traits** - Uses `<UCard>` instead of `<UAccordion>`
2. **Always-visible sections** - Proficiencies and Languages not in accordion
3. **Unique side-by-side layout** - Traits + image in flex layout (not used elsewhere)
4. **Double border bug** - Random tables nested in traits create `border-l-4` conflict
5. **Horizontal scrollbars** - Random tables not rendering full-width

---

## Design Goals

1. **Consistency**: Match the standard pattern used by classes/feats/races pages
2. **Simplicity**: Single unified accordion for all expandable sections
3. **Clean nesting**: Fix random tables border conflict with minimal complexity
4. **Mobile-first**: Responsive design that works 375px → 1440px+
5. **Maintainability**: Reduce special cases, follow established patterns

---

## Architecture

### Standard Entity Page Pattern

All entity detail pages follow this structure:

```
1. Breadcrumb navigation
2. Header with badges
3. Entity image (optional, dedicated section)
4. Description (always visible if present)
5. Single UAccordion type="multiple" with all expandable sections
6. Debug panel (dev mode)
```

### Backgrounds Page Target Structure

```vue
1. <UiBackLink> - Breadcrumb
2. <UiDetailPageHeader> - Title + "Background" badge
3. <UiDetailEntityImage> - Standalone image section (if present)
4. <UCard> - Description (if present, always visible)
5. <UAccordion type="multiple"> - ALL expandable sections:
   - Background Traits (with nested random tables)
   - Skill Proficiencies
   - Languages
   - Starting Equipment
   - Source
   - Tags
6. <JsonDebugPanel> - Debug data
```

---

## Component Changes

### Change 1: Remove Border from UiAccordionRandomTablesList

**File**: `app/components/ui/accordion/UiAccordionRandomTablesList.vue`

**Current** (lines 93-102):
```vue
<div
  v-if="tables.length > 0"
  class="p-4 space-y-6"
>
  <div
    v-for="table in tables"
    :key="table.id"
    class="space-y-2 border-l-4 pl-4"  <!-- ❌ PROBLEM -->
    :class="`border-${borderColor}`"
  >
```

**New**:
```vue
<div
  v-if="tables.length > 0"
  class="p-4 space-y-6"
>
  <div
    v-for="table in tables"
    :key="table.id"
    class="space-y-2"  <!-- ✅ Border removed -->
  >
```

**Rationale**: When random tables are nested inside `UiAccordionTraitsList`, they inherit visual grouping from the trait's `border-l-4`. Adding another border creates:
- Visual clutter (double left borders)
- Nesting hierarchy confusion
- Styling conflicts

Removing the border entirely simplifies the component and prevents future nesting issues. Random tables remain visually distinct through spacing and table styling.

**Test Impact**: Update component tests to expect no `border-l-4` or `border-${color}` classes.

---

### Change 2: Refactor Backgrounds Page Template

**File**: `app/pages/backgrounds/[slug].vue`

#### Before (Current Structure)

```vue
<!-- Lines 62-90: Traits in UCard with side-by-side layout -->
<UCard v-if="entity.traits && entity.traits.length > 0">
  <template #header>
    <h2>Background Traits</h2>
  </template>
  <div class="flex flex-col lg:flex-row gap-6">
    <div :class="imagePath ? 'lg:w-2/3' : 'w-full'">
      <UiAccordionTraitsList :traits="entity.traits" />
    </div>
    <div v-if="imagePath" class="lg:w-1/3">
      <UiDetailEntityImage :image-path="imagePath" />
    </div>
  </div>
</UCard>

<!-- Lines 93-104: Description in UCard -->
<UCard v-if="entity.description">
  <template #header>
    <h2>Description</h2>
  </template>
  <div class="prose dark:prose-invert max-w-none">
    <p>{{ entity.description }}</p>
  </div>
</UCard>

<!-- Lines 106-127: Proficiencies and Languages in separate UCards -->
<UCard v-if="entity.proficiencies">
  <template #header>
    <h2>Skill Proficiencies</h2>
  </template>
  <UiAccordionBulletList :items="entity.proficiencies" />
</UCard>

<UCard v-if="entity.languages">
  <template #header>
    <h2>Languages</h2>
  </template>
  <UiAccordionBadgeList :items="entity.languages" />
</UCard>

<!-- Lines 129-176: Small accordion with only 3 sections -->
<UAccordion :items="[equipment, source, tags]" type="multiple">
  <!-- slots -->
</UAccordion>
```

#### After (New Structure)

```vue
<!-- NEW: Dedicated image section (after header, before description) -->
<div v-if="imagePath" class="rounded-lg overflow-hidden">
  <UiDetailEntityImage
    :image-path="imagePath"
    :image-alt="`${entity.name} background illustration`"
  />
</div>

<!-- Description: Always visible (if present) -->
<UCard v-if="entity.description">
  <template #header>
    <h2 class="text-lg font-semibold text-gray-900 dark:text-gray-100">
      Description
    </h2>
  </template>
  <div class="prose dark:prose-invert max-w-none">
    <p class="whitespace-pre-line text-base text-gray-700 dark:text-gray-300 leading-relaxed">
      {{ entity.description }}
    </p>
  </div>
</UCard>

<!-- Single unified accordion with ALL sections -->
<UAccordion
  :items="[
    ...(entity.traits && entity.traits.length > 0 ? [{
      label: 'Background Traits',
      slot: 'traits',
      defaultOpen: false
    }] : []),
    ...(entity.proficiencies && entity.proficiencies.length > 0 ? [{
      label: 'Skill Proficiencies',
      slot: 'proficiencies',
      defaultOpen: false
    }] : []),
    ...(entity.languages && entity.languages.length > 0 ? [{
      label: 'Languages',
      slot: 'languages',
      defaultOpen: false
    }] : []),
    ...(entity.equipment && entity.equipment.length > 0 ? [{
      label: 'Starting Equipment',
      slot: 'equipment',
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
  <!-- Traits Slot -->
  <template
    v-if="entity.traits && entity.traits.length > 0"
    #traits
  >
    <UiAccordionTraitsList
      :traits="entity.traits"
      :show-category="true"
      border-color="purple-500"
    />
  </template>

  <!-- Proficiencies Slot -->
  <template
    v-if="entity.proficiencies && entity.proficiencies.length > 0"
    #proficiencies
  >
    <UiAccordionBulletList :items="entity.proficiencies" />
  </template>

  <!-- Languages Slot -->
  <template
    v-if="entity.languages && entity.languages.length > 0"
    #languages
  >
    <UiAccordionBadgeList
      :items="entity.languages"
      color="neutral"
    />
  </template>

  <!-- Equipment Slot -->
  <template
    v-if="entity.equipment && entity.equipment.length > 0"
    #equipment
  >
    <UiAccordionEquipmentList
      :equipment="entity.equipment"
      type="background"
    />
  </template>

  <!-- Source Slot -->
  <template
    v-if="entity.sources && entity.sources.length > 0"
    #source
  >
    <UiSourceDisplay :sources="entity.sources" />
  </template>

  <!-- Tags Slot -->
  <template
    v-if="entity.tags && entity.tags.length > 0"
    #tags
  >
    <UiTagsDisplay :tags="entity.tags" />
  </template>
</UAccordion>
```

**Key Changes**:

1. **Image moved** from side-by-side flex layout to dedicated section (before description)
2. **Description moved** from line 93 to earlier position (after image)
3. **Traits moved** from `<UCard>` to accordion slot
4. **Proficiencies moved** from `<UCard>` to accordion slot
5. **Languages moved** from `<UCard>` to accordion slot
6. **All sections** merged into single accordion with `type="multiple"`

**Spread Operator Pattern**: The `...(condition ? [item] : [])` pattern builds the items array declaratively. Sections without data don't appear in the accordion at all—cleaner than v-if throughout the template.

---

## Testing Strategy

### Unit Tests (TDD - Write FIRST)

**Test 1: UiAccordionRandomTablesList.test.ts**
```typescript
describe('UiAccordionRandomTablesList', () => {
  it('should NOT render border-l-4 class', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })
    const tableWrapper = wrapper.find('[data-testid="random-table-wrapper"]')
    expect(tableWrapper.classes()).not.toContain('border-l-4')
    expect(tableWrapper.classes()).not.toContain('pl-4')
  })

  it('should render spacing classes', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })
    expect(wrapper.find('.space-y-6').exists()).toBe(true)
    expect(wrapper.find('.space-y-2').exists()).toBe(true)
  })
})
```

**Test 2: backgrounds/slug.test.ts**
```typescript
describe('Backgrounds Detail Page', () => {
  it('should render single UAccordion with type="multiple"', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage)
    const accordions = wrapper.findAllComponents({ name: 'UAccordion' })
    expect(accordions).toHaveLength(1) // Only ONE accordion
    expect(accordions[0].props('type')).toBe('multiple')
  })

  it('should render traits in accordion slot (not UCard)', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage)
    // Traits should be in accordion, not standalone UCard
    const traitsCard = wrapper.find('[data-testid="traits-card"]')
    expect(traitsCard.exists()).toBe(false)
  })

  it('should render image before description', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage)
    const image = wrapper.findComponent(UiDetailEntityImage)
    const description = wrapper.find('[data-testid="description-card"]')

    // Image DOM position should be before description
    const imageDomIndex = wrapper.element.indexOf(image.element)
    const descDomIndex = wrapper.element.indexOf(description.element)
    expect(imageDomIndex).toBeLessThan(descDomIndex)
  })

  it('should render description outside accordion', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage)
    const accordion = wrapper.findComponent({ name: 'UAccordion' })
    const description = wrapper.find('[data-testid="description-card"]')

    // Description should not be inside accordion
    expect(accordion.element.contains(description.element)).toBe(false)
  })
})
```

### Visual Regression Testing

**Manual Tests**:

1. **Test URL**: http://localhost:3000/backgrounds/charlatan
   - ✅ Single accordion with 6 sections
   - ✅ Traits contain random tables (no double borders)
   - ✅ No horizontal scrollbars
   - ✅ Image appears before description
   - ✅ Description always visible

2. **Compare with other entity pages**:
   - `/classes/wizard` - Same accordion pattern?
   - `/feats/grappler` - Same image placement?
   - `/races/elf` - Same description treatment?

3. **Mobile responsive**:
   - 375px (mobile) - Accordion stacks, image full-width
   - 768px (tablet) - Same as mobile
   - 1440px (desktop) - Full layout

4. **Dark mode**: Test light/dark mode toggle

### Performance Testing

- **Accordion expansion**: Should be smooth (no layout shift)
- **Image loading**: Lazy load 512px variant
- **Random tables**: No horizontal scrolling

---

## Implementation Phases

### Phase 1: Fix Random Tables Border (TDD)

**Steps**:
1. ✅ **RED**: Write test expecting no `border-l-4` in UiAccordionRandomTablesList
2. ✅ **GREEN**: Remove `border-l-4 pl-4` from component template (line 100)
3. ✅ **VERIFY**: Run full test suite (`npm run test`)
4. ✅ **COMMIT**: `refactor: Remove border from UiAccordionRandomTablesList for clean nesting`

**Files Modified**:
- `app/components/ui/accordion/UiAccordionRandomTablesList.vue`
- `tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts`

---

### Phase 2: Refactor Backgrounds Page (TDD)

**Steps**:
1. ✅ **RED**: Write tests for new accordion structure
2. ✅ **GREEN**: Restructure template:
   - Move image to dedicated section (after header)
   - Move description up (after image)
   - Create single accordion with all sections
   - Move traits/proficiencies/languages into accordion slots
3. ✅ **VERIFY**: Run tests + manual visual check
4. ✅ **COMMIT**: `refactor: Standardize backgrounds detail page to match entity pattern`

**Files Modified**:
- `app/pages/backgrounds/[slug].vue`
- `tests/pages/backgrounds/slug.test.ts` (if exists, otherwise create)

---

## Success Criteria

### Before (Current State)
- ❌ Traits in standalone UCard (always visible)
- ❌ Proficiencies in standalone UCard (always visible)
- ❌ Languages in standalone UCard (always visible)
- ❌ Image in unique side-by-side flex layout
- ❌ Random tables have double `border-l-4`
- ❌ Horizontal scrollbars on random tables
- ❌ Inconsistent with classes/feats/races pages

### After (Target State)
- ✅ All expandable content in single accordion
- ✅ Accordion `type="multiple"` (multiple sections can be open)
- ✅ Image placement matches feats page (dedicated section)
- ✅ Description always visible (like classes page)
- ✅ Random tables full-width, no horizontal scroll
- ✅ No double borders (clean nesting)
- ✅ Consistent with all other entity detail pages
- ✅ All tests passing (unit + visual)
- ✅ Mobile responsive (375px → 1440px+)

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Breaking existing tests | High | Write new tests first (TDD), update as needed |
| Visual inconsistency with other pages | Medium | Manual comparison with classes/feats pages |
| Random tables display issues | Medium | Test with Charlatan background (has tables) |
| Mobile layout breaks | Medium | Test at 375px/768px/1440px breakpoints |
| Accordion state management | Low | Use `type="multiple"` (existing pattern) |

---

## Dependencies

### Completed Work
- Accordion table normalization (2025-11-23)
- UiAccordionDataTable base component
- All 829 tests passing

### No Breaking Changes
- All components remain backward compatible
- Only template structure changes (no props/API changes)

---

## Rollback Plan

If critical issues arise:

1. **Git revert** to commit before refactoring
2. **Keep border fix** for UiAccordionRandomTablesList (no downside)
3. **Delay page refactor** if needed (border fix is independent)

---

## Related Documentation

- **Handover**: `docs/HANDOVER-2025-11-23-BACKGROUNDS-DETAIL-PAGE-REDESIGN.md`
- **Gold Standard**: `app/pages/classes/[slug].vue` (reference implementation)
- **Component Docs**: `docs/3D-DICE-IMPLEMENTATION.md` (entity image usage)
- **Test Patterns**: `tests/helpers/cardBehavior.ts`

---

## Approval

**Approved by**: User (2025-11-23)
**Design Decision**: Remove border entirely from UiAccordionRandomTablesList
**Implementation Approach**: Two-phase TDD refactoring

**Ready for implementation**: ✅ Yes

---

**Next Steps**: Create implementation plan using `superpowers:writing-plans` skill.
