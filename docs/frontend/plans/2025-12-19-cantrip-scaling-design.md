# Cantrip Damage Scaling Display

**Issue:** #809
**Date:** 2025-12-19
**Status:** Ready for implementation

## Overview

Display pre-calculated cantrip damage at the character's current level using the `scaled_effects` field from the backend.

## Design Decisions

| Decision | Choice |
|----------|--------|
| Display locations | Both collapsed and expanded views |
| Collapsed format | `2d10 fire • Cantrip Evocation` (damage before school) |
| Expanded location | New "Damage" row in stats grid |
| Non-damage cantrips | Show "Utility" label |
| Damage type styling | Plain text (no colored badges) |

## Implementation

### 1. Type Changes (`app/types/character.ts`)

Add `scaled_effects` to `CharacterSpellData`:

```typescript
export interface CharacterSpellData {
  // ... existing fields ...

  scaled_effects?: Array<{
    effect_type: string      // "damage"
    dice_formula: string     // "2d10"
    damage_type: string      // "Fire"
  }>
}
```

### 2. SpellCard.vue Computed Properties

```typescript
const scaledDamage = computed(() => {
  if (!props.spell.scaled_effects?.length) return null
  const dmg = props.spell.scaled_effects.find(e => e.effect_type === 'damage')
  if (!dmg) return null
  return `${dmg.dice_formula} ${dmg.damage_type.toLowerCase()}`
})

const cantripLabel = computed(() => {
  if (!isCantrip.value) return null
  return scaledDamage.value ? scaledDamage.value : 'Utility'
})
```

### 3. Collapsed View Template

Replace subtitle for cantrips:

```vue
<template v-if="isCantrip && cantripLabel">
  {{ cantripLabel }} • Cantrip {{ spellData.school }}
</template>
<template v-else>
  {{ formatSpellLevel(spellData.level) }} {{ spellData.school }}
</template>
```

### 4. Expanded View Template

Add Damage row to stats grid:

```vue
<p v-if="isCantrip && cantripLabel">
  <span class="font-medium text-gray-500 dark:text-gray-400">Damage</span>
  {{ cantripLabel }}
</p>
```

## Test Cases

1. **Damage cantrip** — Fire Bolt with scaled_effects
   - Collapsed: "2d10 fire • Cantrip Evocation"
   - Expanded: Damage row shows "2d10 fire"

2. **Utility cantrip** — Mage Hand without scaled_effects
   - Collapsed: "Utility • Cantrip Conjuration"
   - Expanded: Damage row shows "Utility"

3. **Non-cantrip spell** — Fireball (level 3)
   - Collapsed: unchanged "Level 3 Evocation"
   - Expanded: no Damage row

4. **Multiple damage types** — Uses first damage effect

## Files

- `app/types/character.ts` — Add scaled_effects type
- `app/components/character/sheet/SpellCard.vue` — Display logic
- `tests/components/character/sheet/SpellCard.test.ts` — Test cases
