# Equipment Paperdoll Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add visual paperdoll equipment display with 11 slots and smart equip flow with slot picker modal.

**Architecture:** Create slot mapping utility, paperdoll display component, and slot picker modal. Modify inventory page layout and equip flow to use new components.

**Tech Stack:** Vue 3, NuxtUI 4, TypeScript, Vitest, Pinia

**Design Doc:** `wrapper/docs/frontend/plans/2025-12-13-equipment-paperdoll-design.md`

---

## Task 1: Equipment Slots Utility

**Files:**
- Create: `app/utils/equipmentSlots.ts`
- Create: `tests/utils/equipmentSlots.test.ts`

### Step 1: Write failing tests for slot mapping

```typescript
// tests/utils/equipmentSlots.test.ts
import { describe, it, expect } from 'vitest'
import {
  getValidSlots,
  getDefaultSlot,
  needsSlotPicker,
  guessSlotFromName,
  BODY_SLOTS
} from '~/utils/equipmentSlots'

describe('equipmentSlots', () => {
  describe('getValidSlots', () => {
    it('returns armor slot for armor types', () => {
      expect(getValidSlots('Light Armor')).toEqual(['armor'])
      expect(getValidSlots('Medium Armor')).toEqual(['armor'])
      expect(getValidSlots('Heavy Armor')).toEqual(['armor'])
    })

    it('returns off_hand for shield', () => {
      expect(getValidSlots('Shield')).toEqual(['off_hand'])
    })

    it('returns main_hand and off_hand for melee weapon', () => {
      expect(getValidSlots('Melee Weapon')).toEqual(['main_hand', 'off_hand'])
    })

    it('returns main_hand only for ranged weapon', () => {
      expect(getValidSlots('Ranged Weapon')).toEqual(['main_hand'])
    })

    it('returns ring slots for ring', () => {
      expect(getValidSlots('Ring')).toEqual(['ring_1', 'ring_2'])
    })

    it('returns body slots for wondrous item', () => {
      expect(getValidSlots('Wondrous Item')).toEqual(BODY_SLOTS)
    })

    it('returns empty array for non-equippable items', () => {
      expect(getValidSlots('Potion')).toEqual([])
      expect(getValidSlots('Scroll')).toEqual([])
      expect(getValidSlots('Adventuring Gear')).toEqual([])
    })
  })

  describe('getDefaultSlot', () => {
    it('returns armor for armor types', () => {
      expect(getDefaultSlot('Light Armor')).toBe('armor')
    })

    it('returns main_hand for weapons', () => {
      expect(getDefaultSlot('Melee Weapon')).toBe('main_hand')
      expect(getDefaultSlot('Ranged Weapon')).toBe('main_hand')
    })

    it('returns ring_1 for ring', () => {
      expect(getDefaultSlot('Ring')).toBe('ring_1')
    })

    it('returns null for wondrous item', () => {
      expect(getDefaultSlot('Wondrous Item')).toBeNull()
    })
  })

  describe('needsSlotPicker', () => {
    it('returns false for single-slot items', () => {
      expect(needsSlotPicker('Light Armor')).toBe(false)
      expect(needsSlotPicker('Shield')).toBe(false)
    })

    it('returns true for wondrous item', () => {
      expect(needsSlotPicker('Wondrous Item')).toBe(true)
    })

    it('returns true for ring (user chooses slot)', () => {
      expect(needsSlotPicker('Ring')).toBe(true)
    })
  })

  describe('guessSlotFromName', () => {
    it('guesses feet for boot items', () => {
      expect(guessSlotFromName('Boots of Speed')).toBe('feet')
      expect(guessSlotFromName('Winged Boots')).toBe('feet')
    })

    it('guesses cloak for cloak items', () => {
      expect(guessSlotFromName('Cloak of Elvenkind')).toBe('cloak')
      expect(guessSlotFromName('Cape of the Mountebank')).toBe('cloak')
    })

    it('guesses belt for belt items', () => {
      expect(guessSlotFromName('Belt of Giant Strength')).toBe('belt')
      expect(guessSlotFromName('Girdle of Femininity')).toBe('belt')
    })

    it('guesses head for helm items', () => {
      expect(guessSlotFromName('Helm of Brilliance')).toBe('head')
      expect(guessSlotFromName('Circlet of Blasting')).toBe('head')
      expect(guessSlotFromName('Hat of Disguise')).toBe('head')
    })

    it('guesses neck for amulet items', () => {
      expect(guessSlotFromName('Amulet of Health')).toBe('neck')
      expect(guessSlotFromName('Necklace of Fireballs')).toBe('neck')
      expect(guessSlotFromName('Periapt of Proof against Poison')).toBe('neck')
    })

    it('guesses hands for glove items', () => {
      expect(guessSlotFromName('Gloves of Missile Snaring')).toBe('hands')
      expect(guessSlotFromName('Gauntlets of Ogre Power')).toBe('hands')
      expect(guessSlotFromName('Bracers of Defense')).toBe('hands')
    })

    it('returns null for unknown items', () => {
      expect(guessSlotFromName('Bag of Holding')).toBeNull()
      expect(guessSlotFromName('Wand of Magic Missiles')).toBeNull()
    })
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/utils/equipmentSlots.test.ts
```

