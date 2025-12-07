# Spell List Generator Refactor - Phase 1 Complete

**Date:** 2025-11-25
**Status:** ⚠️ **PHASE 1 COMPLETE - PHASE 2 REQUIRED**
**Commit:** `cf2dab4` - refactor: Separate cantrip and leveled spell tracking (TDD Phase 1)

---

## 🎯 Project Goal

Refactor spell list generator to follow **proper D&D 5e spell selection rules**:
- Cantrips tracked separately (don't count against prepared/known limit)
- Leveled spells (1st-9th) tracked separately with independent limit
- Any mix of spell levels allowed within leveled spell limit

## ✅ Phase 1 Complete: Composable Logic (TDD)

### What Was Completed

**TDD Methodology (RED-GREEN-REFACTOR):**
1. ✅ **RED:** Wrote 3 failing tests describing correct D&D 5e behavior
2. ✅ **Verified RED:** Tests failed correctly (missing properties)
3. ✅ **GREEN:** Implemented minimal code to pass tests
4. ✅ **Verified GREEN:** All 11 tests passing (3 new + 8 updated)

**Composable Changes:**

```typescript
// NEW: Separate tracking
selectedCantrips: Ref<Set<number>>        // Cantrips only (level 0)
selectedLeveledSpells: Ref<Set<number>>   // Leveled spells (1st-9th)

// NEW: Separate limits
maxCantrips: ComputedRef<number>          // From cantrips_known
maxLeveledSpells: ComputedRef<number>     // From class rules (prepared/known)

// NEW: Separate counts
cantripCount: ComputedRef<number>         // selectedCantrips.size
leveledSpellCount: ComputedRef<number>    // selectedLeveledSpells.size

// UPDATED: Now requires spell level
toggleSpell(spellId: number, spellLevel: number): boolean
  // spellLevel = 0 for cantrips
  // spellLevel = 1-9 for leveled spells

// DEPRECATED (but still work for backwards compat)
selectedSpells: Ref<Set<number>>          // Use selectedCantrips + selectedLeveledSpells
maxSpells: ComputedRef<number>            // Use maxLeveledSpells
selectionCount: ComputedRef<number>       // Use cantripCount + leveledSpellCount
```

**Test Coverage:**
- ✅ `tracks cantrips separately from leveled spells`
- ✅ `enforces separate limits for cantrips and leveled spells`
- ✅ `allows any mix of spell levels within leveled spell limit`
- ✅ All 11 tests passing

**Files Changed:**
- `app/composables/useSpellListGenerator.ts` - Core logic
- `tests/composables/useSpellListGenerator.test.ts` - 3 new + 8 updated tests

---

## ⏳ Phase 2 TODO: UI Updates

### Critical Issues (MUST FIX)

**🔴 BREAKING:** The page still calls `toggleSpell(spell.id)` without spell level!

**Current (BROKEN):**
```vue
<UCheckbox
  @update:model-value="() => {
    const success = toggleSpell(spell.id)  // ❌ Missing spell.level!
    // ...
  }"
/>
```

**Required (CORRECT):**
```vue
<UCheckbox
  @update:model-value="() => {
    const success = toggleSpell(spell.id, spell.level)  // ✅ Pass spell.level
    // ...
  }"
/>
```

### Required Changes

#### 1. Update Page Template (`app/pages/spells/list-generator.vue`)

**Line ~336-342:** Update checkbox handler
```vue
<!-- BEFORE -->
<UCheckbox
  @update:model-value="() => {
    const success = toggleSpell(spell.id)
    if (!success && !selectedSpells.has(spell.id)) {
      alert(`You've reached the maximum of ${maxSpells} spells...`)
    }
  }"
/>

<!-- AFTER -->
<UCheckbox
  @update:model-value="() => {
    const success = toggleSpell(spell.id, spell.level)
    const isCantrip = spell.level === 0
    if (!success) {
      if (isCantrip) {
        alert(`You've reached the maximum of ${maxCantrips} cantrips for this class.`)
      } else {
        alert(`You've reached the maximum of ${maxLeveledSpells} spells for this class and level.`)
      }
    }
  }"
/>
```

**Line ~335:** Update disabled check
```vue
<!-- BEFORE -->
:disabled="!selectedSpells.has(spell.id) && selectionCount >= maxSpells"

<!-- AFTER -->
:disabled="!selectedSpells.has(spell.id) && (
  spell.level === 0
    ? cantripCount >= maxCantrips
    : leveledSpellCount >= maxLeveledSpells
)"
```

#### 2. Update Summary Sidebar (Line ~380-387)

**BEFORE:**
```vue
<div class="text-gray-600 dark:text-gray-400">
  Selected: {{ selectionCount }} / {{ maxSpells }} spells
</div>
```

**AFTER:**
```vue
<div class="text-gray-600 dark:text-gray-400 space-y-1">
  <div>Cantrips: {{ cantripCount }} / {{ maxCantrips }}</div>
  <div>Spells: {{ leveledSpellCount }} / {{ maxLeveledSpells }}</div>
</div>
```

#### 3. Update Spell Info Display (Line ~250-256)

**Add cantrip count to spell slots display:**
```vue
<div class="flex items-center gap-4 text-sm">
  <span class="font-medium">📊 Spell Slots:</span>
  <span>Cantrips: {{ spellSlots.cantrips }}</span>
  <!-- ... existing slots ... -->
</div>
<div class="text-sm">
  <span class="font-medium">🪄 Cantrips to Select:</span>
  {{ maxCantrips }}
</div>
<div class="text-sm">
  <span class="font-medium">📝 Spells to Prepare:</span>
  {{ maxLeveledSpells }} ({{ characterLevel }} + 3 modifier)
