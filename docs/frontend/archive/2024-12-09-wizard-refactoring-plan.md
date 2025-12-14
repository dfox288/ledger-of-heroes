# Wizard Components Refactoring Plan

**Created:** 2024-12-09
**Updated:** 2024-12-09 (refined via brainstorming session)
**Status:** Ready for Implementation
**Estimated Impact:** ~2,000 lines of code reduction (~30% of wizard codebase)

## Overview

This plan addresses significant code duplication identified in the character wizard components. After brainstorming, we've refined the scope to focus on the highest-value abstractions while keeping edge cases in their respective components.

## Design Decisions

### Key Principles (from brainstorming)

1. **Components handle data fetching** - Composables accept entities/choices as Refs, not fetch functions. Avoids Nuxt SSR hydration issues with `useAsyncData`.

2. **Composables call store actions, components handle navigation** - `confirmSelection()` calls the store action; component calls `nextStep()` after.

3. **YAGNI on edge cases** - StepRace's change confirmation modal and StepSubrace's nullable selection stay in their components.

4. **Built-in cross-choice validation** - Choice composable detects conflicts across choices and already-granted items.

---

## Phase 1: `useWizardEntitySelection` (#420)

### Purpose

Consolidate entity selection pattern for StepRace, StepClass, StepBackground.

**Not used by:** StepSubrace (nullable selection, fetches full detail - too different)

### Interface

```typescript
// app/composables/useWizardEntitySelection.ts

interface EntitySelectionConfig<T> {
  /** Store action to call when confirming selection */
  storeAction: (entity: T) => Promise<void>

  /** Existing selection from store (for initialization) */
  existingSelection?: ComputedRef<T | null>

  /** Fields to search (defaults to ['name']) */
  searchableFields?: Array<keyof T>
}

interface EntitySelectionReturn<T> {
  // State
  localSelected: Ref<T | null>
  searchQuery: Ref<string>
  filtered: ComputedRef<T[]>

  // Validation
  canProceed: ComputedRef<boolean>

  // Actions
  handleSelect: (entity: T) => void
  confirmSelection: () => Promise<void>

  // Detail modal (via useDetailModal)
  detailModal: {
    open: Ref<boolean>
    item: Ref<T | null>
    show: (entity: T) => void
    close: () => void
  }
}

function useWizardEntitySelection<T extends { id: number; name: string }>(
  entities: Ref<T[] | null | undefined>,
  config: EntitySelectionConfig<T>
): EntitySelectionReturn<T>
```

### Usage Example (StepClass)

```vue
<script setup lang="ts">
const store = useCharacterWizardStore()
const { selections, isLoading, sourceFilterString } = storeToRefs(store)
const { nextStep } = useCharacterWizard()
const { apiFetch } = useApi()
const toast = useToast()

// Component handles data fetching
const { data: classes, pending } = await useAsyncData(
  `builder-classes-${sourceFilterString.value}`,
  () => apiFetch<{ data: CharacterClass[] }>(`/classes?filter=...`),
  { transform: (r) => r.data, watch: [sourceFilterString] }
)

// Composable handles selection logic
const {
  localSelected,
  searchQuery,
  filtered,
  canProceed,
  handleSelect,
  confirmSelection,
  detailModal
} = useWizardEntitySelection(classes, {
  storeAction: (cls) => store.selectClass(cls),
  existingSelection: computed(() => selections.value.class)
})

// Component handles navigation
async function handleConfirm() {
  try {
    await confirmSelection()
    nextStep()
  } catch (err) {
    wizardErrors.saveFailed(err, toast)
  }
}
</script>
```

### Estimated Reduction

- StepRace: 246 → ~100 lines (60% reduction)
- StepClass: 189 → ~90 lines (52% reduction)
- StepBackground: 178 → ~80 lines (55% reduction)
- **Total: ~340 lines saved**

---

## Phase 2: `useWizardChoiceSelection` (#421)

### Purpose

Consolidate choice selection pattern for StepProficiencies, StepLanguages, StepSpells, StepFeats.

**Not used by:** StepEquipment (single-select mode, different patterns)

### Interface

