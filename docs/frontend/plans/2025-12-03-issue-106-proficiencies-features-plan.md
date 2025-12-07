# Character Proficiencies & Features Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a conditional "Proficiency Choices" step to the character builder wizard and display proficiencies/features on the Review step.

**Architecture:** New Nitro routes proxy 4 backend endpoints. Store gains state for pending choices and user selections. New `StepProficiencies.vue` component handles skill selection UI. Edit page routing updated for conditional step. Review step enhanced with proficiency/feature display sections.

**Tech Stack:** Nuxt 4, NuxtUI 4, Pinia, TypeScript, Vitest

**Design Doc:** `docs/plans/2025-12-03-issue-106-proficiencies-features-design.md`

**Test Commands:**
- Single test: `docker compose exec nuxt npx vitest run <path>`
- Related tests: `docker compose exec nuxt npm run test`
- TypeCheck: `docker compose exec nuxt npm run typecheck`

---

## Task 1: Create Nitro Route - GET proficiencies

**Files:**
- Create: `server/api/characters/[id]/proficiencies.get.ts`

**Step 1: Create the route file**

```typescript
// server/api/characters/[id]/proficiencies.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/proficiencies`)
  return data
})
```

**Step 2: Verify route works**

Run: `curl -s http://localhost:4000/api/characters/1/proficiencies | jq '.data'`
Expected: JSON array (may be empty if no proficiencies yet)

**Step 3: Commit**

```bash
git add server/api/characters/[id]/proficiencies.get.ts
git commit -m "feat(api): add GET /characters/:id/proficiencies route"
```

---

## Task 2: Create Nitro Route - GET proficiency-choices

**Files:**
- Create: `server/api/characters/[id]/proficiency-choices.get.ts`

**Step 1: Create the route file**

```typescript
// server/api/characters/[id]/proficiency-choices.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/proficiency-choices`)
  return data
})
```

**Step 2: Verify route works**

Run: `curl -s http://localhost:4000/api/characters/1/proficiency-choices | jq '.data'`
Expected: JSON with `class`, `race`, `background` keys

**Step 3: Commit**

```bash
git add server/api/characters/[id]/proficiency-choices.get.ts
git commit -m "feat(api): add GET /characters/:id/proficiency-choices route"
```

---

## Task 3: Create Nitro Route - POST proficiency-choices

**Files:**
- Create: `server/api/characters/[id]/proficiency-choices.post.ts`

**Step 1: Create the route file**

```typescript
// server/api/characters/[id]/proficiency-choices.post.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const body = await readBody(event)

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/proficiency-choices`, {
    method: 'POST',
    body
  })
  return data
})
```

**Step 2: Commit**

```bash
git add server/api/characters/[id]/proficiency-choices.post.ts
git commit -m "feat(api): add POST /characters/:id/proficiency-choices route"
```

---

## Task 4: Create Nitro Route - GET features

**Files:**
- Create: `server/api/characters/[id]/features.get.ts`

**Step 1: Create the route file**

```typescript
// server/api/characters/[id]/features.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/features`)
  return data
})
```

**Step 2: Verify route works**

Run: `curl -s http://localhost:4000/api/characters/1/features | jq '.data'`
Expected: JSON array (may be empty)

**Step 3: Commit**

```bash
git add server/api/characters/[id]/features.get.ts
git commit -m "feat(api): add GET /characters/:id/features route"
```

---

## Task 5: Add TypeScript Types for Proficiency Choices

**Files:**
- Create: `app/types/proficiencies.ts`

**Step 1: Create types file**

```typescript
// app/types/proficiencies.ts

export interface SkillOption {
  type: 'skill'
  skill_id: number
  skill: {
    id: number
    name: string
    slug: string
  }
}

export interface ProficiencyChoice {
  quantity: number
  remaining: number
  options: SkillOption[]
}

export interface ProficiencyChoicesResponse {
  data: {
    class: Record<string, ProficiencyChoice>
    race: Record<string, ProficiencyChoice>
    background: Record<string, ProficiencyChoice>
  }
}

export interface CharacterProficiency {
  id: number
  source: 'class' | 'race' | 'background'
  type: 'skill' | 'saving_throw' | 'armor' | 'weapon' | 'tool' | 'language'
  name: string
  ability_score?: {
    id: number
    name: string
    code: string
  }
}

export interface CharacterFeature {
  id: number
  source: 'class' | 'race' | 'background'
  name: string
  description: string
  level?: number
}

export interface ProficienciesResponse {
  data: CharacterProficiency[]
}

export interface FeaturesResponse {
  data: CharacterFeature[]
}
```

**Step 2: Commit**

```bash
git add app/types/proficiencies.ts
git commit -m "feat(types): add proficiency and feature type definitions"
```

---

