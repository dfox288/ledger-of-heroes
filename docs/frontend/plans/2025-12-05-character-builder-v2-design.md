# Character Builder v2 - Design Document

**Date:** 2025-12-05
**Status:** Approved
**Related Issues:** #175, #176, #177, #178, #179, #180, #181, #184

## Overview

A complete rebuild of the character builder wizard with proper D&D 5e 2014 PHB compliance. The new architecture uses a lean store pattern where the backend is the source of truth for all calculated stats.

### Goals

1. Follow D&D 5e 2014 PHB rules accurately
2. Handle level-1 subclass selection (Cleric, Sorcerer, Warlock)
3. Display all calculated combat stats using backend `/stats` endpoint
4. Logical step order (Race → Class → Subclass → Background → Abilities → ...)
5. Use `subrace_required` flag for optional subraces
6. Show ability modifiers throughout, not just raw scores

### Non-Goals

- D&D 2024 rules support (background ability bonuses, origin feats)
- Multi-class support beyond level 1
- Character leveling workflow

---

## Architecture

### Route Structure

```
/characters/new                    → Redirects to /characters/new/sourcebooks
/characters/new/[step]             → Dynamic wizard step
/characters/[id]                   → Character view (existing)
/characters/[id]/edit/[step]       → Edit existing character (reuses wizard)
```

The `/new` flow creates a draft character on first meaningful selection (race), then continues with that character ID.

### Step Order

| # | Step | Visibility | Purpose |
|---|------|------------|---------|
| 1 | `sourcebooks` | Always | Filter available content by sourcebook |
| 2 | `race` | Always | Select base race |
| 3 | `subrace` | Conditional | If `subrace_required` OR has optional subraces |
| 4 | `class` | Always | Select class |
| 5 | `subclass` | Conditional | If `class.subclass_level === 1` |
| 6 | `background` | Always | Select background |
| 7 | `abilities` | Always | Assign ability scores |
| 8 | `proficiencies` | Conditional | If has skill choices |
| 9 | `languages` | Conditional | If has language choices |
| 10 | `equipment` | Always | Choose starting equipment |
| 11 | `spells` | Conditional | If spellcaster with spells at level 1 |
| 12 | `details` | Always | Name and alignment (moved to end) |
| 13 | `review` | Always | Full character sheet preview |

### State Management

**Core Principle:** Backend is source of truth. Store holds only local UI state.

```typescript
// stores/characterWizard.ts (~200 lines)
export const useCharacterWizardStore = defineStore('characterWizard', () => {
  // Character identity
  const characterId = ref<number | null>(null)
  const selectedSources = ref<string[]>([])

  // Local selections (before saving)
  const selections = ref({
    race: null as Race | null,
    subrace: null as Race | null,
    class: null as CharacterClass | null,
    subclass: null as Subclass | null,
    background: null as Background | null,
    abilityScores: defaultScores(),
    abilityMethod: 'standard_array' as AbilityMethod,
    name: '',
    alignment: null as Alignment | null,
  })

  // Pending multi-select choices
  const pendingChoices = ref({
    proficiencies: new Map<string, Set<number>>(),
    languages: new Map<string, Set<number>>(),
    equipment: new Map<string, number>(),
    spells: new Set<number>(),
  })

  // Backend data (read-only)
  const stats = ref<CharacterStats | null>(null)
  const summary = ref<CharacterSummary | null>(null)

  // UI state
  const isLoading = ref(false)
  const error = ref<string | null>(null)
})
```

### Composables

```typescript
// 1. useCharacterWizard - Navigation & orchestration
export function useCharacterWizard() {
  // Step registry, navigation, validation gates
  return { activeSteps, currentStep, canProceed, nextStep, previousStep, saveAndContinue }
}

// 2. useCharacterStats - Calculated stats display
export function useCharacterStats(characterId: Ref<number | null>) {
  // Fetches /stats, provides formatted display values
  return { stats, hitPoints, armorClass, proficiencyBonus, savingThrows, spellSaveDC, ... }
}

// 3. useAbilityScores - Score assignment logic
export function useAbilityScores() {
  // Standard array, point buy, manual with live modifiers
  return { method, scores, modifiers, pointBuyRemaining, isValid, formatScore }
}
```

