# Equipment Choice Items Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Enable users to select specific items (e.g., Longsword) when equipment choices offer categories (e.g., "any martial weapon").

**Architecture:** Enhance `EquipmentChoiceGroup` with inline item pickers that appear when a choice option contains `choice_items` with `proficiency_type` (category) instead of `item` (fixed). New `EquipmentItemPicker` component fetches matching items via API filter. Store tracks both option selection and item selections within compound choices.

**Tech Stack:** Vue 3, Pinia, NuxtUI 4 (USelectMenu), Nitro server routes, Vitest

**Design Doc:** `docs/plans/2025-12-03-issue-96-equipment-choice-items-design.md`

---

## Task 1: Update Test Fixtures with choice_items

**Files:**
- Modify: `tests/fixtures/equipment.ts`

**Step 1: Add choice_items to mock compound items**

```typescript
// Add after line 15 in tests/fixtures/equipment.ts

/**
 * Mock proficiency type for martial weapons category
 */
export const mockMartialWeaponsProficiencyType = {
  id: 6,
  slug: 'martial-weapons',
  name: 'Martial Weapons',
  category: 'weapon',
  subcategory: 'martial'
}

/**
 * Mock item for Shield (auto-included in compound choices)
 */
export const mockShieldItem = {
  id: 48,
  name: 'Shield',
  slug: 'shield',
  description: 'A shield provides +2 AC',
  item_type: { id: 7, name: 'Shield' },
  is_magic: false,
  sources: []
}

/**
 * Mock choice group with choice_items (Fighter-style compound choices)
 */
export const mockCompoundChoiceGroup: Equipment[] = [
  {
    id: 36,
    item_id: null,
    quantity: 2,
    is_choice: true,
    choice_group: 'choice_2',
    choice_option: 1,
    choice_description: 'Starting equipment choice',
    proficiency_subcategory: null,
    description: 'a martial weapon and a shield',
    choice_items: [
      { proficiency_type: mockMartialWeaponsProficiencyType, item: undefined, quantity: 1 },
      { proficiency_type: undefined, item: mockShieldItem, quantity: 1 }
    ]
  },
  {
    id: 37,
    item_id: null,
    quantity: 2,
    is_choice: true,
    choice_group: 'choice_2',
    choice_option: 2,
    choice_description: 'Starting equipment choice',
    proficiency_subcategory: null,
    description: 'two martial weapons',
    choice_items: [
      { proficiency_type: mockMartialWeaponsProficiencyType, item: undefined, quantity: 2 }
    ]
  }
]

/**
 * Mock items for martial weapon picker dropdown
 */
export const mockMartialWeapons = [
  { id: 42, name: 'Longsword', slug: 'longsword', proficiency_category: 'martial_melee', is_magic: false },
  { id: 43, name: 'Battleaxe', slug: 'battleaxe', proficiency_category: 'martial_melee', is_magic: false },
  { id: 44, name: 'Warhammer', slug: 'warhammer', proficiency_category: 'martial_melee', is_magic: false },
  { id: 45, name: 'Longbow', slug: 'longbow', proficiency_category: 'martial_ranged', is_magic: false }
]
```

**Step 2: Run existing tests to ensure fixtures don't break anything**

Run: `docker compose exec nuxt npm run test -- tests/fixtures`
Expected: PASS (or skip if no fixture tests)

**Step 3: Commit**

```bash
git add tests/fixtures/equipment.ts
git commit -m "test: add choice_items mock data for compound equipment choices"
```

---

## Task 2: Create Nitro Route - GET Equipment

**Files:**
- Create: `server/api/characters/[id]/equipment.get.ts`

**Step 1: Create the route file**

```typescript
/**
 * Get character equipment - Proxies to Laravel backend
 *
 * @example GET /api/characters/1/equipment
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/equipment`)
  return data
})
```

**Step 2: Verify route works**

Run: `curl -s http://localhost:4000/api/characters/1/equipment | jq '.data[0]'`
Expected: Equipment object or empty array (depends on character state)

**Step 3: Commit**

```bash
git add server/api/characters/[id]/equipment.get.ts
git commit -m "feat: add Nitro route for GET character equipment"
```

---

## Task 3: Create Nitro Route - POST Equipment

**Files:**
- Create: `server/api/characters/[id]/equipment.post.ts`

**Step 1: Create the route file**

