# Inventory Tab Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a full inventory management page with item actions, equipment status sidebar, add loot/shop flows, and optional encumbrance tracking.

**Architecture:** URL-routed page at `/characters/[publicId]/inventory`. Two-column layout: scrollable item list (left) + sticky sidebar (right). Item-centric actions (click item → menu). Backend provides pre-sorted items (equipped first, grouped by type).

**Tech Stack:** Nuxt 4 | NuxtUI 4 | TypeScript | Pinia | Vitest | MSW | Playwright

**Design Document:** `docs/frontend/plans/2025-12-13-inventory-tab-design-v2.md`

**GitHub Issue:** #567

---

## Progress Summary

| Phase | Status | Notes |
|-------|--------|-------|
| 1. Foundation | ✅ COMPLETE | Tab navigation, page routing, CharacterPageHeader |
| 2. Sidebar | ✅ COMPLETE | EquipmentStatus, EncumbranceBar, Currency section |
| 3. Item Display | ✅ COMPLETE | ItemTable (grouped), ItemDetailModal (fetch on open) |
| 4. Add Loot Modal | ✅ COMPLETE | Tabbed UI (Search/Custom), full-width inputs |
| 5. Shop Modal | 🔄 NEXT | To be implemented next session |
| 6. Integration | ✅ MOSTLY DONE | Page wired up, play mode working |

**Last Updated:** 2025-12-13

---

## Phase 1: Foundation (Tab Navigation + Routing)

### Task 1.1: Tab Navigation Component

**Files:**
- Create: `app/components/character/TabNavigation.vue`
- Test: `tests/components/character/TabNavigation.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/character/TabNavigation.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import TabNavigation from '~/components/character/TabNavigation.vue'

// Mock useRoute
vi.mock('#app', async () => {
  const actual = await vi.importActual('#app')
  return {
    ...actual,
    useRoute: () => ({ path: '/characters/test-char-123/inventory' })
  }
})

describe('TabNavigation', () => {
  it('renders all tabs with correct routes', async () => {
    const wrapper = await mountSuspended(TabNavigation, {
      props: { publicId: 'test-char-123' }
    })

    expect(wrapper.text()).toContain('Overview')
    expect(wrapper.text()).toContain('Inventory')

    const links = wrapper.findAll('a')
    expect(links.some(l => l.attributes('href')?.includes('/characters/test-char-123'))).toBe(true)
    expect(links.some(l => l.attributes('href')?.includes('/characters/test-char-123/inventory'))).toBe(true)
  })

  it('highlights active tab based on current route', async () => {
    const wrapper = await mountSuspended(TabNavigation, {
      props: { publicId: 'test-char-123' }
    })

    // Inventory tab should be active (mocked route)
    const activeTab = wrapper.find('[data-testid="tab-inventory"]')
    expect(activeTab.classes()).toContain('text-primary')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/TabNavigation.test.ts`
Expected: FAIL with "Cannot find module"

**Step 3: Write minimal implementation**

```vue
<!-- app/components/character/TabNavigation.vue -->
<script setup lang="ts">
interface Props {
  publicId: string
  isSpellcaster?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  isSpellcaster: false
})

const route = useRoute()

const tabs = computed(() => {
  const baseTabs = [
    { key: 'overview', label: 'Overview', to: `/characters/${props.publicId}` },
    { key: 'inventory', label: 'Inventory', to: `/characters/${props.publicId}/inventory` }
  ]

  if (props.isSpellcaster) {
    baseTabs.push({ key: 'spells', label: 'Spells', to: `/characters/${props.publicId}/spells` })
  }

  // Future: battle tab
  return baseTabs
})

function isActive(tab: { key: string, to: string }): boolean {
  if (tab.key === 'overview') {
    // Overview is active only on exact match (not /inventory, /spells, etc.)
    return route.path === tab.to
  }
  return route.path.startsWith(tab.to)
}
</script>

<template>
  <nav class="border-b border-gray-200 dark:border-gray-700 mb-6">
    <div class="flex gap-6">
      <NuxtLink
        v-for="tab in tabs"
        :key="tab.key"
        :to="tab.to"
        :data-testid="`tab-${tab.key}`"
        class="pb-3 text-sm font-medium border-b-2 transition-colors"
        :class="isActive(tab)
          ? 'border-primary text-primary'
          : 'border-transparent text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200'"
      >
        {{ tab.label }}
      </NuxtLink>
    </div>
  </nav>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/TabNavigation.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/TabNavigation.vue tests/components/character/TabNavigation.test.ts
git commit -m "feat(inventory): add TabNavigation component"
```