---

## Data Flow

```
User selects race
        │
        ▼
┌───────────────────────────────┐
│  characterId exists?          │
│  NO → POST /characters        │
│  YES → PATCH /characters/{id} │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│  GET /characters/{id}/stats   │
│  GET /characters/{id}/summary │
└───────────────┬───────────────┘
                │
                ▼
        Store updated
        UI reflects new stats
```

**Key Points:**

1. **Character Creation** - Happens on Race selection (not Name)
2. **Stats Sync** - Called after every save via `syncStats()`
3. **Review Step** - Pure display from `/stats` endpoint, no client calculations

### API Endpoints Used

| Endpoint | Purpose |
|----------|---------|
| `POST /characters` | Create draft (race_id) |
| `PATCH /characters/{id}` | Update selections |
| `POST /characters/{id}/classes` | Add class |
| `PUT /characters/{id}/classes/{id}/subclass` | Set subclass |
| `GET /characters/{id}/stats` | All calculated stats |
| `GET /characters/{id}/summary` | Pending choices, completion status |
| `POST /characters/{id}/proficiency-choices` | Save skill choices |
| `POST /characters/{id}/language-choices` | Save language choices |
| `POST /characters/{id}/equipment` | Save equipment |
| `POST /characters/{id}/spells` | Save spell selections |

---

## Component Structure

```
app/
├── pages/characters/
│   └── new/
│       ├── index.vue              # Redirect to sourcebooks
│       └── [step].vue             # Dynamic step router
│
├── components/character/
│   ├── wizard/
│   │   ├── WizardLayout.vue       # Sidebar + content + footer
│   │   ├── WizardSidebar.vue      # Step list with progress
│   │   ├── WizardFooter.vue       # Back/Next buttons
│   │   ├── StepSourcebooks.vue
│   │   ├── StepRace.vue
│   │   ├── StepSubrace.vue
│   │   ├── StepClass.vue
│   │   ├── StepSubclass.vue       # NEW
│   │   ├── StepBackground.vue
│   │   ├── StepAbilities.vue
│   │   ├── StepProficiencies.vue
│   │   ├── StepLanguages.vue
│   │   ├── StepEquipment.vue
│   │   ├── StepSpells.vue
│   │   ├── StepDetails.vue        # NEW (name + alignment)
│   │   └── StepReview.vue
│   │
│   ├── stats/
│   │   ├── CombatStatsCard.vue    # HP, AC, Initiative
│   │   ├── SavingThrowsCard.vue   # 6 saves with proficiency
│   │   ├── SkillsCard.vue         # Skills with bonuses
│   │   ├── SpellcastingCard.vue   # DC, attack, slots
│   │   └── AbilityScoreDisplay.vue
│   │
│   └── picker/
│       └── SubclassPickerCard.vue # NEW
```

### Reuse Strategy

| Component | Verdict |
|-----------|---------|
| `RacePickerCard`, `ClassPickerCard`, `SubracePickerCard`, `BackgroundPickerCard`, `SpellPickerCard` | Reuse as-is |
| `RaceDetailModal`, `ClassDetailModal`, `BackgroundDetailModal`, `SpellDetailModal`, `SubraceDetailModal` | Reuse as-is |
| `StandardArrayInput`, `PointBuyInput`, `ManualInput` | Reuse as-is |
| `EquipmentChoiceGroup`, `EquipmentItemPicker` | Reuse as-is |
| Step components (`StepRace.vue`, etc.) | Rewrite - rewire to new composables |
| `ProgressBar` | Replace with `WizardSidebar` |

**Reuse: ~60% | Rewrite: ~40%**

---