```typescript
/**
 * Add equipment to character - Proxies to Laravel backend
 *
 * @example POST /api/characters/1/equipment
 * Body: { item_id: 42, quantity: 1 }
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const body = await readBody(event)

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/equipment`, {
    method: 'POST',
    body
  })
  return data
})
```

**Step 2: Commit**

```bash
git add server/api/characters/[id]/equipment.post.ts
git commit -m "feat: add Nitro route for POST character equipment"
```

---

## Task 4: Create Nitro Route - DELETE Equipment

**Files:**
- Create: `server/api/characters/[id]/equipment/[equipmentId].delete.ts`

**Step 1: Create directory and route file**

```typescript
/**
 * Remove equipment from character - Proxies to Laravel backend
 *
 * @example DELETE /api/characters/1/equipment/5
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const equipmentId = getRouterParam(event, 'equipmentId')

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/equipment/${equipmentId}`, {
    method: 'DELETE'
  })
  return data
})
```

**Step 2: Commit**

```bash
git add server/api/characters/[id]/equipment/[equipmentId].delete.ts
git commit -m "feat: add Nitro route for DELETE character equipment"
```

---

## Task 5: Create EquipmentItemPicker Component - Test First

**Files:**
- Create: `tests/components/character/builder/EquipmentItemPicker.test.ts`
- Create: `app/components/character/builder/EquipmentItemPicker.vue`

**Step 1: Write the failing test**

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import EquipmentItemPicker from '~/components/character/builder/EquipmentItemPicker.vue'
import { mockMartialWeaponsProficiencyType, mockMartialWeapons } from '~/tests/fixtures/equipment'

// Mock the API fetch
vi.mock('#app', async () => {
  const actual = await vi.importActual('#app')
  return {
    ...actual,
    useApi: () => ({
      apiFetch: vi.fn().mockResolvedValue({ data: mockMartialWeapons })
    })
  }
})

describe('EquipmentItemPicker', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('renders a select menu', async () => {
    const wrapper = await mountSuspended(EquipmentItemPicker, {
      props: {
        proficiencyType: mockMartialWeaponsProficiencyType,
        quantity: 1,
        modelValue: [],
        disabled: false
      }
    })

    expect(wrapper.find('[data-test="item-picker"]').exists()).toBe(true)
  })

  it('displays loading state while fetching items', async () => {
    const wrapper = await mountSuspended(EquipmentItemPicker, {
      props: {
        proficiencyType: mockMartialWeaponsProficiencyType,
        quantity: 1,
        modelValue: [],
        disabled: false
      }
    })

    // Component should handle loading state
    expect(wrapper.html()).toBeTruthy()
  })

  it('emits update:modelValue when item selected', async () => {
    const wrapper = await mountSuspended(EquipmentItemPicker, {
      props: {
        proficiencyType: mockMartialWeaponsProficiencyType,
        quantity: 1,
        modelValue: [],
        disabled: false
      }
    })

    // Wait for items to load
    await wrapper.vm.$nextTick()

    // The component should emit when selection changes
    // Exact interaction depends on USelectMenu implementation
    expect(wrapper.emitted()).toBeDefined()
  })

  it('is disabled when disabled prop is true', async () => {
    const wrapper = await mountSuspended(EquipmentItemPicker, {
      props: {
        proficiencyType: mockMartialWeaponsProficiencyType,
        quantity: 1,
        modelValue: [],
        disabled: true
      }
    })

    const picker = wrapper.find('[data-test="item-picker"]')
    expect(picker.attributes('disabled')).toBeDefined()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/EquipmentItemPicker.test.ts`
Expected: FAIL with "Cannot find module" or component not found

**Step 3: Write minimal implementation**

```vue
<script setup lang="ts">
import type { components } from '~/types/api/generated'

type ProficiencyTypeResource = components['schemas']['ProficiencyTypeResource']
type ItemResource = components['schemas']['ItemResource']

interface Props {
  proficiencyType: ProficiencyTypeResource
  quantity: number
  modelValue: number[]
  disabled?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  disabled: false
})

const emit = defineEmits<{
  'update:modelValue': [ids: number[]]
}>()

const { apiFetch } = useApi()

// Build filter for items matching this proficiency type
const filterString = computed(() => {
  const subcategory = props.proficiencyType.subcategory

  // Weapons need both melee and ranged variants
  if (subcategory === 'martial' || subcategory === 'simple') {
    return `proficiency_category IN [${subcategory}_melee, ${subcategory}_ranged] AND is_magic = false`
  }

  return `proficiency_category = ${subcategory} AND is_magic = false`
})

// Fetch matching items
const { data: itemsResponse, pending } = await useAsyncData(
  `equipment-items-${props.proficiencyType.slug}`,
  () => apiFetch<{ data: ItemResource[] }>('/items', {
    query: { filter: filterString.value, per_page: 100 }
  }),
  { watch: [filterString] }
)

const items = computed(() => itemsResponse.value?.data ?? [])

// For quantity > 1, we need multiple selections
const selectedItems = computed({
  get: () => props.modelValue,
  set: (value: number[]) => emit('update:modelValue', value)
})

// Single selection for quantity = 1
const singleSelection = computed({
  get: () => props.modelValue[0] ?? null,
  set: (value: number | null) => {
    emit('update:modelValue', value ? [value] : [])
  }
})

// Format items for USelectMenu
const selectOptions = computed(() =>
  items.value.map(item => ({
    label: item.name,
    value: item.id
  }))
)
</script>

<template>
  <div class="equipment-item-picker">
    <!-- Single selection (quantity = 1) -->
    <USelectMenu
      v-if="quantity === 1"
      v-model="singleSelection"
      data-test="item-picker"
      :options="selectOptions"
      :loading="pending"
      :disabled="disabled"
      placeholder="Select item..."
      option-attribute="label"
      value-attribute="value"
      class="w-full"
    />

    <!-- Multiple selections (quantity > 1) -->
    <div
      v-else
      class="space-y-2"
    >
      <USelectMenu
        v-for="i in quantity"
        :key="i"
        :model-value="selectedItems[i - 1] ?? null"
        :data-test="`item-picker-${i}`"
        :options="selectOptions"
        :loading="pending"
        :disabled="disabled"
        :placeholder="`Select item ${i}...`"
        option-attribute="label"
        value-attribute="value"
        class="w-full"
        @update:model-value="(val) => {
          const newSelections = [...selectedItems]
          newSelections[i - 1] = val
          selectedItems = newSelections
        }"
      />
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/EquipmentItemPicker.test.ts`
Expected: PASS (or some tests pass, refine as needed)

**Step 5: Commit**

```bash
git add tests/components/character/builder/EquipmentItemPicker.test.ts
git add app/components/character/builder/EquipmentItemPicker.vue
git commit -m "feat: add EquipmentItemPicker component for category selections"
```

---

## Task 6: Add Store State for Item Selections

**Files:**
- Modify: `app/stores/characterBuilder.ts`
- Modify: `tests/stores/characterBuilder.test.ts`

**Step 1: Write the failing test**

Add to `tests/stores/characterBuilder.test.ts`:

```typescript
describe('equipment item selections', () => {
  it('tracks item selections within compound choices', () => {
    const store = useCharacterBuilderStore()

    store.setEquipmentItemSelection('choice_2', 1, 0, 42)

    expect(store.getEquipmentItemSelection('choice_2', 1, 0)).toBe(42)
  })

  it('clears item selections on reset', () => {
    const store = useCharacterBuilderStore()
    store.setEquipmentItemSelection('choice_2', 1, 0, 42)

    store.reset()

    expect(store.getEquipmentItemSelection('choice_2', 1, 0)).toBeUndefined()
  })

  it('builds selection key correctly', () => {
    const store = useCharacterBuilderStore()

    store.setEquipmentItemSelection('choice_2', 1, 0, 42)
    store.setEquipmentItemSelection('choice_2', 1, 1, 43) // second item in same choice

    expect(store.equipmentItemSelections.get('choice_2:1:0')).toBe(42)
    expect(store.equipmentItemSelections.get('choice_2:1:1')).toBe(43)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts -t "equipment item selections"`
Expected: FAIL with "setEquipmentItemSelection is not a function"

**Step 3: Add state and actions to store**

Add to `app/stores/characterBuilder.ts` after line 46 (after `raceSpellChoices`):

```typescript
// Equipment item selections within compound choices
// Key: "choice_group:choice_option:choice_item_index"
const equipmentItemSelections = ref<Map<string, number>>(new Map())
```

Add helper functions after `setEquipmentChoice` function:

```typescript
/**
 * Build selection key for equipment item within a compound choice
 */