Expected: FAIL - module not found

### Step 3: Implement the utility

```typescript
// app/utils/equipmentSlots.ts

export type EquipmentSlot =
  | 'head'
  | 'neck'
  | 'cloak'
  | 'armor'
  | 'belt'
  | 'hands'
  | 'ring_1'
  | 'ring_2'
  | 'feet'
  | 'main_hand'
  | 'off_hand'

export const BODY_SLOTS: EquipmentSlot[] = ['head', 'neck', 'cloak', 'belt', 'hands', 'feet']

export const ALL_SLOTS: EquipmentSlot[] = [
  'head', 'neck', 'cloak', 'armor', 'belt', 'hands',
  'ring_1', 'ring_2', 'feet', 'main_hand', 'off_hand'
]

export const SLOT_LABELS: Record<EquipmentSlot, string> = {
  head: 'Head',
  neck: 'Neck',
  cloak: 'Cloak',
  armor: 'Armor',
  belt: 'Belt',
  hands: 'Hands',
  ring_1: 'Ring',
  ring_2: 'Ring',
  feet: 'Feet',
  main_hand: 'Main Hand',
  off_hand: 'Off Hand'
}

// Item type to valid slots mapping
const SLOT_MAPPING: Record<string, EquipmentSlot[]> = {
  'Light Armor': ['armor'],
  'Medium Armor': ['armor'],
  'Heavy Armor': ['armor'],
  'Shield': ['off_hand'],
  'Melee Weapon': ['main_hand', 'off_hand'],
  'Ranged Weapon': ['main_hand'],
  'Staff': ['main_hand'],
  'Rod': ['main_hand'],
  'Wand': ['main_hand'],
  'Ring': ['ring_1', 'ring_2'],
  'Wondrous Item': BODY_SLOTS
}

// Types that need slot picker (multiple options or no default)
const PICKER_TYPES = new Set(['Wondrous Item', 'Ring'])

// Name patterns for guessing slots
const NAME_PATTERNS: Array<{ pattern: RegExp; slot: EquipmentSlot }> = [
  { pattern: /\b(boots?|shoes?)\b/i, slot: 'feet' },
  { pattern: /\b(cloak|cape|mantle)\b/i, slot: 'cloak' },
  { pattern: /\b(belt|girdle|sash)\b/i, slot: 'belt' },
  { pattern: /\b(helm|helmet|hat|circlet|crown|headband)\b/i, slot: 'head' },
  { pattern: /\b(amulet|necklace|periapt|medallion|pendant|brooch)\b/i, slot: 'neck' },
  { pattern: /\b(gloves?|gauntlets?|bracers?)\b/i, slot: 'hands' }
]

/**
 * Get valid equipment slots for an item type
 */
export function getValidSlots(itemType: string | null): EquipmentSlot[] {
  if (!itemType) return []
  return SLOT_MAPPING[itemType] ?? []
}

/**
 * Get the default slot for auto-equipping
 * Returns null if user must choose (wondrous items)
 */
export function getDefaultSlot(itemType: string | null): EquipmentSlot | null {
  if (!itemType) return null
  const slots = SLOT_MAPPING[itemType]
  if (!slots || slots.length === 0) return null
  if (itemType === 'Wondrous Item') return null
  return slots[0]
}

/**
 * Check if item type requires slot picker modal
 */
export function needsSlotPicker(itemType: string | null): boolean {
  if (!itemType) return false
  return PICKER_TYPES.has(itemType)
}

/**
 * Guess equipment slot from item name (for pre-selection)
 */
export function guessSlotFromName(itemName: string): EquipmentSlot | null {
  for (const { pattern, slot } of NAME_PATTERNS) {
    if (pattern.test(itemName)) {
      return slot
    }
  }
  return null
}
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/utils/equipmentSlots.test.ts
```

