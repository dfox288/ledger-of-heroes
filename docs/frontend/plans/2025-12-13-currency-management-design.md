# Currency Management Design

**Date:** 2025-12-13
**Status:** Approved
**Depends on:** Backend endpoint (see GitHub issue)

## Overview

Add currency management to the character sheet, allowing players to add, subtract, or set coin values with automatic conversion when breaking higher-denomination coins.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Input approach | Five parallel inputs | More comfortable for most players |
| Input syntax | `-5`, `+10`, `25` | Consistent with HP modal pattern |
| Conversion | Auto-convert on insufficient | Seamless, backend handles logic |
| Insufficient funds | Block with error | Strict validation, no debt allowed |
| Modal behavior | Stay open until backend confirms | Money is consequential |
| Trigger | Click CurrencyCard | Consistent with HP card pattern |
| Validation source | Backend only | Single source of truth |

## API Contract

```
PATCH /api/characters/{id}/currency

Request:
{
  "gp": "-5",      // subtract 5 gold
  "sp": "+10",     // add 10 silver
  "cp": "25"       // set copper to 25
}

Success Response (200):
{
  "data": {
    "pp": 0,
    "gp": 145,
    "ep": 0,
    "sp": 40,
    "cp": 25
  }
}

Insufficient Funds Response (422):
{
  "message": "Insufficient funds",
  "errors": {
    "currency": ["Cannot afford this transaction. Need 50 CP but only 20 CP available after conversion."]
  }
}
```

## Component Architecture

### New Components

```
app/components/character/sheet/
├── CurrencyCard.vue          # Existing - add click handler + cursor-pointer
└── CurrencyEditModal.vue     # New - modal with 5 inputs
```

### Data Flow

```
1. User clicks CurrencyCard
   └─→ Character page opens CurrencyEditModal

2. User enters values (e.g., gp: "-5", cp: "+50")
   └─→ Modal tracks local input state

3. User clicks Apply
   └─→ Modal emits 'apply' with payload { gp: "-5", cp: "+50" }
   └─→ Character page calls PATCH /api/characters/{id}/currency
   └─→ On success: close modal, refresh character stats
   └─→ On 422: keep modal open, show error toast

4. Character page refreshes stats
   └─→ CurrencyCard re-renders with new values
```

### Props & Events

```typescript
// CurrencyEditModal.vue
interface Props {
  open: boolean
  currency: CharacterCurrency  // Current values for display
}

interface Emits {
  'update:open': [value: boolean]
  'apply': [payload: CurrencyDelta]
}

interface CurrencyDelta {
  pp?: string  // "-5", "+10", or "25"
  gp?: string
  ep?: string
  sp?: string
  cp?: string
}
```

## Modal UI Design

```
┌─────────────────────────────────────────┐
│  Manage Currency                    [X] │
├─────────────────────────────────────────┤
│                                         │
│  Current: 5 PP · 150 GP · 0 EP ·       │
│           30 SP · 75 CP                 │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ PP  [________________]          │   │
│  │ GP  [________________]          │   │
│  │ EP  [________________]          │   │
│  │ SP  [________________]          │   │
│  │ CP  [________________]          │   │
│  └─────────────────────────────────┘   │
│                                         │
│  Hint: -5 (subtract), +5 (add),        │
│        or 5 (set to value)              │
│                                         │
├─────────────────────────────────────────┤
│                    [Cancel]   [Apply]   │
└─────────────────────────────────────────┘
```

### UI Details

- **Header:** "Manage Currency"
- **Current display:** Compact single line showing all 5 coin types
- **Inputs:** Vertically stacked, coin label (colored) on left, input on right
- **Input properties:** `type="text"`, `inputmode="numeric"`, placeholder=`"-5, +5, or 5"`
- **Coin colors:** PP (gray), GP (yellow), EP (gray), SP (slate), CP (orange)
- **Apply:** Disabled when all inputs empty, shows spinner during request

## Error Handling

| Scenario | HTTP Status | Frontend Behavior |
|----------|-------------|-------------------|
| Insufficient funds | 422 | Toast error with backend message, modal stays open |
| Invalid input format | 422 | Toast error, modal stays open |
| Character not found | 404 | Toast error, modal closes |
| Server error | 500 | Generic error toast, modal stays open |

### Frontend Validation (Minimal)

- At least one input has a value before enabling Apply
- Input matches pattern: optional `+`/`-`, then digits
- Empty inputs excluded from payload

### Loading State

- Apply button shows spinner and is disabled
- Cancel button remains enabled
- Inputs become readonly

## Integration

### CurrencyCard Changes

```vue
<div
  class="... cursor-pointer hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors"
  @click="emit('click')"
>
```

### Character Sheet Page

```typescript
const isCurrencyModalOpen = ref(false)

async function handleCurrencyUpdate(payload: CurrencyDelta) {
  try {
    await apiFetch(`/characters/${publicId}/currency`, {
      method: 'PATCH',
      body: payload
    })
    await refreshStats()
    isCurrencyModalOpen.value = false
    toast.add({ title: 'Currency updated', color: 'success' })
  } catch (error) {
    const message = error.data?.message || 'Failed to update currency'
    toast.add({ title: message, color: 'error' })
  }
}
```

### Nitro Proxy Route

New file: `server/api/characters/[id]/currency.patch.ts`

## Testing Strategy

### CurrencyEditModal Tests

```typescript
describe('CurrencyEditModal', () => {
  // Rendering
  it('displays current currency values')
  it('renders all 5 coin inputs with correct labels')
  it('shows hint text for input format')

  // Input parsing
  it('parses negative values as subtract (-5)')
  it('parses positive values with plus as add (+5)')
  it('parses plain numbers as set value (5)')
  it('excludes empty inputs from payload')

  // Validation
  it('disables Apply when all inputs empty')
  it('enables Apply when at least one input has value')

  // Interaction
  it('emits apply with correct payload on submit')
  it('emits update:open false on cancel')
  it('submits on Enter key when valid')
  it('clears inputs when modal opens')

  // Loading state
  it('disables Apply button when loading')
  it('shows spinner during loading')
})
```

### CurrencyCard Tests (Update)

```typescript
it('emits click when card is clicked')
it('shows hover state on mouse over')
```

## Files Changed

| File | Change |
|------|--------|
| `app/components/character/sheet/CurrencyEditModal.vue` | New |
| `app/components/character/sheet/CurrencyCard.vue` | Add click + hover |
| `app/pages/characters/[publicId]/index.vue` | Modal integration |
| `server/api/characters/[id]/currency.patch.ts` | New proxy |
| `tests/components/character/sheet/CurrencyEditModal.test.ts` | New |
| `tests/components/character/sheet/CurrencyCard.test.ts` | Update |
