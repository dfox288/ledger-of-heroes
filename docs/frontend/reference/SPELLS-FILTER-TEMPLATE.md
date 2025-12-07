# Spells Filter Template - Gold Standard for List Pages

**Created:** 2025-11-25
**Status:** ✅ Production Template
**Reference:** `/app/pages/spells/index.vue`

---

## Overview

This document provides a step-by-step template for refactoring list page filters to match the **spells filter pattern**, which is now the gold standard for all 7 entity list pages.

**Why this template?**
- ✅ Consistent UX across all entity types
- ✅ Simpler multiselect pattern (no complex toggles/sliders)
- ✅ Better space efficiency (Clear filters on same row)
- ✅ Proven with TDD (all tests passing)
- ✅ Production-ready and browser-verified

---

## Pages to Refactor

| Page | Priority | Complexity | Estimated Effort |
|------|----------|------------|------------------|
| `/items` | 🔴 High | Medium | 2-3 hours |
| `/monsters` | 🔴 High | Medium | 2-3 hours |
| `/races` | 🟡 Medium | Low | 1-2 hours |
| `/classes` | 🟡 Medium | Low | 1-2 hours |
| `/backgrounds` | 🟢 Low | Low | 1 hour |
| `/feats` | 🟢 Low | Low | 1 hour |

**Total Estimated Effort:** 9-13 hours

---

## Filter Component Reference

### When to Use Each Component

| Component | Use Case | Example |
|-----------|----------|---------|
| `UiFilterMultiSelect` | Select multiple discrete values | Levels: [Cantrip, 3rd, 9th] |
| `USelectMenu` | Select single value from list | School: Evocation |
| `UiFilterToggle` | Binary or ternary choice | Concentration: All/Yes/No |

### Component Patterns

#### 1. UiFilterMultiSelect (Multiselect)

```vue
<UiFilterMultiSelect
  v-model="selectedLevels"
  data-testid="level-filter-multiselect"
  :options="levelOptions"
  placeholder="All Levels"
  color="primary"
  class="w-full sm:w-48"
/>
```

**Props:**
- `v-model`: String array (`string[]`)
- `options`: `Array<{ label: string, value: string }>`
- `placeholder`: Text when nothing selected
- `color`: Semantic color (primary, spell, item, etc.)
- `class`: Width styling

**State:**
```typescript
const selectedLevels = ref<string[]>([])
```

**Options:**
```typescript
const levelOptions = [
  { label: 'Cantrip', value: '0' },
  { label: '1st Level', value: '1' },
  // ... (string values!)
]
```

**Filter Logic:**
```typescript
{
  ref: selectedLevels,
  field: 'level',
  type: 'in',
  transform: (levels) => levels.map(Number) // Convert to numbers for API
}
```

---

#### 2. USelectMenu (Single Select)

```vue
<USelectMenu
  v-model="selectedSchool"
  :items="schoolOptions"
  value-key="value"
  placeholder="All Schools"
  size="md"
  class="w-full sm:w-48"
/>
```

**Props:**
- `v-model`: Single value (`number | string | null`)
- `items`: `Array<{ label: string, value: T }>`
- `value-key`: Property name for value
- `placeholder`: Text when nothing selected
- `size`: md (default)
- `class`: Width styling

**State:**
```typescript
const selectedSchool = ref<number | null>(null)
```

**Options:**
```typescript
const schoolOptions = computed(() => [
  { label: 'All Schools', value: null },
  ...spellSchools.value.map(s => ({ label: s.name, value: s.id }))
])
```

**Filter Logic:**
```typescript
{
  ref: selectedSchool,
  field: 'school_code',
  transform: (id) => spellSchools.value?.find(s => s.id === id)?.code || null
}
```

---

#### 3. UiFilterToggle (Binary/Ternary)

```vue
<UiFilterToggle
  v-model="concentrationFilter"
  label="Concentration"
  color="primary"
  :options="[
    { value: null, label: 'All' },
    { value: '1', label: 'Yes' },
    { value: '0', label: 'No' }
  ]"
/>
```

**Props:**
- `v-model`: String (`'0'` | `'1'` | `null`)
- `label`: Filter name
- `color`: Semantic color
- `options`: Array of `{ value, label }`