Expected: All tests PASS

### Step 5: Commit

```bash
git add app/utils/equipmentSlots.ts tests/utils/equipmentSlots.test.ts
git commit -m "feat(inventory): add equipment slot mapping utility

- Type-to-slot mapping for all item types
- Name pattern matching for wondrous item pre-selection
- Helper functions for equip flow logic"
```

---

## Task 2: Equipment Paperdoll Component

**Files:**
- Create: `app/components/character/inventory/EquipmentPaperdoll.vue`
- Create: `tests/components/character/inventory/EquipmentPaperdoll.test.ts`

### Step 1: Write failing tests

```typescript
// tests/components/character/inventory/EquipmentPaperdoll.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import EquipmentPaperdoll from '~/components/character/inventory/EquipmentPaperdoll.vue'
import type { CharacterEquipment } from '~/types/character'

const mockEquipment: CharacterEquipment[] = [
  {
    id: 1,
    item_slug: 'phb:longsword',
    item: { name: 'Longsword', item_type: 'Melee Weapon' },
    custom_name: null,
    custom_description: null,
    quantity: 1,
    equipped: true,
    location: 'main_hand',
    is_attuned: false
  },
  {
    id: 2,
    item_slug: 'phb:chain-mail',
    item: { name: 'Chain Mail', item_type: 'Heavy Armor', armor_class: 16 },
    custom_name: null,
    custom_description: null,
    quantity: 1,
    equipped: true,
    location: 'armor',
    is_attuned: false
  },
  {
    id: 3,
    item_slug: 'dmg:cloak-of-elvenkind',
    item: { name: 'Cloak of Elvenkind', item_type: 'Wondrous Item' },
    custom_name: null,
    custom_description: null,
    quantity: 1,
    equipped: true,
    location: 'cloak',
    is_attuned: true
  }
]

describe('EquipmentPaperdoll', () => {
  it('renders all 11 equipment slots', async () => {
    const wrapper = await mountSuspended(EquipmentPaperdoll, {
      props: { equipment: [] }
    })

    expect(wrapper.find('[data-testid="slot-head"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-neck"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-cloak"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-armor"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-belt"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-hands"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-ring_1"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-ring_2"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-feet"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-main_hand"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-off_hand"]').exists()).toBe(true)
  })

  it('shows "Empty" for unoccupied slots', async () => {
    const wrapper = await mountSuspended(EquipmentPaperdoll, {
      props: { equipment: [] }
    })

    const headSlot = wrapper.find('[data-testid="slot-head"]')
    expect(headSlot.text()).toContain('Empty')
  })

  it('shows item name in occupied slot', async () => {
    const wrapper = await mountSuspended(EquipmentPaperdoll, {
      props: { equipment: mockEquipment }
    })

    const mainHandSlot = wrapper.find('[data-testid="slot-main_hand"]')
    expect(mainHandSlot.text()).toContain('Longsword')

    const armorSlot = wrapper.find('[data-testid="slot-armor"]')
    expect(armorSlot.text()).toContain('Chain Mail')
  })

  it('emits item-click when clicking equipped item', async () => {
    const wrapper = await mountSuspended(EquipmentPaperdoll, {
      props: { equipment: mockEquipment }
    })

    const mainHandItem = wrapper.find('[data-testid="slot-main_hand"] button')
    await mainHandItem.trigger('click')

    expect(wrapper.emitted('item-click')).toBeTruthy()
    expect(wrapper.emitted('item-click')![0]).toEqual([1])
  })

  it('does not emit click for empty slots', async () => {
    const wrapper = await mountSuspended(EquipmentPaperdoll, {
      props: { equipment: [] }
    })

    const headSlot = wrapper.find('[data-testid="slot-head"]')
    await headSlot.trigger('click')

    expect(wrapper.emitted('item-click')).toBeFalsy()
  })

  it('shows custom name if provided', async () => {
    const customEquipment: CharacterEquipment[] = [{
      id: 1,
      item_slug: 'phb:longsword',
      item: { name: 'Longsword' },
      custom_name: 'Dragonfang',
      custom_description: null,
      quantity: 1,
      equipped: true,
      location: 'main_hand',
      is_attuned: false
    }]

    const wrapper = await mountSuspended(EquipmentPaperdoll, {
      props: { equipment: customEquipment }
    })

    expect(wrapper.text()).toContain('Dragonfang')
    expect(wrapper.text()).not.toContain('Longsword')
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/components/character/inventory/EquipmentPaperdoll.test.ts
```

