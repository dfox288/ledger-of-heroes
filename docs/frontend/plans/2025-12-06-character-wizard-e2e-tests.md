# Character Wizard E2E Test Suite Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a comprehensive Playwright E2E test suite that verifies the complete character creation wizard flow, creating a Human Bard with Acolyte background using manual ability score entry.

**Architecture:** The test suite will be organized as a single spec file with multiple test blocks. Tests will navigate through all wizard steps, verifying UI elements, validation rules, and step transitions. A helper function will encapsulate common navigation patterns.

**Tech Stack:** Playwright | TypeScript | Page Object Pattern (lightweight)

---

## Test Character Specification

| Property | Value | Notes |
|----------|-------|-------|
| Race | Human | No subraces → subrace step will be skipped |
| Subrace | None | Automatically skipped |
| Class | Bard | Spellcaster → spells step will appear |
| Subclass | None | Bard chooses subclass at level 3 → subclass step skipped |
| Background | Acolyte | Standard background |
| Abilities | Manual | Direct number entry (15, 14, 13, 12, 10, 8) |

## Wizard Steps for This Character

Based on the test character, these steps will be **visible**:
1. `sourcebooks` - Select PHB
2. `race` - Select Human
3. ~~`subrace`~~ - **SKIPPED** (Human has no subraces)
4. `class` - Select Bard
5. ~~`subclass`~~ - **SKIPPED** (Bard picks at level 3)
6. `background` - Select Acolyte
7. `abilities` - Manual entry
8. `proficiencies` - Select skill proficiencies (Bard gets 3)
9. `languages` - May be skipped if no choices
10. `equipment` - Starting equipment
11. `spells` - Select Bard spells (Bard is a spellcaster)
12. `details` - Name and alignment
13. `review` - Final confirmation

---

## Task 1: Create Test File Structure and Helper Utilities

**Files:**
- Create: `tests/e2e/character-wizard.spec.ts`

**Step 1: Create the test file with imports and constants**

```typescript
// tests/e2e/character-wizard.spec.ts
import { test, expect, type Page, type Locator } from '@playwright/test'

/**
 * E2E Tests: Character Creation Wizard
 *
 * Tests the complete character creation flow with:
 * - Race: Human (no subrace)
 * - Class: Bard (spellcaster)
 * - Background: Acolyte
 * - Abilities: Manual entry
 */

// ════════════════════════════════════════════════════════════════
// CONSTANTS
// ════════════════════════════════════════════════════════════════

const WIZARD_BASE_URL = '/characters/new'

/** Test character ability scores (valid range: 3-20) */
const TEST_ABILITY_SCORES = {
  strength: 8,
  dexterity: 14,
  constitution: 13,
  intelligence: 10,
  wisdom: 12,
  charisma: 15
} as const

/** Test character name */
const TEST_CHARACTER_NAME = 'Test Human Bard'

// ════════════════════════════════════════════════════════════════
// HELPER FUNCTIONS
// ════════════════════════════════════════════════════════════════

/**
 * Get the Next button in the wizard footer
 */
function getNextButton(page: Page): Locator {
  return page.locator('[data-test="next-button"]')
}

/**
 * Get the Back button in the wizard footer
 */
function getBackButton(page: Page): Locator {
  return page.locator('[data-test="back-button"]')
}

/**
 * Wait for the loading spinner to disappear
 */
async function waitForLoading(page: Page): Promise<void> {
  // Wait for any loading spinners to finish
  await page.locator('.animate-spin').waitFor({ state: 'hidden', timeout: 10000 }).catch(() => {
    // No spinner found, that's fine
  })
}

/**
 * Click the Next button and wait for navigation
 */
async function clickNextAndWait(page: Page): Promise<void> {
  const nextButton = getNextButton(page)
  await expect(nextButton).toBeEnabled({ timeout: 10000 })
  await nextButton.click()
  await waitForLoading(page)
}

/**
 * Navigate to a specific wizard step
 */
async function goToStep(page: Page, step: string): Promise<void> {
  await page.goto(`${WIZARD_BASE_URL}/${step}`)
  await waitForLoading(page)
}
```

