# Feats API Enhancement Proposals

**Date:** 2025-11-26
**Status:** Proposal
**API Endpoint:** `/api/v1/feats`
**Overall Assessment:** 🟢 PASS - Excellent data structure with comprehensive prerequisite modeling

---

## Executive Summary

The Feats API is **production-ready** with 138 feats covering PHB, XGE, TCE, and setting-specific books. The prerequisite system properly handles ability score minimums, proficiency requirements, and spellcasting requirements.

### Current Strengths
- Comprehensive prerequisite system (ability scores, proficiencies, custom text)
- Ability score increase modifiers properly structured
- Condition effects (advantage, disadvantage negation) modeled
- Proficiency grants (saving throws, tools) included
- Source references for all feats
- Feats with choices split into variants (Resilient → Resilient (Charisma), etc.)
- 138 total feats across multiple sourcebooks

### Design Decision: Variant Feats
Feats with choices are split into separate entries:
- Resilient → 6 variants (one per ability score)
- Ritual Caster → 6 variants (one per class spell list)
- Observant → 2 variants (Intelligence, Wisdom)

This simplifies character builders but increases total feat count.

---

## Logical Correctness Analysis ✅

### Feat Verification (PHB Reference)

| Feat | Page | Prerequisites | API Status |
|------|------|---------------|------------|
| Alert | 165 | None | ✅ Correct |
| Actor | 165 | None | ✅ Correct (+1 CHA) |
| Great Weapon Master | 167 | None | ✅ Correct |
| Heavy Armor Master | 167 | Heavy armor proficiency | ✅ Correct |
| Lucky | 167 | None | ✅ Correct |
| Sentinel | 169 | None | ✅ Correct |
| Sharpshooter | 170 | None | ✅ Correct |
| War Caster | 170 | Spellcasting ability | ✅ Correct |

### Prerequisite Types Verification

| Type | Example | API Modeling | Status |
|------|---------|--------------|--------|
| Ability Score | Ritual Caster (INT/WIS 13+) | `prerequisite_type: "AbilityScore"`, `minimum_value: 13` | ✅ |
| Proficiency | Heavy Armor Master | `prerequisite_type: "ProficiencyType"` | ✅ |
| Spellcasting | War Caster | `description: "The ability to cast at least one spell"` | ✅ |
| Race | Prodigy (Human only) | Check if modeled | ⚠️ |
| OR logic | Ritual Caster (INT OR WIS) | Same `group_id` for alternatives | ✅ |

### Ability Score Increase Verification

| Feat | Increase | PHB | API Status |
|------|----------|-----|------------|
| Actor | +1 CHA | 165 | ✅ `modifier_category: "ability_score"` |
| Alert | +5 initiative | 165 | ✅ `modifier_category: "initiative"` |
| Heavy Armor Master | +1 STR | 167 | ✅ Correct |
| Observant | +1 INT or WIS | 168 | ✅ Split into variants |
| Resilient | +1 chosen | 168 | ✅ Split into 6 variants |

---

## Structural Soundness Analysis ✅

### What Works Excellently

1. **Prerequisite System**
   ```json
   {
     "prerequisites_text": "Intelligence or Wisdom 13 or higher",
     "prerequisites": [
       {
         "prerequisite_type": "App\\Models\\AbilityScore",
         "prerequisite_id": 4,
         "minimum_value": 13,
         "ability_score": { "code": "INT", "name": "Intelligence" },
         "group_id": 1
       },
       {
         "prerequisite_type": "App\\Models\\AbilityScore",
         "prerequisite_id": 5,
         "minimum_value": 13,
         "ability_score": { "code": "WIS", "name": "Wisdom" },
         "group_id": 1
       }
     ]
   }
   ```
   - `group_id` allows OR logic (same group = alternatives)
   - `minimum_value` for ability score thresholds
   - `prerequisites_text` for human-readable display
   - Links to actual AbilityScore/ProficiencyType models

2. **Modifier System**
   ```json
   {
     "modifiers": [
       {
         "modifier_category": "ability_score",
         "ability_score": { "code": "STR", "name": "Strength" },
         "value": "1"
       },
       {
         "modifier_category": "initiative",
         "value": "5"
       }
     ]
   }
   ```
   - Consistent with race/class modifier structure
   - Supports various modifier categories

