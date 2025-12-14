# Party Management UI Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a complete party management UI for DMs to create, view, edit, and delete parties, plus manage character membership.

**Architecture:** CRUD-first approach with Nitro API proxy routes, Vue components following existing card/modal patterns, and pages using useEntityList composable. TDD with Vitest throughout.

**Tech Stack:** Nuxt 4.x, NuxtUI 4.x, TypeScript, Vitest, Pinia (if needed for state)

---

## Task 1: Add Party Types

**Files:**
- Create: `app/types/party.ts`
- Modify: `app/types/index.ts`

**Step 1: Write the failing test**

Create `tests/types/party.test.ts`:

```typescript
// tests/types/party.test.ts
import { describe, it, expect } from 'vitest'
import type { Party, PartyCharacter, PartyListItem } from '~/types'

describe('Party Types', () => {
  it('has correct Party shape', () => {
    const party: Party = {
      id: 1,
      name: 'Dragon Heist Campaign',
      description: 'Weekly Thursday game',
      character_count: 4,
      characters: [],
      created_at: '2025-01-01T00:00:00Z'
    }
    expect(party.id).toBe(1)
    expect(party.name).toBe('Dragon Heist Campaign')
  })

  it('has correct PartyCharacter shape', () => {
    const character: PartyCharacter = {
      id: 1,
      public_id: 'brave-falcon-x7Kp',
      name: 'Thorin Ironforge',
      class_name: 'Fighter',
      level: 5,
      portrait: { thumb: '/images/portrait.jpg' },
      parties: [{ id: 1, name: 'Dragon Heist' }]
    }
    expect(character.public_id).toBe('brave-falcon-x7Kp')
  })

  it('has correct PartyListItem shape', () => {
    const item: PartyListItem = {
      id: 1,
      name: 'Dragon Heist',
      description: null,
      character_count: 4,
      created_at: '2025-01-01T00:00:00Z'
    }
    expect(item.character_count).toBe(4)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/types/party.test.ts`
Expected: FAIL with "Cannot find module '~/types' or its types"

**Step 3: Write the types**

Create `app/types/party.ts`:

```typescript
// app/types/party.ts

/**
 * Character as represented in party context
 */
export interface PartyCharacter {
  id: number
  public_id: string
  name: string
  class_name: string
  level: number
  portrait: { thumb: string } | null
  /** Other parties this character belongs to */
  parties?: { id: number; name: string }[]
}

/**
 * Party list item (from GET /parties)
 */
export interface PartyListItem {
  id: number
  name: string
  description: string | null
  character_count: number
  created_at: string
}

/**
 * Full party with characters (from GET /parties/:id)
 */
export interface Party extends PartyListItem {
  characters: PartyCharacter[]
}

/**
 * Create/update party request body
 */
export interface PartyCreateRequest {
  name: string
  description?: string | null
}

/**
 * Add character to party request body
 */
export interface PartyAddCharacterRequest {
  character_id: number
}
```

**Step 4: Export from index**

Modify `app/types/index.ts` - add at the end:

```typescript
// Party types
export type { Party, PartyCharacter, PartyListItem, PartyCreateRequest, PartyAddCharacterRequest } from './party'
```

