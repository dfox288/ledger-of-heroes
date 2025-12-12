# Conditions Panel Design

**Issue:** #532 - Play Mode: Conditions Panel
**Date:** 2025-12-13
**Status:** Ready for implementation

## Overview

Extend the existing `Conditions.vue` component to support adding and removing conditions when in play mode. The component currently displays conditions in a warning alert; this adds CRUD capabilities via an `editable` prop.

## Decisions Made

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Empty state UI | Keep UAlert pattern, show placeholder button when editable + empty | Minimal change, matches other play mode features |
| Exhaustion management | Inline +/- stepper buttons | Fast for frequent changes during play |
| Condition selection | Grid of clickable chips (15 conditions) | Visual, one-click, no scrolling needed |
| Remove confirmation | Only for exhaustion | Most conditions are temporary combat effects; exhaustion is significant |

## Component Architecture

### Files to Create/Modify

```
app/components/character/sheet/
├── Conditions.vue              # MODIFY - add editable prop, remove buttons, exhaustion stepper
└── AddConditionModal.vue       # CREATE - modal with condition grid, source/duration fields

server/api/characters/[id]/conditions/
├── index.post.ts               # CREATE - POST to add condition
└── [slug].delete.ts            # CREATE - DELETE to remove condition
```

### Props & Emits (Conditions.vue)

```typescript
interface Props {
  conditions?: CharacterCondition[]
  editable?: boolean                    // Enable play mode interactions
  availableConditions?: Condition[]     // 15 D&D conditions for modal
}

interface Emits {
  add: [payload: { condition: string; source?: string; duration?: string; level?: number }]
  remove: [conditionSlug: string]
  'update-level': [payload: { slug: string; level: number }]
}
```

### Data Flow

- Parent (character sheet page) passes `editable={isPlayMode}` and fetches `availableConditions` from `/api/conditions`
- Component emits events, parent calls API and refreshes character data
- Matches DeathSaves pattern - component is "dumb", parent handles persistence

## UI Layout

### When `editable=false` (unchanged)

Only shows UAlert when conditions exist, hidden otherwise.

### When `editable=true` + no conditions

```
┌─────────────────────────────────────────┐
│  [+ Add Condition]                      │
└─────────────────────────────────────────┘
```

### When `editable=true` + has conditions

```
┌─────────────────────────────────────────┐
│ ⚠️ 2 Active Conditions    [+ Add]       │
│                                         │
│  • Poisoned - 2 hours              [✕]  │
│  • Exhaustion 2 - Until rest  [−][+][✕] │
│                                         │
└─────────────────────────────────────────┘
```

### UI Elements

- **Add button** - Always visible when editable (top right or standalone when empty)
- **Remove button (✕)** - Each condition row, right side, ghost style until hover
- **Exhaustion stepper** - [−] [+] buttons only for exhaustion condition
- **Level warnings** - Exhaustion 5+: amber; Exhaustion 6: red with "Death!" badge

## Add Condition Modal

```
┌─────────────────────────────────────────────────┐
│  Add Condition                              [✕] │
├─────────────────────────────────────────────────┤
│  Select a condition:                            │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │🙈Blinded│ │💕Charmed│ │🔇Deafened│          │
│  └─────────┘ └─────────┘ └─────────┘           │
│  ... (all 15 in ~3 rows)                        │
│                                                 │
│  [When Exhaustion selected]                     │
│  Level: (1) (2) (3) (4) (5) (6)                 │
│                                                 │
│  Source (optional):  [________________]         │
│  Duration (optional): [________________]        │
├─────────────────────────────────────────────────┤
│                     [Cancel]    [Add Condition] │
└─────────────────────────────────────────────────┘
```

**Behavior:**
- Condition chips use selected state styling
- Only one condition selectable at a time
- Exhaustion level picker appears conditionally (default level 1)
- "Add Condition" disabled until condition selected
- Modal closes on successful add

## Exhaustion Removal Confirm

```
┌─────────────────────────────────────────┐
│  Remove Exhaustion?                     │
├─────────────────────────────────────────┤
│  This will remove all exhaustion levels │
│  from the character.                    │
├─────────────────────────────────────────┤
│              [Cancel]  [Remove]         │
└─────────────────────────────────────────┘
```

Only shown when clicking ✕ on exhaustion, not when decrementing to 0 via stepper.

## Error Handling

- API failures show toast: "Failed to add/remove condition"
- No optimistic UI - wait for API confirmation
- Loading state: Disable buttons while API call in flight

**Edge cases:**
- Adding existing condition → API upserts (updates source/duration/level)
- Decrementing exhaustion from 1 → Removes entirely (with confirm)
- Level 6 exhaustion → Show warning badge but allow it

## Testing Strategy

### Conditions.vue Unit Tests

**Display mode (existing):** Already covered

**Editable mode (new):**
- Shows "Add Condition" button when editable + empty
- Shows "Add Condition" button when editable + has conditions
- Shows remove button on each condition when editable
- Clicking remove emits 'remove' with slug
- Shows +/- stepper only on exhaustion condition
- Clicking + on exhaustion emits 'update-level' with incremented value
- Clicking - on exhaustion emits 'update-level' with decremented value
- Does not show interactive elements when editable=false

### AddConditionModal.vue Unit Tests

- Renders all 15 condition chips
- Selecting condition highlights it
- Shows level picker only when exhaustion selected
- Disables "Add" button until condition selected
- Emits 'add' with correct payload on submit
- Source and duration fields are optional

### Notes

Modal interaction tests may be flaky due to Vue test-utils limitations with portals. Focus unit tests on emit/logic layer; defer full integration to E2E (Issue #512).

## API Endpoints

Already exist in backend:

```
GET  /api/v1/lookups/conditions                    # 15 D&D conditions
POST /api/v1/characters/{id}/conditions            # Add/upsert condition
DELETE /api/v1/characters/{id}/conditions/{slug}   # Remove condition
```

Nitro routes to create:
- `server/api/characters/[id]/conditions/index.post.ts`
- `server/api/characters/[id]/conditions/[slug].delete.ts`
