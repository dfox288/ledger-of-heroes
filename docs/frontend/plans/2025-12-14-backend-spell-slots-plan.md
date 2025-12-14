# Backend: Spell Slot Tracking & Preparation Toggle

**Issue:** #612
**Created:** 2025-12-14
**For:** Backend implementation

## Overview

Two new endpoints needed for interactive spell management in the Character Sheet Spells Tab.

---

## Endpoint 1: Toggle Spell Preparation

### Route
```
PATCH /api/v1/characters/{character}/spells/{characterSpell}
```

### Request Body
```json
{
  "is_prepared": true
}
```

### Response (Success - 200)
Return the updated `CharacterSpellResource`:
```json
{
  "data": {
    "id": 42,
    "spell": { "id": 123, "name": "Fireball", "slug": "phb:fireball", ... },
    "spell_slug": "phb:fireball",
    "is_dangling": false,
    "preparation_status": "prepared",
    "source": "class",
    "level_acquired": 1,
    "is_prepared": true,
    "is_always_prepared": false
  }
}
```

### Validation Rules

| Rule | Error Message |
|------|---------------|
| `characterSpell` belongs to `character` | 404 Not Found |
| `is_always_prepared` spells cannot be unprepared | "This spell is always prepared and cannot be unprepared." |
| Cannot prepare cantrips (level 0) | "Cantrips do not need to be prepared." |
| Cannot exceed `preparation_limit` when preparing | "Cannot prepare more than {limit} spells." |
| Known casters (Sorcerer, Bard) don't prepare | "This character does not prepare spells." |

### Implementation Notes

1. **Check preparation type by class:**
   - Preparation casters: Cleric, Druid, Paladin, Wizard
   - Known casters (no preparation): Sorcerer, Bard, Warlock, Ranger

2. **Preparation limit calculation:**
   - Already exists in `CharacterStatsResource` as `preparation_limit`
   - Typically: ability modifier + class level (varies by class)

3. **Update `preparation_status` field:**
   - `"prepared"` when `is_prepared = true`
   - `"known"` when `is_prepared = false` (for prep casters)

### Form Request
```php
// app/Http/Requests/UpdateCharacterSpellRequest.php
public function rules(): array
{
    return [
        'is_prepared' => ['required', 'boolean'],
    ];
}
```

### Controller Method
```php
// app/Http/Controllers/CharacterSpellController.php
public function update(UpdateCharacterSpellRequest $request, Character $character, CharacterSpell $characterSpell)
{
    // Validation handled in Form Request or here
    // Update is_prepared
    // Return CharacterSpellResource
}
```

---

## Endpoint 2: Update Spell Slot Usage

### Route
```
PATCH /api/v1/characters/{character}/spell-slots/{level}
```

Where `{level}` is the spell slot level (1-9).

### Request Body (Option A - Direct Value)
```json
{
  "spent": 2
}
```

### Request Body (Option B - Action-Based) *Recommended*
```json
{ "action": "use" }      // spent += 1
{ "action": "restore" }  // spent -= 1
{ "action": "reset" }    // spent = 0
```

Action-based is recommended because:
- Prevents race conditions from concurrent requests
- Clearer intent in API logs
- Frontend doesn't need current state to make request

### Response (Success - 200)
```json
{
  "data": {
    "level": 1,
    "total": 4,
    "spent": 2,
    "available": 2
  }
}
```

### Validation Rules

| Rule | Error Message |
|------|---------------|
| `level` must be 1-9 | "Invalid spell slot level." |
| Character must have slots at this level | "Character has no spell slots at level {level}." |
| `use`: Cannot exceed total slots | "No spell slots remaining at level {level}." |
| `restore`: Cannot go below 0 spent | "No spent slots to restore at level {level}." |
| `spent` value must be 0 to total | "Spent slots must be between 0 and {total}." |

### Implementation Notes

1. **Spell slot storage:**
   - If slots are computed (not stored), need new table/column
   - Suggested: `character_spell_slots` table or JSON column on characters

2. **Schema for tracking:**
   ```sql
   CREATE TABLE character_spell_slots (
       id BIGINT PRIMARY KEY,
       character_id BIGINT REFERENCES characters(id),
       level TINYINT NOT NULL,          -- 1-9
       spent TINYINT NOT NULL DEFAULT 0,
       created_at TIMESTAMP,
       updated_at TIMESTAMP,
       UNIQUE(character_id, level)
   );
   ```

3. **Pact Magic (Warlock):**
   - Separate endpoint or same with `type` parameter?
   - Suggest: `PATCH /characters/{id}/spell-slots/pact` for pact slots
   - Or include `"type": "pact"` in request body

### Form Request
```php
// app/Http/Requests/UpdateSpellSlotRequest.php
public function rules(): array
{
    return [
        'action' => ['required_without:spent', 'in:use,restore,reset'],
        'spent' => ['required_without:action', 'integer', 'min:0'],
    ];
}
```

---

## Endpoint 3: Long Rest Integration

### Existing Route
```
POST /api/v1/characters/{character}/long-rest
```

### Required Change
Add spell slot reset to existing long rest logic:

```php
// In LongRestAction or CharacterController::longRest()

// Existing: Reset HP, hit dice, etc.

// Add: Reset spell slots
$character->spellSlots()->update(['spent' => 0]);

// Or if using JSON column:
$character->update([
    'spell_slots_spent' => [] // Clear all spent tracking
]);
```

### Response Update
Include spell slots in response:
```json
{
  "data": {
    "message": "Long rest completed",
    "hit_points": { "current": 45, "max": 45 },
    "hit_dice": [...],
    "spell_slots": {
      "1": { "total": 4, "spent": 0, "available": 4 },
      "2": { "total": 3, "spent": 0, "available": 3 }
    }
  }
}
```

---

## Nitro Server Routes Needed (Frontend)

After backend implements these, frontend needs:

```
server/api/characters/[id]/spells/[spellId].patch.ts
server/api/characters/[id]/spell-slots/[level].patch.ts
```

(Long rest route already exists)

---

## Testing Checklist

### Preparation Toggle
- [ ] Can prepare a spell (under limit)
- [ ] Can unprepare a spell
- [ ] Cannot unprepare always-prepared spell
- [ ] Cannot exceed preparation limit
- [ ] Cannot prepare for known casters (Sorcerer)
- [ ] Cannot prepare cantrips
- [ ] 404 for spell not belonging to character

### Spell Slot Usage
- [ ] Can use a slot (action: use)
- [ ] Can restore a slot (action: restore)
- [ ] Can reset all slots at level (action: reset)
- [ ] Can set exact spent value
- [ ] Cannot use when no slots remaining
- [ ] Cannot restore when none spent
- [ ] 404 for invalid level
- [ ] Works for pact magic (if applicable)

### Long Rest Integration
- [ ] Spell slots reset to 0 spent
- [ ] Response includes updated slot state
- [ ] Existing HP/hit dice behavior unchanged

---

## Migration Order

1. Add `character_spell_slots` table (or JSON column)
2. Implement `PATCH /spell-slots/{level}` endpoint
3. Implement `PATCH /spells/{characterSpell}` endpoint
4. Update long rest to reset slots
5. Update `CharacterSpellSlotsResource` if needed
