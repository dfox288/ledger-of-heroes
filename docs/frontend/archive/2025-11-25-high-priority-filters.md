# High Priority Filters Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement 8 high-priority missing filters across Items, Monsters, and Classes pages discovered in comprehensive filter audit.

**Architecture:** Follow established patterns from Spells page (gold standard). Use `useMeilisearchFilters` composable for filter logic, `useReferenceData` for dropdown options, standard NuxtUI filter components. All filters follow TDD with tests before implementation.

**Tech Stack:** Vue 3 Composition API, TypeScript, Nuxt 4, NuxtUI 4, Vitest

**Priority Order:**
1. **Items** - Cost & AC filters (2 filters, 50 min)
2. **Monsters** - AC & HP filters (2 filters, 20 min)
3. **Classes** - Hit Die, Spellcasting Ability, Parent Class, Sources (4 filters, 2.5 hours)

**Total Estimated Time:** ~4 hours for 8 filters

---

## Task 1: Items - Cost Range Filter (30 min)

**Goal:** Add cost range filter to Items page using predefined price ranges (Under 1gp, 1-10gp, 10-100gp, etc.)

**Files:**
- Modify: `/Users/dfox/Development/dnd/frontend/app/pages/items/index.vue`
- Test: `/Users/dfox/Development/dnd/frontend/tests/pages/items-cost-filter.test.ts` (create new)

**Reference:** Audit report confirms `cost_cp` field is filterable, 424 items have cost data, API tested and working.

**API Field:** `cost_cp` (integer, copper pieces)

---

### Step 1.1: Write failing test for cost filter state management

**File:** `tests/pages/items-cost-filter.test.ts` (create)

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import ItemsIndexPage from '~/pages/items/index.vue'

describe('Items Cost Filter', () => {
  it('renders cost range filter dropdown', async () => {
    const wrapper = await mountSuspended(ItemsIndexPage)

    // Find cost filter select menu
    const costFilter = wrapper.find('[data-testid="cost-filter"]')
    expect(costFilter.exists()).toBe(true)
  })

  it('has correct cost range options', async () => {
    const wrapper = await mountSuspended(ItemsIndexPage)

    // Verify options exist (will implement in Step 1.3)
    const costFilter = wrapper.find('[data-testid="cost-filter"]')
    expect(costFilter.text()).toContain('All Prices')
  })
})
```

**Run:**
```bash
docker compose exec nuxt npm run test -- tests/pages/items-cost-filter.test.ts
```

**Expected:** FAIL - component doesn't have cost filter yet

---

### Step 1.2: Add cost filter state to Items page

**File:** `app/pages/items/index.vue`

**Location:** Add after `selectedSources` ref (around line 50)

```typescript
// Cost filter
const selectedCostRange = ref<string | null>(null)
const costRangeOptions = [
  { label: 'All Prices', value: null },
  { label: 'Under 1 gp', value: 'under-100' },
  { label: '1-10 gp', value: '100-1000' },
  { label: '10-100 gp', value: '1000-10000' },
  { label: '100-1000 gp', value: '10000-100000' },
  { label: '1000+ gp', value: 'over-100000' }
]
```

---

### Step 1.3: Add cost filter to queryBuilder

**File:** `app/pages/items/index.vue`

**Location:** Inside `queryBuilder` computed property (around line 140)

```typescript
// Add after existing filters, before the return statement
if (selectedCostRange.value) {
  const ranges: Record<string, string> = {
    'under-100': 'cost_cp < 100',
    '100-1000': 'cost_cp >= 100 AND cost_cp <= 1000',
    '1000-10000': 'cost_cp >= 1000 AND cost_cp <= 10000',
    '10000-100000': 'cost_cp >= 10000 AND cost_cp <= 100000',
    'over-100000': 'cost_cp >= 100000'
  }
  meilisearchFilters.push(ranges[selectedCostRange.value])
}
```

---

### Step 1.4: Add cost filter UI component

**File:** `app/pages/items/index.vue`

**Location:** In ADVANCED filters section template (around line 280)

```vue
<!-- Cost Range Filter -->
<div>
  <label class="text-sm font-medium mb-2 block">Cost</label>
  <USelectMenu
    v-model="selectedCostRange"
    :items="costRangeOptions"
    value-key="value"
    placeholder="All Prices"
    size="md"
    class="w-full sm:w-44"
    data-testid="cost-filter"
  />