---

### Task 1.2: Inventory Page Skeleton

**Files:**
- Create: `app/pages/characters/[publicId]/inventory.vue`
- Test: `tests/pages/characters/inventory.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/pages/characters/inventory.test.ts
import { describe, it, expect, beforeEach, afterEach, beforeAll, afterAll } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import InventoryPage from '~/pages/characters/[publicId]/inventory.vue'
import { server } from '@/tests/msw/server'

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())

describe('Inventory Page', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders tab navigation and main layout', async () => {
    const wrapper = await mountSuspended(InventoryPage, {
      route: { params: { publicId: 'test-char-123' } }
    })

    // Tab navigation should be present
    expect(wrapper.find('[data-testid="tab-navigation"]').exists()).toBe(true)

    // Two-column layout should be present
    expect(wrapper.find('[data-testid="inventory-layout"]').exists()).toBe(true)
  })

  it('shows loading state initially', async () => {
    const wrapper = await mountSuspended(InventoryPage, {
      route: { params: { publicId: 'test-char-123' } }
    })

    // Should show skeleton or loading indicator
    expect(wrapper.find('[data-testid="loading-skeleton"]').exists()).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/pages/characters/inventory.test.ts`
Expected: FAIL with "Cannot find module"

**Step 3: Write minimal implementation**

```vue
<!-- app/pages/characters/[publicId]/inventory.vue -->
<script setup lang="ts">
/**
 * Inventory Management Page
 *
 * Full inventory UI with item actions, equipment status sidebar,
 * add loot/shop modals, and optional encumbrance tracking.
 *
 * @see Design: docs/frontend/plans/2025-12-13-inventory-tab-design-v2.md
 */

const route = useRoute()
const publicId = computed(() => route.params.publicId as string)

// Fetch character data for page context
const { apiFetch } = useApi()
const { data: characterData, pending: characterPending } = useAsyncData(
  `inventory-character-${publicId.value}`,
  () => apiFetch<{ data: { id: number, name: string } }>(`/characters/${publicId.value}`)
)

// Fetch equipment data
const { data: equipmentData, pending: equipmentPending, refresh: refreshEquipment } = useAsyncData(
  `inventory-equipment-${publicId.value}`,
  () => apiFetch<{ data: unknown[] }>(`/characters/${publicId.value}/equipment`)
)

// Fetch stats for carrying capacity
const { data: statsData, pending: statsPending } = useAsyncData(
  `inventory-stats-${publicId.value}`,
  () => apiFetch<{ data: { carrying_capacity?: number, push_drag_lift?: number, spellcasting?: unknown } }>(
    `/characters/${publicId.value}/stats`
  )
)

const loading = computed(() => characterPending.value || equipmentPending.value || statsPending.value)
const character = computed(() => characterData.value?.data ?? null)
const equipment = computed(() => equipmentData.value?.data ?? [])
const stats = computed(() => statsData.value?.data ?? null)
const isSpellcaster = computed(() => !!stats.value?.spellcasting)

useSeoMeta({
  title: () => character.value ? `${character.value.name} - Inventory` : 'Inventory'
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-6xl">
    <!-- Back Link -->
    <div class="mb-4">
      <UButton
        :to="`/characters/${publicId}`"
        variant="ghost"
        icon="i-heroicons-arrow-left"
      >
        Back to Character
      </UButton>
    </div>

    <!-- Tab Navigation -->
    <CharacterTabNavigation
      data-testid="tab-navigation"
      :public-id="publicId"
      :is-spellcaster="isSpellcaster"
    />

    <!-- Loading State -->
    <div
      v-if="loading"
      data-testid="loading-skeleton"
      class="space-y-4"
    >
      <USkeleton class="h-12 w-full" />
      <div class="grid lg:grid-cols-[1fr_280px] gap-6">
        <USkeleton class="h-96" />
        <USkeleton class="h-96" />
      </div>
    </div>

    <!-- Main Content -->
    <div
      v-else
      data-testid="inventory-layout"
      class="grid lg:grid-cols-[1fr_280px] gap-6"
    >
      <!-- Left Column: Item List -->
      <div class="space-y-4">
        <!-- Search -->
        <UInput
          placeholder="Search items..."
          icon="i-heroicons-magnifying-glass"
          data-testid="item-search"
        />

        <!-- Item List Placeholder -->
        <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-8 text-center text-gray-500">
          Item list will go here
          <br>
          {{ equipment.length }} items loaded
        </div>

        <!-- Action Buttons -->
        <div class="flex gap-3">
          <UButton
            data-testid="add-loot-btn"
            icon="i-heroicons-plus"
          >
            Add Loot
          </UButton>
          <UButton
            data-testid="shop-btn"
            variant="outline"
            icon="i-heroicons-shopping-cart"
          >
            Shop
          </UButton>
        </div>
      </div>

      <!-- Right Column: Sidebar -->
      <div class="space-y-4">
        <!-- Sidebar Placeholder -->
        <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4">
          <p class="text-sm text-gray-500">Equipment status sidebar</p>
        </div>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/pages/characters/inventory.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/pages/characters/[publicId]/inventory.vue tests/pages/characters/inventory.test.ts
git commit -m "feat(inventory): add inventory page skeleton with routing"
```

