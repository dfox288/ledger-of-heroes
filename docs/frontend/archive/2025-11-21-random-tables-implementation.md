# Random Tables Visualization Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add component to display D&D random tables (dice rolls, results) for background traits with full TDD workflow.

**Architecture:** Create new `UiAccordionRandomTablesList.vue` component that renders HTML tables, integrate into existing `UiAccordionTraitsList.vue` to display tables inline after trait descriptions.

**Tech Stack:** Vue 3 Composition API, TypeScript, Vitest, @nuxt/test-utils, NuxtUI 4, Tailwind CSS

---

## Task 1: Create UiAccordionRandomTablesList Component Tests (RED)

**Files:**
- Create: `tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts`

**Step 1: Write the test file with basic structure and first 3 tests**

Create the test file with imports and test data:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionRandomTablesList from '~/components/ui/accordion/UiAccordionRandomTablesList.vue'

describe('UiAccordionRandomTablesList', () => {
  const mockTables = [
    {
      id: 1,
      table_name: 'Entertainer Routines',
      dice_type: 'd10',
      description: 'Choose your performance style',
      entries: [
        {
          id: 1,
          roll_min: 1,
          roll_max: 1,
          result_text: 'Actor',
          sort_order: 0
        },
        {
          id: 2,
          roll_min: 2,
          roll_max: 2,
          result_text: 'Dancer',
          sort_order: 1
        },
        {
          id: 3,
          roll_min: 3,
          roll_max: 5,
          result_text: 'Fire-eater',
          sort_order: 2
        }
      ]
    }
  ]

  const mockMultipleTables = [
    mockTables[0],
    {
      id: 2,
      table_name: 'Personality Trait',
      dice_type: 'd8',
      description: null,
      entries: [
        {
          id: 4,
          roll_min: 1,
          roll_max: 1,
          result_text: 'I know a story relevant to almost every situation.',
          sort_order: 0
        }
      ]
    }
  ]

  it('renders table name and dice type', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    expect(wrapper.text()).toContain('Entertainer Routines')
    expect(wrapper.text()).toContain('(d10)')
  })

  it('displays table description when present', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    expect(wrapper.text()).toContain('Choose your performance style')
  })

  it('hides description when null', async () => {
    const tableWithoutDescription = [{
      ...mockTables[0],
      description: null
    }]

    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: tableWithoutDescription }
    })

    expect(wrapper.find('p').exists()).toBe(false)
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- UiAccordionRandomTablesList.test.ts`

Expected: FAIL with "Cannot find module '~/components/ui/accordion/UiAccordionRandomTablesList.vue'"

**Step 3: Add remaining table structure tests**

Add these tests to the same describe block:

```typescript
  it('renders table header correctly', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    const headers = wrapper.findAll('th')
    expect(headers).toHaveLength(2)
    expect(headers[0].text()).toContain('Roll')
    expect(headers[1].text()).toContain('Result')
  })

  it('renders all table entries', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    const rows = wrapper.findAll('tbody tr')
    expect(rows).toHaveLength(3)
  })

  it('formats single roll correctly', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    const firstRow = wrapper.findAll('tbody tr')[0]
    const rollCell = firstRow.findAll('td')[0]
    expect(rollCell.text()).toBe('1')
  })

  it('formats roll range correctly', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    const thirdRow = wrapper.findAll('tbody tr')[2]
    const rollCell = thirdRow.findAll('td')[0]
    expect(rollCell.text()).toBe('3-5')
  })

  it('renders result text correctly', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    const firstRow = wrapper.findAll('tbody tr')[0]
    const resultCell = firstRow.findAll('td')[1]
    expect(resultCell.text()).toBe('Actor')
  })

  it('handles multiple tables with proper spacing', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockMultipleTables }
    })

    const tables = wrapper.findAll('table')
    expect(tables).toHaveLength(2)

    // Verify both table names appear
    expect(wrapper.text()).toContain('Entertainer Routines')
    expect(wrapper.text()).toContain('Personality Trait')
  })

  it('handles empty tables array gracefully', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: [] }
    })

    expect(wrapper.find('table').exists()).toBe(false)
  })

  it('maintains sort order', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    const rows = wrapper.findAll('tbody tr')
    const results = rows.map(row => row.findAll('td')[1].text())

    expect(results).toEqual(['Actor', 'Dancer', 'Fire-eater'])
  })

  it('component mounts without errors', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    expect(wrapper.exists()).toBe(true)
  })

  it('accepts borderColor prop', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: {
        tables: mockTables,
        borderColor: 'purple-500'
      }
    })

    // Component should accept the prop without error
    expect(wrapper.exists()).toBe(true)
  })