</div>
```

---

### Step 1.5: Add cost filter to active filter count

**File:** `app/pages/items/index.vue`

**Location:** In `useFilterCount` call (around line 180)

```typescript
const activeFilterCount = useFilterCount(
  // ... existing filters
  selectedCostRange  // Add this line
)
```

---

### Step 1.6: Add cost filter to clearFilters function

**File:** `app/pages/items/index.vue`

**Location:** In `clearFilters` function (around line 195)

```typescript
const clearFilters = () => {
  clearBaseFilters()
  // ... existing clears
  selectedCostRange.value = null  // Add this line
}
```

---

### Step 1.7: Add cost filter chip

**File:** `app/pages/items/index.vue`

**Location:** In active filters chips section (around line 320)

```vue
<!-- Cost Range Chip -->
<UButton
  v-if="selectedCostRange"
  size="xs"
  color="primary"
  variant="soft"
  @click="selectedCostRange = null"
>
  {{ costRangeOptions.find(o => o.value === selectedCostRange)?.label }} ✕
</UButton>
```

---

### Step 1.8: Run tests to verify implementation

**Run:**
```bash
docker compose exec nuxt npm run test -- tests/pages/items-cost-filter.test.ts
```

**Expected:** All tests PASS

---

### Step 1.9: Add API integration test

**File:** `tests/pages/items-cost-filter.test.ts`

**Add:**
```typescript
describe('Items Cost Filter - API Integration', () => {
  it('filters items under 1 gp', async () => {
    const { data } = await apiFetch<PaginatedResponse<Item>>(
      '/items?filter=cost_cp < 100&per_page=5'
    )

    expect(data).toBeDefined()
    expect(data.length).toBeGreaterThan(0)
    data.forEach(item => {
      if (item.cost_cp) {
        expect(item.cost_cp).toBeLessThan(100)
      }
    })
  })

  it('filters items 100-1000 gp', async () => {
    const { data } = await apiFetch<PaginatedResponse<Item>>(
      '/items?filter=cost_cp >= 10000 AND cost_cp <= 100000&per_page=5'
    )

    expect(data).toBeDefined()
    expect(data.length).toBeGreaterThan(0)
    data.forEach(item => {
      expect(item.cost_cp).toBeGreaterThanOrEqual(10000)
      expect(item.cost_cp).toBeLessThanOrEqual(100000)
    })
  })
})
```

---

### Step 1.10: Verify TypeScript compilation

**Run:**
```bash
docker compose exec nuxt npm run typecheck
```

**Expected:** No errors

---

### Step 1.11: Update CHANGELOG.md

**File:** `CHANGELOG.md`

**Add under `### Added`:**
```markdown
- Cost range filter for Items with 5 predefined ranges (Under 1gp to 1000+ gp) (2025-11-25)
```

---

### Step 1.12: Commit cost filter

**Run:**
```bash
git add app/pages/items/index.vue tests/pages/items-cost-filter.test.ts CHANGELOG.md
git commit -m "feat(items): Add cost range filter with 5 price tiers

- Added cost_cp filter with predefined ranges
- Ranges: Under 1gp, 1-10gp, 10-100gp, 100-1000gp, 1000+ gp
- Covers 424 items with cost data
- Added 4 tests (all passing)
- Integrated into ADVANCED filters section

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Items - Armor Class Filter (20 min)

**Goal:** Add AC range filter to Items page using predefined ranges (Light 11-14, Medium 15-16, Heavy 17+)

**Files:**
- Modify: `/Users/dfox/Development/dnd/frontend/app/pages/items/index.vue`
- Modify: `/Users/dfox/Development/dnd/frontend/tests/pages/items-cost-filter.test.ts` (rename to items-filters-advanced.test.ts)

**API Field:** `armor_class` (integer)

---

### Step 2.1: Write failing test for AC filter

**File:** `tests/pages/items-filters-advanced.test.ts` (rename from items-cost-filter.test.ts)

**Add:**
```typescript
describe('Items AC Filter', () => {
  it('renders AC range filter dropdown', async () => {
    const wrapper = await mountSuspended(ItemsIndexPage)

    const acFilter = wrapper.find('[data-testid="ac-filter"]')
    expect(acFilter.exists()).toBe(true)
  })

  it('has correct AC range options', async () => {
    const wrapper = await mountSuspended(ItemsIndexPage)

    const acFilter = wrapper.find('[data-testid="ac-filter"]')
    expect(acFilter.text()).toContain('All AC')
    expect(acFilter.text()).toContain('Light (11-14)')
  })
})
```

**Run:**
```bash
docker compose exec nuxt npm run test -- tests/pages/items-filters-advanced.test.ts
```

**Expected:** FAIL - AC filter not implemented yet

---

### Step 2.2: Add AC filter state

**File:** `app/pages/items/index.vue`

**Location:** Add after `selectedCostRange` ref

```typescript
// AC filter
const selectedACRange = ref<string | null>(null)
const acRangeOptions = [
  { label: 'All AC', value: null },
  { label: 'Light (11-14)', value: '11-14' },
  { label: 'Medium (15-16)', value: '15-16' },
  { label: 'Heavy (17+)', value: '17-21' }
]
```

---

### Step 2.3: Add AC filter to queryBuilder

**File:** `app/pages/items/index.vue`

**Location:** Inside `queryBuilder` computed property, after cost filter

```typescript
if (selectedACRange.value) {
  const ranges: Record<string, string> = {
    '11-14': 'armor_class >= 11 AND armor_class <= 14',
    '15-16': 'armor_class >= 15 AND armor_class <= 16',
    '17-21': 'armor_class >= 17 AND armor_class <= 21'
  }
  meilisearchFilters.push(ranges[selectedACRange.value])
}
```

---

### Step 2.4: Add AC filter UI component

**File:** `app/pages/items/index.vue`

**Location:** After cost filter in ADVANCED section

```vue
<!-- AC Range Filter -->
<div>
  <label class="text-sm font-medium mb-2 block">Armor Class</label>
  <USelectMenu
    v-model="selectedACRange"
    :items="acRangeOptions"
    value-key="value"
    placeholder="All AC"
    size="md"
    class="w-full sm:w-44"
    data-testid="ac-filter"
  />
