# Migration Plan: Unified Character Choice System

**Issue:** #246
**Date:** 2025-12-06
**Status:** READY TO IMPLEMENT

**Backend Status:** Endpoints available now, no transition period needed

## Executive Summary

The backend has implemented a **Unified Choice System** that consolidates all character creation choices (proficiencies, languages, equipment, spells, subclass, ASI/feats, etc.) into a single API pattern. This replaces our current fragmented approach with separate endpoints for each choice type.

### Key Benefits
- **Single endpoint** for all pending choices: `GET /characters/{id}/pending-choices`
- **Unified resolution**: `POST /characters/{id}/choices/{choiceId}`
- **Consistent data structure** across all choice types
- **Automatic choice generation** when class/race/background changes
- **Built-in progress tracking** via `summary` object

### Breaking Changes
The following endpoints are being deprecated:
- `GET/POST /characters/{id}/proficiency-choices`
- `GET/POST /characters/{id}/language-choices`
- `POST /characters/{id}/spells` (for creation - replaced by choice resolution)
- `POST /characters/{id}/equipment` (for creation - replaced by choice resolution)

---

## Current vs New Architecture

### Current Architecture (Fragmented)

```
Wizard Step          API Calls
─────────────────────────────────────────────────────
Race                 POST /characters (create)
                     PATCH /characters/{id} (update race)
Subrace              PATCH /characters/{id}
Class                POST/PUT /characters/{id}/classes
Subclass             PUT /characters/{id}/classes/{classId}/subclass
Background           PATCH /characters/{id}
Abilities            PATCH /characters/{id}
Proficiencies        GET /proficiency-choices
                     POST /proficiency-choices (per source:group)
Languages            GET /language-choices
                     POST /language-choices (per source)
Equipment            DELETE /equipment/* (clear all)
                     POST /equipment (per item - loop)
Spells               GET /available-spells
                     POST /spells (batch)
Details              PATCH /characters/{id}
```

### New Architecture (Unified)

```
Wizard Step          API Calls
─────────────────────────────────────────────────────
Race                 POST /characters (create)
                     PATCH /characters/{id}
Subrace              PATCH /characters/{id}
Class                POST/PUT /characters/{id}/classes
Subclass             POST /choices/{choiceId}              ← NEW
Background           PATCH /characters/{id}
Abilities            PATCH /characters/{id}
Choices (unified)    GET /pending-choices                  ← NEW
                     POST /choices/{choiceId}              ← NEW (per choice)
Details              PATCH /characters/{id}

The new pending-choices endpoint returns ALL pending choices:
- proficiency choices (skill, tool, weapon, armor)
- language choices
- equipment choices
- spell choices (cantrips + spells known)
- subclass choice (for L1 subclass classes)
- ASI/feat choices (at appropriate levels)
- optional feature choices (invocations, metamagic, etc.)
- hit point choices (at level-up)
```

---

## New API Structure

### GET /characters/{id}/pending-choices

Returns all unresolved choices with consistent structure:

```typescript
interface PendingChoicesResponse {
  data: {
    choices: PendingChoice[];
    summary: {
      total_pending: number;
      required_pending: number;
      optional_pending: number;
      by_type: Record<string, number>;
      by_source: Record<string, number>;
    };
  };
}

interface PendingChoice {
  id: string;                    // e.g., "proficiency:class:5:1:skill_choice_1"
  type: ChoiceType;              // proficiency | language | equipment | spell | subclass | asi_or_feat | optional_feature | expertise | fighting_style | hit_points
  subtype: string | null;        // skill | tool | cantrip | spells_known | invocation | etc.
  source: ChoiceSource;          // class | race | background | feat
  source_name: string;           // "Rogue", "High Elf", "Acolyte"
  level_granted: number;         // Character level when choice became available
  required: boolean;             // Blocks completion if unresolved
  quantity: number;              // How many selections needed
  remaining: number;             // quantity - already selected
  selected: (string | number)[]; // Already chosen option IDs/slugs
  options: ChoiceOption[] | null;// Available options (null if external endpoint)
  options_endpoint: string | null;// URL for dynamic options (e.g., available-spells)
  metadata: Record<string, unknown>; // Type-specific extra data
}

type ChoiceType =
  | 'proficiency'
  | 'language'
  | 'equipment'
  | 'spell'
  | 'subclass'
  | 'asi_or_feat'
  | 'optional_feature'
  | 'expertise'
  | 'fighting_style'
  | 'hit_points';

type ChoiceSource = 'class' | 'race' | 'background' | 'feat';
```

