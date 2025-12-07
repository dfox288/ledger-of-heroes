# Spell List Generator Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build an MVP spell list generator that helps players choose spells for their D&D character based on class and level.

**Architecture:** Single-page app at `/spells/list-generator` with three sections: character setup (class/level), spell selection (checkboxes grouped by level), and summary sidebar. Uses new `useSpellListGenerator` composable for state management and LocalStorage persistence.

**Tech Stack:** Nuxt 4, Vue 3 Composition API, NuxtUI 4, TypeScript, Vitest

---

## Task 1: Create Spell List Generator Composable (TDD)

**Files:**
- Create: `app/composables/useSpellListGenerator.ts`
- Create: `tests/composables/useSpellListGenerator.test.ts`

**Step 1: Write the failing test for spell slot calculation**

Create `tests/composables/useSpellListGenerator.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { useSpellListGenerator } from '~/composables/useSpellListGenerator'

describe('useSpellListGenerator', () => {
  it('calculates spell slots from level progression', () => {
    const { setClassData, characterLevel, spellSlots } = useSpellListGenerator()

    // Mock class with level_progression
    const mockClass = {
      id: 1,
      slug: 'wizard',
      name: 'Wizard',
      level_progression: [
        {
          level: 1,
          cantrips_known: 3,
          spell_slots_1st: 2,
          spell_slots_2nd: 0,
          spell_slots_3rd: 0
        },
        {
          level: 2,
          cantrips_known: 3,
          spell_slots_1st: 3,
          spell_slots_2nd: 0,
          spell_slots_3rd: 0
        },
        {
          level: 3,
          cantrips_known: 3,
          spell_slots_1st: 4,
          spell_slots_2nd: 2,
          spell_slots_3rd: 0
        }
      ]
    } as any

    setClassData(mockClass)
    characterLevel.value = 3

    expect(spellSlots.value).toEqual({
      cantrips: 3,
      '1st': 4,
      '2nd': 2,
      '3rd': 0
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useSpellListGenerator.test.ts`

Expected: FAIL with "Cannot find module '~/composables/useSpellListGenerator'"

**Step 3: Write minimal implementation**

Create `app/composables/useSpellListGenerator.ts`:

```typescript
import { ref, computed, type Ref, type ComputedRef } from 'vue'
import type { CharacterClass } from '~/types'

interface SpellSlots {
  cantrips: number
  [key: string]: number // '1st', '2nd', etc.
}

export interface UseSpellListGeneratorReturn {
  selectedClass: Ref<CharacterClass | null>
  characterLevel: Ref<number>
  selectedSpells: Ref<Set<number>>
  spellSlots: ComputedRef<SpellSlots>
  setClassData: (classData: CharacterClass) => void
}

export function useSpellListGenerator(): UseSpellListGeneratorReturn {
  const selectedClass = ref<CharacterClass | null>(null)
  const characterLevel = ref(1)
  const selectedSpells = ref<Set<number>>(new Set())

  const setClassData = (classData: CharacterClass) => {
    selectedClass.value = classData
  }

  const spellSlots = computed(() => {
    if (!selectedClass.value?.level_progression) {
      return { cantrips: 0 }
    }

    const progression = selectedClass.value.level_progression[characterLevel.value - 1]
    if (!progression) {
      return { cantrips: 0 }
    }

    const slots: SpellSlots = {
      cantrips: progression.cantrips_known || 0
    }

    // Map spell_slots_1st, spell_slots_2nd, etc. to '1st', '2nd', etc.
    for (let i = 1; i <= 9; i++) {
      const key = `spell_slots_${i === 1 ? '1st' : i === 2 ? '2nd' : i === 3 ? '3rd' : `${i}th`}`
      const value = (progression as any)[key] || 0
      if (value > 0 || i <= 3) { // Always include 1st-3rd
        const slotKey = i === 1 ? '1st' : i === 2 ? '2nd' : i === 3 ? '3rd' : `${i}th`
        slots[slotKey] = value
      }
    }

    return slots
  })

  return {
    selectedClass,
    characterLevel,
    selectedSpells,
    spellSlots,
    setClassData
  }
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useSpellListGenerator.test.ts`

Expected: PASS (1 test)

**Step 5: Commit**