Expected: FAIL - component not found

### Step 3: Implement the component

```vue
<!-- app/components/character/inventory/EquipmentPaperdoll.vue -->
<script setup lang="ts">
/**
 * Equipment Paperdoll
 *
 * Visual display of all 11 equipment slots arranged around a character silhouette.
 * Click equipped item → scroll to it in item table.
 */

import type { CharacterEquipment } from '~/types/character'
import { ALL_SLOTS, SLOT_LABELS, type EquipmentSlot } from '~/utils/equipmentSlots'

interface Props {
  equipment: CharacterEquipment[]
}

const props = defineProps<Props>()

const emit = defineEmits<{
  'item-click': [itemId: number]
}>()

// Get equipped item for a slot
function getEquippedItem(slot: EquipmentSlot): CharacterEquipment | undefined {
  return props.equipment.find(e => e.equipped && e.location === slot)
}

// Get display name for an item
function getItemName(equipment: CharacterEquipment): string {
  if (equipment.custom_name) return equipment.custom_name
  const item = equipment.item as { name?: string } | null
  return item?.name ?? 'Unknown'
}

// Handle click on equipped item
function handleItemClick(item: CharacterEquipment | undefined) {
  if (item) {
    emit('item-click', item.id)
  }
}

// Slot positions for grid layout (row, col)
const slotPositions: Record<EquipmentSlot, { row: number; col: number }> = {
  head: { row: 1, col: 2 },
  neck: { row: 2, col: 2 },
  main_hand: { row: 3, col: 1 },
  armor: { row: 3, col: 2 },
  off_hand: { row: 3, col: 3 },
  cloak: { row: 4, col: 2 },
  hands: { row: 5, col: 1 },
  belt: { row: 5, col: 2 },
  ring_1: { row: 5, col: 3 },
  feet: { row: 6, col: 2 },
  ring_2: { row: 6, col: 3 }
}
</script>

<template>
  <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4">
    <h3 class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-4">
      Equipment
    </h3>

    <!-- Paperdoll Grid -->
    <div class="grid grid-cols-3 gap-2">
      <template v-for="slot in ALL_SLOTS" :key="slot">
        <div
          :data-testid="`slot-${slot}`"
          :style="{
            gridRow: slotPositions[slot].row,
            gridColumn: slotPositions[slot].col
          }"
          class="bg-gray-100 dark:bg-gray-700 rounded p-2 min-h-[60px] flex flex-col justify-center"
        >
          <!-- Slot label -->
          <span class="text-[10px] font-medium text-gray-400 dark:text-gray-500 uppercase tracking-wider">
            {{ SLOT_LABELS[slot] }}
          </span>

          <!-- Equipped item or empty -->
          <button
            v-if="getEquippedItem(slot)"
            class="text-xs font-medium text-gray-900 dark:text-white hover:text-primary transition-colors text-left truncate"
            @click="handleItemClick(getEquippedItem(slot))"
          >
            {{ getItemName(getEquippedItem(slot)!) }}
          </button>
          <span
            v-else
            class="text-xs text-gray-400 dark:text-gray-500 italic"
          >
            Empty
          </span>
        </div>
      </template>
    </div>
  </div>
</template>
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/components/character/inventory/EquipmentPaperdoll.test.ts
```

Expected: All tests PASS

### Step 5: Commit

```bash
git add app/components/character/inventory/EquipmentPaperdoll.vue tests/components/character/inventory/EquipmentPaperdoll.test.ts
git commit -m "feat(inventory): add EquipmentPaperdoll component

- Grid layout showing all 11 equipment slots
- Click equipped item to scroll to table
- Shows custom name if provided"
```

---

## Task 3: Slot Picker Modal Component