</div>
```

---

### Step 2.5: Add AC to filter count and clear

**File:** `app/pages/items/index.vue`

**Update `useFilterCount`:**
```typescript
const activeFilterCount = useFilterCount(
  // ... existing
  selectedCostRange,
  selectedACRange  // Add this
)
```

**Update `clearFilters`:**
```typescript
selectedACRange.value = null  // Add this
```

---

### Step 2.6: Add AC filter chip

**File:** `app/pages/items/index.vue`

**Location:** After cost chip

```vue
<!-- AC Range Chip -->
<UButton
  v-if="selectedACRange"
  size="xs"
  color="primary"
  variant="soft"
  @click="selectedACRange = null"
>
  AC: {{ acRangeOptions.find(o => o.value === selectedACRange)?.label }} ✕
</UButton>
```

---

### Step 2.7: Add AC API integration tests

**File:** `tests/pages/items-filters-advanced.test.ts`

**Add:**
```typescript
describe('Items AC Filter - API Integration', () => {
  it('filters light armor (AC 11-14)', async () => {
    const { data } = await apiFetch<PaginatedResponse<Item>>(
      '/items?filter=armor_class >= 11 AND armor_class <= 14&per_page=5'
    )

    expect(data).toBeDefined()
    expect(data.length).toBeGreaterThan(0)
    data.forEach(item => {
      expect(item.armor_class).toBeGreaterThanOrEqual(11)
      expect(item.armor_class).toBeLessThanOrEqual(14)
    })
  })

  it('filters heavy armor (AC 17+)', async () => {
    const { data } = await apiFetch<PaginatedResponse<Item>>(
      '/items?filter=armor_class >= 17 AND armor_class <= 21&per_page=5'
    )

    expect(data).toBeDefined()
    data.forEach(item => {
      expect(item.armor_class).toBeGreaterThanOrEqual(17)
    })
  })
})
```

---

### Step 2.8: Run all items filter tests

**Run:**
```bash
docker compose exec nuxt npm run test -- tests/pages/items-filters-advanced.test.ts
docker compose exec nuxt npm run typecheck
```

**Expected:** All tests PASS, no TypeScript errors

---

### Step 2.9: Update CHANGELOG

**File:** `CHANGELOG.md`

**Add:**
```markdown
- Armor Class range filter for Items (Light 11-14, Medium 15-16, Heavy 17+) (2025-11-25)
```

---

### Step 2.10: Commit AC filter

**Run:**
```bash
git add app/pages/items/index.vue tests/pages/items-filters-advanced.test.ts CHANGELOG.md
git commit -m "feat(items): Add armor class range filter

- Added armor_class filter with 3 predefined ranges
- Ranges: Light (11-14), Medium (15-16), Heavy (17+)
- Covers 595 armor items
- Added 4 tests (all passing)
- Integrated into ADVANCED filters section

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Monsters - Armor Class Range Filter (10 min)

**Goal:** Add AC range filter to Monsters page using range slider

**Files:**
- Modify: `/Users/dfox/Development/dnd/frontend/app/pages/monsters/index.vue`
- Test: `/Users/dfox/Development/dnd/frontend/tests/pages/monsters-filters.test.ts` (modify existing)

