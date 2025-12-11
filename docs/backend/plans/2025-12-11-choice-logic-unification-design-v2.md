# Choice Logic Unification Design (Revised)

**Date:** 2025-12-11
**Status:** Approved (Revision 2)
**GitHub Issue:** [#501](https://github.com/dfox288/ledger-of-heroes/issues/501)
**Supersedes:** `2025-12-11-choice-logic-unification-design.md`

---

## Overview

Unify fragmented choice logic across **5 polymorphic tables** into a consistent, extensible system.

### Goals

1. **Single source of truth** - All choice definitions in `entity_choice_groups` + `choice_group_options`
2. **Consistent API** - One pattern for reading/resolving choices across all types
3. **Simplified handlers** - Replace 4 handlers with resolvers, update 1 handler
4. **Preserve API contract** - Existing `PendingChoice` DTO unchanged

### Approach

- Clean break: new tables, reimport data, remove old columns
- Resolvers replace handlers for 4 choice types
- SpellChoiceHandler updated to read from new tables (for subclass features)
- OptionalFeatureChoiceHandler unchanged (uses class_counters, out of scope)

---

## Tables with `is_choice` (Current State)

| Table | Choice Columns | Handler |
|-------|---------------|---------|
| `entity_proficiencies` | `is_choice`, `choice_group`, `choice_option`, `quantity` | ProficiencyChoiceHandler |
| `entity_languages` | `is_choice`, `choice_group`, `choice_option`, `quantity` | LanguageChoiceHandler |
| `entity_modifiers` | `is_choice`, `choice_count`, `choice_constraint` | AbilityScoreChoiceHandler |
| `entity_spells` | `is_choice`, `choice_count`, `choice_group`, `max_level`, `school_id`, `class_id`, `is_ritual_only` | SpellChoiceHandler (subclass features) |
| `entity_items` | `is_choice`, `choice_group`, `choice_option`, `choice_description` | EquipmentChoiceHandler |

---

## Database Schema

### New Tables

```sql
-- What choices an entity offers
CREATE TABLE entity_choice_groups (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    reference_type VARCHAR(255) NOT NULL,     -- Race, CharacterClass, Background, Feat, ClassFeature
    reference_id BIGINT UNSIGNED NOT NULL,
    level TINYINT UNSIGNED DEFAULT NULL,      -- NULL for race/background, set for class features
    group_key VARCHAR(100) NOT NULL,          -- 'skill_choice_1', 'language_choice', 'ability_score_1'
    choice_type VARCHAR(50) NOT NULL,         -- 'proficiency', 'spell', 'language', 'equipment', 'ability_score'
    choose_count TINYINT UNSIGNED NOT NULL DEFAULT 1,
    is_optional BOOLEAN DEFAULT FALSE,

    -- Ability score specific
    value TINYINT DEFAULT NULL,               -- +1, +2, or negative for future use
    choice_constraint VARCHAR(50) DEFAULT NULL, -- 'different' = can't pick same twice

    INDEX idx_reference (reference_type, reference_id),
    INDEX idx_type (choice_type),
    UNIQUE KEY uk_group (reference_type, reference_id, level, group_key)
);

-- Available options within a choice group
CREATE TABLE choice_group_options (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    choice_group_id BIGINT UNSIGNED NOT NULL,
    option_type VARCHAR(255) NOT NULL,        -- Skill, Language, Spell, Item, ProficiencyType, AbilityScore
    option_id BIGINT UNSIGNED DEFAULT NULL,   -- NULL = "any matching constraints"
    quantity TINYINT UNSIGNED DEFAULT 1,
    sort_order TINYINT UNSIGNED DEFAULT 0,

    -- Equipment: which option letter (a, b, c)
    option_letter CHAR(1) DEFAULT NULL,
    option_label VARCHAR(255) DEFAULT NULL,   -- "a rapier", "two handaxes"

    -- Spell constraints (when option_id is NULL)
    max_spell_level TINYINT UNSIGNED DEFAULT NULL,
    spell_school_id BIGINT UNSIGNED DEFAULT NULL,
    spell_class_id BIGINT UNSIGNED DEFAULT NULL,
    is_ritual_only BOOLEAN DEFAULT FALSE,

    -- Proficiency constraints (when option_id is NULL)
    proficiency_subcategory VARCHAR(50) DEFAULT NULL,

    FOREIGN KEY (choice_group_id) REFERENCES entity_choice_groups(id) ON DELETE CASCADE,
    INDEX idx_choice_group (choice_group_id)
);
```

### Columns to Drop (After Migration)

| Table | Columns |
|-------|---------|
| `entity_proficiencies` | `is_choice`, `choice_group`, `choice_option`, `quantity` |
| `entity_languages` | `is_choice`, `choice_group`, `choice_option`, `quantity` |
| `entity_modifiers` | `is_choice`, `choice_count`, `choice_constraint` |
| `entity_spells` | `is_choice`, `choice_count`, `choice_group`, `max_level`, `school_id`, `class_id`, `is_ritual_only` |
| `entity_items` | `is_choice`, `choice_group`, `choice_option`, `choice_description` |

---

## Architecture

### Choice Types & Resolvers

| Choice Type | Resolver | Replaces | Selection Storage |
|-------------|----------|----------|-------------------|
| `proficiency` | ProficiencyChoiceResolver | ProficiencyChoiceHandler | character_proficiencies |
| `language` | LanguageChoiceResolver | LanguageChoiceHandler | character_languages |
| `ability_score` | AbilityScoreChoiceResolver | AbilityScoreChoiceHandler | character_ability_scores |
| `equipment` | EquipmentChoiceResolver | EquipmentChoiceHandler | character_equipment |
| `spell` | (SpellChoiceHandler updated) | N/A | character_spells |

**Key insight:** Selections continue to be stored in `character_*` tables. The new system only changes how choices are **defined**, not how they're **stored**.

### Handler Changes

| Handler | Action |
|---------|--------|
| ProficiencyChoiceHandler | DELETE - replaced by resolver |
| LanguageChoiceHandler | DELETE - replaced by resolver |
| AbilityScoreChoiceHandler | DELETE - replaced by resolver |
| EquipmentChoiceHandler | DELETE - replaced by resolver |
| SpellChoiceHandler | UPDATE - read subclass spells from new tables |
| OptionalFeatureChoiceHandler | KEEP - uses class_counters (out of scope) |

### Resolver Interface

```php
interface ChoiceResolverInterface
{
    public function getType(): string;

    /**
     * Get pending choices for a character.
     * Must return PendingChoice with all 13 parameters to preserve API contract.
     */
    public function getChoices(Character $character): Collection;

    public function resolve(Character $character, PendingChoice $choice, array $selection): void;

    public function canUndo(Character $character, PendingChoice $choice): bool;

    public function undo(Character $character, PendingChoice $choice): void;
}
```

### SpellChoiceHandler Split

SpellChoiceHandler reads from **two sources**:

| Spell Choice Type | Data Source | After Unification |
|-------------------|-------------|-------------------|
| Class cantrips | `class_level_progression.cantrips_known` | Unchanged |
| Class spells known | `class_level_progression.spells_known` | Unchanged |
| Subclass feature spells | `entity_spells.is_choice` | Reads from `entity_choice_groups` |

One handler, two data sources. API surface unchanged (type = `'spell'`).

---

## Parser Updates

Parsers that write `is_choice` data need to use new tables:

| Parser | Choice Types |
|--------|--------------|
| ClassXmlParser | proficiency, equipment |
| RaceXmlParser | proficiency, language, ability_score |
| BackgroundXmlParser | proficiency, language, equipment |
| FeatXmlParser | proficiency, language |
| SubclassXmlParser | proficiency, spell |
| ClassFeatureXmlParser | spell |

### New Trait: ParsesChoiceGroups

```php
trait ParsesChoiceGroups
{
    protected function createChoiceGroup(
        string $referenceType,
        int $referenceId,
        string $choiceType,
        string $groupKey,
        int $chooseCount,
        ?int $level = null,
        array $extra = []
    ): EntityChoiceGroup;

    protected function addChoiceOption(
        EntityChoiceGroup $group,
        string $optionType,
        ?int $optionId = null,
        array $attributes = []
    ): ChoiceGroupOption;

    protected function addUnrestrictedOption(
        EntityChoiceGroup $group,
        string $optionType,
        array $constraints = []
    ): ChoiceGroupOption;
}
```

---

## PendingChoice DTO (Unchanged)

The existing DTO must be preserved to maintain API contract:

```php
new PendingChoice(
    id: string,              // Deterministic: {type}|{source}|{slug}|{level}|{group}
    type: string,            // proficiency, language, spell, equipment, ability_score
    subtype: ?string,        // skill, tool, cantrip, etc.
    source: string,          // class, race, background, feat, subclass_feature
    sourceName: string,      // "Rogue", "High Elf", etc.
    levelGranted: int,
    required: bool,
    quantity: int,
    remaining: int,
    selected: array,
    options: ?array,
    optionsEndpoint: ?string,
    metadata: array,
)
```

---

## Implementation Phases

| Phase | Description | Risk |
|-------|-------------|------|
| 1. Scaffolding | Migration, models, factories | Low |
| 2. Resolvers | Interface + 5 resolvers | Medium |
| 3. Parser Trait | ParsesChoiceGroups trait | Low |
| 4. Parser Updates | Update 6 parsers | Medium |
| 5. SpellChoiceHandler | Update for new tables | Low |
| 6. Wire Up | Register resolvers with CharacterChoiceService | Low |
| 7. Drop Columns | Migration to remove old columns | Low |
| 8. Reimport | Fresh import with new structure | Low |
| 9. Delete Old Handlers | Remove 4 handlers + tests | Low |
| 10. Verify | Full test suite, wizard flow tests | Critical |

---

## Out of Scope

- **OptionalFeatureChoiceHandler** - Uses `class_counters`, not entity_* tables
- **Class cantrip/spell choices** - Uses `class_level_progression`, not entity_* tables
- **character_choice_selections table** - Selections stay in existing character_* tables

---

## Migration Strategy

1. Create new tables (additive)
2. Update parsers to write to new tables
3. Reimport all data
4. Verify new tables populated correctly
5. Update handlers/resolvers to read from new tables
6. Drop old columns (destructive)
7. Delete old handler classes

**No data migration** - Alpha stage, clean reimport is acceptable.
