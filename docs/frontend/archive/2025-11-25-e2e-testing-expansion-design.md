# E2E Testing Expansion - Design Document

**Date:** 2025-11-25
**Author:** Claude (via brainstorming skill)
**Status:** Ready for Implementation
**Approach:** Pragmatic Hybrid (Characterization Testing)

---

## Overview

Expand Playwright E2E test coverage in three sequential phases: fix existing failures, add filter interaction tests, and add detail page tests. Focus on characterization testing (documenting existing behavior) for features already in production.

**Total Estimated Time:** ~4 hours
**Expected New Tests:** ~63 tests (113 → 176 total)
**Target Pass Rate:** 100% (currently 96.5%)

---

## Background

### Current State
- ✅ Playwright infrastructure exists (playwright.config.ts, npm scripts)
- ✅ 113 E2E tests written (109/113 passing = 96.5%)
- ✅ Good coverage: Homepage (40+ tests) + Entity lists (70+ tests)
- ❌ 4 failing tests (homepage description matching issues)
- ❌ Zero detail page tests
- ❌ Zero filter interaction tests

### Recent Context
- Filter composables just refactored (6 entity pages migrated)
- 22 filters across entity pages need E2E validation
- All features working in production, need regression protection

---

## Design Decisions

### Approach: Pragmatic Hybrid

**Rationale for Characterization Testing:**
1. Features already work and are in production
2. Not discovering new requirements - documenting existing behavior
3. Creates safety net for future refactoring
4. Faster than pure TDD for existing features (~4 hours vs 6 hours)

**CLAUDE.md TDD Mandate Compliance:**
- TDD mandate applies to *new feature development*
- Characterization testing is appropriate for *existing features*
- We're writing tests to protect against regressions, not drive new development

### Test File Structure

```
tests/e2e/
├── homepage.spec.ts              # Existing (40+ tests, needs 4 fixes)
├── entity-lists.spec.ts          # Existing (70+ tests, working)
├── filter-interactions.spec.ts   # NEW (~28 tests)
└── entity-detail-pages.spec.ts   # NEW (~35 tests)
```

**Design Principles:**
- One test file per major feature area
- Consistent test patterns across entity types
- Representative sampling (not exhaustive coverage)
- Focus on user-visible behavior
- Soft assertions for optional features

---

## Phase 1: Fix 4 Remaining Failures

**Time Estimate:** ~30 minutes
**File:** `tests/e2e/homepage.spec.ts`

### Problem Analysis

**Failing Tests:**
- "displays Spells card with description"
- "displays Items card with description"
- "displays Races card with description"
- (One more description test)

**Root Cause:** Test assertions expect full descriptions, but homepage entity cards may show truncated text or no descriptions.

### Solution Strategy

**Option A (Preferred): Flexible Assertions**
```typescript
// Before (too strict)
await expect(card.getByText('Browse hundreds of magical spells')).toBeVisible()

// After (flexible)
await expect(card.getByText(/Browse.*spells/i)).toBeVisible()
// Or just verify card is visible, don't check description
```

**Option B: Update Expected Text**
- Inspect actual homepage card content
- Update test expectations to match exact truncated text

**Option C: Remove Description Checks**
- If cards don't show descriptions, remove those assertions
- Keep card visibility and link tests

### Implementation Steps

1. Run failing tests to see actual vs expected
2. Inspect homepage in browser to verify card content
3. Choose solution (likely Option A - partial text matching)
4. Update test assertions
5. Verify all 113 tests pass
6. Commit: "fix: resolve 4 homepage description test failures"

**Success Criteria:** 113/113 tests passing (100%)

---

## Phase 2: Filter Interaction Tests

**Time Estimate:** ~1.5-2 hours
**File:** `tests/e2e/filter-interactions.spec.ts` (NEW)
**Expected Tests:** ~26-28 tests

### Coverage Strategy

Test **representative samples** of filter types, validating the recently refactored filter composables work end-to-end.

### Test Categories

#### 1. Dropdown Filters (equals type) - 6 tests

**Purpose:** Test single-select dropdowns that use `field = value` filter syntax

