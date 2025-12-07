# Character Builder Phase 2: Race & Class Selection - Design Document

**Date:** 2025-12-02
**Status:** Approved for Implementation
**Parent Issue:** #89
**Phase 1:** Complete (foundation, store, stepper, name step)

---

## Overview

### Goal

Implement Steps 2 (Race Selection) and 3 (Class Selection) of the character creation wizard, allowing players to choose their character's race and class with full information available via detail modals.

### User Outcome

- Players can browse available races/classes in a grid layout
- Click-to-select cards with visual feedback (selected state)
- "View Details" button opens modal with full entity information
- Subrace selection appears inline when a base race with subraces is selected
- Class selection determines whether the Spells step appears (caster detection)

---

## Architecture Decision

### Approach: Wrapper Components

Create new picker card components that wrap existing `RaceCard`/`ClassCard`:

**Rationale:**
- Reuses existing card components (DRY)
- Keeps selection behavior separate from display-only cards
- Easy to test selection logic in isolation
- Original cards remain pure "display + navigate"

---

## Component Structure

```
app/components/character/builder/
├── StepRace.vue          # Grid + search + selection logic
├── StepClass.vue         # Grid + search + selection logic
├── RacePickerCard.vue    # Wraps RaceCard, adds selection + "View Details"
├── ClassPickerCard.vue   # Wraps ClassCard, adds selection + "View Details"
├── RaceDetailModal.vue   # Full race info in modal
└── ClassDetailModal.vue  # Full class info in modal
```

---

## Data Flow

### Race Selection Flow

1. `StepRace` fetches all races via `useAsyncData`
2. User clicks card → `RacePickerCard` emits `@select` with race object
3. `StepRace` updates local `selectedRace` ref (UI highlighting)
4. If race has subraces → inline subrace selector appears
5. User clicks "View Details" → `RaceDetailModal` opens
6. User clicks "Next" → store's `selectRace(race, subrace?)` called
7. Store PATCHes API, refreshes stats, advances step

### Class Selection Flow

1. `StepClass` fetches base classes via `useAsyncData` with filter
2. User clicks card → `ClassPickerCard` emits `@select`
3. User clicks "Next" → store's `selectClass(cls)` called
4. Store PATCHes API, refreshes stats
5. `isCaster` computed updates → `totalSteps` changes if needed
6. Step advances

---

## Store Actions

### `selectRace(race: Race, subrace?: Race): Promise<void>`

```typescript
async function selectRace(race: Race, subrace?: Race): Promise<void> {
  isLoading.value = true
  error.value = null
  try {
    // API expects the actual race_id (subrace ID if subrace selected)
    await apiFetch(`/characters/${characterId.value}`, {
      method: 'PATCH',
      body: { race_id: subrace?.id ?? race.id }
    })
    raceId.value = race.id
    subraceId.value = subrace?.id ?? null
    selectedRace.value = subrace ?? race
    await refreshStats()
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Failed to save race'
    throw err
  } finally {
    isLoading.value = false
  }
}
```

### `selectClass(cls: CharacterClass): Promise<void>`

```typescript
async function selectClass(cls: CharacterClass): Promise<void> {
  isLoading.value = true
  error.value = null
  try {
    await apiFetch(`/characters/${characterId.value}`, {
      method: 'PATCH',
      body: { class_id: cls.id }
    })
    classId.value = cls.id
    selectedClass.value = cls
    await refreshStats()
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Failed to save class'
    throw err
  } finally {
    isLoading.value = false
  }
}
```

### `refreshStats(): Promise<void>`

```typescript
async function refreshStats(): Promise<void> {
  if (!characterId.value) return
  const response = await apiFetch<{ data: CharacterStats }>(
    `/characters/${characterId.value}/stats`
  )
  characterStats.value = response.data
}
```

---

## UI Specifications

### RacePickerCard

- Wraps `RaceCard` component
- Adds selected state: `ring-2 ring-race-500` border
- Removes `NuxtLink` navigation (not clickable to detail page)
- Adds "View Details" button (opens modal)
- Shows "Has X Subraces" badge when applicable
- Click anywhere on card (except View Details) → emits `@select`

### Subrace Selector