## Review Step Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  CHARACTER NAME                                      [Edit ✎]  │
│  Race • Class • Level 1                                        │
│  Alignment                                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────────────────────────┐  │
│  │ COMBAT STATS    │  │ ABILITY SCORES                      │  │
│  ├─────────────────┤  ├─────────────────────────────────────┤  │
│  │ HP:  ##         │  │ STR ## (+#)  │  INT ## (+#)        │  │
│  │ AC:  ##         │  │ DEX ## (+#)  │  WIS ## (+#)        │  │
│  │ Initiative: +#  │  │ CON ## (+#)  │  CHA ## (+#)        │  │
│  │ Speed: ## ft    │  └─────────────────────────────────────┘  │
│  │ Prof: +2        │                                           │
│  └─────────────────┘  ┌─────────────────────────────────────┐  │
│                       │ SAVING THROWS                        │  │
│                       │ ● = proficient                       │  │
│                       └─────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ SKILLS (● = proficient)                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ PROFICIENCIES (Armor, Weapons, Tools, Languages)         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ FEATURES & TRAITS                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ EQUIPMENT                                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ SPELLCASTING (if applicable)                              │  │
│  │ Spell Save DC: ## | Attack: +# | Slots: # × 1st          │  │
│  │ Cantrips: ...                                             │  │
│  │ Prepared/Known: ...                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│                    [ Finish & View Character ]                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Conditional Step Logic

```typescript
// Subrace step visibility
const needsSubraceStep = computed(() => {
  if (!selections.value.race) return false
  const race = selections.value.race
  // Show if required OR if optional subraces exist
  return race.subrace_required === true || (race.subraces?.length ?? 0) > 0
})

// Subclass step visibility
const needsSubclassStep = computed(() => {
  return selections.value.class?.subclass_level === 1
})

// Spells step visibility
const isSpellcaster = computed(() => {
  const cls = selections.value.class
  if (!cls?.spellcasting_ability) return false
  const level1 = cls.level_progression?.find(p => p.level === 1)
  return (level1?.cantrips_known ?? 0) > 0 || (level1?.spells_known ?? 0) > 0
})
```

---

## Testing Strategy

```
tests/
├── components/character/
│   ├── wizard/
│   │   ├── WizardLayout.test.ts
│   │   ├── WizardSidebar.test.ts
│   │   ├── StepSubclass.test.ts
│   │   ├── StepDetails.test.ts
│   │   └── StepReview.test.ts
│   ├── stats/
│   │   ├── CombatStatsCard.test.ts
│   │   ├── SavingThrowsCard.test.ts
│   │   └── SpellcastingCard.test.ts
│   └── picker/
│       └── SubclassPickerCard.test.ts
├── composables/
│   ├── useCharacterWizard.test.ts
│   ├── useCharacterStats.test.ts
│   └── useAbilityScores.test.ts
├── stores/
│   └── characterWizard.test.ts
└── pages/characters/new/
    └── [step].test.ts
```

### Key Test Cases

| Area | Test Cases |
|------|------------|
| Step Visibility | Subrace shows for races with subraces; Subclass shows for Cleric/Sorcerer/Warlock |
| Navigation | Can't proceed without required selection; Skips hidden steps |
| Stats Display | Formats modifiers correctly; Shows proficiency indicators |
| Character Creation | Draft created on race selection; ID persists |

---

## Migration Notes

### Files to Deprecate

```
app/pages/characters/[id]/edit/*.vue  → Replace with new wizard
app/stores/characterBuilder.ts        → Replace with characterWizard.ts
app/composables/useWizardSteps.ts     → Replace with useCharacterWizard.ts
```

### Files to Keep

```
app/pages/characters/index.vue        → Character list (unchanged)
app/pages/characters/[id]/index.vue   → Character view (unchanged)
app/pages/characters/create.vue       → Redirect to /new (update)
app/components/character/builder/*PickerCard.vue  → Reuse
app/components/character/builder/*DetailModal.vue → Reuse
app/components/character/builder/*Input.vue       → Reuse
```

---

## Success Criteria

- [ ] All 13 steps functional
- [ ] Subclass selection works for Cleric, Sorcerer, Warlock
- [ ] All calculated stats displayed from `/stats` endpoint
- [ ] Review step shows complete character sheet
- [ ] `subrace_required` flag respected
- [ ] Tests pass for all new components/composables
- [ ] GitHub issues #175-181, #184 can be closed
