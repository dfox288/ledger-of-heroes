# Backgrounds Detail Page Redesign - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Standardize backgrounds detail page to match the consistent accordion pattern used by classes/feats/races pages, fixing visual bugs and improving UX.

**Architecture:** Two-phase refactoring: (1) Remove border-l-4 from UiAccordionRandomTablesList to fix nested border conflict, (2) Consolidate all expandable sections (traits, proficiencies, languages, equipment, source, tags) into single UAccordion with type="multiple".

**Tech Stack:** Nuxt 4, Vue 3 Composition API, NuxtUI 4, TypeScript, Vitest, @nuxt/test-utils

---

## Prerequisites

**Verify before starting:**

```bash
# 1. Backend running
curl http://localhost:8080/api/v1/backgrounds/charlatan

# 2. Frontend dev server running
curl http://localhost:3000/backgrounds

# 3. Tests passing
docker compose exec nuxt npm run test
```

**Expected:** All commands succeed with HTTP 200 responses and tests passing.

---

## Phase 1: Remove Border from UiAccordionRandomTablesList

### Task 1.1: Write failing test for border removal

**Files:**
- Modify: `tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts`

**Step 1: Check if test file exists**

```bash
ls -la tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts
```

**Expected:** File exists OR needs to be created

**Step 2: Add test for no border-l-4 class**

If file exists, add this test. If not, create the file with full test suite:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionRandomTablesList from '~/components/ui/accordion/UiAccordionRandomTablesList.vue'

describe('UiAccordionRandomTablesList', () => {
  const mockTables = [
    {
      id: '1',
      table_name: 'Personality Traits',
      dice_type: 'd8',
      description: 'Choose or roll for personality traits',
      entries: [
        {
          id: '1',
          roll_min: 1,
          roll_max: 1,
          result_text: 'I idolize a particular hero'
        },
        {
          id: '2',
          roll_min: 2,
          roll_max: 2,
          result_text: 'I can find common ground'
        }
      ]
    }
  ]

  it('should NOT render border-l-4 class on table wrapper', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    // Find the wrapper div for each table
    const tableWrappers = wrapper.findAll('[class*="space-y-2"]')
    expect(tableWrappers.length).toBeGreaterThan(0)

    // Check that none have border-l-4 class
    tableWrappers.forEach(tableWrapper => {
      expect(tableWrapper.classes()).not.toContain('border-l-4')
      expect(tableWrapper.classes()).not.toContain('pl-4')
    })
  })

  it('should NOT render border color classes', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: {
        tables: mockTables,
        borderColor: 'purple-500'
      }
    })

    const html = wrapper.html()
    expect(html).not.toContain('border-purple-500')
    expect(html).not.toContain('border-primary-500')
  })

  it('should still render spacing classes', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    // Outer wrapper should have p-4 and space-y-6
    const outerWrapper = wrapper.find('[class*="space-y-6"]')
    expect(outerWrapper.exists()).toBe(true)
    expect(outerWrapper.classes()).toContain('p-4')

    // Inner wrappers should have space-y-2
    const innerWrappers = wrapper.findAll('[class*="space-y-2"]')
    expect(innerWrappers.length).toBeGreaterThan(0)
  })

  it('should render table name and dice type', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    expect(wrapper.text()).toContain('Personality Traits')
    expect(wrapper.text()).toContain('(d8)')
  })

  it('should render table entries via UiAccordionDataTable', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    const dataTable = wrapper.findComponent({ name: 'UiAccordionDataTable' })
    expect(dataTable.exists()).toBe(true)
  })
})
```

**Step 3: Run test to verify it fails**

```bash
docker compose exec nuxt npm run test -- UiAccordionRandomTablesList
```

**Expected:** Tests FAIL with errors about border-l-4 class being present

**Step 4: Commit the failing test**

```bash
git add tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts
git commit -m "test: Add failing tests for UiAccordionRandomTablesList border removal

