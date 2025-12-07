# D&D 5e Lookup Endpoints Audit Report

**Date:** 2025-11-26
**API Base:** `http://localhost:8080/api/v1/lookups/`
**Status:** Audit Complete

---

## Executive Summary

| Endpoint | Count | D&D Accuracy | Status |
|----------|-------|--------------|--------|
| `/ability-scores` | 6 | Perfect | PASS |
| `/damage-types` | 13 | Perfect | PASS |
| `/conditions` | 15 | Perfect | PASS |
| `/spell-schools` | 8 | Perfect | PASS |
| `/skills` | 18 | Perfect | PASS |
| `/languages` | 30 | Comprehensive | PASS |
| `/sizes` | 6 | Perfect | PASS |
| `/item-types` | 16 | Good | PASS |
| `/alignments` | 23 | Needs cleanup | IMPROVE |
| `/proficiency-types` | 84 | Excellent | PASS |
| `/sources` | 9 | Missing many | IMPROVE |

**Overall: EXCELLENT** - Core D&D 5e reference data is accurate and complete.

---

## Detailed Analysis

### 1. Ability Scores - PERFECT

**Count:** 6 | **PHB Reference:** p.173

| API Value | PHB Reference | Status |
|-----------|---------------|--------|
| STR - Strength | Correct | Pass |
| DEX - Dexterity | Correct | Pass |
| CON - Constitution | Correct | Pass |
| INT - Intelligence | Correct | Pass |
| WIS - Wisdom | Correct | Pass |
| CHA - Charisma | Correct | Pass |

**Assessment:** Complete and accurate. The canonical 6 ability scores per PHB.

---

### 2. Damage Types - PERFECT

**Count:** 13 | **PHB Reference:** p.196

All 13 official damage types present and correct:
- Acid, Bludgeoning, Cold, Fire, Force, Lightning, Necrotic, Piercing, Poison, Psychic, Radiant, Slashing, Thunder

---

### 3. Conditions - PERFECT

**Count:** 15 | **PHB Reference:** Appendix A (p.290-292)

All 14 standard conditions + Exhaustion with accurate descriptions matching PHB Appendix A:
- Blinded, Charmed, Deafened, Frightened, Grappled, Incapacitated, Invisible, Paralyzed, Petrified, Poisoned, Prone, Restrained, Stunned, Unconscious, Exhaustion

**Assessment:** Excellent! Descriptions match PHB word-for-word.

---

### 4. Spell Schools - PERFECT

**Count:** 8 | **PHB Reference:** p.203

All 8 schools of magic present:
- Abjuration, Conjuration, Divination, Enchantment, Evocation, Illusion, Necromancy, Transmutation

**Enhancement Opportunity:** Add `description` field with PHB school definitions (currently null).

---

### 5. Skills - PERFECT

**Count:** 18 | **PHB Reference:** p.174-179

All 18 skills with correct ability score associations:

| Skill | Ability | Status |
|-------|---------|--------|
| Acrobatics | DEX | Pass |
| Animal Handling | WIS | Pass |
| Arcana | INT | Pass |
| Athletics | STR | Pass |
| Deception | CHA | Pass |
| History | INT | Pass |
| Insight | WIS | Pass |
| Intimidation | CHA | Pass |
| Investigation | INT | Pass |
| Medicine | WIS | Pass |
| Nature | INT | Pass |
| Perception | WIS | Pass |
| Performance | CHA | Pass |
| Persuasion | CHA | Pass |
| Religion | INT | Pass |
| Sleight of Hand | DEX | Pass |
| Stealth | DEX | Pass |
| Survival | WIS | Pass |

**Assessment:** Excellent data structure with nested `ability_score` object.

---

### 6. Languages - COMPREHENSIVE

**Count:** 30 | **PHB Reference:** p.123