## Task 6: Add Store State and Computed for Proficiency Choices

**Files:**
- Modify: `app/stores/characterBuilder.ts`

**Step 1: Read current store structure**

Review lines 1-100 of `app/stores/characterBuilder.ts` to understand existing state pattern.

**Step 2: Add imports and new state**

Add after existing imports (around line 5):

```typescript
import type { ProficiencyChoicesResponse } from '~/types/proficiencies'
```

Add new state inside `defineStore` function (after existing state refs, around line 70):

```typescript
  // Proficiency choices from API
  const proficiencyChoices = ref<ProficiencyChoicesResponse | null>(null)

  // User's pending selections: Map<"source:choice_group", Set<skillId>>
  const pendingProficiencySelections = ref<Map<string, Set<number>>>(new Map())
```

**Step 3: Add hasPendingChoices computed**

Add after existing computed properties (around line 110):

```typescript
  // Does this character have any pending proficiency choices?
  const hasPendingChoices = computed(() => {
    if (!proficiencyChoices.value) return false
    const { class: cls, race, background } = proficiencyChoices.value.data
    return Object.keys(cls).length > 0 ||
           Object.keys(race).length > 0 ||
           Object.keys(background).length > 0
  })
```

**Step 4: Update totalSteps computed**

Find the `totalSteps` computed (around line 19-21) and update:

```typescript
  const totalSteps = computed(() => {
    let steps = 7 // Base: Name, Race, Class, Abilities, Background, Equipment, Review
    if (hasPendingChoices.value) steps++
    if (isCaster.value) steps++
    return steps
  })
```

**Step 5: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

**Step 6: Commit**

```bash
git add app/stores/characterBuilder.ts
git commit -m "feat(store): add proficiency choices state and hasPendingChoices computed"
```

---

## Task 7: Add Store Actions for Proficiency Choices

**Files:**
- Modify: `app/stores/characterBuilder.ts`
- Test: `tests/stores/characterBuilder.test.ts`

**Step 1: Write failing tests**

Add new describe block at end of test file:

```typescript
describe('proficiency choices', () => {
  it('fetchProficiencyChoices populates proficiencyChoices state', async () => {
    const store = useCharacterBuilderStore()
    store.characterId = 1

    // Mock the API response
    vi.mocked(apiFetch).mockResolvedValueOnce({
      data: {
        class: {
          skill_choice_1: {
            quantity: 2,
            remaining: 2,
            options: [
              { type: 'skill', skill_id: 1, skill: { id: 1, name: 'Acrobatics', slug: 'acrobatics' } }
            ]
          }
        },
        race: {},
        background: {}
      }
    })

    await store.fetchProficiencyChoices()

    expect(store.proficiencyChoices).not.toBeNull()
    expect(store.proficiencyChoices?.data.class.skill_choice_1.quantity).toBe(2)
  })

  it('toggleProficiencySelection adds and removes skills', () => {
    const store = useCharacterBuilderStore()

    store.toggleProficiencySelection('class', 'skill_choice_1', 5)
    expect(store.pendingProficiencySelections.get('class:skill_choice_1')?.has(5)).toBe(true)

    store.toggleProficiencySelection('class', 'skill_choice_1', 5)
    expect(store.pendingProficiencySelections.get('class:skill_choice_1')?.has(5)).toBe(false)
  })

  it('hasPendingChoices returns true when choices exist', () => {
    const store = useCharacterBuilderStore()

    store.proficiencyChoices = {
      data: {
        class: { skill_choice_1: { quantity: 2, remaining: 2, options: [] } },
        race: {},
        background: {}
      }
    }

    expect(store.hasPendingChoices).toBe(true)
  })

  it('hasPendingChoices returns false when no choices', () => {
    const store = useCharacterBuilderStore()

    store.proficiencyChoices = {
      data: { class: {}, race: {}, background: {} }
    }

    expect(store.hasPendingChoices).toBe(false)
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npx vitest run tests/stores/characterBuilder.test.ts`
Expected: FAIL - functions not defined

**Step 3: Implement fetchProficiencyChoices action**

Add in the store after existing actions:

```typescript
  // Fetch pending proficiency choices from API
  async function fetchProficiencyChoices(): Promise<void> {
    if (!characterId.value) return

    const response = await apiFetch<ProficiencyChoicesResponse>(
      `/characters/${characterId.value}/proficiency-choices`
    )
    proficiencyChoices.value = response
  }
```

**Step 4: Implement toggleProficiencySelection action**

```typescript
  // Toggle a skill selection in pending state
  function toggleProficiencySelection(
    source: 'class' | 'race' | 'background',
    choiceGroup: string,
    skillId: number
  ): void {
    const key = `${source}:${choiceGroup}`
    const current = pendingProficiencySelections.value.get(key) ?? new Set<number>()

    if (current.has(skillId)) {
      current.delete(skillId)
    } else {
      current.add(skillId)
    }

    pendingProficiencySelections.value.set(key, current)
  }
```

