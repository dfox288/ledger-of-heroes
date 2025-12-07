# Classes Detail Page: Backend Data Fixes Proposal

**Date**: 2025-11-26
**Author**: Claude (API Verification Audit)
**Status**: ⚠️ SUPERSEDED - See Comprehensive Audit
**Priority**: High
**Last Verified**: 2025-11-29

> **Note**: This document tracked initial UI/display issues which have been resolved.
> A comprehensive D&D 5e rules accuracy audit was conducted on 2025-11-29 which found
> additional critical issues with class data (Sneak Attack, Invocations, etc.).
> See: **`CLASSES-COMPREHENSIVE-AUDIT-2025-11-29.md`** for the full rules audit.

---

## Executive Summary

A comprehensive audit of the Classes detail page API responses revealed several data consistency issues, structural problems, and missing fields that impact the frontend display and D&D 5e rules accuracy. This proposal outlines required backend changes to fix these issues.

### Verification Status (2025-11-29 - ALL RESOLVED ✅)

| Status | Count | Issues |
|--------|-------|--------|
| ✅ Resolved | 15 | hit_die fix, subclass descriptions, multiclass flag, wizard columns, fighter columns, higher_levels description, Fighting Style is_choice_option, Pact Boon is_choice_option, subclass_level populated, Totem options removed, Champion L10 simplified, `archetype` field added, **Barbarian rage_damage**, **Monk martial_arts/ki**, **Rogue sneak_attack** |
| ❌ Outstanding | 0 | **None! All issues resolved!** |

### Latest Verification (2025-11-29 - Third Pass)

**🎉 ALL ISSUES NOW RESOLVED!**

- ✅ `is_choice_option: true` works for Fighting Style options (Fighter, Ranger, Paladin)
- ✅ `is_choice_option: true` works for Pact Boon options (Warlock)
- ✅ `subclass_level` is populated (Fighter=3, Wizard=2, Cleric=1, etc.)
- ✅ **Totem Warrior**: Individual Bear/Eagle/Wolf options **removed** from feature list (cleaner approach!)
- ✅ **Champion L10**: Individual style options **removed**, only "Additional Fighting Style (Champion)" shown
- ✅ **`archetype` field** added instead of `subclass_name` (e.g., "Martial Archetype", "Arcane Tradition", "Divine Domain")
- ✅ **Barbarian**: Now has `rage` AND `rage_damage` columns (matches PHB p.47)
- ✅ **Monk**: Now has `martial_arts` and `ki` columns (matches PHB p.77)
- ✅ **Rogue**: Now has `sneak_attack` column (matches PHB p.95)

### Original Impact Summary

| Priority | Issues | Effort |
|----------|--------|--------|
| Critical | 2 | Low-Medium |
| High | 4 | Medium |
| Medium | 5 | Medium-High |
| Low | 3 | Low |

---

## Table of Contents

