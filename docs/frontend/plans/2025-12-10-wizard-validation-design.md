# Wizard Step Validation Design

**Date:** 2025-12-10
**Issue:** [#437](https://github.com/dfox288/ledger-of-heroes/issues/437)
**Status:** Approved

## Problem

The character wizard allows users to proceed through steps without completing required choices. Five validation TODOs remain in `app/composables/useCharacterWizard.ts`:

- Line 277: feats
- Line 286: proficiencies
- Line 290: languages
- Line 294: equipment
- Line 298: spells

Additionally, the `size` and `abilities` steps return `true` without proper validation.

## Solution

Use the store's `summary.pending_choices` counts to validate step completion. When `pending_choices.{type} === 0`, all choices for that type are complete.

### Validation Approach

**Option chosen:** Block navigation (Option A)
- Next button disabled until ALL required choices for that step are complete
- Prevents incomplete character creation

**Data source:** Store summary (Approach A)
- Already available via `store.summary.pending_choices`
- No new API calls needed
- Single source of truth

## Complete Step Validation Map

| Step | Current | Proposed | Source |
|------|---------|----------|--------|
| `sourcebooks` | `true` | No change | - |
| `race` | `race !== null` | No change | Store |
| `subrace` | `subrace !== null \|\| !required` | No change | Store |
| `size` | `true` | `pending_choices.size === 0` | Summary |
| `class` | `class !== null` | No change | Store |
| `subclass` | `subclass !== null` | No change | Store |
| `background` | `background !== null` | No change | Store |
| `feats` | `true` (TODO) | `pending_choices.feats === 0` | Summary |
| `abilities` | `true` (TODO) | `pending_choices.asi === 0` | Summary |
| `proficiencies` | `true` (TODO) | `pending_choices.proficiencies === 0` | Summary |
| `languages` | `true` (TODO) | `pending_choices.languages === 0` | Summary |
| `equipment` | `true` (TODO) | `true` (keep)* | Component |
| `spells` | `true` (TODO) | `pending_choices.spells === 0` | Summary |
| `details` | `name.trim().length > 0` | No change | Store |
| `review` | `true` | No change | - |

*Equipment note: Not in `pending_choices` summary. Component validates via `allEquipmentChoicesMade`. Follow-up issue may add to backend.

## Implementation

### File Changes

**`app/composables/useCharacterWizard.ts`**

Replace TODO cases in `canProceed` computed:

```typescript
case 'size':
  if (!store.summary) return false
  return store.summary.pending_choices.size === 0

case 'feats':
  if (!store.summary) return false
  return store.summary.pending_choices.feats === 0

case 'abilities':
  if (!store.summary) return false
  return store.summary.pending_choices.asi === 0

case 'proficiencies':
  if (!store.summary) return false
  return store.summary.pending_choices.proficiencies === 0

case 'languages':
  if (!store.summary) return false
  return store.summary.pending_choices.languages === 0

case 'equipment':
  // Equipment validation handled by StepEquipment component
  // Not in pending_choices summary
  return true

case 'spells':
  if (!store.summary) return false
  return store.summary.pending_choices.spells === 0
```

### Testing

**New file:** `tests/composables/useCharacterWizard.validation.test.ts`

Test cases (6 steps × 3 scenarios = 18 tests):
- Returns `false` when summary is null (loading state)
- Returns `false` when `pending_choices.{type} > 0`
- Returns `true` when `pending_choices.{type} === 0`

## Edge Cases

1. **Loading state:** When `store.summary` is null, return `false` to prevent proceeding before data loads
2. **Zero choices:** Steps with no choices are skipped via `shouldSkip`, so `pending_choices` will be 0
3. **Equipment:** Validated locally by component; global `canProceed` returns `true`

## Acceptance Criteria

- [ ] Each step's `canProceed` returns `false` until choices complete
- [ ] Loading state blocks navigation (summary null)
- [ ] 18 new tests covering all validation scenarios
- [ ] All existing tests continue to pass