3. **Condition Effects**
   ```json
   {
     "conditions": [
       {
         "effect_type": "advantage",
         "description": "Constitution saving throws to maintain concentration"
       },
       {
         "effect_type": "negates_disadvantage",
         "description": "your ranged weapon attack rolls"
       }
     ]
   }
   ```
   - Models advantage on specific checks
   - Models negating disadvantage (Sharpshooter)

4. **Proficiency Grants**
   ```json
   {
     "proficiencies": [
       {
         "proficiency_type": "saving_throw",
         "proficiency_name": "saving throws using the chosen ability",
         "grants": true
       },
       {
         "proficiency_type": "tool",
         "proficiency_name": "type of artisan's tools",
         "is_choice": true
       }
     ]
   }
   ```

5. **Variant Feat Pattern**
   Feats with choices are pre-split:
   ```
   Resilient → Resilient (Charisma), Resilient (Constitution), etc.
   Ritual Caster → Ritual Caster (Bard), Ritual Caster (Cleric), etc.
   Observant → Observant (Intelligence), Observant (Wisdom)
   ```

   **Benefit:** Simplifies character builder logic
   **Tradeoff:** 138 total feats instead of ~50 base feats

---

## Feature Completeness Analysis ✅

### Essential Fields Present

| Field | Present | Notes |
|-------|---------|-------|
| name, slug | ✅ | All feats |
| description | ✅ | Full feat text |
| prerequisites_text | ✅ | Human-readable |
| prerequisites | ✅ | Structured data |
| modifiers | ✅ | Ability scores, initiative, etc. |
| proficiencies | ✅ | Saving throws, tools |
| conditions | ✅ | Advantage/disadvantage effects |
| sources | ✅ | Book and page references |
| tags | ✅ | (Usually empty) |

### Modifier Categories Found

| Category | Example Feat | Purpose |
|----------|--------------|---------|
| `ability_score` | Actor (+1 CHA) | ASI feats |
| `initiative` | Alert (+5) | Initiative bonus |
| `bonus` | Observant (+5 passive) | Miscellaneous bonuses |

### Condition Effect Types Found

| Effect Type | Example | Purpose |
|-------------|---------|---------|
| `advantage` | War Caster (concentration) | Grants advantage |
| `negates_disadvantage` | Sharpshooter (long range) | Removes disadvantage |

---

## Medium Priority Enhancements

### 1. Add `is_half_feat` Boolean

**D&D Context:** "Half feats" grant +1 to an ability score. Players often take these at odd ASI levels to round up ability scores.

**Proposed Enhancement:**
```json
{
  "is_half_feat": true,
  "ability_score_increase": { "code": "CHA", "value": 1 }
}
```

**Benefit:** Filter "Show half-feats for CHA" in character builders.

---

### 2. Add `has_prerequisites` Boolean

**Current State:** Must check if `prerequisites` array is non-empty.

**Proposed Enhancement:**
```json
{
  "has_prerequisites": true,
  "prerequisites_summary": "Heavy armor proficiency"
}
```

**Benefit:** Enables quick filtering for "feats anyone can take."

---

### 3. Add `parent_feat_slug` for Variants

**Current State:** Variants are separate entries with no explicit link.

**Proposed Enhancement:**
```json
{
  "name": "Resilient (Charisma)",
  "parent_feat_slug": "resilient",
  "variant_type": "ability_score",
  "is_variant": true
}
```

**Benefit:**
- Group variants in UI
- Show "Resilient" once with sub-options
- Reduce visual clutter in feat lists

---

### 4. Add `grants_spellcasting` Boolean

**D&D Context:** Some feats grant spellcasting ability (Magic Initiate, Ritual Caster, Fey Touched).

**Proposed Enhancement:**
```json
{
  "grants_spellcasting": true,
  "spellcasting_ability": { "code": "INT" },
  "spells_granted": [
    { "spell_slug": "find-familiar", "usage": "1/long rest" }
  ]
}
```

**Benefit:** Filter "feats that grant spells" for non-caster builds.

---

### 5. Add `feat_category` Field

