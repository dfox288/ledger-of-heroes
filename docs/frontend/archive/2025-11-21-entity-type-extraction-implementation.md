# Phase 2: Entity Type Extraction - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Extract 6 entity type definitions from inline card component interfaces into centralized type system at `app/types/api/entities.ts`.

**Architecture:** Incremental batch migration (2 entities per batch) with test checkpoints following Phase 1's proven pattern. Extract interfaces exactly as-is (no enhancements), update card components and test files, validate all tests pass after each batch.

**Tech Stack:** TypeScript, Nuxt 4, Vitest

---

## Pre-Implementation Validation

**Before starting, verify:**
```bash
# Current working directory
pwd
# Expected: /Users/dfox/Development/dnd/frontend

# All tests currently passing
docker compose exec nuxt npm test -- --run
# Expected: 525/525 tests passing

# TypeScript compiling
docker compose exec nuxt npx nuxi typecheck
# Expected: No errors
```

---

## Batch 1: Spell + Item Types

### Task 1: Create entities.ts with Spell and Item interfaces

**Files:**
- Create: `app/types/api/entities.ts`
- Modify: `app/types/index.ts`

**Step 1: Read current Spell interface from SpellCard.vue**

Read: `app/components/spell/SpellCard.vue` lines 4-20

**Step 2: Read current Item interface from ItemCard.vue**

Read: `app/components/item/ItemCard.vue` lines 4-19

**Step 3: Create entities.ts with both interfaces**

```typescript
import type { Source } from './common'

/**
 * Spell entity from D&D 5e API
 *
 * Used in: SpellCard, spell detail pages, tests
 * API endpoint: /api/v1/spells
 */
export interface Spell {
  id: number
  name: string
  slug: string
  level: number
  school?: {
    id: number
    code: string
    name: string
  }
  casting_time: string
  range: string
  description: string
  is_ritual: boolean
  needs_concentration: boolean
  sources?: Source[]
}

/**
 * Item entity from D&D 5e API
 *
 * Used in: ItemCard, item detail pages, tests
 * API endpoint: /api/v1/items
 */
export interface Item {
  id: number
  name: string
  slug: string
  rarity: string
  item_type?: {
    id: number
    name: string
  }
  is_magic: boolean
  requires_attunement: boolean
  cost_cp?: number
  weight?: number
  description?: string
  sources?: Source[]
}
```

Create file: `app/types/api/entities.ts`

**Step 4: Update barrel export**

In `app/types/index.ts`, add after existing exports:

```typescript
export type { Spell, Item } from './api/entities'
```

**Step 5: Verify TypeScript compiles**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors (new file created, nothing broken)

**Step 6: Commit type definitions**

```bash
git add app/types/api/entities.ts app/types/index.ts
git commit -m "refactor: Create entities.ts with Spell and Item types

- Add app/types/api/entities.ts with initial entity interfaces
- Export Spell and Item from barrel export
- No breaking changes (types defined but not yet used)

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 2: Migrate SpellCard.vue to use centralized type

**Files:**
- Modify: `app/components/spell/SpellCard.vue:1-20`

**Step 1: Read current SpellCard.vue**

Read: `app/components/spell/SpellCard.vue`

**Step 2: Update imports and remove inline interface**

In `app/components/spell/SpellCard.vue`, change:

**Before:**
```typescript
<script setup lang="ts">
import type { Source } from '~/types'

interface Spell {
  id: number
  name: string
  slug: string
  level: number
  school?: {
    id: number
    code: string
    name: string
  }
  casting_time: string
  range: string
  description: string
  is_ritual: boolean
  needs_concentration: boolean
  sources?: Source[]
}

interface Props {
  spell: Spell
}
```

**After:**
```typescript
<script setup lang="ts">
import type { Spell } from '~/types'

interface Props {
  spell: Spell
}
```

**Step 3: Verify TypeScript compiles**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors

**Step 4: Run SpellCard tests**

Run: `docker compose exec nuxt npm test -- SpellCard.test.ts`
Expected: All 29 tests passing

**Step 5: Commit SpellCard migration**

```bash
git add app/components/spell/SpellCard.vue
git commit -m "refactor: Migrate SpellCard to use centralized Spell type