**API Field:** `armor_class` (integer, range 10-25)

---

### Step 3.1: Write failing test for monsters AC filter

**File:** `tests/pages/monsters-filters.test.ts`

**Add:**
```typescript
describe('Monsters AC Filter', () => {
  it('filters monsters by AC range', async () => {
    const { data } = await apiFetch<PaginatedResponse<Monster>>(
      '/monsters?filter=armor_class >= 18&per_page=5'
    )

    expect(data).toBeDefined()
    expect(data.length).toBeGreaterThan(0)
    data.forEach(monster => {
      expect(monster.armor_class).toBeGreaterThanOrEqual(18)
    })
  })
})
```

**Run:**
```bash
docker compose exec nuxt npm run test -- tests/pages/monsters-filters.test.ts -t "AC Filter"
```

**Expected:** PASS (API works, UI not implemented yet)

---

### Step 3.2: Add AC filter state

**File:** `app/pages/monsters/index.vue`

**Location:** Add after existing filter refs (around line 50)

```typescript
// AC filter
const selectedACRange = ref<string | null>(null)
const acRangeOptions = [
  { label: 'All AC', value: null },
  { label: 'Low (10-14)', value: '10-14' },
  { label: 'Medium (15-17)', value: '15-17' },
  { label: 'High (18+)', value: '18-25' }
]
```

---

### Step 3.3: Add AC filter to useMeilisearchFilters

**File:** `app/pages/monsters/index.vue`

**Location:** In queryBuilder, after existing filters

```typescript
if (selectedACRange.value) {
  const ranges: Record<string, string> = {
    '10-14': 'armor_class >= 10 AND armor_class <= 14',
    '15-17': 'armor_class >= 15 AND armor_class <= 17',
    '18-25': 'armor_class >= 18 AND armor_class <= 25'
  }
  meilisearchFilters.push(ranges[selectedACRange.value])
}
```

---

### Step 3.4: Add AC filter UI

**File:** `app/pages/monsters/index.vue`

**Location:** In ADVANCED filters section

```vue
<!-- AC Range Filter -->
<div>
  <label class="text-sm font-medium mb-2 block">Armor Class</label>
  <USelectMenu
    v-model="selectedACRange"
    :items="acRangeOptions"
    value-key="value"
    placeholder="All AC"
    size="md"
    class="w-full sm:w-44"
    data-testid="ac-filter"
  />
</div>
```

---

### Step 3.5: Add to filter count, clear, and chip

**File:** `app/pages/monsters/index.vue`

**Update filter count:**
```typescript
const activeFilterCount = useFilterCount(
  // ... existing
  selectedACRange
)
```

**Update clearFilters:**
```typescript
selectedACRange.value = null
```

**Add chip:**
```vue
<UButton
  v-if="selectedACRange"
  size="xs"
  color="info"
  variant="soft"
  @click="selectedACRange = null"
>
  AC: {{ acRangeOptions.find(o => o.value === selectedACRange)?.label }} ✕
</UButton>
```

---

### Step 3.6: Run tests and commit

**Run:**
```bash
docker compose exec nuxt npm run test -- tests/pages/monsters-filters.test.ts
docker compose exec nuxt npm run typecheck
```

**Expected:** All tests PASS

**Update CHANGELOG:**
```markdown
- Armor Class range filter for Monsters (Low 10-14, Medium 15-17, High 18+) (2025-11-25)
```