---

### Task 1.3: Remove Equipment Tab from Main Sheet

**Files:**
- Modify: `app/pages/characters/[publicId]/index.vue` (lines ~1064-1078)

**Step 1: Update tabItems computed to remove Equipment**

In `app/pages/characters/[publicId]/index.vue`, find the `tabItems` computed property and remove the Equipment entry. Also add TabNavigation component.

**Step 2: Add TabNavigation to template**

Add `<CharacterTabNavigation>` after the back button, before the loading state.

**Step 3: Remove Equipment tab from UTabs**

Remove the `{ label: 'Equipment', slot: 'equipment', icon: 'i-heroicons-briefcase' }` from tabItems and the corresponding `<template #equipment>` slot.

**Step 4: Run tests to verify nothing broke**

Run: `docker compose exec nuxt npm run test:character`
Expected: PASS (some tests may need minor adjustments for removed tab)

**Step 5: Commit**

```bash
git add app/pages/characters/[publicId]/index.vue
git commit -m "refactor(inventory): move equipment to dedicated page, add tab navigation"
```

---

## Phase 2: Equipment Status Sidebar

### Task 2.1: EquipmentStatus Component (Read-Only Display)

**Files:**
- Create: `app/components/character/inventory/EquipmentStatus.vue`
- Test: `tests/components/character/inventory/EquipmentStatus.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/character/inventory/EquipmentStatus.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import EquipmentStatus from '~/components/character/inventory/EquipmentStatus.vue'

const mockEquipment = [
  { id: 1, item: { name: 'Longsword' }, equipped: true, location: 'main_hand', quantity: 1 },
  { id: 2, item: { name: 'Shield' }, equipped: true, location: 'off_hand', quantity: 1 },
  { id: 3, item: { name: 'Chain Mail' }, equipped: true, location: 'worn', quantity: 1 },
  { id: 4, item: { name: 'Ring of Protection' }, equipped: true, location: 'attuned', quantity: 1 }
]

describe('EquipmentStatus', () => {
  it('displays wielded items (main hand and off hand)', async () => {
    const wrapper = await mountSuspended(EquipmentStatus, {
      props: { equipment: mockEquipment }
    })

    expect(wrapper.text()).toContain('Longsword')
    expect(wrapper.text()).toContain('Shield')
    expect(wrapper.text()).toContain('Main Hand')
    expect(wrapper.text()).toContain('Off Hand')
  })

  it('displays worn armor with AC', async () => {
    const equipmentWithAC = [
      { id: 3, item: { name: 'Chain Mail', armor_class: 16 }, equipped: true, location: 'worn', quantity: 1 }
    ]

    const wrapper = await mountSuspended(EquipmentStatus, {
      props: { equipment: equipmentWithAC }
    })

    expect(wrapper.text()).toContain('Chain Mail')
    expect(wrapper.text()).toContain('AC 16')
  })

  it('displays attuned items with count', async () => {
    const wrapper = await mountSuspended(EquipmentStatus, {
      props: { equipment: mockEquipment }
    })

    expect(wrapper.text()).toContain('Attuned')
    expect(wrapper.text()).toContain('1/3')
    expect(wrapper.text()).toContain('Ring of Protection')
  })

  it('shows empty slots when nothing equipped', async () => {
    const wrapper = await mountSuspended(EquipmentStatus, {
      props: { equipment: [] }
    })

    expect(wrapper.text()).toContain('Empty')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/inventory/EquipmentStatus.test.ts`