### POST /characters/{id}/choices/{choiceId}

Resolves a choice with selected options:

```typescript
// Standard resolution (most choice types)
{
  selected: (string | number)[]  // IDs or slugs of selected options
}

// Equipment resolution (when option has sub-choices)
{
  selected: "a",                 // Option letter
  item_selections?: {            // For "any martial weapon" type choices
    "martial_weapon_1": 456      // Item ID
  }
}

// ASI/Feat resolution
{
  type: "asi" | "feat",
  selected?: number,             // Feat ID (if type=feat)
  increases?: Record<string, number> // Ability increases (if type=asi)
}

// Spell swap resolution (level-up)
{
  selected: [789],
  swap?: {
    remove: 123,
    add: 790
  }
}
```

### DELETE /characters/{id}/choices/{choiceId}

Undoes a resolved choice (if allowed).

---

## Migration Strategy

### Phase 1: Types & Infrastructure

1. **Sync API types** from backend
   ```bash
   npm run types:sync
   ```

2. **Create new TypeScript interfaces**
   - `PendingChoice` interface
   - `ChoiceType` union type
   - `ResolveChoicePayload` interface variants
   - Update `CharacterSummaryData` to match new format

3. **Create Nitro server routes**
   ```
   server/api/characters/[id]/pending-choices.get.ts
   server/api/characters/[id]/choices/[choiceId].post.ts
   server/api/characters/[id]/choices/[choiceId].delete.ts
   ```

### Phase 2: New Composable

Create `useUnifiedChoices` composable:

```typescript
// app/composables/useUnifiedChoices.ts
export function useUnifiedChoices(characterId: Ref<number | null>) {
  const { apiFetch } = useApi()

  // Fetch all pending choices
  const { data: pendingChoices, refresh } = useLazyAsyncData(
    () => characterId.value
      ? apiFetch<PendingChoicesResponse>(`/characters/${characterId.value}/pending-choices`)
      : null
  )

  // Filter choices by type
  const choicesByType = computed(() => {
    const choices = pendingChoices.value?.data.choices ?? []
    return {
      proficiencies: choices.filter(c => c.type === 'proficiency'),
      languages: choices.filter(c => c.type === 'language'),
      equipment: choices.filter(c => c.type === 'equipment'),
      spells: choices.filter(c => c.type === 'spell'),
      subclass: choices.find(c => c.type === 'subclass'),
      // ... etc
    }
  })

  // Resolve a choice
  async function resolveChoice(choiceId: string, payload: ResolveChoicePayload) {
    await apiFetch(`/characters/${characterId.value}/choices/${choiceId}`, {
      method: 'POST',
      body: payload
    })
    await refresh()
  }

  // Undo a choice
  async function undoChoice(choiceId: string) {
    await apiFetch(`/characters/${characterId.value}/choices/${choiceId}`, {
      method: 'DELETE'
    })
    await refresh()
  }

  return {
    pendingChoices,
    choicesByType,
    summary: computed(() => pendingChoices.value?.data.summary),
    refresh,
    resolveChoice,
    undoChoice
  }
}
```

### Phase 3: Store Simplification

Update `characterWizardStore.ts`:

