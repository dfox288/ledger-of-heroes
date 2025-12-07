# Design: Edit Level 1 Characters

**Issue:** #105
**Date:** 2025-12-03
**Status:** Approved

## Overview

Allow users to edit characters that are still at level 1, enabling them to fix mistakes or change choices before advancing.

## Architecture Decision

**Approach:** Unified edit page with create-as-redirect

When creating a new character:
1. POST to `/api/characters` creates empty character in database
2. Redirect to `/characters/[id]/edit`
3. User fills wizard steps (same flow as editing)

This means:
- Single wizard page (`edit.vue`) handles both new and existing characters
- Character always exists in database (no "draft in memory" state)
- URL is shareable/resumable
- Simpler store logic - always editing, never creating

## Constraints

- Only level 1 characters can be edited
- Changing class shows warning about equipment/spell reset
- Equipment choices are re-selected on edit (no recovery from API)

## Implementation

### 1. Store Changes (`stores/characterBuilder.ts`)

**New actions:**

```typescript
// Load existing character for editing
async function loadCharacterForEditing(id: number): Promise<void>

// Update character name (for edit mode)
async function updateName(newName: string): Promise<void>

// Determine starting step based on completion
function determineStartingStep(character: Character): number
```

**loadCharacterForEditing flow:**
1. Fetch character from `GET /characters/:id`
2. Check `level === 1` (throw if not)
3. Populate store fields from character data
4. Map ability scores from API format (STR) to store format (strength)
5. Fetch full reference data (race, class, background) via slug
6. Handle subrace detection via `parent_race`
7. Fetch spells if caster class
8. Set `currentStep` based on what's filled

### 2. New Page (`pages/characters/[id]/edit.vue`)

The main wizard page:
- Calls `store.loadCharacterForEditing(id)` on mount
- Shows loading state while fetching
- Redirects to detail page if level > 1
- Contains stepper, step components, navigation (same UI as current create.vue)

### 3. Modified Page (`pages/characters/create.vue`)

Becomes a simple redirect:
- POST `/api/characters` with `{ name: '' }`
- `navigateTo(`/characters/${id}/edit`)`
- Shows spinner while creating

### 4. Step Component Changes

**StepName.vue:**
- If `store.characterId` exists, call `store.updateName()` instead of `createDraft()`

**StepClass.vue:**
- If class is changing during edit, show confirmation dialog
- Warn that equipment and spells will be reset
- Clear `equipmentChoices`, `equipmentItemSelections`, `selectedSpells` on confirm

### 5. Character Card Changes

Add Edit/Continue button:
- Show only when `character.level === 1`
- Label: "Continue" if incomplete, "Edit" if complete
- Links to `/characters/[id]/edit`

## File Changes Summary

| File | Change |
|------|--------|
| `stores/characterBuilder.ts` | Add `loadCharacterForEditing()`, `updateName()`, `determineStartingStep()` |
| `pages/characters/create.vue` | Simplify to POST + redirect |
| `pages/characters/[id]/edit.vue` | **New file** - the wizard page |
| `components/character/builder/StepName.vue` | Check edit mode, call `updateName()` |
| `components/character/builder/StepClass.vue` | Add class change warning dialog |
| `components/character/CharacterCard.vue` | Add Edit/Continue button for level 1 |

## Data Mapping

API ability scores to store format:
```
STR → strength
DEX → dexterity
CON → constitution
INT → intelligence
WIS → wisdom
CHA → charisma
```

## Testing Strategy

1. **Store tests:** `loadCharacterForEditing()` with various character states
2. **Page tests:** Edit page loads character, redirects if level > 1
3. **Component tests:** StepName handles edit mode, StepClass shows warning
4. **Integration:** Full edit flow from card click to save

## Out of Scope

- Equipment choice recovery (user re-selects)
- Editing characters above level 1
- Batch editing multiple characters