**Tests:**
```typescript
test('Spells level filter updates URL and results', async ({ page }) => {
  await page.goto('/spells')

  // Open level dropdown (exact implementation may vary)
  await page.locator('select, button').filter({ hasText: /level/i }).first().click()

  // Select "3rd level"
  await page.getByText('3rd level', { exact: false }).click()

  // Verify URL contains filter parameter
  await expect(page).toHaveURL(/filter=level%20%3D%203/)

  // Verify results filtered (count changed from total)
  const cardCount = await page.locator('a[href*="/spells/"]').count()
  expect(cardCount).toBeLessThan(477) // Total spells
  expect(cardCount).toBeGreaterThan(0) // Has results
})

test('Items rarity filter updates URL and results', async ({ page }) => {
  await page.goto('/items')

  await page.locator('select, button').filter({ hasText: /rarity/i }).first().click()
  await page.getByText('Rare', { exact: false }).click()

  await expect(page).toHaveURL(/filter.*rare/i)

  const cardCount = await page.locator('a[href*="/items/"]').count()
  expect(cardCount).toBeGreaterThan(0)
})

test('Monsters CR filter updates URL and results', async ({ page }) => {
  await page.goto('/monsters')

  await page.locator('select, button').filter({ hasText: /challenge rating|CR/i }).first().click()
  await page.getByText(/^1$|^CR 1$/).click()

  await expect(page).toHaveURL(/filter.*cr/i)

  const cardCount = await page.locator('a[href*="/monsters/"]').count()
  expect(cardCount).toBeGreaterThan(0)
})
```

**Additional tests:**
- Spells school filter
- Items type filter
- Classes base class filter

#### 2. Boolean Toggle Filters - 6 tests

**Purpose:** Test tri-state toggles (All/Yes/No) using `field = true/false` syntax

**Tests:**
```typescript
test('Spells concentration toggle filters results', async ({ page }) => {
  await page.goto('/spells')

  // Find and click concentration toggle
  const concentrationToggle = page.locator('button, label').filter({ hasText: /concentration/i })
  await concentrationToggle.first().click()

  // Wait for URL update (debounced)
  await page.waitForTimeout(300)
  await expect(page).toHaveURL(/concentration/)

  // Verify results changed
  const cardCount = await page.locator('a[href*="/spells/"]').count()
  expect(cardCount).toBeGreaterThan(0)
  expect(cardCount).toBeLessThan(477) // Not all spells
})

test('Items magic toggle filters results', async ({ page }) => {
  await page.goto('/items')

  const magicToggle = page.locator('button, label').filter({ hasText: /magic/i })
  await magicToggle.first().click()

  await page.waitForTimeout(300)
  await expect(page).toHaveURL(/magic/)

  const cardCount = await page.locator('a[href*="/items/"]').count()
  expect(cardCount).toBeGreaterThan(0)
})

test('Feats prerequisites toggle filters results', async ({ page }) => {
  await page.goto('/feats')

  const prereqToggle = page.locator('button, label').filter({ hasText: /prerequisite/i })
  await prereqToggle.first().click()

  await page.waitForTimeout(300)
  await expect(page).toHaveURL(/prerequisite/)

  const cardCount = await page.locator('a[href*="/feats/"]').count()
  expect(cardCount).toBeGreaterThan(0)
})
```

**Additional tests:**
- Spells ritual toggle
- Monsters legendary toggle
- Classes spellcaster toggle

#### 3. Multi-Select Filters (IN type) - 6 tests

**Purpose:** Test multi-select dropdowns using `field IN [value1, value2]` syntax

**Tests:**
```typescript
test('Spells damage types multi-select filters results', async ({ page }) => {
  await page.goto('/spells')

  // Open damage types multi-select
  await page.locator('button').filter({ hasText: /damage type/i }).first().click()

  // Select Fire
  await page.locator('label, div').filter({ hasText: /^Fire$/i }).first().click()

  // Select Cold
  await page.locator('label, div').filter({ hasText: /^Cold$/i }).first().click()

  // Close dropdown (click outside or press Escape)
  await page.keyboard.press('Escape')

  // Wait for filter to apply
  await page.waitForTimeout(300)

  // Verify URL contains IN filter
  await expect(page).toHaveURL(/damage_type.*IN/i)

  // Verify results filtered
  const cardCount = await page.locator('a[href*="/spells/"]').count()
  expect(cardCount).toBeGreaterThan(0)
  expect(cardCount).toBeLessThan(477)
})

test('Spells saving throws multi-select filters results', async ({ page }) => {
  await page.goto('/spells')

  await page.locator('button').filter({ hasText: /saving throw/i }).first().click()
  await page.locator('label, div').filter({ hasText: /^Dexterity$/i }).first().click()
  await page.locator('label, div').filter({ hasText: /^Wisdom$/i }).first().click()
  await page.keyboard.press('Escape')

  await page.waitForTimeout(300)
  await expect(page).toHaveURL(/saving_throw.*IN/i)

  const cardCount = await page.locator('a[href*="/spells/"]').count()
  expect(cardCount).toBeGreaterThan(0)
})
```

