# Issue 130: Race/Subrace Step Separation Design

## Summary

Split race and subrace selection into two distinct wizard steps in the character builder for better UX alignment with D&D 5e character creation flow.

## Current State

- `StepRace.vue` handles both race AND subrace selection in one step
- Flow: User picks base race → conditional subrace picker appears below → both saved together
- Store tracks `raceId` and `subraceId` separately
- API receives `race_id: subrace?.id ?? race.id`

## Proposed Design

### Step Flow

```
Current:  Name → Race (with inline subrace) → Class → ...
Proposed: Name → Race → Subrace (conditional) → Class → ...
```

### Conditional Step Logic

- New computed: `hasSubraces` = `selectedRace?.subraces?.length > 0`
- Subrace step inserted between Race and Class when `hasSubraces` is true
- `totalSteps` adjusts: 7 base + 1 subrace (conditional) + 1 proficiencies (conditional) + 1 spells (conditional)

### Component Changes

#### 1. StepRace.vue (modify)

- Remove all subrace selection UI
- Only display base races (`parent_race === null`)
- On race selection: save to store, auto-advance
- Back navigation: if user returns and changes race while `subraceId` exists → show confirmation modal

#### 2. StepSubrace.vue (new)

- Fetch/display subraces for `selectedRace`
- Subrace cards showing traits and ability modifiers
- Detail modal for expanded information
- On selection: save `subraceId` to store, advance to Class step

#### 3. edit.vue (modify)

- Insert Subrace step conditionally after Race step
- Adjust step numbering/mapping logic
- Handle dynamic step index calculation

#### 4. characterBuilder.ts store (modify)

- Add `hasSubraces` computed property
- Add `clearSubrace()` action
- Update `totalSteps` to include conditional subrace step

### Back Navigation Behavior

1. User on Subrace step → clicks Race in stepper → navigates back to Race step
2. User selects different race → confirmation modal appears
3. Modal text: "Changing race will clear your subrace selection. Continue?"
4. If confirmed: clear `subraceId`, save new race, subrace step updates/hides accordingly

### Step Number Mapping

Dynamic step calculation based on selections:

| Scenario | Steps |
|----------|-------|
| No subrace race, non-caster, no proficiency choices | 7 |
| Subrace race, non-caster, no proficiency choices | 8 |
| Subrace race, caster, proficiency choices | 10 |

### Files to Change

1. `app/components/character/builder/StepRace.vue` - Remove subrace UI, add confirmation modal
2. `app/components/character/builder/StepSubrace.vue` - New component
3. `app/pages/characters/[id]/edit.vue` - Conditional step insertion
4. `app/stores/characterBuilder.ts` - Add hasSubraces, clearSubrace, update totalSteps
5. `tests/components/character/builder/StepRace.test.ts` - Update tests
6. `tests/components/character/builder/StepSubrace.test.ts` - New test file

## Acceptance Criteria

- [ ] Race selection is its own wizard step
- [ ] Subrace selection appears as separate step only when applicable
- [ ] User can navigate back from subrace to change race selection
- [ ] Confirmation shown when changing race with existing subrace selection
- [ ] Selected race/subrace data properly stored in character builder store
- [ ] Tests cover races with and without subraces
- [ ] Edit mode correctly loads and positions at appropriate step
