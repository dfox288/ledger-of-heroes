# Wizard Choice Step Consolidation Design

**Issue:** #697
**Date:** 2025-12-16
**Status:** Approved

## Overview

Extract shared UI patterns from 4 wizard choice steps into reusable components. Each step retains entity-specific logic while sharing common UI structure.

## Current State

| Step | Lines | Choice Type |
|------|-------|-------------|
| `StepLanguages.vue` | 512 | Languages |
| `StepProficiencies.vue` | 524 | Skill/tool proficiencies |
| `StepSpells.vue` | 521 | Spells |
| `StepFeats.vue` | 288 | Feats |

**Total:** ~1,845 lines with ~70% duplication

## New Components

### 1. ChoiceSelectionGrid.vue

Handles the selection grid with header and count badge.

**Props:**
- `choiceId: string` - Choice identifier
- `quantity: number` - How many to select
- `selectedCount: number` - Current selection count
- `options: Array<{ slug: string; name: string }>` - Available options

**Slots:**
- `item-label` - Label text (e.g., "language(s)", "spell(s)")
- `option` - Custom option rendering with props: `{ option, selected, disabled, disabledReason, onToggle }`

**Injection:** Uses `inject('choiceSelection')` for selection state API.

### 2. ChoiceToggleButton.vue

Default toggle button for simple options (languages, proficiencies).

**Props:**
- `name: string` - Display name
- `selected: boolean` - Selection state
- `disabled: boolean` - Disabled state
- `disabledReason?: string` - Why disabled

**Slots:**
- `subtitle` - Additional info (script, prerequisites)

### 3. GrantedItemsSection.vue

Displays "already granted" items before choices.

**Props:**
- `title?: string` - Section title (default: "Already Granted")
- `groups: GrantedGroup[]` - Items grouped by source

**GrantedGroup interface:**
```typescript
interface GrantedGroup {
  label: string   // "From Race (Human)"
  color: string   // "race", "class", "background"
  items: Array<{ id: number | string; name: string }>
}
```

### 4. WizardStepChrome Components

Small utility components for consistent step structure:

| Component | Purpose |
|-----------|---------|
| `WizardStepHeader` | Title + subtitle |
| `WizardStepError` | Error alert |
| `WizardStepLoading` | Loading spinner |
| `WizardStepEmpty` | No choices needed state |
| `WizardStepContinue` | Continue button |

## Selection API (Injection Pattern)

Steps provide selection state to children via Vue's provide/inject:

```typescript
// In step component
provide('choiceSelection', {
  isOptionSelected: (choiceId: string, slug: string) => boolean,
  isOptionDisabled: (choiceId: string, slug: string) => boolean,
  getDisabledReason: (choiceId: string, slug: string) => string | null
})
```

This avoids prop drilling through ChoiceSelectionGrid to each option.

## Entity-Specific Logic (Stays in Steps)

| Step | Custom Logic |
|------|--------------|
| Languages | Filter `is_learnable`, cross-choice duplicate detection |
| Proficiencies | Group by proficiency type with icons |
| Spells | Group by subtype, fetch full Spell data for cards |
| Feats | Race prerequisite checks, source-based API fetch |

## File Structure

```
app/components/character/wizard/
├── choice/
│   ├── ChoiceSelectionGrid.vue
│   ├── ChoiceToggleButton.vue
│   └── GrantedItemsSection.vue
├── WizardStepHeader.vue      (may already exist)
├── WizardStepError.vue
├── WizardStepLoading.vue
├── WizardStepEmpty.vue
├── WizardStepContinue.vue
├── StepLanguages.vue         (refactored)
├── StepProficiencies.vue     (refactored)
├── StepSpells.vue            (refactored)
└── StepFeats.vue             (refactored)
```

## Expected Impact

- **Lines removed:** ~600 (from ~1,845 to ~1,200)
- **New component lines:** ~250
- **Net reduction:** ~350 lines
- **Maintainability:** Changes to grid/button styling apply everywhere

## Testing Strategy

1. Create tests for new shared components
2. Existing step tests should continue passing (UI unchanged)
3. Add integration test for provide/inject pattern

## Implementation Order

1. Create `ChoiceToggleButton` (simplest, no dependencies)
2. Create `GrantedItemsSection` (standalone)
3. Create `ChoiceSelectionGrid` (uses inject)
4. Create WizardStep* chrome components
5. Refactor `StepLanguages` first (simplest step)
6. Refactor remaining steps one at a time
7. Run full test suite after each step

## Acceptance Criteria

- [ ] All 4 shared components created with tests
- [ ] All 4 steps refactored to use shared components
- [ ] All existing tests pass
- [ ] UI/UX unchanged (visual regression check)
- [ ] ~350+ net lines removed