When base race with subraces is selected, shows inline below the selected card:

```
┌─────────────────────────────────────────┐
│ ✓ Dwarf                    [View Details]│
└─────────────────────────────────────────┘
  Choose a Subrace:
  ┌──────────────┐  ┌──────────────┐
  │ Hill Dwarf   │  │○ Mountain   │
  │   ● Selected │  │   Dwarf     │
  └──────────────┘  └──────────────┘
```

- Radio-style selection (only one subrace)
- Subrace must be selected before "Next" is enabled
- Each subrace shows key differentiators (ability bonuses)

### ClassPickerCard

- Wraps `ClassCard` component
- Adds selected state: `ring-2 ring-class-500` border
- Removes `NuxtLink` navigation
- Adds "View Details" button (opens modal)
- Shows spellcasting indicator prominently
- Click anywhere on card (except View Details) → emits `@select`

### Detail Modals

Both modals show comprehensive entity information:

**RaceDetailModal:**
- Race name, size, speed
- Ability score bonuses
- Racial traits list
- Languages
- Source information

**ClassDetailModal:**
- Class name, hit die
- Primary ability, saving throws
- Armor/weapon proficiencies
- Spellcasting info (if applicable)
- Level 1 features preview
- Source information

---

## API Endpoints

### Fetch Races
```
GET /races?per_page=100
```
Returns all races and subraces. Frontend filters to show base races in grid.

### Fetch Base Classes
```
GET /classes?filter=is_base_class=true&per_page=50
```
Returns only base classes (not subclasses).

### Update Character Race
```
PATCH /characters/{id}
Body: { race_id: number }
```
Note: `race_id` should be the subrace ID when a subrace is selected.

### Update Character Class
```
PATCH /characters/{id}
Body: { class_id: number }
```

### Refresh Stats
```
GET /characters/{id}/stats
```
Returns computed stats including `validation_status`.

---

## Validation

### Step 2 (Race)
- Race must be selected
- If selected race has subraces, a subrace must be selected
- "Next" button disabled until valid selection

### Step 3 (Class)
- Class must be selected
- "Next" button disabled until valid selection

---

## Testing Strategy

### Component Tests

| Component | Test Focus |
|-----------|------------|
| `RacePickerCard` | Selection emit, selected styling, View Details click |
| `ClassPickerCard` | Selection emit, selected styling, View Details click |
| `StepRace` | Race grid rendering, selection state, subrace flow, Next button state |
| `StepClass` | Class grid rendering, selection state, Next button state |
| `RaceDetailModal` | Modal open/close, content display |
| `ClassDetailModal` | Modal open/close, content display |

### Store Tests

| Action | Test Focus |
|--------|------------|
| `selectRace` | API call params, state updates, error handling |
| `selectClass` | API call params, state updates, isCaster change |
| `refreshStats` | Stats fetched and stored |

### Integration Tests

- Complete race → class flow
- Subrace selection flow
- Caster class updates totalSteps

---

## Implementation Phases

### Phase 2a: Store Actions (1 hour)
- Add `selectRace`, `selectClass`, `refreshStats` to store
- Add store tests

### Phase 2b: Picker Cards (2 hours)
- Create `RacePickerCard` with selection behavior
- Create `ClassPickerCard` with selection behavior
- Add component tests

### Phase 2c: Detail Modals (1.5 hours)
- Create `RaceDetailModal`
- Create `ClassDetailModal`
- Add modal tests

### Phase 2d: Step Components (2 hours)
- Implement `StepRace` with grid, selection, subrace handling
- Implement `StepClass` with grid, selection
- Add step component tests

### Phase 2e: Integration (0.5 hours)
- Wire up Next button to trigger store actions
- Verify step navigation works
- End-to-end flow testing

**Total Estimated: 7 hours**

---

## Out of Scope

- Search/filter within wizard steps (keep simple for v1)
- Detailed subrace cards (simple radio buttons sufficient)
- Remembering previous selections on back navigation (store already handles this)

---

## Related Documents

- Parent design: `docs/plans/2025-12-01-character-builder-frontend-design.md`
- Backend API: `importer/docs/plans/2025-11-30-character-builder-api-design.md`
- Issue: #89