```

**Step 4: Run tests again to verify they still fail**

Run: `docker compose exec nuxt npm run test -- UiAccordionRandomTablesList.test.ts`

Expected: FAIL with "Cannot find module" (component doesn't exist yet)

**Step 5: Commit the test file**

```bash
git add tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts
git commit -m "test: Add UiAccordionRandomTablesList tests (RED phase)

- Created 13 comprehensive tests for random tables component
- Tests cover table rendering, formatting, and edge cases
- All tests currently failing (component not created yet)
- Following TDD workflow

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Create UiAccordionRandomTablesList Component (GREEN)

**Files:**
- Create: `app/components/ui/accordion/UiAccordionRandomTablesList.vue`

**Step 1: Create component file with TypeScript interface**

```vue
<script setup lang="ts">
interface RandomTableEntry {
  id: number
  roll_min: number
  roll_max: number
  result_text: string
  sort_order: number
}

interface RandomTable {
  id: number
  table_name: string
  dice_type: string
  description?: string | null
  entries: RandomTableEntry[]
}

interface Props {
  tables: RandomTable[]
  borderColor?: string
}

const props = withDefaults(defineProps<Props>(), {
  borderColor: 'primary-500'
})

const formatRollRange = (min: number, max: number): string => {
  return min === max ? `${min}` : `${min}-${max}`
}
</script>

<template>
  <div v-if="tables.length > 0" class="space-y-6 pl-4">
    <div v-for="table in tables" :key="table.id" class="space-y-2">
      <!-- Table Name and Dice Type -->
      <h4 class="text-base font-semibold text-gray-900 dark:text-gray-100">
        {{ table.table_name }}
        <span class="text-sm font-normal text-gray-600 dark:text-gray-400">
          ({{ table.dice_type }})
        </span>
      </h4>

      <!-- Optional Description -->
      <p v-if="table.description" class="text-sm text-gray-700 dark:text-gray-300">
        {{ table.description }}
      </p>

      <!-- HTML Table -->
      <table class="min-w-full border border-gray-200 dark:border-gray-700 rounded-lg overflow-hidden">
        <thead class="bg-gray-50 dark:bg-gray-800">
          <tr>
            <th class="px-4 py-2 text-left text-xs font-semibold uppercase tracking-wider text-gray-700 dark:text-gray-300 w-24">
              Roll
            </th>
            <th class="px-4 py-2 text-left text-xs font-semibold uppercase tracking-wider text-gray-700 dark:text-gray-300">
              Result
            </th>
          </tr>
        </thead>
        <tbody class="divide-y divide-gray-200 dark:divide-gray-700 bg-white dark:bg-gray-900">
          <tr
            v-for="entry in table.entries"
            :key="entry.id"
            class="hover:bg-gray-50 dark:hover:bg-gray-800/50 transition-colors"
          >
            <td class="px-4 py-3 text-sm font-medium text-gray-900 dark:text-gray-100">
              {{ formatRollRange(entry.roll_min, entry.roll_max) }}
            </td>
            <td class="px-4 py-3 text-sm text-gray-700 dark:text-gray-300">
              {{ entry.result_text }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>
```

**Step 2: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- UiAccordionRandomTablesList.test.ts`

Expected: PASS - All 13 tests should pass

**Step 3: Commit the component**

```bash
git add app/components/ui/accordion/UiAccordionRandomTablesList.vue
git commit -m "feat: Create UiAccordionRandomTablesList component (GREEN phase)

- Implemented component to display random tables
- HTML table format with borders and dark mode
- formatRollRange helper for roll display (1 vs 1-3)
- Handles empty arrays, null descriptions
- All 13 tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Update UiAccordionTraitsList Tests (RED)

**Files:**
- Modify: `tests/components/ui/accordion/UiAccordionTraitsList.test.ts`

**Step 1: Check existing test file structure**

Run: `cat tests/components/ui/accordion/UiAccordionTraitsList.test.ts`

Expected: See existing tests for the traits list component

**Step 2: Add integration tests for random tables**

Add these tests to the existing describe block:

```typescript
  it('renders random tables when trait has them', async () => {
    const traitsWithTables = [
      {
        id: 1,
        name: 'Test Trait',
        description: 'A trait with tables',
        random_tables: [
          {
            id: 1,
            table_name: 'Test Table',
            dice_type: 'd10',
            description: null,
            entries: [
              {
                id: 1,
                roll_min: 1,
                roll_max: 1,
                result_text: 'Result 1',
                sort_order: 0
              }
            ]
          }
        ]
      }
    ]

    const wrapper = await mountSuspended(UiAccordionTraitsList, {
      props: { traits: traitsWithTables }
    })

    // Verify random tables component is rendered
    expect(wrapper.text()).toContain('Test Table')
    expect(wrapper.text()).toContain('(d10)')
    expect(wrapper.text()).toContain('Result 1')
  })

  it('does not render random tables when array is empty', async () => {
    const traitsWithoutTables = [
      {
        id: 1,
        name: 'Test Trait',
        description: 'A trait without tables',
        random_tables: []
      }
    ]

    const wrapper = await mountSuspended(UiAccordionTraitsList, {
      props: { traits: traitsWithoutTables }
    })

    // Should not render any table elements
    expect(wrapper.find('table').exists()).toBe(false)
  })

  it('passes borderColor to random tables component', async () => {
    const traitsWithTables = [
      {
        id: 1,
        name: 'Test Trait',
        description: 'A trait with tables',
        random_tables: [
          {
            id: 1,
            table_name: 'Test Table',
            dice_type: 'd6',
            description: null,
            entries: [
              {
                id: 1,
                roll_min: 1,
                roll_max: 1,
                result_text: 'Result',
                sort_order: 0
              }
            ]
          }
        ]
      }
    ]

    const wrapper = await mountSuspended(UiAccordionTraitsList, {
      props: {
        traits: traitsWithTables,
        borderColor: 'purple-500'
      }
    })

    // Component should render without errors
    expect(wrapper.exists()).toBe(true)
    expect(wrapper.text()).toContain('Test Table')
  })
```

**Step 3: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- UiAccordionTraitsList.test.ts`

Expected: FAIL - New tests fail because component doesn't integrate random tables yet

**Step 4: Commit the updated tests**

```bash
git add tests/components/ui/accordion/UiAccordionTraitsList.test.ts
git commit -m "test: Add random tables integration tests (RED phase)

- Added 3 integration tests for UiAccordionTraitsList
- Tests verify random tables render when present
- Tests verify nothing renders when empty
- All new tests currently failing (integration not done)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Integrate Random Tables into UiAccordionTraitsList (GREEN)

**Files:**
- Modify: `app/components/ui/accordion/UiAccordionTraitsList.vue`

**Step 1: Update TypeScript interface**

Find the existing interface (around line 2-9) and add the `random_tables` field:

```typescript
interface RandomTableEntry {
  id: number
  roll_min: number
  roll_max: number
  result_text: string
  sort_order: number
}

interface RandomTable {
  id: number
  table_name: string
  dice_type: string
  description?: string | null
  entries: RandomTableEntry[]
}

interface Trait {
  id: number
  name: string
  description: string
  level?: number
  category?: string
  feature_name?: string
  random_tables?: RandomTable[]  // NEW
}
```

**Step 2: Update template to render random tables**

Find the template section and modify the loop structure. Change from:

```vue
<template>
  <div class="p-4 space-y-4">
    <div
      v-for="trait in traits"
      :key="trait.id"
      class="border-l-4 pl-4 py-2"
      :class="`border-${borderColor}`"
    >
      <!-- existing content -->
    </div>
  </div>
</template>
```

To:

```vue
<template>
  <div class="p-4 space-y-4">
    <div v-for="trait in traits" :key="trait.id" class="space-y-3">
      <!-- Existing trait display -->
      <div class="border-l-4 pl-4 py-2" :class="`border-${borderColor}`">
        <div class="flex items-center gap-2 mb-1">
          <UBadge v-if="showLevel && trait.level" color="info" variant="soft" size="xs">
            Level {{ trait.level }}
          </UBadge>
          <span class="font-semibold text-gray-900 dark:text-gray-100">
            {{ trait.feature_name || trait.name }}
          </span>
          <UBadge v-if="showCategory && trait.category" color="purple" variant="soft" size="xs">
            {{ trait.category }}
          </UBadge>
        </div>
        <div v-if="trait.description" class="text-sm text-gray-700 dark:text-gray-300 whitespace-pre-line">
          {{ trait.description }}
        </div>
      </div>

      <!-- NEW: Random tables display -->
      <UiAccordionRandomTablesList
        v-if="trait.random_tables && trait.random_tables.length > 0"
        :tables="trait.random_tables"
        :borderColor="borderColor"
      />
    </div>
  </div>
</template>
```