**State:**
```typescript
const concentrationFilter = ref<string | null>(null)
```

**Filter Logic:**
```typescript
{
  ref: concentrationFilter,
  field: 'concentration',
  type: 'boolean' // Converts '1'→true, '0'→false
}
```

---

## Complete Refactoring Checklist

### Phase 1: Analysis (15-30 min)

- [ ] Read current filter implementation
- [ ] Identify all filter types (multiselect, single-select, toggle)
- [ ] Check if API supports needed filter fields
- [ ] Review existing tests
- [ ] Check reference data endpoints (`/spell-schools`, `/damage-types`, etc.)

### Phase 2: Write Tests First (30-60 min)

**Follow TDD - Write tests BEFORE implementation!**

```typescript
// tests/pages/items-level-filter.test.ts (example)
describe('Items Page - Level Filtering', () => {
  describe('UI components', () => {
    it('displays level filter multiselect', async () => {
      const wrapper = await mountSuspended(ItemsPage)
      const component = wrapper.vm as any
      component.filtersOpen = true
      await wrapper.vm.$nextTick()

      const multiselect = wrapper.find('[data-testid="level-filter-multiselect"]')
      expect(multiselect.exists()).toBe(true)
    })

    it('does not display old range/exact toggle', async () => {
      const wrapper = await mountSuspended(ItemsPage)
      const component = wrapper.vm as any
      component.filtersOpen = true
      await wrapper.vm.$nextTick()

      const toggle = wrapper.find('[data-testid="level-filter-mode-toggle"]')
      expect(toggle.exists()).toBe(false)
    })
  })

  describe('multiselect behavior', () => {
    it('allows selecting multiple levels', async () => {
      const wrapper = await mountSuspended(ItemsPage)
      const component = wrapper.vm as any

      component.selectedLevels = ['0', '1', '2']
      await wrapper.vm.$nextTick()

      expect(component.selectedLevels).toEqual(['0', '1', '2'])
    })

    it('initializes as empty array', async () => {
      const wrapper = await mountSuspended(ItemsPage)
      const component = wrapper.vm as any

      expect(Array.isArray(component.selectedLevels)).toBe(true)
      expect(component.selectedLevels.length).toBe(0)
    })
  })

  describe('filter chip display', () => {
    it('shows chip with selected level labels', async () => {
      const wrapper = await mountSuspended(ItemsPage)
      const component = wrapper.vm as any
      component.selectedLevels = ['0', '1', '2']
      await wrapper.vm.$nextTick()

      const chip = wrapper.find('[data-testid="level-filter-chip"]')
      expect(chip.exists()).toBe(true)
      expect(chip.text()).toContain('Levels')
      expect(chip.text()).toContain('Common')
      expect(chip.text()).toContain('Uncommon')
      expect(chip.text()).toContain('Rare')
    })

    it('clicking chip clears level filter', async () => {
      const wrapper = await mountSuspended(ItemsPage)
      const component = wrapper.vm as any
      component.selectedLevels = ['0', '1']
      await wrapper.vm.$nextTick()

      const chip = wrapper.find('[data-testid="level-filter-chip"]')
      await chip.trigger('click')

      expect(component.selectedLevels).toEqual([])
    })

    it('does not show chip when no levels selected', async () => {
      const wrapper = await mountSuspended(ItemsPage)
      const component = wrapper.vm as any
      component.selectedLevels = []
      await wrapper.vm.$nextTick()

      const chip = wrapper.find('[data-testid="level-filter-chip"]')
      expect(chip.exists()).toBe(false)
    })
  })
})
```

**Run tests - they should FAIL (RED phase):**
```bash
docker compose exec nuxt npm run test -- tests/pages/items-level-filter.test.ts
```

---

### Phase 3: Update State (10-15 min)

#### Remove Old State

```typescript
// ❌ DELETE these
const levelFilterMode = ref<'exact' | 'range'>('exact')
const selectedLevel = ref<number | null>(null)
const minLevel = ref<number | null>(null)
const maxLevel = ref<number | null>(null)
const sliderRange = ref<[number, number]>([0, 9])
```

#### Add New State

