# Class Resources Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add class resource counters (Rage, Ki Points, Bardic Inspiration, etc.) to the character sheet with spend/restore functionality.

**Architecture:** Three-component structure following existing HitDice pattern - a presentational counter component, a display wrapper, and a manager with API logic. Uses slug-based API endpoint for counter updates.

**Tech Stack:** Vue 3, TypeScript, Vitest, MSW, NuxtUI

**Design Doc:** `docs/frontend/plans/2025-12-15-class-resources-design.md`

---

## Task 1: Create Counter Type Definition

**Files:**
- Modify: `app/types/character.ts`

**Step 1: Add Counter interface**

Add to `app/types/character.ts`:

```typescript
/**
 * Class resource counter (Rage, Ki Points, Bardic Inspiration, etc.)
 * @see Issue #632
 */
export interface Counter {
  id: number
  slug: string
  name: string
  current: number
  max: number
  reset_on: 'short_rest' | 'long_rest' | null
  source: string
  source_type: string
  unlimited: boolean
}
```

**Step 2: Commit**

```bash
git add app/types/character.ts
git commit -m "feat(types): Add Counter interface for class resources #632"
```

---

## Task 2: ClassResourceCounter - Test Setup and Empty State

**Files:**
- Create: `tests/components/character/sheet/ClassResourceCounter.test.ts`
- Create: `app/components/character/sheet/ClassResourceCounter.vue`

**Step 1: Write test file with first tests**

Create `tests/components/character/sheet/ClassResourceCounter.test.ts`:

```typescript
// tests/components/character/sheet/ClassResourceCounter.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import ClassResourceCounter from '~/components/character/sheet/ClassResourceCounter.vue'
import type { Counter } from '~/types/character'

const createCounter = (overrides: Partial<Counter> = {}): Counter => ({
  id: 1,
  slug: 'phb:bard:bardic-inspiration',
  name: 'Bardic Inspiration',
  current: 3,
  max: 5,
  reset_on: 'long_rest',
  source: 'Bard',
  source_type: 'class',
  unlimited: false,
  ...overrides
})

describe('ClassResourceCounter', () => {
  it('displays counter name', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter() }
    })
    expect(wrapper.text()).toContain('Bardic Inspiration')
  })

  it('displays current/max values', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ current: 3, max: 5 }) }
    })
    expect(wrapper.text()).toContain('3')
    expect(wrapper.text()).toContain('5')
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourceCounter.test.ts -v`

Expected: FAIL - component not found

**Step 3: Create minimal component**

Create `app/components/character/sheet/ClassResourceCounter.vue`:

```vue
<!-- app/components/character/sheet/ClassResourceCounter.vue -->
<script setup lang="ts">
import type { Counter } from '~/types/character'

defineProps<{
  counter: Counter
  editable?: boolean
  disabled?: boolean
}>()

defineEmits<{
  spend: [slug: string]
  restore: [slug: string]
}>()
</script>

<template>
  <div class="space-y-1">
    <div class="flex items-center justify-between text-xs">
      <span class="text-gray-700 dark:text-gray-300 font-medium">
        {{ counter.name }}
      </span>
      <span class="text-gray-500 dark:text-gray-400">
        {{ counter.current }}/{{ counter.max }}
      </span>
    </div>
  </div>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourceCounter.test.ts -v`

Expected: PASS

**Step 5: Commit**

```bash
git add tests/components/character/sheet/ClassResourceCounter.test.ts app/components/character/sheet/ClassResourceCounter.vue
git commit -m "feat(character-sheet): Add ClassResourceCounter component skeleton #632"
```

---

## Task 3: ClassResourceCounter - Icon Mode (max <= 6)

**Files:**
- Modify: `tests/components/character/sheet/ClassResourceCounter.test.ts`
- Modify: `app/components/character/sheet/ClassResourceCounter.vue`

**Step 1: Add icon mode tests**

Add to test file:

```typescript
describe('Icon Mode (max <= 6)', () => {
  it('shows icons when max <= 6', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ max: 5 }) }
    })
    const icons = wrapper.findAll('[data-testid^="counter-icon"]')
    expect(icons.length).toBe(5)
  })

  it('shows filled icons for current value', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ current: 3, max: 5 }) }
    })
    const filled = wrapper.findAll('[data-testid="counter-icon-filled"]')
    const empty = wrapper.findAll('[data-testid="counter-icon-empty"]')
    expect(filled.length).toBe(3)
    expect(empty.length).toBe(2)
  })

  it('shows all empty icons when depleted', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ current: 0, max: 4 }) }
    })
    const filled = wrapper.findAll('[data-testid="counter-icon-filled"]')
    const empty = wrapper.findAll('[data-testid="counter-icon-empty"]')
    expect(filled.length).toBe(0)
    expect(empty.length).toBe(4)
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourceCounter.test.ts -v`

Expected: FAIL - icons not found

**Step 3: Implement icon mode**

Update component template:

```vue
<template>
  <div class="space-y-1">
    <div class="flex items-center justify-between text-xs">
      <span class="text-gray-700 dark:text-gray-300 font-medium">
        {{ counter.name }}
      </span>
      <span class="text-gray-500 dark:text-gray-400">
        {{ counter.current }}/{{ counter.max }}
      </span>
    </div>

    <!-- Icon Mode (max <= 6) -->
    <div
      v-if="counter.max <= 6"
      class="flex gap-1 flex-wrap"
    >
      <UIcon
        v-for="i in counter.current"
        :key="`filled-${i}`"
        name="i-heroicons-bolt-solid"
        class="w-5 h-5 text-primary-600 dark:text-primary-500"
        data-testid="counter-icon-filled"
      />
      <UIcon
        v-for="i in (counter.max - counter.current)"
        :key="`empty-${i}`"
        name="i-heroicons-bolt"
        class="w-5 h-5 text-gray-300 dark:text-gray-600"
        data-testid="counter-icon-empty"
      />
    </div>
  </div>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourceCounter.test.ts -v`

Expected: PASS

**Step 5: Commit**

```bash
git add tests/components/character/sheet/ClassResourceCounter.test.ts app/components/character/sheet/ClassResourceCounter.vue
git commit -m "feat(character-sheet): Add icon mode for ClassResourceCounter #632"
```

---

## Task 4: ClassResourceCounter - Numeric Mode (max > 6)

**Files:**
- Modify: `tests/components/character/sheet/ClassResourceCounter.test.ts`
- Modify: `app/components/character/sheet/ClassResourceCounter.vue`

**Step 1: Add numeric mode tests**

Add to test file:

```typescript
describe('Numeric Mode (max > 6)', () => {
  it('shows numeric display when max > 6', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ current: 12, max: 15 }) }
    })
    // Should NOT show icons
    const icons = wrapper.findAll('[data-testid^="counter-icon"]')
    expect(icons.length).toBe(0)
    // Should show +/- buttons
    expect(wrapper.find('[data-testid="counter-decrement"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="counter-increment"]').exists()).toBe(true)
  })

  it('shows +/- buttons in numeric mode', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ max: 20 }), editable: true }
    })
    expect(wrapper.find('[data-testid="counter-decrement"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="counter-increment"]').exists()).toBe(true)
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourceCounter.test.ts -v`

Expected: FAIL

**Step 3: Implement numeric mode**

Update component template to add numeric mode after icon mode:

```vue
    <!-- Numeric Mode (max > 6) -->
    <div
      v-else
      class="flex items-center gap-2"
    >
      <UButton
        data-testid="counter-decrement"
        icon="i-heroicons-minus"
        color="neutral"
        variant="soft"
        size="xs"
        :disabled="!editable || disabled || counter.current <= 0"
        @click="$emit('spend', counter.slug)"
      />
      <span class="text-sm font-medium text-gray-700 dark:text-gray-300 min-w-12 text-center">
        {{ counter.current }} / {{ counter.max }}
      </span>
      <UButton
        data-testid="counter-increment"
        icon="i-heroicons-plus"
        color="neutral"
        variant="soft"
        size="xs"
        :disabled="!editable || disabled || counter.current >= counter.max"
        @click="$emit('restore', counter.slug)"
      />
    </div>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourceCounter.test.ts -v`