- Tests expect no border-l-4 class
- Tests expect no border color classes
- Tests verify spacing classes remain

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 1.2: Remove border from UiAccordionRandomTablesList component

**Files:**
- Modify: `app/components/ui/accordion/UiAccordionRandomTablesList.vue:93-102`

**Step 1: Read current component**

```bash
cat app/components/ui/accordion/UiAccordionRandomTablesList.vue | grep -A 10 "v-for=\"table in tables\""
```

**Expected:** See current code with `border-l-4 pl-4` classes

**Step 2: Remove border and padding classes**

Find this code (lines 93-102):

```vue
<template>
  <div
    v-if="tables.length > 0"
    class="p-4 space-y-6"
  >
    <div
      v-for="table in tables"
      :key="table.id"
      class="space-y-2 border-l-4 pl-4"
      :class="`border-${borderColor}`"
    >
```

Replace with:

```vue
<template>
  <div
    v-if="tables.length > 0"
    class="p-4 space-y-6"
  >
    <div
      v-for="table in tables"
      :key="table.id"
      class="space-y-2"
    >
```

**Changes:**
- Line 100: Remove `border-l-4 pl-4` from class attribute
- Line 101: Remove entire `:class` binding line

**Step 3: Verify TypeScript compiles**

```bash
docker compose exec nuxt npm run typecheck
```

**Expected:** No TypeScript errors

**Step 4: Run tests to verify they pass**

```bash
docker compose exec nuxt npm run test -- UiAccordionRandomTablesList
```

**Expected:** All tests PASS (including new border removal tests)

**Step 5: Visual verification**

```bash
# Ensure dev server is running
curl http://localhost:3000/backgrounds/charlatan
```

**Manual check:** Open http://localhost:3000/backgrounds/charlatan in browser
- ✅ Traits section visible (still in UCard for now)
- ✅ Random tables display without double border
- ✅ Tables still have spacing and structure

**Step 6: Commit the implementation**