**Files:**
- Create: `app/components/character/inventory/EquipSlotPickerModal.vue`
- Create: `tests/components/character/inventory/EquipSlotPickerModal.test.ts`

### Step 1: Write failing tests

```typescript
// tests/components/character/inventory/EquipSlotPickerModal.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import EquipSlotPickerModal from '~/components/character/inventory/EquipSlotPickerModal.vue'

describe('EquipSlotPickerModal', () => {
  const defaultProps = {
    open: true,
    itemName: 'Cloak of Elvenkind',
    validSlots: ['head', 'neck', 'cloak', 'belt', 'hands', 'feet'] as const,
    suggestedSlot: 'cloak' as const
  }

  it('renders modal when open', async () => {
    const wrapper = await mountSuspended(EquipSlotPickerModal, {
      props: defaultProps
    })

    expect(wrapper.text()).toContain('Equip: Cloak of Elvenkind')
    expect(wrapper.text()).toContain('Where do you want to equip this?')
  })

  it('shows all valid slot options', async () => {
    const wrapper = await mountSuspended(EquipSlotPickerModal, {
      props: defaultProps
    })

    expect(wrapper.find('[data-testid="slot-option-head"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-option-neck"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-option-cloak"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-option-belt"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-option-hands"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-option-feet"]').exists()).toBe(true)
  })

  it('pre-selects suggested slot', async () => {
    const wrapper = await mountSuspended(EquipSlotPickerModal, {
      props: defaultProps
    })

    const cloakRadio = wrapper.find('[data-testid="slot-option-cloak"] input')
    expect((cloakRadio.element as HTMLInputElement).checked).toBe(true)
  })

  it('emits select with chosen slot on confirm', async () => {
    const wrapper = await mountSuspended(EquipSlotPickerModal, {
      props: defaultProps
    })

    // Select a different slot
    const neckRadio = wrapper.find('[data-testid="slot-option-neck"] input')
    await neckRadio.setValue(true)

    // Click equip button
    const equipBtn = wrapper.find('[data-testid="equip-btn"]')
    await equipBtn.trigger('click')

    expect(wrapper.emitted('select')).toBeTruthy()
    expect(wrapper.emitted('select')![0]).toEqual(['neck'])
  })

  it('emits update:open false on cancel', async () => {
    const wrapper = await mountSuspended(EquipSlotPickerModal, {
      props: defaultProps
    })

    const cancelBtn = wrapper.find('[data-testid="cancel-btn"]')
    await cancelBtn.trigger('click')

    expect(wrapper.emitted('update:open')).toBeTruthy()
    expect(wrapper.emitted('update:open')![0]).toEqual([false])
  })

  it('shows ring slots for ring items', async () => {
    const wrapper = await mountSuspended(EquipSlotPickerModal, {
      props: {
        open: true,
        itemName: 'Ring of Protection',
        validSlots: ['ring_1', 'ring_2'] as const,
        suggestedSlot: 'ring_1' as const
      }
    })

    expect(wrapper.find('[data-testid="slot-option-ring_1"]').exists()).toBe(true)
    expect(wrapper.find('[data-testid="slot-option-ring_2"]').exists()).toBe(true)
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/components/character/inventory/EquipSlotPickerModal.test.ts
```

Expected: FAIL - component not found

### Step 3: Implement the component

