# Character Sheet Redesign - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Redesign `/characters/[id]` page to display comprehensive D&D 5e character sheet with all available data.

**Architecture:** Single `useCharacterSheet` composable fetches 7 API endpoints in parallel, computes skill modifiers client-side. 10 flat components in `app/components/character/sheet/`. Tabbed bottom sections using NuxtUI `UTabs`.

**Tech Stack:** Nuxt 4, NuxtUI 4, TypeScript, Vitest, Pinia (not used - composable only)

**Design Document:** See `docs/plans/2025-12-05-character-sheet-redesign.md` for full architecture details.

---

## Task 1: Add Type Definitions

**Files:**
- Modify: `app/types/character.ts`

**Step 1: Add new type definitions to character.ts**

Add these types after the existing `CharacterStats` interface (around line 151):

```typescript
/**
 * Skill with computed modifier for character sheet display
 */
export interface CharacterSkill {
  id: number
  name: string
  slug: string
  ability_code: AbilityScoreCode
  modifier: number
  proficient: boolean
  expertise: boolean
}

/**
 * Saving throw with computed values for character sheet display
 */
export interface CharacterSavingThrow {
  ability: AbilityScoreCode
  modifier: number
  proficient: boolean
}

/**
 * Character feature from API
 */
export interface CharacterFeature {
  id: number
  source: 'class' | 'race' | 'background'
  level_acquired: number
  feature_type: string
  uses_remaining: number | null
  max_uses: number | null
  has_limited_uses: boolean
  feature: {
    id: number
    name: string
    description: string
    category: string | null
  }
}

/**
 * Character equipment item from API
 */
export interface CharacterEquipment {
  id: number
  item: {
    id: number
    name: string
    slug: string
  } | null
  quantity: number
  equipped: boolean
  description: string | null
}

/**
 * Character language from API
 */
export interface CharacterLanguage {
  id: number
  source: string
  language: {
    id: number
    name: string
    slug: string
    script: string
  }
}

/**
 * Character spell from API
 */
export interface CharacterSpell {
  id: number
  spell: {
    id: number
    name: string
    slug: string
    level: number
    school: string
  }
  prepared: boolean
  always_prepared: boolean
  source: string
}

/**
 * Character proficiency from API (skill or tool)
 */
export interface CharacterProficiency {
  id: number
  source: string
  expertise: boolean
  skill?: {
    id: number
    name: string
    slug: string
    ability_code: AbilityScoreCode
  }
  proficiency_type?: {
    id: number
    name: string
    slug: string
    category: string
  }
}

/**
 * Skill reference data from /skills endpoint
 */
export interface SkillReference {
  id: number
  name: string
  slug: string
  ability_code: AbilityScoreCode
}
```

**Step 2: Run typecheck to verify**

Run: `docker compose exec nuxt npm run typecheck`
Expected: PASS (no type errors)

**Step 3: Commit**

```bash
git add app/types/character.ts
git commit -m "feat(types): add character sheet type definitions (#172)"
```

---

## Task 2: Create useCharacterSheet Composable - Test First

**Files:**
- Create: `tests/composables/useCharacterSheet.test.ts`
- Create: `app/composables/useCharacterSheet.ts`

**Step 1: Write the failing test**

```typescript
// tests/composables/useCharacterSheet.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { ref, nextTick } from 'vue'

// Mock useAsyncData and useApi before importing composable
const mockCharacter = {
  id: 1,
  name: 'Test Hero',
  level: 3,
  proficiency_bonus: 2,
  is_complete: true,
  has_inspiration: false,
  race: { id: 1, name: 'Human', slug: 'human' },
  class: { id: 1, name: 'Fighter', slug: 'fighter' },
  classes: [{ class: { id: 1, name: 'Fighter', slug: 'fighter' }, level: 3 }],
  background: { id: 1, name: 'Soldier', slug: 'soldier' },
  ability_scores: { STR: 16, DEX: 14, CON: 15, INT: 10, WIS: 12, CHA: 8 },
  speed: 30,
  size: 'Medium'
}

const mockStats = {
  character_id: 1,
  level: 3,
  proficiency_bonus: 2,
  ability_scores: {
    STR: { score: 16, modifier: 3 },
    DEX: { score: 14, modifier: 2 },
    CON: { score: 15, modifier: 2 },
    INT: { score: 10, modifier: 0 },
    WIS: { score: 12, modifier: 1 },
    CHA: { score: 8, modifier: -1 }
  },
  saving_throws: {
    STR: 5,
    DEX: 2,
    CON: 4,
    INT: 0,
    WIS: 1,
    CHA: -1
  },
  armor_class: 16,
  hit_points: { max: 28, current: 28, temporary: 0 },
  initiative_bonus: 2,
  passive_perception: 11,
  spellcasting: null
}

const mockProficiencies = [
  { id: 1, source: 'class', expertise: false, skill: { id: 3, name: 'Athletics', slug: 'athletics', ability_code: 'STR' } },
  { id: 2, source: 'background', expertise: false, skill: { id: 8, name: 'Intimidation', slug: 'intimidation', ability_code: 'CHA' } }
]

const mockSkillsReference = [
  { id: 1, name: 'Acrobatics', slug: 'acrobatics', ability_code: 'DEX' },
  { id: 2, name: 'Animal Handling', slug: 'animal-handling', ability_code: 'WIS' },
  { id: 3, name: 'Athletics', slug: 'athletics', ability_code: 'STR' },
  { id: 8, name: 'Intimidation', slug: 'intimidation', ability_code: 'CHA' }
]

// Create mock functions
const mockApiFetch = vi.fn()

vi.mock('#app', () => ({
  useAsyncData: vi.fn((key: string, fetcher: () => Promise<any>) => {
    // Simulate async data based on key
    const pending = ref(false)
    const error = ref(null)
    let data = ref(null)

    if (key.includes('character-') && !key.includes('stats') && !key.includes('proficiencies')) {
      data = ref({ data: mockCharacter })
    } else if (key.includes('stats')) {
      data = ref({ data: mockStats })
    } else if (key.includes('proficiencies')) {
      data = ref({ data: mockProficiencies })
    } else if (key.includes('features')) {
      data = ref({ data: [] })
    } else if (key.includes('equipment')) {
      data = ref({ data: [] })
    } else if (key.includes('spells')) {
      data = ref({ data: [] })
    } else if (key.includes('languages')) {
      data = ref({ data: [] })
    }

    return { data, pending, error, refresh: vi.fn() }
  })
}))

vi.mock('~/composables/useApi', () => ({
  useApi: () => ({ apiFetch: mockApiFetch })
}))

vi.mock('~/composables/useReferenceData', () => ({
  useReferenceData: () => ({
    data: ref(mockSkillsReference),
    loading: ref(false),
    error: ref(null)
  })
}))

describe('useCharacterSheet', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('should be defined', async () => {
    const { useCharacterSheet } = await import('~/composables/useCharacterSheet')
    expect(useCharacterSheet).toBeDefined()
  })

  it('returns character data', async () => {
    const { useCharacterSheet } = await import('~/composables/useCharacterSheet')
    const characterId = ref(1)
    const result = useCharacterSheet(characterId)

    expect(result.character.value).toBeDefined()
    expect(result.character.value?.name).toBe('Test Hero')
  })

  it('calculates skill modifiers correctly', async () => {
    const { useCharacterSheet } = await import('~/composables/useCharacterSheet')
    const characterId = ref(1)
    const result = useCharacterSheet(characterId)

    await nextTick()

    // Athletics should be STR mod (3) + proficiency (2) = 5
    const athletics = result.skills.value.find(s => s.slug === 'athletics')
    expect(athletics?.modifier).toBe(5)
    expect(athletics?.proficient).toBe(true)

    // Acrobatics should be DEX mod (2) only, no proficiency
    const acrobatics = result.skills.value.find(s => s.slug === 'acrobatics')
    expect(acrobatics?.modifier).toBe(2)
    expect(acrobatics?.proficient).toBe(false)
  })

  it('provides loading state', async () => {
    const { useCharacterSheet } = await import('~/composables/useCharacterSheet')
    const characterId = ref(1)
    const result = useCharacterSheet(characterId)

    expect(result.loading).toBeDefined()
    expect(typeof result.loading.value).toBe('boolean')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="useCharacterSheet"`
