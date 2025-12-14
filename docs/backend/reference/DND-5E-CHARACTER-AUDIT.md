# D&D 5e Character Endpoints Audit

**Date:** 2025-12-11
**Re-audited:** 2025-12-12
**Status:** Re-audit Complete - Infrastructure Analysis Added
**Epic:** [#495](https://github.com/dfox288/ledger-of-heroes/issues/495)

## Executive Summary

The character API implements many core D&D 5e mechanics well. A re-audit on 2025-12-12 found that **several items from the original audit already have infrastructure in place** - they just need wiring. The actual gap count is lower than originally estimated.

**Key Findings:**
- 5 items already fully implemented (remove from scope)
- 6 items have infrastructure ready (simpler fixes)
- ~15 items are genuine gaps requiring new code
- 1 data import issue (senses not imported from XML)

---

## What's Already Done Well

| Feature | Status |
|---------|--------|
| Multiclass spell slot calculation | Correct PHB p164 math |
| Third-caster support | Eldritch Knight, Arcane Trickster |
| Warlock pact magic | Separate from standard slots |
| Saving throw proficiency | From primary class |
| Skill proficiency + expertise | Full support |
| Passive skills | Perception, Investigation, Insight |
| Hit dice per class | Multiclass tracking |
| Death save tracking | 3 success/failure threshold |
| Short/long rest mechanics | Proper resource recovery |
| Feature use tracking | Reset timing support via `ClassCounter` (153 counters) |
| AC from equipped items | Light/medium/heavy calculation |
| Proficiency penalty warnings | Non-proficient armor/weapons |
| Racial ability bonuses | Fixed and chosen |
| Movement speeds | walk, fly, swim, climb, burrow (field exists) |
| Damage resistances/immunities | From race and feats |
| Condition immunities | From race and feats |
| Skill advantages | From features |
| Currency from inventory | Derived from equipment |
| Validation status | Missing fields flagged |
| Dangling references | Graceful handling |
| **Temp HP no-stacking** | Higher-wins logic in `HitPointService::modifyHp()` |
| **Hit dice recovery min 1** | `HitDiceService::recover()` enforces `max(1, ...)` |
| **Spell ritual/concentration fields** | `Spell` model has `is_ritual`, `needs_concentration` |
| **Active conditions tracking** | `CharacterCondition` model with level, source, duration |

---

## Items Removed from Scope (Already Implemented)

### ~~3.5 Temporary HP No-Stacking~~ ✅ DONE

**Location:** `HitPointService::modifyHp()` lines 296-303

```php
// Process temp HP if provided (higher-wins logic, except 0 always clears)
if ($tempHp !== null) {
    if ($tempHp === 0) {
        $currentTempHp = 0;
    } else {
        $currentTempHp = max($currentTempHp, $tempHp);
    }
}
```

### ~~4.1 Burrow Speed~~ ✅ DONE

**Location:** `CharacterStatsDTO::buildSpeed()` line 371

```php
return [
    'walk' => $race?->speed ?? 30,
    'fly' => $race?->fly_speed,
    'swim' => $race?->swim_speed,
    'climb' => $race?->climb_speed,
    'burrow' => null, // Field exists, just needs race data
];
```

### ~~Hit Dice Recovery Minimum~~ ✅ DONE

**Location:** `HitDiceService::recover()` line 131

```php
$quantity = max(1, (int) floor($totalMax / 2));
```

---

## Phase 1: Critical Combat Accuracy

### 1.1 Unarmored Defense Not Calculated Dynamically

**Status:** 🔶 INFRASTRUCTURE EXISTS - needs wiring

**Issue:** AC calculation only uses equipped armor or `armor_class_override`

**D&D Rules:**
- Barbarian: `10 + DEX + CON` (no armor)
- Monk: `10 + DEX + WIS` (no armor)
- Draconic Resilience (Sorcerer): `13 + DEX` (no armor)

**Infrastructure Found:**
- Special tags ARE stored: `ClassFeatureSpecialTag` contains `"Unarmored Defense: Constitution"` for Barbarian
- Monk has `"Unarmored Defense: Wisdom"` tag

**Fix:** Wire `CharacterStatCalculator::calculateArmorClass()` to:
1. Check if character has "Unarmored Defense" feature via `CharacterFeature`
2. Look up the special tag to determine which ability to use
3. Apply correct formula when unarmored

**Effort:** Low - data exists, just needs calculation logic

---

### 1.2 Exhaustion Not Tracked

**Status:** ❌ MISSING - needs implementation

**Issue:** No exhaustion level field exists on character

**D&D Rules:** 6 levels with cumulative effects:

| Level | Effect |
|-------|--------|
| 1 | Disadvantage on ability checks |
| 2 | Speed halved |
| 3 | Disadvantage on attacks/saves |
| 4 | HP max halved |
| 5 | Speed reduced to 0 |
| 6 | Death |

Long rest reduces exhaustion by 1 (with food/water).

**Fix:**
1. Add `exhaustion_level` (0-6) to Character model
2. Apply effects in `CharacterStatsDTO`
3. Update `RestService::longRest()` to reduce by 1

**Effort:** Medium

---

### 1.3 Class Resource Pools

**Status:** 🔶 MOSTLY DONE - minor gaps

**Infrastructure Found:**
- `ClassCounter` table has **153 counters** including:
  - Ki ✅
  - Sorcery Points ✅
  - Rage (2-6 uses by level) ✅
  - Lay on Hands ✅
  - Superiority Die ✅
  - Channel Divinity ✅
  - Wild Shape ✅
  - Action Surge ✅
  - Second Wind ✅
- `FeatureUseService` handles initialization and recovery
- `CharacterFeature.uses_remaining` tracks current uses

**Missing:**
- Bardic Inspiration counter not in `ClassCounter` (needs import fix)
- Die size progression (d6→d12) not tracked

**Fix:**
1. Add Bardic Inspiration to importer
2. Consider adding `die_size` field if needed for display

**Effort:** Low for counter, Medium if die size tracking needed

---

### 1.4 Initiative Modifiers from Features

**Status:** 🔶 INFRASTRUCTURE EXISTS - needs wiring

**Issue:** Initiative only uses DEX modifier

**Infrastructure Found:**
- `Modifier` table has `initiative` category
- Alert feat has modifier stored: `reference_type=Feat, value=5`
- Items also have initiative modifiers stored

**Current:** `CharacterStatsDTO::fromCharacter()` line 141 only uses DEX mod

**Fix:**
1. Query character's feat modifiers where `modifier_category = 'initiative'`
2. Sum values and add to initiative calculation

**Effort:** Low - modifiers already stored

---

### 1.5 Concentration Not Tracked

**Status:** ❌ MISSING - needs implementation

**Issue:** No tracking of active concentration spell

**D&D Rules:** Only one concentration spell at a time.

**Fix:** Add `concentrating_on_spell_slug` to Character model, add endpoint to set/clear.

**Effort:** Low-Medium

---

## Phase 2: Class Feature Completeness

### 2.1 Jack of All Trades (Bard) Not Implemented

**Status:** 🔶 FEATURE EXISTS - needs calculation

**Issue:** Half-proficiency for non-proficient ability checks not applied

**Infrastructure Found:**
- Feature exists: `ClassFeature` "Jack of All Trades" at Bard level 2 (class_id=10)

**Fix:**
1. Check if character has Jack of All Trades feature
2. Apply `floor(proficiencyBonus / 2)` to non-proficient skills and initiative

**Effort:** Low

---

### 2.2 Reliable Talent (Rogue 11) Not Flagged

**Status:** 🔶 FEATURE EXISTS - needs flag

**Issue:** No indication of minimum roll guarantee

**Infrastructure Found:**
- Feature exists: `ClassFeature` "Reliable Talent" at Rogue level 11 (class_id=83)

**Fix:** Add `has_reliable_talent: true` flag to proficient skill output when character has this feature

**Effort:** Low

---

### 2.3 Saving Throw Proficiency from Feats

**Status:** 🔶 MODIFIERS EXIST - needs wiring

**Issue:** Only checks primary class for saving throw proficiency

**Infrastructure Found:**
- `Modifier` table has categories: `saving_throw_str`, `saving_throw_dex`, etc.
- Resilient feat grants saving throw proficiency via modifiers

**Fix:** Also check `Modifier` table for character's feats with saving throw categories

**Effort:** Low

---

### 2.4 Always-Prepared Spells Not Distinguished

**Status:** 🔶 LOGIC EXISTS - needs verification

**Infrastructure Found:**
- `ClassFeature::getIsAlwaysPreparedAttribute()` exists
- Returns true for Cleric, Druid, Paladin subclass spells
- `preparation_status` has `always_prepared` value

**Issue:** No character spells exist to test (table empty)

**Fix:** Verify logic works when character spells are added

**Effort:** Low - just verification

---

### 2.5 Fighting Styles Not Applied

**Status:** ❌ TAGS EXIST - needs calculation

**Issue:** Fighting styles stored as features but not calculated into stats

**Infrastructure Found:**
- `ClassFeatureSpecialTag` contains tags like:
  - "fighting style archery"
  - "fighting style defense"
  - "fighting style dueling"
  - etc. (11 fighting styles stored)

**Fix:** Check equipped weapons/armor and apply relevant bonuses:
- Defense: +1 AC when wearing armor
- Archery: +2 ranged attack (needs weapon attack calculation first - see 3.1)

**Effort:** Medium - depends on weapon attack implementation

---

### 2.6 Ritual Casting Not Exposed

**Status:** 🔶 SPELL DATA EXISTS - needs exposure

**Issue:** Character spell list doesn't show ritual availability

**Infrastructure Found:**
- `Spell` model has `is_ritual` boolean field
- Exposed in `SpellResource` as `'ritual' => $this->is_ritual`

**Fix:** Include `is_ritual` in character spell list response

**Effort:** Low

---

## Phase 3: Equipment & Combat

### 3.1 Weapon Attack/Damage Not Calculated

**Status:** ❌ MISSING - needs implementation

**Issue:** No endpoint provides attack bonus or damage dice for equipped weapons

**D&D Rules:**
```
Attack Roll = d20 + ability mod + proficiency (if proficient) + magic bonus
Damage = weapon dice + ability mod + magic bonus + feature bonuses
```

**Fix:** Add weapon stats to equipped summary or separate endpoint

**Effort:** Medium-High

---

### 3.2 Attunement Slots Not Tracked

**Status:** ❌ MISSING - needs implementation

**Issue:** No tracking of magic item attunement

**Fix:** Add `is_attuned` boolean to `CharacterEquipment`, add `attunement_slots_used` to stats

**Effort:** Medium

---

### 3.3 Encumbrance Effects Not Applied

**Status:** ❌ MISSING - needs implementation

**Issue:** Carrying capacity calculated but effects not applied

**D&D Rules (Variant Encumbrance):**
- Over STR × 5 lbs: Speed reduced by 10 ft
- Over STR × 10 lbs: Speed reduced by 20 ft, disadvantage on STR/DEX/CON checks

**Fix:** Calculate total equipment weight, apply penalties

**Effort:** Medium

---

### 3.4 Darkvision/Senses Not Exposed

**Status:** ❌ DATA MISSING - needs import fix

**Issue:** `EntitySense` table has 0 records

**Infrastructure Found:**
- `HasSenses` trait exists on Race model
- `EntitySense` model exists
- `Sense` lookup table exists
- Race `toSearchableArray()` references senses

**Problem:** Senses are not being imported from XML source data

**Fix:**
1. Update race importer to parse and store senses
2. Add senses array to character stats from race data

**Effort:** Medium (import fix + exposure)

---

### ~~3.5 Temporary HP No-Stacking~~ ✅ REMOVED - Already Done

---

## Phase 4: Polish & Edge Cases

### ~~4.1 Burrow Speed~~ ✅ REMOVED - Already Done

---

### 4.2 Tool Proficiencies Not Categorized

**Status:** ❌ MISSING

**Fix:** Add tool category to proficiency type or group in API response

**Effort:** Low

---

### 4.3 Spell Component Tracking

**Status:** ❌ MISSING

**Fix:** Ensure spell data includes component cost and consumed flags

**Effort:** Low-Medium

---

### 4.4 Racial Spells by Level

**Status:** ❌ MISSING

**Fix:** Track level requirements on racial spells, filter by character level

**Effort:** Medium

---

### 4.5 Condition Effects on Stats

**Status:** ❌ INFRASTRUCTURE EXISTS - needs calculation

**Infrastructure Found:**
- `CharacterCondition` model exists with level, source, duration
- Characters can have active conditions

**Fix:** Add `active_effects` to stats showing penalties from active conditions

**Effort:** Medium

---

### 4.6 Languages from Items/Features

**Status:** ❌ MISSING

**Effort:** Low

---

### 4.7 XP-to-Level Mapping

**Status:** ❌ MISSING

**Fix:** Add `next_level_xp` and `xp_to_next_level` to character response

**Effort:** Low

---

### 4.8 Milestone Leveling Support

**Status:** ❌ MISSING

**Fix:** Add `leveling_method` enum to character

**Effort:** Low

---

### 4.9 Warlock Invocation Prerequisites

**Status:** ❌ MISSING

**Fix:** Validate prerequisites in available invocations endpoint

**Effort:** Medium

---

### 4.10 Prepared Spell Swap Timing

**Status:** ❌ LOW PRIORITY

**Fix:** Document in API or track last change time

**Effort:** Low

---

## Data Import Issues

### Senses Not Imported

**Issue:** `EntitySense` table has 0 records despite infrastructure existing

**Fix:** Update `RaceImporter` to parse `<sense>` elements from XML and store in `entity_senses` table

### Bardic Inspiration Counter Missing

**Issue:** `ClassCounter` has no entries for Bardic Inspiration uses

**Fix:** Add to `ClassXmlParser::addSpecialCaseCounters()` or fix XML parsing

---

## Implementation Priority (Revised)

### Quick Wins (Infrastructure Exists)

| Issue | Effort | Impact |
|-------|--------|--------|
| 1.1 Unarmored Defense | Low | High |
| 1.4 Initiative Modifiers | Low | Medium |
| 2.1 Jack of All Trades | Low | Medium |
| 2.2 Reliable Talent | Low | Low |
| 2.3 Save Prof from Feats | Low | Medium |
| 2.6 Ritual Casting | Low | Low |

### Medium Effort

| Issue | Effort | Impact |
|-------|--------|--------|
| 1.2 Exhaustion | Medium | High |
| 1.5 Concentration | Medium | Medium |
| 3.2 Attunement | Medium | Medium |
| 3.4 Senses (import) | Medium | Medium |

### Larger Features

| Issue | Effort | Impact |
|-------|--------|--------|
| 3.1 Weapon Attack/Damage | High | High |
| 2.5 Fighting Styles | Medium | Medium (depends on 3.1) |
| 3.3 Encumbrance | Medium | Low |

---

## Related Files

- `app/Models/Character.php`
- `app/Services/CharacterStatCalculator.php`
- `app/DTOs/CharacterStatsDTO.php`
- `app/Http/Resources/CharacterResource.php`
- `app/Http/Resources/CharacterStatsResource.php`
- `app/Services/RestService.php`
- `app/Services/FeatureUseService.php`
- `app/Services/HitPointService.php` - Temp HP logic
- `app/Services/HitDiceService.php` - Recovery minimum
- `app/Models/ClassFeatureSpecialTag.php` - Unarmored Defense tags
- `app/Models/Modifier.php` - Initiative/saving throw modifiers
- `app/Models/ClassCounter.php` - Resource tracking