```vue
<!-- app/components/character/inventory/EquipSlotPickerModal.vue -->
<script setup lang="ts">
/**
 * Equipment Slot Picker Modal
 *
 * Shown when equipping items with multiple valid slots (wondrous items, rings).
 * Pre-selects suggested slot based on item name pattern matching.
 */

import { SLOT_LABELS, type EquipmentSlot } from '~/utils/equipmentSlots'

interface Props {
  open: boolean
  itemName: string
  validSlots: EquipmentSlot[]
  suggestedSlot?: EquipmentSlot | null
  loading?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  suggestedSlot: null,
  loading: false
})

const emit = defineEmits<{
  'update:open': [value: boolean]
  'select': [slot: EquipmentSlot]
}>()

// Selected slot (defaults to suggested or first valid)
const selectedSlot = ref<EquipmentSlot>(
  props.suggestedSlot ?? props.validSlots[0]
)

// Reset selection when modal opens with new item
watch(() => props.open, (isOpen) => {
  if (isOpen) {
    selectedSlot.value = props.suggestedSlot ?? props.validSlots[0]
  }
})

function handleCancel() {
  emit('update:open', false)
}

function handleEquip() {
  emit('select', selectedSlot.value)
}
</script>

<template>
  <UModal
    :open="open"
    @update:open="emit('update:open', $event)"
  >
    <template #header>
      <div class="flex items-center gap-2">
        <UIcon name="i-heroicons-hand-raised" class="w-5 h-5" />
        <span>Equip: {{ itemName }}</span>
      </div>
    </template>

    <template #body>
      <p class="text-sm text-gray-600 dark:text-gray-400 mb-4">
        Where do you want to equip this?
      </p>

      <div class="space-y-2">
        <label
          v-for="slot in validSlots"
          :key="slot"
          :data-testid="`slot-option-${slot}`"
          class="flex items-center gap-3 p-3 rounded-lg border cursor-pointer transition-colors"
          :class="[
            selectedSlot === slot
              ? 'border-primary bg-primary/5'
              : 'border-gray-200 dark:border-gray-700 hover:border-gray-300 dark:hover:border-gray-600'
          ]"
        >
          <input
            v-model="selectedSlot"
            type="radio"
            :value="slot"
            class="text-primary focus:ring-primary"
          >
          <span class="font-medium text-gray-900 dark:text-white">
            {{ SLOT_LABELS[slot] }}
          </span>
        </label>
      </div>
    </template>

    <template #footer>
      <div class="flex justify-end gap-2">
        <UButton
          data-testid="cancel-btn"
          variant="ghost"
          :disabled="loading"
          @click="handleCancel"
        >
          Cancel
        </UButton>
        <UButton
          data-testid="equip-btn"
          color="primary"
          :loading="loading"
          @click="handleEquip"
        >
          Equip
        </UButton>
      </div>
    </template>
  </UModal>
</template>
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/components/character/inventory/EquipSlotPickerModal.test.ts
```

Expected: All tests PASS

### Step 5: Commit

```bash
git add app/components/character/inventory/EquipSlotPickerModal.vue tests/components/character/inventory/EquipSlotPickerModal.test.ts
git commit -m "feat(inventory): add EquipSlotPickerModal component

- Radio button selection for valid slots
- Pre-selects suggested slot from name matching
- Loading state for equip action"
```

---

## Task 4: Update EquipmentStatus to Use Paperdoll

**Files:**
- Modify: `app/components/character/inventory/EquipmentStatus.vue`
- Modify: `tests/components/character/inventory/EquipmentStatus.test.ts`

### Step 1: Update tests for new structure

The component will now render the paperdoll for equipment and keep the attunement section separate.

```typescript
// Update tests/components/character/inventory/EquipmentStatus.test.ts
// Add new tests for paperdoll integration

describe('EquipmentStatus with Paperdoll', () => {
  it('renders EquipmentPaperdoll component', async () => {
    const wrapper = await mountSuspended(EquipmentStatus, {
      props: { equipment: mockEquipment }
    })

    // Should have paperdoll section
    expect(wrapper.find('[data-testid="equipment-paperdoll"]').exists()).toBe(true)
  })

  it('still shows attunement section separately', async () => {
    const wrapper = await mountSuspended(EquipmentStatus, {
      props: { equipment: mockEquipment }
    })

    // Attunement section should still exist
    expect(wrapper.text()).toContain('Attuned')
    expect(wrapper.text()).toContain('0/3')
  })

  it('passes item-click events from paperdoll', async () => {
    const wrapper = await mountSuspended(EquipmentStatus, {
      props: { equipment: mockEquipment }
    })

    // Find paperdoll and trigger item click
    const paperdoll = wrapper.findComponent({ name: 'EquipmentPaperdoll' })
    paperdoll.vm.$emit('item-click', 1)

    expect(wrapper.emitted('item-click')).toBeTruthy()
    expect(wrapper.emitted('item-click')![0]).toEqual([1])
  })
})
```

### Step 2: Run tests to see what needs updating

```bash
docker compose exec nuxt npm run test -- tests/components/character/inventory/EquipmentStatus.test.ts
```

### Step 3: Update the component