Expected: FAIL with "Cannot find module" or "useCharacterSheet is not defined"

**Step 3: Create the composable implementation**

```typescript
// app/composables/useCharacterSheet.ts
import { computed, type Ref, type ComputedRef } from 'vue'
import type {
  Character,
  CharacterStats,
  CharacterProficiency,
  CharacterFeature,
  CharacterEquipment,
  CharacterSpell,
  CharacterLanguage,
  CharacterSkill,
  CharacterSavingThrow,
  SkillReference,
  AbilityScoreCode
} from '~/types/character'

export interface UseCharacterSheetReturn {
  // Raw API data
  character: ComputedRef<Character | null>
  stats: ComputedRef<CharacterStats | null>
  proficiencies: ComputedRef<CharacterProficiency[]>
  features: ComputedRef<CharacterFeature[]>
  equipment: ComputedRef<CharacterEquipment[]>
  spells: ComputedRef<CharacterSpell[]>
  languages: ComputedRef<CharacterLanguage[]>

  // Computed/derived
  skills: ComputedRef<CharacterSkill[]>
  savingThrows: ComputedRef<CharacterSavingThrow[]>

  // State
  loading: ComputedRef<boolean>
  error: ComputedRef<Error | null>
  refresh: () => Promise<void>
}

/**
 * Composable for fetching and computing all character sheet data
 *
 * Fetches 7 API endpoints in parallel and computes derived values
 * like skill modifiers and saving throw proficiencies.
 */
export function useCharacterSheet(characterId: Ref<string | number>): UseCharacterSheetReturn {
  const { apiFetch } = useApi()

  // Fetch character base data
  const { data: characterData, pending: characterPending, error: characterError, refresh: refreshCharacter } =
    useAsyncData(
      `character-${characterId.value}`,
      () => apiFetch<{ data: Character }>(`/characters/${characterId.value}`)
    )

  // Fetch character stats
  const { data: statsData, pending: statsPending, refresh: refreshStats } =
    useAsyncData(
      `character-${characterId.value}-stats`,
      () => apiFetch<{ data: CharacterStats }>(`/characters/${characterId.value}/stats`)
    )

  // Fetch proficiencies
  const { data: proficienciesData, pending: proficienciesPending, refresh: refreshProficiencies } =
    useAsyncData(
      `character-${characterId.value}-proficiencies`,
      () => apiFetch<{ data: CharacterProficiency[] }>(`/characters/${characterId.value}/proficiencies`)
    )

  // Fetch features
  const { data: featuresData, pending: featuresPending, refresh: refreshFeatures } =
    useAsyncData(
      `character-${characterId.value}-features`,
      () => apiFetch<{ data: CharacterFeature[] }>(`/characters/${characterId.value}/features`)
    )

  // Fetch equipment
  const { data: equipmentData, pending: equipmentPending, refresh: refreshEquipment } =
    useAsyncData(
      `character-${characterId.value}-equipment`,
      () => apiFetch<{ data: CharacterEquipment[] }>(`/characters/${characterId.value}/equipment`)
    )

  // Fetch spells
  const { data: spellsData, pending: spellsPending, refresh: refreshSpells } =
    useAsyncData(
      `character-${characterId.value}-spells`,
      () => apiFetch<{ data: CharacterSpell[] }>(`/characters/${characterId.value}/spells`)
    )

  // Fetch languages
  const { data: languagesData, pending: languagesPending, refresh: refreshLanguages } =
    useAsyncData(
      `character-${characterId.value}-languages`,
      () => apiFetch<{ data: CharacterLanguage[] }>(`/characters/${characterId.value}/languages`)
    )

  // Fetch skills reference data
  const { data: skillsReference } = useReferenceData<SkillReference>('/skills')

  // Computed: Extract data from responses
  const character = computed(() => characterData.value?.data ?? null)
  const stats = computed(() => statsData.value?.data ?? null)
  const proficiencies = computed(() => proficienciesData.value?.data ?? [])
  const features = computed(() => featuresData.value?.data ?? [])
  const equipment = computed(() => equipmentData.value?.data ?? [])
  const spells = computed(() => spellsData.value?.data ?? [])
  const languages = computed(() => languagesData.value?.data ?? [])

  // Computed: Aggregate loading state
  const loading = computed(() =>
    characterPending.value ||
    statsPending.value ||
    proficienciesPending.value ||
    featuresPending.value ||
    equipmentPending.value ||
    spellsPending.value ||
    languagesPending.value
  )

  // Computed: First error encountered
  const error = computed(() => characterError.value as Error | null)

  // Computed: Calculate all 18 skill modifiers
  const skills = computed<CharacterSkill[]>(() => {
    if (!skillsReference.value || !stats.value || !character.value) return []

    const proficiencyBonus = character.value.proficiency_bonus

    return skillsReference.value.map((skill) => {
      // Find if character is proficient in this skill
      const profRecord = proficiencies.value.find(
        (p) => p.skill?.slug === skill.slug
      )
      const isProficient = !!profRecord
      const hasExpertise = profRecord?.expertise ?? false

      // Get ability modifier from stats
      const abilityMod = stats.value?.ability_scores[skill.ability_code]?.modifier ?? 0

      // Calculate modifier: ability mod + proficiency (+ expertise)
      let modifier = abilityMod
      if (isProficient) modifier += proficiencyBonus
      if (hasExpertise) modifier += proficiencyBonus

      return {
        id: skill.id,
        name: skill.name,
        slug: skill.slug,
        ability_code: skill.ability_code,
        modifier,
        proficient: isProficient,
        expertise: hasExpertise
      }
    })
  })

  // Computed: Saving throws with proficiency info
  const savingThrows = computed<CharacterSavingThrow[]>(() => {
    if (!stats.value) return []

    const abilities: AbilityScoreCode[] = ['STR', 'DEX', 'CON', 'INT', 'WIS', 'CHA']

    return abilities.map((ability) => {
      const modifier = stats.value?.saving_throws[ability] ?? 0
      // Determine proficiency by comparing save modifier to ability modifier
      // If save > ability mod, they're proficient
      const abilityMod = stats.value?.ability_scores[ability]?.modifier ?? 0
      const proficient = modifier > abilityMod

      return {
        ability,
        modifier,
        proficient
      }
    })
  })

  // Refresh all data
  const refresh = async () => {
    await Promise.all([
      refreshCharacter(),
      refreshStats(),
      refreshProficiencies(),
      refreshFeatures(),
      refreshEquipment(),
      refreshSpells(),
      refreshLanguages()
    ])
  }

  return {
    character,
    stats,
    proficiencies,
    features,
    equipment,
    spells,
    languages,
    skills,
    savingThrows,
    loading,
    error,
    refresh
  }
}
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="useCharacterSheet"`
Expected: PASS

