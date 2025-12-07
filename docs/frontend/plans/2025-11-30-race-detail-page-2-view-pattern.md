# Race Detail Page 2-View Pattern

**GitHub Issue:** [#2 - Apply 3-view pattern to Race detail page](https://github.com/dfox288/dnd-rulebook-project/issues/2)
**Created:** 2025-11-30
**Status:** Planning
**Last Updated:** 2025-11-30 (API Audit Complete)

---

## API Audit Summary (2025-11-30)

Full audit of the Race API resource completed. Key findings:

### Data Structure Discoveries

| Field | Structure | Notes |
|-------|-----------|-------|
| `traits` | Array with `category` field | Categories: `description`, `species`, `subspecies`, `null` |
| `modifiers` | Ability scores AND damage resistances | `modifier_category`: `ability_score`, `damage_resistance` |
| `senses` | Structured array | `{type, name, range, is_limited, notes}` |
| `spells` | With level requirements | `level_requirement`, `usage_limit` (e.g., "1/long rest") |
| `languages` | Rich nested data | Includes `script`, `typical_speakers` |
| `conditions` | With effect type | `effect_type`: `advantage`, `immunity`, `resistance` |
| `inherited_data` | **Subraces only** | Contains parent's traits, modifiers, proficiencies, languages, conditions, senses |

### Trait Categories (for filtering)

```
description  → Lore text (Description, Names, History)
species      → Core racial traits (Fey Ancestry, Darkvision, Trance)
subspecies   → Subrace-specific traits (Elf Weapon Training, Cantrip)
null         → Mechanical traits (Age, Alignment, Size, Speed, Languages)
```

### Sample Data Verified

| Race | Type | ASIs | Senses | Spells | Conditions |
|------|------|------|--------|--------|------------|
| Elf | Base | DEX +2 | None (elves don't have darkvision by default) | None | Charmed (advantage) |
| High Elf | Subrace | INT +1 (+ inherited DEX +2) | None | None | Inherited |
| Dwarf | Base | CON +2 | Darkvision 60ft | None | Poisoned (advantage) |
| Tiefling | Base | CHA +2, INT +1 | Darkvision 60ft | 3 spells (leveled) | None |

---

## D&D Nerd Analysis: Ideal Race Detail Page

### What a D&D Player Needs from a Race Page

When choosing a race, players care about:

1. **Quick Decision Data** (at-a-glance)
   - Size & Speed (combat positioning)
   - Ability Score Increases (build optimization)
   - Senses (darkvision = dungeon utility)
   - Damage Resistances (defensive bonuses)

2. **Character Identity** (roleplay & story)
   - Description/Lore (who are these people?)
   - Languages (communication & secrets)
   - Subraces (regional/cultural variants)

3. **Mechanical Benefits** (the crunch)
   - Racial Traits (features, abilities)
   - Proficiencies (weapons, armor, tools, skills)
   - Spells (innate magic like Drow, Tiefling)
   - Condition Advantages/Immunities (defensive bonuses)

---

## Class 3-View Pattern Analysis

| View | Purpose | Races Equivalent? |
|------|---------|-------------------|
| **Overview** | Quick combat stats, resources, subclass gallery | ✅ **Yes** - Quick stats, ASIs, subrace gallery |
| **Journey** | Level 1-20 progression timeline | ❌ **No** - Races don't progress by level |
| **Reference** | Complete data dump, all features expanded | ✅ **Yes** - All traits, full descriptions |

### Conclusion: Races Need a **2-View Pattern**

Classes have a *progression* structure (level 1-20), while Races have a *static* structure (all traits available at character creation). This fundamental difference means the "Journey" view doesn't make sense for races - you don't "progress" through racial traits.

**Exception:** Tiefling racial spells unlock at levels 3 and 5, but this is handled in the spell display, not a full journey view.

---

## Reusable Components from Classes

| Component | Classes Usage | Races Usage |
|-----------|---------------|-------------|
| `DetailHeader` | Name, badges, parent link | ✅ Same pattern (subrace of parent) |
| `ViewNavigation` | Overview/Journey/Reference tabs | ✅ Overview/Reference tabs (2 views) |
| `SubclassCards` / `SubraceCards` | Gallery of subclasses | ✅ Gallery of subraces |
| `QuickStatsCard` | HP, saving throws, armor/weapons | ✅ Size, Speed, Senses |
| `DescriptionCard` | Class lore | ✅ Race description (from `category: description` traits) |
| `AccordionTraitsList` | Class features | ✅ Racial traits (filter by `category: species/subspecies`) |
| `AccordionBulletList` | Proficiencies, equipment | ✅ Proficiencies |
| `AccordionBadgeList` | - | ✅ Languages, Spells |
| `AccordionConditions` | - | ✅ Condition advantages/immunities |
| `SourceDisplay` | Source books | ✅ Source books |
| `ProgressionTable` | Level 1-20 table | ❌ Not applicable |
| `Timeline/LevelNode` | Journey view | ❌ Not applicable |
| `CombatBasicsGrid` | HP, saves, proficiencies | ⚠️ Could adapt for races |
| `ResourcesCard` | Ki, Rage, etc. | ❌ Not applicable |

---

## Race-Specific Components Needed

| Component | Purpose |
|-----------|---------|
| `RaceDetailHeader` | Shared header with size badge, parent race link |
| `RaceViewNavigation` | 2-tab navigation (Overview / Reference) |
| `RaceOverviewAbilityScores` | Prominent ASI display (the most important info!) |
| `RaceOverviewResistances` | Damage resistances from modifiers |
| `RaceOverviewSenses` | Darkvision/Blindsight display with range |
| `RaceOverviewTraitsPreview` | Top 3-5 species traits with "see all" link |
| `RaceOverviewSpells` | Innate spellcasting with level requirements |
| `RaceSubraceCards` | Gallery cards for subraces |

---

## Proposed Race 2-View Architecture

### Overview View (`/races/[slug]`)

```
┌─────────────────────────────────────────────────┐
│ [Breadcrumb: Races > Elf > High Elf]            │
├─────────────────────────────────────────────────┤
│ HIGH ELF                        [Medium] [Subrace]│
│ Subrace of Elf                                  │
├─────────────────────────────────────────────────┤
│ ┌───────────────────────────┐  ┌──────────────┐ │
│ │ ABILITY SCORES            │  │ [Image]      │ │
│ │ ┌─────┐ ┌─────┐           │  │              │ │
│ │ │DEX  │ │INT  │           │  │              │ │
│ │ │ +2  │ │ +1  │           │  │              │ │
│ │ └─────┘ └─────┘           │  │              │ │
│ ├───────────────────────────┤  │              │ │
│ │ Speed: 30 ft.             │  └──────────────┘ │
│ │ Darkvision: 60 ft.        │                   │
│ │ Fire Resistance           │                   │
│ └───────────────────────────┘                   │
├─────────────────────────────────────────────────┤
│ [OVERVIEW]  [REFERENCE]                         │
├─────────────────────────────────────────────────┤
│ ┌─── Description ───────────────────────────┐   │
│ │ Elves are a magical people of otherworldly│   │
│ │ grace, living in places of ethereal beauty│   │
│ └───────────────────────────────────────────┘   │
├─────────────────────────────────────────────────┤
│ KEY RACIAL TRAITS                               │
│ ┌───────────────────────────────────────────┐   │
│ │ Fey Ancestry                              │   │
│ │ Advantage on saves vs. being charmed...   │   │
│ ├───────────────────────────────────────────┤   │
│ │ Trance                                    │   │
│ │ Elves meditate 4 hours instead of sleep...│   │
│ └───────────────────────────────────────────┘   │
│ [View all 5 traits in Reference →]              │
├─────────────────────────────────────────────────┤
│ INNATE SPELLCASTING (Tiefling example)          │
│ ┌─────────────────────────────────────────┐     │
│ │ Thaumaturgy (cantrip)                   │     │
│ │ Hellish Rebuke (3rd level, 1/long rest) │     │
│ │ Darkness (5th level, 1/long rest)       │     │
│ └─────────────────────────────────────────┘     │
├─────────────────────────────────────────────────┤
│ LANGUAGES                                       │
│ [Common] [Elvish] [+1 of your choice]          │
├─────────────────────────────────────────────────┤
│ [Accordion: Proficiencies]                      │
│   Perception, Rapier, Shortsword, Hand Crossbow │
├─────────────────────────────────────────────────┤
│ SUBRACES (if base race)                        │
│ ┌───────────┐ ┌───────────┐ ┌───────────┐      │
│ │  High Elf │ │  Wood Elf │ │   Drow    │      │
│ │  INT +1   │ │  WIS +1   │ │  CHA +1   │      │
│ └───────────┘ └───────────┘ └───────────┘      │
└─────────────────────────────────────────────────┘
```

### Reference View (`/races/[slug]/reference`)

```
┌─────────────────────────────────────────────────┐
│ [Same header + view navigation]                 │
│ [OVERVIEW]  [REFERENCE] ← active                │
├─────────────────────────────────────────────────┤
│ ALL RACIAL TRAITS                               │
│ ───────── Species Traits ─────────              │
│ Keen Senses                                     │
│   You have proficiency in the Perception skill. │
│ ────────────────────────────────────────────────│
│ Fey Ancestry                                    │
│   You have advantage on saving throws against   │
│   being charmed, and magic can't put you to     │
│   sleep.                                        │
│ ────────────────────────────────────────────────│
│ Trance                                          │
│   Elves don't need to sleep. Instead they...   │
│ ───────── Subrace Traits (if subrace) ─────────│
│ Elf Weapon Training                             │
│   You have proficiency with longsword...        │
│ ────────────────────────────────────────────────│
│ Cantrip                                         │
│   You know one cantrip from the wizard list...  │
├─────────────────────────────────────────────────┤
│ [Accordion: Proficiencies]                      │
│ [Accordion: Languages]                          │
│ [Accordion: Spells (if any)]                    │
│ [Accordion: Condition Advantages]               │
│ [Accordion: Lore & Description]                 │
│ [Accordion: Source]                             │
└─────────────────────────────────────────────────┘
```

---

## Implementation Checklist

### New Files to Create

- [ ] `app/pages/races/[slug]/index.vue` (Overview)
- [ ] `app/pages/races/[slug]/reference.vue` (Reference)
- [ ] `app/composables/useRaceDetail.ts` (shared data like `useClassDetail.ts`)
- [ ] `app/components/race/DetailHeader.vue`
- [ ] `app/components/race/ViewNavigation.vue`
- [ ] `app/components/race/overview/AbilityScoresCard.vue`
- [ ] `app/components/race/overview/ResistancesDisplay.vue`
- [ ] `app/components/race/overview/SensesDisplay.vue`
- [ ] `app/components/race/overview/TraitsPreview.vue`
- [ ] `app/components/race/overview/SpellsCard.vue` (for Tiefling-style innate spellcasting)

### Files to Refactor

- [ ] Move current `app/pages/races/[slug].vue` → `[slug]/index.vue`
- [ ] Rename `UiClassSubclassCards` → `UiEntitySubCards` (generic for races/classes)

### Can Reuse As-Is

- `UiDetailPageHeader`
- `UiDetailQuickStatsCard`
- `UiDetailDescriptionCard`
- `UiDetailEntityImage`
- `UiDetailBreadcrumb`
- `UiAccordionTraitsList`
- `UiAccordionBadgeList`
- `UiAccordionBulletList`
- `UiAccordionConditions`
- `UiSourceDisplay`
- `UiModifiersDisplay`

---

## Key Differences from Classes

| Aspect | Classes | Races |
|--------|---------|-------|
| Views | 3 (Overview/Journey/Reference) | 2 (Overview/Reference) |
| Progression | Level 1-20 | Static (creation only)* |
| Primary stat | Hit Die + Saving Throws | Ability Score Increases |
| Resources | Ki, Rage, Spell Slots | None |
| Spellcasting | Class feature (spell slots) | Innate (usage limits, level requirements) |
| Subentities | Subclasses (choose at subclass level) | Subraces (choose at character creation) |
| Senses | N/A | Darkvision, Blindsight, Tremorsense, etc. |
| Damage Resistances | Rare (subclass features) | Common (Tiefling fire, Dwarf poison) |
| Inheritance | Subclass gets parent features | Subrace gets `inherited_data` from parent |

*Exception: Some racial spells unlock at levels 3 and 5 (e.g., Tiefling Infernal Legacy)

---

## Implementation Order

### Phase 1: Foundation
- [ ] Create `useRaceDetail.ts` composable
  - Fetch race data with `useEntityDetail`
  - Computed properties for trait filtering by category
  - Handle `inherited_data` for subraces
  - Extract ability scores from modifiers
  - Extract damage resistances from modifiers
- [ ] Create `RaceDetailHeader.vue` (adapt from `ClassDetailHeader`)
- [ ] Create `RaceViewNavigation.vue` (2 tabs instead of 3)

### Phase 2: Overview View
- [ ] Convert current `[slug].vue` to `[slug]/index.vue`
- [ ] Create `AbilityScoresCard.vue` - Large, prominent ASI display
- [ ] Create `SensesDisplay.vue` - Darkvision with range
- [ ] Create `ResistancesDisplay.vue` - Damage resistances
- [ ] Create `TraitsPreview.vue` - Top 3 species traits with link to Reference
- [ ] Create `SpellsCard.vue` - Innate spellcasting with level requirements
- [ ] Add subrace gallery for base races

### Phase 3: Reference View
- [ ] Create `[slug]/reference.vue`
- [ ] Full traits list grouped by category (species, subspecies)
- [ ] All accordion sections (proficiencies, languages, spells, conditions, lore, source)
- [ ] Handle inherited data display for subraces

### Phase 4: Polish & Testing
- [ ] Ensure subrace → parent race navigation works
- [ ] Test all race types:
  - Base race with subraces (Elf, Dwarf, Halfling)
  - Subrace with inherited data (High Elf, Hill Dwarf)
  - Race with spells (Tiefling, Drow)
  - Race with damage resistance (Tiefling fire, Dwarf poison)
  - Race without subraces (Dragonborn, Human)
- [ ] Mobile responsiveness
- [ ] Write component tests (TDD)
- [ ] Write E2E tests for navigation

---

## API Data Reference

### RaceResource (from API audit)

```typescript
RaceResource: {
  id: number;
  slug: string;
  name: string;
  size?: SizeResource;
  speed: number;
  is_subrace: boolean;  // Note: API returns boolean, not string
  traits?: TraitResource[];
  modifiers?: ModifierResource[];  // Includes ASIs AND damage resistances
  sources?: EntitySourceResource[];
  parent_race?: RaceResource;  // Only for subraces
  subraces?: RaceResource[];   // Only for base races
  proficiencies?: ProficiencyResource[];
  languages?: EntityLanguageResource[];
  conditions?: EntityConditionResource[];
  spells?: EntitySpellResource[];  // With level_requirement & usage_limit
  senses?: EntitySenseResource[];
  tags?: TagResource[];
  inherited_data?: {  // Only for subraces
    traits: TraitResource[];
    modifiers: ModifierResource[];
    proficiencies: ProficiencyResource[];
    languages: EntityLanguageResource[];
    conditions: EntityConditionResource[];
    senses: EntitySenseResource[];
  };
}
```

### TraitResource Categories

```typescript
TraitResource: {
  id: number;
  name: string;
  category: 'description' | 'species' | 'subspecies' | null;
  description: string;
  sort_order: number;
  data_tables: DataTableResource[];
}
```

### ModifierResource (for ASIs and resistances)

```typescript
ModifierResource: {
  id: number;
  modifier_category: 'ability_score' | 'damage_resistance';
  ability_score?: AbilityScoreResource;  // For ability_score category
  damage_type?: DamageTypeResource;       // For damage_resistance category
  value: string;  // "+2" for ASI, "resistance" for damage
  condition: string | null;
  is_choice: boolean;
}
```

### EntitySpellResource (for innate spellcasting)

```typescript
EntitySpellResource: {
  id: number;
  spell: SpellResource;
  ability_score: AbilityScoreResource;
  level_requirement: number | null;  // e.g., 3, 5
  usage_limit: string | null;        // e.g., "1/long rest"
  is_cantrip: boolean;
}
```

### EntitySenseResource

```typescript
EntitySenseResource: {
  type: string;        // "darkvision", "blindsight", "tremorsense"
  name: string;        // "Darkvision"
  range: number;       // 60, 120
  is_limited: boolean;
  notes: string | null;
}
```

---

## Notes

- The "Journey" view is intentionally omitted because races don't have level progression
- Subraces should display BOTH their own traits AND inherited parent traits (use `inherited_data`)
- Racial spells with `level_requirement` should show "(unlocked at level X)"
- Consider linking racial spells to the spells detail page
- Damage resistances come from `modifiers` where `modifier_category: 'damage_resistance'`
- Trait categories can be used to filter:
  - `species` = Core racial abilities (show in Overview preview)
  - `subspecies` = Subrace-specific (show under subrace section)
  - `description` = Lore text (collapse in accordion)
  - `null` = Mechanical flavor (Age, Size, Speed, Languages descriptions)

---

## Related Issues

- [#39 - Display race senses on detail page](https://github.com/dfox288/dnd-rulebook-project/issues/39)
- [#37 - Display monster senses on detail page](https://github.com/dfox288/dnd-rulebook-project/issues/37) (same component can be reused)