1. [Critical Issues](#1-critical-issues)
   - 1.1 [Subclass hit_die: 0 Bug](#11-subclass-hit_die-0-bug)
   - 1.2 [Subclass Description Placeholder](#12-subclass-description-placeholder)
2. [High Priority Issues](#2-high-priority-issues)
   - 2.1 [Choice Options as Separate Features](#21-choice-options-as-separate-features)
   - 2.2 [Multiclass Rules Mixed with Core Features](#22-multiclass-rules-mixed-with-core-features)
   - 2.3 [Feature Count Inflation](#23-feature-count-inflation)
   - 2.4 [Progression Table Feature List Cluttered](#24-progression-table-feature-list-cluttered)
3. [Medium Priority Issues](#3-medium-priority-issues)
   - 3.1 [Missing Subclass Entry Level](#31-missing-subclass-entry-level)
   - 3.2 [Counters Not Consolidated](#32-counters-not-consolidated)
   - 3.3 [Irrelevant Progression Columns](#33-irrelevant-progression-columns)
   - 3.4 [Higher Levels Description Incorrect](#34-higher-levels-description-incorrect)
   - 3.5 [Duplicate Description Content](#35-duplicate-description-content)
4. [Low Priority Issues](#4-low-priority-issues)
5. [Implementation Recommendations](#5-implementation-recommendations)
6. [Appendix A: Affected Classes/Subclasses](#6-appendix-a-affected-classessubclasses)
7. [Appendix B: Canonical PHB Progression Table Columns](#7-appendix-b-canonical-phb-progression-table-columns)

---

## 1. Critical Issues

### 1.1 Subclass hit_die: 0 Bug

**Severity**: Critical
**Affected Endpoints**: `GET /api/v1/classes/{slug}` (subclass responses)
**Status**: ✅ RESOLVED (via `effective_hit_die` field)

#### Problem

Some subclasses return `hit_die: 0` instead of inheriting from their parent class:

```json
// GET /api/v1/classes/cleric-death-domain
{
  "data": {
    "name": "Death Domain",
    "hit_die": 0,  // ❌ Should be 8 (Cleric's hit die)
    "parent_class_id": 23,
    "computed": {
      "hit_points": null  // ❌ Can't compute without valid hit_die
    }
  }
}
```

#### D&D 5e Rules Reference

> "Your hit points are determined by your Hit Dice. At 1st level, you have 1 Hit Die, and the die type is determined by your class." - PHB p. 12

Subclasses do not have their own hit dice; they use their parent class's hit die.

#### Affected Subclasses

| Subclass | Current hit_die | Parent | Expected |
|----------|-----------------|--------|----------|
| Cleric - Death Domain | 0 | Cleric | 8 |
| Paladin - Oathbreaker | 0 | Paladin | 10 |
| *(potentially others)* | 0 | - | - |

#### Implementation (2025-11-26)

**Approach Taken**: Added new `effective_hit_die` field instead of modifying raw `hit_die`

```json
// Current API Response
{
  "data": {
    "name": "Death Domain",
    "hit_die": 0,              // Raw value preserved
    "effective_hit_die": 8,    // ✅ NEW: Correct inherited value
    "computed": {
      "hit_points": {
        "hit_die": "d8",
        "hit_die_numeric": 8,
        "first_level": { "value": 8, "description": "8 + your Constitution modifier" },
        "higher_levels": { "description": "1d8 (or 5) + your Constitution modifier per cleric level after 1st" }
      }
    }
  }
}
```

**Frontend Action**: Use `effective_hit_die` for display instead of `hit_die`.

#### Acceptance Criteria

- [x] All subclasses return non-zero `effective_hit_die` matching parent class
- [x] `computed.hit_points` is populated for all subclasses
- [x] `computed.hit_points.higher_levels.description` uses correct hit die

---

### 1.2 Subclass Description Placeholder

**Severity**: Critical
**Affected Endpoints**: `GET /api/v1/classes/{slug}` (subclass responses)
**Status**: ✅ RESOLVED

#### Problem

All subclass descriptions are generic placeholders instead of actual content:

```json
// GET /api/v1/classes/wizard-school-of-abjuration
{
  "data": {
    "name": "School of Abjuration",
    "description": "Subclass of Wizard"  // ❌ Not useful
  }
}
```

The actual descriptive content exists in the first feature:

```json
{
  "features": [
    {
      "feature_name": "Arcane Tradition: School of Abjuration",
      "description": "The School of Abjuration emphasizes magic that blocks, banishes, or protects..."  // ✅ This is the real description
    }
  ]
}
```

#### D&D 5e Rules Reference

Each subclass in the PHB has an introductory paragraph explaining its theme and flavor. For School of Abjuration (PHB p. 115):

> "The School of Abjuration emphasizes magic that blocks, banishes, or protects. Detractors of this school say that its tradition is about denial, negation rather than positive assertion..."

#### Implementation (2025-11-26)

Subclass descriptions are now populated with actual flavor text from the source material:

```json
// Current API Response - School of Abjuration
{
  "data": {
    "name": "School of Abjuration",
    "description": "The School of Abjuration emphasizes magic that blocks, banishes, or protects. Detractors of this school say that its tradition is about denial, negation rather than positive assertion. You understand, however, that ending harmful effects, protecting the weak, and banishing evil influences is anything but a philosophical void. It is a proud and respected vocation.\n\tCalled abjurers, members of this school are sought when baleful spirits require exorcism, when important locations must be guarded against magical spying, and when portals to other planes of existence must be closed.\n\nSource:\tPlayer's Handbook (2014) p. 115"
  }
}

// Current API Response - Champion
{
  "data": {
    "name": "Champion",
    "description": "The archetypal Champion focuses on the development of raw physical power honed to deadly perfection. Those who model themselves on this archetype combine rigorous training with physical excellence to deal devastating blows.\n\nSource:\tPlayer's Handbook (2014) p. 72"
  }
}
```

#### Acceptance Criteria

- [x] Subclass `description` contains thematic flavor text
- [x] Description is 1-3 paragraphs (includes Source reference)
- [x] Placeholder "Subclass of X" is replaced for all subclasses

---

## 2. High Priority Issues

### 2.1 Choice Options as Separate Features

**Severity**: High
**Affected Endpoints**: All class detail endpoints
**Status**: ✅ RESOLVED (2025-11-29)

#### Problem

Features that represent choices (pick one option) are stored as separate features, inflating feature counts and cluttering the UI:

```json
// Fighter at Level 1
{
  "features": [
    { "feature_name": "Fighting Style" },
    { "feature_name": "Fighting Style: Archery" },
    { "feature_name": "Fighting Style: Defense" },
    { "feature_name": "Fighting Style: Dueling" },
    { "feature_name": "Fighting Style: Great Weapon Fighting" },
    { "feature_name": "Fighting Style: Protection" },
    { "feature_name": "Fighting Style: Two-Weapon Fighting" }
  ]
}
// Shows as 7 features, but player only GETS 1 choice
```

Similarly for:
- Totem Warrior: Bear, Eagle, Wolf options at levels 3, 6, 14
- Eldritch Invocations: Many options listed separately
- Maneuvers: Listed as separate "spells"

#### D&D 5e Rules Reference

PHB p. 72 (Fighting Style):
> "You adopt a particular style of fighting as your specialty. Choose one of the following options."

The key word is **choose one** - these are options, not separate features gained.

#### Required Fix

**Add new fields to `class_features` table:**

```sql
ALTER TABLE class_features ADD COLUMN is_choice_option BOOLEAN DEFAULT FALSE;
ALTER TABLE class_features ADD COLUMN parent_feature_id BIGINT UNSIGNED NULL;
ALTER TABLE class_features ADD COLUMN choice_group VARCHAR(50) NULL;

-- Add foreign key
ALTER TABLE class_features
ADD CONSTRAINT fk_parent_feature
FOREIGN KEY (parent_feature_id) REFERENCES class_features(id);

-- Create index
CREATE INDEX idx_choice_group ON class_features(class_id, choice_group);
```

**Update data:**

```sql
-- Mark Fighting Style options
UPDATE class_features
SET is_choice_option = TRUE,
    parent_feature_id = (
        SELECT id FROM class_features f2
        WHERE f2.class_id = class_features.class_id
        AND f2.feature_name = 'Fighting Style'
        AND f2.level = class_features.level
    ),
    choice_group = 'fighting_style'
WHERE feature_name LIKE 'Fighting Style: %';

-- Mark Totem options
UPDATE class_features
SET is_choice_option = TRUE,
    choice_group = 'totem_spirit_3'
WHERE feature_name IN ('Bear (Path of the Totem Warrior)', 'Eagle (Path of the Totem Warrior)', 'Wolf (Path of the Totem Warrior)')
AND level = 3;
```

**Update ClassFeatureResource:**

```php
class ClassFeatureResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'level' => $this->level,
            'feature_name' => $this->feature_name,
            'description' => $this->description,
            'is_optional' => $this->is_optional,
            'is_choice_option' => $this->is_choice_option,  // NEW
            'parent_feature_id' => $this->parent_feature_id,  // NEW
            'choice_group' => $this->choice_group,  // NEW
            'choice_options' => $this->when(
                $this->childOptions->isNotEmpty(),
                ClassFeatureResource::collection($this->childOptions)
            ),  // NEW: nested options
            // ...
        ];
    }
}
```

#### Expected API Response

```json
{
  "features": [
    {
      "id": 100,
      "level": 1,
      "feature_name": "Fighting Style",
      "is_choice_option": false,
      "choice_options": [
        { "id": 101, "feature_name": "Fighting Style: Archery", "is_choice_option": true, "parent_feature_id": 100 },
        { "id": 102, "feature_name": "Fighting Style: Defense", "is_choice_option": true, "parent_feature_id": 100 }
        // ... other options nested
      ]
    }
    // Fighting Style options no longer appear as top-level features
  ]
}
```

#### Acceptance Criteria

- [x] `is_choice_option` field added to features
- [ ] Choice options have `parent_feature_id` linking to parent
- [ ] Options can be nested under parent feature in response
- [ ] Feature count excludes choice options by default

#### Verification Notes (2025-11-29 - Second Pass)

**All choice option issues now resolved!**

Backend took a **cleaner approach** than flagging: **removed individual choice options entirely** from the feature list.

**What's working now:**
- ✅ `is_choice_option: true` for Fighting Style options (Fighter, Ranger, Paladin)
- ✅ `is_choice_option: true` for Pact Boon options (Warlock)
- ✅ **Totem Warrior**: Bear/Eagle/Wolf options **removed** - only parent "Totem Spirit (Path of the Totem Warrior)" shown
- ✅ **Champion L10**: Individual style options **removed** - only "Additional Fighting Style (Champion)" shown

```json
// Totem Warrior - RESOLVED (options removed, not flagged)
{
  "features": [
    { "feature_name": "Totem Spirit (Path of the Totem Warrior)", "level": 3 },
    { "feature_name": "Aspect of the Beast (Path of the Totem Warrior)", "level": 6 },
    { "feature_name": "Totemic Attunement (Path of the Totem Warrior)", "level": 14 }
    // Individual Bear/Eagle/Wolf options now in feature descriptions, not as separate features
  ]
}

// Champion Level 10 - RESOLVED (options removed)
{
  "features": [
    { "feature_name": "Additional Fighting Style (Champion)", "level": 10 }
    // Individual style options removed
  ]
}
```

**Frontend impact:** The pattern matching fallbacks in `useFeatureFiltering.ts` are no longer needed for these cases, but remain as defensive code.

#### Verification Notes (2025-11-26) [HISTORICAL]

**Previous state**: Fighting Style options were marked with `is_optional: true`, but this was a general flag. Now they properly use `is_choice_option: true`.

```json
// Current - all 7 Fighting Style features returned flat
{
  "features": [
    { "feature_name": "Fighting Style", "is_optional": false },
    { "feature_name": "Fighting Style: Archery", "is_optional": true },
    { "feature_name": "Fighting Style: Defense", "is_optional": true },
    // ... 4 more options
  ]
}
```

**Workaround**: Frontend can filter features where `feature_name` starts with parent feature name + `:`.

---

### 2.2 Multiclass Rules Mixed with Core Features

**Severity**: High
**Affected Endpoints**: All base class detail endpoints
**Status**: ✅ RESOLVED (via `is_multiclass_only` field)

#### Problem

Multiclass rules appear as Level 1 "features" alongside core class features:

```json
{
  "level": 1,
  "features": "Starting Wizard, Multiclass Wizard, Multiclass Features, Spellcasting"
}
```

Multiclassing is an **optional rule** (PHB p. 163) that many tables don't use. It shouldn't be presented as a primary feature.

#### D&D 5e Rules Reference

PHB p. 163:
> "Multiclassing allows you to gain levels in multiple classes... This rule is an **optional rule**. Ask your DM before using this rule."

#### Required Fix

**Add field to distinguish optional rules:**

```sql
ALTER TABLE class_features ADD COLUMN is_optional_rule BOOLEAN DEFAULT FALSE;

-- Mark multiclass features
UPDATE class_features
SET is_optional_rule = TRUE
WHERE feature_name LIKE '%Multiclass%';
```

**Update ClassResource to filter/separate:**

```php
class ClassResource extends JsonResource
{
    public function toArray($request): array
    {
        $features = $this->features;

        return [
            // Core features (default view)
            'features' => ClassFeatureResource::collection(
                $features->where('is_optional_rule', false)
                         ->where('is_choice_option', false)
            ),

            // Optional rule features (separate section)
            'optional_rule_features' => ClassFeatureResource::collection(
                $features->where('is_optional_rule', true)
            ),

            // ...
        ];
    }
}
```

#### Implementation (2025-11-26)

**Approach Taken**: Added `is_multiclass_only` field (named differently than proposed `is_optional_rule`)

```json
// Current API Response - Fighter features
{
  "features": [
    { "feature_name": "Starting Fighter", "is_optional": true, "is_multiclass_only": false },
    { "feature_name": "Multiclass Fighter", "is_optional": true, "is_multiclass_only": true },
    { "feature_name": "Multiclass Features", "is_optional": true, "is_multiclass_only": true },
    { "feature_name": "Second Wind", "is_optional": false, "is_multiclass_only": false },
    { "feature_name": "Fighting Style", "is_optional": false, "is_multiclass_only": false }
  ]
}
```

**Frontend Action**: Filter features where `is_multiclass_only === false` for primary display.

#### Acceptance Criteria

- [x] Multiclass features marked with `is_multiclass_only: true`
- [ ] Core features separated from optional rules in response (frontend responsibility)
- [ ] Progression table excludes optional rule features (still shows multiclass in features string)

---

### 2.3 Feature Count Inflation

**Status**: ❌ OUTSTANDING

**Severity**: High
**Affected Endpoints**: All class list and detail endpoints

#### Problem

Feature counts include choice options, making classes appear to have more features than they actually do:

| Class/Subclass | Current Count | Actual Features |
|----------------|---------------|-----------------|
| Fighter | 25+ | ~12 |
| Champion | 13 | 6 |
| Totem Warrior | 15 | 7 |

#### Required Fix

**Add computed field for countable features:**

```php
// In ClassResource or as model accessor
'feature_count' => $this->features
    ->where('is_choice_option', false)
    ->where('is_optional_rule', false)
    ->count(),

'total_feature_count' => $this->features->count(),  // includes everything
```

#### Acceptance Criteria

- [ ] `feature_count` excludes choice options and optional rules
- [ ] `total_feature_count` available if full count needed
- [ ] List endpoints show accurate feature counts

---

### 2.4 Progression Table Feature List Cluttered

**Severity**: High
**Affected Endpoints**: All class detail endpoints (computed.progression_table)
**Status**: ❌ OUTSTANDING

#### Problem

The progression table's "Features" column includes everything:

```json
{
  "level": 1,
  "features": "Starting Fighter, Multiclass Fighter, Multiclass Features, Second Wind, Fighting Style, Fighting Style: Archery, Fighting Style: Defense, Fighting Style: Dueling, Fighting Style: Great Weapon Fighting, Fighting Style: Protection, Fighting Style: Two-Weapon Fighting"
}
```

This is unreadable and incorrect - players don't gain all fighting styles.

#### Required Fix

When generating progression table, filter the features column:

```php
// In progression table generation
$featureNames = $levelFeatures
    ->where('is_choice_option', false)
    ->where('is_optional_rule', false)
    ->pluck('feature_name')
    ->join(', ');

// For choice features, show parent with indicator
// Result: "Fighting Style (choose 1), Second Wind"
```

#### Expected Output

```json
{
  "level": 1,
  "features": "Second Wind, Fighting Style"
}
```

Or with choice indicator:

```json
{
  "level": 1,
  "features": "Second Wind, Fighting Style (choose 1)"
}
```

#### Acceptance Criteria

- [ ] Progression table features exclude choice options
- [ ] Progression table features exclude optional rules
- [ ] Choice parent features optionally show "(choose N)" indicator

#### Verification Notes (2025-11-26)

**Current state**: Progression table still shows all features including Fighting Style options:

```json
// Current API Response - Fighter Level 1
{
  "level": 1,
  "proficiency_bonus": "+2",
  "features": "Starting Fighter, Second Wind, Fighting Style, Fighting Style: Archery, Fighting Style: Defense, Fighting Style: Dueling, Fighting Style: Great Weapon Fighting, Fighting Style: Protection, Fighting Style: Two-Weapon Fighting"
}
```

**Note**: Multiclass features ("Multiclass Fighter", "Multiclass Features") appear to be excluded from this string, which is partial progress.

---

## 3. Medium Priority Issues

### 3.1 Missing Subclass Entry Level

**Severity**: Medium
**Affected Endpoints**: Base class detail endpoints
**Status**: ✅ RESOLVED (2025-11-29)

#### Problem

There's no field indicating at what level a class gets to choose their subclass:

- Cleric: Level 1 (Divine Domain)
- Fighter: Level 3 (Martial Archetype)
- Wizard: Level 2 (Arcane Tradition)

#### D&D 5e Rules Reference

Different classes get subclasses at different levels:

| Class | Subclass Name | Level |
|-------|---------------|-------|
| Barbarian | Primal Path | 3 |
| Bard | Bard College | 3 |
| Cleric | Divine Domain | 1 |
| Druid | Druid Circle | 2 |
| Fighter | Martial Archetype | 3 |
| Monk | Monastic Tradition | 3 |
| Paladin | Sacred Oath | 3 |
| Ranger | Ranger Archetype | 3 |
| Rogue | Roguish Archetype | 3 |
| Sorcerer | Sorcerous Origin | 1 |
| Warlock | Otherworldly Patron | 1 |
| Wizard | Arcane Tradition | 2 |

#### Required Fix

**Add field to base classes:**

```sql
ALTER TABLE character_classes ADD COLUMN subclass_level TINYINT UNSIGNED NULL;

-- Populate
UPDATE character_classes SET subclass_level = 1 WHERE slug = 'cleric';
UPDATE character_classes SET subclass_level = 1 WHERE slug = 'sorcerer';
UPDATE character_classes SET subclass_level = 1 WHERE slug = 'warlock';
UPDATE character_classes SET subclass_level = 2 WHERE slug = 'druid';
UPDATE character_classes SET subclass_level = 2 WHERE slug = 'wizard';
UPDATE character_classes SET subclass_level = 3 WHERE slug IN ('barbarian', 'bard', 'fighter', 'monk', 'paladin', 'ranger', 'rogue');
```

**Also add subclass_name for display:**

```sql
ALTER TABLE character_classes ADD COLUMN subclass_name VARCHAR(50) NULL;

UPDATE character_classes SET subclass_name = 'Divine Domain' WHERE slug = 'cleric';
UPDATE character_classes SET subclass_name = 'Arcane Tradition' WHERE slug = 'wizard';
UPDATE character_classes SET subclass_name = 'Martial Archetype' WHERE slug = 'fighter';
-- etc.
```

#### Expected API Response

```json
{
  "data": {
    "name": "Wizard",
    "subclass_level": 2,
    "subclass_name": "Arcane Tradition",
    "subclasses": [...]
  }
}
```

#### Implementation (2025-11-29 - Second Pass)

**FULLY RESOLVED!** Both fields now populated:
- `subclass_level` - populated for all base classes
- `archetype` - used instead of `subclass_name` (clearer naming for API consumers)

```json
// Current API Response - All base classes verified (2025-11-29)
{ "name": "Fighter", "archetype": "Martial Archetype", "subclass_level": 3 }
{ "name": "Wizard", "archetype": "Arcane Tradition", "subclass_level": 2 }
{ "name": "Cleric", "archetype": "Divine Domain", "subclass_level": 1 }
{ "name": "Barbarian", "archetype": "Primal Path", "subclass_level": 3 }
{ "name": "Bard", "archetype": "Bard College", "subclass_level": 3 }
{ "name": "Druid", "archetype": "Druid Circle", "subclass_level": 2 }
{ "name": "Monk", "archetype": "Monastic Tradition", "subclass_level": 3 }
{ "name": "Paladin", "archetype": "Sacred Oath", "subclass_level": 3 }
{ "name": "Ranger", "archetype": "Ranger Archetype", "subclass_level": 3 }
{ "name": "Rogue", "archetype": "Roguish Archetype", "subclass_level": 3 }
{ "name": "Sorcerer", "archetype": "Sorcerous Origin", "subclass_level": 1 }
{ "name": "Warlock", "archetype": "Otherworldly Patron", "subclass_level": 1 }
{ "name": "Artificer", "archetype": "Artificer Specialist", "subclass_level": 3 }
```

#### Implementation (2025-11-29 - First Pass) [HISTORICAL]

**subclass_level was populated**, but `subclass_name` returned null.

#### Implementation (2025-11-26) [HISTORICAL]

**Schema Added**: Fields `subclass_level` and `subclass_name` existed but returned `null`:

```json
// Current API Response - Fighter
{
  "data": {
    "name": "Fighter",
    "subclass_level": null,   // Schema exists, needs data
    "subclass_name": null     // Schema exists, needs data
  }
}
```

#### Acceptance Criteria

- [x] `subclass_level` field added to base classes ✅
- [x] `archetype` field added (instead of `subclass_name`) ✅
- [x] `subclass_level` values populated for all 12+ base classes ✅
- [x] `archetype` values populated for all 12+ base classes ✅

---

### 3.2 Counters Not Consolidated

**Severity**: Medium
**Affected Endpoints**: Class detail endpoints (counters array)
**Status**: ❌ OUTSTANDING

#### Problem

Counters that scale with level appear as separate entries:

```json
{
  "counters": [
    { "level": 3, "counter_name": "Superiority Die", "counter_value": 4 },
    { "level": 7, "counter_name": "Superiority Die", "counter_value": 5 },
    { "level": 15, "counter_name": "Superiority Die", "counter_value": 6 }
  ]
}
```

#### Required Fix

**Option A**: Add computed consolidated counters

```php
// In ClassResource
'counters' => ClassCounterResource::collection($this->counters),
'consolidated_counters' => $this->getConsolidatedCounters(),

// Method
protected function getConsolidatedCounters(): array
{
    return $this->counters
        ->groupBy('counter_name')
        ->map(function ($group, $name) {
            return [
                'counter_name' => $name,
                'reset_timing' => $group->first()->reset_timing,
                'progression' => $group->map(fn ($c) => [
                    'level' => $c->level,
                    'value' => $c->counter_value
                ])->values()
            ];
        })
        ->values();
}
```

#### Expected API Response

```json
{
  "consolidated_counters": [
    {
      "counter_name": "Superiority Die",
      "reset_timing": "Short Rest",
      "progression": [
        { "level": 3, "value": 4 },
        { "level": 7, "value": 5 },
        { "level": 15, "value": 6 }
      ]
    }
  ]
}
```

#### Acceptance Criteria

- [ ] Counters with same name consolidated into progression array
- [ ] Original `counters` array preserved for backward compatibility

---

### 3.3 Irrelevant Progression Columns

**Severity**: Medium
**Affected Endpoints**: Subclass detail endpoints (computed.progression_table)
**Status**: ⚠️ PARTIALLY RESOLVED

#### Problem

Progression tables include columns specific to other subclasses:

- Oathbreaker (Paladin) shows `undying_sentinel` column (Oath of the Ancients feature)
- All Wizard subclasses show identical spell slot columns (correct, but redundant)

#### Required Fix

Generate progression table columns based on the specific class/subclass:

```php
// When building progression table
$columns = $this->getBaseColumns(); // level, proficiency, features

// Add class-specific columns
if ($this->hasSpellcasting()) {
    $columns = array_merge($columns, $this->getSpellSlotColumns());
}

// Add subclass-specific columns (only if this subclass has them)
foreach ($this->getClassCounters() as $counter) {
    if ($this->counters->contains('counter_name', $counter)) {
        $columns[] = $this->getCounterColumn($counter);
    }
}
```

#### Acceptance Criteria

- [ ] Subclass progression tables only show relevant columns
- [ ] Class-specific counters only appear if that class/subclass has them

#### Verification Notes (2025-11-26)

**Resolved**:
- ✅ Wizard: `arcane_recovery` column removed (only shows: level, proficiency_bonus, features, cantrips_known, spell_slots)
- ✅ Fighter: Only has level, proficiency_bonus, features (matches PHB)

**Outstanding**:
- ❌ Barbarian: Has `rage` but missing `rage_damage` (PHB has both "Rages" and "Rage Damage")
- ❌ Monk: Has `ki`, `wholeness_of_body` but should have `martial_arts`, `ki_points`, `unarmored_movement`
- ❌ Rogue: Has `stroke_of_luck` but should have `sneak_attack`

See **Appendix B** for full PHB column audit results.

---

### 3.4 Higher Levels Description Incorrect

**Severity**: Medium
**Affected Endpoints**: Subclass detail endpoints (computed.hit_points)
**Status**: ✅ RESOLVED

#### Problem

The `higher_levels.description` uses subclass name instead of class name:

```json
{
  "computed": {
    "hit_points": {
      "higher_levels": {
        "description": "1d6 (or 4) + your Constitution modifier per school of abjuration level after 1st"
      }
    }
  }
}
```

Should be "per **Wizard** level" since hit points are based on class levels.

#### Required Fix

```php
// In hit points computation
$className = $this->is_base_class ? $this->name : $this->parentClass->name;

$higherLevels['description'] = sprintf(
    '%s (or %d) + your Constitution modifier per %s level after 1st',
    $this->computed->hit_points->hit_die,
    $averageRoll,
    strtolower($className)
);
```

#### Implementation (2025-11-26)

Subclass hit points now correctly reference the parent class name:

```json
// Current API Response - Death Domain (Cleric subclass)
{
  "computed": {
    "hit_points": {
      "higher_levels": {
        "description": "1d8 (or 5) + your Constitution modifier per cleric level after 1st"
      }
    }
  }
}
```

#### Acceptance Criteria

- [x] Subclass hit points description references parent class name
- [x] Base class hit points description unchanged

---

### 3.5 Duplicate Description Content

**Status**: Not verified (low priority)

**Severity**: Medium
**Affected Endpoints**: Base class detail endpoints

#### Problem

Base class `description` duplicates the first trait's content:

```json
{
  "description": "Clad in the silver robes that denote her station...",
  "traits": [
    {
      "name": "Wizard",
      "description": "Clad in the silver robes that denote her station..."  // Same text
    }
  ]
}
```

#### Required Fix

**Option A**: Remove duplicate trait

```sql
DELETE FROM class_traits
WHERE name = class_name
AND description = (SELECT description FROM character_classes WHERE id = class_traits.class_id);
```

**Option B**: Keep both but mark trait as intro (frontend can hide)

```sql
ALTER TABLE class_traits ADD COLUMN is_intro_trait BOOLEAN DEFAULT FALSE;

UPDATE class_traits t
SET is_intro_trait = TRUE
WHERE t.sort_order = 0
AND EXISTS (
    SELECT 1 FROM character_classes c
    WHERE c.id = t.class_id
    AND t.description = c.description
);
```

#### Acceptance Criteria

- [ ] Intro trait either removed or marked distinctly
- [ ] No visible duplication in frontend display

---

## 4. Low Priority Issues

### 4.1 Spellcasting Ability Inheritance

Some subclasses don't inherit `spellcasting_ability` from parent.

**Fix**: Ensure subclass detail includes parent's spellcasting ability if not overridden.

### 4.2 Sort Order Inconsistencies

Features at the same level sometimes have non-sequential sort_order values.

**Fix**: Ensure sort_order is sequential within each level during import.

### 4.3 Source Reference in Descriptions

Many feature descriptions end with "Source: Player's Handbook (2014) p. X" which should be in a separate field.

**Fix**: Parse and move to `source` relationship during import.

---

## 5. Implementation Recommendations

### Migration Order

1. **Phase 1 - Data Fixes** (no schema changes)
   - Fix `hit_die: 0` values
   - Populate subclass descriptions
   - Remove non-canonical progression table columns (see Appendix B)

2. **Phase 2 - Schema Additions**
   - Add `is_choice_option`, `parent_feature_id`, `choice_group` to features
   - Add `is_optional_rule` to features
   - Add `subclass_level`, `subclass_name` to classes

3. **Phase 3 - Data Population**
   - Mark choice options
   - Mark optional rules
   - Populate subclass levels

4. **Phase 4 - API Updates**
   - Update ClassResource with new fields
   - Update ClassFeatureResource with nesting
   - Update progression table generation

### Testing Requirements

- [ ] All 12 base classes return valid data
- [ ] All subclasses return valid `hit_die` and `hit_points`
- [ ] Feature counts match expected values
- [ ] Progression tables are readable
- [ ] Backward compatibility maintained (no breaking changes to existing fields)

### Frontend Coordination

After backend changes, frontend will need to:

1. Update TypeScript types for new fields
2. Collapse choice options under parent features
3. Move multiclass rules to separate section
4. Show subclass entry level on base class pages

---

## 6. Appendix A: Affected Classes/Subclasses

### Subclasses with hit_die: 0

Run this query to find all affected:

```sql
SELECT c.slug, c.name, c.hit_die, p.name as parent_name, p.hit_die as parent_hit_die
FROM character_classes c
LEFT JOIN character_classes p ON c.parent_class_id = p.id
WHERE c.is_base_class = false AND c.hit_die = 0;
```

### Classes with Choice Features

| Class | Choice Feature | Options Count |
|-------|---------------|---------------|
| Fighter | Fighting Style | 6 |
| Paladin | Fighting Style | 4 |
| Ranger | Fighting Style | 3 |
| Champion | Additional Fighting Style | 6 |
| Totem Warrior | Totem Spirit | 3 |
| Totem Warrior | Aspect of the Beast | 3 |
| Totem Warrior | Totemic Attunement | 3 |
| Battle Master | Maneuvers | 16+ |
| Warlock | Eldritch Invocations | 20+ |
| Warlock | Pact Boon | 3 |

### Multiclass Features to Mark

```sql
SELECT id, class_id, feature_name
FROM class_features
WHERE feature_name LIKE '%Multiclass%'
ORDER BY class_id, level;
```

---

## 7. Appendix B: Canonical PHB Progression Table Columns

This appendix defines which columns should appear in each class's progression table based on the official Player's Handbook tables. This serves as a reference for the `ClassProgressionTableGenerator` to ensure API output matches official source material.

### Design Principles

1. **Match PHB presentation** - Columns should match what appears in the official class tables
2. **No formula columns** - Features like Arcane Recovery that use formulas (ceil(level/2)) belong in feature descriptions, not as numeric columns
3. **Subclasses inherit parent columns** - Subclass tables should use the same columns as their parent class

### Column Types

| Type | Description | Examples |
|------|-------------|----------|
| `level` | Character level (always present) | 1-20 |
| `proficiency_bonus` | Proficiency bonus (always present) | +2 to +6 |
| `features` | Features gained at each level (always present) | "Rage, Unarmored Defense" |
| `resource` | Numeric resource that scales with level | Rage (2→6), Ki (2→20) |
| `dice` | Damage dice that scale with level | Sneak Attack (1d6→10d6), Martial Arts (1d4→1d10) |
| `spell_slots` | Spell slots per level (spellcasters only) | 1st: 2, 2nd: 0, etc. |

### Canonical Columns by Class

#### Barbarian (PHB p. 47)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Features | features | |
| Rages | resource | 2 at L1 → 6 at L17, Unlimited at L20 |
| Rage Damage | resource | +2 at L1 → +4 at L16 |

**NOT included**: No other columns. Brutal Critical dice scaling is in feature description.

---

#### Bard (PHB p. 53)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Features | features | |
| Cantrips Known | resource | 2 at L1 → 4 at L10 |
| Spells Known | resource | 4 at L1 → 22 at L20 |
| Spell Slots (1st-9th) | spell_slots | Full caster progression |

**NOT included**: Bardic Inspiration die (in feature description).

---

#### Cleric (PHB p. 57)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Features | features | |
| Cantrips Known | resource | 3 at L1 → 5 at L10 |
| Spell Slots (1st-9th) | spell_slots | Full caster progression |

**NOT included**: Channel Divinity uses (in feature description), Destroy Undead CR (in feature description).

---

#### Druid (PHB p. 65)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Features | features | |
| Cantrips Known | resource | 2 at L1 → 4 at L10 |
| Spell Slots (1st-9th) | spell_slots | Full caster progression |

**NOT included**: Wild Shape details (in feature description).

---

#### Fighter (PHB p. 71)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Features | features | |

**NOT included**: The PHB Fighter table has NO numeric columns beyond the standard three. Action Surge uses, Indomitable uses, and Extra Attacks are all listed in the Features column.

---

#### Monk (PHB p. 77)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Martial Arts | dice | 1d4 at L1 → 1d10 at L17 |
| Ki Points | resource | — at L1, 2 at L2 → 20 at L20 |
| Unarmored Movement | resource | — at L1, +10 ft at L2 → +30 ft at L18 |
| Features | features | |

**Note**: Monk is unique in having Martial Arts die as a column.

---

#### Paladin (PHB p. 83)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Features | features | |
| Spell Slots (1st-5th) | spell_slots | Half caster progression, starts at L2 |

**NOT included**: Lay on Hands pool (formula: level × 5, belongs in feature description).

---

#### Ranger (PHB p. 90)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Features | features | |
| Spells Known | resource | — at L1, 2 at L2 → 11 at L19 |
| Spell Slots (1st-5th) | spell_slots | Half caster progression, starts at L2 |

---

#### Rogue (PHB p. 95)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Sneak Attack | dice | 1d6 at L1 → 10d6 at L19 |
| Features | features | |

---

#### Sorcerer (PHB p. 100)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Sorcery Points | resource | — at L1, 2 at L2 → 20 at L20 |
| Features | features | |
| Cantrips Known | resource | 4 at L1 → 6 at L10 |
| Spells Known | resource | 2 at L1 → 15 at L20 |
| Spell Slots (1st-9th) | spell_slots | Full caster progression |

---

#### Warlock (PHB p. 106)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Features | features | |
| Cantrips Known | resource | 2 at L1 → 4 at L10 |
| Spells Known | resource | 2 at L1 → 15 at L19 |
| Spell Slots | resource | 1 at L1 → 4 at L17 |
| Slot Level | resource | 1st at L1 → 5th at L9 |
| Invocations Known | resource | — at L1, 2 at L2 → 8 at L18 |

**Note**: Warlock has unique Pact Magic columns (Spell Slots, Slot Level) instead of standard spell slot progression.

---

#### Wizard (PHB p. 113)
| Column | Type | Notes |
|--------|------|-------|
| Level | level | |
| Proficiency Bonus | proficiency_bonus | |
| Features | features | |
| Cantrips Known | resource | 3 at L1 → 5 at L10 |
| Spell Slots (1st-9th) | spell_slots | Full caster progression |

**NOT included**:
- ❌ `arcane_recovery` - This is NOT in the PHB table. Arcane Recovery is a feature that lets you recover spell slot levels equal to ceil(wizard_level/2). The scaling is a formula in the feature description, not a tracked progression.

---

### Summary: Columns to Add/Remove

#### Columns to REMOVE (not in PHB tables)

| Class | Column | Reason |
|-------|--------|--------|
| Wizard | `arcane_recovery` | Formula-based feature, not in PHB table |
| Cleric | `channel_divinity` | Listed in Features column in PHB |
| Paladin | `lay_on_hands` | Formula-based (level × 5), not in PHB table |
| Paladin | `channel_divinity` | Listed in Features column in PHB |
| Paladin | `undying_sentinel` | Subclass-specific, only Oath of Ancients |
| Fighter | `action_surge` | Listed in Features column in PHB |
| Fighter | `indomitable` | Listed in Features column in PHB |
| Fighter | `second_wind` | Listed in Features column in PHB |

#### Columns to VERIFY (Updated 2025-11-26)

| Class | PHB Columns | Current API Columns | Status |
|-------|-------------|---------------------|--------|
| Fighter | level, prof, features | level, proficiency_bonus, features | ✅ Correct |
| Wizard | level, prof, features, cantrips, slots | level, proficiency_bonus, features, cantrips_known, spell_slots | ✅ Correct |
| Barbarian | level, prof, features, rages, rage_damage | level, proficiency_bonus, features, rage | ❌ Missing `rage_damage` |
| Monk | level, prof, martial_arts, ki, unarmored_movement, features | level, proficiency_bonus, features, ki, wholeness_of_body | ❌ Wrong columns |
| Rogue | level, prof, sneak_attack, features | level, proficiency_bonus, features, stroke_of_luck | ❌ Wrong column |
| Sorcerer | level, prof, sorcery_points, features, cantrips, spells, slots | *Not verified* | ⚠️ Needs check |
| Warlock | level, prof, features, cantrips, spells, slots, slot_level, invocations | *Not verified* | ⚠️ Needs check |

#### Column Discrepancies Requiring Fix

| Class | Remove | Add | Notes |
|-------|--------|-----|-------|
| Barbarian | - | `rage_damage` | PHB has "Rage Damage" column (+2→+4) |
| Monk | `wholeness_of_body` | `martial_arts`, `unarmored_movement` | Wholeness of Body is a L6 feature, not a scaling column |
| Rogue | `stroke_of_luck` | `sneak_attack` | Stroke of Luck is a L20 feature, not a scaling column |

---

### Implementation Notes

```php
// Example: Define canonical columns per class
const CANONICAL_COLUMNS = [
    'barbarian' => ['level', 'proficiency_bonus', 'features', 'rages', 'rage_damage'],
    'bard' => ['level', 'proficiency_bonus', 'features', 'cantrips_known', 'spells_known', ...SPELL_SLOTS],
    'cleric' => ['level', 'proficiency_bonus', 'features', 'cantrips_known', ...SPELL_SLOTS],
    'druid' => ['level', 'proficiency_bonus', 'features', 'cantrips_known', ...SPELL_SLOTS],
    'fighter' => ['level', 'proficiency_bonus', 'features'],  // No resource columns!
    'monk' => ['level', 'proficiency_bonus', 'martial_arts', 'ki_points', 'unarmored_movement', 'features'],
    'paladin' => ['level', 'proficiency_bonus', 'features', ...HALF_CASTER_SLOTS],
    'ranger' => ['level', 'proficiency_bonus', 'features', 'spells_known', ...HALF_CASTER_SLOTS],
    'rogue' => ['level', 'proficiency_bonus', 'sneak_attack', 'features'],
    'sorcerer' => ['level', 'proficiency_bonus', 'sorcery_points', 'features', 'cantrips_known', 'spells_known', ...SPELL_SLOTS],
    'warlock' => ['level', 'proficiency_bonus', 'features', 'cantrips_known', 'spells_known', 'spell_slots', 'slot_level', 'invocations_known'],
    'wizard' => ['level', 'proficiency_bonus', 'features', 'cantrips_known', ...SPELL_SLOTS],
];

// Subclasses inherit from parent
public function getProgressionColumns(): array
{
    $baseClass = $this->is_base_class ? $this : $this->parentClass;
    return self::CANONICAL_COLUMNS[$baseClass->slug] ?? self::DEFAULT_COLUMNS;
}
```

---

## Changelog

| Date | Author | Changes |
|------|--------|---------|
| 2025-11-26 | Claude | Initial proposal based on API verification audit |
| 2025-11-26 | Claude | Removed Arcane Recovery from Critical Issues (not a bug - design choice). Added Appendix B with canonical PHB column definitions. |
| 2025-11-26 | Claude | **Verification Update**: Audited current API against proposal. Marked 6 issues as RESOLVED (hit_die via effective_hit_die, subclass descriptions, is_multiclass_only flag, wizard/fighter columns, higher_levels description). Marked 1 issue IN PROGRESS (subclass_level schema exists, needs data). Updated Appendix B with actual column verification showing Barbarian/Monk/Rogue need fixes. |
