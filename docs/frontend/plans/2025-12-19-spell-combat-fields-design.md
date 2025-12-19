# Spell Combat Fields Display

**Issue:** #808
**Date:** 2025-12-19
**Status:** Ready for implementation

## Overview

Display spell combat fields (saving throw, attack type, damage types) in the SpellCard expanded view.

## Design Decisions

| Decision | Choice |
|----------|--------|
| Display location | Stats grid in expanded view |
| Fields to show | All three: attack type, saving throw, damage types |
| Attack/Save format | Single "Mechanic" row (mutually exclusive) |
| Damage format | Comma-separated text |
| Cantrip handling | Use scaled_effects, leveled spells use damage_types |

## Implementation

### 1. Type Changes (`app/types/character.ts`)

Add combat fields to `CharacterSpellData`:

```typescript
export interface CharacterSpellData {
  // ... existing fields ...

  // Combat fields (Issue #808)
  saving_throw?: string      // "DEX", "WIS", etc.
  attack_type?: string       // "melee", "ranged", "none"
  damage_types?: string[]    // ["Fire", "Cold"]
}
```

### 2. SpellCard.vue Computed Properties

```typescript
// Combat mechanic: "Ranged Attack", "DEX Save", or null
const combatMechanic = computed(() => {
  if (!spellData.value) return null
  if (spellData.value.attack_type && spellData.value.attack_type !== 'none') {
    const type = spellData.value.attack_type
    return `${type.charAt(0).toUpperCase() + type.slice(1)} Attack`
  }
  if (spellData.value.saving_throw) {
    return `${spellData.value.saving_throw} Save`
  }
  return null
})

// Unified damage display: cantrips use scaled_effects, leveled spells use damage_types
const damageDisplay = computed(() => {
  if (!spellData.value) return null

  // Cantrips: use scaled_effects (e.g., "2d10 fire" or "Utility")
  if (isCantrip.value) {
    return cantripLabel.value
  }

  // Leveled spells: use damage_types (e.g., "Fire, Cold")
  if (spellData.value.damage_types?.length) {
    return spellData.value.damage_types.join(', ')
  }

  return null
})
```

### 3. Template Changes

Add to stats grid (expanded view):

```vue
<!-- Combat Mechanic (Issue #808) -->
<div v-if="combatMechanic">
  <span class="text-gray-500 dark:text-gray-400">Mechanic</span>
  <p class="font-medium">{{ combatMechanic }}</p>
</div>

<!-- Damage (unified: cantrip scaled or leveled spell types) -->
<div v-if="damageDisplay">
  <span class="text-gray-500 dark:text-gray-400">Damage</span>
  <p class="font-medium">{{ damageDisplay }}</p>
</div>
```

## Test Cases

1. **Attack spell** — `attack_type: "ranged"` → "Ranged Attack"
2. **Save spell** — `saving_throw: "DEX"` → "DEX Save"
3. **Multiple damage types** — `damage_types: ["Acid", "Cold", "Fire"]` → "Acid, Cold, Fire"
4. **Utility cantrip** — No combat data → "Utility"
5. **Cantrip with scaled damage** — Uses `scaled_effects` → "2d10 fire"
6. **Melee attack** — `attack_type: "melee"` → "Melee Attack"

## Files

- `app/types/character.ts` — Add combat fields to CharacterSpellData
- `app/components/character/sheet/SpellCard.vue` — Computed properties + template
- `tests/components/character/sheet/SpellCard.test.ts` — Test cases