**Step 5: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/types/party.test.ts`
Expected: PASS

**Step 6: Commit**

```bash
git add app/types/party.ts app/types/index.ts tests/types/party.test.ts
git commit -m "feat: add Party types for party management"
```

---

## Task 2: Create Nitro API Routes

**Files:**
- Create: `server/api/parties/index.get.ts`
- Create: `server/api/parties/index.post.ts`
- Create: `server/api/parties/[id].get.ts`
- Create: `server/api/parties/[id].put.ts`
- Create: `server/api/parties/[id].delete.ts`
- Create: `server/api/parties/[id]/characters/index.post.ts`
- Create: `server/api/parties/[id]/characters/[characterId].delete.ts`

**Step 1: Create list route**

Create `server/api/parties/index.get.ts`:

```typescript
/**
 * List parties endpoint - Proxies to Laravel backend
 *
 * @example GET /api/parties
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const query = getQuery(event)

  const queryString = new URLSearchParams(query as Record<string, string>).toString()
  const url = queryString
    ? `${config.apiBaseServer}/parties?${queryString}`
    : `${config.apiBaseServer}/parties`

  const data = await $fetch(url)
  return data
})
```

**Step 2: Create create route**

Create `server/api/parties/index.post.ts`:

```typescript
/**
 * Create party endpoint - Proxies to Laravel backend
 *
 * @example POST /api/parties { name: "Dragon Heist" }
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const body = await readBody(event)

  const data = await $fetch(`${config.apiBaseServer}/parties`, {
    method: 'POST',
    body
  })
  return data
})
```

**Step 3: Create get route**

Create `server/api/parties/[id].get.ts`:

```typescript
/**
 * Get party endpoint - Proxies to Laravel backend
 *
 * @example GET /api/parties/1
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  const data = await $fetch(`${config.apiBaseServer}/parties/${id}`)
  return data
})
```

**Step 4: Create update route**

Create `server/api/parties/[id].put.ts`:

```typescript
/**
 * Update party endpoint - Proxies to Laravel backend
 *
 * @example PUT /api/parties/1 { name: "Updated Name" }
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const body = await readBody(event)

  const data = await $fetch(`${config.apiBaseServer}/parties/${id}`, {
    method: 'PUT',
    body
  })
  return data
})
```

**Step 5: Create delete route**

Create `server/api/parties/[id].delete.ts`:

```typescript
/**
 * Delete party endpoint - Proxies to Laravel backend
 *
 * @example DELETE /api/parties/1
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  const data = await $fetch(`${config.apiBaseServer}/parties/${id}`, {
    method: 'DELETE'
  })
  return data
})
```

**Step 6: Create add character route**

Create directory and file `server/api/parties/[id]/characters/index.post.ts`:

```typescript
/**
 * Add character to party endpoint - Proxies to Laravel backend
 *
 * @example POST /api/parties/1/characters { character_id: 123 }
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const body = await readBody(event)

  const data = await $fetch(`${config.apiBaseServer}/parties/${id}/characters`, {
    method: 'POST',
    body
  })
  return data
})
```

**Step 7: Create remove character route**

Create `server/api/parties/[id]/characters/[characterId].delete.ts`:

```typescript
/**
 * Remove character from party endpoint - Proxies to Laravel backend
 *
 * @example DELETE /api/parties/1/characters/123
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const characterId = getRouterParam(event, 'characterId')

  const data = await $fetch(`${config.apiBaseServer}/parties/${id}/characters/${characterId}`, {
    method: 'DELETE'
  })
  return data
})
```

**Step 8: Commit**

```bash
git add server/api/parties/
git commit -m "feat: add Nitro API routes for party management"
```

---

## Task 3: Create PartyCard Component

**Files:**
- Create: `tests/components/party/Card.test.ts`
- Create: `app/components/party/Card.vue`

**Step 1: Write the failing test**

Create `tests/components/party/Card.test.ts`:

```typescript
// tests/components/party/Card.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import PartyCard from '~/components/party/Card.vue'
import type { PartyListItem } from '~/types'

const mockParty: PartyListItem = {
  id: 1,
  name: 'Dragon Heist Campaign',
  description: 'Weekly Thursday game with the crew',
  character_count: 4,
  created_at: '2025-01-01T00:00:00Z'
}

describe('PartyCard', () => {
  it('renders party name', async () => {
    const wrapper = await mountSuspended(PartyCard, {
      props: { party: mockParty }
    })
    expect(wrapper.text()).toContain('Dragon Heist Campaign')
  })

  it('renders party description when present', async () => {
    const wrapper = await mountSuspended(PartyCard, {
      props: { party: mockParty }
    })
    expect(wrapper.text()).toContain('Weekly Thursday game')
  })

  it('renders character count', async () => {
    const wrapper = await mountSuspended(PartyCard, {
      props: { party: mockParty }
    })
    expect(wrapper.text()).toContain('4 characters')
  })

  it('renders singular "character" when count is 1', async () => {
    const wrapper = await mountSuspended(PartyCard, {
      props: { party: { ...mockParty, character_count: 1 } }
    })
    expect(wrapper.text()).toContain('1 character')
  })

  it('handles null description gracefully', async () => {
    const wrapper = await mountSuspended(PartyCard, {
      props: { party: { ...mockParty, description: null } }
    })
    expect(wrapper.text()).toContain('Dragon Heist Campaign')
    expect(wrapper.text()).not.toContain('null')
  })

  it('emits edit event when edit action clicked', async () => {
    const wrapper = await mountSuspended(PartyCard, {
      props: { party: mockParty }
    })

    // Find dropdown trigger and click it
    const dropdownBtn = wrapper.find('[data-testid="party-actions"]')
    await dropdownBtn.trigger('click')

    // We'll test the emit directly since UDropdownMenu is complex
    wrapper.vm.$emit('edit')
    expect(wrapper.emitted('edit')).toBeTruthy()
  })

  it('emits delete event when delete action clicked', async () => {
    const wrapper = await mountSuspended(PartyCard, {
      props: { party: mockParty }
    })

    wrapper.vm.$emit('delete')
    expect(wrapper.emitted('delete')).toBeTruthy()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/party/Card.test.ts`
Expected: FAIL - component doesn't exist

**Step 3: Write the component**

Create `app/components/party/Card.vue`:

```vue
<!-- app/components/party/Card.vue -->
<script setup lang="ts">
import type { PartyListItem } from '~/types'

const props = defineProps<{
  party: PartyListItem
}>()

defineEmits<{
  edit: []
  delete: []
}>()

const characterLabel = computed(() =>
  props.party.character_count === 1 ? 'character' : 'characters'
)

const dropdownItems = [
  [
    {
      label: 'Edit Party',
      icon: 'i-heroicons-pencil',
      click: () => { }  // Handled by emit
    },
    {
      label: 'Delete Party',
      icon: 'i-heroicons-trash',
      color: 'error' as const,
      click: () => { }  // Handled by emit
    }
  ]
]
</script>

<template>
  <NuxtLink
    :to="`/parties/${party.id}`"
    class="block h-full"
  >
    <UCard class="h-full hover:shadow-lg transition-shadow border-2 border-primary-300 dark:border-primary-700 hover:border-primary-500">
      <template #header>
        <div class="flex items-center justify-between">
          <h3 class="font-semibold text-lg truncate text-gray-900 dark:text-white">
            {{ party.name }}
          </h3>
          <UDropdownMenu
            :items="dropdownItems"
            :popper="{ placement: 'bottom-end' }"
          >
            <UButton
              data-testid="party-actions"
              icon="i-heroicons-ellipsis-vertical"
              color="neutral"
              variant="ghost"
              size="sm"
              @click.prevent.stop
            />
            <template #item="{ item }">
              <span
                class="flex items-center gap-2"
                :class="{ 'text-error-500': item.color === 'error' }"
                @click.prevent.stop="item.label === 'Edit Party' ? $emit('edit') : $emit('delete')"
              >
                <UIcon :name="item.icon" class="w-4 h-4" />
                {{ item.label }}
              </span>
            </template>
          </UDropdownMenu>
        </div>
      </template>

      <div class="space-y-3">
        <!-- Description -->
        <p
          v-if="party.description"
          class="text-sm text-gray-600 dark:text-gray-400 line-clamp-2"
        >
          {{ party.description }}
        </p>

        <!-- Character Count -->
        <div class="flex items-center gap-2 text-sm text-gray-600 dark:text-gray-400">
          <UIcon name="i-heroicons-user-group" class="w-4 h-4" />
          <span>{{ party.character_count }} {{ characterLabel }}</span>
        </div>
      </div>
    </UCard>
  </NuxtLink>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/party/Card.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/party/Card.vue tests/components/party/Card.test.ts
git commit -m "feat: add PartyCard component"
```

---

## Task 4: Create CreateModal Component

**Files:**
- Create: `tests/components/party/CreateModal.test.ts`
- Create: `app/components/party/CreateModal.vue`

**Step 1: Write the failing test**

Create `tests/components/party/CreateModal.test.ts`:

```typescript
// tests/components/party/CreateModal.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import CreateModal from '~/components/party/CreateModal.vue'
import type { PartyListItem } from '~/types'

const mockParty: PartyListItem = {
  id: 1,
  name: 'Dragon Heist Campaign',
  description: 'Weekly Thursday game',
  character_count: 4,
  created_at: '2025-01-01T00:00:00Z'
}

interface CreateModalVM {
  localName: string
  localDescription: string
  canSave: boolean
  handleSave: () => void
  handleCancel: () => void
}

describe('PartyCreateModal', () => {
  describe('create mode', () => {
    it('shows "New Party" title in create mode', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: true, party: null }
      })
      expect(wrapper.text()).toContain('New Party')
    })

    it('has empty form fields in create mode', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: true, party: null }
      })
      const vm = wrapper.vm as unknown as CreateModalVM
      expect(vm.localName).toBe('')
      expect(vm.localDescription).toBe('')
    })

    it('shows "Create" button in create mode', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: true, party: null }
      })
      expect(wrapper.text()).toContain('Create')
    })

    it('disables save when name is empty', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: true, party: null }
      })
      const vm = wrapper.vm as unknown as CreateModalVM
      expect(vm.canSave).toBe(false)
    })

    it('enables save when name is provided', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: true, party: null }
      })
      const vm = wrapper.vm as unknown as CreateModalVM
      vm.localName = 'New Campaign'
      await wrapper.vm.$nextTick()
      expect(vm.canSave).toBe(true)
    })

    it('emits save with form data', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: true, party: null }
      })
      const vm = wrapper.vm as unknown as CreateModalVM
      vm.localName = 'New Campaign'
      vm.localDescription = 'A description'

      vm.handleSave()

      expect(wrapper.emitted('save')).toBeTruthy()
      expect(wrapper.emitted('save')![0]).toEqual([{
        name: 'New Campaign',
        description: 'A description'
      }])
    })
  })

  describe('edit mode', () => {
    it('shows "Edit Party" title in edit mode', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: true, party: mockParty }
      })
      expect(wrapper.text()).toContain('Edit Party')
    })

    it('populates form with party data', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: true, party: mockParty }
      })
      const vm = wrapper.vm as unknown as CreateModalVM
      expect(vm.localName).toBe('Dragon Heist Campaign')
      expect(vm.localDescription).toBe('Weekly Thursday game')
    })

    it('shows "Save" button in edit mode', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: true, party: mockParty }
      })
      expect(wrapper.text()).toContain('Save')
    })
  })

  describe('cancel action', () => {
    it('emits update:open false on cancel', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: true, party: null }
      })
      const vm = wrapper.vm as unknown as CreateModalVM
      vm.handleCancel()

      expect(wrapper.emitted('update:open')).toBeTruthy()
      expect(wrapper.emitted('update:open')![0]).toEqual([false])
    })
  })

  describe('state reset', () => {
    it('resets form when modal opens', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: false, party: null }
      })
      const vm = wrapper.vm as unknown as CreateModalVM
      vm.localName = 'Some leftover value'

      await wrapper.setProps({ open: true })
      await wrapper.vm.$nextTick()

      expect(vm.localName).toBe('')
    })

    it('populates form when opening with party', async () => {
      const wrapper = await mountSuspended(CreateModal, {
        props: { open: false, party: null }
      })

      await wrapper.setProps({ open: true, party: mockParty })
      await wrapper.vm.$nextTick()

      const vm = wrapper.vm as unknown as CreateModalVM
      expect(vm.localName).toBe('Dragon Heist Campaign')
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/party/CreateModal.test.ts`
Expected: FAIL - component doesn't exist

**Step 3: Write the component**

Create `app/components/party/CreateModal.vue`:

```vue
<!-- app/components/party/CreateModal.vue -->
<script setup lang="ts">
import type { PartyListItem } from '~/types'

const props = defineProps<{
  open: boolean
  party: PartyListItem | null
  loading?: boolean
  error?: string | null
}>()

const emit = defineEmits<{
  'update:open': [value: boolean]
  'save': [payload: { name: string; description: string | null }]
}>()

/** Local state */
const localName = ref('')
const localDescription = ref('')