**Step 5: Implement saveProficiencyChoices action**

```typescript
  // Save all pending proficiency selections to API
  async function saveProficiencyChoices(): Promise<void> {
    if (!characterId.value) return

    for (const [key, skillIds] of pendingProficiencySelections.value) {
      if (skillIds.size === 0) continue

      const [source, choiceGroup] = key.split(':')
      await apiFetch(`/characters/${characterId.value}/proficiency-choices`, {
        method: 'POST',
        body: {
          source,
          choice_group: choiceGroup,
          skill_ids: [...skillIds]
        }
      })
    }

    // Refresh choices to update remaining counts
    await fetchProficiencyChoices()
  }
```

**Step 6: Export new state and actions in return statement**

Find the return statement and add:

```typescript
    // Proficiency choices
    proficiencyChoices,
    pendingProficiencySelections,
    hasPendingChoices,
    fetchProficiencyChoices,
    toggleProficiencySelection,
    saveProficiencyChoices,
```

**Step 7: Run tests to verify they pass**

Run: `docker compose exec nuxt npx vitest run tests/stores/characterBuilder.test.ts`
Expected: All tests PASS

**Step 8: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "feat(store): add proficiency choices actions and tests"
```

---

## Task 8: Create StepProficiencies Component - Basic Structure

**Files:**
- Create: `app/components/character/builder/StepProficiencies.vue`
- Create: `tests/components/character/builder/StepProficiencies.test.ts`

**Step 1: Write failing test for basic rendering**

```typescript
// tests/components/character/builder/StepProficiencies.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StepProficiencies from '~/components/character/builder/StepProficiencies.vue'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