**Step 3: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- UiAccordionTraitsList.test.ts`

Expected: PASS - All tests including new integration tests should pass

**Step 4: Run full test suite to check for regressions**

Run: `docker compose exec nuxt npm run test`

Expected: PASS - All tests across entire project should pass

**Step 5: Commit the integration**

```bash
git add app/components/ui/accordion/UiAccordionTraitsList.vue
git commit -m "feat: Integrate random tables into traits display (GREEN phase)

- Updated TypeScript interface to include random_tables
- Added UiAccordionRandomTablesList to template
- Tables render inline after trait descriptions
- Changed spacing from space-y-4 to space-y-3 for grouping
- All integration tests passing
- No regressions in full test suite

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Manual Browser Testing

**Files:**
- Test: `http://localhost:3000/backgrounds/entertainer`

**Step 1: Ensure Docker containers are running**

Run: `docker compose ps`

Expected: See `nuxt` and backend services running

If not running:
```bash
docker compose up -d
```

**Step 2: Navigate to entertainer background**

Open browser: `http://localhost:3000/backgrounds/entertainer`

Expected: Page loads successfully

**Step 3: Verify random tables render correctly**

Check for these 5 tables (scroll down to Background Traits section):
1. ✅ Entertainer Routines (d10) - 10 entries
2. ✅ Personality Trait (d8) - 8 entries
3. ✅ Ideal (d6) - 6 entries
4. ✅ Bond (d6) - 6 entries
5. ✅ Flaw (d6) - 6 entries

Verify each table has:
- ✅ Table name as heading with dice type in parentheses
- ✅ "Roll" and "Result" column headers
- ✅ Proper roll formatting (e.g., "1", "2", not "1-1", "2-2")
- ✅ Result text displays correctly
- ✅ Border styling matches design
- ✅ Proper spacing between tables

**Step 4: Test dark mode**

Click dark mode toggle in navbar

Verify:
- ✅ Table backgrounds invert properly
- ✅ Text remains readable
- ✅ Borders have good contrast
- ✅ Hover states work

Click dark mode toggle again to return to light mode

**Step 5: Test responsive behavior**

Resize browser window to test breakpoints:

**Mobile (375px):**
- Open DevTools (F12)
- Toggle device toolbar (Ctrl+Shift+M / Cmd+Shift+M)
- Select "iPhone SE" or similar
- Verify tables remain readable (narrow Roll column helps)

**Tablet (768px):**
- Select "iPad Mini" or similar
- Verify layout looks good

**Desktop (1440px):**
- Disable device toolbar
- Verify tables don't stretch too wide

**Step 6: Test other backgrounds**

Navigate to: `http://localhost:3000/backgrounds`

Click on different backgrounds to verify:
- ✅ Backgrounds without random tables don't show errors
- ✅ Backgrounds with tables display them correctly
- ✅ No console errors

Test at least 2-3 other backgrounds

**Step 7: Check browser console for errors**

Open DevTools Console tab

Expected: No errors or warnings

**Step 8: Document manual test results**

Create a simple checklist file:

```bash
cat > /tmp/manual-test-results.txt << 'EOF'
Manual Testing Results - Random Tables Feature
===============================================

Date: $(date)

Entertainer Background:
[✓] All 5 tables render correctly
[✓] Table names and dice types display
[✓] Roll ranges format correctly (1, 2, 3 not 1-1, 2-2)
[✓] Result text appears
[✓] Styling matches design

Dark Mode:
[✓] Tables display correctly in dark mode
[✓] Text contrast is good
[✓] Borders visible
[✓] Hover states work

Responsive:
[✓] Mobile (375px) - tables readable
[✓] Tablet (768px) - layout good
[✓] Desktop (1440px) - tables look good

Other Backgrounds:
[✓] Tested 3+ other backgrounds
[✓] No errors on backgrounds without tables
[✓] Tables display when present

Console:
[✓] No errors or warnings

Overall: PASS ✅
EOF

cat /tmp/manual-test-results.txt
```

