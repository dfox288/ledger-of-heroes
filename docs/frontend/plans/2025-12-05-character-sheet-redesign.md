# Character Sheet Redesign - Design Document

**Issue:** #172 - Redesign Character View Page to Match D&D 5e Character Sheet Layout
**Date:** 2025-12-05
**Status:** Design Complete

## Overview

Redesign the character view page (`/characters/[id]`) from a basic placeholder to a comprehensive digital character sheet inspired by the official D&D 5e layout, adapted for web with responsive design.

## Architecture Decisions

### 1. Data Fetching: Single Composable

**Decision:** Create `useCharacterSheet` composable that fetches all data in parallel and exposes computed values.

**Rationale:**
- Cleaner page component with minimal data management code
- Easier to test - composable can be unit tested independently
- Single loading state aggregated from all fetches
- Computed skills and saving throws calculated once, used by multiple components

### 2. Component Organization: Flat Structure

**Decision:** All components in `app/components/character/sheet/` folder.

**Components:**
- `Header.vue` - Name, race, class(es), level, badges, edit button
- `AbilityScoreBlock.vue` - Vertical ability score display with modifiers
- `CombatStatsGrid.vue` - HP, AC, Initiative, Speed, Proficiency Bonus, Passives
- `SavingThrowsList.vue` - 6 saves with proficiency indicators
- `SkillsList.vue` - All 18 skills with proficiency/expertise indicators
- `FeaturesPanel.vue` - Class/race/background features (tab content)
- `ProficienciesPanel.vue` - Weapons/armor/tools/languages (tab content)
- `EquipmentPanel.vue` - Item list with quantities (tab content)
- `SpellsPanel.vue` - Spell slots, cantrips, prepared spells (tab content)
- `LanguagesPanel.vue` - Simple tag display (tab content)

**Naming:** Auto-imports as `CharacterSheetHeader`, `CharacterSheetSkillsList`, etc.

### 3. Bottom Sections: Tabs

**Decision:** Use NuxtUI `UTabs` for Features, Proficiencies, Equipment, Spells, Languages sections.

**Rationale:**
- Clean, familiar navigation pattern
- One section visible at a time reduces cognitive load
- Collapses gracefully on mobile
- Spells tab conditionally rendered only for casters

## API Endpoints Used

All endpoints have existing Nitro routes:

| Endpoint | Data Provided |
|----------|---------------|
| `/characters/{id}` | Basic info, ability scores, speeds, HP, AC, classes array |
| `/characters/{id}/stats` | Detailed ability scores with modifiers, saving throws, hit dice, spellcasting stats, passive scores |
| `/characters/{id}/proficiencies` | Skill proficiencies with expertise flag, skill details |
| `/characters/{id}/features` | Class/race/background features with descriptions |
| `/characters/{id}/equipment` | Character's items with quantities |
| `/characters/{id}/spells` | Known and prepared spells |
| `/characters/{id}/languages` | Character's known languages |
| `/skills` (reference) | All 18 skills with ability codes |

## Key Technical Details

### Skills Calculation

The API does not provide computed skill modifiers. They must be calculated client-side:

```typescript
// For each of the 18 skills:
modifier = ability_modifier + (proficiency_bonus * isProficient) + (proficiency_bonus * hasExpertise)
```

Where:
- `ability_modifier` comes from `/stats` endpoint
- `proficiency_bonus` comes from `/characters/{id}` (level-based)
- `isProficient` determined by checking if skill exists in `/proficiencies` response
- `hasExpertise` from `expertise` field in proficiency record

### Saving Throws

The `/stats` endpoint returns saving throw modifiers. Proficiency is determined by class features.

### Conditional Rendering

- **SpellsPanel:** Only render if `stats.spellcasting !== null`
- **Edit button:** Only show if `!character.is_complete`
- **Inspiration badge:** Only show if `character.has_inspiration === true`
- **Empty states:** Each panel handles empty arrays with placeholder messages

## Responsive Layout

### Desktop (lg+)