**Step 5: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: PASS

**Step 6: Commit**

```bash
git add app/composables/useCharacterSheet.ts tests/composables/useCharacterSheet.test.ts
git commit -m "feat(composables): add useCharacterSheet for character sheet data (#172)"
```

---

## Task 3: Create CharacterSheetHeader Component

**Files:**
- Create: `tests/components/character/sheet/Header.test.ts`
- Create: `app/components/character/sheet/Header.vue`

**Step 1: Write the failing test**

```typescript
// tests/components/character/sheet/Header.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import Header from '~/components/character/sheet/Header.vue'

const mockCharacter = {
  id: 1,
  name: 'Thorin Ironforge',
  level: 5,
  is_complete: true,
  has_inspiration: true,
  race: { id: 1, name: 'Dwarf', slug: 'dwarf' },
  class: { id: 1, name: 'Fighter', slug: 'fighter' },
  classes: [
    { class: { id: 1, name: 'Fighter', slug: 'fighter' }, level: 3, is_primary: true },
    { class: { id: 2, name: 'Cleric', slug: 'cleric' }, level: 2, is_primary: false }
  ],
  background: { id: 1, name: 'Soldier', slug: 'soldier' }
}

describe('CharacterSheetHeader', () => {
  it('displays character name', async () => {
    const wrapper = await mountSuspended(Header, {
      props: { character: mockCharacter }
    })
    expect(wrapper.text()).toContain('Thorin Ironforge')
  })

  it('displays race name', async () => {
    const wrapper = await mountSuspended(Header, {
      props: { character: mockCharacter }
    })
    expect(wrapper.text()).toContain('Dwarf')
  })

  it('displays all classes with levels', async () => {
    const wrapper = await mountSuspended(Header, {
      props: { character: mockCharacter }
    })
    expect(wrapper.text()).toContain('Fighter 3')
    expect(wrapper.text()).toContain('Cleric 2')
  })

  it('shows Complete badge when is_complete is true', async () => {
    const wrapper = await mountSuspended(Header, {
      props: { character: mockCharacter }
    })
    expect(wrapper.text()).toContain('Complete')
  })

  it('shows Draft badge when is_complete is false', async () => {
    const wrapper = await mountSuspended(Header, {
      props: { character: { ...mockCharacter, is_complete: false } }
    })
    expect(wrapper.text()).toContain('Draft')
  })

  it('shows Inspiration badge when has_inspiration is true', async () => {
    const wrapper = await mountSuspended(Header, {
      props: { character: mockCharacter }
    })
    expect(wrapper.find('[data-test="inspiration-badge"]').exists()).toBe(true)
  })

  it('hides Inspiration badge when has_inspiration is false', async () => {
    const wrapper = await mountSuspended(Header, {
      props: { character: { ...mockCharacter, has_inspiration: false } }
    })
    expect(wrapper.find('[data-test="inspiration-badge"]').exists()).toBe(false)
  })

  it('shows Edit button for incomplete characters', async () => {
    const wrapper = await mountSuspended(Header, {
      props: { character: { ...mockCharacter, is_complete: false } }
    })
    expect(wrapper.find('[data-test="edit-button"]').exists()).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheetHeader"`
Expected: FAIL with component not found

**Step 3: Create the component**

```vue
<!-- app/components/character/sheet/Header.vue -->
<script setup lang="ts">
import type { Character } from '~/types/character'

const props = defineProps<{
  character: Character
}>()

/**
 * Format classes display string
 * Shows each class with its level, separated by " / "
 */
const classesDisplay = computed(() => {
  if (!props.character.classes?.length) {
    return props.character.class?.name ?? 'No class'
  }
  return props.character.classes
    .map((c) => `${c.class.name} ${c.level}`)
    .join(' / ')
})
</script>

<template>
  <div class="flex flex-col sm:flex-row sm:items-center sm:justify-between gap-4">
    <!-- Left: Name and info -->
    <div>
      <h1 class="text-3xl font-bold text-gray-900 dark:text-white">
        {{ character.name }}
      </h1>
      <p class="mt-1 text-lg text-gray-600 dark:text-gray-400">
        <span v-if="character.race">{{ character.race.name }}</span>
        <span v-if="character.race && character.classes?.length"> &bull; </span>
        <span>{{ classesDisplay }}</span>
        <span v-if="character.background"> &bull; {{ character.background.name }}</span>
      </p>
    </div>

    <!-- Right: Badges and actions -->
    <div class="flex items-center gap-2 flex-wrap">
      <!-- Inspiration Badge -->
      <UBadge
        v-if="character.has_inspiration"
        data-test="inspiration-badge"
        color="warning"
        variant="solid"
        size="lg"
      >
        <UIcon
          name="i-heroicons-star-solid"
          class="w-4 h-4 mr-1"
        />
        Inspired
      </UBadge>

      <!-- Status Badge -->
      <UBadge
        :color="character.is_complete ? 'success' : 'warning'"
        variant="subtle"
        size="lg"
      >
        {{ character.is_complete ? 'Complete' : 'Draft' }}
      </UBadge>

      <!-- Edit Button (for incomplete characters) -->
      <UButton
        v-if="!character.is_complete"
        data-test="edit-button"
        :to="`/characters/${character.id}/edit`"
        variant="outline"
        size="sm"
        icon="i-heroicons-pencil"
      >
        Edit
      </UButton>
    </div>
  </div>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheetHeader"`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/sheet/Header.vue tests/components/character/sheet/Header.test.ts
git commit -m "feat(components): add CharacterSheetHeader component (#172)"
```

---

## Task 4: Create CharacterSheetAbilityScoreBlock Component

**Files:**
- Create: `tests/components/character/sheet/AbilityScoreBlock.test.ts`
- Create: `app/components/character/sheet/AbilityScoreBlock.vue`

**Step 1: Write the failing test**

```typescript
// tests/components/character/sheet/AbilityScoreBlock.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import AbilityScoreBlock from '~/components/character/sheet/AbilityScoreBlock.vue'

const mockStats = {
  ability_scores: {
    STR: { score: 16, modifier: 3 },
    DEX: { score: 14, modifier: 2 },
    CON: { score: 15, modifier: 2 },
    INT: { score: 10, modifier: 0 },
    WIS: { score: 12, modifier: 1 },
    CHA: { score: 8, modifier: -1 }
  }
}

