# Monsters Page - Fix Chips + Add Filters

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fix 5 critical chip bugs, then add 6 new high-value filters to Monsters page

**Architecture:** Follow established patterns. Fix existing chips first, then add new filters using `useMeilisearchFilters` composable.

**Tech Stack:** Vue 3 Composition API, TypeScript, Nuxt 4, NuxtUI 4, Vitest

**Total Estimated Time:** 2-3 hours

---

## Part 1: Fix Critical Bugs (5 Missing Chips) - 30 min

### Current Status
These 5 filters WORK correctly but have NO FILTER CHIPS, making them invisible to users:
1. Alignment filter
2. Has Fly filter
3. Has Swim filter
4. Has Burrow filter
5. Has Climb filter

### Task 1: Fix Alignment Filter Chips (10 min)

**File:** `/Users/dfox/Development/dnd/frontend/app/pages/monsters/index.vue`

**Problem:** Alignment multiselect works, but selected values don't show as chips

**Step 1.1: Write failing test**
```typescript
// tests/pages/monsters-filters.test.ts
it('displays alignment filter chips when alignments selected', async () => {
  const wrapper = await mountSuspended(MonstersIndexPage)
  const vm = wrapper.vm as any

  vm.selectedAlignments = ['Lawful Good', 'Chaotic Evil']
  await wrapper.vm.$nextTick()

  expect(wrapper.text()).toContain('Lawful Good')
  expect(wrapper.text()).toContain('Chaotic Evil')
})
```

**Step 1.2: Add alignment chips to template**

**Location:** Around line 320 (in chips section)

```vue
<!-- Alignment Chips -->
<UButton
  v-for="alignment in selectedAlignments"
  :key="alignment"
  size="xs"
  color="info"
  variant="soft"
  @click="selectedAlignments = selectedAlignments.filter(a => a !== alignment)"
>
  {{ alignment }} ✕
</UButton>
```

**Step 1.3: Verify test passes, commit**
```bash
git add app/pages/monsters/index.vue tests/pages/monsters-filters.test.ts
git commit -m "fix(monsters): Add missing alignment filter chips

- Added chips display for selectedAlignments array
- Users can now see and remove individual alignment filters
- Fixes critical UX consistency bug

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 2: Fix Movement Speed Filter Chips (20 min)

**Problem:** Has Fly, Has Swim, Has Burrow, Has Climb toggles work but don't show chips

**Step 2.1: Write failing tests**
```typescript
it('displays fly speed chip when filter active', async () => {
  const wrapper = await mountSuspended(MonstersIndexPage)
  const vm = wrapper.vm as any

  vm.hasFlySpeed = '1'  // Filter active
  await wrapper.vm.$nextTick()

  expect(wrapper.text()).toContain('Has Fly Speed')
})

// Similar tests for swim, burrow, climb
```

**Step 2.2: Add chips for all 4 movement filters**

**Location:** After alignment chips

```vue
<!-- Movement Speed Chips -->
<UButton
  v-if="hasFlySpeed === '1'"
  size="xs"
  color="info"
  variant="soft"
  @click="hasFlySpeed = null"
>
  Has Fly Speed ✕
</UButton>

<UButton
  v-if="hasSwimSpeed === '1'"
  size="xs"
  color="info"
  variant="soft"
  @click="hasSwimSpeed = null"
>
  Has Swim Speed ✕
</UButton>

<UButton
  v-if="hasBurrowSpeed === '1'"
  size="xs"
  color="info"
  variant="soft"
  @click="hasBurrowSpeed = null"
>
  Has Burrow Speed ✕
</UButton>

<UButton
  v-if="hasClimbSpeed === '1'"
  size="xs"
  color="info"
  variant="soft"
  @click="hasClimbSpeed = null"
>
  Has Climb Speed ✕
</UButton>
```

**Step 2.3: Update CHANGELOG, verify tests, commit**
```markdown
### Fixed
- Missing filter chips for Alignment, Has Fly, Has Swim, Has Burrow, Has Climb (2025-11-25)
```

```bash
git commit -m "fix(monsters): Add missing chips for 4 movement speed filters

- Added chips for Has Fly, Has Swim, Has Burrow, Has Climb
- Users can now see active movement filters
- Completes chip consistency across all 12 existing filters

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Part 2: Add New Filters (6 filters) - 2 hours