describe('StepProficiencies', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders header and description', async () => {
    const store = useCharacterBuilderStore()
    store.proficiencyChoices = {
      data: { class: {}, race: {}, background: {} }
    }

    const wrapper = await mountSuspended(StepProficiencies)

    expect(wrapper.text()).toContain('Choose Your Proficiencies')
  })

  it('shows message when no choices are needed', async () => {
    const store = useCharacterBuilderStore()
    store.proficiencyChoices = {
      data: { class: {}, race: {}, background: {} }
    }

    const wrapper = await mountSuspended(StepProficiencies)

    expect(wrapper.text()).toContain('No additional choices needed')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npx vitest run tests/components/character/builder/StepProficiencies.test.ts`
Expected: FAIL - component not found

**Step 3: Create basic component structure**

```vue
<!-- app/components/character/builder/StepProficiencies.vue -->
<script setup lang="ts">
const store = useCharacterBuilderStore()

const hasAnyChoices = computed(() => {
  if (!store.proficiencyChoices) return false
  const { class: cls, race, background } = store.proficiencyChoices.data
  return Object.keys(cls).length > 0 ||
         Object.keys(race).length > 0 ||
         Object.keys(background).length > 0
})
</script>

<template>
  <div class="step-proficiencies">
    <div class="text-center mb-8">
      <h2 class="text-2xl font-bold text-primary">
        Choose Your Proficiencies
      </h2>
      <p class="text-muted mt-2">
        Your class, race, and background grant the following choices
      </p>
    </div>

    <!-- No choices needed -->
    <div
      v-if="!hasAnyChoices"
      class="text-center py-8"
    >
      <UIcon
        name="i-heroicons-check-circle"
        class="w-12 h-12 text-success mx-auto mb-4"
      />
      <p class="text-lg">No additional choices needed</p>
      <p class="text-muted">
        All your proficiencies have been automatically assigned
      </p>
    </div>

    <!-- Choices will go here -->
    <div v-else>
      <!-- TODO: Render choice groups -->
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npx vitest run tests/components/character/builder/StepProficiencies.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/StepProficiencies.vue tests/components/character/builder/StepProficiencies.test.ts
git commit -m "feat(component): add StepProficiencies basic structure"
```

---

## Task 9: StepProficiencies - Render Choice Groups

**Files:**
- Modify: `app/components/character/builder/StepProficiencies.vue`
- Modify: `tests/components/character/builder/StepProficiencies.test.ts`

**Step 1: Write failing test for choice group rendering**

Add to test file:

```typescript
  it('renders class skill choices when present', async () => {
    const store = useCharacterBuilderStore()
    store.proficiencyChoices = {
      data: {
        class: {
          skill_choice_1: {
            quantity: 2,
            remaining: 2,
            options: [
              { type: 'skill', skill_id: 1, skill: { id: 1, name: 'Acrobatics', slug: 'acrobatics' } },
              { type: 'skill', skill_id: 4, skill: { id: 4, name: 'Athletics', slug: 'athletics' } }
            ]
          }
        },
        race: {},
        background: {}
      }
    }
    store.selectedClass = { name: 'Bard' } as any

    const wrapper = await mountSuspended(StepProficiencies)

    expect(wrapper.text()).toContain('From Class: Bard')
    expect(wrapper.text()).toContain('Choose 2 skills')
    expect(wrapper.text()).toContain('Acrobatics')
    expect(wrapper.text()).toContain('Athletics')
  })
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npx vitest run tests/components/character/builder/StepProficiencies.test.ts`
Expected: FAIL

**Step 3: Update component to render choice groups**

Replace the `<!-- TODO: Render choice groups -->` section:

```vue
<script setup lang="ts">
const store = useCharacterBuilderStore()

const hasAnyChoices = computed(() => {
  if (!store.proficiencyChoices) return false
  const { class: cls, race, background } = store.proficiencyChoices.data
  return Object.keys(cls).length > 0 ||
         Object.keys(race).length > 0 ||
         Object.keys(background).length > 0
})

// Organize choices by source for display
const choicesBySource = computed(() => {
  if (!store.proficiencyChoices) return []

  const sources: Array<{
    source: 'class' | 'race' | 'background'
    label: string
    entityName: string
    groups: Array<{
      groupName: string
      quantity: number
      remaining: number
      options: Array<{ skill_id: number; skill: { id: number; name: string; slug: string } }>
    }>
  }> = []

  const { class: cls, race, background } = store.proficiencyChoices.data

  if (Object.keys(cls).length > 0) {
    sources.push({
      source: 'class',
      label: 'From Class',
      entityName: store.selectedClass?.name ?? 'Unknown',
      groups: Object.entries(cls).map(([groupName, group]) => ({
        groupName,
        quantity: group.quantity,
        remaining: group.remaining,
        options: group.options
      }))
    })
  }

  if (Object.keys(race).length > 0) {
    sources.push({
      source: 'race',
      label: 'From Race',
      entityName: store.selectedRace?.name ?? 'Unknown',
      groups: Object.entries(race).map(([groupName, group]) => ({
        groupName,
        quantity: group.quantity,
        remaining: group.remaining,
        options: group.options
      }))
    })
  }

  if (Object.keys(background).length > 0) {
    sources.push({
      source: 'background',
      label: 'From Background',
      entityName: store.selectedBackground?.name ?? 'Unknown',
      groups: Object.entries(background).map(([groupName, group]) => ({
        groupName,
        quantity: group.quantity,
        remaining: group.remaining,
        options: group.options
      }))
    })
  }

  return sources
})

// Get selected count for a choice group
function getSelectedCount(source: string, groupName: string): number {
  const key = `${source}:${groupName}`
  return store.pendingProficiencySelections.get(key)?.size ?? 0
}

// Check if a skill is selected
function isSkillSelected(source: string, groupName: string, skillId: number): boolean {
  const key = `${source}:${groupName}`
  return store.pendingProficiencySelections.get(key)?.has(skillId) ?? false
}

// Handle skill toggle
function handleSkillToggle(source: 'class' | 'race' | 'background', groupName: string, skillId: number, quantity: number) {
  const key = `${source}:${groupName}`
  const current = store.pendingProficiencySelections.get(key)?.size ?? 0
  const isSelected = isSkillSelected(source, groupName, skillId)

  // Don't allow selecting more than quantity
  if (!isSelected && current >= quantity) return

  store.toggleProficiencySelection(source, groupName, skillId)
}
</script>

<template>
  <div class="step-proficiencies">
    <div class="text-center mb-8">
      <h2 class="text-2xl font-bold text-primary">
        Choose Your Proficiencies
      </h2>
      <p class="text-muted mt-2">
        Your class, race, and background grant the following choices
      </p>
    </div>

    <!-- No choices needed -->
    <div
      v-if="!hasAnyChoices"
      class="text-center py-8"
    >
      <UIcon
        name="i-heroicons-check-circle"
        class="w-12 h-12 text-success mx-auto mb-4"
      />
      <p class="text-lg">No additional choices needed</p>
      <p class="text-muted">
        All your proficiencies have been automatically assigned
      </p>
    </div>

    <!-- Choice groups by source -->
    <div
      v-else
      class="space-y-8"
    >
      <div
        v-for="sourceData in choicesBySource"
        :key="sourceData.source"
        class="choice-source"
      >
        <!-- Source header -->
        <h3 class="text-lg font-semibold mb-4">
          {{ sourceData.label }}: {{ sourceData.entityName }}
        </h3>

        <!-- Choice groups within source -->
        <div
          v-for="group in sourceData.groups"
          :key="group.groupName"
          class="mb-6"
        >
          <div class="flex items-center justify-between mb-3">
            <span class="text-sm font-medium">
              Choose {{ group.quantity }} skills:
            </span>
            <UBadge
              :color="getSelectedCount(sourceData.source, group.groupName) === group.quantity ? 'success' : 'neutral'"
              size="md"
            >
              {{ getSelectedCount(sourceData.source, group.groupName) }}/{{ group.quantity }} selected
            </UBadge>
          </div>

          <!-- Skill options grid -->
          <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-3">
            <button
              v-for="option in group.options"
              :key="option.skill_id"
              type="button"
              class="skill-option p-3 rounded-lg border text-left transition-all"
              :class="{
                'border-primary bg-primary/10': isSkillSelected(sourceData.source, group.groupName, option.skill_id),
                'border-muted hover:border-primary/50': !isSkillSelected(sourceData.source, group.groupName, option.skill_id)
              }"
              @click="handleSkillToggle(sourceData.source, group.groupName, option.skill_id, group.quantity)"
            >
              <div class="flex items-center gap-2">
                <UIcon
                  :name="isSkillSelected(sourceData.source, group.groupName, option.skill_id) ? 'i-heroicons-check-circle-solid' : 'i-heroicons-circle'"
                  class="w-5 h-5"
                  :class="{
                    'text-primary': isSkillSelected(sourceData.source, group.groupName, option.skill_id),
                    'text-muted': !isSkillSelected(sourceData.source, group.groupName, option.skill_id)
                  }"
                />
                <span class="font-medium">{{ option.skill.name }}</span>
              </div>
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npx vitest run tests/components/character/builder/StepProficiencies.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/StepProficiencies.vue tests/components/character/builder/StepProficiencies.test.ts
git commit -m "feat(component): StepProficiencies renders choice groups with skill selection"
```

---

## Task 10: StepProficiencies - Validation and Selection Tests

**Files:**
- Modify: `tests/components/character/builder/StepProficiencies.test.ts`

**Step 1: Add selection interaction tests**

```typescript
  it('toggles skill selection when clicked', async () => {
    const store = useCharacterBuilderStore()
    store.proficiencyChoices = {
      data: {
        class: {
          skill_choice_1: {
            quantity: 2,
            remaining: 2,
            options: [
              { type: 'skill', skill_id: 1, skill: { id: 1, name: 'Acrobatics', slug: 'acrobatics' } },
              { type: 'skill', skill_id: 4, skill: { id: 4, name: 'Athletics', slug: 'athletics' } }
            ]
          }
        },
        race: {},
        background: {}
      }
    }
    store.selectedClass = { name: 'Bard' } as any

    const wrapper = await mountSuspended(StepProficiencies)

    // Click first skill
    const buttons = wrapper.findAll('.skill-option')
    await buttons[0].trigger('click')

    // Verify store was updated
    expect(store.pendingProficiencySelections.get('class:skill_choice_1')?.has(1)).toBe(true)
  })

  it('shows correct selection count badge', async () => {
    const store = useCharacterBuilderStore()
    store.proficiencyChoices = {
      data: {
        class: {
          skill_choice_1: {
            quantity: 2,
            remaining: 2,
            options: [
              { type: 'skill', skill_id: 1, skill: { id: 1, name: 'Acrobatics', slug: 'acrobatics' } }
            ]
          }
        },
        race: {},
        background: {}
      }
    }
    store.selectedClass = { name: 'Bard' } as any

    // Pre-select a skill
    store.pendingProficiencySelections.set('class:skill_choice_1', new Set([1]))

    const wrapper = await mountSuspended(StepProficiencies)

    expect(wrapper.text()).toContain('1/2 selected')
  })

  it('prevents selecting more than allowed quantity', async () => {
    const store = useCharacterBuilderStore()
    store.proficiencyChoices = {
      data: {
        class: {
          skill_choice_1: {
            quantity: 1,
            remaining: 1,
            options: [
              { type: 'skill', skill_id: 1, skill: { id: 1, name: 'Acrobatics', slug: 'acrobatics' } },
              { type: 'skill', skill_id: 4, skill: { id: 4, name: 'Athletics', slug: 'athletics' } }
            ]
          }
        },
        race: {},
        background: {}
      }
    }
    store.selectedClass = { name: 'Rogue' } as any

    const wrapper = await mountSuspended(StepProficiencies)
    const buttons = wrapper.findAll('.skill-option')

    // Select first skill
    await buttons[0].trigger('click')
    expect(store.pendingProficiencySelections.get('class:skill_choice_1')?.size).toBe(1)

    // Try to select second skill (should not add)
    await buttons[1].trigger('click')
    expect(store.pendingProficiencySelections.get('class:skill_choice_1')?.size).toBe(1)
  })
