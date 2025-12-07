# Character Builder Wizard Stepper Refactor

**Issue:** #136
**Date:** 2025-12-03
**Status:** Design Approved

## Problem Statement

The character builder wizard stepper has become difficult to maintain:

- Step numbers are hardcoded and depend on class type (caster vs non-caster)
- `totalSteps` in store calculated separately from `steps` array in page - can drift
- Adding/removing steps requires updates in multiple places
- More conditional steps coming: Feats, Languages, Sourcebook selection
- Horizontal stepper UI won't scale beyond 7-8 steps

## Solution Overview

**Route-based steps** with a **compact progress bar** UI.

### Key Changes

1. Convert single `edit.vue` page into nested routes (`edit/name.vue`, `edit/race.vue`, etc.)
2. Step registry defines all steps with visibility conditions
3. Navigation composable handles prev/next/goto based on active steps
4. Replace horizontal stepper with compact progress bar
5. Route middleware guards conditional steps

## Architecture

### Route Structure

```
app/pages/characters/[id]/
├── edit.vue                  # Layout wrapper with progress bar + <NuxtPage />
└── edit/
    ├── name.vue              # Step: Name
    ├── race.vue              # Step: Race
    ├── class.vue             # Step: Class
    ├── abilities.vue         # Step: Abilities
    ├── background.vue        # Step: Background
    ├── proficiencies.vue     # Step: Proficiencies (conditional)
    ├── equipment.vue         # Step: Equipment
    ├── spells.vue            # Step: Spells (conditional)
    ├── feats.vue             # Step: Feats (future, conditional)
    ├── languages.vue         # Step: Languages (future, conditional)
    └── review.vue            # Step: Review
```

**URLs:**
- `/characters/5/edit/name`
- `/characters/5/edit/race`
- `/characters/5/edit/spells` (only accessible if caster)

### Step Registry

```typescript
// composables/useWizardSteps.ts

interface WizardStep {
  name: string           // Route segment: 'name', 'race', 'spells'
  label: string          // Display label: 'Abilities', 'Spells'
  icon: string           // Heroicon for tooltips
  visible: () => boolean // Visibility condition
}

const stepRegistry: WizardStep[] = [
  { name: 'name', label: 'Name', icon: 'i-heroicons-user', visible: () => true },
  { name: 'race', label: 'Race', icon: 'i-heroicons-globe-alt', visible: () => true },
  { name: 'class', label: 'Class', icon: 'i-heroicons-shield-check', visible: () => true },
  { name: 'abilities', label: 'Abilities', icon: 'i-heroicons-chart-bar', visible: () => true },
  { name: 'background', label: 'Background', icon: 'i-heroicons-book-open', visible: () => true },
  { name: 'proficiencies', label: 'Proficiencies', icon: 'i-heroicons-academic-cap',
    visible: () => store.hasPendingChoices },
  { name: 'equipment', label: 'Equipment', icon: 'i-heroicons-briefcase', visible: () => true },
  { name: 'spells', label: 'Spells', icon: 'i-heroicons-sparkles',
    visible: () => store.isCaster },
  { name: 'review', label: 'Review', icon: 'i-heroicons-check-circle', visible: () => true },
]
```

### Navigation Composable

```typescript
// composables/useWizardNavigation.ts

export function useWizardNavigation() {
  const route = useRoute()
  const store = useCharacterBuilderStore()

  // Filter to only visible steps
  const activeSteps = computed(() =>
    stepRegistry.filter(step => step.visible())
  )

  // Current position
  const currentStepIndex = computed(() =>
    activeSteps.value.findIndex(s => s.name === route.params.step)
  )

  const currentStep = computed(() => activeSteps.value[currentStepIndex.value])
  const totalSteps = computed(() => activeSteps.value.length)
  const isFirstStep = computed(() => currentStepIndex.value === 0)
  const isLastStep = computed(() => currentStepIndex.value === totalSteps.value - 1)

  // Navigation
  function nextStep() {
    const next = activeSteps.value[currentStepIndex.value + 1]
    if (next) navigateTo(`/characters/${store.characterId}/edit/${next.name}`)
  }

  function previousStep() {
    const prev = activeSteps.value[currentStepIndex.value - 1]
    if (prev) navigateTo(`/characters/${store.characterId}/edit/${prev.name}`)
  }

  function goToStep(stepName: string) {
    const step = activeSteps.value.find(s => s.name === stepName)
    if (step) navigateTo(`/characters/${store.characterId}/edit/${stepName}`)
  }

  return {
    activeSteps,
    currentStep,
    currentStepIndex,
    totalSteps,
    isFirstStep,
    isLastStep,
    nextStep,
    previousStep,
    goToStep
  }
}
```