Expected: FAIL

**Step 3: Write minimal implementation**

```vue
<!-- app/components/character/inventory/EquipmentStatus.vue -->
<script setup lang="ts">
import type { CharacterEquipment } from '~/types/character'

interface Props {
  equipment: CharacterEquipment[]
}

const props = defineProps<Props>()

const emit = defineEmits<{
  'item-click': [itemId: number]
}>()

// Filter equipped items by location
const mainHand = computed(() =>
  props.equipment.find(e => e.equipped && e.location === 'main_hand')
)

const offHand = computed(() =>
  props.equipment.find(e => e.equipped && e.location === 'off_hand')
)

const wornArmor = computed(() =>
  props.equipment.find(e => e.equipped && e.location === 'worn')
)

const attunedItems = computed(() =>
  props.equipment.filter(e => e.equipped && e.location === 'attuned')
)

function getItemName(item: CharacterEquipment): string {
  if (item.custom_name) return item.custom_name
  const itemData = item.item as { name?: string } | null
  return itemData?.name ?? 'Unknown'
}

function getArmorClass(item: CharacterEquipment): number | null {
  const itemData = item.item as { armor_class?: number } | null
  return itemData?.armor_class ?? null
}
</script>

<template>
  <div class="space-y-4">
    <!-- Wielded -->
    <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4">
      <h3 class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-3">
        Wielded
      </h3>
      <div class="space-y-2">
        <div class="flex justify-between items-center">
          <span class="text-sm text-gray-500 dark:text-gray-400">Main Hand</span>
          <button
            v-if="mainHand"
            class="text-sm font-medium text-gray-900 dark:text-white hover:text-primary transition-colors"
            @click="emit('item-click', mainHand.id)"
          >
            {{ getItemName(mainHand) }}
          </button>
          <span
            v-else
            class="text-sm text-gray-400 dark:text-gray-500 italic"
          >Empty</span>
        </div>
        <div class="flex justify-between items-center">
          <span class="text-sm text-gray-500 dark:text-gray-400">Off Hand</span>
          <button
            v-if="offHand"
            class="text-sm font-medium text-gray-900 dark:text-white hover:text-primary transition-colors"
            @click="emit('item-click', offHand.id)"
          >
            {{ getItemName(offHand) }}
          </button>
          <span
            v-else
            class="text-sm text-gray-400 dark:text-gray-500 italic"
          >Empty</span>
        </div>
      </div>
    </div>

    <!-- Armor -->
    <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4">
      <h3 class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-3">
        Armor
      </h3>
      <div
        v-if="wornArmor"
        class="flex justify-between items-center"
      >
        <button
          class="text-sm font-medium text-gray-900 dark:text-white hover:text-primary transition-colors"
          @click="emit('item-click', wornArmor.id)"
        >
          {{ getItemName(wornArmor) }}
        </button>
        <span
          v-if="getArmorClass(wornArmor)"
          class="text-sm text-gray-500 dark:text-gray-400"
        >
          AC {{ getArmorClass(wornArmor) }}
        </span>
      </div>
      <span
        v-else
        class="text-sm text-gray-400 dark:text-gray-500 italic"
      >No armor</span>
    </div>

    <!-- Attuned -->
    <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4">
      <h3 class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-3">
        Attuned
        <span class="text-gray-400">({{ attunedItems.length }}/3)</span>
      </h3>
      <div class="space-y-2">
        <button
          v-for="item in attunedItems"
          :key="item.id"
          class="block w-full text-left text-sm font-medium text-gray-900 dark:text-white hover:text-primary transition-colors"
          @click="emit('item-click', item.id)"
        >
          {{ getItemName(item) }}
        </button>
        <span
          v-for="i in (3 - attunedItems.length)"
          :key="`empty-${i}`"
          class="block text-sm text-gray-400 dark:text-gray-500 italic"
        >
          Empty
        </span>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/inventory/EquipmentStatus.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/inventory/EquipmentStatus.vue tests/components/character/inventory/EquipmentStatus.test.ts
git commit -m "feat(inventory): add EquipmentStatus sidebar component"
```