**Step 2: Verify file was created correctly**

Run: `ls -la tests/e2e/character-wizard.spec.ts`
Expected: File exists

**Step 3: Commit**

```bash
git add tests/e2e/character-wizard.spec.ts
git commit -m "test(e2e): scaffold character wizard test file with helpers"
```

---

## Task 2: Test Sourcebooks Step

**Files:**
- Modify: `tests/e2e/character-wizard.spec.ts`

**Step 1: Add sourcebooks step tests**

Append to the test file:

```typescript
// ════════════════════════════════════════════════════════════════
// TESTS
// ════════════════════════════════════════════════════════════════

test.describe('Character Creation Wizard', () => {
  test.describe('Step 1: Sourcebooks', () => {
    test.beforeEach(async ({ page }) => {
      await goToStep(page, 'sourcebooks')
    })

    test('displays sourcebooks step with correct title', async ({ page }) => {
      await expect(page.getByRole('heading', { name: /source/i })).toBeVisible()
    })

    test('has PHB sourcebook available', async ({ page }) => {
      // PHB should be visible and selectable
      await expect(page.getByText('PHB')).toBeVisible()
    })

    test('Next button is enabled when sourcebook selected', async ({ page }) => {
      // PHB should be selected by default
      await expect(getNextButton(page)).toBeEnabled()
    })

    test('can proceed to race step', async ({ page }) => {
      await clickNextAndWait(page)
      await expect(page).toHaveURL(/\/characters\/new\/race/)
    })
  })
})
```

**Step 2: Run the test to verify it passes**

