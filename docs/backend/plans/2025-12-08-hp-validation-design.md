# Design: HP Validation for Calculated Characters

**Issue:** #357
**Date:** 2025-12-08
**Status:** Approved

## Overview

Characters using `hp_calculation_method: 'calculated'` should not allow manual HP overrides that bypass the calculation system. Additionally, players should be able to use physical dice for HP rolls.

## Requirements

1. Reject direct `max_hit_points` updates for calculated characters
2. Allow mode switching between 'calculated' and 'manual'
3. Support manual/physical dice rolls during HP choice resolution

## Design

### 1. Manual Roll Support in HitPointRollChoiceHandler

**Current selections:**
```php
['selected' => 'roll']     // Server generates random_int(1, $hitDie)
['selected' => 'average']  // Server calculates floor($hitDie/2) + 1
```

**New selection:**
```php
['selected' => 'manual', 'roll_result' => 5]  // Client provides roll
```

**Validation:**
- `roll_result` required when `selected === 'manual'`
- Must be integer between 1 and hit_die
- Throws `InvalidSelectionException` if out of range

**HP calculation:** Same formula as server roll:
```php
$hpGained = max(1, $rollResult + $conModifier);
```

### 2. Form Request Validation

**CharacterUpdateRequest changes:**

```php
'max_hit_points' => [
    'sometimes',
    'nullable',
    'integer',
    'min:1',
    'prohibited_if:hp_calculation_method,calculated'
],

'hp_calculation_method' => [
    'sometimes',
    'string',
    'in:calculated,manual'
],
```

**Behavior:**
- Calculated mode: reject `max_hit_points` in PATCH body (422 error)
- Manual mode: allow direct HP updates
- Mode switch + HP update in same request: allowed (mode processed first)

**Error response:**
```json
{
    "message": "The given data was invalid.",
    "errors": {
        "max_hit_points": ["Cannot modify max_hit_points when hp_calculation_method is 'calculated'."]
    }
}
```

### 3. Mode Switching

**Via standard PATCH:**
```json
PATCH /api/v1/characters/{id}
{"hp_calculation_method": "manual"}
```

**Switching to manual:**
- HP values preserved
- User takes full control
- CON changes no longer auto-adjust HP

**Switching to calculated:**
- HP values preserved (no recalculation)
- Future CON changes auto-adjust HP
- Future level-ups use HP choice system

## Files to Modify

| File | Changes |
|------|---------|
| `app/Services/ChoiceHandlers/HitPointRollChoiceHandler.php` | Add 'manual' selection handling |
| `app/Http/Requests/Character/CharacterUpdateRequest.php` | Add prohibited_if rule, hp_calculation_method validation |
| `tests/Unit/Services/ChoiceHandlers/HitPointRollChoiceHandlerTest.php` | Add manual roll tests |
| `tests/Feature/Api/CharacterControllerTest.php` | Add HP protection tests |

## Test Cases

### Unit Tests (HitPointRollChoiceHandlerTest)
- it accepts manual roll with valid roll_result
- it rejects manual roll without roll_result
- it rejects manual roll with roll_result below 1
- it rejects manual roll with roll_result above hit_die
- it calculates HP correctly for manual roll with CON modifier

### Feature Tests (CharacterControllerTest)
- it rejects max_hit_points update for calculated mode
- it allows max_hit_points update for manual mode
- it allows switching from calculated to manual mode
- it allows switching from manual to calculated mode
- it allows max_hit_points when switching to manual in same request

## Acceptance Criteria

- [ ] API rejects direct HP updates for calculated characters
- [ ] Users can switch to manual mode if desired
- [ ] Users can submit physical dice rolls for HP choices
- [ ] Roll results validated against hit die range
- [ ] Tests verify all validation behavior