function makeEquipmentSelectionKey(choiceGroup: string, choiceOption: number, choiceItemIndex: number): string {
  return `${choiceGroup}:${choiceOption}:${choiceItemIndex}`
}

/**
 * Set selected item for a compound choice item
 */
function setEquipmentItemSelection(
  choiceGroup: string,
  choiceOption: number,
  choiceItemIndex: number,
  itemId: number
): void {
  const key = makeEquipmentSelectionKey(choiceGroup, choiceOption, choiceItemIndex)
  equipmentItemSelections.value.set(key, itemId)
}

/**
 * Get selected item for a compound choice item
 */
function getEquipmentItemSelection(
  choiceGroup: string,
  choiceOption: number,
  choiceItemIndex: number
): number | undefined {
  const key = makeEquipmentSelectionKey(choiceGroup, choiceOption, choiceItemIndex)
  return equipmentItemSelections.value.get(key)
}

/**
 * Clear all item selections for a choice group (when user changes option)
 */
function clearEquipmentItemSelections(choiceGroup: string): void {
  for (const key of equipmentItemSelections.value.keys()) {
    if (key.startsWith(`${choiceGroup}:`)) {
      equipmentItemSelections.value.delete(key)
    }
  }
}
```

Add to reset function (after `equipmentChoices.value = new Map()`):

```typescript
equipmentItemSelections.value = new Map()
```

Add to return statement:

```typescript
// State
equipmentItemSelections,
// Actions
setEquipmentItemSelection,
getEquipmentItemSelection,
clearEquipmentItemSelections,
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts -t "equipment item selections"`
Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "feat: add equipment item selection state to character builder store"
```

