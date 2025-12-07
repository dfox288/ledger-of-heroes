# Classes & Backgrounds Missing Fields - Implementation Plan

**Date:** 2025-11-23
**Status:** 📋 Ready for Implementation
**Priority:** HIGH
**Estimated Time:** 3-4 hours

---

## 📋 Executive Summary

Audit revealed that **Classes** and **Backgrounds** detail pages are missing several important data fields that the API already provides. This document outlines the implementation plan to display these missing fields.

### Missing Fields Summary

**Classes (5 fields):**
- ✅ `level_progression` - Spell slots table (CRITICAL for spellcasters)
- ✅ `equipment` - Starting gear and proficiencies
- ⚠️ `spells` - Class spell list (low priority)
- ⚠️ `traits` - Class lore/flavor text (low priority)
- ⚠️ `counters` - Resource tracking (low priority)

**Backgrounds (1 field):**
- ✅ `equipment` - Starting equipment list

**This plan focuses on HIGH priority items only:**
1. Class level progression table
2. Class equipment
3. Background equipment

---

## 🎯 Goals

1. Display spell slot progression for spellcasting classes
2. Show starting equipment/proficiencies for classes
3. Show starting equipment for backgrounds
4. Maintain visual consistency with existing UI
5. Follow TDD principles (write tests first)

---

## 📊 Data Structure Analysis

### Class Level Progression

**API Response Example (Wizard):**
```json
{
  "level_progression": [
    {
      "id": 176,
      "level": 1,
      "cantrips_known": 3,
      "spells_known": null,
      "spell_slots_1st": 2,
      "spell_slots_2nd": 0,
      "spell_slots_3rd": 0,
      "spell_slots_4th": 0,
      "spell_slots_5th": 0,
      "spell_slots_6th": 0,
      "spell_slots_7th": 0,
      "spell_slots_8th": 0,
      "spell_slots_9th": 0
    },
    // ... levels 2-20
  ]
}
```

**Key Fields:**
- `level` - Character level (1-20)
- `cantrips_known` - Number of cantrips known
- `spells_known` - Number of spells known (null for prepared casters like Wizard)
- `spell_slots_1st` through `spell_slots_9th` - Spell slots per level

**Edge Cases:**
- Non-spellcasters (Rogue, Fighter) have `level_progression: []`
- Half-casters (Paladin, Ranger) start at level 2
- Third-casters (Arcane Trickster, Eldritch Knight) start at level 3
- Prepared casters (Wizard, Cleric) have `spells_known: null`
- Known casters (Sorcerer, Bard) have specific `spells_known` values

---

### Class Equipment

**API Response Example (Rogue):**
```json
{
  "equipment": [
    {
      "id": 45,
      "item_id": null,
      "quantity": 1,
      "is_choice": false,
      "choice_description": null,
      "proficiency_subcategory": null,
      "description": "level Rogue, you begin play with 8 + your Constitution modifier hit points."
    },
    {
      "id": 46,
      "item_id": null,
      "quantity": 1,
      "is_choice": false,
      "choice_description": null,
      "proficiency_subcategory": null,
      "description": "-- Armor: light armor\n\t--- Weapons: simple weapons, hand crossbows, longswords, rapiers, shortswords\n\t--- Tools: thieves' tools\n\t--- Skills: Choose 4 from Acrobatics, Athletics, Deception, Insight, Intimidation, Investigation, Perception, Performance, Persuasion, Sleight Of Hand, Stealth"
    },
    {
      "id": 47,
      "item_id": null,
      "quantity": 1,
      "is_choice": true,
      "choice_description": "Starting equipment choice",
      "proficiency_subcategory": null,
      "description": "a rapier"
    },
    {
      "id": 48,
      "item_id": null,
      "quantity": 1,
      "is_choice": true,
      "choice_description": "Starting equipment choice",
      "proficiency_subcategory": null,
      "description": "a dungeoneer's pack,"
    }
  ]
}
```

**Key Fields:**
- `is_choice` - Whether this is an optional choice (OR condition)
- `description` - Equipment text (may include formatting)
- `choice_description` - Groups related choices together
- `item_id` - Link to item (often null for generic descriptions)
- `quantity` - Amount of the item

**Structure:**
- First item is usually hit points calculation
- Second item is usually proficiencies summary
- Remaining items are starting equipment with choices

---

### Background Equipment