```vue
<!-- app/components/character/inventory/EquipmentStatus.vue -->
<script setup lang="ts">
/**
 * Equipment Status Sidebar
 *
 * Contains:
 * - Equipment Paperdoll (all 11 slots)
 * - Attunement section (up to 3 items)
 *
 * Clicking an item emits 'item-click' to scroll to it in the item list.
 */

import type { CharacterEquipment } from '~/types/character'

interface Props {
  equipment: CharacterEquipment[]
}

const props = defineProps<Props>()

const emit = defineEmits<{
  'item-click': [itemId: number]
}>()

// Attuned items: filter by is_attuned boolean (can be in any slot)
const attunedItems = computed(() =>
  props.equipment.filter(e => e.is_attuned === true)
)

// Helper functions
function getItemName(equipment: CharacterEquipment): string {
  if (equipment.custom_name) return equipment.custom_name
  const item = equipment.item as { name?: string } | null
  return item?.name ?? 'Unknown'
}

function handlePaperdollClick(itemId: number) {
  emit('item-click', itemId)
}
</script>

<template>
  <div class="space-y-4">
    <!-- Equipment Paperdoll -->
    <CharacterInventoryEquipmentPaperdoll
      data-testid="equipment-paperdoll"
      :equipment="equipment"
      @item-click="handlePaperdollClick"
    />

    <!-- Attuned Section -->
    <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4">
      <h3 class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-3">
        Attuned
        <span class="text-gray-400 dark:text-gray-500">({{ attunedItems.length }}/3)</span>
      </h3>
      <!-- Attuned items list -->
      <div
        v-if="attunedItems.length > 0"
        class="space-y-2"
      >
        <button
          v-for="item in attunedItems"
          :key="item.id"
          :data-testid="`attuned-item-${item.id}`"
          class="block w-full text-left text-sm font-medium text-gray-900 dark:text-white hover:text-primary transition-colors"
          @click="emit('item-click', item.id)"
        >
          {{ getItemName(item) }}
        </button>
      </div>
      <!-- Empty state -->
      <span
        v-else
        data-testid="attuned-empty"
        class="text-sm text-gray-400 dark:text-gray-500 italic"
      >
        None
      </span>
    </div>
  </div>
</template>
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/components/character/inventory/EquipmentStatus.test.ts
```

Expected: All tests PASS

### Step 5: Commit

```bash
git add app/components/character/inventory/EquipmentStatus.vue tests/components/character/inventory/EquipmentStatus.test.ts
git commit -m "refactor(inventory): integrate paperdoll into EquipmentStatus

- Replace wielded/armor sections with paperdoll component
- Keep attunement section separate
- Simplify component structure"
```

---

## Task 5: Update Inventory Page Layout and Equip Flow

**Files:**
- Modify: `app/pages/characters/[publicId]/inventory.vue`
- Modify: `app/components/character/inventory/ItemTable.vue`

### Step 1: Update inventory page layout to 2fr/1fr

In `inventory.vue`, change:
```vue
<!-- Old -->
<div class="grid lg:grid-cols-[1fr_280px] gap-6 mt-6">

<!-- New -->
<div class="grid lg:grid-cols-[2fr_1fr] gap-6 mt-6">
```

### Step 2: Add slot picker modal state and handler

In `inventory.vue`, add:
```typescript
// Slot picker modal state
const isSlotPickerOpen = ref(false)
const slotPickerItem = ref<CharacterEquipment | null>(null)
const slotPickerValidSlots = ref<EquipmentSlot[]>([])
const slotPickerSuggestedSlot = ref<EquipmentSlot | null>(null)
const isEquipping = ref(false)
```

### Step 3: Update ItemTable to emit equip-with-picker

In `ItemTable.vue`, update handleEquip:
```typescript
import { getValidSlots, getDefaultSlot, needsSlotPicker, guessSlotFromName } from '~/utils/equipmentSlots'

function handleEquip(item: CharacterEquipment) {
  const itemData = item.item as { item_type?: string; name?: string } | null
  const itemType = itemData?.item_type ?? null
  const itemName = item.custom_name || itemData?.name || ''

  // Check if we need slot picker
  if (needsSlotPicker(itemType)) {
    const validSlots = getValidSlots(itemType)
    const suggestedSlot = guessSlotFromName(itemName)
    emit('equip-with-picker', item.id, validSlots, suggestedSlot)
    return
  }

  // Auto-equip to default slot
  const defaultSlot = getDefaultSlot(itemType)
  if (defaultSlot) {
    emit('equip', item.id, defaultSlot)
  }
}
```