**Commit:**
```bash
git add app/pages/monsters/index.vue tests/pages/monsters-filters.test.ts CHANGELOG.md
git commit -m "feat(monsters): Add armor class range filter

- Added armor_class filter with 3 ranges
- Ranges: Low (10-14), Medium (15-17), High (18+)
- Covers all 598 monsters
- Added 1 test (passing)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Monsters - Hit Points Range Filter (10 min)

**Goal:** Add HP range filter to Monsters page

**Files:**
- Modify: `/Users/dfox/Development/dnd/frontend/app/pages/monsters/index.vue`
- Modify: `/Users/dfox/Development/dnd/frontend/tests/pages/monsters-filters.test.ts`

**API Field:** `hit_points_average` (integer, range 1-600)

---

### Step 4.1: Write failing test

**File:** `tests/pages/monsters-filters.test.ts`

**Add:**
```typescript
describe('Monsters HP Filter', () => {
  it('filters monsters by HP range', async () => {
    const { data } = await apiFetch<PaginatedResponse<Monster>>(
      '/monsters?filter=hit_points_average >= 200&per_page=5'
    )

    expect(data).toBeDefined()
    expect(data.length).toBeGreaterThan(0)
    data.forEach(monster => {
      expect(monster.hit_points_average).toBeGreaterThanOrEqual(200)
    })
  })
})
```

---

### Step 4.2: Add HP filter state

**File:** `app/pages/monsters/index.vue`

```typescript
// HP filter
const selectedHPRange = ref<string | null>(null)
const hpRangeOptions = [
  { label: 'All HP', value: null },
  { label: 'Low (1-50)', value: '1-50' },
  { label: 'Medium (51-150)', value: '51-150' },
  { label: 'High (151-300)', value: '151-300' },
  { label: 'Very High (301+)', value: '301-600' }
]
```

---

### Step 4.3: Add HP filter to queryBuilder

**File:** `app/pages/monsters/index.vue`

```typescript
if (selectedHPRange.value) {
  const ranges: Record<string, string> = {
    '1-50': 'hit_points_average >= 1 AND hit_points_average <= 50',
    '51-150': 'hit_points_average >= 51 AND hit_points_average <= 150',
    '151-300': 'hit_points_average >= 151 AND hit_points_average <= 300',
    '301-600': 'hit_points_average >= 301 AND hit_points_average <= 600'
  }
  meilisearchFilters.push(ranges[selectedHPRange.value])
}
```

---

### Step 4.4: Add HP filter UI

**File:** `app/pages/monsters/index.vue`

```vue
<!-- HP Range Filter -->
<div>
  <label class="text-sm font-medium mb-2 block">Hit Points</label>
  <USelectMenu
    v-model="selectedHPRange"
    :items="hpRangeOptions"
    value-key="value"
    placeholder="All HP"
    size="md"
    class="w-full sm:w-44"
    data-testid="hp-filter"
  />
</div>
```

---

### Step 4.5: Add to filter count, clear, and chip

**Update filter count, clearFilters, add chip (same pattern as AC)**

---

### Step 4.6: Run tests and commit

**Run tests, update CHANGELOG, commit with descriptive message**

```bash
git commit -m "feat(monsters): Add hit points range filter

- Added hit_points_average filter with 4 ranges
- Ranges: Low (1-50), Medium (51-150), High (151-300), Very High (301+)
- Added 1 test (passing)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Classes - Hit Die Filter (40 min)

**Goal:** Add hit die filter to Classes page (d6, d8, d10, d12 multiselect)

**Files:**
- Modify: `/Users/dfox/Development/dnd/frontend/app/pages/classes/index.vue`
- Test: `/Users/dfox/Development/dnd/frontend/tests/pages/classes-filters-expanded.test.ts` (create new)

**API Field:** `hit_die` (integer: 6, 8, 10, 12)

**Reference:** Audit report confirms 15 filterable fields available

---

### Step 5.1: Write failing test for hit die filter

**File:** `tests/pages/classes-filters-expanded.test.ts` (create)

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { apiFetch } from '~/lib/api-fetch'
import type { PaginatedResponse, CharacterClass } from '~/types/api'
import ClassesIndexPage from '~/pages/classes/index.vue'

describe('Classes Hit Die Filter', () => {
  it('renders hit die multiselect filter', async () => {
    const wrapper = await mountSuspended(ClassesIndexPage)

    const hitDieFilter = wrapper.find('[data-testid="hit-die-filter"]')
    expect(hitDieFilter.exists()).toBe(true)
  })

  it('filters classes by hit die d12', async () => {
    const { data } = await apiFetch<PaginatedResponse<CharacterClass>>(
      '/classes?filter=hit_die IN [12]&per_page=15'
    )

    expect(data).toBeDefined()
    expect(data.length).toBeGreaterThan(0)
    data.forEach(cls => {
      expect(cls.hit_die).toBe(12)
    })
  })

  it('filters classes by multiple hit dice', async () => {
    const { data } = await apiFetch<PaginatedResponse<CharacterClass>>(
      '/classes?filter=hit_die IN [6,8]&per_page=15'
    )

    expect(data).toBeDefined()
    expect(data.length).toBeGreaterThan(0)
    data.forEach(cls => {
      expect([6, 8]).toContain(cls.hit_die)
    })
  })
})
```

**Run:**
```bash
docker compose exec nuxt npm run test -- tests/pages/classes-filters-expanded.test.ts
```

**Expected:** FAIL - filter not implemented

---

### Step 5.2: Add hit die filter state

**File:** `app/pages/classes/index.vue`

**Location:** Add after existing filter refs

```typescript
// Hit Die filter
const selectedHitDice = ref<number[]>([])
const hitDieOptions = [
  { label: 'd6', value: 6 },
  { label: 'd8', value: 8 },
  { label: 'd10', value: 10 },
  { label: 'd12', value: 12 }
]
```

---

### Step 5.3: Add hit die to useMeilisearchFilters

**File:** `app/pages/classes/index.vue`

**Location:** Find the `useMeilisearchFilters` call and add:

```typescript
const { queryParams } = useMeilisearchFilters([
  // ... existing filters
  { ref: selectedHitDice, field: 'hit_die', type: 'in' }
])
```

---

### Step 5.4: Add hit die filter UI component

**File:** `app/pages/classes/index.vue`

**Location:** Add to filters section (create ADVANCED section if doesn't exist)

```vue
<!-- Hit Die Filter -->
<UiFilterMultiSelect
  v-model="selectedHitDice"
  :options="hitDieOptions"
  placeholder="All Hit Dice"
  color="primary"
  data-testid="hit-die-filter"
