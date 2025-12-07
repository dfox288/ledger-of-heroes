# Multiclass API Migration Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Migrate character builder from legacy `PATCH /characters/{id}` with `class_id` to new `/characters/{id}/classes` endpoints.

**Architecture:** Replace single `classId` ref with `characterClasses` array. Add backwards-compatible computed properties (`classId`, `selectedClass`) so UI code changes are minimal. Create Nitro server routes for new endpoints.

**Tech Stack:** Vue 3, Pinia, Nuxt 4, TypeScript, Vitest

---

## Task 1: Create Nitro Server Routes for Class Endpoints

**Files:**
- Create: `server/api/characters/[id]/classes/index.get.ts`
- Create: `server/api/characters/[id]/classes/index.post.ts`
- Create: `server/api/characters/[id]/classes/[classId].delete.ts`
- Create: `server/api/characters/[id]/classes/[classId]/subclass.put.ts`

**Step 1: Create GET /classes route**

```typescript
// server/api/characters/[id]/classes/index.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/classes`)
  return data
})
```

**Step 2: Create POST /classes route**

```typescript
// server/api/characters/[id]/classes/index.post.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const body = await readBody(event)

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/classes`, {
    method: 'POST',
    body
  })
  return data
})
```

**Step 3: Create DELETE /classes/:classId route**

```typescript
// server/api/characters/[id]/classes/[classId].delete.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const classId = getRouterParam(event, 'classId')

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/classes/${classId}`, {
    method: 'DELETE'
  })
  return data
})
```

**Step 4: Create PUT /classes/:classId/subclass route**

```bash
mkdir -p server/api/characters/\[id\]/classes/\[classId\]
```

```typescript
// server/api/characters/[id]/classes/[classId]/subclass.put.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const classId = getRouterParam(event, 'classId')
  const body = await readBody(event)

  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/classes/${classId}/subclass`, {
    method: 'PUT',
    body
  })
  return data
})
```

**Step 5: Commit**

```bash
git add server/api/characters/\[id\]/classes/
git commit -m "feat: add Nitro server routes for character class endpoints (#92)"
```

---

## Task 2: Add CharacterClassEntry Type

**Files:**
- Modify: `app/types/character.ts`

**Step 1: Write the failing test**

```typescript
// tests/types/character.test.ts (create file)
import { describe, it, expectTypeOf } from 'vitest'
import type { CharacterClassEntry } from '~/types/character'

describe('CharacterClassEntry type', () => {
  it('has required properties', () => {
    const entry: CharacterClassEntry = {
      classId: 1,
      subclassId: null,
      level: 1,
      isPrimary: true,
      order: 0,
      classData: null
    }
    expectTypeOf(entry.classId).toBeNumber()
    expectTypeOf(entry.subclassId).toEqualTypeOf<number | null>()
    expectTypeOf(entry.level).toBeNumber()
    expectTypeOf(entry.isPrimary).toBeBoolean()
    expectTypeOf(entry.order).toBeNumber()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- --run tests/types/character.test.ts`
Expected: FAIL - type `CharacterClassEntry` not found

**Step 3: Add CharacterClassEntry type**

Add to `app/types/character.ts` after AbilityScores interface:

```typescript
import type { CharacterClass, Subclass } from './api/entities'

/**
 * Character class entry for multiclass support
 * Stores class info and cached full data for UI
 */
export interface CharacterClassEntry {
  classId: number
  subclassId: number | null
  level: number
  isPrimary: boolean
  order: number
  /** Cached full class data for UI display */
  classData: CharacterClass | null
  /** Cached subclass data if selected */
  subclassData?: Subclass | null
}
```

**Step 4: Export from types/index.ts**

Add to exports in `app/types/index.ts`:

```typescript
export type { CharacterClassEntry } from './character'
```

**Step 5: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- --run tests/types/character.test.ts`
Expected: PASS

**Step 6: Commit**

```bash
git add app/types/character.ts app/types/index.ts tests/types/character.test.ts
git commit -m "feat: add CharacterClassEntry type for multiclass support (#92)"
```

---

## Task 3: Update Store State - Add characterClasses Array

**Files:**
- Modify: `app/stores/characterBuilder.ts:33-75`
- Test: `tests/stores/characterBuilder.test.ts`

**Step 1: Write failing tests for new state**

Add to `tests/stores/characterBuilder.test.ts` in the `describe('initial state')` block:

```typescript
    it('has empty characterClasses array initially', () => {
      const store = useCharacterBuilderStore()
      expect(store.characterClasses).toEqual([])
    })

    it('primaryClass returns null when no classes', () => {
      const store = useCharacterBuilderStore()
      expect(store.primaryClass).toBeNull()
    })

    it('classId computed returns null when no classes', () => {
      const store = useCharacterBuilderStore()
      expect(store.classId).toBeNull()
    })

    it('selectedClass computed returns null when no classes', () => {
      const store = useCharacterBuilderStore()
      expect(store.selectedClass).toBeNull()
    })
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- --run tests/stores/characterBuilder.test.ts`
Expected: FAIL - characterClasses property doesn't exist

**Step 3: Add characterClasses state and computed properties**

In `app/stores/characterBuilder.ts`, replace lines 37 and 72-74:

```typescript
// OLD (line 37):
// const classId = ref<number | null>(null)