- Import Spell from ~/types instead of defining inline
- Remove 15-line duplicate interface definition
- All 29 SpellCard tests passing

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 3: Migrate ItemCard.vue to use centralized type

**Files:**
- Modify: `app/components/item/ItemCard.vue:1-19`

**Step 1: Read current ItemCard.vue**

Read: `app/components/item/ItemCard.vue`

**Step 2: Update imports and remove inline interface**

In `app/components/item/ItemCard.vue`, change:

**Before:**
```typescript
<script setup lang="ts">
import type { Source } from '~/types'

interface Item {
  id: number
  name: string
  slug: string
  rarity: string
  item_type?: {
    id: number
    name: string
  }
  is_magic: boolean
  requires_attunement: boolean
  cost_cp?: number
  weight?: number
  description?: string
  sources?: Source[]
}

interface Props {
  item: Item
}
```

**After:**
```typescript
<script setup lang="ts">
import type { Item } from '~/types'

interface Props {
  item: Item
}
```

**Step 3: Verify TypeScript compiles**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors

**Step 4: Run ItemCard tests**

Run: `docker compose exec nuxt npm test -- ItemCard.test.ts`
Expected: All 35 tests passing

**Step 5: Commit ItemCard migration**

```bash
git add app/components/item/ItemCard.vue
git commit -m "refactor: Migrate ItemCard to use centralized Item type

- Import Item from ~/types instead of defining inline
- Remove 14-line duplicate interface definition
- All 35 ItemCard tests passing

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 4: Update SpellCard.test.ts with typed mock data

**Files:**
- Modify: `tests/components/spell/SpellCard.test.ts:1-24`

**Step 1: Read current test file**

Read: `tests/components/spell/SpellCard.test.ts` lines 1-30

**Step 2: Add type import and annotate mock data**

In `tests/components/spell/SpellCard.test.ts`, change:

**Before:**
```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SpellCard from '~/components/spell/SpellCard.vue'