/>
```

---

### Step 5.5: Add to filter count and clear

**File:** `app/pages/classes/index.vue`

**Update `useFilterCount`:**
```typescript
const activeFilterCount = useFilterCount(
  selectedIsBaseClass,
  selectedIsSpellcaster,
  selectedHitDice  // Add this
)
```

**Update `clearFilters`:**
```typescript
selectedHitDice.value = []
```

---

### Step 5.6: Add hit die filter chips

**File:** `app/pages/classes/index.vue`

**Location:** In active filters section

```vue
<!-- Hit Die Chips -->
<UButton
  v-for="hitDie in selectedHitDice"
  :key="hitDie"
  size="xs"
  color="primary"
  variant="soft"
  @click="selectedHitDice = selectedHitDice.filter(d => d !== hitDie)"
>
  d{{ hitDie }} ✕
</UButton>
```

---

### Step 5.7: Run tests and verify

**Run:**
```bash
docker compose exec nuxt npm run test -- tests/pages/classes-filters-expanded.test.ts
docker compose exec nuxt npm run typecheck
```

**Expected:** All 3 tests PASS, no TypeScript errors

---

### Step 5.8: Update CHANGELOG and commit

**File:** `CHANGELOG.md`

```markdown
- Hit die multiselect filter for Classes (d6, d8, d10, d12) (2025-11-25)
```

**Commit:**
```bash
git add app/pages/classes/index.vue tests/pages/classes-filters-expanded.test.ts CHANGELOG.md
git commit -m "feat(classes): Add hit die multiselect filter

- Added hit_die IN filter with 4 options (d6, d8, d10, d12)
- Covers all 99 classes and subclasses
- Added 3 tests (all passing)
- Integrated into filters section

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Classes - Spellcasting Ability Filter (30 min)

**Goal:** Add spellcasting ability filter (INT, WIS, CHA dropdown)

**Files:**
- Modify: `/Users/dfox/Development/dnd/frontend/app/pages/classes/index.vue`
- Modify: `/Users/dfox/Development/dnd/frontend/tests/pages/classes-filters-expanded.test.ts`

**API Field:** `spellcasting_ability` (string: "INT", "WIS", "CHA")

---

### Step 6.1: Write failing test

**File:** `tests/pages/classes-filters-expanded.test.ts`

**Add:**
```typescript
describe('Classes Spellcasting Ability Filter', () => {
  it('filters classes by spellcasting ability CHA', async () => {
    const { data } = await apiFetch<PaginatedResponse<CharacterClass>>(
      '/classes?filter=spellcasting_ability = "CHA"&per_page=30'
    )

    expect(data).toBeDefined()
    expect(data.length).toBeGreaterThan(0)
    data.forEach(cls => {
      expect(cls.spellcasting_ability).toBe('CHA')
    })
  })
})
```

---

### Step 6.2: Add spellcasting ability state

**File:** `app/pages/classes/index.vue`

```typescript
// Spellcasting Ability filter
const selectedSpellcastingAbility = ref<string | null>(null)
const spellcastingAbilityOptions = [
  { label: 'All Abilities', value: null },
  { label: 'Intelligence', value: 'INT' },
  { label: 'Wisdom', value: 'WIS' },
  { label: 'Charisma', value: 'CHA' }
]
```

---

### Step 6.3: Add to useMeilisearchFilters

**File:** `app/pages/classes/index.vue`

```typescript
const { queryParams } = useMeilisearchFilters([
  // ... existing
  { ref: selectedSpellcastingAbility, field: 'spellcasting_ability' }
])
```

---

### Step 6.4: Add UI component

**File:** `app/pages/classes/index.vue`