// NEW - Replace with:
import type { CharacterClassEntry } from '~/types'

// Array of character classes (supports multiclass, but level 1 uses just one)
const characterClasses = ref<CharacterClassEntry[]>([])

// OLD (lines 72-74):
// const selectedClass = ref<CharacterClass | null>(null)

// NEW - Replace selectedClass ref with computed:
// (keep the ref for now, we'll remove it in Task 5)
```

Add computed properties after line 74:

```typescript
// ══════════════════════════════════════════════════════════════
// MULTICLASS COMPUTED PROPERTIES
// ══════════════════════════════════════════════════════════════

// Primary class (first/only class for level 1 characters)
const primaryClass = computed(() =>
  characterClasses.value.find(c => c.isPrimary) ?? null
)

// Backwards-compatible classId (for existing code)
const classId = computed(() => primaryClass.value?.classId ?? null)

// Backwards-compatible selectedClass (for existing code)
const selectedClass = computed(() => primaryClass.value?.classData ?? null)
```

**Step 4: Update return statement**

Add `characterClasses` and `primaryClass` to the return statement.

**Step 5: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- --run tests/stores/characterBuilder.test.ts`
Expected: Some tests fail (selectClass tests expect different API calls)

**Step 6: Commit work-in-progress**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "wip: add characterClasses array and computed properties (#92)"
```

---

## Task 4: Update selectClass Action to Use New API

**Files:**
- Modify: `app/stores/characterBuilder.ts:358-384` (selectClass action)
- Modify: `tests/stores/characterBuilder.test.ts:313-400` (selectClass tests)

**Step 1: Update selectClass tests to expect new API calls**

Replace the selectClass test block starting at line 332:

```typescript
    it('calls POST /classes endpoint and fetches full detail', async () => {
      // Mock POST and GET calls
      mockApiFetch
        .mockResolvedValueOnce({ data: {} }) // POST /classes
        .mockResolvedValueOnce({ data: mockClass }) // GET /classes/{slug}
        .mockResolvedValueOnce({ data: {} }) // refreshStats

      const store = useCharacterBuilderStore()
      store.characterId = 42

      await store.selectClass(mockClass)

      expect(mockApiFetch).toHaveBeenCalledWith('/characters/42/classes', {
        method: 'POST',
        body: { class_id: 1 }
      })
      expect(mockApiFetch).toHaveBeenCalledWith('/classes/fighter')
    })

    it('clears existing classes before adding new one', async () => {
      mockApiFetch
        .mockResolvedValueOnce({ data: {} }) // DELETE existing class
        .mockResolvedValueOnce({ data: {} }) // POST new class
        .mockResolvedValueOnce({ data: mockClass }) // GET detail
        .mockResolvedValueOnce({ data: {} }) // refreshStats

      const store = useCharacterBuilderStore()
      store.characterId = 42
      // Simulate existing class
      store.characterClasses = [{
        classId: 99,
        subclassId: null,
        level: 1,
        isPrimary: true,
        order: 0,
        classData: null
      }]

      await store.selectClass(mockClass)

      expect(mockApiFetch).toHaveBeenCalledWith('/characters/42/classes/99', {
        method: 'DELETE'
      })
    })

    it('updates characterClasses array with new entry', async () => {
      const fullClassDetail = { ...mockClass, equipment: [{ id: 1, item: { name: 'Sword' } }] }
      mockApiFetch
        .mockResolvedValueOnce({ data: {} }) // POST
        .mockResolvedValueOnce({ data: fullClassDetail }) // GET detail
        .mockResolvedValueOnce({ data: {} }) // refreshStats

      const store = useCharacterBuilderStore()
      store.characterId = 42

      await store.selectClass(mockClass)

      expect(store.characterClasses).toHaveLength(1)
      expect(store.characterClasses[0]).toEqual({
        classId: 1,
        subclassId: null,
        level: 1,
        isPrimary: true,
        order: 0,
        classData: fullClassDetail
      })
    })

    it('classId computed returns class ID after selection', async () => {
      mockApiFetch
        .mockResolvedValueOnce({ data: {} })
        .mockResolvedValueOnce({ data: mockClass })
        .mockResolvedValueOnce({ data: {} })

      const store = useCharacterBuilderStore()
      store.characterId = 42

      await store.selectClass(mockClass)

      expect(store.classId).toBe(1)
    })

    it('selectedClass computed returns full class data', async () => {
      const fullClassDetail = { ...mockClass, equipment: [] }
      mockApiFetch
        .mockResolvedValueOnce({ data: {} })
        .mockResolvedValueOnce({ data: fullClassDetail })
        .mockResolvedValueOnce({ data: {} })

      const store = useCharacterBuilderStore()
      store.characterId = 42

      await store.selectClass(mockClass)

      expect(store.selectedClass).toEqual(fullClassDetail)
    })
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- --run tests/stores/characterBuilder.test.ts`
Expected: FAIL - API calls don't match

**Step 3: Update selectClass action implementation**

Replace the selectClass action in `app/stores/characterBuilder.ts`:

```typescript
  /**
   * Step 3: Select class
   * Uses new /classes endpoints for multiclass support
   * Clears existing classes first (level 1 characters have exactly one class)
   */
  async function selectClass(cls: CharacterClass): Promise<void> {
    isLoading.value = true
    error.value = null

    try {
      // Clear existing classes first (for re-selection)
      for (const entry of characterClasses.value) {
        await apiFetch(`/characters/${characterId.value}/classes/${entry.classId}`, {
          method: 'DELETE'
        })
      }

      // Add the new class
      await apiFetch(`/characters/${characterId.value}/classes`, {
        method: 'POST',
        body: { class_id: cls.id }
      })

      // Fetch full class detail to get equipment data
      const fullClass = await apiFetch<{ data: CharacterClass }>(`/classes/${cls.slug}`)

      // Update characterClasses array
      characterClasses.value = [{
        classId: cls.id,
        subclassId: null,
        level: 1,
        isPrimary: true,
        order: 0,
        classData: fullClass.data
      }]

      await refreshStats()
    } catch (err: unknown) {
      error.value = 'Failed to save class'
      throw err
    } finally {
      isLoading.value = false
    }
  }
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- --run tests/stores/characterBuilder.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "feat: update selectClass to use new /classes endpoint (#92)"
```

---

## Task 5: Update loadCharacterForEditing for classes Array

**Files:**
- Modify: `app/stores/characterBuilder.ts` (loadCharacterForEditing action)
- Modify: `tests/stores/characterBuilder.test.ts`

**Step 1: Write failing test for edit mode**

Add to the `loadCharacterForEditing` test section:

```typescript
    it('loads character classes from classes array', async () => {
      const mockCharacter = {
        id: 42,
        name: 'Test Character',
        level: 1,
        ability_scores: { STR: 10, DEX: 12, CON: 14, INT: 8, WIS: 10, CHA: 15 },
        race: { id: 1, name: 'Human', slug: 'human' },
        class: { id: 2, name: 'Fighter', slug: 'fighter' }, // Legacy field
        classes: [{
          class: { id: 2, name: 'Fighter', slug: 'fighter' },
          subclass: null,
          level: 1,
          is_primary: true,
          order: 0,
          hit_dice: { die: 'd10', max: 1, spent: 0, available: 1 }
        }],
        background: null
      }

      const mockRace = { id: 1, name: 'Human', slug: 'human' }
      const mockClass = { id: 2, name: 'Fighter', slug: 'fighter', hit_die: 10 }

      mockApiFetch
        .mockResolvedValueOnce({ data: mockCharacter }) // GET /characters/42
        .mockResolvedValueOnce({ data: mockRace }) // GET /races/human
        .mockResolvedValueOnce({ data: mockClass }) // GET /classes/fighter

      const store = useCharacterBuilderStore()
      await store.loadCharacterForEditing(42)

      expect(store.characterClasses).toHaveLength(1)
      expect(store.characterClasses[0].classId).toBe(2)
      expect(store.characterClasses[0].isPrimary).toBe(true)
      expect(store.characterClasses[0].classData).toEqual(mockClass)
    })
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- --run tests/stores/characterBuilder.test.ts`
Expected: FAIL

**Step 3: Update loadCharacterForEditing**

Update the class loading section (around line 877):

```typescript
      // Load classes from character.classes array
      if (character.classes && character.classes.length > 0) {
        characterClasses.value = await Promise.all(
          character.classes.map(async (pivot) => {
            const fullClass = await apiFetch<{ data: CharacterClass }>(`/classes/${pivot.class.slug}`)
            return {
              classId: pivot.class.id,
              subclassId: pivot.subclass?.id ?? null,
              level: typeof pivot.level === 'string' ? parseInt(pivot.level, 10) : pivot.level,
              isPrimary: pivot.is_primary === true || pivot.is_primary === 'true' || pivot.is_primary === '1',
              order: typeof pivot.order === 'string' ? parseInt(pivot.order, 10) : pivot.order,
              classData: fullClass.data
            }
          })
        )
      }
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- --run tests/stores/characterBuilder.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "feat: update loadCharacterForEditing for classes array (#92)"
```

---

## Task 6: Update reset() to Clear characterClasses

**Files:**
- Modify: `app/stores/characterBuilder.ts` (reset action)
- Modify: `tests/stores/characterBuilder.test.ts`

**Step 1: Write failing test**

Add to reset tests:

```typescript
    it('clears characterClasses array', () => {
      const store = useCharacterBuilderStore()
      store.characterClasses = [{
        classId: 1,
        subclassId: null,
        level: 1,
        isPrimary: true,
        order: 0,
        classData: null
      }]

      store.reset()

      expect(store.characterClasses).toEqual([])
    })
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- --run tests/stores/characterBuilder.test.ts`
Expected: FAIL

**Step 3: Update reset() action**

Add to reset() function:

```typescript
    characterClasses.value = []
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- --run tests/stores/characterBuilder.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "feat: clear characterClasses in reset() (#92)"
```

---

## Task 7: Run Full Test Suite and Fix Any Regressions

**Step 1: Run full test suite**

Run: `docker compose exec nuxt npm run test -- --run`
Expected: All tests pass

**Step 2: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

**Step 3: Fix any failing tests**

If any tests fail, update them to use the new computed `classId` and `selectedClass` instead of expecting refs.

**Step 4: Run lint**

Run: `docker compose exec nuxt npm run lint`
Expected: No new errors

**Step 5: Commit any fixes**

```bash
git add -A
git commit -m "fix: resolve test regressions from multiclass migration (#92)"
```

---

## Task 8: Manual Browser Testing

**Step 1: Start dev server**

Run: `docker compose exec nuxt npm run dev`

**Step 2: Test character creation**

1. Go to character creator
2. Enter name, proceed
3. Select race, proceed
4. Select class (Fighter), proceed
5. Verify class shows in review step

**Step 3: Test class re-selection**

1. Go back to class step
2. Select different class (Wizard)
3. Proceed to review
4. Verify new class shows

**Step 4: Test edit mode**

1. Create a complete character
2. Edit character
3. Verify class loaded correctly

**Step 5: Document any issues found**

---

## Task 9: Create PR

**Step 1: Push branch**

```bash
git push -u origin feature/issue-92-multiclass-api-migration
```

**Step 2: Create PR**

```bash
gh pr create --title "feat: migrate character builder to multiclass API endpoints (#92)" --body "$(cat <<'EOF'
## Summary