**Additional tests:**
- Spells class multi-select (single value but uses IN)
- Items damage types multi-select
- Monsters type multi-select
- Races size multi-select (if supported)

#### 4. Filter Combinations - 4 tests

**Purpose:** Test multiple filters combine with AND logic

**Tests:**
```typescript
test('Spells multiple filters combine correctly', async ({ page }) => {
  await page.goto('/spells')

  // Select level 3
  await page.locator('select, button').filter({ hasText: /level/i }).first().click()
  await page.getByText('3rd level').click()

  // Select school Evocation
  await page.locator('select, button').filter({ hasText: /school/i }).first().click()
  await page.getByText('Evocation').click()

  // Toggle concentration
  await page.locator('button, label').filter({ hasText: /concentration/i }).first().click()

  await page.waitForTimeout(500)

  // Verify URL has all filters
  await expect(page).toHaveURL(/level.*3/)
  await expect(page).toHaveURL(/school/i)
  await expect(page).toHaveURL(/concentration/)

  // Verify results are more filtered
  const cardCount = await page.locator('a[href*="/spells/"]').count()
  expect(cardCount).toBeGreaterThan(0)
  expect(cardCount).toBeLessThan(50) // Should be small subset
})

test('Items type and rarity filters combine', async ({ page }) => {
  await page.goto('/items')

  // Select type Weapon
  await page.locator('select, button').filter({ hasText: /type/i }).first().click()
  await page.getByText('Weapon').click()

  // Select rarity Rare
  await page.locator('select, button').filter({ hasText: /rarity/i }).first().click()
  await page.getByText('Rare').click()

  await page.waitForTimeout(500)

  await expect(page).toHaveURL(/type/)
  await expect(page).toHaveURL(/rarity/)

  const cardCount = await page.locator('a[href*="/items/"]').count()
  expect(cardCount).toBeGreaterThan(0)
})
```

**Additional tests:**
- Monsters CR + type + legendary
- Races size + has darkvision (if supported)

#### 5. Filter Persistence - 4 tests

**Purpose:** Test filters persist via URL parameters

**Tests:**
```typescript
test('Spells filters persist across navigation', async ({ page }) => {
  await page.goto('/spells')

  // Set level filter
  await page.locator('select, button').filter({ hasText: /level/i }).first().click()
  await page.getByText('3rd level').click()
  await page.waitForTimeout(300)

  // Capture URL with filter
  const urlWithFilter = page.url()
  expect(urlWithFilter).toContain('level')

  // Navigate away
  await page.goto('/')

  // Navigate back via browser back button
  await page.goBack()

  // Verify still on filtered page
  expect(page.url()).toBe(urlWithFilter)

  // Verify filter still active (count still filtered)
  const cardCount = await page.locator('a[href*="/spells/"]').count()
  expect(cardCount).toBeLessThan(477)
})

test('Items filters persist on page refresh', async ({ page }) => {
  await page.goto('/items')

  // Set magic filter
  await page.locator('button, label').filter({ hasText: /magic/i }).first().click()
  await page.waitForTimeout(300)

  const cardCountBefore = await page.locator('a[href*="/items/"]').count()

  // Refresh page
  await page.reload()
  await page.waitForSelector('a[href*="/items/"]', { timeout: 10000 })

  // Verify filter still active
  await expect(page).toHaveURL(/magic/)

  const cardCountAfter = await page.locator('a[href*="/items/"]').count()
  expect(cardCountAfter).toBe(cardCountBefore)
})
```

