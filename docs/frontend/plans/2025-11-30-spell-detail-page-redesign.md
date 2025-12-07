# Spell Detail Page Redesign

**Date:** 2025-11-30
**Status:** Design Approved
**Issue:** [#78](https://github.com/dfox288/dnd-rulebook-project/issues/78)
**Primary Use Case:** Rules Reference (deep dive into spell mechanics)

## Overview

Redesign the spell detail page (`app/pages/spells/[slug].vue`) from a generic accordion-heavy layout to a "Spell Tome" layout that surfaces all spell information prominently, optimized for rules reference during play.

### Key Changes

1. **Combat-critical info promoted** - Damage, saves, AoE move from accordions to prominent cards
2. **Scaling table** - Visual representation of spell power by slot level
3. **Classes always visible** - Grouped by base class vs subclass
4. **Minimal accordion usage** - Only for truly secondary data (data_tables, tags)

## Design Philosophy

The "Spell Tome" layout mirrors official D&D sourcebooks - dense but scannable, everything visible without clicks. For a Rules Reference use case, hiding information behind accordions is the enemy of quick lookup.

## Page Structure

```
app/pages/spells/[slug].vue
├── UiDetailBreadcrumb (existing)
├── SpellHero (NEW)
│   ├── Level/School badges (prominent)
│   ├── Ritual/Concentration badges (large, not afterthoughts)
│   ├── Spell name (h1)
│   └── Image (1/3 width on desktop)
├── SpellCastingStats (NEW)
│   ├── Casting Time, Range, Duration, Area of Effect
│   └── Components with V/S/M breakdown + material details
├── UiDetailDescriptionCard (existing - always visible)
├── SpellCombatMechanics (NEW - conditional)
│   ├── Damage display (dice + type badge)
│   ├── Save display (ability + success/fail effects)
│   └── Condition card (if spell inflicts conditions)
├── SpellScalingTable (NEW - conditional)
│   └── Table: slot level → damage/effects
├── SpellHigherLevels (NEW - conditional)
│   └── "At Higher Levels" formatted text
├── SpellClassList (NEW)
│   └── Classes grouped: base classes, then subclasses with parent
├── UAccordion (minimal)
│   ├── Data Tables (rare)
│   └── Tags (rare)
├── SpellSourceFooter (NEW)
└── UiDetailPageBottomNav (existing)
```

## New Components

### 1. SpellHero

**Purpose:** Spell-specific header replacing generic `UiDetailPageHeader`

**Props:**
```typescript
interface Props {
  spell: Spell
  imagePath: string | null
}
```

**Layout:**
- 2/3 content, 1/3 image grid (matches ItemHero pattern)
- Level badge: Uses spell-level color scale
- School badge: Uses school-specific colors
- Ritual badge: Large, prominent (🔮 Ritual)
- Concentration badge: Large, prominent (⭐ Concentration)

**Badge Colors:**
| Level | Color |
|-------|-------|
| Cantrip | gray |
| 1-2 | green |
| 3-4 | blue |
| 5-6 | purple |
| 7-8 | orange |
| 9 | red |

### 2. SpellCastingStats

**Purpose:** Core casting mechanics at a glance

**Props:**
```typescript
interface Props {
  castingTime: string
  castingTimeType: string
  range: string
  duration: string
  areaOfEffect: { type: string; size: number } | null
  components: string
  requiresVerbal: boolean
  requiresSomatic: boolean
  requiresMaterial: boolean
  materialComponents: string | null
  materialCostGp: number | null
  materialConsumed: boolean | null
}
```

**Layout:**
- 4-column grid on desktop (Casting Time, Range, Duration, Area)
- 2x2 grid on mobile
- Components row below with icons: 🗣️ V, 👋 S, 💎 M
- Material details expand below when present
- Costly materials (>= 100gp) get warning styling
- Consumed materials clearly marked

### 3. SpellCombatMechanics

**Purpose:** At-a-glance combat reference

**Conditional:** Only renders if spell has effects, saving_throws, or area_of_effect

**Props:**
```typescript
interface Props {
  effects: SpellEffect[]
  savingThrows: SpellSavingThrow[]
  areaOfEffect: { type: string; size: number } | null
}
```

**Layout:**
- Card with "⚔️ COMBAT MECHANICS" header
- 2-3 columns depending on available data:
  - 💥 Damage: dice formula + damage type badge
  - 🎯 Saving Throw: ability + success/fail effects
  - 📐 Area: size + shape
  - 😵 Condition: inline condition effects (if applicable)

**Damage Type Colors:**
| Type | Color |
|------|-------|
| Fire | red/orange |
| Cold | blue |
| Lightning | yellow |
| Acid | green |
| Poison | purple |
| Necrotic | dark green |
| Radiant | gold |
| Force | blue-purple |
| Psychic | pink |
| Thunder | gray-blue |

### 4. SpellScalingTable

**Purpose:** Visual damage/effect progression by level

**Conditional:** Only renders if `effects.length > 1` with different `min_spell_slot` values

**Props:**
```typescript
interface Props {
  effects: SpellEffect[]
  baseLevel: number // spell's minimum level
  scalingType: 'spell_slot_level' | 'character_level'
}
```

**Layout for spell slot scaling (Fireball):**
```
📈 DAMAGE BY SPELL SLOT
┌──────────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ Slot     │ 3rd  │ 4th  │ 5th  │ 6th  │ 7th  │ 8th  │ 9th  │
├──────────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤
│ Damage   │ 8d6  │ 9d6  │ 10d6 │ 11d6 │ 12d6 │ 13d6 │ 14d6 │
│ Average  │ 28   │ 31   │ 35   │ 38   │ 42   │ 45   │ 49   │
└──────────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
```

**Layout for character level scaling (Eldritch Blast):**
```
📈 BEAMS BY CHARACTER LEVEL
┌─────────────────┬───────┬────────┬─────────┬────────┐
│ Character Level │ 1-4   │ 5-10   │ 11-16   │ 17+    │
├─────────────────┼───────┼────────┼─────────┼────────┤
│ Beams           │ 1     │ 2      │ 3       │ 4      │
│ Max Damage      │ 1d10  │ 2d10   │ 3d10    │ 4d10   │
└─────────────────┴───────┴────────┴─────────┴────────┘
```

**Features:**
- Horizontal scroll on mobile with shadow indicators
- Base level column highlighted
- Average damage calculated for dice formulas
- Adapts to effect type (damage vs healing vs targets)

### 5. SpellHigherLevels

**Purpose:** Display "At Higher Levels" text when no computed effects exist

**Conditional:** Only renders if `higher_levels` exists AND scaling table not shown

**Props:**
```typescript
interface Props {
  higherLevels: string
}
```

**Layout:**
- Simple card with "📈 AT HIGHER LEVELS" header
- Formatted text content

### 6. SpellClassList

**Purpose:** Show which classes can learn the spell

**Props:**
```typescript
interface Props {
  classes: SpellClass[]
}
```

**Layout:**
```
📚 SPELL LISTS

BASE CLASSES
[Sorcerer] [Wizard]

SUBCLASSES
[Light Domain (Cleric)] [The Fiend (Warlock)] [Artillerist (Artificer)]
```

**Features:**
- Group by `is_base_class` field
- Subclasses show parent class in muted text
- Each badge links to class detail page
- Class-appropriate colors (use existing class color utilities)

### 7. SpellSourceFooter

**Purpose:** Small, always-visible source reference

**Props:**
```typescript
interface Props {
  sources: Source[]
}
```

**Layout:**
- Compact inline display: "Source: PHB p.241"
- Multiple sources comma-separated

## Composable: useSpellDetail

Create a dedicated composable (following `useItemDetail` pattern) to encapsulate:

```typescript
// app/composables/useSpellDetail.ts
export function useSpellDetail(slug: Ref<string> | string) {
  // Fetch spell data
  const { data: entity, pending, error } = useEntityDetail<Spell>({...})

  // Computed properties
  const spellLevelText = computed(() => {...})
  const hasScalingEffects = computed(() => {...})
  const scalingType = computed(() => {...})
  const baseDamage = computed(() => {...})
  const combatMechanicsVisible = computed(() => {...})
  const groupedClasses = computed(() => ({
    baseClasses: [...],
    subclasses: [...]
  }))
  const parsedAreaOfEffect = computed(() => {...})

  return {
    entity,
    pending,
    error,
    spellLevelText,
    hasScalingEffects,
    scalingType,
    baseDamage,
    combatMechanicsVisible,
    groupedClasses,
    parsedAreaOfEffect
  }
}
```

## Data Requirements

### Available Now (works today)
- All core spell fields
- Effects with `min_spell_slot` scaling (Fireball, Cure Wounds, etc.)
- Saving throws with ability and effect
- Classes with `is_base_class` and `parent_class`
- Sources

### Backend Issues Created
- [#75](https://github.com/dfox288/dnd-rulebook-project/issues/75) - Magic Missile: needs per-slot dart effects
- [#76](https://github.com/dfox288/dnd-rulebook-project/issues/76) - Scorching Ray: needs per-slot ray effects
- [#77](https://github.com/dfox288/dnd-rulebook-project/issues/77) - Eldritch Blast: needs character-level scaling effects

### Graceful Degradation
- If no effects data: show "At Higher Levels" text only
- If no scaling: don't show scaling table
- If no combat mechanics: don't show combat card
- Design handles missing data without breaking

## Mobile Considerations

| Component | Desktop | Mobile |
|-----------|---------|--------|
| SpellHero | 2/3 + 1/3 grid | Stacked (image below) |
| SpellCastingStats | 4-column | 2x2 grid |
| SpellCombatMechanics | Horizontal cards | Vertical stack |
| SpellScalingTable | Full table | Horizontal scroll |
| SpellClassList | Inline badges | Wrapped badges |

## Files to Create/Modify

### New Files
```
app/components/spell/SpellHero.vue
app/components/spell/SpellCastingStats.vue
app/components/spell/SpellCombatMechanics.vue
app/components/spell/SpellScalingTable.vue
app/components/spell/SpellHigherLevels.vue
app/components/spell/SpellClassList.vue
app/components/spell/SpellSourceFooter.vue
app/composables/useSpellDetail.ts
```

### Modified Files
```
app/pages/spells/[slug].vue (major rewrite)
```

### Test Files
```
tests/components/spell/SpellHero.test.ts
tests/components/spell/SpellCastingStats.test.ts
tests/components/spell/SpellCombatMechanics.test.ts
tests/components/spell/SpellScalingTable.test.ts
tests/components/spell/SpellClassList.test.ts
tests/composables/useSpellDetail.test.ts
tests/pages/spells/detail.test.ts (update existing)
```

## Implementation Order

1. **useSpellDetail composable** - Data layer first
2. **SpellHero** - Establishes page structure
3. **SpellCastingStats** - Core mechanics display
4. **SpellCombatMechanics** - Combat reference card
5. **SpellScalingTable** - The "killer feature"
6. **SpellClassList** - Class availability
7. **SpellHigherLevels + SpellSourceFooter** - Simple components
8. **Page integration** - Wire it all together
9. **Tests** - TDD throughout

## Success Criteria

- [ ] All spell data visible without accordion clicks
- [ ] Combat mechanics (damage, save, AoE) at-a-glance
- [ ] Scaling table shows for spells with rich effects data
- [ ] Classes grouped by base vs subclass
- [ ] Mobile-responsive (tested in browser)
- [ ] Matches existing design patterns (ItemHero, etc.)
- [ ] All tests passing
- [ ] TypeScript compiles cleanly

## Visual Reference

### Current State (Problems)
- Generic header
- Combat info buried in accordions
- Scaling shown as list of effects
- Classes in accordion

### Target State
```
┌─────────────────────────────────────────────────────────────┐
│ [3rd Level] [Evocation] [⭐ Concentration]                   │
│                                                             │
│ ████ FIREBALL ████                               [IMAGE]    │
├─────────────────────────────────────────────────────────────┤
│ ⏱️ 1 action │ 📏 150 feet │ ⏳ Instant │ 📐 20ft sphere     │
│                                                             │
│ Components: 🗣️ V  👋 S  💎 M (bat guano and sulfur)         │
├─────────────────────────────────────────────────────────────┤
│ [DESCRIPTION - full text, always visible]                   │
├─────────────────────────────────────────────────────────────┤
│ ⚔️ COMBAT MECHANICS                                         │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐          │
│ │ 💥 8d6 [Fire]│ │ 🎯 DEX/half  │ │ 📐 20ft sphere│         │
│ └──────────────┘ └──────────────┘ └──────────────┘          │
├─────────────────────────────────────────────────────────────┤
│ 📈 DAMAGE BY SPELL SLOT                                     │
│ │ 3rd │ 4th │ 5th │ 6th │ 7th │ 8th │ 9th │                 │
│ │ 8d6 │ 9d6 │10d6 │11d6 │12d6 │13d6 │14d6 │                 │
├─────────────────────────────────────────────────────────────┤
│ 📚 SPELL LISTS                                              │
│ Sorcerer • Wizard • Light Domain • The Fiend • Artillerist  │
├─────────────────────────────────────────────────────────────┤
│ Source: PHB p.241                                           │
└─────────────────────────────────────────────────────────────┘
```

## Related Issues

- Backend: #75, #76, #77 (spell effects data gaps)
