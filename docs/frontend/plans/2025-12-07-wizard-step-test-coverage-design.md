# Wizard Step Test Coverage Design

**Date:** 2025-12-07
**Issue:** #310 - Improve test coverage
**Focus:** Character wizard step components

## Problem Statement

The character builder wizard has 9 untested step components (out of 10 total). These components are high-risk because:
- They're frequently modified during feature work
- They integrate with the `characterWizard` Pinia store
- They fetch data from the API and manage complex user flows
- Regressions here break the core character creation experience

## Current State

| Metric | Value |
|--------|-------|
| Total wizard steps | 10 |
| Tested | 1 (StepDetails) |
| Untested | 9 |
| Total lines in untested steps | ~3,000 |

## Goals

1. **Catch regressions** - Focus on high-change areas that frequently break
2. **Confidence for refactoring** - Enable safe improvements to wizard flow
3. **TDD foundation** - Establish patterns for future wizard development

## Approach: Integration + Shared Behavior Hybrid

### Strategy

1. **Integration-style tests** with real Pinia store and mocked API responses
2. **Shared behavior helper** (`wizardStepBehavior.ts`) for common patterns
3. **Step-specific tests** for unique behaviors per step

This mirrors the existing `StepDetails.test.ts` pattern and leverages the test infrastructure already in place.

## Scope & Priority

| Priority | Step | Lines | Risk Factor |
|----------|------|-------|-------------|
| 1 | StepRace | 251 | Gateway step, affects subrace flow |
| 2 | StepClass | 194 | Core choice, affects subclass/spells |
| 3 | StepBackground | 182 | Affects proficiencies/equipment |
| 4 | StepSubrace | 265 | Conditional on race, complex logic |
| 5 | StepSubclass | 195 | Conditional on class level |
| 6 | StepAbilities | 295 | Multiple input methods |
| 7 | StepSourcebooks | 223 | Filters all downstream data |
| 8 | StepSpells | 487 | Complex selection with limits |
| 9 | StepProficiencies | 595 | Most complex, many choice types |

**Deferred:** StepEquipment (467), StepLanguages (350), StepReview (160)

## Test Structure

### File Organization

```
tests/components/character/wizard/
├── StepRace.test.ts
├── StepClass.test.ts
├── StepBackground.test.ts
├── StepSubrace.test.ts
├── StepSubclass.test.ts
├── StepAbilities.test.ts
├── StepSourcebooks.test.ts
├── StepSpells.test.ts
├── StepProficiencies.test.ts
└── StepDetails.test.ts        # (existing)

tests/helpers/
├── wizardStepBehavior.ts      # NEW
└── mockFactories.ts           # (extend with wizard mocks)
```

### Shared Behavior Helper

```typescript
// tests/helpers/wizardStepBehavior.ts

interface WizardStepTestConfig {
  component: Component
  stepTitle: string
  storeField: keyof WizardSelections
  mockData?: unknown
  canProceedWhen?: () => void
}

export function testWizardStepBehavior(config: WizardStepTestConfig) {
  describe(`${config.stepTitle} - Common Behavior`, () => {
    // Structure
    it('renders step title')
    it('renders within expected container structure')

    // Store integration
    it('reads initial selection from store')
    it('updates store when selection changes')

    // Validation
    it('canProceed is false when required selection missing')
    it('canProceed is true when required selection made')

    // Loading states
    it('shows loading indicator while fetching data')
    it('shows error state on fetch failure')
  })
}
```

**Coverage:** ~8-10 common tests per step (~40% boilerplate reduction)

### Step-Specific Tests

#### StepRace
```typescript
describe('StepRace - Specific', () => {
  it('filters out subraces from the race list')
  it('applies sourceFilterString to API query')
  it('search input filters displayed races')
  it('clicking race card updates localSelectedRace')
  it('shows confirmation modal when changing race with existing subrace')
  it('confirming race change clears subrace selection')
  it('View Details opens RaceDetailModal')
})
```

#### StepClass
```typescript
describe('StepClass - Specific', () => {
  it('displays all classes from API')
  it('shows class features preview')
  it('selecting class with subclass_level=1 advances to subclass')
  it('View Details opens ClassDetailModal')
})
```

#### StepAbilities
```typescript
describe('StepAbilities - Specific', () => {
  it('renders three method tabs: Point Buy, Standard Array, Manual')
  it('Point Buy: starts with 27 points, enforces 8-15 range')
  it('Standard Array: allows assignment of fixed values')
  it('Manual: allows free-form entry')
  it('applies racial modifiers to final scores')
  it('shows modifier preview')
})
```

## Mocking Strategy

### API Response Mocking

Extend `tests/setup.ts` with wizard-specific responses:

```typescript
// In createMockApiResponse
if (url.includes('/races')) {
  return Promise.resolve({
    data: [
      { id: 1, name: 'Elf', slug: 'elf', subrace_required: true, ... },
      { id: 2, name: 'Dwarf', slug: 'dwarf', subrace_required: true, ... },
      { id: 3, name: 'Human', slug: 'human', subrace_required: false, ... },
    ]
  })
}

if (url.includes('/classes')) {
  return Promise.resolve({
    data: [
      { id: 1, name: 'Fighter', slug: 'fighter', ... },
      { id: 2, name: 'Wizard', slug: 'wizard', is_spellcaster: true, ... },
    ]
  })
}
```

### Store Setup Pattern

```typescript
beforeEach(() => {
  setActivePinia(createPinia())
  const store = useCharacterWizardStore()
  store.reset()
  // Pre-populate for mid-flow tests
  store.selections.sourcebooks = ['PHB']
})
```

## Success Criteria

Each wizard step test file should have:

- [ ] **10-15 tests minimum** (shared + specific)
- [ ] **Store sync verified** (selection <-> store)
- [ ] **Validation tested** (canProceed logic)
- [ ] **Error states covered** (loading, fetch failure)
- [ ] **Key user flows** (select, change, confirm)

## Expected Output

| Deliverable | Count |
|-------------|-------|
| New test files | 8 |
| New tests | ~100 |
| New helper file | 1 (`wizardStepBehavior.ts`) |
| Extended mock factories | Yes |

## Implementation Order

1. Create `wizardStepBehavior.ts` helper
2. Extend mock factories with wizard entities
3. Implement StepRace tests (establish pattern)
4. Implement StepClass tests (validate pattern)
5. Implement StepBackground tests
6. Extract more shared patterns if discovered
7. Complete remaining steps in priority order

## Risk Mitigation

- **Async data fetching**: Use `mountSuspended` from `@nuxt/test-utils/runtime`
- **Store isolation**: Fresh Pinia instance per test via `setActivePinia(createPinia())`
- **Component auto-unmount**: Already handled by `tests/setup.ts`
- **Test runtime**: Keep tests focused; target <30s for wizard suite