```vue
<!-- Spellcasting Ability Filter -->
<div>
  <label class="text-sm font-medium mb-2 block">Spellcasting Ability</label>
  <USelectMenu
    v-model="selectedSpellcastingAbility"
    :items="spellcastingAbilityOptions"
    value-key="value"
    placeholder="All Abilities"
    size="md"
    class="w-full sm:w-44"
    data-testid="spellcasting-ability-filter"
  />
</div>
```

---

### Step 6.5: Add to filter count, clear, and chip

**Follow same pattern as previous filters**

---

### Step 6.6: Run tests and commit

**Run tests, update CHANGELOG, commit**

---

## Task 7: Classes - Parent Class Filter (30 min)

**Goal:** Add parent class dropdown for browsing subclasses by parent (Fighter → 10 subclasses)

**Files:**
- Modify: `/Users/dfox/Development/dnd/frontend/app/pages/classes/index.vue`
- Modify: `/Users/dfox/Development/dnd/frontend/tests/pages/classes-filters-expanded.test.ts`

**API Field:** `parent_class_name` (string: "Fighter", "Wizard", etc.)

---

### Step 7.1: Write failing test

```typescript
describe('Classes Parent Class Filter', () => {
  it('filters subclasses by parent Fighter', async () => {
    const { data } = await apiFetch<PaginatedResponse<CharacterClass>>(
      '/classes?filter=parent_class_name = "Fighter"&per_page=15'
    )

    expect(data).toBeDefined()
    expect(data.length).toBeGreaterThan(0)
    data.forEach(cls => {
      expect(cls.parent_class_name).toBe('Fighter')
    })
  })
})
```

---

### Step 7.2: Fetch parent class options with useReferenceData

**File:** `app/pages/classes/index.vue`

**Add after existing refs:**

```typescript
// Parent Class filter - fetch base classes for dropdown
const { data: baseClasses } = useReferenceData<CharacterClass>('/classes', {
  transform: (data) => data.filter(c => c.is_base_class === true)
})

const parentClassOptions = computed(() => {
  const options = [{ label: 'All Classes', value: null }]
  if (baseClasses.value) {
    baseClasses.value.forEach(cls => {
      options.push({ label: cls.name, value: cls.name })
    })
  }
  return options
})
```

---

### Step 7.3: Add parent class filter state

```typescript
const selectedParentClass = ref<string | null>(null)
```

---

### Step 7.4: Add to useMeilisearchFilters

```typescript
{ ref: selectedParentClass, field: 'parent_class_name' }
```

---

### Step 7.5: Add UI component, filter count, clear, chip

**Follow established patterns**

---

### Step 7.6: Run tests and commit

---

## Task 8: Classes - Source Codes Filter (30 min)

**Goal:** Add source multiselect filter (PHB, XGE, TCE, etc.)

**Files:**
- Modify: `/Users/dfox/Development/dnd/frontend/app/pages/classes/index.vue`
- Modify: `/Users/dfox/Development/dnd/frontend/tests/pages/classes-filters-expanded.test.ts`

**API Field:** `source_codes` (array of strings)

---

### Step 8.1: Write failing test

```typescript
describe('Classes Source Filter', () => {
  it('filters classes by source PHB', async () => {
    const { data } = await apiFetch<PaginatedResponse<CharacterClass>>(
      '/classes?filter=source_codes IN [PHB]&per_page=15'
    )

    expect(data).toBeDefined()
    expect(data.length).toBeGreaterThan(0)
    data.forEach(cls => {
      expect(cls.source_codes).toContain('PHB')
    })
  })
})
```

---

### Step 8.2: Fetch source options with useReferenceData

**File:** `app/pages/classes/index.vue`

```typescript
const { data: sources } = useReferenceData<Source>('/sources')

const sourceOptions = computed(() =>
  sources.value?.map(s => ({ label: s.name, value: s.code })) || []
)
```

---

### Step 8.3: Add source filter state

```typescript
const selectedSources = ref<string[]>([])
```

---

### Step 8.4: Add to useMeilisearchFilters

```typescript
{ ref: selectedSources, field: 'source_codes', type: 'in' }
```

---

### Step 8.5: Add UI component (UiFilterMultiSelect)

```vue
<UiFilterMultiSelect
  v-model="selectedSources"
  :options="sourceOptions"
  placeholder="All Sources"
  color="secondary"
  data-testid="source-filter"
/>
```

---

### Step 8.6: Add to filter count, clear, and chips

**Follow established patterns**

---

### Step 8.7: Run all classes tests

**Run:**
```bash
docker compose exec nuxt npm run test -- tests/pages/classes
docker compose exec nuxt npm run typecheck
```

**Expected:** All tests PASS

---

