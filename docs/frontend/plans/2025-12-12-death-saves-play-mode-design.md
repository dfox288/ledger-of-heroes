# Death Saves Play Mode - Design

**Date:** 2025-12-12
**Issue:** #533
**Status:** Approved

## Overview

Extend the existing `DeathSaves.vue` component to support interactive death save tracking when in play mode.

## Design Decisions

| Aspect | Decision |
|--------|----------|
| **Interaction** | Click circles directly to toggle successes/failures |
| **Dice rolls** | Deferred - can add later as optional toggle |
| **Reset** | Button to clear all saves |
| **API** | `PATCH /characters/{id}` with `death_save_successes/failures` |
| **Visual feedback** | Badges for Stabilized (3 success) / Dead (3 fail) |

## Component Extension

**File:** `app/components/character/sheet/DeathSaves.vue`

### Props

```typescript
defineProps<{
  successes: number       // existing (0-3)
  failures: number        // existing (0-3)
  editable?: boolean      // NEW - enables clicking circles
}>()
```

### Emits

```typescript
defineEmits<{
  'update:successes': [value: number]  // 0-3
  'update:failures': [value: number]   // 0-3
  'reset': []                          // clear all
}>()
```

### Behavior

- **editable=false (default):** Works exactly as today - pure display
- **editable=true:**
  - Click empty circle → fill it (increment)
  - Click filled circle → unfill it (decrement)
  - Cursor changes to pointer on hover
  - Subtle hover state on circles
  - Reset button visible

### Visual States

```
Normal:      Successes: ●●○   Failures: ●○○

Stabilized:  Successes: ●●●   [Stabilized badge, green glow]

Dead:        Failures:  ●●●   [Dead badge, red glow]
```

## API Integration

**Endpoint:** `PATCH /api/v1/characters/{id}`

**Request body:**
```json
{
  "death_save_successes": 2,
  "death_save_failures": 1
}
```

**Verified working:** 2025-12-12

**Nitro route:** Existing `server/api/characters/[id].patch.ts` - no changes needed.

## Test Cases

```typescript
describe('DeathSaves - editable mode', () => {
  it('renders clickable circles when editable=true')
  it('clicking empty success circle emits update:successes with incremented value')
  it('clicking filled success circle emits update:successes with decremented value')
  it('clicking empty failure circle emits update:failures with incremented value')
  it('clicking filled failure circle emits update:failures with decremented value')
  it('shows "Stabilized" badge at 3 successes')
  it('shows "Dead" badge at 3 failures')
  it('reset button emits reset event')
  it('circles are not clickable when editable=false')
  it('does not show reset button when editable=false')
})
```

## Files to Modify

1. `app/components/character/sheet/DeathSaves.vue` - Extend with editable prop
2. `tests/components/character/sheet/DeathSaves.test.ts` - Add editable mode tests

## Future Enhancements (Deferred)

- Dice roll modal with d20 input
- Natural 1/20 special handling
- Sound/haptic feedback for dramatic moments
- Integration with HP (auto-show when HP=0)