```bash
git add app/composables/useSpellListGenerator.ts tests/composables/useSpellListGenerator.test.ts
git commit -m "feat: add useSpellListGenerator composable with spell slots calculation

- Create composable for spell list generator state
- Calculate spell slots from class level_progression
- Add test coverage for spell slot calculation
- TDD: test written first, implementation follows

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Add Max Prepared/Known Calculation (TDD)

**Files:**
- Modify: `app/composables/useSpellListGenerator.ts`
- Modify: `tests/composables/useSpellListGenerator.test.ts`

**Step 1: Write the failing test for prepared caster**

Add to `tests/composables/useSpellListGenerator.test.ts`:

```typescript
it('calculates max prepared spells for prepared casters', () => {
  const { setClassData, characterLevel, maxSpells } = useSpellListGenerator()

  const wizardClass = {
    id: 1,
    slug: 'wizard',
    name: 'Wizard',
    level_progression: [/* ... */]
  } as any

  setClassData(wizardClass)
  characterLevel.value = 5

  // Wizard: level + 3 (default modifier) = 8
  expect(maxSpells.value).toBe(8)
  expect(maxSpells.value).toBe(5 + 3)
})

it('calculates max known spells for known casters', () => {
  const { setClassData, characterLevel, maxSpells } = useSpellListGenerator()

  const bardClass = {
    id: 2,
    slug: 'bard',
    name: 'Bard',
    level_progression: [/* ... */]
  } as any

  setClassData(bardClass)
  characterLevel.value = 5

  // Bard level 5: 8 spells known (from table)
  expect(maxSpells.value).toBe(8)
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useSpellListGenerator.test.ts`

Expected: FAIL with "Property 'maxSpells' does not exist"

**Step 3: Write implementation**

Add to `app/composables/useSpellListGenerator.ts`:

```typescript
// Add to top of file
const PREPARED_CASTER_CLASSES = ['wizard', 'cleric', 'druid', 'paladin', 'artificer']

const KNOWN_SPELLS_BY_CLASS: Record<string, number[]> = {
  'bard': [4, 5, 6, 7, 8, 9, 10, 11, 12, 14, 15, 15, 16, 18, 19, 19, 20, 22, 22, 22],
  'sorcerer': [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 12, 13, 13, 14, 14, 15, 15, 15, 15],
  'warlock': [2, 3, 4, 5, 6, 7, 8, 9, 10, 10, 11, 11, 12, 12, 13, 13, 14, 14, 15, 15],
  'ranger': [0, 2, 3, 3, 4, 4, 5, 5, 6, 6, 7, 7, 8, 8, 9, 9, 10, 10, 11, 11],
  'eldritch-knight': [0, 0, 2, 3, 3, 4, 4, 4, 4, 5, 6, 6, 7, 7, 7, 8, 8, 8, 9, 9],
  'arcane-trickster': [0, 0, 2, 3, 3, 4, 4, 4, 4, 5, 6, 6, 7, 7, 7, 8, 8, 8, 9, 9]
}

const DEFAULT_ABILITY_MODIFIER = 3

// Add to composable function body
const maxSpells = computed(() => {
  if (!selectedClass.value) return 0

  const classSlug = selectedClass.value.slug
  const level = characterLevel.value

  // Prepared casters: level + ability modifier
  if (PREPARED_CASTER_CLASSES.includes(classSlug)) {
    return level + DEFAULT_ABILITY_MODIFIER
  }

  // Known casters: lookup from table
  if (KNOWN_SPELLS_BY_CLASS[classSlug]) {
    return KNOWN_SPELLS_BY_CLASS[classSlug][level - 1] || 0
  }

  // Fallback for unknown classes
  return level + DEFAULT_ABILITY_MODIFIER
})

// Add to return statement
return {
  // ... existing returns
  maxSpells
}

// Update interface
export interface UseSpellListGeneratorReturn {
  // ... existing properties
  maxSpells: ComputedRef<number>
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useSpellListGenerator.test.ts`

Expected: PASS (3 tests)

**Step 5: Commit**

```bash
git add app/composables/useSpellListGenerator.ts tests/composables/useSpellListGenerator.test.ts
git commit -m "feat: add max prepared/known spell calculation

- Support prepared casters (level + modifier)
- Support known casters (lookup from table)
- Hardcode known spells tables for 6 classes
- Default ability modifier to +3 for MVP

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Add Spell Selection Logic (TDD)

**Files:**
- Modify: `app/composables/useSpellListGenerator.ts`
- Modify: `tests/composables/useSpellListGenerator.test.ts`

**Step 1: Write the failing test for spell toggling**

Add to `tests/composables/useSpellListGenerator.test.ts`:

```typescript
it('toggles spell selection', () => {
  const { selectedSpells, toggleSpell } = useSpellListGenerator()

  toggleSpell(1)
  expect(selectedSpells.value.has(1)).toBe(true)

  toggleSpell(1)
  expect(selectedSpells.value.has(1)).toBe(false)
})

it('tracks selection count', () => {
  const { selectedSpells, toggleSpell, selectionCount } = useSpellListGenerator()

  expect(selectionCount.value).toBe(0)

  toggleSpell(1)
  toggleSpell(2)
  expect(selectionCount.value).toBe(2)

  toggleSpell(1)
  expect(selectionCount.value).toBe(1)
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useSpellListGenerator.test.ts`

Expected: FAIL

**Step 3: Write implementation**

Add to `app/composables/useSpellListGenerator.ts`:

```typescript
// Add to composable function body
const toggleSpell = (spellId: number) => {
  if (selectedSpells.value.has(spellId)) {
    selectedSpells.value.delete(spellId)
  } else {
    selectedSpells.value.add(spellId)
  }
  // Trigger reactivity
  selectedSpells.value = new Set(selectedSpells.value)
}

const selectionCount = computed(() => selectedSpells.value.size)

// Add to return statement
return {
  // ... existing
  toggleSpell,
  selectionCount
}

// Update interface
export interface UseSpellListGeneratorReturn {
  // ... existing
  toggleSpell: (spellId: number) => void
  selectionCount: ComputedRef<number>
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useSpellListGenerator.test.ts`

Expected: PASS (5 tests)

**Step 5: Commit**

```bash
git add app/composables/useSpellListGenerator.ts tests/composables/useSpellListGenerator.test.ts
git commit -m "feat: add spell selection toggle logic

- Toggle spell selection with ID
- Track selection count
- Reactive Set for spell IDs

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Add LocalStorage Persistence (TDD)

**Files:**
- Modify: `app/composables/useSpellListGenerator.ts`
- Modify: `tests/composables/useSpellListGenerator.test.ts`

**Step 1: Write the failing test for save/load**

Add to `tests/composables/useSpellListGenerator.test.ts`:

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest'

// Mock localStorage
const localStorageMock = (() => {
  let store: Record<string, string> = {}
  return {
    getItem: (key: string) => store[key] || null,
    setItem: (key: string, value: string) => { store[key] = value },
    clear: () => { store = {} }
  }
})()

global.localStorage = localStorageMock as any

describe('useSpellListGenerator - LocalStorage', () => {
  beforeEach(() => {
    localStorageMock.clear()
  })

  it('saves selections to localStorage', () => {
    const { setClassData, characterLevel, toggleSpell, saveToStorage } = useSpellListGenerator()

    const wizardClass = { slug: 'wizard', name: 'Wizard' } as any
    setClassData(wizardClass)
    characterLevel.value = 5
    toggleSpell(1)
    toggleSpell(2)

    saveToStorage()

    const saved = JSON.parse(localStorage.getItem('spell-list-wizard-5')!)
    expect(saved.classSlug).toBe('wizard')
    expect(saved.level).toBe(5)
    expect(saved.selectedSpells).toEqual([1, 2])
  })

  it('loads selections from localStorage', () => {
    const stored = {
      classSlug: 'wizard',
      className: 'Wizard',
      level: 5,
      selectedSpells: [1, 2, 3],
      savedAt: new Date().toISOString()
    }
    localStorage.setItem('spell-list-wizard-5', JSON.stringify(stored))

    const { setClassData, characterLevel, selectedSpells, loadFromStorage } = useSpellListGenerator()

    const wizardClass = { slug: 'wizard', name: 'Wizard' } as any
    setClassData(wizardClass)
    characterLevel.value = 5
    loadFromStorage()

    expect(selectedSpells.value.has(1)).toBe(true)
    expect(selectedSpells.value.has(2)).toBe(true)
    expect(selectedSpells.value.has(3)).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useSpellListGenerator.test.ts`

Expected: FAIL

**Step 3: Write implementation**

Add to `app/composables/useSpellListGenerator.ts`:

```typescript
// Add to composable function body
const getStorageKey = () => {
  if (!selectedClass.value) return null
  return `spell-list-${selectedClass.value.slug}-${characterLevel.value}`
}

const saveToStorage = () => {
  const key = getStorageKey()
  if (!key || !selectedClass.value) return

  const data = {
    classSlug: selectedClass.value.slug,
    className: selectedClass.value.name,
    level: characterLevel.value,
    selectedSpells: Array.from(selectedSpells.value),
    savedAt: new Date().toISOString()
  }

  try {
    localStorage.setItem(key, JSON.stringify(data))
  } catch (error) {
    console.error('Failed to save to localStorage:', error)
  }
}

const loadFromStorage = () => {
  const key = getStorageKey()
  if (!key) return

  try {
    const stored = localStorage.getItem(key)
    if (!stored) return

    const data = JSON.parse(stored)
    selectedSpells.value = new Set(data.selectedSpells || [])
  } catch (error) {
    console.error('Failed to load from localStorage:', error)
  }
}

const clearAll = () => {
  selectedSpells.value = new Set()
  const key = getStorageKey()
  if (key) {
    try {
      localStorage.removeItem(key)
    } catch (error) {
      console.error('Failed to clear localStorage:', error)
    }
  }
}

// Add to return statement
return {
  // ... existing
  saveToStorage,
  loadFromStorage,
  clearAll
}

// Update interface
export interface UseSpellListGeneratorReturn {
  // ... existing
  saveToStorage: () => void
  loadFromStorage: () => void
  clearAll: () => void
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useSpellListGenerator.test.ts`

Expected: PASS (7 tests)

**Step 5: Commit**

```bash
git add app/composables/useSpellListGenerator.ts tests/composables/useSpellListGenerator.test.ts
git commit -m "feat: add LocalStorage persistence for spell selections

- Save spell list to localStorage by class/level
- Load spell list on mount
- Clear all selections
- Error handling for localStorage failures

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Create Spell List Generator Page (Basic Structure)

**Files:**
- Create: `app/pages/spells/list-generator.vue`
- Create: `tests/pages/spells/list-generator.test.ts`

**Step 1: Write the failing test for page mounting**

Create `tests/pages/spells/list-generator.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import ListGeneratorPage from '~/pages/spells/list-generator.vue'

describe('Spell List Generator Page', () => {
  it('mounts successfully', async () => {
    const wrapper = await mountSuspended(ListGeneratorPage)
    expect(wrapper.find('h1').text()).toContain('Spell List Generator')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/pages/spells/list-generator.test.ts`

Expected: FAIL

**Step 3: Write minimal page structure**

Create `app/pages/spells/list-generator.vue`:

```vue
<script setup lang="ts">
definePageMeta({
  title: 'Spell List Generator - D&D 5e Compendium',
  description: 'Create and manage your spell list for any D&D 5e spellcasting class'
})

useSeoMeta({
  title: 'Spell List Generator - D&D 5e Compendium',
  description: 'Create and manage your spell list for any D&D 5e spellcasting class'
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <h1 class="text-3xl font-bold mb-4">Spell List Generator</h1>
    <p class="text-gray-600 dark:text-gray-400">
      Choose spells for your character based on class and level.
    </p>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/pages/spells/list-generator.test.ts`

Expected: PASS

**Step 5: Verify in browser**

Run: `docker compose exec nuxt npm run dev`
Navigate to: `http://localhost:3000/spells/list-generator`
Expected: See heading and description

**Step 6: Commit**

```bash
git add app/pages/spells/list-generator.vue tests/pages/spells/list-generator.test.ts
git commit -m "feat: create spell list generator page skeleton

- Add /spells/list-generator route
- Basic page structure with heading
- SEO metadata
- Test for page mounting

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Add Character Setup Section

**Files:**
- Modify: `app/pages/spells/list-generator.vue`
- Modify: `tests/pages/spells/list-generator.test.ts`

**Step 1: Write the failing test**

Add to `tests/pages/spells/list-generator.test.ts`:

```typescript
it('displays class and level dropdowns', async () => {
  const wrapper = await mountSuspended(ListGeneratorPage)

  // Check for class dropdown
  const selects = wrapper.findAllComponents({ name: 'USelectMenu' })
  expect(selects.length).toBeGreaterThanOrEqual(2)
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/pages/spells/list-generator.test.ts`

Expected: FAIL

**Step 3: Implement character setup section**

Update `app/pages/spells/list-generator.vue`:

```vue
<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import type { CharacterClass } from '~/types'

definePageMeta({
  title: 'Spell List Generator - D&D 5e Compendium',
  description: 'Create and manage your spell list for any D&D 5e spellcasting class'
})

useSeoMeta({
  title: 'Spell List Generator - D&D 5e Compendium',
  description: 'Create and manage your spell list for any D&D 5e spellcasting class'
})

const { apiFetch } = useApi()

// Fetch spellcasting classes
const { data: classes, loading: classesLoading } = await useAsyncData<CharacterClass[]>(
  'spellcasting-classes',
  async () => {
    const response = await apiFetch<{ data: CharacterClass[] }>('/classes?per_page=100')
    // Filter to only spellcasting classes
    return response.data.filter(c => c.is_spellcaster === '1')
  }
)

// Use spell list generator composable
const {
  selectedClass,
  characterLevel,
  spellSlots,
  maxSpells,
  setClassData
} = useSpellListGenerator()

// Class dropdown options
const classOptions = computed(() => {
  if (!classes.value) return []
  return classes.value.map(c => ({
    label: c.name,
    value: c.slug,
    class: c
  }))
})

// Level dropdown options
const levelOptions = Array.from({ length: 20 }, (_, i) => ({
  label: `Level ${i + 1}`,
  value: i + 1
}))

// Selected class option (for USelectMenu v-model)
const selectedClassOption = ref<{ label: string; value: string; class: CharacterClass } | null>(null)

// Watch for class selection
watch(selectedClassOption, (option) => {
  if (option) {
    setClassData(option.class)
  }
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <!-- Header -->
    <div class="mb-8">
      <h1 class="text-3xl font-bold mb-2">Spell List Generator</h1>
      <p class="text-gray-600 dark:text-gray-400">
        Choose spells for your character based on class and level.
      </p>
    </div>

    <!-- Character Setup -->
    <div class="mb-8 p-6 bg-white dark:bg-gray-800 rounded-lg border border-gray-200 dark:border-gray-700">
      <h2 class="text-xl font-semibold mb-4">Character Setup</h2>

      <div class="flex flex-wrap gap-4 mb-6">
        <!-- Class Dropdown -->
        <div class="w-64">
          <label class="block text-sm font-medium mb-2">Class</label>
          <USelectMenu
            v-model="selectedClassOption"
            :items="classOptions"
            value-key="value"
            placeholder="Select class"
            :loading="classesLoading"
          />
        </div>

        <!-- Level Dropdown -->
        <div class="w-32">
          <label class="block text-sm font-medium mb-2">Level</label>
          <USelectMenu
            v-model="characterLevel"
            :items="levelOptions"
            value-key="value"
            placeholder="Level"
          />
        </div>
      </div>

      <!-- Spell Info Display (show when class selected) -->
      <div v-if="selectedClass" class="space-y-2">
        <div class="flex items-center gap-4 text-sm">
          <span class="font-medium">📊 Spell Slots:</span>
          <span>Cantrips: {{ spellSlots.cantrips }}</span>
          <span v-if="spellSlots['1st']">1st: {{ spellSlots['1st'] }}</span>
          <span v-if="spellSlots['2nd']">2nd: {{ spellSlots['2nd'] }}</span>
          <span v-if="spellSlots['3rd']">3rd: {{ spellSlots['3rd'] }}</span>
          <span v-if="spellSlots['4th']">4th: {{ spellSlots['4th'] }}</span>
          <span v-if="spellSlots['5th']">5th: {{ spellSlots['5th'] }}</span>
        </div>
        <div class="text-sm">
          <span class="font-medium">📝 Spells to Prepare:</span>
          {{ maxSpells }} ({{ characterLevel }} + 3 modifier)
        </div>
      </div>
    </div>

    <!-- TODO: Spell selection section -->
    <!-- TODO: Summary sidebar -->
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/pages/spells/list-generator.test.ts`

Expected: PASS

**Step 5: Verify in browser**

Navigate to: `http://localhost:3000/spells/list-generator`
Expected: See class and level dropdowns, spell slots display when class selected

**Step 6: Commit**

```bash
git add app/pages/spells/list-generator.vue tests/pages/spells/list-generator.test.ts
git commit -m "feat: add character setup section to spell list generator

- Class dropdown (filtered to spellcasters only)
- Level dropdown (1-20)
- Spell slots display
- Max prepared/known display
- Integrate useSpellListGenerator composable

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Add Spell Fetching and Display

**Files:**
- Modify: `app/pages/spells/list-generator.vue`

**Step 1: Add spell fetching logic**

Update `app/pages/spells/list-generator.vue` script section:

```vue
<script setup lang="ts">
// ... existing imports
import type { Spell } from '~/types'

// ... existing code

// Fetch spells for selected class
const { data: spells, loading: spellsLoading } = await useAsyncData(
  'class-spells',
  async () => {
    if (!selectedClass.value) return []
    const response = await apiFetch<{ data: Spell[] }>(
      `/spells?classes=${selectedClass.value.slug}&per_page=1000`
    )
    return response.data
  },
  {
    watch: [selectedClass],
    immediate: false
  }
)

// Group spells by level
const spellsByLevel = computed(() => {
  if (!spells.value) return new Map()

  const grouped = new Map<number, Spell[]>()

  for (const spell of spells.value) {
    const level = spell.level
    if (!grouped.has(level)) {
      grouped.set(level, [])
    }
    grouped.get(level)!.push(spell)
  }

  return grouped
})

// Get spell level label
const getSpellLevelLabel = (level: number) => {
  if (level === 0) return 'Cantrips'
  if (level === 1) return '1st Level Spells'
  if (level === 2) return '2nd Level Spells'
  if (level === 3) return '3rd Level Spells'
  return `${level}th Level Spells`
}

// Available spell levels (based on character level)
const availableSpellLevels = computed(() => {
  const levels = [0] // Always include cantrips
  const charLevel = characterLevel.value

  // Add leveled spells based on character level
  if (charLevel >= 1) levels.push(1)
  if (charLevel >= 3) levels.push(2)
  if (charLevel >= 5) levels.push(3)
  if (charLevel >= 7) levels.push(4)
  if (charLevel >= 9) levels.push(5)
  if (charLevel >= 11) levels.push(6)
  if (charLevel >= 13) levels.push(7)
  if (charLevel >= 15) levels.push(8)
  if (charLevel >= 17) levels.push(9)

  return levels
})
</script>
```

**Step 2: Add spell display template**

Update `app/pages/spells/list-generator.vue` template section after character setup:

```vue
<!-- Spell Selection Section -->
<div v-if="selectedClass" class="mb-8">
  <h2 class="text-xl font-semibold mb-4">Select Your Spells</h2>

  <!-- Loading state -->
  <div v-if="spellsLoading" class="space-y-4">
    <UiListSkeletonCards />
  </div>

  <!-- No spells available -->
  <div v-else-if="!spells || spells.length === 0" class="text-center py-8">
    <p class="text-gray-600 dark:text-gray-400">
      No spells available for {{ selectedClass.name }}.
    </p>
  </div>

  <!-- Spells grouped by level -->
  <div v-else class="space-y-4">
    <div
      v-for="level in availableSpellLevels"
      :key="level"
      class="border border-gray-200 dark:border-gray-700 rounded-lg p-4"
    >
      <h3 class="text-lg font-semibold mb-3">
        {{ getSpellLevelLabel(level) }}
        <span class="text-sm font-normal text-gray-600 dark:text-gray-400">
          ({{ spellsByLevel.get(level)?.length || 0 }} available)
        </span>
      </h3>

      <div v-if="spellsByLevel.get(level)" class="space-y-2">
        <div
          v-for="spell in spellsByLevel.get(level)"
          :key="spell.id"
          class="flex items-start gap-3 p-3 rounded border border-gray-100 dark:border-gray-800 hover:bg-gray-50 dark:hover:bg-gray-800"
        >
          <UCheckbox
            :model-value="selectedSpells.has(spell.id)"
            @update:model-value="toggleSpell(spell.id)"
          />
          <div class="flex-1">
            <div class="font-medium">{{ spell.name }}</div>
            <div class="text-sm text-gray-600 dark:text-gray-400">
              {{ spell.school?.name }} • {{ spell.range }}
              <UBadge v-if="spell.concentration === '1'" color="primary" variant="soft" size="xs" class="ml-2">
                Concentration
              </UBadge>
              <UBadge v-if="spell.ritual === '1'" color="info" variant="soft" size="xs" class="ml-1">
                Ritual
              </UBadge>
            </div>
          </div>
        </div>
      </div>

      <div v-else class="text-sm text-gray-500 dark:text-gray-500">
        No spells available at this level.
      </div>
    </div>
  </div>
</div>

<!-- Prompt to select class -->
<div v-else class="text-center py-12">
  <p class="text-lg text-gray-600 dark:text-gray-400">
    Select a class and level to begin choosing spells.
  </p>
</div>
```

**Step 3: Verify in browser**

Navigate to: `http://localhost:3000/spells/list-generator`
Select Wizard, Level 5
Expected: See spells grouped by level (Cantrips, 1st, 2nd, 3rd) with checkboxes

**Step 4: Commit**

```bash
git add app/pages/spells/list-generator.vue
git commit -m "feat: add spell fetching and display by level

- Fetch spells filtered by selected class
- Group spells by level (0-9)
- Show only levels available at character level
- Display spells with checkboxes
- Show concentration and ritual badges
- Loading skeleton while fetching

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 8: Add Summary Sidebar and Auto-Save

**Files:**
- Modify: `app/pages/spells/list-generator.vue`

**Step 1: Add auto-save and summary section**

Update `app/pages/spells/list-generator.vue`:

```vue
<script setup lang="ts">
// ... existing imports
import { watchDebounced } from '@vueuse/core'

// ... existing code (after useSpellListGenerator)

// Destructure additional methods
const {
  // ... existing
  saveToStorage,
  loadFromStorage,
  clearAll,
  selectionCount
} = useSpellListGenerator()

// Load saved selections on mount
onMounted(() => {
  if (selectedClass.value) {
    loadFromStorage()
  }
})

// Watch for class/level changes and load
watch([selectedClass, characterLevel], () => {
  if (selectedClass.value) {
    loadFromStorage()
  }
})

// Auto-save on selection changes (debounced)
watchDebounced(
  selectedSpells,
  () => {
    if (selectedClass.value) {
      saveToStorage()
    }
  },
  { debounce: 500, deep: true }
)

// Get selected spell objects
const selectedSpellsList = computed(() => {
  if (!spells.value) return []
  return spells.value.filter(s => selectedSpells.value.has(s.id))
})

// Group selected spells by level
const selectedByLevel = computed(() => {
  const grouped = new Map<number, Spell[]>()

  for (const spell of selectedSpellsList.value) {
    const level = spell.level
    if (!grouped.has(level)) {
      grouped.set(level, [])
    }
    grouped.get(level)!.push(spell)
  }

  // Sort levels
  return new Map([...grouped.entries()].sort((a, b) => a[0] - b[0]))
})

// Clear all handler
const handleClearAll = () => {
  if (confirm('Clear all selected spells?')) {
    clearAll()
  }
}
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <!-- Existing header and character setup -->
    <!-- ... -->

    <!-- Main content area with sidebar layout -->
    <div class="flex flex-col lg:flex-row gap-6">
      <!-- Left: Spell selection (flex-1) -->
      <div class="flex-1">
        <!-- Existing spell selection section -->
        <!-- ... -->
      </div>

      <!-- Right: Summary sidebar (sticky) -->
      <div v-if="selectedClass" class="lg:w-80">
        <div class="sticky top-4 p-6 bg-white dark:bg-gray-800 rounded-lg border border-gray-200 dark:border-gray-700">
          <h3 class="text-lg font-semibold mb-4">📋 Your Spell List</h3>

          <div class="mb-4 text-sm">
            <div class="font-medium">{{ selectedClass.name }} Level {{ characterLevel }}</div>
            <div class="text-gray-600 dark:text-gray-400">
              Selected: {{ selectionCount }} / {{ maxSpells }} spells
            </div>
          </div>

          <!-- Selected spells by level -->
          <div v-if="selectionCount > 0" class="space-y-3 mb-6 max-h-96 overflow-y-auto">
            <div
              v-for="[level, spellsAtLevel] in selectedByLevel"
              :key="level"
            >
              <div class="text-sm font-semibold mb-1">
                {{ getSpellLevelLabel(level) }} ({{ spellsAtLevel.length }})
              </div>
              <ul class="space-y-1 text-sm pl-2">
                <li v-for="spell in spellsAtLevel" :key="spell.id" class="text-gray-700 dark:text-gray-300">
                  • {{ spell.name }}
                </li>
              </ul>
            </div>
          </div>

          <div v-else class="text-sm text-gray-500 dark:text-gray-500 mb-6">
            No spells selected yet.
          </div>

          <!-- Actions -->
          <div class="flex flex-col gap-2">
            <UButton
              color="neutral"
              variant="soft"
              block
              @click="handleClearAll"
              :disabled="selectionCount === 0"
            >
              🗑️ Clear All
            </UButton>
            <div class="text-xs text-center text-gray-500 dark:text-gray-500 mt-2">
              Auto-saved to browser
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Back link -->
    <div class="mt-8">
      <UiBackLink />
    </div>
  </div>
</template>
```

**Step 2: Verify in browser**

Navigate to: `http://localhost:3000/spells/list-generator`
Actions:
1. Select Wizard, Level 5
2. Check some spells
3. See them appear in sidebar
4. Reload page
5. Verify selections persist

Expected: Auto-save works, sidebar updates, reload preserves selections

**Step 3: Commit**

```bash
git add app/pages/spells/list-generator.vue
git commit -m "feat: add summary sidebar and auto-save functionality

- Sticky sidebar showing selected spells
- Group selected spells by level
- Auto-save to LocalStorage (debounced 500ms)
- Auto-load on mount and class/level change
- Clear all button with confirmation
- Selection count display

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 9: Add Navigation Link and Polish

**Files:**
- Modify: `app/pages/spells/index.vue` (add link to generator)
- Modify: `app/pages/spells/list-generator.vue` (final polish)

**Step 1: Add link from spells page**

Add to `app/pages/spells/index.vue` after the header:

```vue
<!-- Add after UiListPageHeader -->
<div class="mb-6">
  <UButton
    to="/spells/list-generator"
    color="primary"
    variant="solid"
    icon="i-heroicons-sparkles"
  >
    🪄 Create Spell List
  </UButton>
</div>
```

**Step 2: Add final polish to generator page**

Update `app/pages/spells/list-generator.vue`:

```vue
<!-- Update header section -->
<div class="mb-8">
  <div class="flex items-center justify-between">
    <div>
      <h1 class="text-3xl font-bold mb-2">🪄 Spell List Generator</h1>
      <p class="text-gray-600 dark:text-gray-400">
        Choose spells for your character based on class and level.
      </p>
    </div>
  </div>
</div>
```

**Step 3: Verify navigation**

1. Navigate to: `http://localhost:3000/spells`
2. Click "Create Spell List" button
3. Verify navigation to `/spells/list-generator`
4. Test full flow

**Step 4: Commit**

```bash
git add app/pages/spells/index.vue app/pages/spells/list-generator.vue
git commit -m "feat: add navigation link and final polish

- Add 'Create Spell List' button to spells index page
- Add emoji to generator page title
- Improve header layout

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 10: Update CHANGELOG

**Files:**
- Modify: `CHANGELOG.md`

**Step 1: Add changelog entry**

Add to top of `CHANGELOG.md` under `## [Unreleased]`:

```markdown
### Added - Spell List Generator (2025-11-24)

**MVP Feature - Complete:**
- **Spell List Generator page** - New tool at `/spells/list-generator` for creating character spell lists based on class and level (2025-11-24)
- **All spellcasting classes supported** - Wizard, Cleric, Druid, Bard, Sorcerer, Warlock, Paladin, Ranger, Artificer, Eldritch Knight, Arcane Trickster (12+ classes) (2025-11-24)
- **Prepared vs Known caster logic** - Automatically calculates spell limits: prepared casters use level + modifier, known casters use fixed tables (2025-11-24)
- **Spell slots calculation** - Displays available spell slots per level from class progression data (2025-11-24)
- **Spell selection with checkboxes** - Browse and select spells grouped by level (Cantrips through 9th level) (2025-11-24)
- **LocalStorage persistence** - Auto-saves selections per class/level combination, survives page reloads (2025-11-24)
- **Summary sidebar** - Shows selected spells grouped by level with count tracking (2025-11-24)
- **useSpellListGenerator composable** - Reusable state management for spell list generation with full test coverage (2025-11-24)

**Technical Details:**
- New composable: `useSpellListGenerator` with spell slots calculation, max prepared/known logic, LocalStorage persistence
- Hardcoded known spells tables for 6 classes (Bard, Sorcerer, Warlock, Ranger, Eldritch Knight, Arcane Trickster)
- Debounced auto-save (500ms) to prevent excessive writes
- Mobile-responsive layout with sticky sidebar
- TDD approach: 7 composable tests, page mounting tests

**Impact:**
- ✅ Unique value proposition - Most D&D sites don't have this tool
- ✅ Leverages existing data and components
- ✅ Opens door to more "builder" features (character builder, encounter builder)
```

**Step 2: Commit**

```bash
git add CHANGELOG.md
git commit -m "docs: update CHANGELOG for spell list generator MVP

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 11: Run Full Test Suite

**Step 1: Run all tests**

```bash
docker compose exec nuxt npm run test
```

Expected: All tests pass (including new 7+ tests for composable and page)

**Step 2: Run typecheck**

```bash
docker compose exec nuxt npm run typecheck
```

Expected: No TypeScript errors

**Step 3: If any issues, fix them and commit**

---

## Post-Implementation Verification

### Browser Testing Checklist

1. ✅ Navigate to `/spells/list-generator`
2. ✅ Select "Wizard" class
3. ✅ Select Level 5
4. ✅ Verify spell slots display: "Cantrips: 4, 1st: 4, 2nd: 3, 3rd: 2"
5. ✅ Verify max spells: "8 (5 + 3 modifier)"
6. ✅ Check 8 spells across different levels
7. ✅ Verify sidebar updates
8. ✅ Reload page - selections persist
9. ✅ Change level - selections reload correctly
10. ✅ Clear all - confirms and clears
11. ✅ Test with Bard (known caster) - shows "8 spells known"
12. ✅ Test mobile view - sidebar stacks below

### Success Criteria

- ✅ All 12+ spellcasting classes work
- ✅ Spell slots calculated correctly
- ✅ Max prepared/known calculated correctly
- ✅ Selections save and load from LocalStorage
- ✅ Summary sidebar updates in real-time
- ✅ Mobile responsive
- ✅ All tests pass
- ✅ No TypeScript errors

---

## Execution Complete!

Total estimated time: 3-4 hours
Total commits: 11
Test coverage: 7+ new tests
Files created: 4
Files modified: 3