**Proposed Categories:**
| Category | Examples |
|----------|----------|
| `combat` | Sentinel, Great Weapon Master, Sharpshooter |
| `spellcasting` | War Caster, Ritual Caster, Magic Initiate |
| `skill` | Skilled, Prodigy, Actor |
| `defensive` | Tough, Heavy Armor Master, Shield Master |
| `racial` | Prodigy (Human), Elven Accuracy |
| `dragonmark` | Aberrant Dragonmark |
| `general` | Lucky, Alert, Observant |

**Benefit:** Browse feats by category rather than alphabetically.

---

## Low Priority / Nice-to-Have

### 6. Add `synergies` Field

**Proposed Enhancement:**
```json
{
  "synergies": [
    { "feat_slug": "polearm-master", "reason": "Combines for opportunity attacks" },
    { "class_slug": "paladin", "reason": "Extra attacks for smites" }
  ]
}
```

**Benefit:** Helps new players discover feat combos (Sentinel + Polearm Master).

---

### 7. Add `repeatable` Boolean

**D&D Context:** Most feats can only be taken once. Some (like Elemental Adept) can be taken multiple times.

**Proposed Enhancement:**
```json
{
  "repeatable": true,
  "repeat_restrictions": "Choose a different damage type each time"
}
```

---

### 8. Consolidate Variant Feats (Alternative)

**Alternative to current pattern:** Instead of separate entries, use a single entry with choices:

```json
{
  "name": "Resilient",
  "ability_score_choice": {
    "type": "any",
    "count": 1,
    "options": ["STR", "DEX", "CON", "INT", "WIS", "CHA"]
  }
}
```

**Tradeoff:** More complex frontend logic but fewer total entries.

---

## Implementation Priority Matrix

| Enhancement | Effort | Impact | Priority |
|-------------|--------|--------|----------|
| Add `is_half_feat` boolean | Low | High | 🟡 Medium |
| Add `has_prerequisites` boolean | Low | Medium | 🟡 Medium |
| Add `parent_feat_slug` for variants | Low | Medium | 🟡 Medium |
| Add `grants_spellcasting` | Medium | Medium | 🟡 Medium |
| Add `feat_category` | Medium | Medium | 🟡 Medium |
| Add `synergies` field | High | Low | 🟢 Low |
| Add `repeatable` boolean | Low | Low | 🟢 Low |
| Consolidate variants | High | Medium | 🟢 Low |

---

## Verification Checklist

The following was verified against official D&D 5e sources:

**No Prerequisites:**
- [x] Alert (PHB p.165)
- [x] Great Weapon Master (PHB p.167)
- [x] Lucky (PHB p.167)
- [x] Sentinel (PHB p.169)
- [x] Sharpshooter (PHB p.170)

**With Prerequisites:**
- [x] Heavy Armor Master: Heavy armor proficiency (PHB p.167)
- [x] War Caster: Ability to cast spells (PHB p.170)
- [x] Ritual Caster: INT or WIS 13+ (PHB p.169)

**Half Feats (+1 ASI):**
- [x] Actor: +1 CHA (PHB p.165)
- [x] Heavy Armor Master: +1 STR (PHB p.167)
- [x] Observant: +1 INT or WIS (PHB p.168)
- [x] Resilient: +1 chosen (PHB p.168)

**Special Effects:**
- [x] Alert: +5 initiative
- [x] Observant: +5 passive Perception/Investigation
- [x] War Caster: Advantage on concentration saves

---

## Summary

The Feats API is **production-ready** with comprehensive modeling of:

- Prerequisites (ability scores, proficiencies, custom requirements)
- OR logic for alternative prerequisites (same `group_id`)
- Ability score increases and other modifiers
- Condition effects (advantage, disadvantage negation)
- Proficiency grants (saving throws, tools)

The variant feat pattern (Resilient → 6 entries) simplifies character builder logic but increases the total feat count from ~50 to 138.

The suggested enhancements focus on:
1. Better filtering (`is_half_feat`, `has_prerequisites`, `feat_category`)
2. Variant grouping (`parent_feat_slug`)
3. Spellcasting feat discovery (`grants_spellcasting`)

---

## Related Documentation

- Frontend feat pages: `app/pages/feats/`
- Feat filter store: `app/stores/featFilters.ts`
- Backend API: `/Users/dfox/Development/dnd/importer`
