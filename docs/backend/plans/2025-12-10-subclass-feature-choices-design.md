# Subclass Feature Choices in Pending-Choices API

**Issue:** #465
**Date:** 2025-12-10
**Status:** Design Complete

## Problem

The backend parses subclass feature data (bonus cantrips, skill choices, spell choices) into the database, but this data isn't surfaced through the pending-choices API for the character wizard.

### Gaps Identified

1. **Proficiency choices from subclass features not surfaced** - `CharacterProficiencyService::getPendingChoices()` only checks class, race, background
2. **Spell choices from subclass features not surfaced** - `SpellChoiceHandler::getChoices()` only checks level progression
3. **Fixed bonus cantrips not auto-granted (bug)** - `wherePivot('level_requirement', '<=', $characterLevel)` excludes NULL values

## Solution

### Part 1: Fix Bonus Cantrip Auto-Grant Bug

**File:** `app/Services/CharacterFeatureService.php`

Change the `assignSpellsFromFeature()` query to include NULL level_requirement:

```php
->where(function ($query) use ($characterLevel) {
    $query->wherePivot('level_requirement', '<=', $characterLevel)
          ->orWhereNull('entity_spells.level_requirement');
})
```

NULL means "no level requirement" = available at the feature's level.

### Part 2: Extend Proficiency Choices for Subclass Features

**File:** `app/Services/CharacterProficiencyService.php`

Add fourth check in `getPendingChoices()` for subclass features:

```php
// Check subclass feature choices
$primaryClassPivot = $character->characterClasses->first();
if ($primaryClassPivot && $primaryClassPivot->subclass) {
    $subclassChoices = [];
    foreach ($primaryClassPivot->subclass->features as $feature) {
        $featureChoices = $this->getChoicesFromEntity(
            $feature,
            $character,
            'subclass_feature'
        );
        foreach ($featureChoices as $group => $data) {
            $key = $feature->feature_name . ':' . $group;
            $subclassChoices[$key] = $data;
        }
    }
    $choices['subclass_feature'] = $subclassChoices;
}
```

**File:** `app/Services/ChoiceHandlers/ProficiencyChoiceHandler.php`

Extend `getSourceSlug()` and `getSourceName()`:
- Source slug: subclass's `full_slug`
- Source name: feature name

### Part 3: Extend Spell Choices for Subclass Features

**File:** `app/Services/ChoiceHandlers/SpellChoiceHandler.php`

After existing level progression loop, add check for subclass feature spell choices:

```php
// Check subclass feature spell choices
foreach ($character->characterClasses as $pivot) {
    $subclass = $pivot->subclass;
    if (!$subclass) continue;

    $spellChoices = EntitySpell::where('reference_type', ClassFeature::class)
        ->whereIn('reference_id', $subclass->features->pluck('id'))
        ->where('is_choice', true)
        ->with(['characterClass', 'reference'])
        ->get();

    foreach ($spellChoices as $spellChoice) {
        $feature = $spellChoice->reference;
        $choice = $this->buildFeatureSpellChoice($character, $pivot, $feature, $spellChoice);
        if ($choice) {
            $choices->push($choice);
        }
    }
}
```

New helper `buildFeatureSpellChoice()`:
- `source: 'subclass_feature'`
- `sourceName`: Feature name
- `subtype`: 'cantrip' or 'spell' based on `is_cantrip`/`max_level`
- `optionsEndpoint`: Filter by `class_id`
- `quantity`: from `choice_count` (default 1)

### Part 4: Supporting Changes

**File:** `app/Enums/CharacterSource.php`
- Add `SUBCLASS_FEATURE = 'subclass_feature'` case

**File:** `app/Services/CharacterProficiencyService.php`
- Extend `makeSkillChoice()` entity lookup for subclass_feature
- Add helper `getSubclassFeatureEntity()` to find ClassFeature by choice_group

**File:** `app/Services/ChoiceHandlers/SpellChoiceHandler.php`
- Extend `resolve()` to handle `source='subclass_feature'`
- Extend `undo()` to clear subclass_feature spells

## Test Cases

### Unit Tests - Bug Fix
- Assigns spells with `level_requirement=NULL`
- Still respects `level_requirement` when set

### Unit Tests - Proficiency Choices
- `getPendingChoices()` returns subclass feature choices
- `makeSkillChoice()` works with `source='subclass_feature'`
- `ProficiencyChoiceHandler` builds correct PendingChoice

### Unit Tests - Spell Choices
- `getChoices()` returns subclass feature cantrip choices
- `resolve()` creates CharacterSpell with `source='subclass_feature'`
- `undo()` clears correct spells

### Integration Tests
- Cleric + Nature Domain → skill choice and cantrip choice in pending-choices
- Cleric + Light Domain → `light` cantrip auto-assigned
- Resolve choices → CharacterProficiency and CharacterSpell records created

## Files to Modify

1. `app/Services/CharacterFeatureService.php` - Bug fix
2. `app/Services/CharacterProficiencyService.php` - Add subclass feature choices
3. `app/Services/ChoiceHandlers/ProficiencyChoiceHandler.php` - Handle new source
4. `app/Services/ChoiceHandlers/SpellChoiceHandler.php` - Add feature spell choices
5. `app/Enums/CharacterSource.php` - Add SUBCLASS_FEATURE
6. Tests for all above