### Step 4: Add new emit to ItemTable

```typescript
const emit = defineEmits<{
  'item-click': [item: CharacterEquipment]
  'equip': [itemId: number, slot: string]
  'equip-with-picker': [itemId: number, validSlots: string[], suggestedSlot: string | null]
  'unequip': [itemId: number]
  // ... other emits
}>()
```

### Step 5: Handle equip-with-picker in inventory page

```typescript
function handleEquipWithPicker(itemId: number, validSlots: EquipmentSlot[], suggestedSlot: EquipmentSlot | null) {
  const item = equipment.value.find(e => e.id === itemId)
  if (!item) return

  slotPickerItem.value = item
  slotPickerValidSlots.value = validSlots
  slotPickerSuggestedSlot.value = suggestedSlot
  isSlotPickerOpen.value = true
}

async function handleSlotSelected(slot: EquipmentSlot) {
  if (!slotPickerItem.value) return

  isEquipping.value = true
  try {
    await equipItem(slotPickerItem.value.id, slot)
    toast.add({ title: 'Item equipped!', color: 'success' })
    isSlotPickerOpen.value = false
    slotPickerItem.value = null
    await refreshEquipment()
  } catch (error) {
    logger.error('Failed to equip item:', error)
    toast.add({ title: 'Failed to equip item', color: 'error' })
  } finally {
    isEquipping.value = false
  }
}
```

### Step 6: Add slot picker modal to template

```vue
<!-- Slot Picker Modal -->
<CharacterInventoryEquipSlotPickerModal
  v-model:open="isSlotPickerOpen"
  :item-name="slotPickerItem?.custom_name || (slotPickerItem?.item as any)?.name || 'Item'"
  :valid-slots="slotPickerValidSlots"
  :suggested-slot="slotPickerSuggestedSlot"
  :loading="isEquipping"
  @select="handleSlotSelected"
/>
```

### Step 7: Update ItemTable template to use new handler

```vue
<CharacterInventoryItemTable
  data-testid="item-table"
  :items="equipment"
  :editable="isPlayMode"
  :search-query="searchQuery"
  @item-click="handleItemClick"
  @equip="handleEquip"
  @equip-with-picker="handleEquipWithPicker"
  @unequip="handleUnequip"
  <!-- ... other handlers -->
/>
```

### Step 8: Run all inventory tests

```bash
docker compose exec nuxt npm run test:character
```

Expected: All tests PASS

### Step 9: Commit

```bash
git add app/pages/characters/[publicId]/inventory.vue app/components/character/inventory/ItemTable.vue
git commit -m "feat(inventory): integrate slot picker into equip flow

- Change layout to 2fr/1fr grid
- Add slot picker modal for wondrous items and rings
- Smart slot detection with name pattern fallback"
```

---

## Task 6: Final Testing and Cleanup

### Step 1: Run full test suite

```bash
docker compose exec nuxt npm run test
```

Expected: All tests PASS

### Step 2: Run TypeScript check

```bash
docker compose exec nuxt npm run typecheck
```

Expected: No errors

### Step 3: Run linter

```bash
docker compose exec nuxt npm run lint:fix
```

### Step 4: Manual browser testing

1. Navigate to character inventory page
2. Verify paperdoll shows all 11 slots
3. Verify equipped items appear in correct slots
4. Click equipped item → scrolls to table
5. Equip armor → auto-assigns to armor slot
6. Equip ring → shows slot picker (ring_1/ring_2)
7. Equip wondrous item → shows slot picker with smart pre-selection
8. Test dark mode appearance

### Step 5: Final commit

```bash
git add -A
git commit -m "chore(inventory): cleanup and finalize paperdoll feature"
```

---

## Summary

| Task | Component | Tests |
|------|-----------|-------|
| 1 | `equipmentSlots.ts` utility | Slot mapping, name patterns |
| 2 | `EquipmentPaperdoll.vue` | Render slots, click behavior |
| 3 | `EquipSlotPickerModal.vue` | Slot selection, pre-selection |
| 4 | `EquipmentStatus.vue` refactor | Paperdoll integration |
| 5 | Inventory page + ItemTable | Equip flow with picker |
| 6 | Final testing | Full suite, browser |

**Total new files:** 4 (utility + 2 components + tests)
**Modified files:** 3 (EquipmentStatus, ItemTable, inventory page)