---

### Task 2.2: EncumbranceBar Component

**Files:**
- Create: `app/components/character/inventory/EncumbranceBar.vue`
- Test: `tests/components/character/inventory/EncumbranceBar.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/character/inventory/EncumbranceBar.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import EncumbranceBar from '~/components/character/inventory/EncumbranceBar.vue'

// Mock localStorage
const localStorageMock = {
  getItem: vi.fn(),
  setItem: vi.fn()
}
vi.stubGlobal('localStorage', localStorageMock)

describe('EncumbranceBar', () => {
  beforeEach(() => {
    localStorageMock.getItem.mockClear()
    localStorageMock.setItem.mockClear()
  })

  it('displays current weight and capacity', async () => {
    const wrapper = await mountSuspended(EncumbranceBar, {
      props: { currentWeight: 45, carryingCapacity: 150, publicId: 'test-123' }
    })

    expect(wrapper.text()).toContain('45')
    expect(wrapper.text()).toContain('150')
    expect(wrapper.text()).toContain('lbs')
  })

  it('shows green bar when under 66% capacity', async () => {
    const wrapper = await mountSuspended(EncumbranceBar, {
      props: { currentWeight: 50, carryingCapacity: 150, publicId: 'test-123' }
    })

    const bar = wrapper.find('[data-testid="encumbrance-fill"]')
    expect(bar.classes()).toContain('bg-success')
  })

  it('shows yellow bar when between 67-99% capacity', async () => {
    const wrapper = await mountSuspended(EncumbranceBar, {
      props: { currentWeight: 120, carryingCapacity: 150, publicId: 'test-123' }
    })

    const bar = wrapper.find('[data-testid="encumbrance-fill"]')
    expect(bar.classes()).toContain('bg-warning')
  })

  it('shows red bar when at or over capacity', async () => {
    const wrapper = await mountSuspended(EncumbranceBar, {
      props: { currentWeight: 160, carryingCapacity: 150, publicId: 'test-123' }
    })

    const bar = wrapper.find('[data-testid="encumbrance-fill"]')
    expect(bar.classes()).toContain('bg-error')
  })

  it('persists toggle state to localStorage', async () => {
    const wrapper = await mountSuspended(EncumbranceBar, {
      props: { currentWeight: 45, carryingCapacity: 150, publicId: 'test-123' }
    })

    const toggle = wrapper.find('[data-testid="encumbrance-toggle"]')
    await toggle.trigger('click')

    expect(localStorageMock.setItem).toHaveBeenCalledWith(
      'encumbrance-tracking-test-123',
      expect.any(String)
    )
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/inventory/EncumbranceBar.test.ts`
Expected: FAIL

**Step 3: Write minimal implementation**

```vue
<!-- app/components/character/inventory/EncumbranceBar.vue -->
<script setup lang="ts">
interface Props {
  currentWeight: number
  carryingCapacity: number
  publicId: string
}

const props = defineProps<Props>()

// Toggle state persisted to localStorage
const isEnabled = ref(false)
const storageKey = computed(() => `encumbrance-tracking-${props.publicId}`)

onMounted(() => {
  const saved = localStorage.getItem(storageKey.value)
  isEnabled.value = saved === 'true'
})

function toggleEnabled() {
  isEnabled.value = !isEnabled.value
  localStorage.setItem(storageKey.value, String(isEnabled.value))
}

// Calculate percentage and color
const percentage = computed(() => {
  if (props.carryingCapacity <= 0) return 0
  return Math.min((props.currentWeight / props.carryingCapacity) * 100, 100)
})

const barColor = computed(() => {
  const pct = (props.currentWeight / props.carryingCapacity) * 100
  if (pct >= 100) return 'bg-error'
  if (pct >= 67) return 'bg-warning'
  return 'bg-success'
})

const isOverCapacity = computed(() => props.currentWeight > props.carryingCapacity)
</script>

<template>
  <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4">
    <div class="flex items-center justify-between mb-3">
      <h3 class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">
        Encumbrance
      </h3>
      <button
        data-testid="encumbrance-toggle"
        class="text-xs text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200"
        @click="toggleEnabled"
      >
        {{ isEnabled ? 'Hide' : 'Show' }}
      </button>
    </div>

    <div
      v-if="isEnabled"
      class="space-y-2"
    >
      <!-- Weight Display -->
      <div class="flex justify-between text-sm">
        <span :class="isOverCapacity ? 'text-error font-medium' : 'text-gray-600 dark:text-gray-300'">
          {{ currentWeight }} / {{ carryingCapacity }} lbs
        </span>
      </div>

      <!-- Progress Bar -->
      <div class="h-2 bg-gray-200 dark:bg-gray-700 rounded-full overflow-hidden">
        <div
          data-testid="encumbrance-fill"
          :class="['h-full transition-all duration-300', barColor]"
          :style="{ width: `${percentage}%` }"
        />
      </div>
    </div>

    <p
      v-else
      class="text-sm text-gray-400 dark:text-gray-500 italic"
    >
      Click "Show" to track weight
    </p>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/inventory/EncumbranceBar.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/inventory/EncumbranceBar.vue tests/components/character/inventory/EncumbranceBar.test.ts
git commit -m "feat(inventory): add EncumbranceBar component with localStorage toggle"
```