```typescript
// ✅ ADD this
const selectedLevels = ref<string[]>(
  route.query.level
    ? (Array.isArray(route.query.level)
        ? route.query.level.map(String)
        : [String(route.query.level)])
    : []
)
```

---

### Phase 4: Update Options (5-10 min)

```typescript
// Remove "All Levels" option (not needed for multiselect)
const levelOptions = [
  { label: 'Common', value: '0' },
  { label: 'Uncommon', value: '1' },
  { label: 'Rare', value: '2' },
  { label: 'Very Rare', value: '3' },
  { label: 'Legendary', value: '4' },
  { label: 'Artifact', value: '5' }
]
```

**Note:** Use string values for UiFilterMultiSelect compatibility!

---

### Phase 5: Update Filter Logic (10-15 min)

#### Remove Old Filter Logic

```typescript
// ❌ DELETE this conditional
const filterParams = useMeilisearchFilters([
  ...(levelFilterMode.value === 'range'
    ? [{ field: 'level', type: 'range', min: minLevel, max: maxLevel }]
    : [{ ref: selectedLevel, field: 'level' }]
  )
])
```

#### Add New Filter Logic

```typescript
// ✅ REPLACE with this
const filterParams = useMeilisearchFilters([
  {
    ref: selectedLevels,
    field: 'rarity', // or 'level' or 'cr' depending on entity
    type: 'in',
    transform: (levels) => levels.map(Number) // Convert strings to numbers
  },
  // ... other filters
])
```

---

### Phase 6: Update Chip Display (15-20 min)

#### Remove Old Chip Logic

```typescript
// ❌ DELETE this complex logic
const getLevelFilterText = computed(() => {
  if (levelFilterMode.value === 'exact' && selectedLevel.value !== null) {
    return `Level ${selectedLevel.value === 0 ? 'Common' : selectedLevel.value}`
  }
  if (levelFilterMode.value === 'range') {
    if (minLevel.value !== null && maxLevel.value !== null) {
      return `Levels ${minLevel.value}-${maxLevel.value}`
    }
    if (minLevel.value !== null) {
      return `Level ${minLevel.value}+`
    }
    if (maxLevel.value !== null) {
      return `Level ${maxLevel.value} or lower`
    }
  }
  return null
})

const clearLevelFilter = () => {
  selectedLevel.value = null
  minLevel.value = null
  maxLevel.value = null
}
```

#### Add New Chip Logic

```typescript
// ✅ REPLACE with this simpler logic

// Helper to get level label for display
const getLevelLabel = (levelStr: string): string => {
  const level = Number(levelStr)
  // Customize based on entity type
  const labels = ['Common', 'Uncommon', 'Rare', 'Very Rare', 'Legendary', 'Artifact']
  return labels[level] || `Level ${level}`
}

// Get level filter display text for chips
const getLevelFilterText = computed(() => {
  if (selectedLevels.value.length === 0) return null

  const labels = selectedLevels.value
    .sort((a, b) => Number(a) - Number(b))
    .map(level => getLevelLabel(level))

  const prefix = selectedLevels.value.length === 1 ? 'Rarity' : 'Rarities'
  return `${prefix}: ${labels.join(', ')}`
})

const clearLevelFilter = () => {
  selectedLevels.value = []
}
```

---

### Phase 7: Update UI Template (15-20 min)

#### Remove Old UI

```vue
<!-- ❌ DELETE this mode toggle + conditional UI -->
<UiFilterToggle
  :model-value="levelFilterMode"
  label="Level Filter"
  :options="[
    { value: 'exact', label: 'Exact' },
    { value: 'range', label: 'Range' }
  ]"
  @update:model-value="..."
/>

<USelectMenu
  v-if="levelFilterMode === 'exact'"
  v-model="selectedLevel"
  ...
/>

<USlider
  v-else
  v-model="sliderRange"
  ...
/>
```

#### Add New UI

```vue
<!-- ✅ REPLACE with single multiselect (NO LABEL!) -->
<UiFilterMultiSelect
  v-model="selectedLevels"
  data-testid="level-filter-multiselect"
  :options="levelOptions"
  placeholder="All Rarities"
  color="item"
  class="w-full sm:w-48"
/>
```