### Route Middleware

```typescript
// middleware/wizard-step.ts

export default defineNuxtRouteMiddleware((to) => {
  const store = useCharacterBuilderStore()
  const stepName = to.params.step as string

  // Find step in registry and check visibility
  const step = stepRegistry.find(s => s.name === stepName)

  if (!step || !step.visible()) {
    // Redirect to first visible step
    return navigateTo(`/characters/${to.params.id}/edit/name`)
  }
})
```

## UI Design

### Compact Progress Bar

Replace the horizontal stepper with a compact progress bar:

```
┌─────────────────────────────────────────────────────────────────┐
│  ← Back    ●●●●○○○○○    Step 4 of 9: Abilities       Next →    │
└─────────────────────────────────────────────────────────────────┘
```

**Elements:**
- Back button (hidden on first step)
- Progress dots: ● = completed, ○ = future, highlighted = current
- Step counter with label: "Step 4 of 9: Abilities"
- Next button (changes to "Finish" on review step)

**Interactions:**
- Clicking a completed dot navigates to that step
- Hovering a dot shows step name tooltip
- Progress bar is responsive (dots may hide on very small screens, showing only counter)

### Component Structure

```
CharacterBuilderProgressBar.vue
├── Props: activeSteps, currentStepIndex
├── Emits: @step-click(stepName)
└── Slots: none
```

## Store Changes

Remove from `characterBuilder.ts`:
- `currentStep` ref (navigation now via route)
- `totalSteps` computed (now in composable)
- `isFirstStep` / `isLastStep` computed (now in composable)
- `nextStep()` / `previousStep()` / `goToStep()` actions (now in composable)

Keep in store:
- All character data and selection state
- `isCaster`, `hasPendingChoices` and other condition flags
- API actions (selectRace, selectClass, etc.)

## Migration Strategy

1. Create new route structure alongside existing page
2. Extract step components from existing file (they're already separate)
3. Create composable and middleware
4. Create new progress bar component
5. Update tests
6. Remove old implementation

## Benefits

| Before | After |
|--------|-------|
| Step numbers hardcoded | Steps identified by name |
| Two sources of truth (store + page) | Single step registry |
| Adding step = edit multiple files | Adding step = registry entry + page file |
| URL doesn't reflect current step | URL = `/edit/abilities` |
| Stepper UI won't scale | Compact bar scales infinitely |
| Can't deep-link to step | Can share URL to specific step |

## Future Extensions

Easy to add:
- `feats.vue` with `visible: () => store.hasAvailableFeats`
- `languages.vue` with `visible: () => store.hasLanguageChoices`
- `sourcebook.vue` with `visible: () => true` (always, at start)

The registry pattern makes this trivial.

## Files to Create/Modify

**Create:**
- `app/composables/useWizardSteps.ts` - Step registry
- `app/composables/useWizardNavigation.ts` - Navigation logic
- `app/middleware/wizard-step.ts` - Route guard
- `app/components/character/builder/ProgressBar.vue` - New UI
- `app/pages/characters/[id]/edit/name.vue`
- `app/pages/characters/[id]/edit/race.vue`
- `app/pages/characters/[id]/edit/class.vue`
- `app/pages/characters/[id]/edit/abilities.vue`
- `app/pages/characters/[id]/edit/background.vue`
- `app/pages/characters/[id]/edit/proficiencies.vue`
- `app/pages/characters/[id]/edit/equipment.vue`
- `app/pages/characters/[id]/edit/spells.vue`
- `app/pages/characters/[id]/edit/review.vue`

**Modify:**
- `app/pages/characters/[id]/edit.vue` - Becomes layout wrapper
- `app/stores/characterBuilder.ts` - Remove navigation state

**Delete:**
- `app/components/character/builder/Stepper.vue` - Replace with ProgressBar