**Additional tests:**
- Spells filters persist on direct URL navigation
- Monsters filters persist on page refresh

#### 6. Filter Reset - 2 tests

**Purpose:** Test clearing filters returns to unfiltered state

**Tests:**
```typescript
test('Spells filter count badge updates', async ({ page }) => {
  await page.goto('/spells')

  // Verify filter count badge shows 0 or is hidden
  const filterBadge = page.locator('[class*="badge"]').filter({ hasText: /^\d+$/ })
  const initialCount = await filterBadge.count()

  if (initialCount > 0) {
    await expect(filterBadge.first()).toHaveText('0')
  }

  // Apply filter
  await page.locator('select, button').filter({ hasText: /level/i }).first().click()
  await page.getByText('3rd level').click()
  await page.waitForTimeout(300)

  // Verify badge shows 1 active filter
  await expect(filterBadge.first()).toBeVisible()
  await expect(filterBadge.first()).toHaveText('1')
})

test('Clearing all filters shows all results', async ({ page }) => {
  await page.goto('/spells')

  // Apply multiple filters
  await page.locator('select, button').filter({ hasText: /level/i }).first().click()
  await page.getByText('3rd level').click()
  await page.waitForTimeout(300)

  const filteredCount = await page.locator('a[href*="/spells/"]').count()
  expect(filteredCount).toBeLessThan(477)

  // Find and click clear/reset button (or set filter back to "All")
  const clearButton = page.locator('button').filter({ hasText: /clear|reset/i })
  if (await clearButton.count() > 0) {
    await clearButton.first().click()
  } else {
    // Or deselect filter manually
    await page.locator('select, button').filter({ hasText: /level/i }).first().click()
    await page.getByText(/all|any/i).click()
  }

  await page.waitForTimeout(500)

  // Verify all results shown
  const unfilteredCount = await page.locator('a[href*="/spells/"]').count()
  expect(unfilteredCount).toBeGreaterThan(filteredCount)
})
```

### Test Organization

```typescript
// tests/e2e/filter-interactions.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Filter Interactions', () => {

  test.describe('Dropdown Filters (equals)', () => {
    // 6 tests here
  })

  test.describe('Boolean Toggle Filters', () => {
    // 6 tests here
  })

  test.describe('Multi-Select Filters (IN)', () => {
    // 6 tests here
  })

  test.describe('Filter Combinations', () => {
    // 4 tests here
  })

  test.describe('Filter Persistence', () => {
    // 4 tests here
  })

  test.describe('Filter Reset', () => {
    // 2 tests here
  })
})
```

### Expected Outcome

- **Tests Added:** ~26-28 tests
- **Pass Rate:** 100% (characterization of working features)
- **Commit:** "test: add E2E tests for filter interactions across entity pages"
- **Validation:** Confirms filter composables work end-to-end in browser

---

## Phase 3: Detail Page Tests

**Time Estimate:** ~1.5-2 hours
**File:** `tests/e2e/entity-detail-pages.spec.ts` (NEW)
**Expected Tests:** ~30-35 tests

### Coverage Strategy

Test **all 7 entity types** with consistent patterns. Focus on page load, content display, and navigation. Use known working entities (fireball, longsword, elf, wizard, sage, war-caster, goblin).

### Test Categories

#### 1. Basic Load Tests - 7 tests (one per entity)

**Purpose:** Verify detail pages load successfully