---

### Phase 8: Update Layout (10-15 min)

#### Remove Actions Template

```vue
<!-- ❌ DELETE this -->
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
```

#### Update Active Filters Section

```vue
<!-- ✅ REPLACE with this layout -->

<!-- Empty actions template -->
<template #actions />

<!-- Active Filter Chips Row -->
<div
  v-if="hasActiveFilters"
  class="flex flex-wrap items-center justify-between gap-2 pt-2"
>
  <!-- Left: Label + Chips -->
  <div class="flex flex-wrap items-center gap-2">
    <span
      v-if="activeFilterCount > 0 || searchQuery"
      class="text-sm font-medium text-gray-600 dark:text-gray-400"
    >
      Active filters:
    </span>

    <!-- All filter chips here -->
    <UButton
      v-if="getLevelFilterText"
      data-testid="level-filter-chip"
      size="xs"
      color="primary"
      variant="soft"
      @click="clearLevelFilter"
    >
      {{ getLevelFilterText }} ✕
    </UButton>

    <!-- Other chips... -->
  </div>

  <!-- Right: Clear Filters Button -->
  <UButton
    v-if="activeFilterCount > 0 || searchQuery"
    color="neutral"
    variant="soft"
    size="sm"
    @click="clearFilters"
  >
    Clear filters
  </UButton>
</div>
```

---

### Phase 9: Update clearFilters (5 min)

```typescript
const clearFilters = () => {
  clearBaseFilters()
  selectedLevels.value = [] // ✅ Clear array, not set to null
  // ... clear other filters
}
```

---

### Phase 10: Update useFilterCount (5 min)

```typescript
// Remove old level filter refs
const activeFilterCount = useFilterCount(
  selectedLevel,     // ❌ REMOVE
  minLevel,          // ❌ REMOVE
  maxLevel,          // ❌ REMOVE
  selectedSchool,
  // ...
)

// Add new multiselect ref
const activeFilterCount = useFilterCount(
  selectedLevels,    // ✅ ADD (arrays auto-counted)
  selectedSchool,
  // ...
)
```

---

### Phase 11: Run Tests (5 min)

```bash
# Run specific test file
docker compose exec nuxt npm run test -- tests/pages/items-level-filter.test.ts

# All tests should PASS (GREEN phase)
```

**Expected Result:** ✅ All 10 tests passing

---

### Phase 12: Browser Verification (10-15 min)

**Manual Checklist:**

1. Navigate to page: `http://localhost:3000/items`
2. Click "Filters" to open filter panel
3. Verify multiselect appears (no toggle, no slider)
4. Select multiple levels (e.g., Common + Rare + Legendary)
5. Verify filter chip shows "Rarities: Common, Rare, Legendary"
6. Click chip ✕ to clear
7. Verify "Active filters:" label (not "Active:")
8. Verify "Clear filters" button on same row, right-aligned
9. Test in dark mode
10. Test on mobile (375px, 768px)

---

### Phase 13: Update CHANGELOG (5 min)

```markdown
### Changed

- **Items Level Filter - Multiselect UX** (2025-11-25) - Replaced exact/range toggle with simpler multiselect
  - Removed complex exact/range mode toggle and slider UI
  - Replaced with single UiFilterMultiSelect component
  - Users can now select multiple discrete rarities (e.g., Common + Rare + Legendary)
  - Filter chip shows "Rarity: Common" or "Rarities: Common, Rare, Legendary"
  - Uses `rarity IN [0,2,4]` Meilisearch filter
  - 10 comprehensive tests (all passing with TDD workflow)
  - Improved layout: removed label, renamed "Active:", moved "Clear filters" button
```

---

### Phase 14: Commit (5 min)