Expected: PASS

**Step 5: Commit**

```bash
git add tests/components/character/sheet/ClassResourceCounter.test.ts app/components/character/sheet/ClassResourceCounter.vue
git commit -m "feat(character-sheet): Add numeric mode for ClassResourceCounter #632"
```

---

## Task 5: ClassResourceCounter - Icon Click Interactions

**Files:**
- Modify: `tests/components/character/sheet/ClassResourceCounter.test.ts`
- Modify: `app/components/character/sheet/ClassResourceCounter.vue`

**Step 1: Add interaction tests**

Add to test file:

```typescript
describe('Icon Interactions', () => {
  it('emits spend when filled icon clicked in editable mode', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ current: 3, max: 5 }), editable: true }
    })
    const filled = wrapper.find('[data-testid="counter-icon-filled"]')
    await filled.trigger('click')
    expect(wrapper.emitted('spend')).toBeTruthy()
    expect(wrapper.emitted('spend')![0]).toEqual(['phb:bard:bardic-inspiration'])
  })

  it('does not emit spend when not editable', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ current: 3, max: 5 }), editable: false }
    })
    const filled = wrapper.find('[data-testid="counter-icon-filled"]')
    await filled.trigger('click')
    expect(wrapper.emitted('spend')).toBeFalsy()
  })

  it('does not emit spend when disabled', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ current: 3, max: 5 }), editable: true, disabled: true }
    })
    const filled = wrapper.find('[data-testid="counter-icon-filled"]')
    await filled.trigger('click')
    expect(wrapper.emitted('spend')).toBeFalsy()
  })

  it('does not emit spend when counter is at 0', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ current: 0, max: 5 }), editable: true }
    })
    // No filled icons to click when current is 0
    const filled = wrapper.findAll('[data-testid="counter-icon-filled"]')
    expect(filled.length).toBe(0)
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourceCounter.test.ts -v`

Expected: FAIL - emit not happening

**Step 3: Add click handlers to icons**

Update script and template:

```vue
<script setup lang="ts">
import type { Counter } from '~/types/character'

const props = defineProps<{
  counter: Counter
  editable?: boolean
  disabled?: boolean
}>()

const emit = defineEmits<{
  spend: [slug: string]
  restore: [slug: string]
}>()

const isInteractive = computed(() => props.editable && !props.disabled)

function handleIconClick() {
  if (!isInteractive.value) return
  if (props.counter.current <= 0) return
  emit('spend', props.counter.slug)
}
</script>
```

Update filled icon in template:

```vue
      <UIcon
        v-for="i in counter.current"
        :key="`filled-${i}`"
        name="i-heroicons-bolt-solid"
        :class="[
          'w-5 h-5 text-primary-600 dark:text-primary-500',
          isInteractive ? 'cursor-pointer hover:opacity-80 transition-opacity' : ''
        ]"
        data-testid="counter-icon-filled"
        @click="handleIconClick"
      />
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourceCounter.test.ts -v`

Expected: PASS

**Step 5: Commit**

```bash
git add tests/components/character/sheet/ClassResourceCounter.test.ts app/components/character/sheet/ClassResourceCounter.vue
git commit -m "feat(character-sheet): Add click interactions for ClassResourceCounter icons #632"
```

---

## Task 6: ClassResourceCounter - Reset Badge

**Files:**
- Modify: `tests/components/character/sheet/ClassResourceCounter.test.ts`
- Modify: `app/components/character/sheet/ClassResourceCounter.vue`

**Step 1: Add reset badge tests**

Add to test file:

```typescript
describe('Reset Badge', () => {
  it('shows "Long" badge for long_rest reset', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ reset_on: 'long_rest' }) }
    })
    expect(wrapper.text()).toContain('Long')
  })

  it('shows "Short" badge for short_rest reset', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ reset_on: 'short_rest' }) }
    })
    expect(wrapper.text()).toContain('Short')
  })

  it('shows no badge when reset_on is null', async () => {
    const wrapper = await mountSuspended(ClassResourceCounter, {
      props: { counter: createCounter({ reset_on: null }) }
    })
    expect(wrapper.find('[data-testid="reset-badge"]').exists()).toBe(false)
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourceCounter.test.ts -v`