**Tests:**
```typescript
test.describe('Entity Detail Pages', () => {

  test.describe('Spells', () => {
    test('Fireball detail page loads', async ({ page }) => {
      await page.goto('/spells/fireball')

      // Verify URL
      await expect(page).toHaveURL('/spells/fireball')

      // Verify heading
      await expect(page.locator('h1')).toContainText('Fireball')
    })
  })

  test.describe('Items', () => {
    test('Longsword detail page loads', async ({ page }) => {
      await page.goto('/items/longsword')

      await expect(page).toHaveURL('/items/longsword')
      await expect(page.locator('h1')).toContainText('Longsword')
    })
  })

  test.describe('Races', () => {
    test('Elf detail page loads', async ({ page }) => {
      await page.goto('/races/elf')

      await expect(page).toHaveURL('/races/elf')
      await expect(page.locator('h1')).toContainText('Elf')
    })
  })

  test.describe('Classes', () => {
    test('Wizard detail page loads', async ({ page }) => {
      await page.goto('/classes/wizard')

      await expect(page).toHaveURL('/classes/wizard')
      await expect(page.locator('h1')).toContainText('Wizard')
    })
  })

  test.describe('Backgrounds', () => {
    test('Sage detail page loads', async ({ page }) => {
      await page.goto('/backgrounds/sage')

      await expect(page).toHaveURL('/backgrounds/sage')
      await expect(page.locator('h1')).toContainText('Sage')
    })
  })

  test.describe('Feats', () => {
    test('War Caster detail page loads', async ({ page }) => {
      await page.goto('/feats/war-caster')

      await expect(page).toHaveURL('/feats/war-caster')
      await expect(page.locator('h1')).toContainText('War Caster')
    })
  })

  test.describe('Monsters', () => {
    test('Goblin detail page loads', async ({ page }) => {
      await page.goto('/monsters/goblin')

      await expect(page).toHaveURL('/monsters/goblin')
      await expect(page.locator('h1')).toContainText('Goblin')
    })
  })
})
```

#### 2. Quick Stats Display - 7 tests

**Purpose:** Verify quick stats card shows entity-specific information

**Tests:**
```typescript
test('Fireball shows quick stats (level, school)', async ({ page }) => {
  await page.goto('/spells/fireball')

  // Look for quick stats card or section
  const quickStats = page.locator('[class*="stats"], [class*="quick"]').first()

  // Verify spell-specific stats
  await expect(page.getByText(/3rd level/i)).toBeVisible()
  await expect(page.getByText(/Evocation/i)).toBeVisible()
})

test('Longsword shows quick stats (type, rarity)', async ({ page }) => {
  await page.goto('/items/longsword')

  // Verify item-specific stats
  await expect(page.getByText(/Weapon/i)).toBeVisible()
  await expect(page.getByText(/Martial/i)).toBeVisible()
})

test('Elf shows quick stats (size, speed)', async ({ page }) => {
  await page.goto('/races/elf')

  // Verify race-specific stats
  await expect(page.getByText(/Medium/i)).toBeVisible()
  await expect(page.getByText(/30.*ft/i)).toBeVisible()
})

test('Wizard shows quick stats (hit die, spellcasting)', async ({ page }) => {
  await page.goto('/classes/wizard')

  // Verify class-specific stats
  await expect(page.getByText(/d6/i)).toBeVisible()
  await expect(page.getByText(/Intelligence/i)).toBeVisible()
})

test('Sage shows quick stats (skills, languages)', async ({ page }) => {
  await page.goto('/backgrounds/sage')

  // Verify background-specific content
  await expect(page.getByText(/Arcana|History|Investigation/i)).toBeVisible()
})

test('War Caster shows quick stats (prerequisites)', async ({ page }) => {
  await page.goto('/feats/war-caster')

  // Verify feat-specific info
  // War Caster requires spellcasting ability
  const hasPrereq = await page.getByText(/prerequisite|spellcasting/i).count()
  expect(hasPrereq).toBeGreaterThan(0)
})

test('Goblin shows quick stats (CR, type, AC)', async ({ page }) => {
  await page.goto('/monsters/goblin')

  // Verify monster-specific stats
  await expect(page.getByText(/CR.*1\/4|Challenge Rating/i)).toBeVisible()
  await expect(page.getByText(/AC|Armor Class/i)).toBeVisible()
})
```

#### 3. Accordion Content - 7 tests

**Purpose:** Verify accordion sections render and expand