</div>
```

#### 4. Update localStorage (Optional Enhancement)

Currently saves all to `selectedSpells`. Could enhance to save separately:

```typescript
// In composable
const saveToStorage = () => {
  const key = getStorageKey()
  if (!key) return

  const data = {
    classSlug: selectedClass.value!.slug,
    characterLevel: characterLevel.value,
    selectedCantrips: Array.from(selectedCantrips.value),
    selectedLeveledSpells: Array.from(selectedLeveledSpells.value),
    // Keep deprecated for backwards compat
    selectedSpells: Array.from(selectedSpells.value)
  }
  localStorage.setItem(key, JSON.stringify(data))
}

const loadFromStorage = () => {
  const key = getStorageKey()
  if (!key) return

  const stored = localStorage.getItem(key)
  if (!stored) return

  try {
    const data = JSON.parse(stored)
    characterLevel.value = data.characterLevel || 1

    // Try new format first
    if (data.selectedCantrips && data.selectedLeveledSpells) {
      selectedCantrips.value = new Set(data.selectedCantrips || [])
      selectedLeveledSpells.value = new Set(data.selectedLeveledSpells || [])
    } else {
      // Fallback to old format (all mixed)
      selectedSpells.value = new Set(data.selectedSpells || [])
    }
  } catch (e) {
    console.error('Failed to load spell list from localStorage:', e)
  }
}
```

---

## 🧪 Testing Checklist

After completing Phase 2 UI updates:

- [ ] Page loads without errors (HTTP 200)
- [ ] Can select cantrips (shows "Cantrips: X/Y")
- [ ] Can select leveled spells (shows "Spells: X/Y")
- [ ] Cantrip limit enforced (alert shows cantrip-specific message)
- [ ] Leveled spell limit enforced (alert shows spell-specific message)
- [ ] Can mix any spell levels (e.g., 4× 1st + 2× 2nd = 6 total)
- [ ] Checkboxes disabled at appropriate limits
- [ ] localStorage saves/loads correctly
- [ ] Level changes update max spells correctly
- [ ] Class changes clear selections (with confirmation)
- [ ] All existing tests still pass

**Example Test Scenario:**
1. Select Wizard, Level 3
2. Should see: "Cantrips: 0/3" and "Spells: 0/6"
3. Select 3 cantrips → "Cantrips: 3/3"
4. Try 4th cantrip → Alert: "reached maximum of 3 cantrips"
5. Select 2× 1st level + 4× 2nd level → "Spells: 6/6"
6. Try 7th spell → Alert: "reached maximum of 6 spells"

---

## 📚 D&D 5e Rules Reference

### Prepared Casters (Wizard, Cleric, Druid, Paladin, Artificer)
- **Cantrips:** Fixed from `cantrips_known` (e.g., 3)
- **Prepared Spells:** `character_level + ability_modifier` (e.g., 3 + 3 = 6)
- **Can prepare any spell levels they can cast** (mix freely)

### Known Casters (Bard, Sorcerer, Warlock, Ranger)
- **Cantrips:** Fixed from `cantrips_known` table
- **Known Spells:** Fixed from lookup table (e.g., Bard L3 = 6 spells)
- **Can know any spell levels they can cast** (mix freely)

### Spell Slots vs Selection
- **Spell Slots:** For CASTING (4× 1st, 2× 2nd, etc.)
- **Selection/Prepared/Known:** For CHOOSING which spells available
- **These are SEPARATE concepts!**

---

## 🎓 Key Learnings

### Why This Refactor Was Needed

**Old (Incorrect) Behavior:**
```
Wizard Level 3:
- maxSpells = 6
- Can select: 6 total (cantrips + leveled mixed)
- Problem: Cantrips counted against prepared limit ❌
```

**New (Correct) Behavior:**
```
Wizard Level 3:
- maxCantrips = 3 (separate limit)
- maxLeveledSpells = 6 (separate limit)
- Can select: 3 cantrips + 6 spells = 9 total ✅
- Cantrips don't count against prepared limit ✅
```

### TDD Benefits on This Refactor

1. **Clear Requirements:** Tests described exact D&D 5e behavior
2. **Safety:** Caught edge cases (spell level 0 vs 1-9)
3. **Confidence:** 11/11 tests passing = composable logic correct
4. **Backwards Compat:** Old tests updated to still work

---

## 📝 Files Reference

**Composable:**
- `app/composables/useSpellListGenerator.ts` - Core logic (183 lines changed)

**Tests:**
- `tests/composables/useSpellListGenerator.test.ts` - All tests (11 passing)

**UI (Phase 2):**
- `app/pages/spells/list-generator.vue` - Needs updates (see above)

**Commits:**
- `cf2dab4` - refactor: Separate cantrip and leveled spell tracking (TDD Phase 1)
- `3499310` - fix: Level selector locked after class selection
- `7d5c1f3` - fix: Level selector not responding to clicks
- `a654b07` - fix: Add UX improvements and validation
- `56ab982` - fix: Reduce per_page limit to 100
- `1bc9683` - fix: Update spell list generator to use Meilisearch API

---

## 🚀 Next Agent Instructions

1. **Read this document fully** before starting
2. **Understand the composable changes** are complete and tested
3. **Focus on UI updates** listed in "Phase 2 TODO" section
4. **Follow the exact code examples** provided above
5. **Test thoroughly** using the testing checklist
6. **Update CHANGELOG.md** when complete
7. **Commit with descriptive message** referencing Phase 2

**Estimated Time:** 30-45 minutes for Phase 2 UI updates

**Priority:** HIGH - Composable works but UI is broken without these changes!

---

**End of Phase 1 Handover**

Next session: Complete Phase 2 (UI updates) to make the refactor functional.