```bash
git add -A && git commit -m "$(cat <<'EOF'
refactor: Replace items rarity filter with multiselect

Replace complex exact/range toggle + slider UI with simpler multiselect.
Follows spells filter template pattern.

Changes:
- Remove UiFilterToggle for exact/range mode switching
- Replace with single UiFilterMultiSelect
- Update filter logic to use 'type: in' with string→number conversion
- Simplify chip display: "Rarity: Common" or "Rarities: Common, Rare"
- Remove label, rename "Active:" to "Active filters"
- Move "Clear filters" button to same row (right-aligned)
- Update tests: 10 new tests (all passing)

All tests passing. Page loads successfully (HTTP 200).

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Entity-Specific Customization

### Items (`/items`)

**Filters to refactor:**
- ✅ Rarity (multiselect: Common, Uncommon, Rare, Very Rare, Legendary, Artifact)
- ✅ Item Type (multiselect: Weapon, Armor, Potion, Ring, etc.)
- ⚠️ Requires Attunement (toggle: All/Yes/No)

**API Fields:**
- `rarity` (number 0-5)
- `item_type_slug` (string)
- `requires_attunement` (boolean)

---

### Monsters (`/monsters`)

**Filters to refactor:**
- ✅ Challenge Rating (multiselect: 0, 1/8, 1/4, 1/2, 1-30)
- ✅ Type (multiselect: Beast, Dragon, Humanoid, etc.)
- ✅ Size (multiselect: Tiny, Small, Medium, Large, Huge, Gargantuan)
- ⚠️ Legendary (toggle: All/Yes/No)

**API Fields:**
- `challenge_rating` (number)
- `type_slug` (string)
- `size_code` (string)
- `is_legendary` (boolean)

**Special Note:** Challenge Rating has fractional values (1/8, 1/4, 1/2). Convert to decimals:
```typescript
const crOptions = [
  { label: '0', value: '0' },
  { label: '1/8', value: '0.125' },
  { label: '1/4', value: '0.25' },
  { label: '1/2', value: '0.5' },
  { label: '1', value: '1' },
  // ... up to 30
]
```

---

### Races (`/races`)

**Filters to refactor:**
- ✅ Size (multiselect: Small, Medium, Large)
- ⚠️ Has Subraces (toggle: All/Yes/No)

**API Fields:**
- `size_code` (string)
- `has_subraces` (boolean)

---

### Classes (`/classes`)

**Filters to refactor:**
- ✅ Primary Ability (multiselect: STR, DEX, CON, INT, WIS, CHA)
- ⚠️ Is Base Class (toggle: All/Yes/No)

**API Fields:**
- `primary_ability_code` (string)
- `is_base_class` (boolean)

---

### Backgrounds (`/backgrounds`)

**Filters to refactor:**
- ✅ Source (multiselect: PHB, XGE, TCE, etc.)

**API Fields:**
- `source_code` (string)

---

### Feats (`/feats`)

**Filters to refactor:**
- ✅ Source (multiselect: PHB, XGE, TCE, etc.)
- ✅ Prerequisite Type (multiselect: Ability Score, Proficiency, Race, etc.)

**API Fields:**
- `source_code` (string)
- `prerequisite_type` (string)

---

## Common Pitfalls & Solutions

### Pitfall 1: Using Number Values in Options

**Problem:**
```typescript
// ❌ WRONG
const levelOptions = [
  { label: 'Common', value: 0 }, // TypeScript error!
]
```

**Solution:**
```typescript
// ✅ CORRECT
const levelOptions = [
  { label: 'Common', value: '0' }, // String value
]
```

**Why:** `UiFilterMultiSelect` expects string values for consistency.

---

### Pitfall 2: Forgetting Transform Function

**Problem:**
```typescript
// ❌ WRONG - sends strings to API
{ ref: selectedLevels, field: 'level', type: 'in' }
```

**API receives:** `level IN ["0", "3", "9"]` (strings!)

**Solution:**
```typescript
// ✅ CORRECT - converts to numbers
{
  ref: selectedLevels,
  field: 'level',
  type: 'in',
  transform: (levels) => levels.map(Number)
}
```

**API receives:** `level IN [0, 3, 9]` (numbers!)

---

### Pitfall 3: Adding Labels to Multiselect

**Problem:**
```vue
<!-- ❌ WRONG - breaks layout consistency -->
<UiFilterMultiSelect
  v-model="selectedLevels"
  label="Item Rarity"
  ...
/>
```

**Solution:**
```vue
<!-- ✅ CORRECT - no label, just placeholder -->
<UiFilterMultiSelect
  v-model="selectedLevels"
  placeholder="All Rarities"
  ...