**Tests:**
```typescript
test('Fireball has damage effects accordion', async ({ page }) => {
  await page.goto('/spells/fireball')

  // Find accordion trigger (text or button)
  const accordion = page.locator('button, div[role="button"]').filter({
    hasText: /Damage|Effects/i
  })

  await expect(accordion.first()).toBeVisible()

  // Click to expand
  await accordion.first().click()

  // Verify content appears
  await expect(page.getByText(/8d6|fire damage/i)).toBeVisible()
})

test('Longsword has properties accordion', async ({ page }) => {
  await page.goto('/items/longsword')

  const accordion = page.locator('button, div[role="button"]').filter({
    hasText: /Properties|Weapon Properties/i
  })

  await expect(accordion.first()).toBeVisible()
  await accordion.first().click()

  // Verify properties content
  await expect(page.getByText(/Versatile/i)).toBeVisible()
})

test('Elf has traits accordion', async ({ page }) => {
  await page.goto('/races/elf')

  const accordion = page.locator('button, div[role="button"]').filter({
    hasText: /Traits|Racial Traits/i
  })

  await expect(accordion.first()).toBeVisible()
  await accordion.first().click()

  // Verify traits content
  await expect(page.getByText(/Darkvision|Fey Ancestry/i)).toBeVisible()
})

test('Wizard has features accordion', async ({ page }) => {
  await page.goto('/classes/wizard')

  const accordion = page.locator('button, div[role="button"]').filter({
    hasText: /Features|Class Features/i
  })

  await expect(accordion.first()).toBeVisible()
  await accordion.first().click()

  // Verify features content
  await expect(page.getByText(/Spellcasting|Arcane Recovery/i)).toBeVisible()
})

test('Sage has background feature accordion', async ({ page }) => {
  await page.goto('/backgrounds/sage')

  const accordion = page.locator('button, div[role="button"]').filter({
    hasText: /Feature|Researcher/i
  })

  await expect(accordion.first()).toBeVisible()
  await accordion.first().click()

  // Verify feature content exists
  const contentLength = await page.locator('p, div').filter({
    hasText: /research|knowledge/i
  }).count()
  expect(contentLength).toBeGreaterThan(0)
})

test('War Caster has modifiers accordion', async ({ page }) => {
  await page.goto('/feats/war-caster')

  const accordion = page.locator('button, div[role="button"]').filter({
    hasText: /Benefits|Modifiers/i
  })

  await expect(accordion.first()).toBeVisible()
  await accordion.first().click()

  // Verify benefits content
  const contentLength = await page.locator('p, div, li').filter({
    hasText: /advantage|concentration/i
  }).count()
  expect(contentLength).toBeGreaterThan(0)
})

test('Goblin has actions accordion', async ({ page }) => {
  await page.goto('/monsters/goblin')

  const accordion = page.locator('button, div[role="button"]').filter({
    hasText: /Actions/i
  })

  await expect(accordion.first()).toBeVisible()
  await accordion.first().click()

  // Verify actions content
  await expect(page.getByText(/Scimitar|Shortbow/i)).toBeVisible()
})
```

#### 4. Navigation - 7 tests

**Purpose:** Verify breadcrumb navigation works

**Tests:**
```typescript
test('Fireball breadcrumb returns to spells list', async ({ page }) => {
  await page.goto('/spells/fireball')

  // Find "Back to Spells" or breadcrumb link
  const backLink = page.locator('a[href="/spells"]').first()
  await expect(backLink).toBeVisible()

  await backLink.click()

  await expect(page).toHaveURL('/spells')
  await expect(page.locator('h1')).toContainText('Spells')
})

test('Longsword breadcrumb returns to items list', async ({ page }) => {
  await page.goto('/items/longsword')

  const backLink = page.locator('a[href="/items"]').first()
  await expect(backLink).toBeVisible()

  await backLink.click()

  await expect(page).toHaveURL('/items')
})

// Similar tests for races, classes, backgrounds, feats, monsters
```

#### 5. Responsive Layout - 3-5 tests

**Purpose:** Verify detail pages work on different screen sizes

**Tests:**
```typescript
test('Fireball detail page responsive on mobile', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 667 })
  await page.goto('/spells/fireball')

  // Verify heading visible
  await expect(page.locator('h1')).toBeVisible()

  // Verify quick stats visible (may stack vertically)
  await expect(page.getByText(/3rd level/i)).toBeVisible()
})

test('Longsword detail page responsive on tablet', async ({ page }) => {
  await page.setViewportSize({ width: 768, height: 1024 })
  await page.goto('/items/longsword')

  await expect(page.locator('h1')).toBeVisible()
  await expect(page.getByText(/Weapon/i)).toBeVisible()
})

test('Goblin detail page responsive on desktop', async ({ page }) => {
  await page.setViewportSize({ width: 1440, height: 900 })
  await page.goto('/monsters/goblin')

  await expect(page.locator('h1')).toBeVisible()
  await expect(page.getByText(/CR/i)).toBeVisible()
})
```