Run: `docker compose exec nuxt npx playwright test character-wizard --grep "Sourcebooks"`
Expected: Tests should pass (or fail if backend not running - that's expected)

**Step 3: Commit**

```bash
git add tests/e2e/character-wizard.spec.ts
git commit -m "test(e2e): add sourcebooks step tests"
```

---

## Task 3: Test Race Step (Human Selection)

**Files:**
- Modify: `tests/e2e/character-wizard.spec.ts`

**Step 1: Add race step tests**

Add within the `Character Creation Wizard` describe block:

```typescript
  test.describe('Step 2: Race', () => {
    test.beforeEach(async ({ page }) => {
      await goToStep(page, 'race')
    })

    test('displays race selection with correct title', async ({ page }) => {
      await expect(page.getByRole('heading', { name: /choose your race/i })).toBeVisible()
    })

    test('displays search input', async ({ page }) => {
      await expect(page.locator('input[placeholder*="Search races"]')).toBeVisible()
    })

    test('displays Human race card', async ({ page }) => {
      await expect(page.getByText('Human')).toBeVisible()
    })

    test('can search for Human', async ({ page }) => {
      await page.locator('input[placeholder*="Search races"]').fill('Human')
      await expect(page.getByText('Human')).toBeVisible()
    })

    test('can select Human race', async ({ page }) => {
      // Click on the Human card
      await page.getByText('Human').first().click()

      // The "Continue with Human" button should appear
      await expect(page.getByRole('button', { name: /continue with human/i })).toBeVisible()
    })

    test('Continue button is disabled until race selected', async ({ page }) => {
      // Initially no race selected - look for a disabled continue button
      const continueButton = page.getByRole('button', { name: /continue with/i })
      // The button text should show "Continue with Selection" when nothing selected
      // and be disabled
      await expect(page.getByRole('button', { name: /continue with selection/i })).toBeDisabled()
    })

    test('selecting Human and continuing skips subrace step', async ({ page }) => {
      // Select Human
      await page.getByText('Human').first().click()

      // Click Continue
      await page.getByRole('button', { name: /continue with human/i }).click()
      await waitForLoading(page)

      // Should skip subrace and go directly to class (Human has no subraces)
      await expect(page).toHaveURL(/\/characters\/new\/class/)
    })
  })
```

**Step 2: Run the test**

Run: `docker compose exec nuxt npx playwright test character-wizard --grep "Race"`
Expected: Tests pass

**Step 3: Commit**

```bash
git add tests/e2e/character-wizard.spec.ts
git commit -m "test(e2e): add race step tests with Human selection"
```

---

## Task 4: Test Class Step (Bard Selection)

**Files:**
- Modify: `tests/e2e/character-wizard.spec.ts`

**Step 1: Add class step tests**

```typescript
  test.describe('Step 3: Class', () => {
    test.beforeEach(async ({ page }) => {
      await goToStep(page, 'class')
    })

    test('displays class selection with correct title', async ({ page }) => {
      await expect(page.getByRole('heading', { name: /choose your class/i })).toBeVisible()
    })

    test('displays search input', async ({ page }) => {
      await expect(page.locator('input[placeholder*="Search classes"]')).toBeVisible()
    })

    test('displays Bard class card', async ({ page }) => {
      await expect(page.getByText('Bard')).toBeVisible()
    })

    test('can search for Bard', async ({ page }) => {
      await page.locator('input[placeholder*="Search classes"]').fill('Bard')
      await expect(page.getByText('Bard')).toBeVisible()
    })

    test('shows spellcasting info when Bard selected', async ({ page }) => {
      // Click on Bard
      await page.getByText('Bard').first().click()

      // Should show info that Bard is a spellcaster
      await expect(page.getByText(/spellcasting class/i)).toBeVisible()
    })

    test('can select Bard and continue', async ({ page }) => {
      // Select Bard
      await page.getByText('Bard').first().click()

      // Click Continue
      await page.getByRole('button', { name: /continue with bard/i }).click()
      await waitForLoading(page)

      // Should skip subclass (Bard picks at level 3) and go to background
      await expect(page).toHaveURL(/\/characters\/new\/background/)
    })
  })
```

**Step 2: Run the test**

Run: `docker compose exec nuxt npx playwright test character-wizard --grep "Class"`
Expected: Tests pass

**Step 3: Commit**

```bash
git add tests/e2e/character-wizard.spec.ts
git commit -m "test(e2e): add class step tests with Bard selection"
```

---

## Task 5: Test Background Step (Acolyte Selection)

**Files:**
- Modify: `tests/e2e/character-wizard.spec.ts`

**Step 1: Add background step tests**

```typescript
  test.describe('Step 4: Background', () => {
    test.beforeEach(async ({ page }) => {
      await goToStep(page, 'background')
    })

    test('displays background selection with correct title', async ({ page }) => {
      await expect(page.getByRole('heading', { name: /choose your background/i })).toBeVisible()
    })

    test('displays Acolyte background', async ({ page }) => {
      await expect(page.getByText('Acolyte')).toBeVisible()
    })

    test('can search for Acolyte', async ({ page }) => {
      await page.locator('input[placeholder*="Search backgrounds"]').fill('Acolyte')
      await expect(page.getByText('Acolyte')).toBeVisible()
    })

    test('can select Acolyte and continue', async ({ page }) => {
      // Select Acolyte
      await page.getByText('Acolyte').first().click()

      // Click Continue
      await page.getByRole('button', { name: /continue with acolyte/i }).click()
      await waitForLoading(page)

      // Should go to abilities
      await expect(page).toHaveURL(/\/characters\/new\/abilities/)
    })
  })
```

**Step 2: Run the test**

Run: `docker compose exec nuxt npx playwright test character-wizard --grep "Background"`
Expected: Tests pass

**Step 3: Commit**

```bash
git add tests/e2e/character-wizard.spec.ts
git commit -m "test(e2e): add background step tests with Acolyte selection"
```

---

## Task 6: Test Abilities Step (Manual Entry)

**Files:**
- Modify: `tests/e2e/character-wizard.spec.ts`

**Step 1: Add abilities step tests**

```typescript
  test.describe('Step 5: Abilities', () => {
    test.beforeEach(async ({ page }) => {
      await goToStep(page, 'abilities')
    })

    test('displays abilities step with correct title', async ({ page }) => {
      await expect(page.getByRole('heading', { name: /assign ability scores/i })).toBeVisible()
    })

    test('displays method selector with three options', async ({ page }) => {
      await expect(page.getByRole('button', { name: 'Standard Array' })).toBeVisible()
      await expect(page.getByRole('button', { name: 'Point Buy' })).toBeVisible()
      await expect(page.getByRole('button', { name: 'Manual' })).toBeVisible()
    })

    test('can switch to Manual method', async ({ page }) => {
      // Click Manual button
      await page.getByRole('button', { name: 'Manual' }).click()

      // Should show number inputs for each ability
      await expect(page.getByText(/enter your ability scores/i)).toBeVisible()
    })

    test('can enter ability scores manually', async ({ page }) => {
      // Switch to Manual
      await page.getByRole('button', { name: 'Manual' }).click()

      // Fill in ability scores using data-testid
      await page.locator('[data-testid="input-strength"]').fill('8')
      await page.locator('[data-testid="input-dexterity"]').fill('14')
      await page.locator('[data-testid="input-constitution"]').fill('13')
      await page.locator('[data-testid="input-intelligence"]').fill('10')
      await page.locator('[data-testid="input-wisdom"]').fill('12')
      await page.locator('[data-testid="input-charisma"]').fill('15')

      // Save button should be enabled
      await expect(page.locator('[data-testid="save-abilities"]')).toBeEnabled()
    })

    test('validates ability score range (3-20)', async ({ page }) => {
      // Switch to Manual
      await page.getByRole('button', { name: 'Manual' }).click()

      // Enter invalid score (too low)
      await page.locator('[data-testid="input-strength"]').fill('2')

      // Save button should be disabled
      await expect(page.locator('[data-testid="save-abilities"]')).toBeDisabled()
    })

    test('can save abilities and continue', async ({ page }) => {
      // Switch to Manual
      await page.getByRole('button', { name: 'Manual' }).click()

      // Fill valid scores
      await page.locator('[data-testid="input-strength"]').fill('8')
      await page.locator('[data-testid="input-dexterity"]').fill('14')
      await page.locator('[data-testid="input-constitution"]').fill('13')
      await page.locator('[data-testid="input-intelligence"]').fill('10')
      await page.locator('[data-testid="input-wisdom"]').fill('12')
      await page.locator('[data-testid="input-charisma"]').fill('15')

      // Click Save & Continue
      await page.locator('[data-testid="save-abilities"]').click()
      await waitForLoading(page)

      // Should proceed to proficiencies
      await expect(page).toHaveURL(/\/characters\/new\/proficiencies/)
    })
  })
```

**Step 2: Run the test**

Run: `docker compose exec nuxt npx playwright test character-wizard --grep "Abilities"`
Expected: Tests pass

**Step 3: Commit**

```bash
git add tests/e2e/character-wizard.spec.ts
git commit -m "test(e2e): add abilities step tests with manual entry"
```

---

## Task 7: Test Remaining Steps (Proficiencies through Review)

**Files:**
- Modify: `tests/e2e/character-wizard.spec.ts`

**Step 1: Add remaining step tests**

```typescript
  test.describe('Step 6: Proficiencies', () => {
    test.beforeEach(async ({ page }) => {
      await goToStep(page, 'proficiencies')
    })

    test('displays proficiencies step', async ({ page }) => {
      // Should show proficiency choices from class/background
      await expect(page.getByRole('heading', { name: /proficiencies/i })).toBeVisible()
    })

    test('can proceed through proficiencies', async ({ page }) => {
      // Select any required proficiencies (Bard gets skill choices)
      // Then proceed
      await clickNextAndWait(page)

      // Should go to languages or equipment
      await expect(page).toHaveURL(/\/characters\/new\/(languages|equipment)/)
    })
  })

  test.describe('Step 7: Languages', () => {
    test('displays languages step if choices available', async ({ page }) => {
      await goToStep(page, 'languages')
      // May be skipped if no language choices
      // Check we're either on languages or moved past
      const url = page.url()
      if (url.includes('/languages')) {
        await expect(page.getByRole('heading', { name: /languages/i })).toBeVisible()
      }
    })
  })

  test.describe('Step 8: Equipment', () => {
    test.beforeEach(async ({ page }) => {
      await goToStep(page, 'equipment')
    })

    test('displays equipment step', async ({ page }) => {
      await expect(page.getByRole('heading', { name: /equipment/i })).toBeVisible()
    })

    test('can proceed through equipment', async ({ page }) => {
      await clickNextAndWait(page)
      // Bard is spellcaster, should go to spells
      await expect(page).toHaveURL(/\/characters\/new\/spells/)
    })
  })

  test.describe('Step 9: Spells (Bard)', () => {
    test.beforeEach(async ({ page }) => {
      await goToStep(page, 'spells')
    })

    test('displays spells step for spellcaster', async ({ page }) => {
      await expect(page.getByRole('heading', { name: /spells/i })).toBeVisible()
    })

    test('shows spell selection options', async ({ page }) => {
      // Bard should have cantrip and spell slots at level 1
      await expect(page.getByText(/cantrip/i)).toBeVisible()
    })

    test('can proceed through spells', async ({ page }) => {
      await clickNextAndWait(page)
      await expect(page).toHaveURL(/\/characters\/new\/details/)
    })
  })

  test.describe('Step 10: Details', () => {
    test.beforeEach(async ({ page }) => {
      await goToStep(page, 'details')
    })

    test('displays details step with name input', async ({ page }) => {
      await expect(page.getByRole('heading', { name: /details/i })).toBeVisible()
      await expect(page.locator('input[type="text"]').first()).toBeVisible()
    })

    test('can enter character name', async ({ page }) => {
      await page.locator('input[type="text"]').first().fill(TEST_CHARACTER_NAME)

      // Name should be filled
      await expect(page.locator('input[type="text"]').first()).toHaveValue(TEST_CHARACTER_NAME)
    })

    test('can proceed to review', async ({ page }) => {
      await page.locator('input[type="text"]').first().fill(TEST_CHARACTER_NAME)
      await clickNextAndWait(page)
      await expect(page).toHaveURL(/\/characters\/new\/review/)
    })
  })

  test.describe('Step 11: Review', () => {
    test.beforeEach(async ({ page }) => {
      await goToStep(page, 'review')
    })

    test('displays review step', async ({ page }) => {
      await expect(page.getByRole('heading', { name: /review/i })).toBeVisible()
    })

    test('shows Finish button instead of Next', async ({ page }) => {
      await expect(getNextButton(page)).toContainText(/finish/i)
    })
  })
```

**Step 2: Run all remaining step tests**

Run: `docker compose exec nuxt npx playwright test character-wizard --grep "(Proficiencies|Languages|Equipment|Spells|Details|Review)"`
Expected: Tests pass

**Step 3: Commit**

```bash
git add tests/e2e/character-wizard.spec.ts
git commit -m "test(e2e): add remaining wizard step tests"
```

---

## Task 8: Add Full Flow Integration Test

**Files:**
- Modify: `tests/e2e/character-wizard.spec.ts`

**Step 1: Add complete happy path test**

Add at the end of the describe block:

```typescript
  test.describe('Complete Character Creation Flow', () => {
    test('creates a Human Bard with manual abilities from start to finish', async ({ page }) => {
      // This is the full happy path test
      test.setTimeout(120000) // 2 minutes for full flow

      // ══════════════════════════════════════════════════════════
      // STEP 1: SOURCEBOOKS
      // ══════════════════════════════════════════════════════════
      await page.goto(`${WIZARD_BASE_URL}/sourcebooks`)
      await waitForLoading(page)

      // PHB should be pre-selected, just continue
      await clickNextAndWait(page)
      await expect(page).toHaveURL(/\/race/)

      // ══════════════════════════════════════════════════════════
      // STEP 2: RACE - Select Human
      // ══════════════════════════════════════════════════════════
      await page.getByText('Human').first().click()
      await page.getByRole('button', { name: /continue with human/i }).click()
      await waitForLoading(page)

      // Should skip subrace (Human has none) → class
      await expect(page).toHaveURL(/\/class/)

      // ══════════════════════════════════════════════════════════
      // STEP 3: CLASS - Select Bard
      // ══════════════════════════════════════════════════════════
      await page.getByText('Bard').first().click()
      await page.getByRole('button', { name: /continue with bard/i }).click()
      await waitForLoading(page)

      // Should skip subclass (Bard picks at level 3) → background
      await expect(page).toHaveURL(/\/background/)

      // ══════════════════════════════════════════════════════════
      // STEP 4: BACKGROUND - Select Acolyte
      // ══════════════════════════════════════════════════════════
      await page.getByText('Acolyte').first().click()
      await page.getByRole('button', { name: /continue with acolyte/i }).click()
      await waitForLoading(page)

      await expect(page).toHaveURL(/\/abilities/)

      // ══════════════════════════════════════════════════════════
      // STEP 5: ABILITIES - Manual Entry
      // ══════════════════════════════════════════════════════════
      await page.getByRole('button', { name: 'Manual' }).click()

      // Enter ability scores for a Bard (high CHA)
      await page.locator('[data-testid="input-strength"]').fill('8')
      await page.locator('[data-testid="input-dexterity"]').fill('14')
      await page.locator('[data-testid="input-constitution"]').fill('13')
      await page.locator('[data-testid="input-intelligence"]').fill('10')
      await page.locator('[data-testid="input-wisdom"]').fill('12')
      await page.locator('[data-testid="input-charisma"]').fill('15')

      await page.locator('[data-testid="save-abilities"]').click()
      await waitForLoading(page)

      await expect(page).toHaveURL(/\/proficiencies/)

      // ══════════════════════════════════════════════════════════
      // STEP 6: PROFICIENCIES
      // ══════════════════════════════════════════════════════════
      // Select any required proficiencies then continue
      await clickNextAndWait(page)

      // May go to languages or skip to equipment
      const afterProfUrl = page.url()

      // ══════════════════════════════════════════════════════════
      // STEP 7: LANGUAGES (if visible)
      // ══════════════════════════════════════════════════════════
      if (afterProfUrl.includes('/languages')) {
        await clickNextAndWait(page)
      }

      // ══════════════════════════════════════════════════════════
      // STEP 8: EQUIPMENT
      // ══════════════════════════════════════════════════════════
      if (page.url().includes('/equipment')) {
        await clickNextAndWait(page)
      }

      // ══════════════════════════════════════════════════════════
      // STEP 9: SPELLS (Bard is spellcaster)
      // ══════════════════════════════════════════════════════════
      await expect(page).toHaveURL(/\/spells/)
      await clickNextAndWait(page)

      // ══════════════════════════════════════════════════════════
      // STEP 10: DETAILS
      // ══════════════════════════════════════════════════════════
      await expect(page).toHaveURL(/\/details/)
      await page.locator('input[type="text"]').first().fill(TEST_CHARACTER_NAME)
      await clickNextAndWait(page)

      // ══════════════════════════════════════════════════════════
      // STEP 11: REVIEW
      // ══════════════════════════════════════════════════════════
      await expect(page).toHaveURL(/\/review/)

      // Verify key information is displayed
      await expect(page.getByText('Human')).toBeVisible()
      await expect(page.getByText('Bard')).toBeVisible()
      await expect(page.getByText('Acolyte')).toBeVisible()
      await expect(page.getByText(TEST_CHARACTER_NAME)).toBeVisible()

      // Verify Finish button
      await expect(getNextButton(page)).toContainText(/finish/i)

      // Complete the wizard
      await getNextButton(page).click()
      await waitForLoading(page)

      // Should redirect to characters list
      await expect(page).toHaveURL(/\/characters/)
    })
  })
```

**Step 2: Run the complete flow test**

Run: `docker compose exec nuxt npx playwright test character-wizard --grep "Complete Character Creation Flow"`
Expected: Test passes (may take ~1-2 minutes)

**Step 3: Commit**

```bash
git add tests/e2e/character-wizard.spec.ts
git commit -m "test(e2e): add complete character creation flow integration test"
```

---

## Task 9: Add Navigation and Edge Case Tests

**Files:**
- Modify: `tests/e2e/character-wizard.spec.ts`

**Step 1: Add navigation and edge case tests**

Add within the main describe block:

```typescript
  test.describe('Navigation', () => {
    test('Back button is disabled on first step', async ({ page }) => {
      await goToStep(page, 'sourcebooks')
      await expect(getBackButton(page)).toBeDisabled()
    })

    test('Back button works on subsequent steps', async ({ page }) => {
      await goToStep(page, 'race')
      await expect(getBackButton(page)).toBeEnabled()

      await getBackButton(page).click()
      await expect(page).toHaveURL(/\/sourcebooks/)
    })

    test('can navigate back and forth without losing data', async ({ page }) => {
      // Start at race, select Human
      await goToStep(page, 'race')
      await page.getByText('Human').first().click()
      await page.getByRole('button', { name: /continue with human/i }).click()
      await waitForLoading(page)

      // Now at class, go back
      await getBackButton(page).click()
      await waitForLoading(page)

      // Should still show Human as selected
      await expect(page.getByRole('button', { name: /continue with human/i })).toBeVisible()
    })

    test('direct URL navigation works for valid steps', async ({ page }) => {
      await page.goto(`${WIZARD_BASE_URL}/abilities`)
      await expect(page.getByRole('heading', { name: /assign ability scores/i })).toBeVisible()
    })

    test('invalid step shows 404', async ({ page }) => {
      const response = await page.goto(`${WIZARD_BASE_URL}/invalid-step`)
      expect(response?.status()).toBe(404)
    })
  })

  test.describe('Sidebar Progress', () => {
    test('shows progress indicator', async ({ page }) => {
      await goToStep(page, 'sourcebooks')
      await expect(page.locator('[data-test="progress-bar"]')).toBeVisible()
    })

    test('clicking completed step navigates there', async ({ page }) => {
      // Complete sourcebooks
      await goToStep(page, 'sourcebooks')
      await clickNextAndWait(page)

      // Now on race, click back to sourcebooks in sidebar
      await page.locator('[data-test="step-item-sourcebooks"]').click()
      await expect(page).toHaveURL(/\/sourcebooks/)
    })
  })
```

**Step 2: Run navigation tests**

Run: `docker compose exec nuxt npx playwright test character-wizard --grep "(Navigation|Sidebar)"`
Expected: Tests pass

**Step 3: Commit**

```bash
git add tests/e2e/character-wizard.spec.ts
git commit -m "test(e2e): add navigation and sidebar tests"
```

---

## Task 10: Final Verification and Documentation

**Step 1: Run the complete test suite**

Run: `docker compose exec nuxt npx playwright test character-wizard`
Expected: All tests pass

**Step 2: Generate test report**

Run: `docker compose exec nuxt npx playwright show-report`
Expected: HTML report opens showing all test results

**Step 3: Update package.json with test script (optional)**

If desired, add a convenience script:

```json
{
  "scripts": {
    "test:e2e:wizard": "npx playwright test character-wizard"
  }
}
```

**Step 4: Final commit with all tests**

```bash
git add -A
git commit -m "test(e2e): complete character wizard E2E test suite

- Sourcebooks step tests
- Race selection (Human) with subrace skip verification
- Class selection (Bard) with spellcaster detection
- Background selection (Acolyte)
- Abilities with manual entry and validation
- Proficiencies, Languages, Equipment steps
- Spells step (Bard-specific)
- Details step with name entry
- Review step with Finish button
- Complete flow integration test
- Navigation and sidebar tests"
```

---

## Summary

| Task | Description | Files |
|------|-------------|-------|
| 1 | Test file setup with helpers | `tests/e2e/character-wizard.spec.ts` |
| 2 | Sourcebooks step tests | Same |
| 3 | Race step tests (Human) | Same |
| 4 | Class step tests (Bard) | Same |
| 5 | Background step tests (Acolyte) | Same |
| 6 | Abilities step tests (Manual) | Same |
| 7 | Remaining step tests | Same |
| 8 | Full flow integration test | Same |
| 9 | Navigation & edge case tests | Same |
| 10 | Final verification | Same |

**Total Estimated Tasks:** 10 bite-sized tasks
**Test Count:** ~35+ individual test cases
**Coverage:** Full wizard flow from sourcebooks to character completion