```

**Step 2: Run tests to verify they pass**

Run: `docker compose exec nuxt npx vitest run tests/components/character/builder/StepProficiencies.test.ts`
Expected: All PASS

**Step 3: Commit**

```bash
git add tests/components/character/builder/StepProficiencies.test.ts
git commit -m "test(StepProficiencies): add selection interaction and validation tests"
```

---

## Task 11: Update Edit Page Step Routing

**Files:**
- Modify: `app/pages/characters/[id]/edit.vue`

**Step 1: Read current edit page structure**

Review `app/pages/characters/[id]/edit.vue` to understand the current step rendering logic.

**Step 2: Import StepProficiencies component**

The component should auto-import, but verify the step mapping logic.

**Step 3: Update step component mapping**

Find the `currentStepComponent` computed or similar step rendering logic and update to include proficiency step:

```typescript
// Helper to calculate actual step index accounting for conditional steps
const getStepComponent = computed(() => {
  const step = store.currentStep

  // Fixed steps 1-5
  if (step === 1) return StepName
  if (step === 2) return StepRace
  if (step === 3) return StepClass
  if (step === 4) return StepAbilities
  if (step === 5) return StepBackground

  // After step 5, need to account for conditional steps
  let currentIndex = 5

  // Proficiency choices step (conditional)
  if (store.hasPendingChoices) {
    currentIndex++
    if (step === currentIndex) return StepProficiencies
  }

  // Equipment step (always present)
  currentIndex++
  if (step === currentIndex) return StepEquipment

  // Spells step (conditional for casters)
  if (store.isCaster) {
    currentIndex++
    if (step === currentIndex) return StepSpells
  }

  // Review is always last
  return StepReview
})
```

**Step 4: Update stepper labels**

Find the stepper labels array and make it dynamic:

```typescript
const stepLabels = computed(() => {
  const labels = ['Name', 'Race', 'Class', 'Abilities', 'Background']

  if (store.hasPendingChoices) {
    labels.push('Proficiencies')
  }

  labels.push('Equipment')

  if (store.isCaster) {
    labels.push('Spells')
  }

  labels.push('Review')

  return labels
})
```

**Step 5: Ensure proficiency choices are fetched after background selection**

In the background step completion handler or store action, add:

```typescript
// After selectBackground completes, fetch proficiency choices
await store.fetchProficiencyChoices()
```

**Step 6: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

**Step 7: Commit**

```bash
git add app/pages/characters/[id]/edit.vue
git commit -m "feat(edit): add conditional proficiency step to wizard routing"
```

---

## Task 12: Call fetchProficiencyChoices After Background Selection

**Files:**
- Modify: `app/stores/characterBuilder.ts`

**Step 1: Update selectBackground action**

Find the `selectBackground` function and add proficiency fetch at the end:

```typescript
async function selectBackground(background: Background): Promise<void> {
  // ... existing code ...

  // Fetch proficiency choices now that all sources are selected
  await fetchProficiencyChoices()
}
```

**Step 2: Also fetch in loadCharacterForEditing**

Find `loadCharacterForEditing` and add proficiency fetch:

```typescript
async function loadCharacterForEditing(id: number): Promise<void> {
  // ... existing code ...

  // Fetch proficiency choices for existing character
  await fetchProficiencyChoices()
}
```

**Step 3: Run tests**

Run: `docker compose exec nuxt npx vitest run tests/stores/characterBuilder.test.ts`
Expected: PASS

**Step 4: Commit**

```bash
git add app/stores/characterBuilder.ts
git commit -m "feat(store): fetch proficiency choices after background selection and on edit load"
```

---

## Task 13: Add allProficiencyChoicesComplete Validation

**Files:**
- Modify: `app/stores/characterBuilder.ts`
- Modify: `tests/stores/characterBuilder.test.ts`

**Step 1: Write failing test**

```typescript
  it('allProficiencyChoicesComplete returns true when all choices made', () => {
    const store = useCharacterBuilderStore()

    store.proficiencyChoices = {
      data: {
        class: {
          skill_choice_1: { quantity: 2, remaining: 2, options: [] }
        },
        race: {},
        background: {}
      }
    }

    // No selections yet
    expect(store.allProficiencyChoicesComplete).toBe(false)

    // Add correct number of selections
    store.pendingProficiencySelections.set('class:skill_choice_1', new Set([1, 5]))
    expect(store.allProficiencyChoicesComplete).toBe(true)
  })
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npx vitest run tests/stores/characterBuilder.test.ts`
Expected: FAIL

**Step 3: Add computed property**

```typescript
  // Are all required proficiency choices complete?
  const allProficiencyChoicesComplete = computed(() => {
    if (!proficiencyChoices.value) return true
    if (!hasPendingChoices.value) return true

    const { class: cls, race, background } = proficiencyChoices.value.data

    for (const [groupName, group] of Object.entries(cls)) {
      const selected = pendingProficiencySelections.value.get(`class:${groupName}`)?.size ?? 0
      if (selected !== group.quantity) return false
    }

    for (const [groupName, group] of Object.entries(race)) {
      const selected = pendingProficiencySelections.value.get(`race:${groupName}`)?.size ?? 0
      if (selected !== group.quantity) return false
    }

    for (const [groupName, group] of Object.entries(background)) {
      const selected = pendingProficiencySelections.value.get(`background:${groupName}`)?.size ?? 0
      if (selected !== group.quantity) return false
    }

    return true
  })
