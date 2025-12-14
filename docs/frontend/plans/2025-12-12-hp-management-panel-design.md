# HP Management Panel Design

**Date:** 2025-12-12
**Issue:** #531
**Epic:** #530 (Character Play Mode)
**Status:** Approved

---

## Overview

Extend `CombatStatsGrid.vue` to support interactive HP management in play mode. Users click the HP cell to open a modal for damage/healing, with a separate flow for temp HP.

---

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| UI Style | Modal only (no quick buttons) | Simpler, YAGNI - modal handles all cases |
| Input Format | Signed values (`+4`, `-12`) | Matches player mental model |
| Temp HP | Separate button/modal | Different concept, cleaner separation |
| Zero HP | Auto-show death saves | D&D rules clear, reduces friction |
| Calculation | Frontend (for now) | Unblocked by backend, migrate later |

---

## Component Architecture

### Files

| File | Action | Purpose |
|------|--------|---------|
| `CombatStatsGrid.vue` | Modify | Add `editable` prop, clickable HP, temp HP button |
| `HpEditModal.vue` | Create | Modal for damage/healing input |
| `TempHpModal.vue` | Create | Modal for setting temp HP |

### New Props/Emits for CombatStatsGrid

```typescript
interface Props {
  character: Character
  stats: CharacterStats
  currency?: CharacterCurrency | null
  editable?: boolean  // NEW: enables play mode
}

const emit = defineEmits<{
  'hp-change': [delta: number]      // +8 for heal, -12 for damage
  'temp-hp-set': [value: number]    // Set temp HP to this value
  'temp-hp-clear': []               // Remove all temp HP
}>()
```

---

## HP Edit Modal

### Layout

```
┌─────────────────────────────────────────┐
│  Modify Hit Points                   X  │
├─────────────────────────────────────────┤
│                                         │
│  Current: 43 / 52                       │
│  (Temp HP: 8 — absorbs damage first)    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  -12                            │    │
│  └─────────────────────────────────┘    │
│  Enter damage (-) or healing (+)        │
│                                         │
│  [Cancel]                    [Apply]    │
└─────────────────────────────────────────┘
```

### Props/Emits

```typescript
interface Props {
  open: boolean
  currentHp: number
  maxHp: number
  tempHp: number
}

const emit = defineEmits<{
  'update:open': [value: boolean]
  'apply': [delta: number]
}>()
```

### Behavior

| Input | With 8 temp HP | Result |
|-------|----------------|--------|
| `-5` | Temp absorbs all | HP: 43, Temp: 3 |
| `-12` | Temp absorbs 8, HP takes 4 | HP: 39, Temp: 0 |
| `+10` | Healing ignores temp | HP: 52 (capped), Temp: 8 |
| `-50` | Overkill floors at 0 | HP: 0, Temp: 0, death saves |

---

## Temp HP Modal

### Layout

```
┌─────────────────────────────────────────┐
│  Set Temporary Hit Points            X  │
├─────────────────────────────────────────┤
│                                         │
│  Current Temp HP: 8                     │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  12                             │    │
│  └─────────────────────────────────┘    │
│  Temp HP doesn't stack — keeps higher   │
│                                         │
│  [Clear Temp HP]             [Apply]    │
└─────────────────────────────────────────┘
```

### Props/Emits

```typescript
interface Props {
  open: boolean
  currentTempHp: number
}

const emit = defineEmits<{
  'update:open': [value: boolean]
  'apply': [value: number]
  'clear': []
}>()
```

### Behavior

- Input lower than current shows hint: "You already have 10 temp HP. This would have no effect."
- D&D rule: Temp HP doesn't stack, you keep the higher value

---

## API Integration

### Endpoint

Use existing `PATCH /characters/{id}`:

```typescript
await apiFetch(`/characters/${id}`, {
  method: 'PATCH',
  body: {
    current_hit_points: newHp,
    temp_hit_points: newTempHp
  }
})
```

### Calculation Logic

```typescript
function applyHpDelta(delta: number) {
  let newTempHp = currentTempHp.value
  let newCurrentHp = currentHp.value

  if (delta < 0) {
    // DAMAGE: temp HP absorbs first
    const damage = Math.abs(delta)
    const tempAbsorbed = Math.min(newTempHp, damage)
    newTempHp -= tempAbsorbed
    newCurrentHp -= (damage - tempAbsorbed)
    newCurrentHp = Math.max(0, newCurrentHp)  // Floor at 0
  } else {
    // HEALING: caps at max HP
    newCurrentHp = Math.min(maxHp.value, newCurrentHp + delta)
  }

  return { newCurrentHp, newTempHp }
}
```

### Optimistic Updates

1. Update UI immediately on Apply
2. Show loading state on Apply button
3. On error: revert to previous values, show toast
4. On success: close modal

---

## Testing Strategy

### Unit Tests - Component Logic

**HpEditModal.vue**
- Renders current HP state correctly
- Input accepts signed values
- Validation: rejects non-numeric input
- Shows temp HP absorption info

**TempHpModal.vue**
- Renders current temp HP
- Shows warning when input < current
- Clear button visible when temp HP > 0

### Integration Tests - UI Interactions

**CombatStatsGrid.vue**
- Opens HP modal when HP cell clicked (editable=true)
- Does NOT open modal when editable=false
- Closes modal and emits hp-change on apply
- Opens temp HP modal when "Add Temp HP" clicked
- Emits temp-hp-set when temp modal applies
- Emits temp-hp-clear when clear clicked

### HP Calculation Tests (Pure Functions)

- Damage subtracts from temp HP first
- Healing caps at max HP
- HP floors at 0
- Temp HP absorbs partial damage

---

## Future Improvements

1. **Backend HP endpoint** - Request `PATCH /characters/{id}/hp` with `{ damage?: number, healing?: number }` so backend owns D&D rules
2. **Component split** - Extract HP cell, AC cell, etc. from CombatStatsGrid into separate components (add to #442)
3. **Damage resistance** - When backend supports it, show adjusted damage in modal

---

## Acceptance Criteria

- [ ] `editable` prop added to CombatStatsGrid
- [ ] HP cell clickable when editable=true, opens HpEditModal
- [ ] HpEditModal accepts signed input (+/-), applies delta
- [ ] Temp HP absorbs damage before regular HP
- [ ] HP cannot exceed max (healing caps)
- [ ] HP cannot go below 0 (floors)
- [ ] When HP hits 0, death saves component shown
- [ ] "Add Temp HP" button visible when editable=true
- [ ] TempHpModal sets/clears temp HP
- [ ] Temp HP doesn't stack (keeps higher value)
- [ ] All tests pass (unit + integration)
- [ ] Mobile touch-friendly
- [ ] Works in dark mode
- [ ] Existing display-only behavior unchanged when editable=false
