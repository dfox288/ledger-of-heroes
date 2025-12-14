# Database Architecture Audit: Ledger of Heroes

**Date:** 2025-12-11
**Auditor:** Database engineering review with D&D domain expertise
**Scope:** Full schema analysis including migrations, models, and relationships

---

## Executive Summary

The database demonstrates **strong architectural foundations** with sophisticated polymorphic patterns and comprehensive D&D 5e coverage. However, there are several areas where the schema could better reflect D&D mechanics and where database engineering practices could be improved.

**Overall Grade: B+** — Solid foundation with room for optimization

### Key Metrics
- **70+ tables** covering D&D 5e game content, character management, and infrastructure
- **12 polymorphic tables** using consistent `reference_type`/`reference_id` patterns
- **Slug-based character references** for export/import portability
- **Source tracking** on all character data for cascade operations

---

## Part 1: D&D Domain Modeling Analysis

### Strengths

#### 1. Spell Effect Modeling
The `spell_effects` table captures scaling mechanics (cantrip progression, upcasting) with `scaling_type`, `min_character_level`, and `min_spell_slot`. This is exactly how D&D 5e spells scale. The `projectile_count` and `projectile_per_level` fields properly model spells like Magic Missile.

#### 2. Class/Subclass Hierarchy
Self-referential `parent_class_id` elegantly handles subclasses. The `class_level_progression` table correctly captures spell slot progression per level.

#### 3. Monster Action Types
Separating `monster_actions`, `monster_legendary_actions`, and `monster_traits` correctly models the distinct D&D action economy (standard actions vs legendary vs passive traits).

#### 4. Equipment Choice System
The `entity_items` → `equipment_choice_items` pattern correctly handles D&D's complex starting equipment choices ("choose a martial weapon OR two handaxes").

#### 5. Character Data Flow
The cascade cleanup on race/background switches shows mature handling of D&D's complex interdependencies. Source tracking enables "clear all race-granted features, then populate from new race" operations.

---

### D&D Domain Issues

#### Issue 1: Monster Traits Aren't Polymorphic (Inconsistency)

**Problem:** Races, Classes, and Backgrounds use `entity_traits` (polymorphic), but Monsters use a separate `monster_traits` table.

**D&D Reality:** Monster traits and racial traits often describe similar things (damage resistances, special senses, innate spellcasting). A Tiefling's "Hellish Resistance" and a Devil's "Devil's Sight" are conceptually identical.

**Impact:**
- Can't query "all entities with fire resistance" uniformly
- Duplicate trait definitions when the same creature is both a race and monster (e.g., Drow)

**Recommendation:** Migrate `monster_traits` to `entity_traits` with `reference_type='App\Models\Monster'`. Keep `monster_actions` and `monster_legendary_actions` separate (those are combat-specific).