**Remove:**
- `pendingChoices.proficiencies` Map
- `pendingChoices.proficiencyChoiceTypes` Map
- `pendingChoices.languages` Map
- `pendingChoices.equipment` Map
- `pendingChoices.equipmentItems` Map
- `pendingChoices.spells` Set
- `toggleProficiencyChoice()`, `toggleLanguageChoice()`, etc.
- `saveProficiencyChoices()`, `saveLanguageChoices()`, etc.
- `clearExistingEquipment()`

**Add:**
- Integration with `useUnifiedChoices` composable
- Single `resolveChoice(choiceId, payload)` action
- Backend data now includes all pending choice state

**Keep:**
- `selections` (race, class, background, abilities, name, alignment)
- `syncWithBackend()` (still needed for stats/summary)
- Step visibility computed properties

### Phase 4: Component Updates

#### StepProficiencies.vue
- Replace `GET /proficiency-choices` with `pending-choices?type=proficiency`
- Use `resolveChoice()` instead of `POST /proficiency-choices`
- Simplify UI to consume `PendingChoice` structure directly

#### StepLanguages.vue
- Replace `GET /language-choices` with `pending-choices?type=language`
- Use `resolveChoice()` instead of `POST /language-choices`

#### StepEquipment.vue
- Get equipment choices from `pending-choices?type=equipment`
- Each choice group is a single `PendingChoice` with options
- Use `resolveChoice()` with option letter + item selections

#### StepSpells.vue
- Get spell choices from `pending-choices?type=spell`
- Different subtypes: `cantrip`, `spells_known`, `spellbook`
- Use `resolveChoice()` instead of `POST /spells`

#### StepSubclass.vue (Potentially Merge)
- Subclass becomes a `PendingChoice` of type `subclass`
- Could potentially merge into a unified "Choices" step
- Or keep separate for UX clarity

### Phase 5: Wizard Flow Restructuring (Optional)

Consider consolidating choice steps into fewer, more unified steps:

**Option A: Keep Current Steps (Minimal Change)**
- Each step uses the new API but maintains current UX
- Easier migration, familiar to users

**Option B: Unified Choice Steps (Moderate Change)**
- Combine Proficiencies + Languages into "Skills & Languages" step
- Combine Equipment + Spells into "Gear & Magic" step
- Reduces wizard length, modern UX

**Option C: Single Dynamic Choice Step (Maximum Change)**
- One "Complete Your Choices" step
- Shows all pending choices in tabs or accordion
- Most flexible for level-up reuse

**Recommendation:** Start with Option A, consider Option B for v2.

### Phase 6: Cleanup & Deprecation

1. Remove old server routes (after transition period):
   - `server/api/characters/[id]/proficiency-choices.get.ts`
   - `server/api/characters/[id]/proficiency-choices.post.ts`
   - `server/api/characters/[id]/language-choices.get.ts`
   - `server/api/characters/[id]/language-choices.post.ts`

2. Remove unused store methods and state

3. Update tests to use new API patterns

---

## New Choice Types to Support

The unified system adds choice types we don't currently handle:

### Equipment Choices (New!)
- **Current:** Manual loop through class/background equipment, complex item saving
- **New:** Backend provides structured choice groups, single resolution per group

### Subclass as Choice
- **Current:** Separate `PUT /classes/{id}/subclass` endpoint
- **New:** Can be resolved via unified choice system (optional)

### Future: ASI/Feat Choices
- For level-up wizard (not creation)
- Automatically generated at levels 4, 8, 12, 16, 19

### Future: Optional Features
- Warlock Invocations, Sorcerer Metamagic, etc.
- Currently not supported in creation wizard

### Future: Hit Point Choices
- Roll vs Average at level-up
- Not needed for creation (max HP at L1)

---

## Data Structure Comparison

### Proficiency Choices

**Current Response:**
```json
{
  "data": {
    "class": {
      "skill_choice_1": {
        "quantity": 4,
        "remaining": 4,
        "options": [{ "id": 1, "name": "Acrobatics" }]
      }
    },
    "race": { },
    "background": { }
  }
}
```