---

## Phase 3: Item List Components

### Task 3.1: ItemRow Component (Expandable with Actions)

**Files:**
- Create: `app/components/character/inventory/ItemRow.vue`
- Test: `tests/components/character/inventory/ItemRow.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/character/inventory/ItemRow.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import ItemRow from '~/components/character/inventory/ItemRow.vue'

const mockItem = {
  id: 1,
  item: {
    name: 'Longsword',
    description: 'A versatile martial weapon',
    weight: 3,
    item_type: { code: 'W', name: 'Weapon' }
  },
  item_slug: 'phb:longsword',
  custom_name: null,
  custom_description: null,
  quantity: 1,
  equipped: true,
  location: 'main_hand'
}

describe('ItemRow', () => {
  it('displays item name and quantity', async () => {
    const wrapper = await mountSuspended(ItemRow, {
      props: { item: mockItem, editable: true }
    })

    expect(wrapper.text()).toContain('Longsword')
  })

  it('shows equipped badge when item is equipped', async () => {
    const wrapper = await mountSuspended(ItemRow, {
      props: { item: mockItem, editable: true }
    })

    expect(wrapper.text()).toContain('Main Hand')
  })

  it('expands to show details when clicked', async () => {
    const wrapper = await mountSuspended(ItemRow, {
      props: { item: mockItem, editable: true }
    })

    // Initially collapsed
    expect(wrapper.find('[data-testid="item-details"]').exists()).toBe(false)

    // Click to expand
    await wrapper.find('[data-testid="item-row"]').trigger('click')

    // Now expanded
    expect(wrapper.find('[data-testid="item-details"]').exists()).toBe(true)
    expect(wrapper.text()).toContain('A versatile martial weapon')
  })

  it('shows action menu when editable', async () => {
    const wrapper = await mountSuspended(ItemRow, {
      props: { item: mockItem, editable: true }
    })

    expect(wrapper.find('[data-testid="item-actions"]').exists()).toBe(true)
  })

  it('hides action menu when not editable', async () => {
    const wrapper = await mountSuspended(ItemRow, {
      props: { item: mockItem, editable: false }
    })

    expect(wrapper.find('[data-testid="item-actions"]').exists()).toBe(false)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/inventory/ItemRow.test.ts`
Expected: FAIL

**Step 3: Write minimal implementation**