**Coverage:**
- 8 Standard Languages (Common, Dwarvish, Elvish, Giant, Gnomish, Goblin, Halfling, Orc)
- 8 Exotic Languages (Abyssal, Celestial, Draconic, Deep Speech, Infernal, Primordial, Sylvan, Undercommon)
- 4 Primordial Dialects (Aquan, Auran, Ignan, Terran)
- 2 Secret Languages (Druidic, Thieves' Cant)
- 5 Monster Languages (Aarakocra, Gnoll, Sahuagin, Sphinx, Worg)
- 3 Setting-Specific (Quori, Gith, Supernal)

**Assessment:** Excellent coverage including PHB core + supplements. `typical_speakers` and `script` fields are valuable additions.

---

### 7. Sizes - PERFECT

**Count:** 6 | **PHB Reference:** p.191

| Size | Code | Status |
|------|------|--------|
| Tiny | T | Pass |
| Small | S | Pass |
| Medium | M | Pass |
| Large | L | Pass |
| Huge | H | Pass |
| Gargantuan | G | Pass |

---

### 8. Item Types - GOOD

**Count:** 16 | **DMG Reference:** Various

| Type | Code | Reference |
|------|------|-----------|
| Ammunition | A | PHB p.146 |
| Melee Weapon | M | PHB p.146 |
| Ranged Weapon | R | PHB p.146 |
| Light Armor | LA | PHB p.144 |
| Medium Armor | MA | PHB p.144 |
| Heavy Armor | HA | PHB p.145 |
| Shield | S | PHB p.144 |
| Adventuring Gear | G | PHB p.148 |
| Trade Goods | $ | PHB p.157 |
| Potion | P | DMG p.187 |
| Rod | RD | DMG p.197 |
| Ring | RG | DMG p.191 |
| Wand | WD | DMG p.211 |
| Scroll | SC | DMG p.199 |
| Staff | ST | DMG p.201 |
| Wondrous Item | W | DMG p.212 |

**Assessment:** Good coverage of major item categories matching DMG magic item categories.

---

### 9. Alignments - NEEDS CLEANUP

**Count:** 23

**Issues Found:**
1. **Duplicate variations:** "typically chaotic evil" vs "Chaotic Evil" (inconsistent casing)
2. **Monster-specific alignments:** "Any Chaotic Alignment", "Any Evil Alignment" - useful for filtering but bloat lookup
3. **Percentage splits:** "Chaotic Good (75%) Neutral Evil (25%)" - monster-specific, not standard alignment

**Recommendations:**
- Normalize casing ("Typically Chaotic Evil")
- Consider separate `monster_alignments` endpoint or `is_variant` flag
- Add `is_core` boolean to distinguish PHB 9 from variants

---

### 10. Proficiency Types - EXCELLENT

**Count:** 84

**Categories Found:**
| Category | Count | PHB Reference |
|----------|-------|---------------|
| Armor | 4 | Light, Medium, Heavy, Shields (p.144) |
| Weapons - Simple | 14 | All PHB simple weapons (p.146) |
| Weapons - Martial | 24 | All PHB martial weapons (p.146) |
| Artisan Tools | 17 | All PHB artisan's tools (p.154) |
| Gaming Sets | 4 | Dice, Dragonchess, Cards, Three-Dragon Ante (p.154) |
| Musical Instruments | 10 | All PHB instruments (p.154) |
| Miscellaneous Tools | 6 | Disguise, Forgery, Herbalism, Navigator, Poisoner, Thieves' (p.154) |
| Vehicles | 2 | Land, Water (p.154) |
| Firearms | 1 | DMG variant (p.267) |

**Assessment:** Outstanding! Complete coverage with excellent `category`/`subcategory` structure.

---

### 11. Sources - INCOMPLETE

**Count:** 9

**Present:**
- DMG (2014), PHB (2014), MM (2014), XGE (2017), VGM (2016), TCE (2020), SCAG (2015), ERLW (2019), TWBTW (2021)

**Missing Major Sources:**
| Code | Book | Priority |
|------|------|----------|
| MTF | Mordenkainen's Tome of Foes | High |
| FTD | Fizban's Treasury of Dragons | Medium |
| GGR | Guildmasters' Guide to Ravnica | Medium |
| MOT | Mythic Odysseys of Theros | Medium |
| VRGR | Van Richten's Guide to Ravenloft | Medium |

**Data Issues:**
1. **XGE publication year** shows 2016, should be 2017
2. **MM description** incorrectly shows PHB description text (copy-paste error)

---

## Enhancement Priorities

### High Priority (Quick Fixes)

| Enhancement | Endpoint | Effort |
|-------------|----------|--------|
| Fix MM description (shows PHB text) | sources | Low |
| Fix XGE year (2016 to 2017) | sources | Low |
| Normalize alignment casing | alignments | Low |

### Medium Priority

| Enhancement | Endpoint | Effort |
|-------------|----------|--------|
| Add missing sourcebooks (MTF, FTD, etc.) | sources | Medium |
| Add `is_core` flag to alignments | alignments | Low |
| Add spell school descriptions | spell-schools | Low |

### Low Priority (Nice-to-Have)

| Enhancement | Endpoint | Effort |
|-------------|----------|--------|
| Add `/creature-types` endpoint | new | Medium |
| Add mount/vehicle item types | item-types | Low |

---

## Missing Endpoint: Creature Types

Consider adding `/api/v1/lookups/creature-types` with the 14 official creature types (MM p.6):

```json
[
  { "slug": "aberration", "name": "Aberration", "description": "Utterly alien beings..." },
  { "slug": "beast", "name": "Beast", "description": "Nonhumanoid creatures..." },
  { "slug": "celestial", "name": "Celestial", "description": "Creatures native to Upper Planes..." },
  { "slug": "construct", "name": "Construct", "description": "Made, not born..." },
  { "slug": "dragon", "name": "Dragon", "description": "Large reptilian creatures..." },
  { "slug": "elemental", "name": "Elemental", "description": "Creatures native to elemental planes..." },
  { "slug": "fey", "name": "Fey", "description": "Magical creatures tied to nature..." },
  { "slug": "fiend", "name": "Fiend", "description": "Wicked creatures native to Lower Planes..." },
  { "slug": "giant", "name": "Giant", "description": "Human-like but towering..." },
  { "slug": "humanoid", "name": "Humanoid", "description": "Main peoples of the world..." },
  { "slug": "monstrosity", "name": "Monstrosity", "description": "Frightening creatures not ordinary..." },
  { "slug": "ooze", "name": "Ooze", "description": "Gelatinous creatures..." },
  { "slug": "plant", "name": "Plant", "description": "Vegetable creatures..." },
  { "slug": "undead", "name": "Undead", "description": "Once-living creatures animated..." }
]
```

---

## Summary

The lookup endpoints are **production-ready** with excellent D&D 5e accuracy:

- **100% accurate:** ability-scores, damage-types, conditions, spell-schools, skills, sizes
- **Comprehensive:** languages (30), proficiency-types (84)
- **Minor cleanup needed:** alignments (casing), sources (missing books, data errors)

The data quality is impressive - conditions descriptions match PHB Appendix A word-for-word, and proficiency-types covers every weapon, tool, and instrument in the PHB with proper categorization.
