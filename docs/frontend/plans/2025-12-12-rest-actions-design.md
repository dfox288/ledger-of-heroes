# Rest Actions Design

**Issue:** #534
**Date:** 2025-12-12
**Status:** Approved

## Overview

Add short rest and long rest functionality to the character sheet play mode, integrated into the existing HitDice component.

## Design Decisions

### Simplicity Over Automation
- Hit dice spending just marks dice as spent - player rolls physical dice
- Player manually adds HP via existing HP modal
- No HP calculations or roll automation in frontend

### Merged Component
- Rest buttons live inside HitDice panel (not separate component)
- Logical grouping: hit dice + actions that affect them
- Buttons only visible in play mode (`editable=true`)

### Confirmation UX
- **Short Rest:** Single click → API → toast
- **Long Rest:** Click → confirmation modal → API → toast

## Components

### HitDice.vue (Extended)

**New Props:**
```typescript
interface Props {
  hitDice: { die: string, total: number, current: number }[]
  editable?: boolean      // NEW: Enable play mode interactions
  characterId?: number    // NEW: Required for API calls when editable
}
```

**New Emits:**
```typescript
emit('spend', { dieType: string })  // When a die is spent
emit('short-rest')                   // When short rest completes
emit('long-rest')                    // When long rest completes
```

**Visual Layout:**
```
┌─────────────────────────────────────┐
│  Hit Dice                           │
│                                     │
│  d8: ●●●○○  (3/5)                  │
│  d6: ●○     (1/2)                  │
│                                     │
│  ─────────────────────────────────  │  ← divider (play mode only)
│  [☀️ Short Rest]  [🌙 Long Rest]   │  ← buttons (play mode only)
└─────────────────────────────────────┘
```

**Icon:** `game-icons:perspective-dice-six` (consistent for all die types)

**Behavior:**
- Filled dice clickable when `editable` → calls spend API
- Empty dice not clickable (can't un-spend)
- Rest buttons trigger API calls

### LongRestConfirmModal.vue (New)

Simple confirmation dialog:
```
┌─────────────────────────────────────┐
│  Take a Long Rest?                  │
│                                     │
│  This will:                         │
│  • Restore HP to maximum            │
│  • Reset all spell slots            │
│  • Recover half your hit dice       │
│  • Clear death saves                │
│                                     │
│  [Cancel]              [Take Rest]  │
└─────────────────────────────────────┘
```

## API Integration

### Backend Endpoints (All Exist)

| Endpoint | Method | Response |
|----------|--------|----------|
| `/characters/{id}/hit-dice/spend` | POST | `{hit_dice, total}` |
| `/characters/{id}/short-rest` | POST | `{pact_magic_reset, features_reset[]}` |
| `/characters/{id}/long-rest` | POST | `{hp_restored, hit_dice_recovered, spell_slots_reset, death_saves_cleared, features_reset[]}` |

### Nitro Routes to Create

```
server/api/characters/[id]/
├── hit-dice/
│   └── spend.post.ts    # POST body: {die_type, quantity}
├── short-rest.post.ts
└── long-rest.post.ts
```

## Character Page Integration

```vue
<CharacterSheetHitDice
  v-if="hitDice.length"
  :hit-dice="hitDice"
  :editable="isPlayMode"
  :character-id="character.id"
  @spend="handleHitDiceSpend"
  @short-rest="handleShortRest"
  @long-rest="handleLongRest"
/>
```

Event handlers will:
1. Call the appropriate API
2. Show toast with result
3. Call `refresh()` to update character data

## Test Plan

### Unit Tests
- HitDice.vue: renders dice with new icon
- HitDice.vue: clickable when editable, not clickable when not
- HitDice.vue: shows rest buttons only when editable
- HitDice.vue: emits events on click
- LongRestConfirmModal.vue: renders, emits on confirm/cancel

### Integration Tests (MSW)
- Spend hit die → API called → state updates
- Short rest → API called → toast shown
- Long rest → confirmation → API called → toast with summary

## Implementation Order

1. Create Nitro API routes (3 files)
2. Extend HitDice.vue (add props, icon, click handlers, rest buttons)
3. Create LongRestConfirmModal.vue
4. Integrate into character page
5. Write tests

## Out of Scope

- HP roll automation (player uses physical dice)
- Features reset display (future enhancement)
- Combat state blocking (future enhancement)