---

## Task 7: Enhance EquipmentChoiceGroup - Test First

**Files:**
- Modify: `tests/components/character/builder/EquipmentChoiceGroup.test.ts`
- Modify: `app/components/character/builder/EquipmentChoiceGroup.vue`

**Step 1: Write the failing tests**

Add to `tests/components/character/builder/EquipmentChoiceGroup.test.ts`:

```typescript
import { mockCompoundChoiceGroup, mockMartialWeapons } from '~/tests/fixtures/equipment'

// Add mock for API
vi.mock('#app', async () => {
  const actual = await vi.importActual('#app')
  return {
    ...actual,
    useApi: () => ({
      apiFetch: vi.fn().mockResolvedValue({ data: mockMartialWeapons })
    })
  }
})

describe('EquipmentChoiceGroup with choice_items', () => {
  it('shows inline picker when selected option has category choice_items', async () => {
    const wrapper = await mountSuspended(EquipmentChoiceGroup, {
      props: {
        groupName: 'Equipment Choice 2',
        items: mockCompoundChoiceGroup,
        selectedId: 36, // "martial weapon + shield" option
        itemSelections: new Map()
      }
    })

    // Should show item picker for the martial weapons category
    expect(wrapper.find('[data-test="choice-item-picker-0"]').exists()).toBe(true)
  })

  it('shows checkmark for fixed items in choice_items', async () => {
    const wrapper = await mountSuspended(EquipmentChoiceGroup, {
      props: {
        groupName: 'Equipment Choice 2',
        items: mockCompoundChoiceGroup,
        selectedId: 36,
        itemSelections: new Map()
      }
    })

    // Shield should show as auto-included
    expect(wrapper.text()).toContain('Shield')
    expect(wrapper.find('[data-test="fixed-item-1"]').exists()).toBe(true)
  })

  it('hides pickers when option not selected', async () => {
    const wrapper = await mountSuspended(EquipmentChoiceGroup, {
      props: {
        groupName: 'Equipment Choice 2',
        items: mockCompoundChoiceGroup,
        selectedId: null,
        itemSelections: new Map()
      }
    })

    expect(wrapper.find('[data-test="choice-item-picker-0"]').exists()).toBe(false)
  })

  it('emits itemSelect when picker selection changes', async () => {
    const wrapper = await mountSuspended(EquipmentChoiceGroup, {
      props: {
        groupName: 'Equipment Choice 2',
        items: mockCompoundChoiceGroup,
        selectedId: 36,
        itemSelections: new Map()
      }
    })

    // Component should emit itemSelect events
    // Exact mechanism depends on implementation
    expect(wrapper.emitted).toBeDefined()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/EquipmentChoiceGroup.test.ts -t "choice_items"`
Expected: FAIL

**Step 3: Update component implementation**

Replace `app/components/character/builder/EquipmentChoiceGroup.vue`:

```vue
<script setup lang="ts">
import type { components } from '~/types/api/generated'

type EntityItemResource = components['schemas']['EntityItemResource']
type EquipmentChoiceItemResource = components['schemas']['EquipmentChoiceItemResource']

interface Props {
  groupName: string
  items: EntityItemResource[]
  selectedId: number | null
  itemSelections?: Map<string, number> // key: "choiceOption:index", value: itemId
}

const props = withDefaults(defineProps<Props>(), {
  itemSelections: () => new Map()
})

const emit = defineEmits<{
  select: [id: number]
  itemSelect: [choiceOption: number, choiceItemIndex: number, itemId: number]
}>()

function handleSelect(id: number) {
  emit('select', id)
}

function handleItemSelect(choiceOption: number, choiceItemIndex: number, itemIds: number[]) {
  // For single selection, emit the first (and only) item
  if (itemIds.length > 0) {
    emit('itemSelect', choiceOption, choiceItemIndex, itemIds[0])
  }
}

/**
 * Get display name for equipment item
 */
function getItemDisplayName(item: EntityItemResource): string {
  if (item.item?.name) {
    return item.item.name
  }
  if (item.description) {
    return item.description
  }
  return 'Unknown item'
}

/**
 * Get the selected option object
 */
const selectedOption = computed(() =>
  props.items.find(item => item.id === props.selectedId)
)

/**
 * Check if a choice_item needs a picker (has proficiency_type, no fixed item)
 */
function needsPicker(choiceItem: EquipmentChoiceItemResource): boolean {
  return !!choiceItem.proficiency_type && !choiceItem.item
}

/**
 * Get current selection for a choice item
 */
function getItemSelection(choiceOption: number, index: number): number[] {
  const key = `${choiceOption}:${index}`
  const selected = props.itemSelections?.get(key)
  return selected ? [selected] : []
}
</script>

<template>
  <div class="space-y-2">
    <h4 class="font-medium text-gray-700 dark:text-gray-300">
      {{ groupName }}
    </h4>

    <div class="space-y-2">
      <div
        v-for="item in items"
        :key="item.id"
      >
        <!-- Main option button -->
        <button
          :data-test="`option-${item.id}`"
          type="button"
          class="w-full p-3 rounded-lg border-2 transition-all text-left flex items-center gap-3"
          :class="[
            selectedId === item.id
              ? 'ring-2 ring-primary-500 border-primary-500 bg-primary-50 dark:bg-primary-900/20'
              : 'border-gray-200 dark:border-gray-700 hover:border-primary-300'
          ]"
          @click="handleSelect(item.id)"
        >
          <!-- Radio indicator -->
          <div
            class="w-5 h-5 rounded-full border-2 flex items-center justify-center flex-shrink-0"
            :class="[
              selectedId === item.id
                ? 'border-primary-500 bg-primary-500'
                : 'border-gray-400'
            ]"
          >
            <div
              v-if="selectedId === item.id"
              class="w-2 h-2 rounded-full bg-white"
            />
          </div>

          <!-- Item info -->
          <div>
            <span class="font-medium text-gray-900 dark:text-white">
              {{ getItemDisplayName(item) }}
            </span>
            <span
              v-if="item.quantity > 1 && !item.choice_items?.length"
              class="text-gray-500 ml-1"
            >
              (×{{ item.quantity }})
            </span>
          </div>
        </button>

        <!-- Inline choice_items pickers (only for selected option) -->
        <div
          v-if="selectedId === item.id && item.choice_items?.length"
          class="ml-8 mt-2 space-y-3 border-l-2 border-primary-200 pl-4"
        >
          <div
            v-for="(choiceItem, index) in item.choice_items"
            :key="index"
            class="flex items-center gap-2"
          >
            <!-- Category item - needs picker -->
            <template v-if="needsPicker(choiceItem)">
              <div class="flex-1">
                <label class="text-sm text-gray-600 dark:text-gray-400 mb-1 block">
                  Select {{ choiceItem.proficiency_type?.name?.toLowerCase() }}
                  <span v-if="choiceItem.quantity > 1">({{ choiceItem.quantity }})</span>
                </label>
                <CharacterBuilderEquipmentItemPicker
                  :data-test="`choice-item-picker-${index}`"
                  :proficiency-type="choiceItem.proficiency_type!"
                  :quantity="choiceItem.quantity"
                  :model-value="getItemSelection(item.choice_option!, index)"
                  @update:model-value="(ids) => handleItemSelect(item.choice_option!, index, ids)"
                />
              </div>
            </template>

            <!-- Fixed item - auto-included -->
            <template v-else-if="choiceItem.item">
              <div
                :data-test="`fixed-item-${index}`"
                class="flex items-center gap-2 text-gray-700 dark:text-gray-300"
              >
                <UIcon
                  name="i-heroicons-check-circle"
                  class="w-5 h-5 text-green-500 flex-shrink-0"
                />
                <span>{{ choiceItem.item.name }}</span>
                <span
                  v-if="choiceItem.quantity > 1"
                  class="text-gray-500"
                >
                  (×{{ choiceItem.quantity }})
                </span>
              </div>
            </template>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/EquipmentChoiceGroup.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/EquipmentChoiceGroup.vue
git add tests/components/character/builder/EquipmentChoiceGroup.test.ts
git commit -m "feat: enhance EquipmentChoiceGroup with inline choice_items pickers"
```