**Related GitHub Issue:** [#502](https://github.com/dfox288/ledger-of-heroes/issues/502)

---

#### Issue 2: Missing Creature Type Normalization

**Current:** `monsters.type` is a string field (`"humanoid"`, `"fiend"`, `"undead"`).

**D&D Reality:** Creature types are a fixed set with specific rules:
- Aberrations are immune to certain spells
- Undead can't be charmed (usually)
- Constructs don't need to eat/sleep
- Humanoids can be targeted by Hold Person

**Impact:**
- No way to query "spells that affect humanoids"
- Can't build creature type filtering with proper tags

**Recommendation:** Create `creature_types` lookup table:
```sql
CREATE TABLE creature_types (
    id INT PRIMARY KEY,
    name VARCHAR(50),            -- 'undead', 'construct', etc.
    typically_immune_to_poison BOOLEAN DEFAULT FALSE,
    typically_immune_to_charmed BOOLEAN DEFAULT FALSE,
    requires_sustenance BOOLEAN DEFAULT TRUE
);

ALTER TABLE monsters
ADD COLUMN creature_type_id INT REFERENCES creature_types(id);
```

**Related GitHub Issue:** [#503](https://github.com/dfox288/ledger-of-heroes/issues/503)

---

#### Issue 3: Currency/Wealth Not Properly Modeled

**Current:** `items.cost_cp` stores everything in copper pieces (good!), but characters have no currency tracking.

**D&D Reality:** Characters carry gold, silver, electrum, platinum. Encumbrance rules apply. Wealth management is core to D&D.

**Missing:**
- No `character_currency` table
- No wealth tracking on characters
- `starting_wealth_dice` on classes exists but nowhere to store the result

**Recommendation:**
```sql
CREATE TABLE character_currency (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    character_id BIGINT REFERENCES characters(id) ON DELETE CASCADE,
    cp INT UNSIGNED DEFAULT 0,
    sp INT UNSIGNED DEFAULT 0,
    ep INT UNSIGNED DEFAULT 0,
    gp INT UNSIGNED DEFAULT 0,
    pp INT UNSIGNED DEFAULT 0,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    UNIQUE KEY (character_id)
);
```

**Related GitHub Issue:** [#504](https://github.com/dfox288/ledger-of-heroes/issues/504)

---

#### Issue 4: Condition Severity Levels Missing

**Current:** `entity_conditions` links entities to conditions but doesn't capture severity or stacking.

**D&D Reality:** Some conditions have levels (Exhaustion 1-6). Some stack (multiple frightened sources). Some replace (different paralysis durations).

**Impact:** Can't properly track exhaustion levels or condition stacking rules.

**Recommendation:**
```sql
ALTER TABLE conditions
ADD COLUMN max_severity_level INT UNSIGNED DEFAULT 1,
ADD COLUMN is_stackable BOOLEAN DEFAULT FALSE;
```

**Related GitHub Issue:** [#505](https://github.com/dfox288/ledger-of-heroes/issues/505) (combined in minor cleanup)

---

#### Issue 5: Spell Component Costs Not Normalized

**Current:** `spells.material_components` is text with embedded costs ("a gem worth 100gp").

**D&D Reality:** Material component costs are mechanically significant:
- Some spells consume components (Revivify)
- Some just require them (Identify)
- Focus/pouch substitution rules depend on cost

**Current Workaround:** Accessors `material_cost_gp` and `material_consumed` parse the text (~90% accuracy).

**Recommendation:** Add normalized fields and populate during import:
```sql
ALTER TABLE spells
ADD COLUMN material_cost_gp INT UNSIGNED DEFAULT NULL,
ADD COLUMN material_consumed BOOLEAN DEFAULT FALSE;
```

**Related GitHub Issue:** [#505](https://github.com/dfox288/ledger-of-heroes/issues/505) (combined in minor cleanup)

---

#### Issue 6: Multiclass Spell Slot Calculation Missing Caster Level

**Current:** `multiclass_spell_slots` lookup table exists but characters don't store calculated caster level.

**D&D Reality:** Multiclass spellcasters combine spell slots in complex ways:
- Full casters: add all levels
- Half casters: add half (rounded down)
- Third casters: add third (rounded down)
- Pact Magic (Warlock): completely separate

**Impact:** Every time you render spell slots, you recalculate caster level.

**Recommendation:** Cache computed values:
```sql
ALTER TABLE characters
ADD COLUMN calculated_caster_level TINYINT UNSIGNED DEFAULT 0,
ADD COLUMN pact_magic_level TINYINT UNSIGNED DEFAULT 0;
```

**Related GitHub Issue:** [#505](https://github.com/dfox288/ledger-of-heroes/issues/505) (combined in minor cleanup)

---

#### Issue 7: Action Economy Not Tracked

**Current:** Features track `uses_remaining` but not action cost.

**D&D Reality:** Features cost different actions:
- Action (Attack, Dash)
- Bonus Action (Cunning Action, Healing Word)
- Reaction (Shield, Counterspell)
- Free (Drop item)
- Movement (part of speed)

**Impact:** UI can't display "bonus action abilities" vs "action abilities" without parsing descriptions.

**Recommendation:** Add `action_cost` enum to `class_features` and `optional_features`:
```php
enum ActionCost: string {
    case ACTION = 'action';
    case BONUS_ACTION = 'bonus_action';
    case REACTION = 'reaction';
    case FREE = 'free';
    case NONE = 'none'; // passive
}
```

**Related GitHub Issue:** [#505](https://github.com/dfox288/ledger-of-heroes/issues/505) (combined in minor cleanup)

---

## Part 2: Database Engineering Analysis

### Strengths

1. **Polymorphic Pattern Consistency** — 12 `entity_*` tables share identical `reference_type`/`reference_id` patterns
2. **Proper Indexing** — All polymorphic tables indexed on `(reference_type, reference_id)`
3. **Cascade Deletes** — Foreign keys use `ON DELETE CASCADE` appropriately
4. **Slug-Based Portability** — Character data uses slugs for export/import support
5. **Source Tracking** — Every character item tracks its origin (race/class/background)
6. **Transaction Safety** — Row locking prevents concurrent modification issues

---

### Database Engineering Issues

#### Issue 1: Polymorphic Type Validation (HIGH PRIORITY)

**Problem:** `reference_type` is VARCHAR(255) with no database constraint.

**Risk:** Application typo creates orphaned data:
```sql
INSERT INTO entity_traits (reference_type, reference_id, name)
VALUES ('App\Models\Races', 1, 'Bad Trait');  -- 'Races' not 'Race'
```

**Recommendation:** Add CHECK constraint:
```sql
ALTER TABLE entity_traits
ADD CONSTRAINT chk_reference_type CHECK (
    reference_type IN (
        'App\\Models\\Race',
        'App\\Models\\CharacterClass',
        'App\\Models\\Background',
        'App\\Models\\Monster'
    )
);
```

Or create a `morph_types` lookup table and use FK constraints.

**Related GitHub Issue:** [#500](https://github.com/dfox288/ledger-of-heroes/issues/500)

---

#### Issue 2: Choice Logic Fragmentation (HIGH PRIORITY)

**Problem:** Four different choice-tracking patterns across tables:

| Table | Fields |
|-------|--------|
| `entity_proficiencies` | `is_choice`, `choice_group`, `choice_option`, `quantity` |
| `entity_spells` | `is_choice`, `choice_count`, `choice_group`, `max_level` |
| `entity_items` | `is_choice`, `choice_group`, `choice_option`, `choice_description` |
| `entity_languages` | `is_choice`, `choice_group`, `choice_option`, `quantity` |

**Impact:** Application must interpret choices differently per table. No shared ChoiceGroup model.

**Recommendation:** Create unified choice tracking:
```sql
CREATE TABLE choice_groups (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    reference_type VARCHAR(255) NOT NULL,
    reference_id BIGINT UNSIGNED NOT NULL,
    group_name VARCHAR(100),
    choose_count TINYINT UNSIGNED DEFAULT 1,
    choice_type ENUM('proficiency', 'spell', 'language', 'equipment', 'ability_score', 'feat'),
    INDEX idx_reference (reference_type, reference_id)
);

CREATE TABLE choice_options (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    choice_group_id BIGINT REFERENCES choice_groups(id) ON DELETE CASCADE,
    option_index TINYINT UNSIGNED NOT NULL,
    entity_type VARCHAR(255) NOT NULL,  -- What's being offered
    entity_id BIGINT UNSIGNED,          -- NULL = "any of this type"
    quantity TINYINT UNSIGNED DEFAULT 1,
    description TEXT,
    INDEX idx_group (choice_group_id)
);
```

**Related GitHub Issue:** [#501](https://github.com/dfox288/ledger-of-heroes/issues/501)

---

#### Issue 3: EntityLanguage Relationship Naming (LOW)

**Problem:** `EntityLanguage.php` uses `entity()` method while all other polymorphic models use `reference()`.

**Location:** `app/Models/EntityLanguage.php:51`

**Impact:** Confusing API, breaks consistency for dynamic polymorphic queries.

**Fix:** Rename to `reference()`:
```php
public function reference(): MorphTo {
    return $this->morphTo(null, 'reference_type', 'reference_id');
}
```

**Related GitHub Issue:** [#505](https://github.com/dfox288/ledger-of-heroes/issues/505) (combined in minor cleanup)

---

#### Issue 4: CharacterTrait Bidirectional DataTable Link (MEDIUM)

**Problem:** `entity_traits` has both:
- `entity_data_table_id` FK (BelongsTo one table)
- `dataTables()` MorphMany (owning many tables)

**Location:** `app/Models/CharacterTrait.php:37-46`

**Ambiguity:** Which is authoritative? A trait can both POINT TO a table AND HAVE CHILD tables?

**Recommendation:** Choose one pattern and document it:
- If traits embed tables → use only `entity_data_table_id`
- If tables belong to traits → use only MorphMany
- If truly bidirectional → document the semantics clearly

**Related GitHub Issue:** [#505](https://github.com/dfox288/ledger-of-heroes/issues/505) (combined in minor cleanup)

---

#### Issue 5: Nullable FK Inconsistency (MEDIUM)

**Pattern:**
| Table | FK | Nullable? | Purpose |
|-------|-----|-----------|---------|
| `entity_spells` | `spell_id` | YES | "Choose any spell" |
| `entity_languages` | `language_id` | YES | "Choose any language" |
| `entity_sources` | `source_id` | NO | Must have source |

**Problem:** No documented rule for when NULL means "unrestricted choice" vs "broken link".

**Recommendation:** Add `is_unrestricted_choice` boolean or document the convention clearly.

**Related GitHub Issue:** [#501](https://github.com/dfox288/ledger-of-heroes/issues/501) (combined with choice logic)

---

#### Issue 6: Dead Listener Code (LOW)

**Location:** `PopulateCharacterAbilities` listener references `race_id` and `background_id`.
**Current Schema:** Character uses `race_slug` and `background_slug`.

**Impact:** Listener probably never fires correctly. Cascade happens in CharacterController instead.

**Recommendation:** Audit listeners and remove dead code.

**Related GitHub Issue:** [#505](https://github.com/dfox288/ledger-of-heroes/issues/505) (combined in minor cleanup)

---

## Part 3: Consolidation Opportunities

### Consolidate: Monster Polymorphic Integration

**Current State:**
- `monster_traits` (separate table)
- `monster_actions` (separate table)
- `monster_legendary_actions` (separate table)

**Recommendation:**
- Keep `monster_actions` and `monster_legendary_actions` — combat-specific
- Migrate `monster_traits` to `entity_traits` — enables unified queries

---

### Consolidate: Damage/Effect Type Unification

**Current:** Damage interactions spread across:
- `entity_modifiers.modifier_category = 'damage_resistance'`
- `entity_conditions.effect_type`
- Monster text descriptions

**Recommendation:** Create unified damage interaction table:
```sql
CREATE TABLE entity_damage_interactions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    reference_type VARCHAR(255) NOT NULL,
    reference_id BIGINT UNSIGNED NOT NULL,
    damage_type_id INT REFERENCES damage_types(id),
    interaction_type ENUM('immunity', 'resistance', 'vulnerability', 'absorption'),
    condition_text VARCHAR(255) NULL,  -- "from nonmagical weapons"
    INDEX idx_reference (reference_type, reference_id),
    INDEX idx_damage_type (damage_type_id)
);
```

---

## Part 4: Expansion Recommendations

### 1. Encounter/Combat Tracking (Future Feature)

If you plan to support live play:
```sql
CREATE TABLE encounters (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    created_by BIGINT REFERENCES users(id),
    is_active BOOLEAN DEFAULT TRUE,
    timestamps
);

CREATE TABLE encounter_participants (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    encounter_id BIGINT REFERENCES encounters(id) ON DELETE CASCADE,
    participant_type VARCHAR(255),  -- Character or Monster
    participant_id BIGINT UNSIGNED,
    initiative TINYINT,
    current_hp INT,
    max_hp INT,
    is_active BOOLEAN DEFAULT TRUE,
    conditions JSON,
    INDEX idx_encounter (encounter_id)
);
```

### 2. Homebrew Content Namespace

**Current:** `full_slug` format is `source:name` (e.g., `phb:fireball`).

**Recommendation:** Formalize homebrew namespace:
- Official: `phb:`, `xge:`, `tce:`
- Homebrew: `homebrew-{user_id}:` or `custom:`

Add `is_homebrew` boolean to major entity tables for filtering.

### 3. Version/Errata Tracking

D&D content gets errata. Consider:
```sql
ALTER TABLE spells
ADD COLUMN errata_version VARCHAR(20) DEFAULT NULL,
ADD COLUMN errata_date DATE DEFAULT NULL;
```

---

## Priority Matrix

| Priority | Issue | Effort | Impact | GitHub Issue |
|----------|-------|--------|--------|--------------|
| **HIGH** | Polymorphic type validation | Low | Prevents data corruption | [#500](https://github.com/dfox288/ledger-of-heroes/issues/500) |
| **HIGH** | Choice logic unification | Medium | Simplifies app logic | [#501](https://github.com/dfox288/ledger-of-heroes/issues/501) |
| **MEDIUM** | Monster traits → polymorphic | Medium | Enables unified queries | [#502](https://github.com/dfox288/ledger-of-heroes/issues/502) |
| **MEDIUM** | Creature type normalization | Low | Better filtering | [#503](https://github.com/dfox288/ledger-of-heroes/issues/503) |
| **MEDIUM** | Currency tracking | Low | Complete character sheet | [#504](https://github.com/dfox288/ledger-of-heroes/issues/504) |
| **LOW** | Minor cleanup (naming, dead code, enums) | Trivial-Low | Code consistency | [#505](https://github.com/dfox288/ledger-of-heroes/issues/505) |

---

## Appendix: Table Inventory

### Core Entity Tables
- `spells`, `spell_effects`
- `monsters`, `monster_actions`, `monster_legendary_actions`, `monster_traits`
- `classes`, `class_features`, `class_level_progression`, `class_counters`
- `races`
- `backgrounds`
- `feats`
- `items`, `item_abilities`
- `optional_features`

### Polymorphic Tables (entity_*)
- `entity_sources` - Publication citations
- `entity_traits` - Racial/class/background traits
- `entity_modifiers` - Stat bonuses, resistances
- `entity_proficiencies` - Granted proficiencies
- `entity_spells` - Innate/granted spells
- `entity_languages` - Language grants
- `entity_conditions` - Condition immunities
- `entity_senses` - Special senses
- `entity_items` - Starting equipment
- `entity_prerequisites` - Requirements
- `entity_data_tables` - Structured data
- `entity_saving_throws` - Save requirements

### Character Tables
- `characters`
- `character_classes` - Multiclass tracking
- `character_spells` - Known/prepared spells
- `character_proficiencies` - Granted proficiencies
- `character_languages` - Known languages
- `character_features` - Tracked features
- `character_equipment` - Inventory
- `character_ability_scores` - Racial bonuses
- `character_conditions` - Active conditions
- `character_spell_slots` - Slot tracking
- `feature_selections` - Optional feature choices

### Lookup Tables
- `sources`, `ability_scores`, `sizes`, `damage_types`
- `spell_schools`, `conditions`, `languages`, `senses`, `skills`
- `item_types`, `item_properties`, `proficiency_types`

---

## Conclusion

The schema is well-architected for D&D 5e content management. The polymorphic patterns, slug-based character references, and source tracking demonstrate thoughtful design.

**Top 3 Priorities:**
1. Add database constraints for polymorphic types
2. Unify choice tracking patterns
3. Migrate monster traits to polymorphic pattern

Implementation of these recommendations will improve data integrity, simplify application logic, and enable more powerful cross-entity queries.