Expected: FAIL

**Step 3: Add reset badge**

Add computed and template element:

```vue
<script setup lang="ts">
// ... existing code ...

const resetLabel = computed(() => {
  if (props.counter.reset_on === 'short_rest') return 'Short'
  if (props.counter.reset_on === 'long_rest') return 'Long'
  return null
})
</script>
```

Update the header row in template:

```vue
    <div class="flex items-center justify-between text-xs">
      <span class="text-gray-700 dark:text-gray-300 font-medium">
        {{ counter.name }}
      </span>
      <div class="flex items-center gap-2">
        <UBadge
          v-if="resetLabel"
          data-testid="reset-badge"
          color="neutral"
          variant="subtle"
          size="xs"
        >
          <UIcon name="i-heroicons-arrow-path" class="w-3 h-3 mr-0.5" />
          {{ resetLabel }}
        </UBadge>
        <span class="text-gray-500 dark:text-gray-400">
          {{ counter.current }}/{{ counter.max }}
        </span>
      </div>
    </div>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourceCounter.test.ts -v`

Expected: PASS

**Step 5: Commit**

```bash
git add tests/components/character/sheet/ClassResourceCounter.test.ts app/components/character/sheet/ClassResourceCounter.vue
git commit -m "feat(character-sheet): Add reset badge to ClassResourceCounter #632"
```

---

## Task 7: ClassResources - Display Component

**Files:**
- Create: `tests/components/character/sheet/ClassResources.test.ts`
- Create: `app/components/character/sheet/ClassResources.vue`

**Step 1: Write tests**

Create `tests/components/character/sheet/ClassResources.test.ts`:

```typescript
// tests/components/character/sheet/ClassResources.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import ClassResources from '~/components/character/sheet/ClassResources.vue'
import type { Counter } from '~/types/character'

const createCounter = (overrides: Partial<Counter> = {}): Counter => ({
  id: 1,
  slug: 'phb:bard:bardic-inspiration',
  name: 'Bardic Inspiration',
  current: 3,
  max: 5,
  reset_on: 'long_rest',
  source: 'Bard',
  source_type: 'class',
  unlimited: false,
  ...overrides
})

describe('ClassResources', () => {
  it('displays title', async () => {
    const wrapper = await mountSuspended(ClassResources, {
      props: { counters: [createCounter()] }
    })
    expect(wrapper.text()).toContain('Class Resources')
  })

  it('renders nothing when counters array is empty', async () => {
    const wrapper = await mountSuspended(ClassResources, {
      props: { counters: [] }
    })
    expect(wrapper.text()).not.toContain('Class Resources')
  })

  it('renders each counter', async () => {
    const counters = [
      createCounter({ name: 'Bardic Inspiration', slug: 'phb:bard:bardic-inspiration' }),
      createCounter({ id: 2, name: 'Ki Points', slug: 'phb:monk:ki', max: 10 })
    ]
    const wrapper = await mountSuspended(ClassResources, {
      props: { counters }
    })
    expect(wrapper.text()).toContain('Bardic Inspiration')
    expect(wrapper.text()).toContain('Ki Points')
  })

  it('emits spend event from child counter', async () => {
    const wrapper = await mountSuspended(ClassResources, {
      props: { counters: [createCounter()], editable: true }
    })
    const icon = wrapper.find('[data-testid="counter-icon-filled"]')
    await icon.trigger('click')
    expect(wrapper.emitted('spend')).toBeTruthy()
    expect(wrapper.emitted('spend')![0]).toEqual(['phb:bard:bardic-inspiration'])
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResources.test.ts -v`

Expected: FAIL

**Step 3: Create component**

Create `app/components/character/sheet/ClassResources.vue`:

```vue
<!-- app/components/character/sheet/ClassResources.vue -->
<script setup lang="ts">
/**
 * Class Resources Display Component
 *
 * Displays class-specific resource counters (Rage, Ki, Bardic Inspiration, etc.)
 * Renders nothing if counters array is empty.
 *
 * @see Issue #632
 */
import type { Counter } from '~/types/character'

defineProps<{
  counters: Counter[]
  editable?: boolean
  disabled?: boolean
}>()

const emit = defineEmits<{
  spend: [slug: string]
  restore: [slug: string]
}>()
</script>

<template>
  <div
    v-if="counters.length > 0"
    class="space-y-3 p-4 bg-gray-50 dark:bg-gray-800 rounded-lg"
  >
    <div class="text-sm font-bold text-gray-700 dark:text-gray-300 text-center">
      Class Resources
    </div>

    <div class="space-y-3">
      <CharacterSheetClassResourceCounter
        v-for="counter in counters"
        :key="counter.slug"
        :counter="counter"
        :editable="editable"
        :disabled="disabled"
        @spend="slug => emit('spend', slug)"
        @restore="slug => emit('restore', slug)"
      />
    </div>
  </div>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResources.test.ts -v`

Expected: PASS

**Step 5: Commit**

```bash
git add tests/components/character/sheet/ClassResources.test.ts app/components/character/sheet/ClassResources.vue
git commit -m "feat(character-sheet): Add ClassResources display component #632"
```

---

## Task 8: Nitro API Route

**Files:**
- Create: `server/api/characters/[id]/counters/[slug].patch.ts`

**Step 1: Create Nitro route**

Create `server/api/characters/[id]/counters/[slug].patch.ts`:

```typescript
/**
 * Counter update endpoint - Proxies to Laravel backend
 *
 * Updates counter usage via action:
 * - "use": Decrements current (spends one use)
 * - "restore": Increments current (recovers one use)
 * - "reset": Sets current to max
 *
 * @example PATCH /api/characters/1/counters/phb:bard:bardic-inspiration { "action": "use" }
 *
 * @see #632 - Class resources
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const slug = getRouterParam(event, 'slug')
  const body = await readBody(event)

  try {
    const data = await $fetch(
      `${config.apiBaseServer}/characters/${id}/counters/${slug}`,
      { method: 'PATCH', body }
    )
    return data
  } catch (error: unknown) {
    const err = error as { statusCode?: number, statusMessage?: string, data?: unknown }
    throw createError({
      statusCode: err.statusCode || 500,
      statusMessage: err.statusMessage || 'Failed to update counter',
      data: err.data
    })
  }
})
```

**Step 2: Commit**

```bash
git add server/api/characters/[id]/counters/[slug].patch.ts
git commit -m "feat(api): Add counter update Nitro route #632"
```

---

## Task 9: ClassResourcesManager - Setup and API Integration

**Files:**
- Create: `tests/components/character/sheet/ClassResourcesManager.test.ts`
- Create: `app/components/character/sheet/ClassResourcesManager.vue`

**Step 1: Write tests with MSW**

Create `tests/components/character/sheet/ClassResourcesManager.test.ts`:

```typescript
// tests/components/character/sheet/ClassResourcesManager.test.ts
import { describe, it, expect, vi, beforeEach, beforeAll, afterEach, afterAll } from 'vitest'
import { mountSuspended, mockNuxtImport } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import { http, HttpResponse } from 'msw'
import { setupServer } from 'msw/node'
import ClassResourcesManager from '~/components/character/sheet/ClassResourcesManager.vue'
import type { Counter } from '~/types/character'

const createCounter = (overrides: Partial<Counter> = {}): Counter => ({
  id: 1,
  slug: 'phb:bard:bardic-inspiration',
  name: 'Bardic Inspiration',
  current: 3,
  max: 5,
  reset_on: 'long_rest',
  source: 'Bard',
  source_type: 'class',
  unlimited: false,
  ...overrides
})

// MSW Server
const server = setupServer(
  http.patch('/api/characters/:id/counters/:slug', async ({ params }) => {
    return HttpResponse.json({
      data: createCounter({ current: 2 }) // Return decremented
    })
  })
)

// Toast mock
const toastMock = { add: vi.fn() }
mockNuxtImport('useToast', () => () => toastMock)

describe('ClassResourcesManager', () => {
  beforeAll(() => server.listen())
  afterEach(() => {
    server.resetHandlers()
    toastMock.add.mockClear()
  })
  afterAll(() => server.close())

  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders ClassResources with counters', async () => {
    const wrapper = await mountSuspended(ClassResourcesManager, {
      props: {
        counters: [createCounter()],
        characterId: 1
      }
    })
    expect(wrapper.text()).toContain('Bardic Inspiration')
  })

  it('calls API when spend event received', async () => {
    let apiCalled = false
    server.use(
      http.patch('/api/characters/:id/counters/:slug', () => {
        apiCalled = true
        return HttpResponse.json({ data: createCounter({ current: 2 }) })
      })
    )

    const wrapper = await mountSuspended(ClassResourcesManager, {
      props: {
        counters: [createCounter()],
        characterId: 1,
        editable: true
      }
    })

    const icon = wrapper.find('[data-testid="counter-icon-filled"]')
    await icon.trigger('click')

    // Wait for API call
    await new Promise(resolve => setTimeout(resolve, 50))
    expect(apiCalled).toBe(true)
  })

  it('shows error toast on API failure', async () => {
    server.use(
      http.patch('/api/characters/:id/counters/:slug', () => {
        return HttpResponse.json({ message: 'No uses remaining' }, { status: 422 })
      })
    )

    const wrapper = await mountSuspended(ClassResourcesManager, {
      props: {
        counters: [createCounter()],
        characterId: 1,
        editable: true
      }
    })

    const icon = wrapper.find('[data-testid="counter-icon-filled"]')
    await icon.trigger('click')

    await new Promise(resolve => setTimeout(resolve, 50))
    expect(toastMock.add).toHaveBeenCalledWith(
      expect.objectContaining({ color: 'error' })
    )
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourcesManager.test.ts -v`

Expected: FAIL

**Step 3: Create manager component**

Create `app/components/character/sheet/ClassResourcesManager.vue`:

```vue
<!-- app/components/character/sheet/ClassResourcesManager.vue -->
<script setup lang="ts">
/**
 * Class Resources Manager Component
 *
 * Handles API calls for spending/restoring class resources.
 * Uses optimistic updates with rollback on error.
 *
 * @see Issue #632
 */
import type { Counter } from '~/types/character'

const props = defineProps<{
  counters: Counter[]
  characterId: number
  editable?: boolean
  isDead?: boolean
}>()

const { apiFetch } = useApi()
const toast = useToast()

// Local reactive copy for optimistic updates
const localCounters = ref<Counter[]>([...props.counters])

// Sync when props change
watch(() => props.counters, (newCounters) => {
  localCounters.value = [...newCounters]
}, { deep: true })

const isDisabled = computed(() => props.isDead)

/**
 * Spend a counter use (decrement)
 */
async function handleSpend(slug: string) {
  const counter = localCounters.value.find(c => c.slug === slug)
  if (!counter || counter.current <= 0) return

  // Optimistic update
  counter.current--

  try {
    await apiFetch(`/characters/${props.characterId}/counters/${encodeURIComponent(slug)}`, {
      method: 'PATCH',
      body: { action: 'use' }
    })
  } catch (error: unknown) {
    // Rollback
    counter.current++
    const err = error as { data?: { message?: string } }
    toast.add({
      title: err.data?.message || 'Failed to update counter',
      color: 'error'
    })
  }
}

/**
 * Restore a counter use (increment)
 */
async function handleRestore(slug: string) {
  const counter = localCounters.value.find(c => c.slug === slug)
  if (!counter || counter.current >= counter.max) return

  // Optimistic update
  counter.current++

  try {
    await apiFetch(`/characters/${props.characterId}/counters/${encodeURIComponent(slug)}`, {
      method: 'PATCH',
      body: { action: 'restore' }
    })
  } catch (error: unknown) {
    // Rollback
    counter.current--
    const err = error as { data?: { message?: string } }
    toast.add({
      title: err.data?.message || 'Failed to update counter',
      color: 'error'
    })
  }
}
</script>

<template>
  <CharacterSheetClassResources
    :counters="localCounters"
    :editable="editable"
    :disabled="isDisabled"
    @spend="handleSpend"
    @restore="handleRestore"
  />
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/character/sheet/ClassResourcesManager.test.ts -v`