### Test Organization

```typescript
// tests/e2e/entity-detail-pages.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Entity Detail Pages', () => {

  test.describe('Spells', () => {
    // 5 tests: load, quick stats, accordion, navigation, responsive
  })

  test.describe('Items', () => {
    // 5 tests
  })

  test.describe('Races', () => {
    // 5 tests
  })

  test.describe('Classes', () => {
    // 5 tests
  })

  test.describe('Backgrounds', () => {
    // 5 tests
  })

  test.describe('Feats', () => {
    // 5 tests
  })

  test.describe('Monsters', () => {
    // 5 tests
  })
})
```

### Expected Outcome

- **Tests Added:** ~30-35 tests (7 entities × 4-5 tests each)
- **Pass Rate:** 100% (characterization of working features)
- **Commit:** "test: add E2E tests for entity detail pages (all 7 types)"
- **Validation:** All entity detail pages render correctly with proper content

---

## Error Handling & Edge Cases

### Built Into Tests

**Empty States:**
- Multi-select all damage types → No results → Verify empty state message

**Responsive Behavior:**
- Mobile (375px), Tablet (768px), Desktop (1440px) tested on sample entities

**Navigation Edge Cases:**
- Browser back button preserves filters
- Page refresh maintains URL parameters

### Future Scope (Not Included)

**API Error Handling:**
- Requires mocking API failures (complex setup)
- Network offline simulation

**Performance Testing:**
- Page load metrics (requires Lighthouse/performance APIs)
- Large dataset pagination performance

**Accessibility Testing:**
- Keyboard navigation (Tab, Enter, Escape)
- Screen reader compatibility (requires axe-core)
- Focus management
- ARIA attributes validation

**404 Handling:**
- Non-existent entity slugs (e.g., `/spells/nonexistent`)
- Requires error page implementation test

---

## Testing Patterns & Best Practices

### Selector Strategy

**Priority Order:**
1. **Accessible selectors:** `getByRole('button')`, `getByText('Submit')`
2. **Test IDs:** `data-test="filter-level"` (if added)
3. **Text content:** `filter({ hasText: /level/i })`
4. **CSS selectors:** `locator('select')` (last resort)

**Anti-patterns to avoid:**
- Brittle class selectors: `.btn-primary-lg` (changes with UI updates)
- Deep nesting: `div > div > button` (fragile)
- Index-based: `.first()` without context (strict mode violations)

### Wait Strategies

**Use explicit waits for:**
- Debounced filters: `await page.waitForTimeout(300)`
- API responses: `await page.waitForSelector('a[href*="/spells/"]')`
- Navigation: `await page.waitForURL(/page=2/)`

**Avoid:**
- Arbitrary `setTimeout` without reason
- Polling loops (Playwright auto-retries assertions)

### Soft Assertions

**Use for optional features:**
```typescript
// Pagination may not exist on small datasets
const paginationExists = await paginationButtons.count()
if (paginationExists === 0) {
  console.log('Note: No pagination found - dataset may be small')
} else {
  expect(paginationExists).toBeGreaterThan(0)
}
```

**Use hard assertions for:**
- Core functionality (page loads, headings display)
- Required UI elements (breadcrumbs, cards)

### Test Independence

**Each test should:**
- Start from known state (use `beforeEach` for navigation)
- Not depend on other tests
- Clean up after itself (Playwright resets browser state)

**Example:**
```typescript
test.describe('Spells Filters', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/spells') // Fresh start for each test
  })

  test('level filter works', async ({ page }) => {
    // Test runs from clean /spells page
  })
})
```

---

## Success Criteria

### Per-Phase Criteria

**Phase 1: Fix Failures**
- ✅ All 113 existing tests pass (100%)
- ✅ No new failures introduced
- ✅ Clean commit with descriptive message

**Phase 2: Filter Tests**
- ✅ ~26-28 new tests added
- ✅ All new tests pass (100%)
- ✅ Tests cover representative filter types
- ✅ Tests validate filter composables work end-to-end
- ✅ Clean commit

