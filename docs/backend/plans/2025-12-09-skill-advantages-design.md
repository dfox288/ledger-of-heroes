# Skill Check Advantages - Design Document

**Issue:** #429 - Parse and store skill check advantages from traits
**Date:** 2025-12-09
**Status:** Ready for Implementation

## Overview

Parse skill check advantages from racial traits and feats into structured data, and expose them via the character stats API.

## Scope

**In Scope:**
- Parse skill advantages from race traits (e.g., Stonecunning)
- Parse skill advantages from feats (already working)
- Expose in `/characters/{id}/stats` API

**Out of Scope (Deferred):**
- Class features (most don't grant skill advantages; can add later if needed)

## Storage Strategy

Use existing `entity_modifiers` table with `modifier_category = 'skill_advantage'`.

**No migration needed** - schema already has:
- `skill_id` (foreign key to skills table)
- `condition` (text field for conditional advantages)
- `value` (stores 'advantage')

**Current State:**
- `FeatXmlParser.parseSkillAdvantages()` already parses feats ✓
- `ImportsModifiers` trait resolves `skill_name` → `skill_id` ✓
- Feat skill advantages stored in DB ✓

## API Response Structure

```json
{
  "skill_advantages": [
    {
      "skill": "History",
      "skill_slug": "history",
      "condition": "related to the origin of stonework",
      "source": "Stonecunning (Dwarf)"
    },
    {
      "skill": "Deception",
      "skill_slug": "deception",
      "condition": null,
      "source": "Actor"
    }
  ]
}
```

Matches existing defensive traits pattern (simple `source` string).

## Implementation Plan

### Task 1: Extract Shared Trait

**Files:**
- Create: `app/Services/Parsers/Concerns/ParsesSkillAdvantages.php`
- Modify: `app/Services/Parsers/FeatXmlParser.php`

**Steps:**
1. Create `ParsesSkillAdvantages` trait with `parseSkillAdvantages(string $text): array` method
2. Move existing logic from `FeatXmlParser::parseSkillAdvantages()` to trait
3. Update `FeatXmlParser` to use trait
4. Run existing feat tests to verify no regression

**Acceptance:** Existing `FeatXmlParserTest` passes unchanged.

---

### Task 2: Add Race Parsing - Tests First

**Files:**
- Create: `tests/Unit/Parsers/Concerns/ParsesSkillAdvantagesTest.php`
- Modify: `tests/Unit/Parsers/RaceXmlParserTest.php`

**Test Cases (ParsesSkillAdvantagesTest):**
```php
it('parses single skill advantage')
// Input: "advantage on Intelligence (History) checks"
// Expected: [['modifier_category' => 'skill_advantage', 'skill_name' => 'History', 'value' => 'advantage', 'condition' => null]]

it('parses dual skill advantages')
// Input: "advantage on Charisma (Deception) and Charisma (Performance) checks"
// Expected: Two modifiers for Deception and Performance

it('captures conditional text after when')
// Input: "advantage on Intelligence (History) checks when related to stonework"
// Expected: condition = "related to stonework"

it('captures conditional text after while')
// Input: "advantage on Wisdom (Perception) checks while underground"
// Expected: condition = "underground"

it('returns empty array for non-matching text')
// Input: "You have proficiency in the Perception skill"
// Expected: []

it('ignores proficiency grants')
// Input: "proficiency in Perception"
// Expected: []
```

**Test Cases (RaceXmlParserTest):**
```php
it('parses skill advantages from trait descriptions')
// Use Dwarf Stonecunning trait XML
// Verify skill_advantage_modifiers returned in parsed data
```

---

### Task 3: Implement Race Parsing

**Files:**
- Modify: `app/Services/Parsers/RaceXmlParser.php`

**Steps:**
1. Add `use ParsesSkillAdvantages` to trait list
2. Create `parseSkillAdvantagesFromTraits(array $traits): array` method
3. Call from `parseRace()` method, iterating over trait descriptions
4. Return skill advantages in parsed race data

**Integration Point:**
```php
// In parseRace(), after parseConditionsAndImmunities():
$data['skill_advantage_modifiers'] = $this->parseSkillAdvantagesFromTraits($traits);
```

---

### Task 4: Add API Exposure - Tests First

**Files:**
- Create: `tests/Feature/Api/CharacterStatsSkillAdvantagesTest.php`

**Test Cases:**
```php
it('returns skill advantages from race')
// Create character with race having skill_advantage modifier
// GET /api/v1/characters/{id}/stats
// Assert skill_advantages array contains expected entry

it('returns skill advantages from feats')
// Create character with feat having skill_advantage modifier
// Assert skill_advantages contains feat entry

it('aggregates skill advantages from multiple sources')
// Character with both race and feat skill advantages
// Assert both appear in skill_advantages array

it('returns empty array when no skill advantages')
// Character with no skill advantage modifiers
// Assert skill_advantages is empty array

it('includes correct structure with skill_slug')
// Assert each entry has: skill, skill_slug, condition, source
```

---

### Task 5: Implement API Exposure

**Files:**
- Modify: `app/DTOs/CharacterStatsDTO.php`
- Modify: `app/Http/Resources/CharacterStatsResource.php`

**CharacterStatsDTO Changes:**

1. Add property:
```php
/** @var array<int, array{skill: string, skill_slug: string, condition: string|null, source: string}> */
public array $skillAdvantages = [];
```

2. Update `extractDefensiveTraitsFromEntity()` signature to accept `&$skillAdvantages` parameter

3. Add skill advantage extraction in modifier loop:
```php
if ($category === 'skill_advantage' && $modifier->skill) {
    $skillAdvantages[] = [
        'skill' => $modifier->skill->name,
        'skill_slug' => $modifier->skill->slug,
        'condition' => $modifier->condition,
        'source' => $sourceName,
    ];
}
```

4. Update eager-loading to include `modifiers.skill`:
```php
if (! $entity->relationLoaded('modifiers')) {
    $entity->load(['modifiers.damageType', 'modifiers.skill']);
}
```

5. Update `buildDefensiveTraits()` to initialize and return `skill_advantages`

6. Update `fromCharacter()` to assign `$dto->skillAdvantages`

**CharacterStatsResource Changes:**

Add to `toArray()`:
```php
/** @var array<int, array{skill: string, skill_slug: string, condition: string|null, source: string}> Skill check advantages from race and feats */
'skill_advantages' => $this->resource->skillAdvantages,
```

---

### Task 6: Update Importer (if needed)

**Files:**
- Modify: `app/Services/Importers/RaceImporter.php`

**Check:** Verify `importEntityModifiers()` is called with parsed skill advantage data.

If not already connected:
```php
// In importRace() or relevant method:
$this->importEntityModifiers($race, $parsedData['skill_advantage_modifiers'] ?? []);
```

---

### Task 7: Integration Verification

**Run test suites:**
```bash
# Unit tests for parser
docker compose exec php ./vendor/bin/pest tests/Unit/Parsers/Concerns/ParsesSkillAdvantagesTest.php

# Feature tests for API
docker compose exec php ./vendor/bin/pest tests/Feature/Api/CharacterStatsSkillAdvantagesTest.php

# Full suite verification
docker compose exec php ./vendor/bin/pest --testsuite=Unit-Pure
docker compose exec php ./vendor/bin/pest --testsuite=Feature-DB
```

## Patterns to Parse

| Source | Trait | Text Pattern | Expected Result |
|--------|-------|--------------|-----------------|
| Dwarf | Stonecunning | "advantage on Intelligence (History) checks related to the origin of stonework" | skill: History, condition: "related to the origin of stonework" |
| Feat: Actor | - | "advantage on Charisma (Deception) and Charisma (Performance) checks when trying to pass yourself off" | skill: Deception + Performance, condition: "when trying to pass yourself off..." |
| Feat: Dungeon Delver | - | "advantage on Wisdom (Perception) and Intelligence (Investigation) checks made to detect secret doors" | skill: Perception + Investigation, condition: "made to detect secret doors" |

## Patterns to Skip

| Text | Reason |
|------|--------|
| "You have proficiency in the Perception skill" | Proficiency, not advantage |
| "You can attempt to hide even when..." | Not an advantage grant |
| "+5 passive Perception" | Passive bonus, not advantage |

## Files Modified Summary

**New Files:**
- `app/Services/Parsers/Concerns/ParsesSkillAdvantages.php`
- `tests/Unit/Parsers/Concerns/ParsesSkillAdvantagesTest.php`
- `tests/Feature/Api/CharacterStatsSkillAdvantagesTest.php`

**Modified Files:**
- `app/Services/Parsers/FeatXmlParser.php` (use trait)
- `app/Services/Parsers/RaceXmlParser.php` (use trait + call parser)
- `app/DTOs/CharacterStatsDTO.php` (add skillAdvantages property + extraction)
- `app/Http/Resources/CharacterStatsResource.php` (expose skill_advantages)
- `app/Services/Importers/RaceImporter.php` (if connection needed)
- `tests/Unit/Parsers/RaceXmlParserTest.php` (add skill advantage test)

## Acceptance Criteria

- [ ] Skill advantages parsed from racial traits
- [ ] Skill advantages parsed from feats (already working, verify no regression)
- [ ] Conditional text captured
- [ ] Exposed in `/characters/{id}/stats` API with structure: `{skill, skill_slug, condition, source}`
- [ ] Tests for parsing logic
- [ ] Tests for API exposure