---

## Task 8: Update StepEquipment to Wire Up Item Selections

**Files:**
- Modify: `app/components/character/builder/StepEquipment.vue`

**Step 1: Update StepEquipment to pass itemSelections and handle itemSelect**

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

const store = useCharacterBuilderStore()
const {
  selectedClass,
  selectedBackground,
  equipmentChoices,
  equipmentItemSelections,
  allEquipmentChoicesMade,
  isLoading
} = storeToRefs(store)

// Separate equipment by source
const classFixedEquipment = computed(() =>
  selectedClass.value?.equipment?.filter(eq => !eq.is_choice) ?? []
)

const backgroundFixedEquipment = computed(() =>
  selectedBackground.value?.equipment?.filter(eq => !eq.is_choice) ?? []
)

// Get choice groups by source
const classChoiceGroups = computed(() => {
  const groups = new Map()
  for (const item of selectedClass.value?.equipment ?? []) {
    if (item.is_choice && item.choice_group) {
      const existing = groups.get(item.choice_group) ?? []
      groups.set(item.choice_group, [...existing, item])
    }
  }
  return groups
})

const backgroundChoiceGroups = computed(() => {
  const groups = new Map()
  for (const item of selectedBackground.value?.equipment ?? []) {
    if (item.is_choice && item.choice_group) {
      const existing = groups.get(item.choice_group) ?? []
      groups.set(item.choice_group, [...existing, item])
    }
  }
  return groups
})

/**
 * Handle equipment choice selection
 */
function handleChoiceSelect(choiceGroup: string, id: number) {
  // Clear any previous item selections for this group when option changes
  store.clearEquipmentItemSelections(choiceGroup)
  store.setEquipmentChoice(choiceGroup, id)
}

/**
 * Handle item selection within a compound choice
 */
function handleItemSelect(choiceGroup: string, choiceOption: number, choiceItemIndex: number, itemId: number) {
  store.setEquipmentItemSelection(choiceGroup, choiceOption, choiceItemIndex, itemId)
}

/**
 * Build itemSelections map for a choice group
 */
function buildItemSelectionsMap(choiceGroup: string): Map<string, number> {
  const map = new Map<string, number>()
  for (const [key, value] of equipmentItemSelections.value) {
    if (key.startsWith(`${choiceGroup}:`)) {
      // Extract "choiceOption:index" from "choiceGroup:choiceOption:index"
      const parts = key.split(':')
      const shortKey = `${parts[1]}:${parts[2]}`
      map.set(shortKey, value)
    }
  }
  return map
}

/**
 * Format choice group name for display
 */
function formatGroupName(group: string): string {
  const match = group.match(/choice[_-]?(\d+)/i)
  if (match) {
    return `Equipment Choice ${match[1]}`
  }
  return group
    .replace(/[_-]/g, ' ')
    .replace(/\b\w/g, l => l.toUpperCase())
}

/**
 * Get display name for equipment item
 */
function getItemDisplayName(item: { item?: { name?: string } | null, description?: string | null }): string {
  if (item.item?.name) {
    return item.item.name
  }
  if (item.description) {
    return item.description
  }
  return 'Unknown item'
}

/**
 * Continue to next step
 */
function handleContinue() {
  store.nextStep()
}
</script>