**API Response Example (House Agent):**
```json
{
  "equipment": [
    {
      "id": 70,
      "item_id": 1848,
      "quantity": 1,
      "is_choice": false,
      "choice_description": null,
      "proficiency_subcategory": null,
      "description": null
    },
    {
      "id": 71,
      "item_id": 1961,
      "quantity": 1,
      "is_choice": false,
      "choice_description": null,
      "proficiency_subcategory": null,
      "description": null
    },
    {
      "id": 72,
      "item_id": null,
      "quantity": 1,
      "is_choice": false,
      "choice_description": null,
      "proficiency_subcategory": null,
      "description": "identification papers"
    },
    {
      "id": 73,
      "item_id": 1942,
      "quantity": 1,
      "is_choice": false,
      "choice_description": null,
      "proficiency_subcategory": null,
      "description": null
    },
    {
      "id": 74,
      "item_id": 1860,
      "quantity": 20,
      "is_choice": false,
      "choice_description": null,
      "proficiency_subcategory": null,
      "description": null
    }
  ]
}
```

**Key Difference from Class Equipment:**
- More likely to have `item_id` references
- Simpler structure (rarely has choices)
- No hit points or proficiency summaries

---

## 🎨 UI Design

### Component 1: Level Progression Table

**File:** `app/components/ui/accordion/UiAccordionLevelProgression.vue`

**Visual Design:**
```
┌─────────────────────────────────────────────────────────────┐
│ Spell Slot Progression                                      │
├─────────────────────────────────────────────────────────────┤
│ Level │ Cantrips │ Spells │ 1st │ 2nd │ 3rd │ 4th │ 5th ... │
├───────┼──────────┼────────┼─────┼─────┼─────┼─────┼─────────┤
│   1   │    3     │   —    │  2  │  0  │  0  │  0  │  0  ... │
│   2   │    3     │   —    │  3  │  0  │  0  │  0  │  0  ... │
│   3   │    3     │   —    │  4  │  2  │  0  │  0  │  0  ... │
│  ...  │   ...    │  ...   │ ... │ ... │ ... │ ... │ ... ... │
└───────┴──────────┴────────┴─────┴─────┴─────┴─────┴─────────┘
```

**Features:**
- Responsive table with horizontal scroll on mobile
- Show only non-zero spell slot columns
- Display "—" for `null` spells_known
- Zebra striping for readability
- Compact font for space efficiency

**Props:**
```typescript
interface Props {
  levelProgression: LevelProgression[]
}

interface LevelProgression {
  id: number
  level: number
  cantrips_known: number | null
  spells_known: number | null
  spell_slots_1st: number
  spell_slots_2nd: number
  // ... through spell_slots_9th
}
```

---

### Component 2: Equipment List

**File:** `app/components/ui/accordion/UiAccordionEquipmentList.vue`

**Visual Design:**
```
┌─────────────────────────────────────────┐
│ Starting Equipment                      │
├─────────────────────────────────────────┤
│ Starting Hit Points                     │
│ • 8 + your Constitution modifier        │
│                                         │
│ Proficiencies                           │
│ • Armor: light armor                    │
│ • Weapons: simple weapons, hand         │
│   crossbows, longswords, rapiers,       │
│   shortswords                           │
│ • Tools: thieves' tools                 │
│ • Skills: Choose 4 from Acrobatics,     │
│   Athletics, Deception...               │
│                                         │
│ Equipment Choices                       │
│ Option 1:                               │
│   • (a) a rapier OR                     │
│   • (b) a shortsword                    │
│                                         │
│ Option 2:                               │
│   • (a) a shortbow and quiver of 20     │
│     arrows OR                           │
│   • (b) a shortsword                    │
│                                         │
│ Standard Equipment                      │
│ • Leather armor                         │
│ • Two daggers                           │
│ • Thieves' tools                        │
└─────────────────────────────────────────┘
```

**Features:**
- Group items by type (hit points, proficiencies, choices, standard)
- Group choices by `choice_description`
- Display as lettered options (a, b, c)
- Preserve formatting from description field
- Handle `item_id` references (future: link to item pages)

**Props:**
```typescript
interface Props {
  equipment: Equipment[]
  type?: 'class' | 'background' // Affects grouping logic
}

interface Equipment {
  id: number
  item_id: number | null
  quantity: number
  is_choice: boolean
  choice_description: string | null
  proficiency_subcategory: string | null
  description: string | null
}
```

---

## 📁 File Structure

```
app/
├── components/
│   └── ui/
│       └── accordion/
│           ├── UiAccordionLevelProgression.vue (NEW)
│           └── UiAccordionEquipmentList.vue (NEW)
├── pages/
│   ├── classes/
│   │   └── [slug].vue (MODIFY - add level_progression, equipment)
│   └── backgrounds/
│       └── [slug].vue (MODIFY - add equipment)
└── types/
    └── api/
        └── entities.ts (MODIFY - add LevelProgression type if missing)

tests/
└── components/
    └── ui/
        └── accordion/
            ├── UiAccordionLevelProgression.test.ts (NEW)
            └── UiAccordionEquipmentList.test.ts (NEW)
```

---

## 🔨 Implementation Steps