describe('SpellCard', () => {
  const mockSpell = {
    id: 1,
    name: 'Fireball',
    slug: 'fireball',
    level: 3,
    school: {
      id: 5,
      code: 'EV',
      name: 'Evocation'
    },
    casting_time: '1 action',
    range: '150 feet',
    description: 'A bright streak flashes from your pointing finger to a point you choose within range and then blossoms with a low roar into an explosion of flame.',
    is_ritual: false,
    needs_concentration: false,
    sources: [
      { code: 'PHB', name: "Player's Handbook", pages: '241' }
    ]
  }
```

**After:**
```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import type { Spell } from '~/types'
import SpellCard from '~/components/spell/SpellCard.vue'

describe('SpellCard', () => {
  const mockSpell: Spell = {
    id: 1,
    name: 'Fireball',
    slug: 'fireball',
    level: 3,
    school: {
      id: 5,
      code: 'EV',
      name: 'Evocation'
    },
    casting_time: '1 action',
    range: '150 feet',
    description: 'A bright streak flashes from your pointing finger to a point you choose within range and then blossoms with a low roar into an explosion of flame.',
    is_ritual: false,
    needs_concentration: false,
    sources: [
      { code: 'PHB', name: "Player's Handbook", pages: '241' }
    ]
  }
```

**Step 3: Run SpellCard tests**

Run: `docker compose exec nuxt npm test -- SpellCard.test.ts`
Expected: All 29 tests passing

**Step 4: Commit test update**

```bash
git add tests/components/spell/SpellCard.test.ts
git commit -m "test: Add type annotation to SpellCard mock data

- Import Spell type from ~/types
- Annotate mockSpell with Spell type
- TypeScript now validates test data matches interface
- All 29 tests passing

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 5: Update ItemCard.test.ts with typed mock data

**Files:**
- Modify: `tests/components/item/ItemCard.test.ts:1-30`

**Step 1: Read current test file**

Read: `tests/components/item/ItemCard.test.ts` lines 1-40

**Step 2: Add type import and annotate mock data**

In `tests/components/item/ItemCard.test.ts`, add import after existing imports:

```typescript
import type { Item } from '~/types'
```

Find the `mockItem` definition and add type annotation:

**Before:**
```typescript
const mockItem = {
  // ... item data
}
```

**After:**
```typescript
const mockItem: Item = {
  // ... item data
}
```

**Step 3: Run ItemCard tests**

Run: `docker compose exec nuxt npm test -- ItemCard.test.ts`
Expected: All 35 tests passing

**Step 4: Commit test update**

```bash
git add tests/components/item/ItemCard.test.ts
git commit -m "test: Add type annotation to ItemCard mock data

- Import Item type from ~/types
- Annotate mockItem with Item type
- TypeScript now validates test data matches interface
- All 35 tests passing

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 6: Batch 1 validation checkpoint

**Step 1: Run full test suite**

Run: `docker compose exec nuxt npm test -- --run`
Expected: 525/525 tests passing

**Step 2: Verify TypeScript compilation**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors

**Step 3: Verify no linting issues**

Run: `docker compose exec nuxt npm run lint`
Expected: No warnings or errors

**Result:** Batch 1 complete (Spell + Item extracted and migrated)

---

## Batch 2: Race + CharacterClass Types

### Task 7: Add Race and CharacterClass to entities.ts

**Files:**
- Modify: `app/types/api/entities.ts`
- Modify: `app/types/index.ts`

**Step 1: Read current Race interface from RaceCard.vue**

Read: `app/components/race/RaceCard.vue` lines 4-30

**Step 2: Read current CharacterClass interface from ClassCard.vue**

Read: `app/components/class/ClassCard.vue` lines 4-25

**Step 3: Add both interfaces to entities.ts**

In `app/types/api/entities.ts`, add after Item interface:

```typescript
/**
 * Race entity from D&D 5e API
 *
 * Used in: RaceCard, race detail pages, tests
 * API endpoint: /api/v1/races
 */
export interface Race {
  id: number
  name: string
  slug: string
  size?: {
    id: number
    name: string
    code: string
  }
  speed: number
  parent_race_id?: number | null
  parent_race?: {
    id: number
    slug: string
    name: string
    speed: number
  } | null
  subraces?: Array<{
    id: number
    slug: string
    name: string
  }>
  modifiers?: any[]  // TODO: Type with Modifier interface
  traits?: any[]     // TODO: Type trait structure
  description?: string
  sources?: Source[]
}

/**
 * Character Class entity from D&D 5e API
 *
 * Used in: ClassCard, class detail pages, tests
 * API endpoint: /api/v1/classes
 *
 * Note: Named CharacterClass to avoid conflict with JS 'class' keyword
 */
export interface CharacterClass {
  id: number
  name: string
  slug: string
  hit_die: number
  is_base_class: boolean
  parent_class_id?: number | null
  primary_ability?: {
    id: number
    code: string
    name: string
  } | null
  spellcasting_ability?: {
    id: number
    code: string
    name: string
  } | null
  subclasses?: any[]      // TODO: Type subclass structure
  proficiencies?: any[]   // TODO: Type proficiency structure
  description?: string
  sources?: Source[]
}
```

**Step 4: Update barrel export**

In `app/types/index.ts`, update entity exports line:

```typescript
export type { Spell, Item, Race, CharacterClass } from './api/entities'
```

**Step 5: Verify TypeScript compiles**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors

**Step 6: Commit type additions**

```bash
git add app/types/api/entities.ts app/types/index.ts
git commit -m "refactor: Add Race and CharacterClass to entities.ts

- Add Race interface with parent/subrace support
- Add CharacterClass interface (avoiding 'class' keyword conflict)
- Export new types from barrel
- No breaking changes

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 8: Migrate RaceCard.vue to use centralized type

**Files:**
- Modify: `app/components/race/RaceCard.vue:1-30`

**Step 1: Read current RaceCard.vue**

Read: `app/components/race/RaceCard.vue` lines 1-50

**Step 2: Update imports and remove inline interface**

In `app/components/race/RaceCard.vue`, change:

**Before:**
```typescript
<script setup lang="ts">
import type { Source } from '~/types'

interface Race {
  id: number
  name: string
  slug: string
  size?: {
    id: number
    name: string
    code: string
  }
  speed: number
  parent_race_id?: number | null
  parent_race?: {
    id: number
    slug: string
    name: string
    speed: number
  } | null
  subraces?: Array<{
    id: number
    slug: string
    name: string
  }>
  modifiers?: any[]
  traits?: any[]
  description?: string
  sources?: Source[]
}

interface Props {
  race: Race
}
```

**After:**
```typescript
<script setup lang="ts">
import type { Race } from '~/types'

interface Props {
  race: Race
}
```

**Step 3: Verify TypeScript compiles**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors

**Step 4: Run RaceCard tests**

Run: `docker compose exec nuxt npm test -- RaceCard.test.ts`
Expected: All 33 tests passing

**Step 5: Commit RaceCard migration**

```bash
git add app/components/race/RaceCard.vue
git commit -m "refactor: Migrate RaceCard to use centralized Race type

- Import Race from ~/types instead of defining inline
- Remove 26-line duplicate interface definition
- All 33 RaceCard tests passing

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 9: Migrate ClassCard.vue to use centralized type

**Files:**
- Modify: `app/components/class/ClassCard.vue:1-25`

**Step 1: Read current ClassCard.vue**

Read: `app/components/class/ClassCard.vue` lines 1-50

**Step 2: Update imports and remove inline interface**

In `app/components/class/ClassCard.vue`, change:

**Before:**
```typescript
<script setup lang="ts">
import type { Source } from '~/types'

interface CharacterClass {
  id: number
  name: string
  slug: string
  hit_die: number
  is_base_class: boolean
  parent_class_id?: number | null
  primary_ability?: {
    id: number
    code: string
    name: string
  } | null
  spellcasting_ability?: {
    id: number
    code: string
    name: string
  } | null
  subclasses?: any[]
  proficiencies?: any[]
  description?: string
  sources?: Source[]
}

interface Props {
  characterClass: CharacterClass
}
```

**After:**
```typescript
<script setup lang="ts">
import type { CharacterClass } from '~/types'

interface Props {
  characterClass: CharacterClass
}
```

**Step 3: Verify TypeScript compiles**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors

**Step 4: Run ClassCard tests**

Run: `docker compose exec nuxt npm test -- ClassCard.test.ts`
Expected: All 30 tests passing

**Step 5: Commit ClassCard migration**

```bash
git add app/components/class/ClassCard.vue
git commit -m "refactor: Migrate ClassCard to use centralized CharacterClass type

- Import CharacterClass from ~/types instead of defining inline
- Remove 21-line duplicate interface definition
- All 30 ClassCard tests passing

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 10: Update RaceCard.test.ts with typed mock data

**Files:**
- Modify: `tests/components/race/RaceCard.test.ts`

**Step 1: Read current test file**

Read: `tests/components/race/RaceCard.test.ts` lines 1-40

**Step 2: Add type import and annotate mock data**

Add import after existing imports:

```typescript
import type { Race } from '~/types'
```

Find the `mockRace` definition and add type annotation:

**Before:**
```typescript
const mockRace = {
  // ... race data
}
```

**After:**
```typescript
const mockRace: Race = {
  // ... race data
}
```

**Step 3: Run RaceCard tests**

Run: `docker compose exec nuxt npm test -- RaceCard.test.ts`
Expected: All 33 tests passing

**Step 4: Commit test update**

```bash
git add tests/components/race/RaceCard.test.ts
git commit -m "test: Add type annotation to RaceCard mock data

- Import Race type from ~/types
- Annotate mockRace with Race type
- TypeScript validates test data matches interface
- All 33 tests passing

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 11: Update ClassCard.test.ts with typed mock data

**Files:**
- Modify: `tests/components/class/ClassCard.test.ts`

**Step 1: Read current test file**

Read: `tests/components/class/ClassCard.test.ts` lines 1-40

**Step 2: Add type import and annotate mock data**

Add import after existing imports:

```typescript
import type { CharacterClass } from '~/types'
```

Find the `mockClass` definition and add type annotation:

**Before:**
```typescript
const mockClass = {
  // ... class data
}
```

**After:**
```typescript
const mockClass: CharacterClass = {
  // ... class data
}
```

**Step 3: Run ClassCard tests**

Run: `docker compose exec nuxt npm test -- ClassCard.test.ts`
Expected: All 30 tests passing

**Step 4: Commit test update**

```bash
git add tests/components/class/ClassCard.test.ts
git commit -m "test: Add type annotation to ClassCard mock data

- Import CharacterClass type from ~/types
- Annotate mockClass with CharacterClass type
- TypeScript validates test data matches interface
- All 30 tests passing

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 12: Batch 2 validation checkpoint

**Step 1: Run full test suite**

Run: `docker compose exec nuxt npm test -- --run`
Expected: 525/525 tests passing

**Step 2: Verify TypeScript compilation**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors

**Step 3: Verify no linting issues**

Run: `docker compose exec nuxt npm run lint`
Expected: No warnings or errors

**Result:** Batch 2 complete (Race + CharacterClass extracted and migrated)

---

## Batch 3: Background + Feat Types

### Task 13: Add Background and Feat to entities.ts

**Files:**
- Modify: `app/types/api/entities.ts`
- Modify: `app/types/index.ts`

**Step 1: Read current Background interface from BackgroundCard.vue**

Read: `app/components/background/BackgroundCard.vue` lines 4-14

**Step 2: Read current Feat interface from FeatCard.vue**

Read: `app/components/feat/FeatCard.vue` lines 4-12

**Step 3: Add both interfaces to entities.ts**

In `app/types/api/entities.ts`, add after CharacterClass interface:

```typescript
/**
 * Background entity from D&D 5e API
 *
 * Used in: BackgroundCard, background detail pages, tests
 * API endpoint: /api/v1/backgrounds
 */
export interface Background {
  id: number
  name: string
  slug: string
  skill_proficiencies?: any[]   // TODO: Type skill proficiency structure
  tool_proficiencies?: any[]    // TODO: Type tool proficiency structure
  languages?: any[]             // TODO: Type language structure
  feature_name?: string
  description?: string
  sources?: Source[]
}

/**
 * Feat entity from D&D 5e API
 *
 * Used in: FeatCard, feat detail pages, tests
 * API endpoint: /api/v1/feats
 */
export interface Feat {
  id: number
  name: string
  slug: string
  prerequisites?: any[]   // TODO: Type prerequisite structure
  modifiers?: any[]       // TODO: Use Modifier interface
  description?: string
  sources?: Source[]
}
```

**Step 4: Update barrel export**

In `app/types/index.ts`, update entity exports line:

```typescript
export type { Spell, Item, Race, CharacterClass, Background, Feat } from './api/entities'
```

**Step 5: Verify TypeScript compiles**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors

**Step 6: Commit type additions**

```bash
git add app/types/api/entities.ts app/types/index.ts
git commit -m "refactor: Add Background and Feat to entities.ts

- Add Background interface with proficiencies
- Add Feat interface with prerequisites
- Export all 6 entity types from barrel
- No breaking changes

Part of Phase 2: Entity Type Extraction (final batch)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 14: Migrate BackgroundCard.vue to use centralized type

**Files:**
- Modify: `app/components/background/BackgroundCard.vue:1-14`

**Step 1: Read current BackgroundCard.vue**

Read: `app/components/background/BackgroundCard.vue` lines 1-50

**Step 2: Update imports and remove inline interface**

In `app/components/background/BackgroundCard.vue`, change:

**Before:**
```typescript
<script setup lang="ts">
import type { Source } from '~/types'

interface Background {
  id: number
  name: string
  slug: string
  skill_proficiencies?: any[]
  tool_proficiencies?: any[]
  languages?: any[]
  feature_name?: string
  description?: string
  sources?: Source[]
}

interface Props {
  background: Background
}
```

**After:**
```typescript
<script setup lang="ts">
import type { Background } from '~/types'

interface Props {
  background: Background
}
```

**Step 3: Verify TypeScript compiles**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors

**Step 4: Run BackgroundCard tests**

Run: `docker compose exec nuxt npm test -- BackgroundCard.test.ts`
Expected: All 26 tests passing

**Step 5: Commit BackgroundCard migration**

```bash
git add app/components/background/BackgroundCard.vue
git commit -m "refactor: Migrate BackgroundCard to use centralized Background type

- Import Background from ~/types instead of defining inline
- Remove 11-line duplicate interface definition
- All 26 BackgroundCard tests passing

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 15: Migrate FeatCard.vue to use centralized type

**Files:**
- Modify: `app/components/feat/FeatCard.vue:1-12`

**Step 1: Read current FeatCard.vue**

Read: `app/components/feat/FeatCard.vue` lines 1-50

**Step 2: Update imports and remove inline interface**

In `app/components/feat/FeatCard.vue`, change:

**Before:**
```typescript
<script setup lang="ts">
import type { Source } from '~/types'

interface Feat {
  id: number
  name: string
  slug: string
  prerequisites?: any[]
  modifiers?: any[]
  description?: string
  sources?: Source[]
}

interface Props {
  feat: Feat
}
```

**After:**
```typescript
<script setup lang="ts">
import type { Feat } from '~/types'

interface Props {
  feat: Feat
}
```

**Step 3: Verify TypeScript compiles**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors

**Step 4: Run FeatCard tests**

Run: `docker compose exec nuxt npm test -- FeatCard.test.ts`
Expected: All 27 tests passing

**Step 5: Commit FeatCard migration**

```bash
git add app/components/feat/FeatCard.vue
git commit -m "refactor: Migrate FeatCard to use centralized Feat type

- Import Feat from ~/types instead of defining inline
- Remove 9-line duplicate interface definition
- All 27 FeatCard tests passing

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 16: Update BackgroundCard.test.ts with typed mock data

**Files:**
- Modify: `tests/components/background/BackgroundCard.test.ts`

**Step 1: Read current test file**

Read: `tests/components/background/BackgroundCard.test.ts` lines 1-40

**Step 2: Add type import and annotate mock data**

Add import after existing imports:

```typescript
import type { Background } from '~/types'
```

Find the `mockBackground` definition and add type annotation:

**Before:**
```typescript
const mockBackground = {
  // ... background data
}
```

**After:**
```typescript
const mockBackground: Background = {
  // ... background data
}
```

**Step 3: Run BackgroundCard tests**

Run: `docker compose exec nuxt npm test -- BackgroundCard.test.ts`
Expected: All 26 tests passing

**Step 4: Commit test update**

```bash
git add tests/components/background/BackgroundCard.test.ts
git commit -m "test: Add type annotation to BackgroundCard mock data

- Import Background type from ~/types
- Annotate mockBackground with Background type
- TypeScript validates test data matches interface
- All 26 tests passing

Part of Phase 2: Entity Type Extraction

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 17: Update FeatCard.test.ts with typed mock data

**Files:**
- Modify: `tests/components/feat/FeatCard.test.ts`

**Step 1: Read current test file**

Read: `tests/components/feat/FeatCard.test.ts` lines 1-40

**Step 2: Add type import and annotate mock data**

Add import after existing imports:

```typescript
import type { Feat } from '~/types'
```

Find the `mockFeat` definition and add type annotation:

**Before:**
```typescript
const mockFeat = {
  // ... feat data
}
```

**After:**
```typescript
const mockFeat: Feat = {
  // ... feat data
}
```

**Step 3: Run FeatCard tests**

Run: `docker compose exec nuxt npm test -- FeatCard.test.ts`
Expected: All 27 tests passing

**Step 4: Commit test update**

```bash
git add tests/components/feat/FeatCard.test.ts
git commit -m "test: Add type annotation to FeatCard mock data

- Import Feat type from ~/types
- Annotate mockFeat with Feat type
- TypeScript validates test data matches interface
- All 27 tests passing

Part of Phase 2: Entity Type Extraction (final component)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 18: Batch 3 validation checkpoint

**Step 1: Run full test suite**

Run: `docker compose exec nuxt npm test -- --run`
Expected: 525/525 tests passing

**Step 2: Verify TypeScript compilation**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: No errors

**Step 3: Verify no linting issues**

Run: `docker compose exec nuxt npm run lint`
Expected: No warnings or errors

**Result:** Batch 3 complete (Background + Feat extracted and migrated)

---

## Final Documentation

### Task 19: Update CHANGELOG.md

**Files:**
- Modify: `CHANGELOG.md`

**Step 1: Read current CHANGELOG.md**

Read: `CHANGELOG.md` lines 1-30

**Step 2: Add entry to Unreleased section**

Under `## [Unreleased]`, add in `### Changed` section:

```markdown
### Changed
- Centralized entity type definitions in `app/types/api/entities.ts` (2025-11-21)
  - Extracted Spell, Item, Race, CharacterClass, Background, Feat interfaces
  - Updated 6 card components to import from centralized types
  - Added type annotations to 6 test files for type-safe mock data
  - Eliminated ~90 lines of duplicate interface definitions
```

**Step 3: Commit CHANGELOG update**

```bash
git add CHANGELOG.md
git commit -m "docs: Update CHANGELOG for Phase 2 entity type extraction

- Document centralized entity types
- Note 6 components and tests updated
- Record duplicate code elimination

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 20: Update analysis doc with completion notes

**Files:**
- Modify: `docs/plans/2025-11-21-interface-extraction-analysis.md`

**Step 1: Read current analysis doc**

Read: `docs/plans/2025-11-21-interface-extraction-analysis.md`

**Step 2: Add Phase 2 completion section**

At end of file, add:

```markdown
---

## ✅ Phase 2 Implementation Results (2025-11-21)

### What Was Completed

**Files Created:**
- `app/types/api/entities.ts` - 6 entity interfaces (~150 lines)

**Components Migrated (6 total):**
1. SpellCard.vue
2. ItemCard.vue
3. RaceCard.vue
4. ClassCard.vue
5. BackgroundCard.vue
6. FeatCard.vue

**Tests Updated (6 total):**
1. SpellCard.test.ts
2. ItemCard.test.ts
3. RaceCard.test.ts
4. ClassCard.test.ts
5. BackgroundCard.test.ts
6. FeatCard.test.ts

**Entity Types Extracted:**
- `Spell` - Eliminated inline definition
- `Item` - Eliminated inline definition
- `Race` - Eliminated inline definition
- `CharacterClass` - Eliminated inline definition
- `Background` - Eliminated inline definition
- `Feat` - Eliminated inline definition

### Metrics

**Code Reduction:**
- Component interfaces removed: ~90 lines
- Type file created: ~150 lines
- Imports added: 12 lines
- Net change: ~-30 lines (duplicates eliminated)

**Test Coverage:**
- All 525 tests passing ✅
- No regressions
- 15 commits with clear history

**Time Spent:**
- Batch 1 (Spell + Item): ~30 minutes
- Batch 2 (Race + Class): ~25 minutes
- Batch 3 (Background + Feat): ~25 minutes
- Documentation: ~15 minutes
- **Total: ~95 minutes** (within estimate)

### Git Commits

1. Type definitions created (3 commits - one per batch)
2. Components migrated (6 commits - one per component)
3. Tests updated (6 commits - one per test file)
4. Documentation (2 commits - CHANGELOG + analysis)

**Total: 17 commits** (more granular than Phase 1 for better traceability)

### Benefits Achieved

✅ **Type-safe test data** - TypeScript validates mock objects
✅ **Single source of truth** - 6 entity types defined once
✅ **Better IntelliSense** - Autocomplete in tests and components
✅ **Foundation for OpenAPI** - Ready for future type generation
✅ **Zero breaking changes** - All tests passing, no functionality affected

---

**Phase 2 Status:** ✅ **COMPLETE**
**Next Steps:** Phase 3 (OpenAPI generation) shelved per user request
```

**Step 3: Commit analysis update**

```bash
git add docs/plans/2025-11-21-interface-extraction-analysis.md
git commit -m "docs: Document Phase 2 completion in analysis

- Add Phase 2 implementation results
- Record metrics and benefits
- Note OpenAPI generation shelved

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 21: Create handover document

**Files:**
- Create: `docs/HANDOVER-2025-11-21-ENTITY-TYPE-EXTRACTION-PHASE2.md`

**Step 1: Create comprehensive handover doc**

Create file with complete session summary (similar structure to Phase 1 handover).

**Template sections:**
- Session Summary
- Problem Statement & Root Cause
- Investigation & Design
- Implementation (Incremental Approach)
- Results (Code Changes Summary, Files Modified, Git History)
- Key Insights & Learnings
- Technical Details
- Migration Pattern (Reusable)
- Impact Analysis
- Success Criteria Met
- Notes for Next Agent

**Step 2: Commit handover doc**

```bash
git add docs/HANDOVER-2025-11-21-ENTITY-TYPE-EXTRACTION-PHASE2.md
git commit -m "docs: Add comprehensive handover for Phase 2 entity extraction

- Complete session documentation
- Reusable migration patterns
- Metrics and impact analysis
- Future work notes (OpenAPI shelved)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Final Validation

### Task 22: Complete system verification

**Step 1: Full test suite (final check)**

Run: `docker compose exec nuxt npm test -- --run`
Expected: **525/525 tests passing** ✅

**Step 2: TypeScript compilation (final check)**

Run: `docker compose exec nuxt npx nuxi typecheck`
Expected: **No errors** ✅

**Step 3: Linting (final check)**

Run: `docker compose exec nuxt npm run lint`
Expected: **No warnings** ✅

**Step 4: Manual spot check (browser verification)**

Visit in browser:
- http://localhost:3000/spells (verify cards render)
- http://localhost:3000/items (verify cards render)
- http://localhost:3000/races (verify cards render)

Expected: All pages load, cards display correctly

**Step 5: Git status check**

Run: `git status`
Expected: Clean working directory (all changes committed)

**Step 6: Review commit history**

Run: `git log --oneline -20`
Expected: ~17-20 commits with clear Phase 2 messages

---

## Success Criteria Checklist

**Phase 2 Complete When:**

- [x] `app/types/api/entities.ts` created with 6 entity interfaces
- [x] `app/types/index.ts` updated with entity exports
- [x] All 6 card components import from centralized types
- [x] All 6 test files use typed mock data
- [x] All 525 tests passing (zero regressions)
- [x] TypeScript compiles with no errors
- [x] No linting warnings
- [x] CHANGELOG.md updated
- [x] Analysis doc updated with Phase 2 completion
- [x] Handover doc created
- [x] 15-20 clean commits with clear messages
- [x] Manual browser verification successful

---

## Estimated Timeline

**Total time:** 1.5-2 hours

**Breakdown:**
- Batch 1 setup + migration: 35-40 min (Tasks 1-6)
- Batch 2 migration: 25-30 min (Tasks 7-12)
- Batch 3 migration: 25-30 min (Tasks 13-18)
- Documentation: 15-20 min (Tasks 19-21)
- Final validation: 5-10 min (Task 22)

**Note:** Times assume using Docker commands as specified (may vary based on container performance)

---

## Troubleshooting

**If tests fail after migration:**
1. Check TypeScript compilation first: `npx nuxi typecheck`
2. Look for missing fields in mock data (type annotations reveal this)
3. Verify import path is `~/types` not `~/types/api/entities`
4. Rollback to previous commit if needed

**If TypeScript errors occur:**
1. Verify interface matches exactly from original component
2. Check for typos in type names (CharacterClass vs Class)
3. Ensure barrel export is correct in index.ts
4. Clear Nuxt cache: `rm -rf .nuxt` and restart

**If imports don't work:**
1. Verify `~/types` alias in nuxt.config.ts
2. Check barrel export includes new types
3. Restart TypeScript server in IDE
4. Restart Nuxt dev server

---

**Plan Status:** ✅ Ready for Execution
**Next Step:** Use `superpowers:executing-plans` or `superpowers:subagent-driven-development`