```vue
<!-- app/components/character/inventory/ItemRow.vue -->
<script setup lang="ts">
import type { CharacterEquipment } from '~/types/character'

interface Props {
  item: CharacterEquipment
  editable: boolean
}

const props = defineProps<Props>()

const emit = defineEmits<{
  'equip': [itemId: number, slot: string]
  'unequip': [itemId: number]
  'sell': [itemId: number]
  'drop': [itemId: number]
  'edit-qty': [itemId: number]
}>()

const isExpanded = ref(false)

function toggleExpand() {
  isExpanded.value = !isExpanded.value
}

// Item data helpers
const itemData = computed(() => props.item.item as {
  name?: string
  description?: string
  weight?: number
  item_type?: { code: string, name: string }
  armor_class?: number
  damage?: string
} | null)

const displayName = computed(() => {
  if (props.item.custom_name) return props.item.custom_name
  return itemData.value?.name ?? 'Unknown Item'
})

const description = computed(() => {
  if (props.item.custom_description) return props.item.custom_description
  return itemData.value?.description ?? ''
})

const locationBadge = computed(() => {
  if (!props.item.equipped) return null
  switch (props.item.location) {
    case 'main_hand': return 'Main Hand'
    case 'off_hand': return 'Off Hand'
    case 'worn': return 'Worn'
    case 'attuned': return 'Attuned'
    default: return 'Equipped'
  }
})

// Icon based on item type
const itemIcon = computed(() => {
  const typeCode = itemData.value?.item_type?.code
  switch (typeCode) {
    case 'W': return 'i-heroicons-bolt'
    case 'A': return 'i-heroicons-shield-check'
    case 'P': return 'i-heroicons-beaker'
    default: return 'i-heroicons-cube'
  }
})
</script>

<template>
  <div class="border border-gray-200 dark:border-gray-700 rounded-lg overflow-hidden">
    <!-- Row Header (always visible) -->
    <div
      data-testid="item-row"
      class="flex items-center gap-3 p-3 cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800/50 transition-colors"
      @click="toggleExpand"
    >
      <UIcon
        :name="itemIcon"
        class="w-5 h-5 text-gray-400 flex-shrink-0"
      />

      <div class="flex-1 min-w-0">
        <div class="flex items-center gap-2">
          <span class="font-medium text-gray-900 dark:text-white truncate">
            {{ displayName }}
          </span>
          <UBadge
            v-if="locationBadge"
            color="primary"
            variant="subtle"
            size="xs"
          >
            {{ locationBadge }}
          </UBadge>
        </div>
      </div>

      <span
        v-if="item.quantity > 1"
        class="text-sm text-gray-500 dark:text-gray-400"
      >
        ×{{ item.quantity }}
      </span>

      <!-- Action Menu -->
      <UDropdown
        v-if="editable"
        data-testid="item-actions"
        :items="[[
          { label: 'Unequip', icon: 'i-heroicons-arrow-down-tray', click: () => emit('unequip', item.id) },
          { label: 'Edit Qty', icon: 'i-heroicons-pencil', click: () => emit('edit-qty', item.id) },
          { label: 'Sell', icon: 'i-heroicons-currency-dollar', click: () => emit('sell', item.id) },
          { label: 'Drop', icon: 'i-heroicons-trash', click: () => emit('drop', item.id) }
        ]]"
        @click.stop
      >
        <UButton
          variant="ghost"
          icon="i-heroicons-ellipsis-vertical"
          size="xs"
        />
      </UDropdown>

      <UIcon
        :name="isExpanded ? 'i-heroicons-chevron-up' : 'i-heroicons-chevron-down'"
        class="w-5 h-5 text-gray-400"
      />
    </div>

    <!-- Expanded Details -->
    <div
      v-if="isExpanded"
      data-testid="item-details"
      class="px-3 pb-3 pt-0 border-t border-gray-200 dark:border-gray-700 bg-gray-50 dark:bg-gray-800/30"
    >
      <p
        v-if="description"
        class="text-sm text-gray-600 dark:text-gray-300 mt-3"
      >
        {{ description }}
      </p>

      <div class="flex flex-wrap gap-4 mt-3 text-sm text-gray-500 dark:text-gray-400">
        <span v-if="itemData?.weight">
          Weight: {{ itemData.weight }} lbs
        </span>
        <span v-if="itemData?.damage">
          Damage: {{ itemData.damage }}
        </span>
        <span v-if="itemData?.armor_class">
          AC: {{ itemData.armor_class }}
        </span>
      </div>

      <!-- Action Buttons (when expanded and editable) -->
      <div
        v-if="editable"
        class="flex flex-wrap gap-2 mt-4"
      >
        <UButton
          v-if="item.equipped"
          size="xs"
          variant="outline"
          @click="emit('unequip', item.id)"
        >
          Unequip
        </UButton>
        <UButton
          size="xs"
          variant="outline"
          @click="emit('edit-qty', item.id)"
        >
          Edit Qty
        </UButton>
        <UButton
          size="xs"
          variant="outline"
          @click="emit('sell', item.id)"
        >
          Sell
        </UButton>
        <UButton
          size="xs"
          variant="outline"
          color="error"
          @click="emit('drop', item.id)"
        >
          Drop
        </UButton>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/inventory/ItemRow.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/inventory/ItemRow.vue tests/components/character/inventory/ItemRow.test.ts
git commit -m "feat(inventory): add ItemRow component with expand/collapse and actions"
```