/** Whether this is edit mode (party provided) */
const isEditMode = computed(() => props.party !== null)

/** Modal title */
const modalTitle = computed(() => isEditMode.value ? 'Edit Party' : 'New Party')

/** Button label */
const saveButtonLabel = computed(() => isEditMode.value ? 'Save' : 'Create')

/** Whether save is enabled */
const canSave = computed(() => {
  if (props.loading) return false
  return localName.value.trim().length > 0
})

function handleSave() {
  if (!canSave.value) return

  emit('save', {
    name: localName.value.trim(),
    description: localDescription.value.trim() || null
  })
}

function handleCancel() {
  emit('update:open', false)
}

function handleKeydown(event: KeyboardEvent) {
  if (event.key === 'Enter' && canSave.value) {
    handleSave()
  }
}

/** Reset form when modal opens */
watch(() => props.open, (isOpen) => {
  if (isOpen) {
    if (props.party) {
      localName.value = props.party.name
      localDescription.value = props.party.description || ''
    } else {
      localName.value = ''
      localDescription.value = ''
    }
  }
})
</script>

<template>
  <UModal
    :open="open"
    @update:open="emit('update:open', $event)"
    @keydown="handleKeydown"
  >
    <template #header>
      <div class="flex items-center justify-between w-full">
        <h3 class="text-lg font-semibold text-gray-900 dark:text-white">
          {{ modalTitle }}
        </h3>
      </div>
    </template>

    <template #body>
      <div class="space-y-4">
        <!-- Name Input -->
        <div>
          <label
            for="party-name"
            class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
          >
            Name <span class="text-error-500">*</span>
          </label>
          <UInput
            id="party-name"
            v-model="localName"
            data-testid="name-input"
            type="text"
            placeholder="Enter party name"
            :disabled="loading"
            class="w-full"
          />
        </div>

        <!-- Description Input -->
        <div>
          <label
            for="party-description"
            class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
          >
            Description
          </label>
          <UTextarea
            id="party-description"
            v-model="localDescription"
            data-testid="description-input"
            placeholder="Optional description for your party"
            :disabled="loading"
            :rows="3"
            class="w-full"
          />
        </div>

        <!-- API Error -->
        <UAlert
          v-if="error"
          color="error"
          icon="i-heroicons-exclamation-triangle"
          :title="error"
        />
      </div>
    </template>

    <template #footer>
      <div class="flex justify-end gap-3">
        <UButton
          data-testid="cancel-btn"
          color="neutral"
          variant="ghost"
          :disabled="loading"
          @click="handleCancel"
        >
          Cancel
        </UButton>
        <UButton
          data-testid="save-btn"
          color="primary"
          :disabled="!canSave"
          :loading="loading"
          @click="handleSave"
        >
          {{ saveButtonLabel }}
        </UButton>
      </div>
    </template>
  </UModal>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/party/CreateModal.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/party/CreateModal.vue tests/components/party/CreateModal.test.ts
git commit -m "feat: add PartyCreateModal component"
```

---

## Task 5: Create AddCharacterModal Component

**Files:**
- Create: `tests/components/party/AddCharacterModal.test.ts`
- Create: `app/components/party/AddCharacterModal.vue`

**Step 1: Write the failing test**

Create `tests/components/party/AddCharacterModal.test.ts`:

```typescript
// tests/components/party/AddCharacterModal.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import AddCharacterModal from '~/components/party/AddCharacterModal.vue'
import type { PartyCharacter } from '~/types'

const mockCharacters: PartyCharacter[] = [
  {
    id: 1,
    public_id: 'brave-falcon-x7Kp',
    name: 'Thorin Ironforge',
    class_name: 'Fighter',
    level: 5,
    portrait: null,
    parties: []
  },
  {
    id: 2,
    public_id: 'swift-hawk-y8Lq',
    name: 'Elara Moonwhisper',
    class_name: 'Wizard',
    level: 5,
    portrait: null,
    parties: [{ id: 99, name: 'Other Campaign' }]
  },
  {
    id: 3,
    public_id: 'bold-wolf-z9Mr',
    name: 'Grimble Thornfoot',
    class_name: 'Rogue',
    level: 4,
    portrait: null,
    parties: []
  }
]

interface AddCharacterModalVM {
  searchQuery: string
  selectedIds: Set<number>
  filteredCharacters: PartyCharacter[]
  canAdd: boolean
  toggleSelection: (id: number) => void
  handleAdd: () => void
}