```

**Step 4: Export in return statement**

```typescript
    allProficiencyChoicesComplete,
```

**Step 5: Run test to verify it passes**

Run: `docker compose exec nuxt npx vitest run tests/stores/characterBuilder.test.ts`
Expected: PASS

**Step 6: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "feat(store): add allProficiencyChoicesComplete validation computed"
```

---

## Task 14: Review Step - Add Proficiencies Section

**Files:**
- Modify: `app/components/character/builder/StepReview.vue`
- Modify: `tests/components/character/builder/StepReview.test.ts`

**Step 1: Write failing test**

Add to StepReview test file:

```typescript
describe('proficiencies section', () => {
  it('displays proficiencies section header', async () => {
    const store = useCharacterBuilderStore()
    // ... setup store with character data ...

    const wrapper = await mountSuspended(StepReview)

    expect(wrapper.text()).toContain('Proficiencies')
  })
})
```

**Step 2: Add proficiencies section to component**

Add after the existing review sections, before the finish button:

```vue
<!-- Proficiencies Section -->
<UCard class="mb-6">
  <template #header>
    <div class="flex items-center justify-between">
      <h3 class="font-semibold">Proficiencies</h3>
      <UButton
        v-if="store.hasPendingChoices"
        variant="ghost"
        size="sm"
        icon="i-heroicons-pencil"
        @click="goToStep(proficiencyStepNumber)"
      >
        Edit
      </UButton>
    </div>
  </template>

  <div class="space-y-4">
    <!-- Skills -->
    <div v-if="selectedSkills.length > 0">
      <span class="text-sm font-medium text-muted">Skills:</span>
      <span class="ml-2">{{ selectedSkills.join(', ') }}</span>
    </div>

    <!-- Message if still loading or no proficiencies -->
    <p
      v-if="selectedSkills.length === 0"
      class="text-muted text-sm"
    >
      No skill proficiencies selected yet
    </p>
  </div>
</UCard>
```

