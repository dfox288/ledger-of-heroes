# Character Builder Phase 4 Design

**Date:** 2025-12-03
**Issue:** [#89](https://github.com/dfox288/dnd-rulebook-project/issues/89)
**Status:** Approved

---

## Overview

Phase 4 implements the remaining Character Builder wizard steps:
- **Step 5:** Background Selection
- **Step 6:** Equipment Selection (NEW)
- **Step 7:** Spell Selection (casters only)
- **Step 8:** Review & Complete

This phase completes the guided character creation wizard.

---

## Wizard Flow (Updated)

```
Step 1: Name           ✅ Complete
Step 2: Race           ✅ Complete
Step 3: Class          ✅ Complete
Step 4: Abilities      ✅ Complete
Step 5: Background     🔲 This phase
Step 6: Equipment      🔲 This phase (NEW step)
Step 7: Spells         🔲 This phase (casters only)
Step 8: Review         🔲 This phase
```

**Step Count:**
- Non-casters: 7 steps (skip Spells)
- Casters: 8 steps

---

## Store Changes

### New State

```typescript
// Equipment choices (class + background combined)
const equipmentChoices = ref<Map<string, number>>(new Map())  // choice_group → selected item_id

// Race spell choices (e.g., High Elf cantrip)
const raceSpellChoices = ref<Map<string, number>>(new Map())  // choice_group → selected spell_id

// Note: selectedSpells already exists for class spells
```

### New Actions

```typescript
// Step 5: Background
async function selectBackground(background: Background): Promise<void>

// Step 6: Equipment
async function setEquipmentChoice(choiceGroup: string, itemId: number): void  // Local state only
async function confirmEquipment(): Promise<void>  // Bulk save to API

// Step 7: Spells
async function learnSpell(spellId: number): Promise<void>
async function unlearnSpell(spellId: number): Promise<void>
async function setRaceSpellChoice(choiceGroup: string, spellId: number): void
async function confirmSpells(): Promise<void>
```

### Updated Computed

```typescript
// Step count changes
const totalSteps = computed(() => isCaster.value ? 8 : 7)

// Combined equipment from class + background
const allEquipment = computed(() => [
  ...(selectedClass.value?.equipment ?? []),
  ...(selectedBackground.value?.equipment ?? [])
])

// Equipment grouped by choice_group
const equipmentByChoiceGroup = computed(() => {
  const groups = new Map<string, EntityItemResource[]>()
  for (const item of allEquipment.value) {
    if (item.is_choice && item.choice_group) {
      const existing = groups.get(item.choice_group) ?? []
      groups.set(item.choice_group, [...existing, item])
    }
  }
  return groups
})

// Fixed equipment (no choice required)
const fixedEquipment = computed(() =>
  allEquipment.value.filter(item => !item.is_choice)
)

// Validation: all equipment choices made?
const allEquipmentChoicesMade = computed(() => {
  for (const [group] of equipmentByChoiceGroup.value) {
    if (!equipmentChoices.value.has(group)) return false
  }
  return true
})
```

---

## Step 5: Background Selection

### Components

| Component | Purpose |
|-----------|---------|
| `StepBackground.vue` | Main step - grid picker with search |
| `BackgroundPickerCard.vue` | Selectable card with checkmark |
| `BackgroundDetailModal.vue` | Full background info modal |

### BackgroundPickerCard.vue

Displays:
- Background name (title)
- Feature name (badge)
- Skill proficiencies (2 skills)
- Languages count
- Tool proficiencies (if any)

Props:
```typescript
interface Props {
  background: Background
  selected: boolean
}

const emit = defineEmits<{
  select: [background: Background]
  viewDetails: []
}>()
```

### BackgroundDetailModal.vue

Full background details including:
- Description
- Feature name + description
- Skill proficiencies
- Tool proficiencies
- Languages
- Equipment preview
- Personality traits / ideals / bonds / flaws tables

### StepBackground.vue

Pattern follows `StepRace.vue`:
1. Fetch backgrounds: `GET /backgrounds?per_page=100`
2. Local state for selection before confirm
3. Search filter
4. Grid of `BackgroundPickerCard`
5. Confirm button calls `store.selectBackground()` + `nextStep()`

### API Integration

```typescript
// Select background
PATCH /characters/{id} { background_id: number }
```

---

## Step 6: Equipment Selection

### Components

| Component | Purpose |
|-----------|---------|
| `StepEquipment.vue` | Main step container |
| `EquipmentChoiceGroup.vue` | Radio group for choice options |
| `EquipmentFixedItem.vue` | Display-only item row |
| `EquipmentSourceSection.vue` | Groups items by source (class/background) |

### Data Flow

```
selectedClass.equipment ──┐
                          ├──► allEquipment ──► UI
selectedBackground.equipment ─┘
```

### EquipmentChoiceGroup.vue

Props:
```typescript
interface Props {
  groupName: string
  items: EntityItemResource[]
  selectedItemId: number | null
}

const emit = defineEmits<{
  select: [itemId: number]
}>()
```

UI: Radio button group with item names and descriptions.

### EquipmentFixedItem.vue

Props:
```typescript
interface Props {
  item: EntityItemResource
}
```

UI: Simple row with item name and quantity.

### StepEquipment.vue Layout

```
┌─────────────────────────────────────────────────┐
│  Choose Your Starting Equipment                  │
├─────────────────────────────────────────────────┤
│                                                  │
│  FROM YOUR CLASS ({className})                   │
│  ─────────────────────────────                   │
│  [EquipmentFixedItem] × n                        │
│  [EquipmentChoiceGroup] × n                      │
│                                                  │
│  FROM YOUR BACKGROUND ({backgroundName})         │
│  ─────────────────────────────                   │
│  [EquipmentFixedItem] × n                        │
│  [EquipmentChoiceGroup] × n (if any)             │
│                                                  │
│           [ Continue with Equipment ]            │
│           (disabled until all choices made)      │
└─────────────────────────────────────────────────┘
```

### API Integration

```typescript
// Add equipment to character inventory
POST /characters/{id}/equipment { item_id: number, quantity: number }
```

Equipment is saved in bulk when user clicks "Continue".

---

## Step 7: Spell Selection (Casters Only)

### Components

| Component | Purpose |
|-----------|---------|
| `StepSpells.vue` | Main step container |
| `SpellPickerCard.vue` | Compact selectable spell card |
| `SpellDetailModal.vue` | Full spell info (may reuse existing) |
| `RaceSpellSection.vue` | Race-granted spells (fixed + choices) |
| `SpellLevelSection.vue` | Class spells grouped by level |

### Three Spell Categories

| Category | Source | UI Treatment |
|----------|--------|--------------|
| Race-granted (fixed) | `selectedRace.spells` where `is_choice: false` | Display only |
| Race-granted (choice) | `selectedRace.spells` where `is_choice: true` | Radio/select |
| Class spells | API: `/characters/{id}/available-spells` | Multi-select |

### Class Spell Rules (Level 1)

| Class | Cantrips | Spells | Type |
|-------|----------|--------|------|
| Wizard | 3 | 6 in spellbook | Known |
| Cleric | 3 | WIS mod + 1 | Prepared |
| Sorcerer | 4 | 2 | Known |
| Bard | 2 | 4 | Known |
| Warlock | 2 | 2 | Known |
| Druid | 2 | WIS mod + 1 | Prepared |
| Ranger | 0 | 0 at L1 | Known |
| Paladin | 0 | 0 at L1 | Prepared |
| Artificer | 2 | INT mod + 1 | Prepared |

### SpellPickerCard.vue

Compact card for selection:
- Spell name
- Level indicator (cantrip / 1st)
- School icon
- Concentration badge (if applicable)
- Checkbox/toggle for selection

Props:
```typescript
interface Props {
  spell: Spell
  selected: boolean
  disabled?: boolean  // If at limit
}
```

### StepSpells.vue Layout

```
┌─────────────────────────────────────────────────┐
│  Select Your Spells                              │
├─────────────────────────────────────────────────┤
│                                                  │
│  RACIAL SPELLS ({raceName})          [if any]   │
│  ─────────────────────────────                   │
│  [Fixed spells - display only]                   │
│  [Choice group - radio buttons]                  │
│                                                  │
│  CANTRIPS ({className}) — {n} of {max} selected  │
│  ─────────────────────────────                   │
│  [Grid of SpellPickerCard]                       │
│                                                  │
│  1ST LEVEL SPELLS — {n} of {max} selected        │
│  ─────────────────────────────                   │
│  [Grid of SpellPickerCard]                       │
│                                                  │
│              [ Continue with Spells ]            │
│              (disabled until requirements met)   │
└─────────────────────────────────────────────────┘
```

### API Integration

```typescript
// Fetch available spells
GET /characters/{id}/available-spells?max_level=1

// Learn a spell
POST /characters/{id}/spells { spell_id: number }

// Unlearn a spell
DELETE /characters/{id}/spells/{spellId}

// Get spell limits from stats
GET /characters/{id}/stats
// Returns: preparation_limit, spell_slots, spellcasting info
```

### Validation

- All racial spell choices made (if any)
- Correct number of cantrips selected
- Correct number of spells selected (or prepared)
- Button disabled until validation passes

---

## Step 8: Review & Complete

### Components

| Component | Purpose |
|-----------|---------|
| `StepReview.vue` | Summary of all selections |
| `ReviewSection.vue` | Collapsible section for each category |

### Sections

1. **Character Identity**
   - Name
   - Race (+ subrace if applicable)
   - Class
   - Background

2. **Ability Scores**
   - Base scores
   - Racial bonuses applied
   - Final modifiers

3. **Proficiencies**
   - Skills (from class + background)
   - Tools (from background)
   - Weapons & Armor (from class)
   - Languages (from race + background)

4. **Starting Equipment**
   - All items with quantities
   - Choices resolved

5. **Spells** (casters only)
   - Cantrips known
   - 1st level spells known/prepared
   - Spellcasting ability & save DC

### Final Action

"Create Character" button:
1. Validates all required fields
2. Marks character as complete via API
3. Redirects to character sheet or list

---

## File Structure

```
app/components/character/builder/
├── StepBackground.vue          # Step 5
├── BackgroundPickerCard.vue
├── BackgroundDetailModal.vue
├── StepEquipment.vue           # Step 6
├── EquipmentChoiceGroup.vue
├── EquipmentFixedItem.vue
├── EquipmentSourceSection.vue
├── StepSpells.vue              # Step 7
├── SpellPickerCard.vue
├── SpellDetailModal.vue
├── RaceSpellSection.vue
├── SpellLevelSection.vue
├── StepReview.vue              # Step 8
└── ReviewSection.vue
```

---

## Test Strategy

Each step gets its own test file:
- `StepBackground.spec.ts`
- `StepEquipment.spec.ts`
- `StepSpells.spec.ts`
- `StepReview.spec.ts`

Plus component tests:
- `BackgroundPickerCard.spec.ts`
- `EquipmentChoiceGroup.spec.ts`
- `SpellPickerCard.spec.ts`

Store action tests in `characterBuilder.spec.ts`.

---

## Dependencies

- Backend must be running for API calls
- Existing mock factories can be extended for new types
- Reuse existing `SpellCard` styling patterns

---

## Success Criteria

- [ ] All 4 steps implemented with TDD
- [ ] Equipment choices from both class + background work
- [ ] Racial spell choices work (High Elf cantrip, etc.)
- [ ] Class spell selection respects limits
- [ ] Review shows complete character summary
- [ ] Character can be marked complete
- [ ] All tests pass
- [ ] TypeScript compiles
- [ ] ESLint passes