---

### Task 3.2: ItemList Component (Search + Render Groups)

**Files:**
- Create: `app/components/character/inventory/ItemList.vue`
- Test: `tests/components/character/inventory/ItemList.test.ts`

*Follow same TDD pattern: Write failing test → Run to verify fail → Implement → Run to verify pass → Commit*

Key features:
- Search input filters items by name
- Renders items in order received from backend (no client-side grouping)
- Emits action events to parent
- Shows empty state when no items

---

## Phase 4: Add Item Modals

### Task 4.1: AddLootModal Component

**Files:**
- Create: `app/components/character/inventory/AddLootModal.vue`
- Test: `tests/components/character/inventory/AddLootModal.test.ts`

Key features:
- Search items from `/items` endpoint
- Select item, set quantity
- Custom item toggle (name + description fields)
- Emit `add` event with item data

---

### Task 4.2: ShopModal Component

**Files:**
- Create: `app/components/character/inventory/ShopModal.vue`
- Test: `tests/components/character/inventory/ShopModal.test.ts`

Key features:
- Search items with type filters
- Show prices from item data
- Editable "Your Price" field
- Currency preview (current → after)
- Insufficient funds warning
- Emit `purchase` event

---

## Phase 5: API Routes & Composables

### Task 5.1: Equipment PATCH Route (Equip/Unequip)

**Files:**
- Create: `server/api/characters/[id]/equipment/[equipmentId].patch.ts`

```typescript
// server/api/characters/[id]/equipment/[equipmentId].patch.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const equipmentId = getRouterParam(event, 'equipmentId')
  const body = await readBody(event)

  const data = await $fetch(
    `${config.apiBaseServer}/characters/${id}/equipment/${equipmentId}`,
    { method: 'PATCH', body }
  )
  return data
})
```

---

### Task 5.2: useInventoryActions Composable

**Files:**
- Create: `app/composables/useInventoryActions.ts`
- Test: `tests/composables/useInventoryActions.test.ts`

Key features:
- `equipItem(itemId, slot)` - PATCH with location
- `unequipItem(itemId)` - PATCH with location: 'inventory'
- `sellItem(itemId, price)` - DELETE + currency PATCH
- `dropItem(itemId)` - DELETE only
- `addItem(itemSlug, quantity)` - POST equipment
- `updateQuantity(itemId, quantity)` - PATCH quantity

---

## Phase 6: Integration & Polish

### Task 6.1: Wire Up Inventory Page

Connect all components in `inventory.vue`:
- EquipmentStatus with real equipment data
- EncumbranceBar with calculated weight
- ItemList with search and actions
- AddLootModal and ShopModal
- Play mode integration

---

### Task 6.2: E2E Tests (Playwright)

**Files:**
- Create: `tests/e2e/inventory.spec.ts`

Key scenarios:
- Navigate to inventory page
- Search and filter items
- Expand item to see details
- Equip/unequip item (play mode)
- Add loot item
- Purchase from shop

---

## Backend Coordination

**Create GitHub Issue for backend team:**

The following backend changes are needed:

1. **`location` field values**: Support `main_hand`, `off_hand`, `worn`, `attuned`, `inventory`
2. **Pre-sorted response**: `/characters/{id}/equipment` returns items sorted: equipped first, then grouped by type
3. **Slot enforcement**: Validation for slot rules (one armor, 3 attunement max)
4. **Auto-unequip**: When equipping to occupied slot, previous item's location → `inventory`

---

## Summary

| Phase | Tasks | Estimated Time |
|-------|-------|----------------|
| 1. Foundation | Tab navigation, page routing | 2-3 hours |
| 2. Sidebar | EquipmentStatus, EncumbranceBar | 2 hours |
| 3. Item List | ItemRow, ItemList | 3-4 hours |
| 4. Modals | AddLootModal, ShopModal | 4-5 hours |
| 5. API/Composables | Routes, useInventoryActions | 2-3 hours |
| 6. Integration | Wire up, E2E tests | 3-4 hours |

**Total: ~16-21 hours**

Each task follows TDD: Write test → Verify fail → Implement → Verify pass → Commit