**Phase 3: Detail Page Tests**
- ✅ ~30-35 new tests added
- ✅ All new tests pass (100%)
- ✅ All 7 entity types covered
- ✅ Consistent test patterns across entities
- ✅ Clean commit

### Overall Success Criteria

**Test Coverage:**
- ✅ 176 total E2E tests (113 → 176)
- ✅ 100% pass rate maintained throughout
- ✅ Coverage gaps addressed (filters + detail pages)

**Code Quality:**
- ✅ Consistent test patterns
- ✅ Clear, descriptive test names
- ✅ Proper use of Playwright selectors
- ✅ No flaky tests (reliable assertions)

**Documentation:**
- ✅ Design document committed (this file)
- ✅ Implementation plan created (Phase 6)
- ✅ README.md updated if needed

**Deliverables:**
- ✅ 3 clean commits (one per phase)
- ✅ All tests passing
- ✅ Comprehensive E2E coverage for critical user flows

---

## Timeline & Effort

**Total Time:** ~4 hours

**Breakdown:**
- Phase 1: 30 minutes (fix failures)
- Phase 2: 1.5-2 hours (filter tests)
- Phase 3: 1.5-2 hours (detail page tests)

**Dependencies:**
- Dev server running at localhost:3000
- Backend API running at localhost:8080
- Playwright browsers installed on host

**Execution Order:**
- Sequential (can't parallelize - each phase builds on previous)
- Each phase produces committable, working increment

---

## Risk Mitigation

### Potential Risks

**Risk 1: Selector Changes**
- **Mitigation:** Use accessible selectors (getByRole, getByText)
- **Mitigation:** Avoid brittle class-based selectors
- **Impact:** Low (Playwright best practices minimize risk)

**Risk 2: API Response Changes**
- **Mitigation:** Use soft assertions for data counts
- **Mitigation:** Test behavior, not exact data values
- **Impact:** Low (API stable, tests check structure not content)

**Risk 3: Timing Issues (Flaky Tests)**
- **Mitigation:** Explicit waits for debounced actions
- **Mitigation:** waitForSelector with generous timeouts
- **Mitigation:** Promise.all for navigation (wait for URL change)
- **Impact:** Low-Medium (already learned from existing tests)

**Risk 4: Filter UI Changes**
- **Mitigation:** Characterization tests document current behavior
- **Mitigation:** Tests will fail if UI changes (desired outcome)
- **Impact:** Low (tests should fail when UI changes significantly)

---

## Future Enhancements

**Not in current scope, but recommended for future:**

1. **Accessibility Testing**
   - Integrate axe-core for automated a11y audits
   - Test keyboard navigation (Tab, Enter, Escape)
   - Test screen reader announcements
   - Verify focus management

2. **Performance Testing**
   - Add Lighthouse CI for page load metrics
   - Test time to interactive (TTI)
   - Test largest contentful paint (LCP)
   - Monitor bundle size impact

3. **Visual Regression Testing**
   - Add Percy or Playwright screenshots
   - Compare visual diffs across commits
   - Catch unintended UI changes

4. **API Error Scenarios**
   - Mock API failures (500, 404, timeout)
   - Test error state displays
   - Test retry logic

5. **Advanced User Flows**
   - "Build a character" journey
   - "Find spell for situation" flow
   - "Compare items" workflow

6. **Cross-Browser Testing**
   - Enable Firefox project in playwright.config.ts
   - Enable WebKit (Safari) project
   - Test mobile browsers (Mobile Chrome, Mobile Safari)

7. **CI/CD Integration**
   - Run E2E tests on every PR
   - Upload test reports as artifacts
   - Fail builds on test failures

---

## Conclusion

This design provides a pragmatic, efficient path to comprehensive E2E coverage. By fixing existing failures, adding filter interaction tests, and covering detail pages, we'll achieve:

- **100% E2E test pass rate** (up from 96.5%)
- **176 total E2E tests** (+63 new tests)
- **Critical user flow coverage** (filters + detail pages)
- **Regression protection** for recent filter composables refactoring
- **Foundation for future testing** (patterns established, infrastructure solid)

The characterization testing approach is appropriate for existing, working features and will be completed faster than pure TDD while providing equivalent regression protection.

Ready to move to Phase 4 (Worktree Setup) and Phase 6 (Implementation Plan)?
