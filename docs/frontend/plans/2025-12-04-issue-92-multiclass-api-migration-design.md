# Multiclass API Migration Design

**Issue:** #92 (Backend multiclass support)
**Date:** 2025-12-04
**Status:** Design Complete

## Overview

Migrate the character builder to use the new `/characters/{id}/classes` endpoints instead of the legacy `PATCH /characters/{id}` with `class_id`. This prepares the frontend for future multiclass support while maintaining the current single-class level 1 wizard flow.

## Goals

1. **Use new API endpoints** for class management
2. **Future-proof store structure** with array-based class storage
3. **Maintain backwards compatibility** in UI through computed properties
4. **Add Nitro server routes** for new endpoints

## Non-Goals

- Multiclass UI in the character creator (level 1 only)
- Level-up workflow (future feature)
- Multiple class selection in wizard

## API Endpoints

### New Endpoints (Backend #92)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/characters/{id}/classes` | List all classes for character |
| POST | `/characters/{id}/classes` | Add a class (`{ class_id, force? }`) |
| DELETE | `/characters/{id}/classes/{classId}` | Remove a class |
| POST | `/characters/{id}/classes/{classId}/level-up` | Level up specific class |
| PUT | `/characters/{id}/classes/{classId}/subclass` | Set subclass (`{ subclass_id }`) |

### Response: CharacterClassPivotResource

```typescript
interface CharacterClassPivotResource {
  class: { id: number; name: string; slug: string }
  subclass: { id: number; name: string; slug: string } | null
  level: number
  is_primary: boolean
  order: number
  hit_dice: {
    die: string
    max: number
    spent: number
    available: number
  }
}
```

## Store Design

### New State Structure

```typescript
// New interface for class entries
interface CharacterClassEntry {
  classId: number
  subclassId: number | null
  level: number
  isPrimary: boolean
  order: number
  // Cached full data for UI
  classData: CharacterClass | null
}

// Replace single refs with array
const characterClasses = ref<CharacterClassEntry[]>([])
```

### Computed Properties (Backwards Compatible)

```typescript
// Primary class helper
const primaryClass = computed(() =>
  characterClasses.value.find(c => c.isPrimary) ?? null
)

// Legacy compatibility
const classId = computed(() => primaryClass.value?.classId ?? null)
const selectedClass = computed(() => primaryClass.value?.classData ?? null)
```

### Actions

```typescript
// Add primary class (level 1 character creation)
async function selectClass(cls: CharacterClass): Promise<void> {
  // Clear existing classes first (for re-selection)
  if (characterClasses.value.length > 0) {
    for (const entry of characterClasses.value) {
      await apiFetch(`/characters/${characterId.value}/classes/${entry.classId}`, {
        method: 'DELETE'
      })
    }
  }

  // Add new class
  await apiFetch(`/characters/${characterId.value}/classes`, {
    method: 'POST',
    body: { class_id: cls.id }
  })

  // Update local state
  characterClasses.value = [{
    classId: cls.id,
    subclassId: null,
    level: 1,
    isPrimary: true,
    order: 0,
    classData: cls
  }]
}

// Set subclass for primary class
async function selectSubclass(subclass: Subclass): Promise<void> {
  const primary = primaryClass.value
  if (!primary) return

  await apiFetch(`/characters/${characterId.value}/classes/${primary.classId}/subclass`, {
    method: 'PUT',
    body: { subclass_id: subclass.id }
  })

  primary.subclassId = subclass.id
}
```

## Nitro Server Routes

### New Routes Needed

```
server/api/characters/[id]/classes/
├── index.get.ts          # GET /api/characters/:id/classes
├── index.post.ts         # POST /api/characters/:id/classes
├── [classId].delete.ts   # DELETE /api/characters/:id/classes/:classId
└── [classId]/
    ├── level-up.post.ts  # POST (future use)
    └── subclass.put.ts   # PUT /api/characters/:id/classes/:classId/subclass
```

### Route Template

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

## Component Updates

### Minimal Changes Required

| Component | Change |
|-----------|--------|
| `StepClass.vue` | Call `selectClass()` - same interface |
| `StepSpells.vue` | Use `selectedClass` computed - unchanged |
| `StepEquipment.vue` | Use `selectedClass.equipment` - unchanged |
| `StepReview.vue` | Use `selectedClass` - unchanged |
| `isCaster` computed | Use `selectedClass` - unchanged |

The backwards-compatible computed properties mean most UI code remains unchanged.

### Edit Mode

```typescript
async function loadCharacterForEditing(id: number): Promise<void> {
  const response = await apiFetch<{ data: Character }>(`/characters/${id}`)
  const character = response.data

  // Load from character.classes array
  characterClasses.value = await Promise.all(
    character.classes.map(async (pivot) => {
      const fullClass = await apiFetch<{ data: CharacterClass }>(
        `/classes/${pivot.class.slug}`
      )
      return {
        classId: pivot.class.id,
        subclassId: pivot.subclass?.id ?? null,
        level: pivot.level,
        isPrimary: pivot.is_primary,
        order: pivot.order,
        classData: fullClass.data
      }
    })
  )
}
```

## Testing Strategy

1. **Unit tests** for new store structure and computed properties
2. **Update existing tests** that mock `classId` to use `characterClasses`
3. **Integration tests** for new Nitro routes
4. **Manual testing** of wizard flow

## Migration Checklist

- [ ] Create Nitro server routes for new endpoints
- [ ] Update store with `characterClasses` array
- [ ] Add backwards-compatible computed properties
- [ ] Update `selectClass` action to use POST endpoint
- [ ] Update `loadCharacterForEditing` for classes array
- [ ] Update `reset()` to clear `characterClasses`
- [ ] Update tests
- [ ] Manual browser testing

## Future Considerations

When implementing multiclass UI (level 2+):
- Add `addClass()` action for additional classes
- Add `removeClass()` action
- Add class order management
- Update hit dice display for multiple pools