### Task 3: Add armor_type Filter (20 min)

**Goal:** Multiselect filter for armor types (Natural Armor, Plate, etc.)

**API Field:** `armor_type` (string, 289 items have "natural armor")

**Step 3.1: Write failing test**
```typescript
it('filters monsters by armor type', async () => {
  const { data } = await apiFetch('/monsters?filter=armor_type = "natural armor"&per_page=5')
  expect(data.length).toBeGreaterThan(0)
  data.forEach(m => expect(m.armor_type).toBe('natural armor'))
})
```

**Step 3.2: Discover all armor_type values**
```bash
curl -s "http://localhost:8080/api/v1/monsters?per_page=100" | jq '[.data[].armor_type] | unique'
# Expected: ["natural armor", "plate", null, ...]
```

**Step 3.3: Add state and options**
```typescript
const selectedArmorTypes = ref<string[]>([])
const armorTypeOptions = [
  { label: 'Natural Armor', value: 'natural armor' },
  { label: 'Plate Armor', value: 'plate' },
  // ... other discovered values
]
```

**Step 3.4: Add to useMeilisearchFilters**
```typescript
{ ref: selectedArmorTypes, field: 'armor_type', type: 'in' }
```

**Step 3.5: Add UI component, chip, filter count**
```vue
<UiFilterMultiSelect
  v-model="selectedArmorTypes"
  :options="armorTypeOptions"
  placeholder="All Armor Types"
  color="secondary"
/>
```

**Step 3.6: Test, update CHANGELOG, commit**

---

### Task 4: Add can_hover Filter (15 min)

**Goal:** Boolean toggle for creatures that can hover

**API Field:** `can_hover` (boolean, 3 items)

**Implementation:** Follow same TDD pattern as Task 3, but use `UiFilterToggle` with boolean type

---

### Task 5: Add has_lair_actions Filter (15 min)

**Goal:** Boolean toggle for boss monsters with lair actions

**API Field:** `has_lair_actions` (boolean, 45 items)

**Note:** API returns null in response but filter WORKS (backend serialization issue)

---

### Task 6: Add has_reactions Filter (15 min)

**Goal:** Boolean toggle for creatures with reactions

**API Field:** `has_reactions` (boolean, 34 items)

---

### Task 7: Add is_spellcaster Filter (15 min)

**Goal:** Boolean toggle for spellcasting monsters

**API Field:** `is_spellcaster` (boolean, 129 items)

---

### Task 8: Add has_magic_resistance Filter (15 min)

**Goal:** Boolean toggle for magic-resistant creatures

**API Field:** `has_magic_resistance` (boolean, 85 items)

---

## Final Task: Run Tests & Create Handover

**Step 1: Run full test suite**
```bash
docker compose exec nuxt npm run test -- tests/pages/monsters
```

**Step 2: Verify in browser**
- Visit http://localhost:3000/monsters
- Test all 5 fixed chips
- Test all 6 new filters

**Step 3: Create handover document**
**File:** `docs/HANDOVER-2025-11-25-MONSTERS-COMPLETE.md`

**Content:**
```markdown
# Monsters Filters - Complete Implementation

**Fixed Bugs:** 5 missing chips
**Added Filters:** 6 new filters
**New Total:** 18 filters (from 12, +50%)

## Changes
- Fixed alignment chips
- Fixed 4 movement speed chips
- Added armor_type multiselect
- Added can_hover toggle
- Added has_lair_actions toggle
- Added has_reactions toggle
- Added is_spellcaster toggle
- Added has_magic_resistance toggle

## Impact
- 100% chip consistency
- 50% more filtering options
- Better DM encounter planning
```

**Step 4: Final commit**
```bash
git add docs/HANDOVER-2025-11-25-MONSTERS-COMPLETE.md
git commit -m "docs: Add handover for monsters filters completion

- Fixed 5 critical chip bugs
- Added 6 new filters
- Monsters page now 100% feature complete

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Success Criteria

- ✅ All 5 chip bugs fixed
- ✅ All 6 new filters implemented
- ✅ All tests passing
- ✅ TypeScript compiles
- ✅ CHANGELOG updated
- ✅ Handover document created
- ✅ Browser verification complete

**Total Commits:** 8-9 (1 per task + final handover)