describe('CharacterSheetAbilityScoreBlock', () => {
  it('displays all 6 ability scores', async () => {
    const wrapper = await mountSuspended(AbilityScoreBlock, {
      props: { stats: mockStats }
    })
    expect(wrapper.text()).toContain('STR')
    expect(wrapper.text()).toContain('DEX')
    expect(wrapper.text()).toContain('CON')
    expect(wrapper.text()).toContain('INT')
    expect(wrapper.text()).toContain('WIS')
    expect(wrapper.text()).toContain('CHA')
  })

  it('displays score values', async () => {
    const wrapper = await mountSuspended(AbilityScoreBlock, {
      props: { stats: mockStats }
    })
    expect(wrapper.text()).toContain('16')
    expect(wrapper.text()).toContain('14')
    expect(wrapper.text()).toContain('8')
  })

  it('displays positive modifiers with + sign', async () => {
    const wrapper = await mountSuspended(AbilityScoreBlock, {
      props: { stats: mockStats }
    })
    expect(wrapper.text()).toContain('+3')
    expect(wrapper.text()).toContain('+2')
  })

  it('displays negative modifiers with - sign', async () => {
    const wrapper = await mountSuspended(AbilityScoreBlock, {
      props: { stats: mockStats }
    })
    expect(wrapper.text()).toContain('-1')
  })

  it('displays zero modifier as +0', async () => {
    const wrapper = await mountSuspended(AbilityScoreBlock, {
      props: { stats: mockStats }
    })
    expect(wrapper.text()).toContain('+0')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheetAbilityScoreBlock"`
Expected: FAIL

**Step 3: Create the component**

```vue
<!-- app/components/character/sheet/AbilityScoreBlock.vue -->
<script setup lang="ts">
import type { CharacterStats, AbilityScoreCode } from '~/types/character'

defineProps<{
  stats: CharacterStats
}>()

const abilities: { code: AbilityScoreCode; label: string }[] = [
  { code: 'STR', label: 'STR' },
  { code: 'DEX', label: 'DEX' },
  { code: 'CON', label: 'CON' },
  { code: 'INT', label: 'INT' },
  { code: 'WIS', label: 'WIS' },
  { code: 'CHA', label: 'CHA' }
]

/**
 * Format modifier with sign
 */
function formatModifier(mod: number | null): string {
  if (mod === null) return '—'
  return mod >= 0 ? `+${mod}` : `${mod}`
}
</script>

<template>
  <div class="space-y-2">
    <div
      v-for="ability in abilities"
      :key="ability.code"
      class="flex items-center justify-between p-3 bg-gray-50 dark:bg-gray-800 rounded-lg"
    >
      <div class="text-sm font-bold text-gray-500 dark:text-gray-400 w-12">
        {{ ability.label }}
      </div>
      <div class="text-2xl font-bold text-gray-900 dark:text-white">
        {{ stats.ability_scores[ability.code]?.score ?? '—' }}
      </div>
      <div class="text-lg font-semibold text-primary-600 dark:text-primary-400 w-12 text-right">
        {{ formatModifier(stats.ability_scores[ability.code]?.modifier ?? null) }}
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheetAbilityScoreBlock"`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/sheet/AbilityScoreBlock.vue tests/components/character/sheet/AbilityScoreBlock.test.ts
git commit -m "feat(components): add CharacterSheetAbilityScoreBlock component (#172)"
```

---

## Task 5: Create CharacterSheetCombatStatsGrid Component

**Files:**
- Create: `tests/components/character/sheet/CombatStatsGrid.test.ts`
- Create: `app/components/character/sheet/CombatStatsGrid.vue`

**Step 1: Write the failing test**

```typescript
// tests/components/character/sheet/CombatStatsGrid.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import CombatStatsGrid from '~/components/character/sheet/CombatStatsGrid.vue'

const mockCharacter = {
  speed: 30,
  proficiency_bonus: 2
}

const mockStats = {
  armor_class: 16,
  hit_points: { max: 28, current: 22, temporary: 5 },
  initiative_bonus: 2,
  passive_perception: 14
}

describe('CharacterSheetCombatStatsGrid', () => {
  it('displays hit points as current/max', async () => {
    const wrapper = await mountSuspended(CombatStatsGrid, {
      props: { character: mockCharacter, stats: mockStats }
    })
    expect(wrapper.text()).toContain('22')
    expect(wrapper.text()).toContain('28')
  })

  it('displays armor class', async () => {
    const wrapper = await mountSuspended(CombatStatsGrid, {
      props: { character: mockCharacter, stats: mockStats }
    })
    expect(wrapper.text()).toContain('16')
    expect(wrapper.text()).toContain('AC')
  })

  it('displays initiative bonus', async () => {
    const wrapper = await mountSuspended(CombatStatsGrid, {
      props: { character: mockCharacter, stats: mockStats }
    })
    expect(wrapper.text()).toContain('+2')
    expect(wrapper.text()).toContain('Initiative')
  })

  it('displays speed', async () => {
    const wrapper = await mountSuspended(CombatStatsGrid, {
      props: { character: mockCharacter, stats: mockStats }
    })
    expect(wrapper.text()).toContain('30')
    expect(wrapper.text()).toContain('ft')
  })

  it('displays proficiency bonus', async () => {
    const wrapper = await mountSuspended(CombatStatsGrid, {
      props: { character: mockCharacter, stats: mockStats }
    })
    expect(wrapper.text()).toContain('+2')
    expect(wrapper.text()).toContain('Prof')
  })

  it('displays passive perception', async () => {
    const wrapper = await mountSuspended(CombatStatsGrid, {
      props: { character: mockCharacter, stats: mockStats }
    })
    expect(wrapper.text()).toContain('14')
    expect(wrapper.text()).toContain('Passive')
  })

  it('shows temporary HP when present', async () => {
    const wrapper = await mountSuspended(CombatStatsGrid, {
      props: { character: mockCharacter, stats: mockStats }
    })
    expect(wrapper.text()).toContain('+5')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheetCombatStatsGrid"`
Expected: FAIL

**Step 3: Create the component**

```vue
<!-- app/components/character/sheet/CombatStatsGrid.vue -->
<script setup lang="ts">
import type { Character, CharacterStats } from '~/types/character'

defineProps<{
  character: Character
  stats: CharacterStats
}>()

function formatModifier(value: number | null): string {
  if (value === null) return '—'
  return value >= 0 ? `+${value}` : `${value}`
}
</script>

<template>
  <div class="grid grid-cols-3 gap-3">
    <!-- Row 1: HP, AC, Initiative -->
    <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4 text-center">
      <div class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-1">
        HP
      </div>
      <div class="text-2xl font-bold text-gray-900 dark:text-white">
        {{ stats.hit_points.current ?? stats.hit_points.max ?? '—' }}
        <span class="text-lg text-gray-400">/</span>
        {{ stats.hit_points.max ?? '—' }}
      </div>
      <div
        v-if="stats.hit_points.temporary > 0"
        class="text-sm text-success-600 dark:text-success-400"
      >
        +{{ stats.hit_points.temporary }} temp
      </div>
    </div>

    <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4 text-center">
      <div class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-1">
        AC
      </div>
      <div class="text-2xl font-bold text-gray-900 dark:text-white">
        {{ stats.armor_class ?? '—' }}
      </div>
    </div>

    <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4 text-center">
      <div class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-1">
        Initiative
      </div>
      <div class="text-2xl font-bold text-gray-900 dark:text-white">
        {{ formatModifier(stats.initiative_bonus) }}
      </div>
    </div>

    <!-- Row 2: Speed, Proficiency, Passive Perception -->
    <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4 text-center">
      <div class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-1">
        Speed
      </div>
      <div class="text-2xl font-bold text-gray-900 dark:text-white">
        {{ character.speed ?? '—' }} <span class="text-sm font-normal">ft</span>
      </div>
    </div>

    <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4 text-center">
      <div class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-1">
        Prof Bonus
      </div>
      <div class="text-2xl font-bold text-gray-900 dark:text-white">
        {{ formatModifier(character.proficiency_bonus) }}
      </div>
    </div>

    <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4 text-center">
      <div class="text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-1">
        Passive Perc
      </div>
      <div class="text-2xl font-bold text-gray-900 dark:text-white">
        {{ stats.passive_perception ?? '—' }}
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheetCombatStatsGrid"`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/sheet/CombatStatsGrid.vue tests/components/character/sheet/CombatStatsGrid.test.ts
git commit -m "feat(components): add CharacterSheetCombatStatsGrid component (#172)"
```

---

## Task 6: Create CharacterSheetSavingThrowsList Component

**Files:**
- Create: `tests/components/character/sheet/SavingThrowsList.test.ts`
- Create: `app/components/character/sheet/SavingThrowsList.vue`

**Step 1: Write the failing test**

```typescript
// tests/components/character/sheet/SavingThrowsList.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SavingThrowsList from '~/components/character/sheet/SavingThrowsList.vue'

const mockSavingThrows = [
  { ability: 'STR', modifier: 5, proficient: true },
  { ability: 'DEX', modifier: 2, proficient: false },
  { ability: 'CON', modifier: 4, proficient: true },
  { ability: 'INT', modifier: 0, proficient: false },
  { ability: 'WIS', modifier: 1, proficient: false },
  { ability: 'CHA', modifier: -1, proficient: false }
]

describe('CharacterSheetSavingThrowsList', () => {
  it('displays all 6 saving throws', async () => {
    const wrapper = await mountSuspended(SavingThrowsList, {
      props: { savingThrows: mockSavingThrows }
    })
    expect(wrapper.text()).toContain('STR')
    expect(wrapper.text()).toContain('DEX')
    expect(wrapper.text()).toContain('CON')
    expect(wrapper.text()).toContain('INT')
    expect(wrapper.text()).toContain('WIS')
    expect(wrapper.text()).toContain('CHA')
  })

  it('displays modifiers with sign', async () => {
    const wrapper = await mountSuspended(SavingThrowsList, {
      props: { savingThrows: mockSavingThrows }
    })
    expect(wrapper.text()).toContain('+5')
    expect(wrapper.text()).toContain('+4')
    expect(wrapper.text()).toContain('-1')
  })

  it('shows proficiency indicator for proficient saves', async () => {
    const wrapper = await mountSuspended(SavingThrowsList, {
      props: { savingThrows: mockSavingThrows }
    })
    const proficientIndicators = wrapper.findAll('[data-test="proficient"]')
    expect(proficientIndicators.length).toBe(2) // STR and CON
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheetSavingThrowsList"`
Expected: FAIL

**Step 3: Create the component**

```vue
<!-- app/components/character/sheet/SavingThrowsList.vue -->
<script setup lang="ts">
import type { CharacterSavingThrow } from '~/types/character'

defineProps<{
  savingThrows: CharacterSavingThrow[]
}>()

function formatModifier(mod: number): string {
  return mod >= 0 ? `+${mod}` : `${mod}`
}
</script>

<template>
  <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4">
    <h3 class="text-sm font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-3">
      Saving Throws
    </h3>
    <div class="grid grid-cols-2 gap-2">
      <div
        v-for="save in savingThrows"
        :key="save.ability"
        class="flex items-center gap-2"
      >
        <!-- Proficiency indicator -->
        <div
          class="w-3 h-3 rounded-full border-2"
          :class="save.proficient
            ? 'bg-success-500 border-success-500'
            : 'border-gray-400 dark:border-gray-500'"
          :data-test="save.proficient ? 'proficient' : 'not-proficient'"
        />
        <span class="text-sm font-medium text-gray-700 dark:text-gray-300 w-8">
          {{ save.ability }}
        </span>
        <span class="text-sm font-bold text-gray-900 dark:text-white">
          {{ formatModifier(save.modifier) }}
        </span>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheetSavingThrowsList"`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/sheet/SavingThrowsList.vue tests/components/character/sheet/SavingThrowsList.test.ts
git commit -m "feat(components): add CharacterSheetSavingThrowsList component (#172)"
```

---

## Task 7: Create CharacterSheetSkillsList Component

**Files:**
- Create: `tests/components/character/sheet/SkillsList.test.ts`
- Create: `app/components/character/sheet/SkillsList.vue`

**Step 1: Write the failing test**

```typescript
// tests/components/character/sheet/SkillsList.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SkillsList from '~/components/character/sheet/SkillsList.vue'

const mockSkills = [
  { id: 1, name: 'Acrobatics', slug: 'acrobatics', ability_code: 'DEX', modifier: 2, proficient: false, expertise: false },
  { id: 2, name: 'Athletics', slug: 'athletics', ability_code: 'STR', modifier: 5, proficient: true, expertise: false },
  { id: 3, name: 'Stealth', slug: 'stealth', ability_code: 'DEX', modifier: 6, proficient: true, expertise: true }
]

describe('CharacterSheetSkillsList', () => {
  it('displays all skills', async () => {
    const wrapper = await mountSuspended(SkillsList, {
      props: { skills: mockSkills }
    })
    expect(wrapper.text()).toContain('Acrobatics')
    expect(wrapper.text()).toContain('Athletics')
    expect(wrapper.text()).toContain('Stealth')
  })

  it('displays skill modifiers', async () => {
    const wrapper = await mountSuspended(SkillsList, {
      props: { skills: mockSkills }
    })
    expect(wrapper.text()).toContain('+2')
    expect(wrapper.text()).toContain('+5')
    expect(wrapper.text()).toContain('+6')
  })

  it('shows ability code for each skill', async () => {
    const wrapper = await mountSuspended(SkillsList, {
      props: { skills: mockSkills }
    })
    expect(wrapper.text()).toContain('DEX')
    expect(wrapper.text()).toContain('STR')
  })

  it('shows proficiency indicator for proficient skills', async () => {
    const wrapper = await mountSuspended(SkillsList, {
      props: { skills: mockSkills }
    })
    const profIndicators = wrapper.findAll('[data-test="proficient"]')
    expect(profIndicators.length).toBe(2) // Athletics and Stealth
  })

  it('shows expertise indicator for expertise skills', async () => {
    const wrapper = await mountSuspended(SkillsList, {
      props: { skills: mockSkills }
    })
    const expertiseIndicators = wrapper.findAll('[data-test="expertise"]')
    expect(expertiseIndicators.length).toBe(1) // Only Stealth
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheetSkillsList"`
Expected: FAIL

**Step 3: Create the component**

```vue
<!-- app/components/character/sheet/SkillsList.vue -->
<script setup lang="ts">
import type { CharacterSkill } from '~/types/character'

defineProps<{
  skills: CharacterSkill[]
}>()

function formatModifier(mod: number): string {
  return mod >= 0 ? `+${mod}` : `${mod}`
}
</script>

<template>
  <div class="bg-gray-50 dark:bg-gray-800 rounded-lg p-4">
    <h3 class="text-sm font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-3">
      Skills
    </h3>
    <div class="space-y-1">
      <div
        v-for="skill in skills"
        :key="skill.id"
        class="flex items-center gap-2 py-1"
      >
        <!-- Proficiency/Expertise indicator -->
        <div
          class="w-3 h-3 rounded-full border-2 flex-shrink-0"
          :class="{
            'bg-primary-500 border-primary-500': skill.expertise,
            'bg-success-500 border-success-500': skill.proficient && !skill.expertise,
            'border-gray-400 dark:border-gray-500': !skill.proficient
          }"
          :data-test="skill.expertise ? 'expertise' : (skill.proficient ? 'proficient' : 'not-proficient')"
        />
        <!-- Modifier -->
        <span class="text-sm font-bold text-gray-900 dark:text-white w-8">
          {{ formatModifier(skill.modifier) }}
        </span>
        <!-- Skill name -->
        <span class="text-sm text-gray-700 dark:text-gray-300 flex-1">
          {{ skill.name }}
        </span>
        <!-- Ability code -->
        <span class="text-xs text-gray-400 dark:text-gray-500 uppercase">
          {{ skill.ability_code }}
        </span>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheetSkillsList"`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/sheet/SkillsList.vue tests/components/character/sheet/SkillsList.test.ts
git commit -m "feat(components): add CharacterSheetSkillsList component (#172)"
```

---

## Task 8: Create Tab Panel Components (FeaturesPanel, ProficienciesPanel, EquipmentPanel, SpellsPanel, LanguagesPanel)

This task creates 5 simple panel components for the tabbed section. Due to similarity, they're grouped.

**Files:**
- Create: `app/components/character/sheet/FeaturesPanel.vue`
- Create: `app/components/character/sheet/ProficienciesPanel.vue`
- Create: `app/components/character/sheet/EquipmentPanel.vue`
- Create: `app/components/character/sheet/SpellsPanel.vue`
- Create: `app/components/character/sheet/LanguagesPanel.vue`
- Create: `tests/components/character/sheet/panels.test.ts`

**Step 1: Write tests for all panels**

```typescript
// tests/components/character/sheet/panels.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import FeaturesPanel from '~/components/character/sheet/FeaturesPanel.vue'
import ProficienciesPanel from '~/components/character/sheet/ProficienciesPanel.vue'
import EquipmentPanel from '~/components/character/sheet/EquipmentPanel.vue'
import SpellsPanel from '~/components/character/sheet/SpellsPanel.vue'
import LanguagesPanel from '~/components/character/sheet/LanguagesPanel.vue'

describe('CharacterSheetFeaturesPanel', () => {
  const mockFeatures = [
    { id: 1, source: 'class', feature: { name: 'Second Wind', description: 'Heal as bonus action' } },
    { id: 2, source: 'race', feature: { name: 'Darkvision', description: 'See in darkness' } },
    { id: 3, source: 'background', feature: { name: 'Military Rank', description: 'Command soldiers' } }
  ]

  it('displays features grouped by source', async () => {
    const wrapper = await mountSuspended(FeaturesPanel, {
      props: { features: mockFeatures }
    })
    expect(wrapper.text()).toContain('Second Wind')
    expect(wrapper.text()).toContain('Darkvision')
    expect(wrapper.text()).toContain('Military Rank')
  })

  it('shows empty state when no features', async () => {
    const wrapper = await mountSuspended(FeaturesPanel, {
      props: { features: [] }
    })
    expect(wrapper.text()).toContain('No features')
  })
})

describe('CharacterSheetProficienciesPanel', () => {
  const mockProficiencies = [
    { id: 1, proficiency_type: { name: 'Longsword', category: 'weapons' } },
    { id: 2, proficiency_type: { name: 'Heavy Armor', category: 'armor' } }
  ]

  it('displays proficiencies', async () => {
    const wrapper = await mountSuspended(ProficienciesPanel, {
      props: { proficiencies: mockProficiencies }
    })
    expect(wrapper.text()).toContain('Longsword')
    expect(wrapper.text()).toContain('Heavy Armor')
  })

  it('shows empty state when no proficiencies', async () => {
    const wrapper = await mountSuspended(ProficienciesPanel, {
      props: { proficiencies: [] }
    })
    expect(wrapper.text()).toContain('No proficiencies')
  })
})

describe('CharacterSheetEquipmentPanel', () => {
  const mockEquipment = [
    { id: 1, item: { name: 'Longsword' }, quantity: 1, equipped: true },
    { id: 2, item: { name: 'Handaxe' }, quantity: 2, equipped: false }
  ]

  it('displays equipment with quantities', async () => {
    const wrapper = await mountSuspended(EquipmentPanel, {
      props: { equipment: mockEquipment }
    })
    expect(wrapper.text()).toContain('Longsword')
    expect(wrapper.text()).toContain('Handaxe')
    expect(wrapper.text()).toContain('2')
  })

  it('shows empty state when no equipment', async () => {
    const wrapper = await mountSuspended(EquipmentPanel, {
      props: { equipment: [] }
    })
    expect(wrapper.text()).toContain('No equipment')
  })
})

describe('CharacterSheetSpellsPanel', () => {
  const mockSpells = [
    { id: 1, spell: { name: 'Fire Bolt', level: 0 }, prepared: true },
    { id: 2, spell: { name: 'Magic Missile', level: 1 }, prepared: true }
  ]
  const mockStats = {
    spellcasting: { ability: 'INT', spell_save_dc: 13, spell_attack_bonus: 5 }
  }

  it('displays spells', async () => {
    const wrapper = await mountSuspended(SpellsPanel, {
      props: { spells: mockSpells, stats: mockStats }
    })
    expect(wrapper.text()).toContain('Fire Bolt')
    expect(wrapper.text()).toContain('Magic Missile')
  })

  it('displays spellcasting stats', async () => {
    const wrapper = await mountSuspended(SpellsPanel, {
      props: { spells: mockSpells, stats: mockStats }
    })
    expect(wrapper.text()).toContain('13') // DC
    expect(wrapper.text()).toContain('+5') // Attack bonus
  })
})

describe('CharacterSheetLanguagesPanel', () => {
  const mockLanguages = [
    { id: 1, language: { name: 'Common' } },
    { id: 2, language: { name: 'Elvish' } }
  ]

  it('displays languages as tags', async () => {
    const wrapper = await mountSuspended(LanguagesPanel, {
      props: { languages: mockLanguages }
    })
    expect(wrapper.text()).toContain('Common')
    expect(wrapper.text()).toContain('Elvish')
  })

  it('shows empty state when no languages', async () => {
    const wrapper = await mountSuspended(LanguagesPanel, {
      props: { languages: [] }
    })
    expect(wrapper.text()).toContain('No languages')
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheet.*Panel"`
Expected: FAIL

**Step 3: Create FeaturesPanel component**

```vue
<!-- app/components/character/sheet/FeaturesPanel.vue -->
<script setup lang="ts">
import type { CharacterFeature } from '~/types/character'

const props = defineProps<{
  features: CharacterFeature[]
}>()

const featuresBySource = computed(() => {
  const grouped: Record<string, CharacterFeature[]> = {
    class: [],
    race: [],
    background: []
  }
  for (const feature of props.features) {
    if (grouped[feature.source]) {
      grouped[feature.source].push(feature)
    }
  }
  return grouped
})

const sourceLabels: Record<string, string> = {
  class: 'Class Features',
  race: 'Racial Traits',
  background: 'Background Feature'
}
</script>

<template>
  <div class="space-y-6">
    <div
      v-if="features.length === 0"
      class="text-center text-gray-500 dark:text-gray-400 py-8"
    >
      No features yet
    </div>

    <template v-else>
      <div
        v-for="(featureList, source) in featuresBySource"
        :key="source"
      >
        <template v-if="featureList.length > 0">
          <h4 class="text-sm font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-2">
            {{ sourceLabels[source] }}
          </h4>
          <ul class="space-y-2">
            <li
              v-for="feature in featureList"
              :key="feature.id"
              class="bg-gray-50 dark:bg-gray-800 rounded-lg p-3"
            >
              <div class="font-medium text-gray-900 dark:text-white">
                {{ feature.feature.name }}
              </div>
              <div
                v-if="feature.feature.description"
                class="text-sm text-gray-600 dark:text-gray-400 mt-1 line-clamp-2"
              >
                {{ feature.feature.description }}
              </div>
            </li>
          </ul>
        </template>
      </div>
    </template>
  </div>
</template>
```

**Step 4: Create ProficienciesPanel component**

```vue
<!-- app/components/character/sheet/ProficienciesPanel.vue -->
<script setup lang="ts">
import type { CharacterProficiency } from '~/types/character'

const props = defineProps<{
  proficiencies: CharacterProficiency[]
}>()

// Filter to only non-skill proficiencies (tools, weapons, armor)
const typeProficiencies = computed(() =>
  props.proficiencies.filter(p => p.proficiency_type)
)

// Group by category
const proficienciesByCategory = computed(() => {
  const grouped: Record<string, CharacterProficiency[]> = {}
  for (const prof of typeProficiencies.value) {
    const category = prof.proficiency_type?.category ?? 'other'
    if (!grouped[category]) grouped[category] = []
    grouped[category].push(prof)
  }
  return grouped
})
</script>

<template>
  <div class="space-y-4">
    <div
      v-if="typeProficiencies.length === 0"
      class="text-center text-gray-500 dark:text-gray-400 py-8"
    >
      No proficiencies yet
    </div>

    <template v-else>
      <div
        v-for="(profs, category) in proficienciesByCategory"
        :key="category"
      >
        <h4 class="text-sm font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider mb-2">
          {{ category }}
        </h4>
        <div class="flex flex-wrap gap-2">
          <UBadge
            v-for="prof in profs"
            :key="prof.id"
            color="neutral"
            variant="subtle"
            size="md"
          >
            {{ prof.proficiency_type?.name }}
          </UBadge>
        </div>
      </div>
    </template>
  </div>
</template>
```

**Step 5: Create EquipmentPanel component**

```vue
<!-- app/components/character/sheet/EquipmentPanel.vue -->
<script setup lang="ts">
import type { CharacterEquipment } from '~/types/character'

defineProps<{
  equipment: CharacterEquipment[]
}>()
</script>

<template>
  <div>
    <div
      v-if="equipment.length === 0"
      class="text-center text-gray-500 dark:text-gray-400 py-8"
    >
      No equipment yet
    </div>

    <ul
      v-else
      class="divide-y divide-gray-200 dark:divide-gray-700"
    >
      <li
        v-for="item in equipment"
        :key="item.id"
        class="py-3 flex items-center justify-between"
      >
        <div class="flex items-center gap-2">
          <UIcon
            v-if="item.equipped"
            name="i-heroicons-check-circle"
            class="w-5 h-5 text-success-500"
          />
          <UIcon
            v-else
            name="i-heroicons-minus-circle"
            class="w-5 h-5 text-gray-400"
          />
          <span class="text-gray-900 dark:text-white">
            {{ item.item?.name ?? item.description ?? 'Unknown item' }}
          </span>
        </div>
        <span
          v-if="item.quantity > 1"
          class="text-sm text-gray-500 dark:text-gray-400"
        >
          ×{{ item.quantity }}
        </span>
      </li>
    </ul>
  </div>
</template>
```

**Step 6: Create SpellsPanel component**

```vue
<!-- app/components/character/sheet/SpellsPanel.vue -->
<script setup lang="ts">
import type { CharacterSpell, CharacterStats } from '~/types/character'

const props = defineProps<{
  spells: CharacterSpell[]
  stats: CharacterStats
}>()

const cantrips = computed(() => props.spells.filter(s => s.spell.level === 0))
const leveledSpells = computed(() => props.spells.filter(s => s.spell.level > 0))

function formatModifier(value: number): string {
  return value >= 0 ? `+${value}` : `${value}`
}
</script>

<template>
  <div class="space-y-4">
    <!-- Spellcasting stats -->
    <div
      v-if="stats.spellcasting"
      class="flex gap-4 flex-wrap"
    >
      <div class="bg-gray-50 dark:bg-gray-800 rounded-lg px-4 py-2 text-center">
        <div class="text-xs text-gray-500 uppercase">
          Spell DC
        </div>
        <div class="text-lg font-bold">
          {{ stats.spellcasting.spell_save_dc }}
        </div>
      </div>
      <div class="bg-gray-50 dark:bg-gray-800 rounded-lg px-4 py-2 text-center">
        <div class="text-xs text-gray-500 uppercase">
          Attack
        </div>
        <div class="text-lg font-bold">
          {{ formatModifier(stats.spellcasting.spell_attack_bonus) }}
        </div>
      </div>
      <div class="bg-gray-50 dark:bg-gray-800 rounded-lg px-4 py-2 text-center">
        <div class="text-xs text-gray-500 uppercase">
          Ability
        </div>
        <div class="text-lg font-bold">
          {{ stats.spellcasting.ability }}
        </div>
      </div>
    </div>

    <!-- Cantrips -->
    <div v-if="cantrips.length > 0">
      <h4 class="text-sm font-semibold text-gray-500 uppercase mb-2">
        Cantrips
      </h4>
      <div class="flex flex-wrap gap-2">
        <UBadge
          v-for="spell in cantrips"
          :key="spell.id"
          color="spell"
          variant="subtle"
          size="md"
        >
          {{ spell.spell.name }}
        </UBadge>
      </div>
    </div>

    <!-- Leveled Spells -->
    <div v-if="leveledSpells.length > 0">
      <h4 class="text-sm font-semibold text-gray-500 uppercase mb-2">
        Spells
      </h4>
      <ul class="space-y-1">
        <li
          v-for="spell in leveledSpells"
          :key="spell.id"
          class="flex items-center gap-2"
        >
          <UIcon
            :name="spell.prepared ? 'i-heroicons-check-circle' : 'i-heroicons-minus-circle'"
            :class="spell.prepared ? 'text-success-500' : 'text-gray-400'"
            class="w-4 h-4"
          />
          <span class="text-gray-900 dark:text-white">{{ spell.spell.name }}</span>
          <UBadge
            color="neutral"
            variant="subtle"
            size="xs"
          >
            Lvl {{ spell.spell.level }}
          </UBadge>
        </li>
      </ul>
    </div>

    <div
      v-if="spells.length === 0"
      class="text-center text-gray-500 dark:text-gray-400 py-8"
    >
      No spells known
    </div>
  </div>
</template>
```

**Step 7: Create LanguagesPanel component**

```vue
<!-- app/components/character/sheet/LanguagesPanel.vue -->
<script setup lang="ts">
import type { CharacterLanguage } from '~/types/character'

defineProps<{
  languages: CharacterLanguage[]
}>()
</script>

<template>
  <div>
    <div
      v-if="languages.length === 0"
      class="text-center text-gray-500 dark:text-gray-400 py-8"
    >
      No languages yet
    </div>

    <div
      v-else
      class="flex flex-wrap gap-2"
    >
      <UBadge
        v-for="lang in languages"
        :key="lang.id"
        color="neutral"
        variant="subtle"
        size="md"
      >
        {{ lang.language.name }}
      </UBadge>
    </div>
  </div>
</template>
```

**Step 8: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="CharacterSheet.*Panel"`
Expected: PASS

**Step 9: Commit**

```bash
git add app/components/character/sheet/*.vue tests/components/character/sheet/panels.test.ts
git commit -m "feat(components): add character sheet tab panel components (#172)"
```

---

## Task 9: Rewrite Character View Page

**Files:**
- Modify: `app/pages/characters/[id]/index.vue`

**Step 1: Backup and rewrite the page**

Replace the entire content of `app/pages/characters/[id]/index.vue`:

```vue
<!-- app/pages/characters/[id]/index.vue -->
<script setup lang="ts">
/**
 * Character View Page - Full Character Sheet
 *
 * Displays comprehensive D&D 5e character sheet with all data sections.
 * Uses useCharacterSheet composable for parallel data fetching.
 */

const route = useRoute()
const characterId = computed(() => route.params.id as string)

const {
  character,
  stats,
  proficiencies,
  features,
  equipment,
  spells,
  languages,
  skills,
  savingThrows,
  loading,
  error
} = useCharacterSheet(characterId)

useSeoMeta({
  title: () => character.value?.name ?? 'Character Sheet',
  description: () => `View ${character.value?.name ?? 'character'} - D&D 5e Character Sheet`
})

// Tab items for bottom section
const tabItems = computed(() => {
  const items = [
    { label: 'Features', slot: 'features', icon: 'i-heroicons-star' },
    { label: 'Proficiencies', slot: 'proficiencies', icon: 'i-heroicons-academic-cap' },
    { label: 'Equipment', slot: 'equipment', icon: 'i-heroicons-briefcase' },
    { label: 'Languages', slot: 'languages', icon: 'i-heroicons-language' }
  ]
  // Only show Spells tab for casters
  if (stats.value?.spellcasting) {
    items.splice(3, 0, { label: 'Spells', slot: 'spells', icon: 'i-heroicons-sparkles' })
  }
  return items
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-5xl">
    <!-- Back Link -->
    <UButton
      to="/characters"
      variant="ghost"
      icon="i-heroicons-arrow-left"
      class="mb-6"
    >
      Back to Characters
    </UButton>

    <!-- Loading State -->
    <div
      v-if="loading"
      class="space-y-6"
    >
      <USkeleton class="h-20 w-full" />
      <div class="grid lg:grid-cols-[200px_1fr] gap-6">
        <USkeleton class="h-80" />
        <div class="space-y-4">
          <USkeleton class="h-32" />
          <USkeleton class="h-48" />
        </div>
      </div>
    </div>

    <!-- Error State -->
    <UAlert
      v-else-if="error"
      color="error"
      icon="i-heroicons-exclamation-triangle"
      title="Failed to load character"
      :description="error.message"
    />

    <!-- Character Sheet -->
    <div
      v-else-if="character && stats"
      class="space-y-6"
    >
      <!-- Header -->
      <CharacterSheetHeader :character="character" />

      <!-- Main Grid: Abilities sidebar + Stats/Skills -->
      <div class="grid lg:grid-cols-[200px_1fr] gap-6">
        <!-- Left Sidebar: Ability Scores -->
        <CharacterSheetAbilityScoreBlock :stats="stats" />

        <!-- Right: Combat Stats + Saves/Skills -->
        <div class="space-y-6">
          <!-- Combat Stats Grid -->
          <CharacterSheetCombatStatsGrid
            :character="character"
            :stats="stats"
          />

          <!-- Saving Throws and Skills -->
          <div class="grid md:grid-cols-2 gap-6">
            <CharacterSheetSavingThrowsList :saving-throws="savingThrows" />
            <CharacterSheetSkillsList :skills="skills" />
          </div>
        </div>
      </div>

      <!-- Bottom Tabs -->
      <UTabs
        :items="tabItems"
        class="mt-8"
      >
        <template #features>
          <CharacterSheetFeaturesPanel :features="features" />
        </template>

        <template #proficiencies>
          <CharacterSheetProficienciesPanel :proficiencies="proficiencies" />
        </template>

        <template #equipment>
          <CharacterSheetEquipmentPanel :equipment="equipment" />
        </template>

        <template #spells>
          <CharacterSheetSpellsPanel
            :spells="spells"
            :stats="stats"
          />
        </template>

        <template #languages>
          <CharacterSheetLanguagesPanel :languages="languages" />
        </template>
      </UTabs>
    </div>
  </div>
</template>
```

**Step 2: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: PASS

**Step 3: Run existing character tests**

Run: `docker compose exec nuxt npm run test -- --run --testNamePattern="character"`
Expected: PASS (or update tests as needed)

**Step 4: Commit**

```bash
git add app/pages/characters/[id]/index.vue
git commit -m "feat(pages): rewrite character view page with full sheet layout (#172)"
```

---

## Task 10: Add Nitro Route for Languages Endpoint

**Files:**
- Create: `server/api/characters/[id]/languages.get.ts`

**Step 1: Create the Nitro route**

```typescript
// server/api/characters/[id]/languages.get.ts
/**
 * Get character languages endpoint - Proxies to Laravel backend
 *
 * @example GET /api/characters/1/languages
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/languages`)
  return data
})
```

**Step 2: Commit**

```bash
git add server/api/characters/[id]/languages.get.ts
git commit -m "feat(api): add Nitro route for character languages (#172)"
```

---

## Task 11: Final Testing and Polish

**Step 1: Run full test suite**

Run: `docker compose exec nuxt npm run test -- --run`
Expected: PASS

**Step 2: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: PASS

**Step 3: Run linter**

Run: `docker compose exec nuxt npm run lint:fix`
Expected: PASS

**Step 4: Manual browser verification**

1. Navigate to `http://localhost:4000/characters`
2. Click on a character to view the new sheet
3. Verify all sections display correctly
4. Test dark mode toggle
5. Test responsive behavior (resize window)

**Step 5: Final commit and push**

```bash
git add -A
git commit -m "chore: final polish for character sheet redesign (#172)"
git push -u origin feature/issue-172-character-sheet-redesign
```

**Step 6: Create PR**

```bash
gh pr create --title "feat: Redesign character view page with D&D 5e character sheet layout (#172)" --body "$(cat <<'EOF'
## Summary

- Add `useCharacterSheet` composable for parallel data fetching from 7 API endpoints
- Create 10 new components in `app/components/character/sheet/`:
  - Header, AbilityScoreBlock, CombatStatsGrid
  - SavingThrowsList, SkillsList
  - FeaturesPanel, ProficienciesPanel, EquipmentPanel, SpellsPanel, LanguagesPanel
- Calculate skill modifiers client-side from ability scores + proficiencies
- Tabbed bottom section for Features/Proficiencies/Equipment/Spells/Languages
- Responsive layout with sidebar on desktop, stacked on mobile
- Full dark mode support

## Test plan

- [ ] All new component tests pass
- [ ] Composable test passes
- [ ] TypeScript compiles
- [ ] View complete character - all sections display
- [ ] View incomplete character - shows Draft badge and Edit button
- [ ] Caster character shows Spells tab
- [ ] Non-caster hides Spells tab
- [ ] Dark mode works
- [ ] Mobile responsive layout works

Closes #172

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

---

## Summary

| Task | Description | Files Created/Modified |
|------|-------------|----------------------|
| 1 | Type definitions | `app/types/character.ts` |
| 2 | useCharacterSheet composable | `app/composables/useCharacterSheet.ts`, test |
| 3 | Header component | `app/components/character/sheet/Header.vue`, test |
| 4 | AbilityScoreBlock component | `app/components/character/sheet/AbilityScoreBlock.vue`, test |
| 5 | CombatStatsGrid component | `app/components/character/sheet/CombatStatsGrid.vue`, test |
| 6 | SavingThrowsList component | `app/components/character/sheet/SavingThrowsList.vue`, test |
| 7 | SkillsList component | `app/components/character/sheet/SkillsList.vue`, test |
| 8 | Tab panel components (5) | 5 panel components, 1 combined test file |
| 9 | Page rewrite | `app/pages/characters/[id]/index.vue` |
| 10 | Languages Nitro route | `server/api/characters/[id]/languages.get.ts` |
| 11 | Final testing | All files, PR creation |

**Total: 11 tasks, ~15-20 commits expected**