/>
```

**Why:** School and Class dropdowns don't have labels, so multiselects shouldn't either.

---

### Pitfall 4: Forgetting to Update clearFilters

**Problem:**
```typescript
// ❌ WRONG - old refs still referenced
const clearFilters = () => {
  clearBaseFilters()
  selectedLevel.value = null  // This ref no longer exists!
  minLevel.value = null       // This ref no longer exists!
}
```

**Solution:**
```typescript
// ✅ CORRECT - clear new array ref
const clearFilters = () => {
  clearBaseFilters()
  selectedLevels.value = []
  // ... clear other filters
}
```

---

### Pitfall 5: Not Using TDD

**Problem:** Writing implementation before tests (or skipping tests entirely)

**Solution:** Always follow RED → GREEN → REFACTOR:

1. **RED:** Write failing tests first
2. **GREEN:** Write minimum code to pass tests
3. **REFACTOR:** Clean up while keeping tests green

**Why:** TDD ensures you don't break existing functionality and provides regression safety.

---

## Testing Patterns

### Test File Template

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import EntityPage from '~/pages/entity-name/index.vue'

describe('Entity Page - Level Filtering', () => {
  describe('UI components', () => {
    it('displays level filter multiselect', async () => {
      const wrapper = await mountSuspended(EntityPage)
      const component = wrapper.vm as any
      component.filtersOpen = true
      await wrapper.vm.$nextTick()

      const multiselect = wrapper.find('[data-testid="level-filter-multiselect"]')
      expect(multiselect.exists()).toBe(true)
    })

    it('does not display old toggle/slider', async () => {
      const wrapper = await mountSuspended(EntityPage)
      const component = wrapper.vm as any
      component.filtersOpen = true
      await wrapper.vm.$nextTick()

      const toggle = wrapper.find('[data-testid="level-filter-mode-toggle"]')
      const slider = wrapper.find('[data-testid="level-range-slider"]')
      expect(toggle.exists()).toBe(false)
      expect(slider.exists()).toBe(false)
    })
  })

  describe('multiselect behavior', () => {
    it('allows selecting multiple levels', async () => {
      const wrapper = await mountSuspended(EntityPage)
      const component = wrapper.vm as any

      component.selectedLevels = ['0', '2', '4']
      await wrapper.vm.$nextTick()

      expect(component.selectedLevels).toEqual(['0', '2', '4'])
    })

    it('initializes as empty array', async () => {
      const wrapper = await mountSuspended(EntityPage)
      const component = wrapper.vm as any

      expect(Array.isArray(component.selectedLevels)).toBe(true)
      expect(component.selectedLevels.length).toBe(0)
    })
  })

  describe('filter chip display', () => {
    it('shows chip with selected level labels', async () => {
      const wrapper = await mountSuspended(EntityPage)
      const component = wrapper.vm as any
      component.selectedLevels = ['0', '2']
      await wrapper.vm.$nextTick()

      const chip = wrapper.find('[data-testid="level-filter-chip"]')
      expect(chip.exists()).toBe(true)
      expect(chip.text()).toContain('Rarities')
      expect(chip.text()).toContain('Common')
      expect(chip.text()).toContain('Rare')
    })

    it('shows single level without plural', async () => {
      const wrapper = await mountSuspended(EntityPage)
      const component = wrapper.vm as any
      component.selectedLevels = ['2']
      await wrapper.vm.$nextTick()

      const chip = wrapper.find('[data-testid="level-filter-chip"]')
      expect(chip.exists()).toBe(true)
      expect(chip.text()).toContain('Rarity')
      expect(chip.text()).toContain('Rare')
    })

    it('clicking chip clears level filter', async () => {
      const wrapper = await mountSuspended(EntityPage)
      const component = wrapper.vm as any
      component.selectedLevels = ['0', '2']
      await wrapper.vm.$nextTick()

      const chip = wrapper.find('[data-testid="level-filter-chip"]')
      await chip.trigger('click')

      expect(component.selectedLevels).toEqual([])
    })

    it('does not show chip when no levels selected', async () => {
      const wrapper = await mountSuspended(EntityPage)
      const component = wrapper.vm as any
      component.selectedLevels = []
      await wrapper.vm.$nextTick()

      const chip = wrapper.find('[data-testid="level-filter-chip"]')
      expect(chip.exists()).toBe(false)
    })
  })
})
```