### Step 8.8: Update CHANGELOG and commit

**File:** `CHANGELOG.md`

```markdown
- Source multiselect filter for Classes (PHB, XGE, TCE, etc.) (2025-11-25)
- Spellcasting ability dropdown for Classes (INT, WIS, CHA) (2025-11-25)
- Parent class filter for browsing subclasses by parent (2025-11-25)
```

**Commit:**
```bash
git add app/pages/classes/index.vue tests/pages/classes-filters-expanded.test.ts CHANGELOG.md
git commit -m "feat(classes): Add 3 major filters (spellcasting ability, parent class, sources)

- Added spellcasting_ability dropdown (INT, WIS, CHA)
- Added parent_class_name dropdown for subclass browsing
- Added source_codes multiselect filter
- Total classes filters: 2 → 6 (200% increase)
- Added 6 tests (all passing)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Final Task: Run Full Test Suite & Create Handover

**Goal:** Verify all implementations, run complete test suite, update documentation

---

### Step 9.1: Run complete test suite

**Run:**
```bash
docker compose exec nuxt npm run test
```

**Expected:** All tests PASS (including new filter tests)

---

### Step 9.2: Run TypeScript checks

**Run:**
```bash
docker compose exec nuxt npm run typecheck
```

**Expected:** No errors

---

### Step 9.3: Verify in browser

**Manual verification:**
1. Visit `http://localhost:3000/items` - Test cost and AC filters
2. Visit `http://localhost:3000/monsters` - Test AC and HP filters
3. Visit `http://localhost:3000/classes` - Test hit die, spellcasting ability, parent class, sources

---

### Step 9.4: Create handover document

**File:** `docs/HANDOVER-2025-11-25-HIGH-PRIORITY-FILTERS.md`

**Content:**
```markdown
# High Priority Filters Implementation - Complete

**Date:** 2025-11-25
**Status:** ✅ COMPLETE

## Summary

Implemented 8 high-priority filters across Items, Monsters, and Classes based on comprehensive filter audit.

### Filters Added

**Items (2 filters):**
- Cost range filter (5 ranges: Under 1gp to 1000+ gp)
- Armor class filter (3 ranges: Light, Medium, Heavy)

**Monsters (2 filters):**
- Armor class filter (3 ranges: Low, Medium, High)
- Hit points filter (4 ranges: Low to Very High)

**Classes (4 filters):**
- Hit die multiselect (d6, d8, d10, d12)
- Spellcasting ability dropdown (INT, WIS, CHA)
- Parent class filter (for subclass browsing)
- Source codes multiselect (PHB, XGE, TCE, etc.)

### Impact

- **Items:** 10 → 12 filters (+20%)
- **Monsters:** 10 → 12 filters (+20%)
- **Classes:** 2 → 6 filters (+200%)
- **Total new filters:** 8
- **Tests added:** 20+

### Files Modified

- `app/pages/items/index.vue`
- `app/pages/monsters/index.vue`
- `app/pages/classes/index.vue`
- `tests/pages/items-filters-advanced.test.ts` (new)
- `tests/pages/classes-filters-expanded.test.ts` (new)
- `tests/pages/monsters-filters.test.ts` (modified)
- `CHANGELOG.md`

### Commits

8 commits total (1 per filter implementation)

### Next Steps

Remaining filters to consider:
- Classes: 9 more filters available (armor/weapon proficiencies, saving throws, etc.)
- Monsters: Ability score ranges (STR, DEX, etc.)
- Items: Strength requirement toggle

See audit reports in `docs/` for complete analysis.
```

---

### Step 9.5: Final commit

**Run:**
```bash
git add docs/HANDOVER-2025-11-25-HIGH-PRIORITY-FILTERS.md
git commit -m "docs: Add handover for high-priority filters implementation

- 8 filters implemented across 3 entities
- Items: +2 filters (cost, AC)
- Monsters: +2 filters (AC, HP)
- Classes: +4 filters (hit die, spellcasting, parent, sources)
- 20+ tests added
- Complete audit-driven implementation

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Plan Complete

**Total Tasks:** 9 (8 filter implementations + 1 final verification)
**Estimated Time:** ~4 hours
**Expected Outcome:** 8 new high-priority filters, 20+ tests, complete documentation

**Test Coverage:**
- Unit tests for each filter
- API integration tests
- TypeScript compilation verified
- Browser testing confirmed

**Quality Gates:**
- TDD followed for all implementations
- Code review after each task (via subagent-driven-development)
- All tests must pass before proceeding
- TypeScript strict mode compliance

---

**Ready for execution with superpowers:subagent-driven-development or superpowers:executing-plans**