```bash
git add app/components/ui/accordion/UiAccordionRandomTablesList.vue
git commit -m "refactor: Remove border-l-4 from UiAccordionRandomTablesList for clean nesting

- Removes redundant border when nested in UiAccordionTraitsList
- Traits already have border-l-4, tables inherit grouping
- Simplifies component, prevents double-border visual bug
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase 2: Refactor Backgrounds Detail Page

### Task 2.1: Write failing tests for new accordion structure

**Files:**
- Create: `tests/pages/backgrounds/slug.test.ts` (if doesn't exist)
- Modify: `tests/pages/backgrounds/slug.test.ts` (if exists)

**Step 1: Check if test file exists**

```bash
ls -la tests/pages/backgrounds/slug.test.ts
```

**Step 2: Create or modify test file**

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import BackgroundsSlugPage from '~/pages/backgrounds/[slug].vue'

describe('Backgrounds [slug] Page', () => {
  // Mock the useEntityDetail composable
  const mockBackground = {
    id: 1,
    name: 'Charlatan',
    slug: 'charlatan',
    description: 'You have always had a way with people.',
    traits: [
      {
        id: '1',
        name: 'Feature: False Identity',
        description: 'You have created a second identity.',
        category: 'Feature',
        random_tables: []
      }
    ],
    proficiencies: ['Deception', 'Sleight of Hand'],
    languages: ['Any two languages'],
    equipment: ['Fine clothes', 'Disguise kit', 'Gold pieces'],
    sources: [{ id: '1', name: "Player's Handbook", code: 'PHB', page: 128 }],
    tags: ['Social', 'Urban']
  }

  it('should render single UAccordion with type="multiple"', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    // Find all UAccordion components
    const accordions = wrapper.findAllComponents({ name: 'UAccordion' })

    // Should be exactly ONE accordion
    expect(accordions).toHaveLength(1)

    // Should have type="multiple"
    expect(accordions[0].props('type')).toBe('multiple')
  })

  it('should NOT render traits in standalone UCard', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    // Look for traits card header outside accordion
    const headers = wrapper.findAll('h2')
    const traitsHeader = headers.find(h => h.text() === 'Background Traits')

    if (traitsHeader) {
      // If header exists, ensure it's inside accordion
      const accordion = wrapper.findComponent({ name: 'UAccordion' })
      expect(accordion.element.contains(traitsHeader.element)).toBe(true)
    }
  })

  it('should render traits in accordion slot', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    // Accordion should have traits slot
    const accordion = wrapper.findComponent({ name: 'UAccordion' })
    expect(accordion.exists()).toBe(true)

    // Should contain UiAccordionTraitsList component
    const traitsList = wrapper.findComponent({ name: 'UiAccordionTraitsList' })
    expect(traitsList.exists()).toBe(true)
  })

  it('should render proficiencies in accordion slot', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    // Should contain UiAccordionBulletList for proficiencies
    const bulletList = wrapper.findComponent({ name: 'UiAccordionBulletList' })
    expect(bulletList.exists()).toBe(true)
  })

  it('should render languages in accordion slot', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    // Should contain UiAccordionBadgeList for languages
    const badgeList = wrapper.findComponent({ name: 'UiAccordionBadgeList' })
    expect(badgeList.exists()).toBe(true)
  })

  it('should render image before description (if present)', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    const image = wrapper.findComponent({ name: 'UiDetailEntityImage' })
    const descriptionCard = wrapper.find('[data-testid="description-card"]')

    if (image.exists() && descriptionCard.exists()) {
      // Get DOM positions
      const allElements = Array.from(wrapper.element.querySelectorAll('*'))
      const imageIndex = allElements.indexOf(image.element as Element)
      const descIndex = allElements.indexOf(descriptionCard.element as Element)

      expect(imageIndex).toBeGreaterThan(-1)
      expect(descIndex).toBeGreaterThan(-1)
      expect(imageIndex).toBeLessThan(descIndex)
    }
  })

  it('should render description outside accordion', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    const accordion = wrapper.findComponent({ name: 'UAccordion' })
    const descriptionCard = wrapper.find('[data-testid="description-card"]')

    if (descriptionCard.exists()) {
      // Description should NOT be inside accordion
      expect(accordion.element.contains(descriptionCard.element)).toBe(false)
    }
  })

  it('should have all sections in accordion items array', async () => {
    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    const accordion = wrapper.findComponent({ name: 'UAccordion' })
    expect(accordion.exists()).toBe(true)

    const items = accordion.props('items')
    expect(Array.isArray(items)).toBe(true)
    expect(items.length).toBeGreaterThan(0)

    // Should contain traits, proficiencies, languages at minimum
    const slots = items.map((item: any) => item.slot)
    expect(slots).toContain('traits')
    expect(slots).toContain('proficiencies')
    expect(slots).toContain('languages')
  })
})
```

**Step 3: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test -- backgrounds/slug
```

**Expected:** Tests FAIL because current structure doesn't match expected accordion pattern

**Step 4: Commit the failing tests**

```bash
git add tests/pages/backgrounds/slug.test.ts
git commit -m "test: Add failing tests for backgrounds page accordion structure

- Tests expect single UAccordion with type='multiple'
- Tests expect traits/proficiencies/languages in accordion
- Tests expect image before description
- Tests expect description outside accordion

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 2.2: Restructure backgrounds page template

**Files:**
- Modify: `app/pages/backgrounds/[slug].vue:31-184`

**Step 1: Read current page structure**

```bash
wc -l app/pages/backgrounds/[slug].vue
```

**Expected:** 186 lines total

**Step 2: Replace template section (lines 31-184)**

**IMPORTANT:** Keep script section unchanged (lines 1-29). Only modify template.

Replace template (lines 31-184) with this new structure:

```vue
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
      <!-- Breadcrumb Navigation -->
      <UiBackLink
        to="/backgrounds"
        label="Back to Backgrounds"
      />

      <!-- Header -->
      <UiDetailPageHeader
        :title="entity.name"
        :badges="[
          { label: 'Background', color: 'success', variant: 'subtle', size: 'lg' }
        ]"
      />

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

      <!-- Description (always visible, outside accordion) -->
      <UCard
        v-if="entity.description"
        data-testid="description-card"
      >
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

      <!-- Single Unified Accordion - ALL expandable sections -->
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

      <!-- JSON Debug Panel -->
      <JsonDebugPanel
        :data="entity"
        title="Background Data"
      />
    </div>
  </div>
</template>
```

**Key changes from original:**
1. **Lines 62-90 (old)** → **Lines 31-38 (new)**: Image moved to dedicated section
2. **Lines 93-104 (old)** → **Lines 41-56 (new)**: Description moved up, added `data-testid`
3. **Lines 62-127 (old)** → **Lines 59-149 (new)**: All sections now in single accordion
4. **Line 148**: `type="multiple"` enables multiple open sections

**Step 3: Verify TypeScript compiles**

```bash
docker compose exec nuxt npm run typecheck
```

**Expected:** No TypeScript errors

**Step 4: Run full test suite**

```bash
docker compose exec nuxt npm run test
```

**Expected:** All tests pass (including new backgrounds/slug tests)

**Step 5: Manual visual verification**

Open http://localhost:3000/backgrounds/charlatan and verify:

- ✅ Image appears in dedicated section (before description)
- ✅ Description always visible (not in accordion)
- ✅ Single accordion with 6 sections (traits, proficiencies, languages, equipment, source, tags)
- ✅ All sections expand/collapse correctly
- ✅ Multiple sections can be open simultaneously (`type="multiple"`)
- ✅ Random tables display cleanly without double borders
- ✅ No horizontal scrollbars

**Step 6: Test mobile responsive**

Test at different breakpoints:
- 375px (mobile): `cmd+opt+i`, responsive mode, iPhone SE
- 768px (tablet): iPad
- 1440px (desktop): Default

Verify:
- ✅ Image scales properly
- ✅ Accordion works on all screen sizes
- ✅ No horizontal overflow

**Step 7: Compare with other entity pages**

Visual consistency check:
- `/classes/wizard` - Does accordion pattern match?
- `/feats/grappler` - Does image placement match?
- `/races/elf` - Does description treatment match?

**Step 8: Commit the refactoring**

```bash
git add app/pages/backgrounds/[slug].vue
git commit -m "refactor: Standardize backgrounds detail page to match entity pattern

- Move image to dedicated section (before description)
- Move description outside accordion (always visible)
- Consolidate ALL sections into single UAccordion type='multiple'
- Move traits from UCard to accordion slot
- Move proficiencies from UCard to accordion slot
- Move languages from UCard to accordion slot
- Merge with existing equipment/source/tags sections
- Consistent with classes/feats/races pages
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 2.3: Update CHANGELOG and handover document

**Files:**
- Modify: `CHANGELOG.md`
- Modify: `docs/HANDOVER-2025-11-23-BACKGROUNDS-DETAIL-PAGE-REDESIGN.md`

**Step 1: Update CHANGELOG.md**

Add to the top of the `### Fixed` section (or create if doesn't exist):

```markdown
### Changed
- Standardized backgrounds detail page to match entity accordion pattern (2025-11-23)
  - All expandable sections now in single accordion
  - Image placement consistent with other pages
  - Description always visible (improved UX)

### Fixed
- Removed double border-l-4 from random tables when nested in traits (2025-11-23)
- Fixed horizontal scrollbars on backgrounds random tables (2025-11-23)
```

**Step 2: Update handover document status**

Change line 5 in `docs/HANDOVER-2025-11-23-BACKGROUNDS-DETAIL-PAGE-REDESIGN.md`:

```markdown
**Status**: ✅ COMPLETED - 2025-11-23
```

Add completion summary at the end:

```markdown
---

## ✅ COMPLETION SUMMARY (2025-11-23)

### What Was Changed

1. **UiAccordionRandomTablesList** - Removed `border-l-4` and `pl-4` for clean nesting
2. **Backgrounds Detail Page** - Complete structural refactoring:
   - Single `UAccordion type="multiple"` with 6 sections
   - Image placement standardized (dedicated section before description)
   - Description always visible (outside accordion)
   - Traits, proficiencies, languages moved to accordion

### Results

- ✅ All 6 success criteria met
- ✅ Random tables display cleanly without double borders
- ✅ No horizontal scrollbars
- ✅ Consistent with classes/feats/races pages
- ✅ All tests passing
- ✅ Mobile responsive (375px → 1440px+)

### Files Modified

- `app/components/ui/accordion/UiAccordionRandomTablesList.vue`
- `app/pages/backgrounds/[slug].vue`
- `tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts`
- `tests/pages/backgrounds/slug.test.ts`
- `CHANGELOG.md`

### Test Coverage

- Unit tests: UiAccordionRandomTablesList (5 tests)
- Page tests: backgrounds/[slug] (8 tests)
- Visual verification: Charlatan background
- Responsive testing: 375px, 768px, 1440px
- Cross-page consistency: classes/feats/races comparison

**Implementation time:** ~2 hours (TDD + testing + verification)
```

**Step 3: Commit documentation updates**

```bash
git add CHANGELOG.md docs/HANDOVER-2025-11-23-BACKGROUNDS-DETAIL-PAGE-REDESIGN.md
git commit -m "docs: Update CHANGELOG and mark backgrounds redesign complete

- Updated CHANGELOG with refactoring changes
- Marked handover document as COMPLETED
- Added completion summary with results

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Final Verification Checklist

Run these commands to verify everything is working:

```bash
# 1. All tests pass
docker compose exec nuxt npm run test
# Expected: All tests passing (including new tests)

# 2. TypeScript compiles
docker compose exec nuxt npm run typecheck
# Expected: No errors

# 3. Lint passes
docker compose exec nuxt npm run lint
# Expected: No errors

# 4. Page loads correctly
curl -I http://localhost:3000/backgrounds/charlatan
# Expected: HTTP 200

# 5. Visual check
open http://localhost:3000/backgrounds/charlatan
# Expected: Single accordion, clean layout, no scrollbars
```

**Success criteria:**
- ✅ All automated tests pass
- ✅ TypeScript and linter happy
- ✅ Page loads with HTTP 200
- ✅ Visual layout matches classes/feats pages
- ✅ No horizontal scrollbars
- ✅ Random tables display cleanly
- ✅ Mobile responsive
- ✅ All commits made with descriptive messages

---

## Insights for Implementation

**Why this order?**
1. **Border fix first** - Independent change, small scope, validates TDD workflow
2. **Page refactor second** - Depends on border fix, larger scope
3. **Docs last** - Only after verification complete

**Why bite-sized commits?**
- Each commit is atomic and reversible
- Easy to review changes
- Bisectable if bugs found later
- Documents implementation journey

**Why TDD?**
- Tests written first ensure we know what "done" looks like
- Prevents regression bugs
- Documents expected behavior
- Builds confidence in refactoring

**Common pitfalls to avoid:**
- ❌ Don't skip visual verification (tests can't catch all layout issues)
- ❌ Don't batch all changes into one commit (loses granularity)
- ❌ Don't forget mobile testing (accordion behavior differs)
- ❌ Don't assume tests pass without running them (always verify)

---

## Rollback Strategy

If critical issues found:

```bash
# View recent commits
git log --oneline -5

# Rollback to before page refactor (keep border fix)
git revert <commit-hash-of-page-refactor>

# OR rollback everything
git reset --hard <commit-before-starting>
```

**Border fix is safe to keep** - no downside, improves nesting in all contexts.

---

**Plan complete!** Ready for execution with TDD discipline and frequent commits.