**Step 9: Take screenshots (optional but recommended)**

```bash
# Create screenshots directory if it doesn't exist
mkdir -p docs/screenshots

# Manual step: Take screenshots and save to docs/screenshots/
# - random-tables-light-mode.png
# - random-tables-dark-mode.png
# - random-tables-mobile.png
```

Note: Screenshots should be taken manually in the browser

---

## Task 6: Final Verification and Commit

**Files:**
- All modified files

**Step 1: Run TypeScript type checking**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No type errors

**Step 2: Run linter**

Run: `docker compose exec nuxt npm run lint`

Expected: No linting errors

If there are auto-fixable issues:
```bash
docker compose exec nuxt npm run lint:fix
```

**Step 3: Run full test suite one final time**

Run: `docker compose exec nuxt npm run test`

Expected: All tests pass

**Step 4: Review all changes**

Run: `git status`

Expected: See all modified files

Run: `git diff`

Review changes to ensure everything looks correct

**Step 5: Final commit (if any linting fixes were made)**

If lint:fix made changes:
```bash
git add -A
git commit -m "style: Apply linting fixes for random tables feature

- Auto-fixed linting issues
- No functional changes

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

**Step 6: Verify git log shows complete implementation**

Run: `git log --oneline -10`

Expected: See all commits from this implementation:
1. test: Add UiAccordionRandomTablesList tests (RED phase)
2. feat: Create UiAccordionRandomTablesList component (GREEN phase)
3. test: Add random tables integration tests (RED phase)
4. feat: Integrate random tables into traits display (GREEN phase)
5. style: Apply linting fixes (if needed)

---

## Success Criteria Checklist

Verify all criteria are met:

**Code Quality:**
- [✓] TypeScript compiles with no errors
- [✓] ESLint passes with no warnings
- [✓] All tests pass (16 total: 13 unit + 3 integration)
- [✓] No console errors in browser

**Functionality:**
- [✓] Random tables display on Entertainer background
- [✓] All 5 tables render correctly
- [✓] Roll ranges format correctly (1 vs 1-3)
- [✓] Table names and dice types appear
- [✓] Descriptions display when present

**Visual Design:**
- [✓] Tables match design document
- [✓] Dark mode works correctly
- [✓] Responsive on mobile, tablet, desktop
- [✓] Hover states work
- [✓] Spacing looks good

**Edge Cases:**
- [✓] Backgrounds without tables show no errors
- [✓] Empty random_tables array handled gracefully
- [✓] Null descriptions don't break layout

**Process:**
- [✓] TDD workflow followed (RED-GREEN-RED-GREEN)
- [✓] Frequent commits with descriptive messages
- [✓] Manual browser testing completed
- [✓] All changes committed

---

## Implementation Complete!

**Summary:**
- ✅ Created `UiAccordionRandomTablesList.vue` component
- ✅ Integrated into `UiAccordionTraitsList.vue`
- ✅ 16 tests written and passing
- ✅ Manual testing completed
- ✅ Dark mode and responsive verified
- ✅ All commits made with proper messages

**What to do next:**

1. **Verify the feature is working** by visiting: `http://localhost:3000/backgrounds/entertainer`

2. **Test a few more backgrounds** to ensure robustness

3. **Optional enhancements** (if desired):
   - Add table export functionality
   - Add interactive dice rolling
   - Add table search/filter

4. **Move on to next feature** - this feature is complete!

---

## Troubleshooting

**If tests fail:**
- Check that component file path is exactly: `app/components/ui/accordion/UiAccordionRandomTablesList.vue`
- Verify TypeScript interfaces match exactly
- Run `docker compose restart nuxt` to clear any caching issues

**If tables don't appear in browser:**
- Check browser console for errors
- Verify backend API is running: `curl http://localhost:8080/api/v1/backgrounds/entertainer`
- Check that traits have `random_tables` in API response
- Clear browser cache and hard refresh (Ctrl+Shift+R)

**If styling looks wrong:**
- Verify Tailwind classes are spelled correctly
- Check dark mode toggle is working
- Inspect element in DevTools to see applied styles

**If TypeScript errors:**
- Verify all interfaces are defined
- Check that imports are correct
- Run `docker compose exec nuxt npm run typecheck` for detailed errors