Expected: PASS

**Step 5: Commit**

```bash
git add tests/components/character/sheet/ClassResourcesManager.test.ts app/components/character/sheet/ClassResourcesManager.vue
git commit -m "feat(character-sheet): Add ClassResourcesManager with API integration #632"
```

---

## Task 10: Page Integration

**Files:**
- Modify: `app/pages/characters/[publicId]/index.vue`

**Step 1: Add ClassResourcesManager to page**

In `app/pages/characters/[publicId]/index.vue`, add the manager component in the left sidebar after HitDiceManager.

Find this section (around line 230-240):

```vue
          <CharacterSheetHitDiceManager
            v-if="hitDice.length && character"
            :hit-dice="hitDice"
            :character-id="character.id"
            :editable="canEdit"
            :initial-is-dead="character.is_dead"
            @refresh-hit-dice="refreshHitDice"
            @refresh-short-rest="refreshForShortRest"
            @refresh-long-rest="refreshForLongRest"
          />
```

Add after it:

```vue
          <CharacterSheetClassResourcesManager
            v-if="character.counters?.length"
            :counters="character.counters"
            :character-id="character.id"
            :editable="canEdit"
            :is-dead="character.is_dead"
          />
```

**Step 2: Run character test suite**

Run: `docker compose exec nuxt npm run test:character`

Expected: All tests pass

**Step 3: Manual verification**

1. Start dev server: `docker compose exec nuxt npm run dev`
2. Navigate to a character with class resources (e.g., character 1 - Bard with Bardic Inspiration)
3. Verify:
   - Class Resources panel appears below Hit Dice
   - Counters display correctly
   - Enable play mode
   - Click to spend works
   - Counter decrements
   - Toast shows on error (if applicable)

**Step 4: Commit**

```bash
git add app/pages/characters/[publicId]/index.vue
git commit -m "feat(character-sheet): Integrate ClassResourcesManager into character page #632"
```

---

## Task 11: Final Verification and Cleanup

**Step 1: Run full test suite**

Run: `docker compose exec nuxt npm run test`

Expected: All tests pass

**Step 2: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No errors

**Step 3: Run lint**

Run: `docker compose exec nuxt npm run lint:fix`

Expected: No errors (or auto-fixed)

**Step 4: Update CHANGELOG**

Add to `CHANGELOG.md` under `## [Unreleased]`:

```markdown
### Added
- Class resource counters on character sheet (Rage, Ki Points, Bardic Inspiration, etc.) (#632)
  - Icon mode for resources with max ≤ 6
  - Numeric mode with +/- buttons for larger resources
  - Reset indicator badges (Short/Long rest)
  - Optimistic updates with error handling
```

**Step 5: Final commit**

```bash
git add CHANGELOG.md
git commit -m "docs: Update CHANGELOG for class resources #632"
```

---

## Summary

| Task | Description | Files |
|------|-------------|-------|
| 1 | Counter type definition | `app/types/character.ts` |
| 2 | ClassResourceCounter skeleton | Component + test |
| 3 | Icon mode (max ≤ 6) | Component + test |
| 4 | Numeric mode (max > 6) | Component + test |
| 5 | Icon click interactions | Component + test |
| 6 | Reset badge | Component + test |
| 7 | ClassResources display | Component + test |
| 8 | Nitro API route | `server/api/.../[slug].patch.ts` |
| 9 | ClassResourcesManager | Component + test |
| 10 | Page integration | `app/pages/characters/[publicId]/index.vue` |
| 11 | Final verification | Tests, typecheck, lint, CHANGELOG |

**Total commits:** 12
**Estimated time:** 60-90 minutes (TDD pace)