**Step 3: Add computed for selected skills**

```typescript
const selectedSkills = computed(() => {
  const skills: string[] = []

  for (const [key, skillIds] of store.pendingProficiencySelections) {
    const [source, groupName] = key.split(':')
    const sourceData = store.proficiencyChoices?.data[source as 'class' | 'race' | 'background']
    const group = sourceData?.[groupName]

    if (group) {
      for (const skillId of skillIds) {
        const option = group.options.find(o => o.skill_id === skillId)
        if (option) {
          skills.push(option.skill.name)
        }
      }
    }
  }

  return skills
})

const proficiencyStepNumber = computed(() => {
  // Proficiency step is step 6 when it exists
  return store.hasPendingChoices ? 6 : -1
})
```

**Step 4: Run tests**

Run: `docker compose exec nuxt npx vitest run tests/components/character/builder/StepReview.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/StepReview.vue tests/components/character/builder/StepReview.test.ts
git commit -m "feat(StepReview): add proficiencies section displaying selected skills"
```

---

## Task 15: Review Step - Add Features Section

**Files:**
- Modify: `app/components/character/builder/StepReview.vue`

**Step 1: Add features section**

Add after proficiencies section:

```vue
<!-- Features Section -->
<UCard class="mb-6">
  <template #header>
    <h3 class="font-semibold">Features & Traits</h3>
  </template>

  <div class="space-y-6">
    <!-- Class Features -->
    <div v-if="classFeatures.length > 0">
      <h4 class="text-sm font-medium text-muted mb-2">Class Features</h4>
      <div class="space-y-2">
        <div
          v-for="feature in classFeatures"
          :key="feature.name"
          class="border-l-2 border-primary pl-3"
        >
          <span class="font-medium">{{ feature.name }}</span>
        </div>
      </div>
    </div>

    <!-- Racial Traits -->
    <div v-if="racialTraits.length > 0">
      <h4 class="text-sm font-medium text-muted mb-2">Racial Traits</h4>
      <div class="space-y-2">
        <div
          v-for="trait in racialTraits"
          :key="trait.name"
          class="border-l-2 border-secondary pl-3"
        >
          <span class="font-medium">{{ trait.name }}</span>
        </div>
      </div>
    </div>

    <!-- Background Feature -->
    <div v-if="backgroundFeature">
      <h4 class="text-sm font-medium text-muted mb-2">Background Feature</h4>
      <div class="border-l-2 border-accent pl-3">
        <span class="font-medium">{{ backgroundFeature.name }}</span>
      </div>
    </div>
  </div>
</UCard>
```