```typescript
// app/composables/useWizardChoiceSelection.ts

interface DisplayOption {
  id: string
  name: string
  description?: string
}

interface ChoiceSelectionConfig {
  /** Function to resolve a choice (from useUnifiedChoices) */
  resolveChoice: (choiceId: string, payload: { selected: string[] }) => Promise<void>

  /** IDs already granted (e.g., racial languages) - disabled in selection */
  alreadyGrantedIds?: ComputedRef<Set<string>>
}

interface ChoiceSelectionReturn {
  // State
  localSelections: Ref<Map<string, Set<string>>>
  isSaving: Ref<boolean>

  // Selection helpers
  getSelectedCount: (choiceId: string) => number
  isOptionSelected: (choiceId: string, optionId: string) => boolean
  isOptionDisabled: (choiceId: string, optionId: string) => boolean
  getDisabledReason: (choiceId: string, optionId: string) => string | null

  // Validation
  allComplete: ComputedRef<boolean>

  // Actions
  handleToggle: (choice: PendingChoice, optionId: string) => void
  saveAllChoices: () => Promise<void>

  // Dynamic options fetching
  fetchOptionsIfNeeded: (choice: PendingChoice) => Promise<void>
  getDisplayOptions: (choice: PendingChoice) => DisplayOption[]
  isOptionsLoading: (choice: PendingChoice) => boolean
}

function useWizardChoiceSelection(
  choices: ComputedRef<PendingChoice[]>,
  config: ChoiceSelectionConfig
): ChoiceSelectionReturn
```

### Cross-Choice Validation

The composable automatically:
1. Detects when an option is selected in another choice → "Already selected from [source]"
2. Detects when an option is in `alreadyGrantedIds` → "Already known"

### Dynamic Options Fetching

For choices with `options_endpoint` instead of inline `options`:
- `fetchOptionsIfNeeded()` fetches and caches options
- `getDisplayOptions()` returns cached options or inline options
- `isOptionsLoading()` indicates fetch in progress

**Note:** Backend issue #428 created to embed options inline (optimization).

### Usage Example (StepLanguages)

```vue
<script setup lang="ts">
const store = useCharacterWizardStore()
const { nextStep } = useCharacterWizard()
const toast = useToast()

// Fetch choices via useUnifiedChoices
const { choicesByType, resolveChoice, fetchChoices } = useUnifiedChoices(
  computed(() => store.characterId)
)

onMounted(() => fetchChoices('language'))

// Fetch already-known languages for cross-validation
const { data: knownLanguages } = await useAsyncData(...)
const knownLanguageIds = computed(() =>
  new Set(knownLanguages.value?.map(l => l.language_slug) ?? [])
)

// Composable handles selection logic
const {
  localSelections,
  isSaving,
  getSelectedCount,
  isOptionSelected,
  isOptionDisabled,
  getDisabledReason,
  allComplete,
  handleToggle,
  saveAllChoices
} = useWizardChoiceSelection(
  computed(() => choicesByType.value.languages),
  {
    resolveChoice,
    alreadyGrantedIds: knownLanguageIds
  }
)

// Component handles navigation
async function handleContinue() {
  try {
    await saveAllChoices()
    await store.syncWithBackend()
    nextStep()
  } catch (err) {
    wizardErrors.choiceResolveFailed(err, toast, 'language')
  }
}
</script>
```

### Estimated Reduction

- StepProficiencies: 597 → ~250 lines (58% reduction)
- StepLanguages: 529 → ~200 lines (62% reduction)
- StepSpells: ~400 → ~180 lines (55% reduction)
- StepFeats: ~350 → ~150 lines (57% reduction)
- **Total: ~1,100 lines saved**

---

## Implementation Order

| Task | Priority | Estimated Effort |
|------|----------|------------------|
| Create `useWizardEntitySelection` | High | 1-2 hours |
| Write tests for entity composable | High | 1 hour |
| Refactor StepRace, StepClass, StepBackground | High | 2 hours |
| Create `useWizardChoiceSelection` | High | 2-3 hours |
| Write tests for choice composable | High | 1-2 hours |
| Refactor StepProficiencies, StepLanguages, StepSpells, StepFeats | High | 3 hours |
| Run full test suite | High | 30 min |

**Total: ~12-14 hours**

---

## Testing Strategy

1. **Unit Tests:** Test new composables in isolation
2. **Component Tests:** Existing wizard step tests should still pass
3. **Integration Tests:** Run `npm run test:character` after each phase
4. **Stress Test:** Run `npm run test:character-stress -- --count=5` at the end

---

## Success Metrics

| Metric | Before | Target |
|--------|--------|--------|
| Lines of Code (7 affected steps) | ~2,500 | ~1,100 |
| Duplication % | ~50% | ~15% |
| Composable test coverage | N/A | >90% |

---

## Components NOT Changed

- **StepSubrace** - Nullable selection, fetches full detail before save
- **StepEquipment** - Single-select mode, grouped choices
- **StepSourcebooks** - Different pattern entirely
- **StepAbilities** - Different pattern (point buy/roll)
- **StepDetails** - Form inputs, not entity selection
- **StepReview** - Summary view, no selection

---

## Related Issues

- [#420 - Create useWizardEntitySelection composable](https://github.com/dfox288/ledger-of-heroes/issues/420)
- [#421 - Create useWizardChoiceSelection composable](https://github.com/dfox288/ledger-of-heroes/issues/421)
- [#428 - API: Embed choice options inline](https://github.com/dfox288/ledger-of-heroes/issues/428) (backend optimization)