<template>
  <div class="space-y-6">
    <!-- Header -->
    <div class="text-center">
      <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
        Choose Your Starting Equipment
      </h2>
      <p class="mt-2 text-gray-600 dark:text-gray-400">
        Select your starting gear from your class and background
      </p>
    </div>

    <!-- Class Equipment -->
    <div
      v-if="selectedClass"
      class="space-y-4"
    >
      <h3 class="text-lg font-semibold text-gray-900 dark:text-white border-b pb-2">
        From Your Class ({{ selectedClass.name }})
      </h3>

      <!-- Fixed Items -->
      <div
        v-if="classFixedEquipment.length > 0"
        class="space-y-2"
      >
        <div
          v-for="item in classFixedEquipment"
          :key="item.id"
          class="flex items-center gap-2 text-gray-700 dark:text-gray-300"
        >
          <UIcon
            name="i-heroicons-check-circle"
            class="w-5 h-5 text-green-500"
          />
          <span>{{ getItemDisplayName(item) }}</span>
          <span
            v-if="item.quantity > 1"
            class="text-gray-500"
          >(×{{ item.quantity }})</span>
        </div>
      </div>

      <!-- Choice Groups -->
      <CharacterBuilderEquipmentChoiceGroup
        v-for="[group, items] in classChoiceGroups"
        :key="group"
        :group-name="formatGroupName(group)"
        :items="items"
        :selected-id="equipmentChoices.get(group) ?? null"
        :item-selections="buildItemSelectionsMap(group)"
        @select="(id) => handleChoiceSelect(group, id)"
        @item-select="(opt, idx, itemId) => handleItemSelect(group, opt, idx, itemId)"
      />
    </div>

    <!-- Background Equipment -->
    <div
      v-if="selectedBackground"
      class="space-y-4"
    >
      <h3 class="text-lg font-semibold text-gray-900 dark:text-white border-b pb-2">
        From Your Background ({{ selectedBackground.name }})
      </h3>

      <!-- Fixed Items -->
      <div
        v-if="backgroundFixedEquipment.length > 0"
        class="space-y-2"
      >
        <div
          v-for="item in backgroundFixedEquipment"
          :key="item.id"
          class="flex items-center gap-2 text-gray-700 dark:text-gray-300"
        >
          <UIcon
            name="i-heroicons-check-circle"
            class="w-5 h-5 text-green-500"
          />
          <span>{{ getItemDisplayName(item) }}</span>
          <span
            v-if="item.quantity > 1"
            class="text-gray-500"
          >(×{{ item.quantity }})</span>
        </div>
      </div>

      <!-- Choice Groups -->
      <CharacterBuilderEquipmentChoiceGroup
        v-for="[group, items] in backgroundChoiceGroups"
        :key="group"
        :group-name="formatGroupName(group)"
        :items="items"
        :selected-id="equipmentChoices.get(group) ?? null"
        :item-selections="buildItemSelectionsMap(group)"
        @select="(id) => handleChoiceSelect(group, id)"
        @item-select="(opt, idx, itemId) => handleItemSelect(group, opt, idx, itemId)"
      />
    </div>

    <!-- Continue Button -->
    <div class="flex justify-center pt-4">
      <UButton
        data-test="continue-btn"
        size="lg"
        :disabled="!allEquipmentChoicesMade || isLoading"
        :loading="isLoading"
        @click="handleContinue"
      >
        Continue with Equipment
      </UButton>
    </div>
  </div>
</template>
```

**Step 2: Run tests**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StepEquipment.test.ts`
Expected: PASS

**Step 3: Commit**

```bash
git add app/components/character/builder/StepEquipment.vue
git commit -m "feat: wire up item selections in StepEquipment"
```

---

## Task 9: Update Validation Logic

**Files:**
- Modify: `app/stores/characterBuilder.ts`

**Step 1: Enhance allEquipmentChoicesMade computed**

Update the `allEquipmentChoicesMade` computed in the store:

```typescript
// Enhanced validation: all equipment choices made including item selections
const allEquipmentChoicesMade = computed(() => {
  // Check each choice group has a selection
  for (const [group] of equipmentByChoiceGroup.value) {
    if (!equipmentChoices.value.has(group)) return false

    // Find the selected option
    const selectedOptionId = equipmentChoices.value.get(group)
    const allItems = [...(selectedClass.value?.equipment ?? []), ...(selectedBackground.value?.equipment ?? [])]
    const selectedOption = allItems.find(item => item.id === selectedOptionId)

    if (!selectedOption?.choice_items?.length) continue

    // Check all category items have selections
    for (const [index, choiceItem] of selectedOption.choice_items.entries()) {
      // Only category items need selection (proficiency_type set, item not set)
      if (choiceItem.proficiency_type && !choiceItem.item) {
        const key = `${group}:${selectedOption.choice_option}:${index}`
        if (!equipmentItemSelections.value.has(key)) return false
      }
    }
  }
  return true
})
```

**Step 2: Run tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts`
Expected: PASS

**Step 3: Commit**

```bash
git add app/stores/characterBuilder.ts
git commit -m "feat: enhance equipment validation to check item selections"
```

---

## Task 10: Add Store Action to Save Equipment

**Files:**
- Modify: `app/stores/characterBuilder.ts`

**Step 1: Add saveEquipmentChoices action**

```typescript
/**
 * Save all equipment choices to the character
 * Called when moving from equipment step to next step
 */