### Phase 1: Level Progression Component (TDD)

**Step 1.1: Write Tests First** ⚠️ MANDATORY
```bash
# Create test file
touch tests/components/ui/accordion/UiAccordionLevelProgression.test.ts
```

**Test Cases:**
1. ✅ Renders table with correct headers
2. ✅ Displays all 20 levels
3. ✅ Shows cantrips_known column when present
4. ✅ Shows spells_known column when present
5. ✅ Displays "—" for null values
6. ✅ Hides columns for spell levels that are always 0
7. ✅ Handles half-casters (starts at level 2)
8. ✅ Handles third-casters (starts at level 3)
9. ✅ Handles empty array gracefully

**Step 1.2: Create Component**
```bash
touch app/components/ui/accordion/UiAccordionLevelProgression.vue
```

**Step 1.3: Run Tests** - Watch them FAIL (RED phase)

**Step 1.4: Implement Minimum Code** - Make tests pass (GREEN phase)

**Step 1.5: Refactor** - Clean up while keeping tests green

---

### Phase 2: Equipment List Component (TDD)

**Step 2.1: Write Tests First** ⚠️ MANDATORY
```bash
touch tests/components/ui/accordion/UiAccordionEquipmentList.test.ts
```

**Test Cases:**
1. ✅ Renders equipment list
2. ✅ Groups items by choice_description
3. ✅ Displays choices with letters (a, b, c)
4. ✅ Separates hit points section
5. ✅ Separates proficiencies section
6. ✅ Handles items with quantities > 1
7. ✅ Preserves whitespace/formatting in descriptions
8. ✅ Handles background equipment (simpler format)
9. ✅ Handles empty equipment array

**Step 2.2: Create Component**
```bash
touch app/components/ui/accordion/UiAccordionEquipmentList.vue
```

**Step 2.3: Run Tests** - Watch them FAIL (RED phase)

**Step 2.4: Implement Minimum Code** - Make tests pass (GREEN phase)

**Step 2.5: Refactor** - Clean up while keeping tests green

---

### Phase 3: Integrate into Classes Detail Page

**File:** `app/pages/classes/[slug].vue`

**Changes:**

1. Add to accordion items array (around line 94-120):
```typescript
<UAccordion
  :items="[
    ...(entity.level_progression && entity.level_progression.length > 0 ? [{
      label: 'Spell Slot Progression',
      slot: 'level-progression',
      defaultOpen: false
    }] : []),
    ...(entity.equipment && entity.equipment.length > 0 ? [{
      label: 'Starting Equipment & Proficiencies',
      slot: 'equipment',
      defaultOpen: false
    }] : []),
    ...(entity.proficiencies && entity.proficiencies.length > 0 ? [{
      label: `Proficiencies (${entity.proficiencies.length})`,
      slot: 'proficiencies',
      defaultOpen: false
    }] : []),
    // ... existing items
  ]"
>
```

2. Add template slots (after existing slots):
```vue
<!-- Level Progression Slot -->
<template
  v-if="entity.level_progression && entity.level_progression.length > 0"
  #level-progression
>
  <UiAccordionLevelProgression :level-progression="entity.level_progression" />
</template>

<!-- Equipment Slot -->
<template
  v-if="entity.equipment && entity.equipment.length > 0"
  #equipment
>
  <UiAccordionEquipmentList :equipment="entity.equipment" type="class" />
</template>
```

**Testing:**
- ✅ Test with Wizard (full caster, starts at level 1)
- ✅ Test with Arcane Trickster (third-caster, starts at level 3)
- ✅ Test with Rogue (no spellcasting, equipment only)
- ✅ Run full test suite: `docker compose exec nuxt npm run test`

---

### Phase 4: Integrate into Backgrounds Detail Page

**File:** `app/pages/backgrounds/[slug].vue`

**Changes:**

1. Add to accordion items array (around line 133-146):
```typescript
<UAccordion
  :items="[
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
    // ... existing items
  ]"
>
```

2. Add template slot (after existing slots):
```vue
<!-- Equipment Slot -->
<template
  v-if="entity.equipment && entity.equipment.length > 0"
  #equipment
>
  <UiAccordionEquipmentList :equipment="entity.equipment" type="background" />
</template>
```

**Testing:**
- ✅ Test with House Agent background
- ✅ Verify equipment displays correctly
- ✅ Run full test suite: `docker compose exec nuxt npm run test`

---

### Phase 5: Type Definitions

**File:** `app/types/api/entities.ts`

Add if missing:

```typescript
export interface LevelProgression {
  id: number
  level: number
  cantrips_known: number | null
  spells_known: number | null
  spell_slots_1st: number
  spell_slots_2nd: number
  spell_slots_3rd: number
  spell_slots_4th: number
  spell_slots_5th: number
  spell_slots_6th: number
  spell_slots_7th: number
  spell_slots_8th: number
  spell_slots_9th: number
}

export interface Equipment {
  id: number
  item_id: number | null
  quantity: number
  is_choice: boolean
  choice_description: string | null
  proficiency_subcategory: string | null
  description: string | null
}

// Update CharacterClass interface
export interface CharacterClass {
  // ... existing fields
  level_progression?: LevelProgression[]
  equipment?: Equipment[]
  // ... rest of fields
}

// Update Background interface
export interface Background {
  // ... existing fields
  equipment?: Equipment[]
  // ... rest of fields
}
```

---

## ✅ Acceptance Criteria

### Must Have:
- [ ] Level progression table displays for spellcasting classes
- [ ] Level progression table correctly handles:
  - [ ] Full casters (Wizard: levels 1-20)
  - [ ] Third-casters (Arcane Trickster: levels 3-20)
  - [ ] Non-casters (Rogue: no table displayed)
- [ ] Equipment displays for all classes
- [ ] Equipment displays for all backgrounds
- [ ] Equipment correctly groups:
  - [ ] Hit points (separate section)
  - [ ] Proficiencies (separate section)
  - [ ] Choices (grouped by choice_description)
  - [ ] Standard equipment
- [ ] All new components have comprehensive tests
- [ ] All tests pass (100% pass rate maintained)
- [ ] TypeScript compiles with 0 errors
- [ ] Visual consistency with existing UI

### Nice to Have:
- [ ] Level progression table is sortable
- [ ] Equipment with `item_id` links to item detail page
- [ ] Equipment choices show tooltips with item details

---

## 🧪 Testing Strategy

### Unit Tests (Vitest + @nuxt/test-utils)

**UiAccordionLevelProgression.test.ts:**
- Rendering tests (headers, rows, data accuracy)
- Edge case tests (null values, empty arrays, different caster types)
- Visual consistency tests (column visibility, formatting)

**UiAccordionEquipmentList.test.ts:**
- Rendering tests (sections, grouping, formatting)
- Edge case tests (null descriptions, missing item_id, quantities)
- Type-specific tests (class vs background differences)

### Integration Tests
- Test actual class pages (Wizard, Rogue, Arcane Trickster)
- Test actual background pages (House Agent)
- Verify accordion behavior
- Test responsive design (mobile, tablet, desktop)

### Manual Testing Checklist
- [ ] Wizard: Full spell progression table (1-20, all 9 levels)
- [ ] Rogue: Equipment only, no spell table
- [ ] Arcane Trickster: Spell table starts at level 3
- [ ] House Agent: Background equipment displays
- [ ] Dark mode works correctly
- [ ] Mobile responsive (table scrolls horizontally)
- [ ] All existing functionality still works

---

## 📝 Notes & Considerations

### Design Decisions

**Why separate UiAccordionEquipmentList from proficiencies?**
- Equipment data structure is different (has choices, item_id, quantities)
- Grouping logic is complex and specific to equipment
- Proficiencies are already handled by `UiAccordionBulletList`

**Why not display `spells` and `traits` arrays?**
- `spells`: Class spell lists are VERY long (Wizard has 200+ spells)
- Would create massive accordion sections
- Better suited for separate "Class Spells" page or modal
- Can be added later if needed

**Why conditional rendering for level_progression?**
- Not all classes have spellcasting (Rogue, Barbarian, Fighter)
- Empty array should not display empty table
- Cleaner UX to hide when not applicable

### Performance Considerations

- Level progression table is 20 rows × 11-12 columns = ~240 cells
- Use `v-for` efficiently, avoid nested watchers
- Consider virtual scrolling if table gets laggy (unlikely)

### Future Enhancements

**Phase 2 (Future):**
- [ ] Add `spells` array display (modal or separate page)
- [ ] Add `traits` array display (class lore section)
- [ ] Add `counters` display (resource tracking)
- [ ] Link equipment `item_id` to item detail pages
- [ ] Add tooltips for equipment with item details
- [ ] Add print-friendly styles for level progression

---

## 🚀 Ready to Implement

This plan provides:
- ✅ Clear scope (what to build)
- ✅ Data structure analysis (what we're working with)
- ✅ UI mockups (what it should look like)
- ✅ Step-by-step implementation (how to build it)
- ✅ Test coverage requirements (how to verify it works)
- ✅ Acceptance criteria (when it's done)

**Estimated Time:** 3-4 hours
- Phase 1 (Level Progression): 1.5 hours
- Phase 2 (Equipment List): 1 hour
- Phase 3-4 (Integration): 30 minutes
- Phase 5 (Types): 15 minutes
- Testing & verification: 45 minutes

**Next Step:** Start Phase 1 with TDD! Write tests first, watch them fail, then implement.