- Migrate character builder from `PATCH /characters/{id}` with `class_id` to new `/characters/{id}/classes` endpoints
- Replace single `classId` ref with `characterClasses` array for future multiclass support
- Add backwards-compatible computed properties so UI code changes are minimal

### Changes

**Nitro Routes:**
- `GET /api/characters/:id/classes` - List character classes
- `POST /api/characters/:id/classes` - Add class
- `DELETE /api/characters/:id/classes/:classId` - Remove class
- `PUT /api/characters/:id/classes/:classId/subclass` - Set subclass

**Store:**
- Add `characterClasses` array state
- Add `primaryClass` computed
- Update `classId` and `selectedClass` to computed properties
- Update `selectClass` to use POST endpoint
- Update `loadCharacterForEditing` for classes array
- Update `reset()` to clear classes

## Test plan

- [x] All unit tests pass
- [x] TypeScript compiles
- [ ] Manual test: Create character with class selection
- [ ] Manual test: Re-select class during creation
- [ ] Manual test: Edit existing character

Relates to backend #92

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

---

## Summary

| Task | Description | Estimated Time |
|------|-------------|----------------|
| 1 | Create Nitro server routes | 5 min |
| 2 | Add CharacterClassEntry type | 5 min |
| 3 | Add characterClasses state | 10 min |
| 4 | Update selectClass action | 15 min |
| 5 | Update loadCharacterForEditing | 10 min |
| 6 | Update reset() | 5 min |
| 7 | Full test suite + fixes | 10 min |
| 8 | Manual browser testing | 10 min |
| 9 | Create PR | 5 min |

**Total: ~75 minutes**