```
┌────────────────────────────────────────────────────────────────────┐
│ [Header - full width]                                               │
├───────────────┬────────────────────────────────────────────────────┤
│ AbilityScore  │ CombatStatsGrid                                     │
│ Block         │─────────────────────────────────────────────────────│
│ (sidebar)     │ SavingThrowsList  |  SkillsList                     │
│               │ (2-column grid for saves and skills)                │
├───────────────┴────────────────────────────────────────────────────┤
│ [UTabs: Features | Proficiencies | Equipment | Spells | Languages] │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Active tab panel content                                      │   │
│ └──────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

### Mobile (sm)

All sections stack vertically:
1. Header
2. AbilityScoreBlock (3x2 grid or horizontal scroll)
3. CombatStatsGrid
4. SavingThrowsList
5. SkillsList
6. Tabs (horizontally scrollable tab bar)

## File Structure

```
app/
├── composables/
│   └── useCharacterSheet.ts          # NEW: Aggregate data fetching
├── components/character/sheet/
│   ├── Header.vue                    # NEW
│   ├── AbilityScoreBlock.vue         # NEW
│   ├── CombatStatsGrid.vue           # NEW
│   ├── SavingThrowsList.vue          # NEW
│   ├── SkillsList.vue                # NEW
│   ├── FeaturesPanel.vue             # NEW
│   ├── ProficienciesPanel.vue        # NEW
│   ├── EquipmentPanel.vue            # NEW
│   ├── SpellsPanel.vue               # NEW
│   └── LanguagesPanel.vue            # NEW
├── pages/characters/[id]/
│   └── index.vue                     # MODIFIED: Complete rewrite
└── types/
    └── character.ts                  # MODIFIED: Add skill types

tests/
├── composables/
│   └── useCharacterSheet.test.ts     # NEW
└── components/character/sheet/
    ├── Header.test.ts                # NEW
    ├── SkillsList.test.ts            # NEW
    └── ... (tests for each component)
```

## Type Definitions

```typescript
// New types needed in app/types/character.ts

/** Skill with computed modifier for display */
export interface CharacterSkill {
  id: number
  name: string
  slug: string
  ability_code: AbilityScoreCode
  modifier: number
  proficient: boolean
  expertise: boolean
}

/** Saving throw with computed values */
export interface CharacterSavingThrow {
  ability: AbilityScoreCode
  modifier: number
  proficient: boolean
}

/** Return type for useCharacterSheet composable */
export interface UseCharacterSheetReturn {
  // Raw API data
  character: ComputedRef<Character | null>
  stats: ComputedRef<CharacterStats | null>
  proficiencies: ComputedRef<CharacterProficiency[]>
  features: ComputedRef<CharacterFeature[]>
  equipment: ComputedRef<CharacterEquipment[]>
  spells: ComputedRef<CharacterSpell[]>
  languages: ComputedRef<CharacterLanguage[]>

  // Computed/derived
  skills: ComputedRef<CharacterSkill[]>
  savingThrows: ComputedRef<CharacterSavingThrow[]>

  // State
  loading: ComputedRef<boolean>
  error: ComputedRef<Error | null>
  refresh: () => Promise<void>
}
```

## Implementation Order

1. **Foundation** - Types and composable
2. **Core Stats** - Header, AbilityScoreBlock, CombatStatsGrid
3. **Skills & Saves** - SavingThrowsList, SkillsList
4. **Tab Panels** - FeaturesPanel, ProficienciesPanel, EquipmentPanel, SpellsPanel, LanguagesPanel
5. **Page Assembly** - Rewrite index.vue with all components
6. **Polish** - Loading states, error handling, responsive testing

## Acceptance Criteria

From issue #172:
- [ ] Character view page displays all available character data
- [ ] Layout matches D&D character sheet conventions
- [ ] Responsive design works on mobile/tablet/desktop
- [ ] Dark mode fully supported
- [ ] Loading states for async data
- [ ] Error handling for failed API calls
- [ ] Edit button links to character builder for incomplete characters
- [ ] Tests cover key display logic

## Out of Scope (Future)

- Print-friendly view (mentioned in issue as future enhancement)
- StepReview.vue updates for consistency (separate task)
- Character editing from view page