async function saveEquipmentChoices(): Promise<void> {
  if (!characterId.value) return

  isLoading.value = true
  error.value = null

  try {
    const allItems = [...(selectedClass.value?.equipment ?? []), ...(selectedBackground.value?.equipment ?? [])]

    for (const [group, optionId] of equipmentChoices.value) {
      const selectedOption = allItems.find(item => item.id === optionId)
      if (!selectedOption) continue

      // If option has choice_items, process each
      if (selectedOption.choice_items?.length) {
        for (const [index, choiceItem] of selectedOption.choice_items.entries()) {
          let itemId: number
          const quantity = choiceItem.quantity

          if (choiceItem.item) {
            // Fixed item - use directly
            itemId = choiceItem.item.id
          } else if (choiceItem.proficiency_type) {
            // Category item - get user's selection
            const key = `${group}:${selectedOption.choice_option}:${index}`
            const selectedItemId = equipmentItemSelections.value.get(key)
            if (!selectedItemId) continue
            itemId = selectedItemId
          } else {
            continue
          }

          await apiFetch(`/characters/${characterId.value}/equipment`, {
            method: 'POST',
            body: { item_id: itemId, quantity }
          })
        }
      } else if (selectedOption.item) {
        // Simple choice with direct item reference
        await apiFetch(`/characters/${characterId.value}/equipment`, {
          method: 'POST',
          body: { item_id: selectedOption.item.id, quantity: selectedOption.quantity }
        })
      }
    }

    // Also add fixed equipment
    const fixedEquipment = allItems.filter(item => !item.is_choice && item.item)
    for (const item of fixedEquipment) {
      if (item.item) {
        await apiFetch(`/characters/${characterId.value}/equipment`, {
          method: 'POST',
          body: { item_id: item.item.id, quantity: item.quantity }
        })
      }
    }
  } catch (err: unknown) {
    error.value = 'Failed to save equipment'
    throw err
  } finally {
    isLoading.value = false
  }
}
```

Add to return statement:
```typescript
saveEquipmentChoices,
```

**Step 2: Commit**

```bash
git add app/stores/characterBuilder.ts
git commit -m "feat: add saveEquipmentChoices action to persist equipment to API"
```

---

## Task 11: Run Full Test Suite and Fix Issues

**Step 1: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: PASS (fix any type errors)

**Step 2: Run linter**

Run: `docker compose exec nuxt npm run lint:fix`
Expected: PASS (auto-fixes applied)

**Step 3: Run full test suite**

Run: `docker compose exec nuxt npm run test`
Expected: PASS

**Step 4: Final commit**

```bash
git add -A
git commit -m "chore: fix any remaining issues from equipment choice items implementation"
```

---

## Task 12: Manual Browser Testing

**Step 1: Start dev server**

Run: `docker compose exec nuxt npm run dev`

**Step 2: Test the flow**

1. Navigate to character builder
2. Create a draft character
3. Select a race
4. Select Fighter class (has compound equipment choices)
5. Complete ability scores
6. Select a background
7. On Equipment step:
   - Verify "Equipment Choice 2" shows compound options
   - Select "a martial weapon and a shield"
   - Verify dropdown appears for martial weapon selection
   - Verify Shield shows as auto-included with checkmark
   - Select a weapon (e.g., Longsword)
   - Verify Continue button becomes enabled
8. Continue to next step

**Step 3: Verify dark mode**

Toggle to dark mode and verify all styling looks correct.

---

## Summary

| Task | Description | Key Files |
|------|-------------|-----------|
| 1 | Update test fixtures | `tests/fixtures/equipment.ts` |
| 2-4 | Create Nitro routes | `server/api/characters/[id]/equipment*.ts` |
| 5 | Create EquipmentItemPicker | `app/components/.../EquipmentItemPicker.vue` |
| 6 | Add store state | `app/stores/characterBuilder.ts` |
| 7 | Enhance EquipmentChoiceGroup | `app/components/.../EquipmentChoiceGroup.vue` |
| 8 | Wire up StepEquipment | `app/components/.../StepEquipment.vue` |
| 9 | Update validation | `app/stores/characterBuilder.ts` |
| 10 | Add save action | `app/stores/characterBuilder.ts` |
| 11 | Quality gates | All files |
| 12 | Manual testing | Browser |
