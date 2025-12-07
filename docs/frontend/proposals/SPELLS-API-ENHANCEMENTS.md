# Spells API Enhancement Proposals

**Date:** 2025-11-26
**Status:** Proposal
**API Endpoint:** `/api/v1/spells`
**Overall Assessment:** 🟢 PASS - Production-ready with optional enhancements

---

## Executive Summary

The Spells API is well-designed with excellent D&D 5e accuracy. All proposed changes are **optional enhancements** - the current implementation handles all core use cases for a D&D 5e Compendium frontend.

### Current Strengths
- Complete D&D 5e data accuracy
- Well-structured response format
- Powerful Meilisearch filtering
- Proper handling of edge cases (cantrips, rituals, reactions)
- Good relationships (classes, schools, sources)
- Excellent effects/scaling model

---

## Medium Priority Enhancements

### 1. Add Structured Material Component Fields

**Current State:**
```json
{
  "material_components": "diamonds worth 300 gp, which the spell consumes",
  "requires_material": true
}
```

**Proposed Enhancement:**
```json
{
  "material_components": "diamonds worth 300 gp, which the spell consumes",
  "requires_material": true,
  "material_cost_gp": 300,
  "material_consumed": true
}
```

**D&D Context:**
Some spells consume their material components (Revivify, Raise Dead, Heroes' Feast) while others reuse them (Chromatic Orb's diamond, Identify's pearl). Costly components significantly affect gameplay economics and spell preparation decisions.

**Use Cases Enabled:**
- Filter: "Show spells with costly components (> 0 gp)"
- Filter: "Show spells that consume materials"
- UI: Display cost warnings on spell cards
- Character building: Budget planning for spellcasters

**Implementation Notes:**
- Parse existing `material_components` text for gp values
- Look for "which the spell consumes" or "consumed" patterns
- Consider regex: `/(\d+)\s*gp/` and `/consume[sd]?/i`

---

### 2. Add Structured Area of Effect Field

**Current State:**
Area information exists only in description text.

**Proposed Enhancement:**
```json
{
  "area_of_effect": {
    "type": "sphere",
    "size": 20,
    "unit": "feet"
  }
}
```

**Supported Types:**
| Type | Example Spells |
|------|----------------|
| `sphere` | Fireball (20 ft), Cloudkill (20 ft) |
| `cone` | Burning Hands (15 ft), Cone of Cold (60 ft) |
| `line` | Lightning Bolt (100 ft x 5 ft) |
| `cube` | Thunderwave (15 ft), Stinking Cloud (20 ft) |
| `cylinder` | Flame Strike (10 ft radius, 40 ft high) |
| `emanation` | Spirit Guardians (15 ft from self) |
| `single` | Most targeted spells |
| `multiple` | Eldritch Blast, Magic Missile |

**D&D Context:**
Spell targeting and area types directly affect tactical combat decisions. A wizard choosing between Fireball (sphere) and Lightning Bolt (line) needs to understand the spatial implications.

**Use Cases Enabled:**
- Filter: "Show all cone spells" (for positioning tactics)
- Filter: "Show AoE spells with radius > 15 feet"
- UI: Display area type icons on spell cards
- Tactical planning: Compare spell coverage

---

### 3. ~~Add Structured Casting Time Fields~~ ✅ IMPLEMENTED

**Status:** ✅ Implemented 2025-11-26

**Current State (NOW WORKING):**
```json
{
  "casting_time": "1 action",
  "casting_time_type": "action"
}
```

**Verified Examples:**
| Spell | casting_time | casting_time_type |
|-------|--------------|-------------------|
| Fireball | "1 action" | `action` |
| Healing Word | "1 bonus action" | `bonus_action` |
| Shield | "1 reaction" | `reaction` |

**Casting Time Types:**
| Type | Examples |
|------|----------|
| `action` | Most spells |
| `bonus_action` | Healing Word, Misty Step, Hex |
| `reaction` | Shield, Counterspell, Feather Fall |
| `minute` | Identify (1 min), Detect Magic ritual (10 min) |
| `hour` | Find Familiar (1 hour), Awaken (8 hours) |
| `special` | Time Stop (instant but complex) |

**D&D Context:**
Action economy is crucial in D&D combat. Players often want "all bonus action spells" or "reaction spells for defense."

**Use Cases NOW ENABLED:**
- Filter: "Show all reaction spells" ✅
- Filter: "Show all bonus action spells" ✅
- UI: Display casting time badges by category ✅
- Character building: Balance action types in spell selection ✅

---

## Low Priority / Nice-to-Have

### 4. Add `searchable_options` to Meta Response

**Issue:**
Currently, querying `?per_page=1` doesn't return available filter fields in meta.

**Proposed Enhancement:**
```json
{
  "meta": {
    "current_page": 1,
    "total": 414,
    "searchable_options": {
      "filterable": ["level", "school_code", "concentration", "ritual", "class_slugs", "damage_types", "requires_verbal", "requires_somatic", "requires_material"],
      "sortable": ["name", "level", "school_code"]
    }
  }
}
```

**Benefit:**
API consumers can discover available filters without documentation.

---

### 5. Add Minimal Response Mode for List Views

**Proposed Enhancement:**
```
GET /api/v1/spells?fields=card
```

Returns only card-essential fields:
```json
{
  "id": 140,
  "slug": "fireball",
  "name": "Fireball",
  "level": 3,
  "school": { "code": "EV", "name": "Evocation" },
  "casting_time": "1 action",
  "concentration": false,
  "ritual": false
}
```

**Current Issue:**
Full response includes complete class descriptions (~5KB per spell), which aren't needed for list cards.

**Benefit:**
Reduced payload size for list pages, faster initial load.

---

### 6. Add Flattened `damage_types` Array

**Current State:**
```json
{
  "effects": [
    {
      "effect_type": "damage",
      "damage_type": { "id": 4, "code": "F", "name": "Fire" }
    }
  ]
}
```

**Proposed Enhancement:**
```json
{
  "damage_types": ["Fire"],
  "effects": [...]
}
```

**Benefit:**
Simplifies damage type filtering and display without parsing nested effects.

---

### 7. Add `reaction_trigger` Field

**For reaction spells only:**
```json
{
  "casting_time": "1 reaction",
  "reaction_trigger": "which you take when you are hit by an attack or targeted by the magic missile spell"
}
```

**Relevant Spells:**
- Shield: "when hit by attack or targeted by magic missile"
- Counterspell: "when you see a creature casting a spell"
- Feather Fall: "when you or a creature within 60 feet falls"
- Absorb Elements: "when you take acid, cold, fire, lightning, or thunder damage"

**Benefit:**
Displays trigger conditions on spell cards without parsing description.

---

## Future Considerations

### Computed Fields for Analysis

1. **`average_damage`** - Pre-calculated average for damage spells
   - Fireball: `8d6` → 28 average
   - Enables damage comparison tables

2. **`upcast_efficiency`** - Damage increase per spell slot level
   - Helps optimize spell slot usage
   - Shows which spells scale well

3. **`spell_tags`** - Gameplay role classification
   - Current: `["Ritual Caster", "Touch Spells"]`
   - Proposed additions: `["Damage", "Healing", "Control", "Buff", "Utility", "Summoning"]`

---

## Implementation Priority Matrix

| Enhancement | Effort | Impact | Priority |
|-------------|--------|--------|----------|
| Material cost/consumed fields | Medium | High | Medium |
| Area of effect structure | Medium | Medium | Medium |
| ~~Casting time structure~~ | ~~Low~~ | ~~Medium~~ | ✅ **DONE** |
| Searchable options in meta | Low | Low | Low |
| Minimal response mode | Medium | Medium | Low |
| Flattened damage_types | Low | Low | Low |
| Reaction trigger field | Low | Low | Low |

---

## Verification Checklist

The following was verified against official D&D 5e sources:

- [x] Fireball (PHB p.241): Level 3, 8d6 fire, 150 feet, 20-foot radius
- [x] Wish (PHB p.288): Level 9, V only, Conjuration
- [x] Shield (PHB p.275): Level 1, reaction, +5 AC
- [x] Detect Magic (PHB p.231): Ritual, Concentration, Divination
- [x] Chromatic Orb (PHB p.221): 50gp diamond (not consumed)
- [x] Revivify (PHB p.272): 300gp diamonds (consumed)
- [x] Identify (PHB p.252): 100gp pearl (not consumed), ritual

---

## Related Documentation

- Frontend filter implementation: `app/composables/useMeilisearchFilters.ts`
- Spells page: `app/pages/spells/index.vue`
- Backend API: `/Users/dfox/Development/dnd/importer`