---

## Quick Reference

### Files to Modify

1. **Page Component:** `app/pages/{entity}/index.vue`
2. **Test File:** `tests/pages/{entity}-level-filter.test.ts` (create new)
3. **Old Test File:** `tests/pages/{entity}-level-range.test.ts` (delete if exists)
4. **CHANGELOG:** `CHANGELOG.md` (add entry)

### Commands to Run

```bash
# 1. Run tests (should FAIL initially)
docker compose exec nuxt npm run test -- tests/pages/{entity}-level-filter.test.ts

# 2. After implementation (should PASS)
docker compose exec nuxt npm run test -- tests/pages/{entity}-level-filter.test.ts

# 3. Run full test suite
docker compose exec nuxt npm run test

# 4. Type check
docker compose exec nuxt npm run typecheck

# 5. Browser verification
curl -s http://localhost:3000/{entity} -o /dev/null -w "HTTP Status: %{http_code}\n"
```

---

## Success Criteria

A refactor is complete when:

- ✅ All new tests pass (10+ tests)
- ✅ Full test suite passes (1010+ tests)
- ✅ TypeScript compiles with no errors
- ✅ Page loads in browser (HTTP 200)
- ✅ Multiselect UI works in light/dark mode
- ✅ Mobile responsive (375px, 768px, 1440px)
- ✅ Filter chips display correctly
- ✅ "Clear filters" button on same row
- ✅ "Active filters:" label (not "Active:")
- ✅ No label on multiselect component
- ✅ CHANGELOG.md updated
- ✅ Changes committed with descriptive message

---

## Resources

### Reference Files
- **Gold Standard:** `/app/pages/spells/index.vue`
- **Test Template:** `/tests/pages/spells-level-filter.test.ts`
- **Component:** `/app/components/ui/filter/UiFilterMultiSelect.vue`
- **Composables:** `/app/composables/useMeilisearchFilters.ts`, `/app/composables/useFilterCount.ts`

### Documentation
- **Handover:** `/docs/HANDOVER-2025-11-25-LEVEL-FILTER-REFACTOR-COMPLETE.md`
- **This Template:** `/docs/SPELLS-FILTER-TEMPLATE.md`
- **UI Guide:** `/docs/UI-FILTER-LAYOUT-GUIDE.md`

---

## Estimated Time Breakdown

| Phase | Time | Cumulative |
|-------|------|------------|
| 1. Analysis | 15-30 min | 0.25-0.5h |
| 2. Write Tests | 30-60 min | 0.75-1.5h |
| 3. Update State | 10-15 min | 1.0-1.75h |
| 4. Update Options | 5-10 min | 1.1-1.9h |
| 5. Update Filter Logic | 10-15 min | 1.25-2.15h |
| 6. Update Chip Display | 15-20 min | 1.5-2.5h |
| 7. Update UI Template | 15-20 min | 1.75-2.83h |
| 8. Update Layout | 10-15 min | 2.0-3.1h |
| 9. Update clearFilters | 5 min | 2.08-3.2h |
| 10. Update useFilterCount | 5 min | 2.17-3.3h |
| 11. Run Tests | 5 min | 2.25-3.4h |
| 12. Browser Verification | 10-15 min | 2.42-3.65h |
| 13. Update CHANGELOG | 5 min | 2.5-3.75h |
| 14. Commit | 5 min | **2.6-3.9h** |

**Total per page:** 2.5-4 hours (depending on complexity)

---

## Next Steps

1. **Pick first page to refactor:** Start with `/items` (highest priority)
2. **Read this template thoroughly:** Don't skip steps!
3. **Follow TDD strictly:** Write tests first (RED → GREEN → REFACTOR)
4. **Commit after each page:** Don't batch multiple pages
5. **Update handover:** Document completion status

**Good luck!** 🚀 The spells page proves this pattern works beautifully.

---

**Last Updated:** 2025-11-25
**Template Version:** 1.0
**Status:** ✅ Production Ready