**Step 2: Add computed properties for features**

```typescript
// Extract features from selected entities
const classFeatures = computed(() => {
  if (!store.selectedClass?.features) return []
  return store.selectedClass.features.filter((f: any) => f.level === 1)
})

const racialTraits = computed(() => {
  if (!store.selectedRace?.traits) return []
  return store.selectedRace.traits
})

const backgroundFeature = computed(() => {
  return store.selectedBackground?.feature ?? null
})
```

**Step 3: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

**Step 4: Commit**

```bash
git add app/components/character/builder/StepReview.vue
git commit -m "feat(StepReview): add features & traits section"
```

---

## Task 16: Save Proficiency Choices on Step Completion

**Files:**
- Modify: `app/pages/characters/[id]/edit.vue`

**Step 1: Add save logic when leaving proficiency step**

Find the next step handler and add:

```typescript
async function handleNextStep() {
  // If leaving proficiency step, save choices
  if (currentStepComponent.value === StepProficiencies) {
    await store.saveProficiencyChoices()
  }

  store.nextStep()
}
```

**Step 2: Run full test suite**

Run: `docker compose exec nuxt npm run test`
Expected: All tests pass

**Step 3: Commit**

```bash
git add app/pages/characters/[id]/edit.vue
git commit -m "feat(edit): save proficiency choices when advancing from step"
```

---

## Task 17: Update CHANGELOG and Final Testing

**Files:**
- Modify: `CHANGELOG.md`

**Step 1: Add changelog entry**

```markdown
- **Character Proficiencies & Features (#106) (2025-12-03)** - Integrate proficiency choices and feature display
  - New conditional "Proficiency Choices" step (Step 6) - only appears when choices are needed
  - Skill selection UI with multi-select checkboxes and validation
  - Review step enhanced with Proficiencies and Features sections
  - 4 new Nitro routes: proficiencies, proficiency-choices (GET/POST), features
  - Store actions: `fetchProficiencyChoices()`, `toggleProficiencySelection()`, `saveProficiencyChoices()`
```

**Step 2: Run full test suite**

Run: `docker compose exec nuxt npm run test`
Expected: All tests pass

**Step 3: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

**Step 4: Run build**

Run: `docker compose exec nuxt npm run build`
Expected: Build succeeds

**Step 5: Commit**

```bash
git add CHANGELOG.md
git commit -m "docs: update CHANGELOG for proficiencies & features (#106)"
```

---

## Task 18: Create Pull Request

**Step 1: Push branch**

```bash
git push -u origin feature/issue-106-character-proficiencies-features
```

**Step 2: Create PR**

```bash
gh pr create --title "feat(character-builder): proficiency choices & features display (#106)" --body "## Summary

- Add conditional 'Proficiency Choices' step to character builder wizard
- Display proficiencies and features on Review step
- 4 new Nitro API routes for backend integration

## Changes

### New Files
- \`server/api/characters/[id]/proficiencies.get.ts\`
- \`server/api/characters/[id]/proficiency-choices.get.ts\`
- \`server/api/characters/[id]/proficiency-choices.post.ts\`
- \`server/api/characters/[id]/features.get.ts\`
- \`app/types/proficiencies.ts\`
- \`app/components/character/builder/StepProficiencies.vue\`
- \`tests/components/character/builder/StepProficiencies.test.ts\`

### Modified Files
- \`app/stores/characterBuilder.ts\` - New state, computed, actions
- \`app/pages/characters/[id]/edit.vue\` - Conditional step routing
- \`app/components/character/builder/StepReview.vue\` - Proficiencies & features sections

## Test Plan

- [x] All unit tests pass
- [x] TypeScript compiles
- [x] Build succeeds
- [ ] Manual: Create Bard (has skill choices) - verify step appears
- [ ] Manual: Create Fighter (no choices) - verify step is skipped
- [ ] Manual: Verify selections persist after save

Closes #106"
```

---

## Summary

| Task | Description | Est. Time |
|------|-------------|-----------|
| 1-4 | Nitro server routes | 15 min |
| 5 | TypeScript types | 5 min |
| 6-7 | Store state and actions | 20 min |
| 8-10 | StepProficiencies component | 30 min |
| 11-12 | Edit page routing updates | 15 min |
| 13 | Validation computed | 10 min |
| 14-15 | Review step enhancements | 20 min |
| 16 | Save on step completion | 5 min |
| 17-18 | Changelog and PR | 10 min |

**Total: ~2.5 hours**