**New Response:**
```json
{
  "data": {
    "choices": [
      {
        "id": "proficiency:class:5:1:skill_choice_1",
        "type": "proficiency",
        "subtype": "skill",
        "source": "class",
        "source_name": "Rogue",
        "quantity": 4,
        "remaining": 4,
        "options": [{ "id": 1, "name": "Acrobatics" }]
      }
    ]
  }
}
```

### Equipment Choices

**Current:** We manually parse `class.equipment` and `background.equipment` arrays

**New Response:**
```json
{
  "id": "equipment:class:5:1:equipment_choice_1",
  "type": "equipment",
  "source": "class",
  "source_name": "Fighter",
  "options": [
    {
      "option": "a",
      "items": [{ "id": 123, "name": "Chain Mail", "quantity": 1 }]
    },
    {
      "option": "b",
      "items": [
        { "id": 456, "name": "Leather Armor", "quantity": 1 },
        { "id": 789, "name": "Longbow", "quantity": 1 }
      ]
    }
  ]
}
```

---

## Test Strategy

### Unit Tests
- `useUnifiedChoices` composable
- Choice resolution payload builders
- Store integration with new API

### Component Tests
- Each step component with mocked `pending-choices` response
- Choice resolution flows
- Error handling

### Integration Tests
- Full wizard flow with new API
- Choice resolution and undo
- Progress tracking accuracy

---

## Rollback Plan

If issues arise:

1. **Keep old endpoints** on backend for transition period
2. **Feature flag** in frontend to switch between old/new API
3. Old store methods remain available (just unused)

---

## Task Breakdown

### Must Have (MVP)
- [ ] Create Nitro routes for new endpoints
- [ ] Create TypeScript types for unified choice system
- [ ] Create `useUnifiedChoices` composable
- [ ] Update StepProficiencies to use new API
- [ ] Update StepLanguages to use new API
- [ ] Update StepEquipment to use new API
- [ ] Update StepSpells to use new API
- [ ] Update store to remove old pending choice tracking
- [ ] Update all tests

### Should Have
- [ ] Subclass step using unified choice (optional)
- [ ] Error handling improvements
- [ ] Loading states per-choice

### Could Have
- [ ] Wizard step consolidation (Option B)
- [ ] Undo choice functionality
- [ ] Choice validation before proceeding

### Won't Have (v1)
- [ ] Level-up wizard (separate epic)
- [ ] ASI/Feat choices (creation is L1 only)
- [ ] Hit point roll choices (L1 is max HP)

---

## Open Questions

1. ~~**Backend availability:** When will the new endpoints be deployed to staging?~~ ✅ Available now
2. ~~**Transition period:** How long will old endpoints remain available?~~ ✅ No transition needed, can start immediately
3. **Equipment choices:** Does backend handle "starting gold alternative"? → **Deferred to v2** (not critical for MVP)
4. **Spell swaps:** Is swap functionality needed for creation (L1 only)? → **No** (swaps are for level-up only)
5. **Validation:** Does backend validate choice completion before marking creation complete? → **See #263** (ticket created)

---

## Related Issues

- #246 - Epic: Unified Character Choice System (CLOSED - backend complete)
- #247 - Core Infrastructure (CLOSED)
- #249-253, #257, #259-262 - Individual choice type migrations (CLOSED)
- #254 - HP Auto-initialization (OPEN - not critical for creation)
- #255 - Enhanced Stats Endpoint (OPEN - nice to have)
- #256 - Feature Uses Tracking (not created yet)
- #263 - Validation questions for unified choice system (OPEN - created during this planning)

---

## Notes

- Backend work is complete (all sub-issues closed)
- API types need to be regenerated (`npm run types:sync`)
- Backend must be running to test locally
- Consider creating a frontend issue to track this migration work
