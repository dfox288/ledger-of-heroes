# Level-Up Choice Steps Design

**Date:** 2025-12-11
**Issues:** #484 (Spells), #485 (Feature Choices), #486 (Proficiencies)
**Status:** Approved

## Overview

Add spell selection, class feature choices, language choices, and proficiency choices to the level-up wizard. Also integrate feature choices into the character creation wizard to fix the Rogue L1 Expertise gap.

## Decisions

| Decision | Choice |
|----------|--------|
| Component reuse strategy | Props-based abstraction (`characterId`, `nextStep` as props) |
| Stats/data handling | Hybrid - steps fetch own data or receive from parent |
| #486 scope | Feat-granted proficiencies only (multiclass deferred to #482) |
| #485 UI | Single "Feature Choices" step with grouped sections |
| Languages | Include for feats like Linguist |
| Creation wizard | Also add StepFeatureChoices to fix Rogue L1 Expertise gap |

## Level-Up Step Order

```
1. hit-points      → Always first (core level-up action)
2. asi-feat        → At ASI levels (triggers downstream choices)
3. feature-choices → Fighting styles, expertise, invocations (#485)
4. spells          → New spells/cantrips gained (#484)
5. languages       → From feats like Linguist
6. proficiencies   → From feats like Skilled (#486)
7. summary         → Always last
```

## Component Refactoring Pattern

Existing step components will be refactored to accept props instead of directly importing stores:

```typescript
// Before: Hardcoded store dependency
const store = useCharacterWizardStore()
const { characterId, selections, stats } = storeToRefs(store)
const { nextStep } = useCharacterWizard()

// After: Props-based, store-agnostic
const props = defineProps<{
  characterId: number
  nextStep: () => void
  spellcastingStats?: SpellcastingInfo | null
}>()
```

Core composables (`useUnifiedChoices`, `useWizardChoiceSelection`) work unchanged - they only need `characterId`.

## Store Changes

### `characterLevelUp.ts`

```typescript
// New state
const pendingChoices = ref<PendingChoice[]>([])

// Fetch after level-up completes
async function fetchPendingChoices(): Promise<void> {
  if (!characterId.value) return
  const response = await apiFetch<{ data: PendingChoice[] }>(
    `/characters/${publicId.value}/pending-choices`
  )
  pendingChoices.value = response.data
}

// Visibility computeds
const hasSpellChoices = computed(() =>
  pendingChoices.value.some(c => c.type === 'spell')
)

const hasFeatureChoices = computed(() =>
  pendingChoices.value.some(c =>
    ['fighting_style', 'expertise', 'optional_feature'].includes(c.type)
  )
)

const hasLanguageChoices = computed(() =>
  pendingChoices.value.some(c => c.type === 'language')
)

const hasProficiencyChoices = computed(() =>
  pendingChoices.value.some(c => c.type === 'proficiency')
)

// Refresh after each step (e.g., after feat selection creates new choices)
async function refreshChoices(): Promise<void> {
  await fetchPendingChoices()
}
```

### `characterWizard.ts`

```typescript
const hasFeatureChoices = computed(() => {
  if (!summary.value) return false
  return (summary.value.pending_choices.expertise ?? 0) > 0
    || (summary.value.pending_choices.fighting_style ?? 0) > 0
    || (summary.value.pending_choices.optional_feature ?? 0) > 0
})
```

## Step Registry Changes

### `useLevelUpWizard.ts`

```typescript
{
  name: 'feature-choices',
  label: 'Features',
  icon: 'i-heroicons-puzzle-piece',
  visible: () => store.hasFeatureChoices,
  shouldSkip: () => !store.hasFeatureChoices
},
{
  name: 'spells',
  label: 'Spells',
  icon: 'i-heroicons-sparkles',
  visible: () => store.hasSpellChoices,
  shouldSkip: () => !store.hasSpellChoices
},
{
  name: 'languages',
  label: 'Languages',
  icon: 'i-heroicons-language',
  visible: () => store.hasLanguageChoices,
  shouldSkip: () => !store.hasLanguageChoices
},
{
  name: 'proficiencies',
  label: 'Proficiencies',
  icon: 'i-heroicons-academic-cap',
  visible: () => store.hasProficiencyChoices,
  shouldSkip: () => !store.hasProficiencyChoices
}
```

## New Component: `StepFeatureChoices.vue`

Handles fighting_style, expertise, and optional_feature choice types:

```vue
<script setup lang="ts">
const props = defineProps<{
  characterId: number
  nextStep: () => void
}>()

const { choicesByType, fetchChoices, resolveChoice } = useUnifiedChoices(
  computed(() => props.characterId)
)

const fightingStyleChoices = computed(() => choicesByType.value.fightingStyles ?? [])
const expertiseChoices = computed(() => choicesByType.value.expertise ?? [])
const optionalFeatureChoices = computed(() => choicesByType.value.optionalFeatures ?? [])

onMounted(() => fetchChoices(['fighting_style', 'expertise', 'optional_feature']))
</script>

<template>
  <div class="space-y-8">
    <section v-if="fightingStyleChoices.length">
      <h3>Fighting Style</h3>
      <!-- Choice grid -->
    </section>

    <section v-if="expertiseChoices.length">
      <h3>Expertise</h3>
      <!-- Skill selection -->
    </section>

    <section v-if="optionalFeatureChoices.length">
      <h3>{{ choice.source_name }}</h3>
      <!-- Feature selection -->
    </section>
  </div>
</template>
```

## Files to Modify

| File | Changes |
|------|---------|
| `StepSpells.vue` | Add props, remove store dependency |
| `StepProficiencies.vue` | Add props, add `'feat'` source type |
| `StepLanguages.vue` | Add props, remove store dependency |
| `characterLevelUp.ts` | Add pendingChoices, visibility computeds, refreshChoices() |
| `characterWizard.ts` | Add hasFeatureChoices computed |
| `useLevelUpWizard.ts` | Add new steps to registry |
| `useCharacterWizard.ts` | Add feature-choices step |
| `useUnifiedChoices.ts` | Add fightingStyles, expertise to choicesByType |
| `LevelUpWizard.vue` | Wire up new step components |

## Files to Create

| File | Purpose |
|------|---------|
| `StepFeatureChoices.vue` | Fighting style, expertise, optional features |

## Test Coverage

| File | Coverage |
|------|----------|
| `StepFeatureChoices.spec.ts` | New component tests |
| `LevelUpWizard.spec.ts` | Update for new steps |
| `useLevelUpWizard.spec.ts` | Step visibility tests |

## Out of Scope

- **#482 Multiclass class selection** - Required for multiclass proficiencies
- **Multiclass proficiencies** - Deferred until #482 is implemented
- **Subclass selection at level-up** - Separate issue

## Creation Wizard Integration

Add StepFeatureChoices after Proficiencies step:

```
Race → Subrace → Class → Subclass → Background → Abilities →
Proficiencies → Features (NEW) → Languages → Spells → Equipment → Details → Review
```

This fixes the Rogue L1 Expertise gap where the creation wizard currently has no way to select expertise skills.