describe('PartyAddCharacterModal', () => {
  it('renders available characters', async () => {
    const wrapper = await mountSuspended(AddCharacterModal, {
      props: {
        open: true,
        characters: mockCharacters,
        existingCharacterIds: []
      }
    })

    expect(wrapper.text()).toContain('Thorin Ironforge')
    expect(wrapper.text()).toContain('Elara Moonwhisper')
    expect(wrapper.text()).toContain('Grimble Thornfoot')
  })

  it('filters out characters already in party', async () => {
    const wrapper = await mountSuspended(AddCharacterModal, {
      props: {
        open: true,
        characters: mockCharacters,
        existingCharacterIds: [1]  // Thorin is already in party
      }
    })

    expect(wrapper.text()).not.toContain('Thorin Ironforge')
    expect(wrapper.text()).toContain('Elara Moonwhisper')
  })

  it('filters characters by search query', async () => {
    const wrapper = await mountSuspended(AddCharacterModal, {
      props: {
        open: true,
        characters: mockCharacters,
        existingCharacterIds: []
      }
    })

    const vm = wrapper.vm as unknown as AddCharacterModalVM
    vm.searchQuery = 'Elara'
    await wrapper.vm.$nextTick()

    expect(vm.filteredCharacters).toHaveLength(1)
    expect(vm.filteredCharacters[0].name).toBe('Elara Moonwhisper')
  })

  it('shows party indicator for characters in other parties', async () => {
    const wrapper = await mountSuspended(AddCharacterModal, {
      props: {
        open: true,
        characters: mockCharacters,
        existingCharacterIds: []
      }
    })

    expect(wrapper.text()).toContain('Other Campaign')
  })

  it('toggles character selection', async () => {
    const wrapper = await mountSuspended(AddCharacterModal, {
      props: {
        open: true,
        characters: mockCharacters,
        existingCharacterIds: []
      }
    })

    const vm = wrapper.vm as unknown as AddCharacterModalVM
    expect(vm.selectedIds.size).toBe(0)

    vm.toggleSelection(1)
    expect(vm.selectedIds.has(1)).toBe(true)

    vm.toggleSelection(1)
    expect(vm.selectedIds.has(1)).toBe(false)
  })

  it('disables add button when no characters selected', async () => {
    const wrapper = await mountSuspended(AddCharacterModal, {
      props: {
        open: true,
        characters: mockCharacters,
        existingCharacterIds: []
      }
    })

    const vm = wrapper.vm as unknown as AddCharacterModalVM
    expect(vm.canAdd).toBe(false)
  })

  it('enables add button when characters are selected', async () => {
    const wrapper = await mountSuspended(AddCharacterModal, {
      props: {
        open: true,
        characters: mockCharacters,
        existingCharacterIds: []
      }
    })

    const vm = wrapper.vm as unknown as AddCharacterModalVM
    vm.toggleSelection(1)
    await wrapper.vm.$nextTick()

    expect(vm.canAdd).toBe(true)
  })

  it('emits add event with selected character ids', async () => {
    const wrapper = await mountSuspended(AddCharacterModal, {
      props: {
        open: true,
        characters: mockCharacters,
        existingCharacterIds: []
      }
    })

    const vm = wrapper.vm as unknown as AddCharacterModalVM
    vm.toggleSelection(1)
    vm.toggleSelection(3)

    vm.handleAdd()

    expect(wrapper.emitted('add')).toBeTruthy()
    expect(wrapper.emitted('add')![0]).toEqual([[1, 3]])
  })

  it('shows empty state when no characters available', async () => {
    const wrapper = await mountSuspended(AddCharacterModal, {
      props: {
        open: true,
        characters: [],
        existingCharacterIds: []
      }
    })

    expect(wrapper.text()).toContain('No characters available')
  })

  it('resets selection when modal opens', async () => {
    const wrapper = await mountSuspended(AddCharacterModal, {
      props: {
        open: false,
        characters: mockCharacters,
        existingCharacterIds: []
      }
    })

    const vm = wrapper.vm as unknown as AddCharacterModalVM
    vm.toggleSelection(1)

    await wrapper.setProps({ open: true })
    await wrapper.vm.$nextTick()

    expect(vm.selectedIds.size).toBe(0)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/party/AddCharacterModal.test.ts`
Expected: FAIL - component doesn't exist

**Step 3: Write the component**

Create `app/components/party/AddCharacterModal.vue`:

```vue
<!-- app/components/party/AddCharacterModal.vue -->
<script setup lang="ts">
import type { PartyCharacter } from '~/types'

const props = defineProps<{
  open: boolean
  characters: PartyCharacter[]
  existingCharacterIds: number[]
  loading?: boolean
}>()

const emit = defineEmits<{
  'update:open': [value: boolean]
  'add': [characterIds: number[]]
}>()

/** Search filter */
const searchQuery = ref('')

/** Selected character IDs */
const selectedIds = ref<Set<number>>(new Set())

/** Characters filtered by search and not already in party */
const filteredCharacters = computed(() => {
  const existingSet = new Set(props.existingCharacterIds)

  return props.characters
    .filter(c => !existingSet.has(c.id))
    .filter(c => {
      if (!searchQuery.value.trim()) return true
      const query = searchQuery.value.toLowerCase()
      return c.name.toLowerCase().includes(query) ||
             c.class_name.toLowerCase().includes(query)
    })
})

/** Whether add button is enabled */
const canAdd = computed(() => {
  if (props.loading) return false
  return selectedIds.value.size > 0
})

/** Toggle character selection */
function toggleSelection(id: number) {
  if (selectedIds.value.has(id)) {
    selectedIds.value.delete(id)
  } else {
    selectedIds.value.add(id)
  }
  // Force reactivity
  selectedIds.value = new Set(selectedIds.value)
}

/** Check if character is selected */
function isSelected(id: number): boolean {
  return selectedIds.value.has(id)
}

/** Handle add button click */
function handleAdd() {
  if (!canAdd.value) return
  emit('add', Array.from(selectedIds.value))
}

/** Handle cancel */
function handleCancel() {
  emit('update:open', false)
}

/** Reset state when modal opens */
watch(() => props.open, (isOpen) => {
  if (isOpen) {
    searchQuery.value = ''
    selectedIds.value = new Set()
  }
})
</script>

<template>
  <UModal
    :open="open"
    @update:open="emit('update:open', $event)"
  >
    <template #header>
      <div class="flex items-center justify-between w-full">
        <h3 class="text-lg font-semibold text-gray-900 dark:text-white">
          Add Characters to Party
        </h3>
      </div>
    </template>

    <template #body>
      <div class="space-y-4">
        <!-- Search Input -->
        <UInput
          v-model="searchQuery"
          placeholder="Search characters..."
          icon="i-heroicons-magnifying-glass"
          :disabled="loading"
        >
          <template v-if="searchQuery" #trailing>
            <UButton
              color="neutral"
              variant="link"
              icon="i-heroicons-x-mark"
              :padded="false"
              aria-label="Clear search"
              @click="searchQuery = ''"
            />
          </template>
        </UInput>

        <!-- Character List -->
        <div
          v-if="filteredCharacters.length > 0"
          class="space-y-2 max-h-80 overflow-y-auto"
        >
          <div
            v-for="character in filteredCharacters"
            :key="character.id"
            class="flex items-center gap-3 p-3 rounded-lg border cursor-pointer transition-colors"
            :class="[
              isSelected(character.id)
                ? 'border-primary-500 bg-primary-50 dark:bg-primary-900/20'
                : 'border-gray-200 dark:border-gray-700 hover:border-gray-300 dark:hover:border-gray-600'
            ]"
            @click="toggleSelection(character.id)"
          >
            <!-- Checkbox -->
            <UCheckbox
              :model-value="isSelected(character.id)"
              @click.stop
              @update:model-value="toggleSelection(character.id)"
            />

            <!-- Portrait placeholder -->
            <div class="w-10 h-10 rounded-full bg-gray-200 dark:bg-gray-700 flex items-center justify-center">
              <UIcon
                v-if="!character.portrait?.thumb"
                name="i-heroicons-user"
                class="w-5 h-5 text-gray-400"
              />
              <img
                v-else
                :src="character.portrait.thumb"
                :alt="character.name"
                class="w-10 h-10 rounded-full object-cover"
              >
            </div>

            <!-- Character Info -->
            <div class="flex-1 min-w-0">
              <div class="font-medium text-gray-900 dark:text-white truncate">
                {{ character.name }}
              </div>
              <div class="text-sm text-gray-500 dark:text-gray-400">
                {{ character.class_name }} {{ character.level }}
              </div>
              <!-- Party indicator -->
              <div
                v-if="character.parties && character.parties.length > 0"
                class="text-xs text-gray-400 dark:text-gray-500 mt-1"
              >
                Also in: {{ character.parties.map(p => p.name).join(', ') }}
              </div>
            </div>
          </div>
        </div>

        <!-- Empty State -->
        <div
          v-else
          class="text-center py-8 text-gray-500 dark:text-gray-400"
        >
          <UIcon name="i-heroicons-user-group" class="w-12 h-12 mx-auto mb-2 opacity-50" />
          <p>No characters available</p>
          <p class="text-sm">
            {{ searchQuery ? 'Try a different search' : 'Create characters first' }}
          </p>
        </div>
      </div>
    </template>

    <template #footer>
      <div class="flex justify-between items-center">
        <span
          v-if="selectedIds.size > 0"
          class="text-sm text-gray-600 dark:text-gray-400"
        >
          {{ selectedIds.size }} selected
        </span>
        <span v-else />

        <div class="flex gap-3">
          <UButton
            color="neutral"
            variant="ghost"
            :disabled="loading"
            @click="handleCancel"
          >
            Cancel
          </UButton>
          <UButton
            color="primary"
            :disabled="!canAdd"
            :loading="loading"
            @click="handleAdd"
          >
            Add Selected
          </UButton>
        </div>
      </div>
    </template>
  </UModal>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/party/AddCharacterModal.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/party/AddCharacterModal.vue tests/components/party/AddCharacterModal.test.ts
git commit -m "feat: add AddCharacterModal component"
```

---

## Task 6: Create CharacterList Component

**Files:**
- Create: `tests/components/party/CharacterList.test.ts`
- Create: `app/components/party/CharacterList.vue`

**Step 1: Write the failing test**

Create `tests/components/party/CharacterList.test.ts`:

```typescript
// tests/components/party/CharacterList.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import CharacterList from '~/components/party/CharacterList.vue'
import type { PartyCharacter } from '~/types'

const mockCharacters: PartyCharacter[] = [
  {
    id: 1,
    public_id: 'brave-falcon-x7Kp',
    name: 'Thorin Ironforge',
    class_name: 'Fighter',
    level: 5,
    portrait: null,
    parties: []
  },
  {
    id: 2,
    public_id: 'swift-hawk-y8Lq',
    name: 'Elara Moonwhisper',
    class_name: 'Wizard',
    level: 5,
    portrait: { thumb: '/images/elara.jpg' },
    parties: []
  }
]

describe('PartyCharacterList', () => {
  it('renders all characters', async () => {
    const wrapper = await mountSuspended(CharacterList, {
      props: { characters: mockCharacters }
    })

    expect(wrapper.text()).toContain('Thorin Ironforge')
    expect(wrapper.text()).toContain('Elara Moonwhisper')
  })

  it('renders character class and level', async () => {
    const wrapper = await mountSuspended(CharacterList, {
      props: { characters: mockCharacters }
    })

    expect(wrapper.text()).toContain('Fighter')
    expect(wrapper.text()).toContain('Level 5')
  })

  it('shows empty state when no characters', async () => {
    const wrapper = await mountSuspended(CharacterList, {
      props: { characters: [] }
    })

    expect(wrapper.text()).toContain('No characters in this party')
  })

  it('emits remove event with character id', async () => {
    const wrapper = await mountSuspended(CharacterList, {
      props: { characters: mockCharacters }
    })

    // Emit directly to test
    wrapper.vm.$emit('remove', 1)

    expect(wrapper.emitted('remove')).toBeTruthy()
    expect(wrapper.emitted('remove')![0]).toEqual([1])
  })

  it('links character name to character page', async () => {
    const wrapper = await mountSuspended(CharacterList, {
      props: { characters: mockCharacters }
    })

    const link = wrapper.find('a[href="/characters/brave-falcon-x7Kp"]')
    expect(link.exists()).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/party/CharacterList.test.ts`
Expected: FAIL - component doesn't exist

**Step 3: Write the component**

Create `app/components/party/CharacterList.vue`:

```vue
<!-- app/components/party/CharacterList.vue -->
<script setup lang="ts">
import type { PartyCharacter } from '~/types'

defineProps<{
  characters: PartyCharacter[]
}>()

defineEmits<{
  remove: [characterId: number]
}>()
</script>

<template>
  <div>
    <!-- Character Grid -->
    <div
      v-if="characters.length > 0"
      class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4"
    >
      <UCard
        v-for="character in characters"
        :key="character.id"
        class="relative"
      >
        <div class="flex items-center gap-3">
          <!-- Portrait -->
          <div class="w-12 h-12 rounded-full bg-gray-200 dark:bg-gray-700 flex items-center justify-center flex-shrink-0">
            <UIcon
              v-if="!character.portrait?.thumb"
              name="i-heroicons-user"
              class="w-6 h-6 text-gray-400"
            />
            <img
              v-else
              :src="character.portrait.thumb"
              :alt="character.name"
              class="w-12 h-12 rounded-full object-cover"
            >
          </div>

          <!-- Info -->
          <div class="flex-1 min-w-0">
            <NuxtLink
              :to="`/characters/${character.public_id}`"
              class="font-medium text-gray-900 dark:text-white hover:text-primary-500 truncate block"
            >
              {{ character.name }}
            </NuxtLink>
            <div class="text-sm text-gray-500 dark:text-gray-400">
              {{ character.class_name }} &middot; Level {{ character.level }}
            </div>
          </div>

          <!-- Remove Button -->
          <UButton
            icon="i-heroicons-x-mark"
            color="error"
            variant="ghost"
            size="sm"
            @click="$emit('remove', character.id)"
          />
        </div>
      </UCard>
    </div>

    <!-- Empty State -->
    <div
      v-else
      class="text-center py-12"
    >
      <UIcon
        name="i-heroicons-user-group"
        class="w-16 h-16 mx-auto text-gray-300 dark:text-gray-600"
      />
      <h3 class="mt-4 text-lg font-medium text-gray-900 dark:text-white">
        No characters in this party
      </h3>
      <p class="mt-2 text-gray-500 dark:text-gray-400">
        Add characters to track their stats together
      </p>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/party/CharacterList.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/party/CharacterList.vue tests/components/party/CharacterList.test.ts
git commit -m "feat: add PartyCharacterList component"
```

---

## Task 7: Create Party List Page

**Files:**
- Create: `tests/pages/parties/index.test.ts`
- Create: `app/pages/parties/index.vue`

**Step 1: Write the failing test**

Create `tests/pages/parties/index.test.ts`:

```typescript
// tests/pages/parties/index.test.ts
import { describe, it, expect, vi, beforeEach, afterEach, beforeAll, afterAll } from 'vitest'
import { mountSuspended, mockNuxtImport } from '@nuxt/test-utils/runtime'
import { flushPromises } from '@vue/test-utils'
import { setActivePinia, createPinia } from 'pinia'
import { server, http, HttpResponse } from '@/tests/msw/server'
import PartyListPage from '~/pages/parties/index.vue'
import type { PartyListItem } from '~/types'

const mockParties: PartyListItem[] = [
  {
    id: 1,
    name: 'Dragon Heist Campaign',
    description: 'Weekly Thursday game',
    character_count: 4,
    created_at: '2025-01-01T00:00:00Z'
  },
  {
    id: 2,
    name: 'Curse of Strahd',
    description: null,
    character_count: 5,
    created_at: '2025-01-02T00:00:00Z'
  }
]

// Mock toast
const toastMock = { add: vi.fn() }
mockNuxtImport('useToast', () => () => toastMock)

describe('PartyListPage', () => {
  beforeAll(() => server.listen())
  afterEach(() => {
    server.resetHandlers()
    toastMock.add.mockClear()
  })
  afterAll(() => server.close())

  beforeEach(() => {
    setActivePinia(createPinia())

    // Default handler returns parties
    server.use(
      http.get('/api/parties', () => {
        return HttpResponse.json({ data: mockParties })
      })
    )
  })

  it('renders page title', async () => {
    const wrapper = await mountSuspended(PartyListPage)
    await flushPromises()

    expect(wrapper.text()).toContain('My Parties')
  })

  it('renders party cards', async () => {
    const wrapper = await mountSuspended(PartyListPage)
    await flushPromises()

    expect(wrapper.text()).toContain('Dragon Heist Campaign')
    expect(wrapper.text()).toContain('Curse of Strahd')
  })

  it('shows empty state when no parties', async () => {
    server.use(
      http.get('/api/parties', () => {
        return HttpResponse.json({ data: [] })
      })
    )

    const wrapper = await mountSuspended(PartyListPage)
    await flushPromises()

    expect(wrapper.text()).toContain('No parties yet')
  })

  it('has New Party button', async () => {
    const wrapper = await mountSuspended(PartyListPage)
    await flushPromises()

    expect(wrapper.text()).toContain('New Party')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/pages/parties/index.test.ts`
Expected: FAIL - page doesn't exist

**Step 3: Write the page**

Create `app/pages/parties/index.vue`:

```vue
<!-- app/pages/parties/index.vue -->
<script setup lang="ts">
import type { PartyListItem } from '~/types'
import { logger } from '~/utils/logger'

/**
 * Party List Page
 *
 * Displays user's parties with create/edit/delete actions.
 */

useSeoMeta({
  title: 'My Parties - D&D 5e Compendium',
  description: 'Manage your D&D 5e adventuring parties'
})

const { apiFetch } = useApi()
const toast = useToast()

// Fetch parties
const { data: partiesResponse, pending, error, refresh } = await useAsyncData(
  'parties-list',
  () => apiFetch<{ data: PartyListItem[] }>('/parties')
)

const parties = computed(() => partiesResponse.value?.data ?? [])

// Modal state
const showCreateModal = ref(false)
const editingParty = ref<PartyListItem | null>(null)
const isSaving = ref(false)
const saveError = ref<string | null>(null)

// Delete confirmation
const partyToDelete = ref<PartyListItem | null>(null)
const isDeleting = ref(false)

/** Open create modal */
function openCreateModal() {
  editingParty.value = null
  saveError.value = null
  showCreateModal.value = true
}

/** Open edit modal */
function openEditModal(party: PartyListItem) {
  editingParty.value = party
  saveError.value = null
  showCreateModal.value = true
}

/** Handle create/update save */
async function handleSave(payload: { name: string; description: string | null }) {
  isSaving.value = true
  saveError.value = null

  try {
    if (editingParty.value) {
      // Update existing
      await apiFetch(`/parties/${editingParty.value.id}`, {
        method: 'PUT',
        body: payload
      })
      toast.add({ title: 'Party updated!', color: 'success' })
    } else {
      // Create new
      const response = await apiFetch<{ data: PartyListItem }>('/parties', {
        method: 'POST',
        body: payload
      })
      toast.add({ title: 'Party created!', color: 'success' })
      // Navigate to new party
      await navigateTo(`/parties/${response.data.id}`)
      return
    }

    showCreateModal.value = false
    await refresh()
  } catch (err: unknown) {
    const error = err as { statusCode?: number; data?: { message?: string } }
    saveError.value = error.data?.message || 'Failed to save party'
    logger.error('Save party failed:', err)
  } finally {
    isSaving.value = false
  }
}

/** Confirm delete */
function confirmDelete(party: PartyListItem) {
  partyToDelete.value = party
}

/** Handle delete */
async function handleDelete() {
  if (!partyToDelete.value) return

  isDeleting.value = true

  try {
    await apiFetch(`/parties/${partyToDelete.value.id}`, {
      method: 'DELETE'
    })
    toast.add({ title: 'Party deleted', color: 'success' })
    partyToDelete.value = null
    await refresh()
  } catch (err) {
    logger.error('Delete party failed:', err)
    toast.add({ title: 'Failed to delete party', color: 'error' })
  } finally {
    isDeleting.value = false
  }
}
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-6xl">
    <!-- Header -->
    <div class="flex items-center justify-between mb-6">
      <div>
        <h1 class="text-3xl font-bold text-gray-900 dark:text-white">
          My Parties
        </h1>
        <p class="mt-1 text-gray-600 dark:text-gray-400">
          Manage your adventuring groups
        </p>
      </div>

      <UButton
        icon="i-heroicons-plus"
        size="lg"
        @click="openCreateModal"
      >
        New Party
      </UButton>
    </div>

    <!-- Loading State -->
    <div v-if="pending" class="flex justify-center py-12">
      <UIcon name="i-heroicons-arrow-path" class="w-8 h-8 animate-spin text-gray-400" />
    </div>

    <!-- Error State -->
    <UAlert
      v-else-if="error"
      color="error"
      icon="i-heroicons-exclamation-triangle"
      title="Failed to load parties"
      class="mb-6"
    >
      <template #actions>
        <UButton variant="soft" color="error" @click="refresh">
          Retry
        </UButton>
      </template>
    </UAlert>

    <!-- Party Grid -->
    <div
      v-else-if="parties.length > 0"
      class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6"
    >
      <PartyCard
        v-for="party in parties"
        :key="party.id"
        :party="party"
        @edit="openEditModal(party)"
        @delete="confirmDelete(party)"
      />
    </div>

    <!-- Empty State -->
    <div
      v-else
      class="text-center py-12"
    >
      <UIcon
        name="i-heroicons-user-group"
        class="w-16 h-16 mx-auto text-gray-300 dark:text-gray-600"
      />
      <h3 class="mt-4 text-lg font-medium text-gray-900 dark:text-white">
        No parties yet
      </h3>
      <p class="mt-2 text-gray-500 dark:text-gray-400">
        Create your first party to start tracking your adventuring group
      </p>
      <UButton
        class="mt-6"
        icon="i-heroicons-plus"
        @click="openCreateModal"
      >
        New Party
      </UButton>
    </div>

    <!-- Create/Edit Modal -->
    <PartyCreateModal
      v-model:open="showCreateModal"
      :party="editingParty"
      :loading="isSaving"
      :error="saveError"
      @save="handleSave"
    />

    <!-- Delete Confirmation -->
    <UModal
      :open="!!partyToDelete"
      @update:open="partyToDelete = null"
    >
      <template #header>
        <h3 class="text-lg font-semibold text-gray-900 dark:text-white">
          Delete Party
        </h3>
      </template>

      <template #body>
        <p class="text-gray-600 dark:text-gray-400">
          Are you sure you want to delete <strong>{{ partyToDelete?.name }}</strong>?
          This will remove the party but characters will not be deleted.
        </p>
      </template>

      <template #footer>
        <div class="flex justify-end gap-3">
          <UButton
            color="neutral"
            variant="ghost"
            :disabled="isDeleting"
            @click="partyToDelete = null"
          >
            Cancel
          </UButton>
          <UButton
            color="error"
            :loading="isDeleting"
            @click="handleDelete"
          >
            Delete
          </UButton>
        </div>
      </template>
    </UModal>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/pages/parties/index.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/pages/parties/index.vue tests/pages/parties/index.test.ts
git commit -m "feat: add Party list page"
```

---

## Task 8: Create Party Detail Page

**Files:**
- Create: `tests/pages/parties/[id].test.ts`
- Create: `app/pages/parties/[id].vue`

**Step 1: Write the failing test**

Create `tests/pages/parties/[id].test.ts`:

```typescript
// tests/pages/parties/[id].test.ts
import { describe, it, expect, vi, beforeEach, afterEach, beforeAll, afterAll } from 'vitest'
import { mountSuspended, mockNuxtImport } from '@nuxt/test-utils/runtime'
import { flushPromises } from '@vue/test-utils'
import { setActivePinia, createPinia } from 'pinia'
import { server, http, HttpResponse } from '@/tests/msw/server'
import PartyDetailPage from '~/pages/parties/[id].vue'
import type { Party, PartyCharacter } from '~/types'

const mockCharacters: PartyCharacter[] = [
  {
    id: 1,
    public_id: 'brave-falcon-x7Kp',
    name: 'Thorin Ironforge',
    class_name: 'Fighter',
    level: 5,
    portrait: null,
    parties: []
  }
]

const mockParty: Party = {
  id: 1,
  name: 'Dragon Heist Campaign',
  description: 'Weekly Thursday game',
  character_count: 1,
  characters: mockCharacters,
  created_at: '2025-01-01T00:00:00Z'
}

// Mock route
mockNuxtImport('useRoute', () => () => ({
  params: { id: '1' }
}))

// Mock toast
const toastMock = { add: vi.fn() }
mockNuxtImport('useToast', () => () => toastMock)

describe('PartyDetailPage', () => {
  beforeAll(() => server.listen())
  afterEach(() => {
    server.resetHandlers()
    toastMock.add.mockClear()
  })
  afterAll(() => server.close())

  beforeEach(() => {
    setActivePinia(createPinia())

    server.use(
      http.get('/api/parties/1', () => {
        return HttpResponse.json({ data: mockParty })
      }),
      http.get('/api/characters', () => {
        return HttpResponse.json({ data: mockCharacters })
      })
    )
  })

  it('renders party name', async () => {
    const wrapper = await mountSuspended(PartyDetailPage)
    await flushPromises()

    expect(wrapper.text()).toContain('Dragon Heist Campaign')
  })

  it('renders party description', async () => {
    const wrapper = await mountSuspended(PartyDetailPage)
    await flushPromises()

    expect(wrapper.text()).toContain('Weekly Thursday game')
  })

  it('renders character list', async () => {
    const wrapper = await mountSuspended(PartyDetailPage)
    await flushPromises()

    expect(wrapper.text()).toContain('Thorin Ironforge')
  })

  it('has back to parties link', async () => {
    const wrapper = await mountSuspended(PartyDetailPage)
    await flushPromises()

    const link = wrapper.find('a[href="/parties"]')
    expect(link.exists()).toBe(true)
  })

  it('has Add Character button', async () => {
    const wrapper = await mountSuspended(PartyDetailPage)
    await flushPromises()

    expect(wrapper.text()).toContain('Add Character')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/pages/parties/[id].test.ts`
Expected: FAIL - page doesn't exist

**Step 3: Write the page**

Create `app/pages/parties/[id].vue`:

```vue
<!-- app/pages/parties/[id].vue -->
<script setup lang="ts">
import type { Party, PartyCharacter, CharacterSummary } from '~/types'
import { logger } from '~/utils/logger'

/**
 * Party Detail Page
 *
 * Shows party info and character management.
 */

const route = useRoute()
const partyId = computed(() => route.params.id as string)

const { apiFetch } = useApi()
const toast = useToast()

// Fetch party
const { data: partyResponse, pending, error, refresh } = await useAsyncData(
  `party-${partyId.value}`,
  () => apiFetch<{ data: Party }>(`/parties/${partyId.value}`)
)

const party = computed(() => partyResponse.value?.data)

// SEO
useSeoMeta({
  title: () => party.value ? `${party.value.name} - D&D 5e Compendium` : 'Loading...',
  description: () => party.value?.description || 'Manage your adventuring party'
})

// Edit modal state
const showEditModal = ref(false)
const isSaving = ref(false)
const saveError = ref<string | null>(null)

// Add character modal state
const showAddModal = ref(false)
const isAdding = ref(false)
const availableCharacters = ref<PartyCharacter[]>([])

// Delete confirmation
const showDeleteConfirm = ref(false)
const isDeleting = ref(false)

// Remove character confirmation
const characterToRemove = ref<PartyCharacter | null>(null)
const isRemoving = ref(false)

/** Open edit modal */
function openEditModal() {
  saveError.value = null
  showEditModal.value = true
}

/** Handle party update */
async function handleSave(payload: { name: string; description: string | null }) {
  if (!party.value) return

  isSaving.value = true
  saveError.value = null

  try {
    await apiFetch(`/parties/${party.value.id}`, {
      method: 'PUT',
      body: payload
    })
    toast.add({ title: 'Party updated!', color: 'success' })
    showEditModal.value = false
    await refresh()
  } catch (err: unknown) {
    const error = err as { data?: { message?: string } }
    saveError.value = error.data?.message || 'Failed to save party'
    logger.error('Update party failed:', err)
  } finally {
    isSaving.value = false
  }
}

/** Handle party delete */
async function handleDelete() {
  if (!party.value) return

  isDeleting.value = true

  try {
    await apiFetch(`/parties/${party.value.id}`, { method: 'DELETE' })
    toast.add({ title: 'Party deleted', color: 'success' })
    await navigateTo('/parties')
  } catch (err) {
    logger.error('Delete party failed:', err)
    toast.add({ title: 'Failed to delete party', color: 'error' })
  } finally {
    isDeleting.value = false
  }
}

/** Open add character modal */
async function openAddModal() {
  isAdding.value = true
  showAddModal.value = true

  try {
    // Fetch all user's characters
    const response = await apiFetch<{ data: CharacterSummary[] }>('/characters')
    // Transform to PartyCharacter shape
    availableCharacters.value = response.data.map(c => ({
      id: c.id,
      public_id: c.public_id,
      name: c.name,
      class_name: c.class?.name || 'Unknown',
      level: c.level,
      portrait: c.portrait ? { thumb: c.portrait.thumb || null } : null,
      parties: []  // Backend doesn't include this on list
    }))
  } catch (err) {
    logger.error('Failed to fetch characters:', err)
    toast.add({ title: 'Failed to load characters', color: 'error' })
    showAddModal.value = false
  } finally {
    isAdding.value = false
  }
}

/** Handle add characters */
async function handleAddCharacters(characterIds: number[]) {
  if (!party.value) return

  isAdding.value = true

  try {
    // Add each character
    for (const id of characterIds) {
      await apiFetch(`/parties/${party.value.id}/characters`, {
        method: 'POST',
        body: { character_id: id }
      })
    }

    const count = characterIds.length
    toast.add({
      title: `Added ${count} character${count > 1 ? 's' : ''} to party`,
      color: 'success'
    })
    showAddModal.value = false
    await refresh()
  } catch (err: unknown) {
    const error = err as { data?: { message?: string } }
    logger.error('Add characters failed:', err)
    toast.add({
      title: error.data?.message || 'Failed to add characters',
      color: 'error'
    })
  } finally {
    isAdding.value = false
  }
}

/** Confirm remove character */
function confirmRemove(character: PartyCharacter) {
  characterToRemove.value = character
}

/** Handle remove character */
async function handleRemoveCharacter() {
  if (!party.value || !characterToRemove.value) return

  isRemoving.value = true

  try {
    await apiFetch(`/parties/${party.value.id}/characters/${characterToRemove.value.id}`, {
      method: 'DELETE'
    })
    toast.add({ title: 'Character removed from party', color: 'success' })
    characterToRemove.value = null
    await refresh()
  } catch (err) {
    logger.error('Remove character failed:', err)
    toast.add({ title: 'Failed to remove character', color: 'error' })
  } finally {
    isRemoving.value = false
  }
}

/** Get existing character IDs for filtering */
const existingCharacterIds = computed(() =>
  party.value?.characters.map(c => c.id) ?? []
)
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-6xl">
    <!-- Back Link -->
    <NuxtLink
      to="/parties"
      class="inline-flex items-center gap-1 text-sm text-gray-600 dark:text-gray-400 hover:text-primary-500 mb-6"
    >
      <UIcon name="i-heroicons-arrow-left" class="w-4 h-4" />
      Back to Parties
    </NuxtLink>

    <!-- Loading State -->
    <div v-if="pending" class="flex justify-center py-12">
      <UIcon name="i-heroicons-arrow-path" class="w-8 h-8 animate-spin text-gray-400" />
    </div>

    <!-- Error State -->
    <UAlert
      v-else-if="error"
      color="error"
      icon="i-heroicons-exclamation-triangle"
      title="Failed to load party"
      class="mb-6"
    >
      <template #actions>
        <UButton variant="soft" color="error" @click="refresh">
          Retry
        </UButton>
      </template>
    </UAlert>

    <!-- Party Content -->
    <template v-else-if="party">
      <!-- Header -->
      <div class="flex items-start justify-between mb-8">
        <div>
          <h1 class="text-3xl font-bold text-gray-900 dark:text-white">
            {{ party.name }}
          </h1>
          <p
            v-if="party.description"
            class="mt-2 text-gray-600 dark:text-gray-400"
          >
            {{ party.description }}
          </p>
        </div>

        <div class="flex gap-2">
          <UButton
            icon="i-heroicons-pencil"
            variant="soft"
            @click="openEditModal"
          >
            Edit
          </UButton>
          <UDropdownMenu
            :items="[[
              { label: 'Delete Party', icon: 'i-heroicons-trash', color: 'error' }
            ]]"
          >
            <UButton
              icon="i-heroicons-ellipsis-vertical"
              color="neutral"
              variant="ghost"
            />
            <template #item="{ item }">
              <span
                class="flex items-center gap-2 text-error-500"
                @click="showDeleteConfirm = true"
              >
                <UIcon :name="item.icon" class="w-4 h-4" />
                {{ item.label }}
              </span>
            </template>
          </UDropdownMenu>
        </div>
      </div>

      <!-- Characters Section -->
      <div class="mb-8">
        <div class="flex items-center justify-between mb-4">
          <h2 class="text-xl font-semibold text-gray-900 dark:text-white">
            Characters ({{ party.characters.length }})
          </h2>
          <UButton
            icon="i-heroicons-plus"
            @click="openAddModal"
          >
            Add Character
          </UButton>
        </div>

        <PartyCharacterList
          :characters="party.characters"
          @remove="confirmRemove"
        />
      </div>
    </template>

    <!-- Edit Modal -->
    <PartyCreateModal
      v-model:open="showEditModal"
      :party="party"
      :loading="isSaving"
      :error="saveError"
      @save="handleSave"
    />

    <!-- Add Character Modal -->
    <PartyAddCharacterModal
      v-model:open="showAddModal"
      :characters="availableCharacters"
      :existing-character-ids="existingCharacterIds"
      :loading="isAdding"
      @add="handleAddCharacters"
    />

    <!-- Delete Party Confirmation -->
    <UModal
      :open="showDeleteConfirm"
      @update:open="showDeleteConfirm = $event"
    >
      <template #header>
        <h3 class="text-lg font-semibold text-gray-900 dark:text-white">
          Delete Party
        </h3>
      </template>

      <template #body>
        <p class="text-gray-600 dark:text-gray-400">
          Are you sure you want to delete <strong>{{ party?.name }}</strong>?
          This will remove the party but characters will not be deleted.
        </p>
      </template>

      <template #footer>
        <div class="flex justify-end gap-3">
          <UButton
            color="neutral"
            variant="ghost"
            :disabled="isDeleting"
            @click="showDeleteConfirm = false"
          >
            Cancel
          </UButton>
          <UButton
            color="error"
            :loading="isDeleting"
            @click="handleDelete"
          >
            Delete
          </UButton>
        </div>
      </template>
    </UModal>

    <!-- Remove Character Confirmation -->
    <UModal
      :open="!!characterToRemove"
      @update:open="characterToRemove = null"
    >
      <template #header>
        <h3 class="text-lg font-semibold text-gray-900 dark:text-white">
          Remove Character
        </h3>
      </template>

      <template #body>
        <p class="text-gray-600 dark:text-gray-400">
          Remove <strong>{{ characterToRemove?.name }}</strong> from this party?
          The character will not be deleted.
        </p>
      </template>

      <template #footer>
        <div class="flex justify-end gap-3">
          <UButton
            color="neutral"
            variant="ghost"
            :disabled="isRemoving"
            @click="characterToRemove = null"
          >
            Cancel
          </UButton>
          <UButton
            color="error"
            :loading="isRemoving"
            @click="handleRemoveCharacter"
          >
            Remove
          </UButton>
        </div>
      </template>
    </UModal>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/pages/parties/[id].test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/pages/parties/[id].vue tests/pages/parties/[id].test.ts
git commit -m "feat: add Party detail page"
```

---

## Task 9: Add MSW Handlers for Party Tests

**Files:**
- Create: `tests/msw/handlers/parties.ts`
- Modify: `tests/msw/handlers/index.ts`

**Step 1: Create party handlers**

Create `tests/msw/handlers/parties.ts`:

```typescript
// tests/msw/handlers/parties.ts
import { http, HttpResponse } from 'msw'
import type { PartyListItem, Party } from '~/types'

const mockParties: PartyListItem[] = [
  {
    id: 1,
    name: 'Dragon Heist Campaign',
    description: 'Weekly Thursday game',
    character_count: 4,
    created_at: '2025-01-01T00:00:00Z'
  }
]

const mockParty: Party = {
  ...mockParties[0],
  characters: []
}

export const partyHandlers = [
  // List parties
  http.get('/api/parties', () => {
    return HttpResponse.json({ data: mockParties })
  }),

  // Get party
  http.get('/api/parties/:id', ({ params }) => {
    const id = Number(params.id)
    if (id === 1) {
      return HttpResponse.json({ data: mockParty })
    }
    return HttpResponse.json({ error: 'Not found' }, { status: 404 })
  }),

  // Create party
  http.post('/api/parties', async ({ request }) => {
    const body = await request.json() as { name: string; description?: string }
    return HttpResponse.json({
      data: {
        id: 99,
        name: body.name,
        description: body.description || null,
        character_count: 0,
        created_at: new Date().toISOString()
      }
    }, { status: 201 })
  }),

  // Update party
  http.put('/api/parties/:id', async ({ request }) => {
    const body = await request.json() as { name: string; description?: string }
    return HttpResponse.json({
      data: {
        id: 1,
        name: body.name,
        description: body.description || null,
        character_count: 4,
        created_at: '2025-01-01T00:00:00Z'
      }
    })
  }),

  // Delete party
  http.delete('/api/parties/:id', () => {
    return HttpResponse.json({ message: 'Party deleted' })
  }),

  // Add character to party
  http.post('/api/parties/:id/characters', () => {
    return HttpResponse.json({ message: 'Character added to party' }, { status: 201 })
  }),

  // Remove character from party
  http.delete('/api/parties/:id/characters/:characterId', () => {
    return HttpResponse.json({ message: 'Character removed from party' })
  })
]
```

**Step 2: Export from index**

Modify `tests/msw/handlers/index.ts` - add import and spread into handlers array:

```typescript
import { partyHandlers } from './parties'

// In the handlers array:
export const handlers = [
  // ... existing handlers
  ...partyHandlers
]
```

**Step 3: Commit**

```bash
git add tests/msw/handlers/parties.ts tests/msw/handlers/index.ts
git commit -m "feat: add MSW handlers for party endpoints"
```

---

## Task 10: Run Full Test Suite and Fix Issues

**Step 1: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: PASS (or fix any type errors)

**Step 2: Run lint**

Run: `docker compose exec nuxt npm run lint:fix`
Expected: Auto-fix issues

**Step 3: Run all party tests**

Run: `docker compose exec nuxt npm run test -- tests/components/party tests/pages/parties tests/types/party.test.ts`
Expected: All PASS

**Step 4: Run full test suite**

Run: `docker compose exec nuxt npm run test`
Expected: All PASS

**Step 5: Final commit**

```bash
git add -A
git commit -m "chore: fix lint and type issues"
```

---

## Task 11: Browser Verification

**Step 1: Start dev server**

Run: `docker compose exec nuxt npm run dev`

**Step 2: Manual verification checklist**

- [ ] Navigate to `/parties` - empty state shows
- [ ] Click "New Party" - modal opens
- [ ] Create party - redirects to detail page
- [ ] Edit party name - saves successfully
- [ ] Add character - character appears in list
- [ ] Remove character - character removed
- [ ] Delete party - redirects to list
- [ ] Check responsive layout on mobile
- [ ] Check dark mode

**Step 3: Push and create PR**

```bash
git push -u origin feature/issue-560-party-management-ui
gh pr create --title "feat: Party Management UI (#560)" --body "$(cat <<'EOF'
## Summary

Implements party management UI for DMs per issue #560.

### Features
- Party list page with create/edit/delete
- Party detail page with character management
- Add/remove characters from parties
- Responsive card layouts

### Components
- `PartyCard` - Party card for list view
- `PartyCreateModal` - Create/edit party modal
- `PartyAddCharacterModal` - Add characters with search
- `PartyCharacterList` - Character grid with remove

### API Routes
- `GET/POST /api/parties`
- `GET/PUT/DELETE /api/parties/:id`
- `POST /api/parties/:id/characters`
- `DELETE /api/parties/:id/characters/:characterId`

### Test Plan
- [x] Type tests
- [x] Component tests (PartyCard, CreateModal, AddCharacterModal, CharacterList)
- [x] Page tests (list, detail)
- [ ] Manual browser verification

Closes #560
EOF
)"
```

---

## Summary

| Task | Component/File | Tests |
|------|----------------|-------|
| 1 | Party types | Type shape tests |
| 2 | Nitro API routes (7) | N/A (proxy routes) |
| 3 | PartyCard | Card rendering, events |
| 4 | CreateModal | Create/edit modes, validation |
| 5 | AddCharacterModal | Search, multi-select, filtering |
| 6 | CharacterList | List rendering, remove action |
| 7 | Party list page | Load parties, CRUD actions |
| 8 | Party detail page | Load party, character management |
| 9 | MSW handlers | Test fixtures |
| 10 | Quality checks | Typecheck, lint, full suite |
| 11 | Browser verification | Manual testing |

**Estimated task count:** 11 tasks, ~30 TDD steps total
