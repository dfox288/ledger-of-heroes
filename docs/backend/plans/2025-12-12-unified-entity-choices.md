# Unified Entity Choices Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use `superpowers-laravel:executing-plans` to implement this plan task-by-task.

## Progress (2025-12-12)

**Branch:** `feature/unified-entity-choices`
**Main Issue:** #523
**Sub-issues:** #524 (importers), #525 (handlers/services), #526 (tests)

### Completed
- [x] Phase 1: Scaffolding (branch + issue)
- [x] Phase 2: Data Model (migration + model + factory)
- [x] Phase 3: Remove Choice Columns (migration + model updates + delete EquipmentChoiceItem)
- [x] Phase 4: Add Relationships (HasEntityChoices trait to 6 models)
- [x] Task 5.1: Create ParsesChoices trait
- [x] **#524 - Update Importers** (NOT parsers - parsers just return data, importers write to DB)
  - Updated 4 importer traits: ImportsLanguages, ImportsEntitySpells, ImportsModifiers, ImportsProficiencies
  - Updated 4 importers: BackgroundImporter, ClassImporter, FeatImporter (RaceImporter uses traits)
  - Updated 3 model traits: HasEntitySpells, HasLanguageScopes, Language
  - Updated 7 resources to remove choice fields from fixed entity resources
  - Updated 2 tests: ProficiencyResourceTest, MulticlassRequirementResourceTest
  - **Unit-Pure: 841 passed** ✅
  - **Unit-DB: 102 failed** (expected - services need #525)
- [x] **#525 - Update Services/Handlers** to read from `entity_choices`
  - Updated services: CharacterLanguageService, CharacterProficiencyService, AbilityBonusService, EquipmentManagerService
  - Updated handlers: AbilityScoreChoiceHandler, SpellChoiceHandler, EquipmentChoiceHandler, LanguageChoiceHandler
  - Updated EquipmentValidator for wizard flow testing
  - Added migration for `choice_group` column on `character_ability_scores`
  - Updated CharacterAbilityScore model with `choice_group` fillable
  - Updated factories to remove deprecated `is_choice` fields
  - **Unit-Pure: 841 passed** ✅
  - **Unit-DB: 1233 passed** ✅ (fixed in #526)
  - **Feature-DB: 577 passed** ✅
  - **Importers: 316 passed** ✅ (fixed in #526)

### Completed
- [x] **#526 - Tests, quality gates, docs** ✅ COMPLETE (2025-12-12)
  - [x] Fixed Importers test suite (25 failures → 316 passed)
    - Fixed slug lookups to use source-prefixed format (phb:*, mm:*, core:*)
    - Removed `is_choice` column references from ExtractFixturesCommandTest
    - Added cache clearing for test isolation (MonsterImporter senses)
    - Updated tests to reflect new behavior (same-name/different-source = separate entities)
  - [x] All test suites passing:
    - Unit-Pure: 841 passed
    - Unit-DB: 1233 passed
    - Feature-DB: 577 passed
    - Importers: 316 passed
  - [x] Fresh import verified - 426 EntityChoice records created
  - [x] CHANGELOG.md updated
  - [x] PROJECT-STATUS.md updated

---

**Goal:** Consolidate all character creation choices from 5 scattered tables into a single `entity_choices` polymorphic table.

**Architecture:** Create new `entity_choices` table with slug-based references for stability. Move all `is_choice=true` rows from `entity_languages`, `entity_spells`, `entity_proficiencies`, `entity_modifiers`, and `entity_items` into the new table. Eliminate the `equipment_choice_items` child table entirely. Update parsers to write to new table, update handlers to read from it.

**Tech Stack:** Laravel 12.x, MySQL 8.0, Pest 3.x, Sail

**Runner:** `sail artisan` / `sail composer`

---

## Phase 1: Scaffolding

### Task 1.1: Create Feature Branch

```bash
git checkout main
git pull origin main
git checkout -b feature/unified-entity-choices
```

### Task 1.2: Create GitHub Issue

```bash
gh issue create --repo dfox288/ledger-of-heroes \
  --title "Consolidate choice definitions into unified entity_choices table" \
  --label "backend,refactor" \
  --body "Consolidate all is_choice=true rows from entity_languages, entity_spells, entity_proficiencies, entity_modifiers, and entity_items into a single polymorphic entity_choices table. Eliminates equipment_choice_items table entirely."
```

---

## Phase 2: Data Model

### Task 2.1: Create Migration for `entity_choices` Table

**Files:**
- Create: `database/migrations/2025_01_01_000020_create_entity_choices_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('entity_choices', function (Blueprint $table) {
            $table->id();

            // Polymorphic reference (matches existing entity_* pattern)
            $table->string('reference_type');
            $table->unsignedBigInteger('reference_id');

            // Core choice fields
            $table->string('choice_type', 50);                    // language, spell, proficiency, ability_score, equipment
            $table->string('choice_group');                       // groups related options
            $table->unsignedTinyInteger('quantity')->default(1);  // how many to pick
            $table->string('constraint', 50)->nullable();         // 'different', etc.
            $table->unsignedTinyInteger('choice_option')->nullable(); // 1=a, 2=b, 3=c (for restricted choices)

            // Target (slug-based for stability across re-imports)
            $table->string('target_type', 50)->nullable();        // 'spell', 'language', 'skill', 'item', 'proficiency_type', 'ability_score'
            $table->string('target_slug')->nullable();            // 'phb:fireball', 'common', 'acrobatics'

            // Spell constraints (promoted for query performance)
            $table->unsignedTinyInteger('spell_max_level')->nullable(); // 0=cantrip, 1-9
            $table->string('spell_list_slug')->nullable();              // phb:wizard, phb:cleric, etc.
            $table->string('spell_school_slug')->nullable();            // evocation, necromancy, etc.

            // Proficiency constraints (promoted)
            $table->string('proficiency_type', 50)->nullable();   // skill, tool, weapon, armor, saving_throw

            // Rare/edge-case constraints (JSON)
            $table->json('constraints')->nullable();

            // Metadata
            $table->text('description')->nullable();
            $table->unsignedSmallInteger('level_granted')->default(1);
            $table->boolean('is_required')->default(true);

            $table->timestamps();

            // Indexes
            $table->index(['reference_type', 'reference_id'], 'entity_choices_reference_idx');
            $table->index(['reference_type', 'reference_id', 'choice_type'], 'entity_choices_ref_type_idx');
            $table->index(['choice_type', 'spell_max_level'], 'entity_choices_spell_level_idx');
            $table->index(['choice_type', 'proficiency_type'], 'entity_choices_prof_type_idx');
            $table->index('choice_group', 'entity_choices_group_idx');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('entity_choices');
    }
};
```

**Commit:** `feat: add entity_choices migration`

---

### Task 2.2: Create EntityChoice Model

**Files:**
- Create: `app/Models/EntityChoice.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Relations\MorphTo;

/**
 * EntityChoice - Unified polymorphic table for all character creation choices.
 *
 * Table: entity_choices
 * Used by: Race, Background, CharacterClass, Feat, ClassFeature, Item
 *
 * Consolidates choices from entity_languages, entity_spells, entity_proficiencies,
 * entity_modifiers, and entity_items into a single table.
 *
 * Choice types:
 * - language: Pick from available languages
 * - spell: Pick cantrips/spells from a list
 * - proficiency: Pick skills, tools, weapons, armor, saving throws
 * - ability_score: Pick ability scores for bonuses (e.g., Half-Elf +1 to two)
 * - equipment: Pick starting equipment options
 *
 * Target types indicate what the choice resolves to:
 * - NULL target = unrestricted (any from the category)
 * - Specific target_type + target_slug = restricted to that option
 */
class EntityChoice extends BaseModel
{
    protected $fillable = [
        'reference_type',
        'reference_id',
        'choice_type',
        'choice_group',
        'quantity',
        'constraint',
        'choice_option',
        'target_type',
        'target_slug',
        'spell_max_level',
        'spell_list_slug',
        'spell_school_slug',
        'proficiency_type',
        'constraints',
        'description',
        'level_granted',
        'is_required',
    ];

    protected $casts = [
        'reference_id' => 'integer',
        'quantity' => 'integer',
        'choice_option' => 'integer',
        'spell_max_level' => 'integer',
        'constraints' => 'array',
        'level_granted' => 'integer',
        'is_required' => 'boolean',
    ];

    /**
     * Valid choice types.
     */
    public const CHOICE_TYPES = [
        'language',
        'spell',
        'proficiency',
        'ability_score',
        'equipment',
    ];

    /**
     * Valid target types for slug resolution.
     */
    public const TARGET_TYPES = [
        'spell',
        'language',
        'skill',
        'item',
        'proficiency_type',
        'ability_score',
    ];

    /**
     * Polymorphic relationship to parent entity (Race, Background, Class, Feat, etc.)
     */
    public function reference(): MorphTo
    {
        return $this->morphTo(__FUNCTION__, 'reference_type', 'reference_id');
    }

    /**
     * Scope to filter by choice type.
     */
    public function scopeOfType($query, string $type)
    {
        return $query->where('choice_type', $type);
    }

    /**
     * Scope to filter by reference.
     */
    public function scopeForReference($query, string $type, int $id)
    {
        return $query->where('reference_type', $type)
            ->where('reference_id', $id);
    }

    /**
     * Scope to get cantrip choices.
     */
    public function scopeCantrips($query)
    {
        return $query->where('choice_type', 'spell')
            ->where('spell_max_level', 0);
    }

    /**
     * Scope to get non-cantrip spell choices.
     */
    public function scopeSpells($query)
    {
        return $query->where('choice_type', 'spell')
            ->where(function ($q) {
                $q->whereNull('spell_max_level')
                    ->orWhere('spell_max_level', '>', 0);
            });
    }

    /**
     * Check if this is an unrestricted choice (any from category).
     */
    public function isUnrestricted(): bool
    {
        return $this->target_type === null && $this->target_slug === null;
    }

    /**
     * Get the constraint value from JSON or column.
     */
    public function getConstraintValue(string $key, mixed $default = null): mixed
    {
        return $this->constraints[$key] ?? $default;
    }
}
```

**Commit:** `feat: add EntityChoice model`

---

### Task 2.3: Create EntityChoice Factory

**Files:**
- Create: `database/factories/EntityChoiceFactory.php`

```php
<?php

namespace Database\Factories;

use App\Models\EntityChoice;
use App\Models\Race;
use Illuminate\Database\Eloquent\Factories\Factory;

class EntityChoiceFactory extends Factory
{
    protected $model = EntityChoice::class;

    public function definition(): array
    {
        return [
            'reference_type' => Race::class,
            'reference_id' => Race::factory(),
            'choice_type' => $this->faker->randomElement(EntityChoice::CHOICE_TYPES),
            'choice_group' => $this->faker->slug(2),
            'quantity' => 1,
            'constraint' => null,
            'choice_option' => null,
            'target_type' => null,
            'target_slug' => null,
            'spell_max_level' => null,
            'spell_list_slug' => null,
            'spell_school_slug' => null,
            'proficiency_type' => null,
            'constraints' => null,
            'description' => null,
            'level_granted' => 1,
            'is_required' => true,
        ];
    }

    /**
     * Language choice (unrestricted).
     */
    public function languageChoice(): static
    {
        return $this->state(fn () => [
            'choice_type' => 'language',
            'choice_group' => 'language_choice',
        ]);
    }

    /**
     * Language choice with specific options.
     */
    public function restrictedLanguageChoice(string $languageSlug, int $option): static
    {
        return $this->state(fn () => [
            'choice_type' => 'language',
            'choice_group' => 'language_choice',
            'choice_option' => $option,
            'target_type' => 'language',
            'target_slug' => $languageSlug,
        ]);
    }

    /**
     * Spell choice (cantrip).
     */
    public function cantripChoice(?string $classSlug = null): static
    {
        return $this->state(fn () => [
            'choice_type' => 'spell',
            'choice_group' => 'cantrip_choice',
            'spell_max_level' => 0,
            'spell_list_slug' => $classSlug,
        ]);
    }

    /**
     * Spell choice (leveled spell).
     */
    public function spellChoice(int $maxLevel = 1, ?string $classSlug = null): static
    {
        return $this->state(fn () => [
            'choice_type' => 'spell',
            'choice_group' => 'spell_choice',
            'spell_max_level' => $maxLevel,
            'spell_list_slug' => $classSlug,
        ]);
    }

    /**
     * Proficiency choice (skill).
     */
    public function skillChoice(int $quantity = 1): static
    {
        return $this->state(fn () => [
            'choice_type' => 'proficiency',
            'choice_group' => 'skill_choice',
            'quantity' => $quantity,
            'proficiency_type' => 'skill',
        ]);
    }

    /**
     * Proficiency choice with restricted skills.
     */
    public function restrictedSkillChoice(string $skillSlug, int $option): static
    {
        return $this->state(fn () => [
            'choice_type' => 'proficiency',
            'choice_group' => 'skill_choice',
            'choice_option' => $option,
            'proficiency_type' => 'skill',
            'target_type' => 'skill',
            'target_slug' => $skillSlug,
        ]);
    }

    /**
     * Ability score choice.
     */
    public function abilityScoreChoice(int $quantity = 1, ?string $constraint = 'different'): static
    {
        return $this->state(fn () => [
            'choice_type' => 'ability_score',
            'choice_group' => 'ability_score_choice',
            'quantity' => $quantity,
            'constraint' => $constraint,
        ]);
    }

    /**
     * Equipment choice.
     */
    public function equipmentChoice(int $option = 1): static
    {
        return $this->state(fn () => [
            'choice_type' => 'equipment',
            'choice_group' => 'equipment_choice',
            'choice_option' => $option,
        ]);
    }

    /**
     * For a Background.
     */
    public function forBackground(): static
    {
        return $this->state(fn () => [
            'reference_type' => \App\Models\Background::class,
            'reference_id' => \App\Models\Background::factory(),
        ]);
    }

    /**
     * For a CharacterClass.
     */
    public function forClass(): static
    {
        return $this->state(fn () => [
            'reference_type' => \App\Models\CharacterClass::class,
            'reference_id' => \App\Models\CharacterClass::factory(),
        ]);
    }

    /**
     * For a Feat.
     */
    public function forFeat(): static
    {
        return $this->state(fn () => [
            'reference_type' => \App\Models\Feat::class,
            'reference_id' => \App\Models\Feat::factory(),
        ]);
    }
}
```

**Commit:** `feat: add EntityChoice factory`

---

### Task 2.4: Run Migration and Verify

```bash
sail artisan migrate:fresh
sail artisan tinker --execute="echo App\Models\EntityChoice::count();"
```

Expected: `0` (empty table, ready for data)

**Commit:** N/A (verification only)

---

## Phase 3: Remove Choice Columns from Existing Tables

### Task 3.1: Create Migration to Drop Choice Columns

**Files:**
- Create: `database/migrations/2025_01_01_000021_drop_choice_columns_from_entity_tables.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        // Drop equipment_choice_items table entirely
        Schema::dropIfExists('equipment_choice_items');

        // entity_languages: drop choice columns
        Schema::table('entity_languages', function (Blueprint $table) {
            $table->dropColumn([
                'is_choice',
                'choice_group',
                'choice_option',
                'condition_type',
                'condition_language_id',
                'quantity',
            ]);
        });

        // entity_spells: drop choice columns
        Schema::table('entity_spells', function (Blueprint $table) {
            $table->dropColumn([
                'is_choice',
                'choice_count',
                'choice_group',
                'max_level',
                'school_id',
                'class_id',
                'is_ritual_only',
            ]);
            // Keep: is_cantrip (needed for fixed cantrip grants)
        });

        // entity_proficiencies: drop choice columns
        Schema::table('entity_proficiencies', function (Blueprint $table) {
            $table->dropColumn([
                'is_choice',
                'choice_group',
                'choice_option',
                'quantity',
            ]);
        });

        // entity_modifiers: drop choice columns
        Schema::table('entity_modifiers', function (Blueprint $table) {
            $table->dropColumn([
                'is_choice',
                'choice_count',
                'choice_constraint',
            ]);
        });

        // entity_items: drop choice columns
        Schema::table('entity_items', function (Blueprint $table) {
            $table->dropColumn([
                'is_choice',
                'choice_group',
                'choice_option',
                'choice_description',
                'proficiency_subcategory',
            ]);
        });
    }

    public function down(): void
    {
        // This migration is not reversible - requires re-import
        throw new \RuntimeException('This migration cannot be reversed. Run migrate:fresh and re-import.');
    }
};
```

**Commit:** `feat: drop choice columns from entity tables`

---

### Task 3.2: Update Entity Models (Remove Choice Fields)

**Files to modify:**
- `app/Models/EntityLanguage.php`
- `app/Models/EntitySpell.php`
- `app/Models/Proficiency.php`
- `app/Models/Modifier.php`
- `app/Models/EntityItem.php`

**EntityLanguage.php** - Remove from fillable and casts:
```php
protected $fillable = [
    'reference_type',
    'reference_id',
    'language_id',
    // Removed: is_choice, choice_group, choice_option, condition_type, condition_language_id, quantity
];

protected $casts = [
    'reference_id' => 'integer',
    'language_id' => 'integer',
    // Removed: is_choice, choice_option, condition_language_id, quantity
];

// Remove conditionLanguage() relationship
```

**EntitySpell.php** - Remove from fillable and casts:
```php
protected $fillable = [
    'reference_type',
    'reference_id',
    'spell_id',
    'ability_score_id',
    'level_requirement',
    'usage_limit',
    'is_cantrip',
    // Removed: is_choice, choice_count, choice_group, max_level, school_id, class_id, is_ritual_only
];

protected $casts = [
    'level_requirement' => 'integer',
    'is_cantrip' => 'boolean',
    // Removed: is_choice, choice_count, max_level, is_ritual_only
];

// Remove school() and characterClass() relationships
```

**Proficiency.php** - Remove from fillable and casts:
```php
protected $fillable = [
    'reference_type',
    'reference_id',
    'proficiency_type',
    'proficiency_subcategory',
    'proficiency_type_id',
    'skill_id',
    'item_id',
    'ability_score_id',
    'proficiency_name',
    'grants',
    'level',
    // Removed: is_choice, choice_group, choice_option, quantity
];

protected $casts = [
    'reference_id' => 'integer',
    'proficiency_type_id' => 'integer',
    'skill_id' => 'integer',
    'item_id' => 'integer',
    'ability_score_id' => 'integer',
    'grants' => 'boolean',
    'level' => 'integer',
    // Removed: is_choice, choice_option, quantity
];
```

**Modifier.php** - Remove from fillable and casts:
```php
protected $fillable = [
    'reference_type',
    'reference_id',
    'modifier_category',
    'ability_score_id',
    'skill_id',
    'damage_type_id',
    'value',
    'condition',
    'level',
    // Removed: is_choice, choice_count, choice_constraint
];

protected $casts = [
    'reference_id' => 'integer',
    'ability_score_id' => 'integer',
    'skill_id' => 'integer',
    'damage_type_id' => 'integer',
    'level' => 'integer',
    // Removed: is_choice, choice_count
];
```

**EntityItem.php** - Remove from fillable, casts, and choiceItems relationship:
```php
protected $fillable = [
    'reference_type',
    'reference_id',
    'item_id',
    'quantity',
    'description',
    // Removed: is_choice, choice_group, choice_option, choice_description, proficiency_subcategory
];

protected $casts = [
    'quantity' => 'integer',
    // Removed: is_choice, choice_option
];

// Remove choiceItems() relationship entirely
```

**Commit:** `refactor: remove choice fields from entity models`

---

### Task 3.3: Delete EquipmentChoiceItem Model

**Files:**
- Delete: `app/Models/EquipmentChoiceItem.php`
- Delete: `database/factories/EquipmentChoiceItemFactory.php` (if exists)

**Commit:** `refactor: remove EquipmentChoiceItem model`

---

## Phase 4: Add Relationships to Parent Models

### Task 4.1: Add `choices()` Relationship to Entities

Add to each model that can have choices: `Race`, `Background`, `CharacterClass`, `Feat`, `ClassFeature`, `Item`

**Example for Race.php:**
```php
use App\Models\EntityChoice;

/**
 * Get all choices granted by this race.
 */
public function choices(): \Illuminate\Database\Eloquent\Relations\MorphMany
{
    return $this->morphMany(EntityChoice::class, 'reference');
}

/**
 * Get language choices.
 */
public function languageChoices(): \Illuminate\Database\Eloquent\Relations\MorphMany
{
    return $this->choices()->ofType('language');
}

/**
 * Get spell choices.
 */
public function spellChoices(): \Illuminate\Database\Eloquent\Relations\MorphMany
{
    return $this->choices()->ofType('spell');
}

/**
 * Get proficiency choices.
 */
public function proficiencyChoices(): \Illuminate\Database\Eloquent\Relations\MorphMany
{
    return $this->choices()->ofType('proficiency');
}

/**
 * Get ability score choices.
 */
public function abilityScoreChoices(): \Illuminate\Database\Eloquent\Relations\MorphMany
{
    return $this->choices()->ofType('ability_score');
}
```

Add similar relationships to: `Background`, `CharacterClass`, `Feat`, `ClassFeature`, `Item`

**Commit:** `feat: add choices() relationships to entity models`

---

## Phase 5: Update Parsers

### Task 5.1: Create ChoiceParserTrait

**Files:**
- Create: `app/Services/Parsers/Traits/ParsesChoices.php`

This trait will be used by all parsers that need to parse choice data and write to `entity_choices`.

```php
<?php

namespace App\Services\Parsers\Traits;

use App\Models\EntityChoice;

trait ParsesChoices
{
    /**
     * Create an unrestricted language choice.
     */
    protected function createLanguageChoice(
        string $referenceType,
        int $referenceId,
        string $choiceGroup,
        int $quantity = 1,
        int $levelGranted = 1,
        ?array $constraints = null
    ): EntityChoice {
        return EntityChoice::create([
            'reference_type' => $referenceType,
            'reference_id' => $referenceId,
            'choice_type' => 'language',
            'choice_group' => $choiceGroup,
            'quantity' => $quantity,
            'level_granted' => $levelGranted,
            'constraints' => $constraints,
            'is_required' => true,
        ]);
    }

    /**
     * Create a restricted language choice option.
     */
    protected function createRestrictedLanguageChoice(
        string $referenceType,
        int $referenceId,
        string $choiceGroup,
        string $languageSlug,
        int $choiceOption,
        int $quantity = 1,
        int $levelGranted = 1
    ): EntityChoice {
        return EntityChoice::create([
            'reference_type' => $referenceType,
            'reference_id' => $referenceId,
            'choice_type' => 'language',
            'choice_group' => $choiceGroup,
            'quantity' => $quantity,
            'choice_option' => $choiceOption,
            'target_type' => 'language',
            'target_slug' => $languageSlug,
            'level_granted' => $levelGranted,
            'is_required' => true,
        ]);
    }

    /**
     * Create a spell choice (cantrip or leveled).
     */
    protected function createSpellChoice(
        string $referenceType,
        int $referenceId,
        string $choiceGroup,
        int $quantity = 1,
        int $maxLevel = 0,
        ?string $classSlug = null,
        ?string $schoolSlug = null,
        int $levelGranted = 1,
        ?array $constraints = null
    ): EntityChoice {
        return EntityChoice::create([
            'reference_type' => $referenceType,
            'reference_id' => $referenceId,
            'choice_type' => 'spell',
            'choice_group' => $choiceGroup,
            'quantity' => $quantity,
            'spell_max_level' => $maxLevel,
            'spell_list_slug' => $classSlug,
            'spell_school_slug' => $schoolSlug,
            'level_granted' => $levelGranted,
            'constraints' => $constraints,
            'is_required' => true,
        ]);
    }

    /**
     * Create a proficiency choice (skill, tool, weapon, etc).
     */
    protected function createProficiencyChoice(
        string $referenceType,
        int $referenceId,
        string $choiceGroup,
        string $proficiencyType,
        int $quantity = 1,
        int $levelGranted = 1,
        ?array $constraints = null
    ): EntityChoice {
        return EntityChoice::create([
            'reference_type' => $referenceType,
            'reference_id' => $referenceId,
            'choice_type' => 'proficiency',
            'choice_group' => $choiceGroup,
            'quantity' => $quantity,
            'proficiency_type' => $proficiencyType,
            'level_granted' => $levelGranted,
            'constraints' => $constraints,
            'is_required' => true,
        ]);
    }

    /**
     * Create a restricted proficiency choice option.
     */
    protected function createRestrictedProficiencyChoice(
        string $referenceType,
        int $referenceId,
        string $choiceGroup,
        string $proficiencyType,
        string $targetType,
        string $targetSlug,
        int $choiceOption,
        int $quantity = 1,
        int $levelGranted = 1
    ): EntityChoice {
        return EntityChoice::create([
            'reference_type' => $referenceType,
            'reference_id' => $referenceId,
            'choice_type' => 'proficiency',
            'choice_group' => $choiceGroup,
            'quantity' => $quantity,
            'choice_option' => $choiceOption,
            'proficiency_type' => $proficiencyType,
            'target_type' => $targetType,
            'target_slug' => $targetSlug,
            'level_granted' => $levelGranted,
            'is_required' => true,
        ]);
    }

    /**
     * Create an ability score choice.
     */
    protected function createAbilityScoreChoice(
        string $referenceType,
        int $referenceId,
        string $choiceGroup,
        int $quantity = 1,
        ?string $constraint = 'different',
        int $levelGranted = 1,
        ?array $constraints = null
    ): EntityChoice {
        return EntityChoice::create([
            'reference_type' => $referenceType,
            'reference_id' => $referenceId,
            'choice_type' => 'ability_score',
            'choice_group' => $choiceGroup,
            'quantity' => $quantity,
            'constraint' => $constraint,
            'level_granted' => $levelGranted,
            'constraints' => $constraints,
            'is_required' => true,
        ]);
    }

    /**
     * Create an equipment choice option.
     */
    protected function createEquipmentChoice(
        string $referenceType,
        int $referenceId,
        string $choiceGroup,
        int $choiceOption,
        ?string $itemSlug = null,
        ?string $categorySlug = null,
        ?string $description = null,
        int $levelGranted = 1,
        ?array $constraints = null
    ): EntityChoice {
        $targetType = null;
        $targetSlug = null;

        if ($itemSlug) {
            $targetType = 'item';
            $targetSlug = $itemSlug;
        } elseif ($categorySlug) {
            $targetType = 'proficiency_type';
            $targetSlug = $categorySlug;
        }

        return EntityChoice::create([
            'reference_type' => $referenceType,
            'reference_id' => $referenceId,
            'choice_type' => 'equipment',
            'choice_group' => $choiceGroup,
            'quantity' => 1,
            'choice_option' => $choiceOption,
            'target_type' => $targetType,
            'target_slug' => $targetSlug,
            'description' => $description,
            'level_granted' => $levelGranted,
            'constraints' => $constraints,
            'is_required' => true,
        ]);
    }
}
```

**Commit:** `feat: add ParsesChoices trait for unified choice parsing`

---

### Task 5.2: Update RaceXmlParser

**Files:**
- Modify: `app/Services/Parsers/Traits/ParsesRaces.php`

Update to use `ParsesChoices` trait and write to `entity_choices` instead of individual entity tables for choice data.

Key changes:
1. Add `use ParsesChoices;`
2. Replace `EntityLanguage::create([...is_choice => true...])` with `$this->createLanguageChoice(...)`
3. Replace `EntitySpell::create([...is_choice => true...])` with `$this->createSpellChoice(...)`
4. Replace `Proficiency::create([...is_choice => true...])` with `$this->createProficiencyChoice(...)`
5. Replace `Modifier::create([...is_choice => true...])` with `$this->createAbilityScoreChoice(...)`

**Commit:** `refactor: update RaceXmlParser to use entity_choices`

---

### Task 5.3: Update BackgroundXmlParser

**Files:**
- Modify: `app/Services/Parsers/Traits/ParsesBackgrounds.php`

Same pattern as Task 5.2.

**Commit:** `refactor: update BackgroundXmlParser to use entity_choices`

---

### Task 5.4: Update ClassXmlParser

**Files:**
- Modify: `app/Services/Parsers/Traits/ParsesClasses.php`

Update to use `ParsesChoices` for:
- Proficiency choices (skill selections)
- Equipment choices (starting equipment)

**Commit:** `refactor: update ClassXmlParser to use entity_choices`

---

### Task 5.5: Update FeatXmlParser

**Files:**
- Modify: `app/Services/Parsers/Traits/ParsesFeats.php`

Update to use `ParsesChoices` for spell and proficiency choices.

**Commit:** `refactor: update FeatXmlParser to use entity_choices`

---

### Task 5.6: Update Other Parsers as Needed

Review and update any other parsers that create choice data:
- `ParsesClassFeatures.php`
- `ParsesItems.php` (for items with proficiency requirements that are choices)

**Commit:** `refactor: update remaining parsers to use entity_choices`

---

## Phase 6: Update Choice Handlers

### Task 6.1: Create EntityChoiceQueryService

**Files:**
- Create: `app/Services/Choices/EntityChoiceQueryService.php`

Central service for querying choices from the unified table:

```php
<?php

namespace App\Services\Choices;

use App\Models\EntityChoice;
use Illuminate\Support\Collection;

class EntityChoiceQueryService
{
    /**
     * Get all choices for an entity.
     */
    public function getChoicesFor(string $referenceType, int $referenceId): Collection
    {
        return EntityChoice::forReference($referenceType, $referenceId)->get();
    }

    /**
     * Get choices of a specific type for an entity.
     */
    public function getChoicesOfType(string $referenceType, int $referenceId, string $choiceType): Collection
    {
        return EntityChoice::forReference($referenceType, $referenceId)
            ->ofType($choiceType)
            ->get();
    }

    /**
     * Get language choices for an entity.
     */
    public function getLanguageChoices(string $referenceType, int $referenceId): Collection
    {
        return $this->getChoicesOfType($referenceType, $referenceId, 'language');
    }

    /**
     * Get spell choices (cantrips and leveled) for an entity.
     */
    public function getSpellChoices(string $referenceType, int $referenceId): Collection
    {
        return $this->getChoicesOfType($referenceType, $referenceId, 'spell');
    }

    /**
     * Get cantrip choices only.
     */
    public function getCantripChoices(string $referenceType, int $referenceId): Collection
    {
        return EntityChoice::forReference($referenceType, $referenceId)
            ->cantrips()
            ->get();
    }

    /**
     * Get proficiency choices for an entity.
     */
    public function getProficiencyChoices(string $referenceType, int $referenceId): Collection
    {
        return $this->getChoicesOfType($referenceType, $referenceId, 'proficiency');
    }

    /**
     * Get ability score choices for an entity.
     */
    public function getAbilityScoreChoices(string $referenceType, int $referenceId): Collection
    {
        return $this->getChoicesOfType($referenceType, $referenceId, 'ability_score');
    }

    /**
     * Get equipment choices for an entity.
     */
    public function getEquipmentChoices(string $referenceType, int $referenceId): Collection
    {
        return $this->getChoicesOfType($referenceType, $referenceId, 'equipment');
    }

    /**
     * Group choices by choice_group.
     */
    public function groupByChoiceGroup(Collection $choices): Collection
    {
        return $choices->groupBy('choice_group');
    }
}
```

**Commit:** `feat: add EntityChoiceQueryService`

---

### Task 6.2: Update LanguageChoiceHandler

**Files:**
- Modify: `app/Services/ChoiceHandlers/LanguageChoiceHandler.php`

Update to query from `entity_choices` instead of `entity_languages`.

**Commit:** `refactor: update LanguageChoiceHandler to use entity_choices`

---

### Task 6.3: Update SpellChoiceHandler

**Files:**
- Modify: `app/Services/ChoiceHandlers/SpellChoiceHandler.php`

Update to query from `entity_choices` instead of `entity_spells`.

**Commit:** `refactor: update SpellChoiceHandler to use entity_choices`

---

### Task 6.4: Update ProficiencyChoiceHandler

**Files:**
- Modify: `app/Services/ChoiceHandlers/ProficiencyChoiceHandler.php`

Update to query from `entity_choices` instead of `entity_proficiencies`.

**Commit:** `refactor: update ProficiencyChoiceHandler to use entity_choices`

---

### Task 6.5: Update AbilityScoreChoiceHandler

**Files:**
- Modify: `app/Services/ChoiceHandlers/AbilityScoreChoiceHandler.php`

Update to query from `entity_choices` instead of `entity_modifiers`.

**Commit:** `refactor: update AbilityScoreChoiceHandler to use entity_choices`

---

### Task 6.6: Update EquipmentChoiceHandler

**Files:**
- Modify: `app/Services/ChoiceHandlers/EquipmentChoiceHandler.php`

Update to query from `entity_choices` instead of `entity_items` + `equipment_choice_items`.

**Commit:** `refactor: update EquipmentChoiceHandler to use entity_choices`

---

## Phase 7: Tests

### Task 7.1: Unit Tests for EntityChoice Model

**Files:**
- Create: `tests/Unit/Models/EntityChoiceTest.php`

```php
<?php

use App\Models\EntityChoice;
use App\Models\Race;

describe('EntityChoice Model', function () {
    it('creates an unrestricted language choice', function () {
        $race = Race::factory()->create();
        $choice = EntityChoice::factory()
            ->languageChoice()
            ->create([
                'reference_type' => Race::class,
                'reference_id' => $race->id,
            ]);

        expect($choice->choice_type)->toBe('language');
        expect($choice->isUnrestricted())->toBeTrue();
    });

    it('creates a restricted language choice', function () {
        $race = Race::factory()->create();
        $choice = EntityChoice::factory()
            ->restrictedLanguageChoice('common', 1)
            ->create([
                'reference_type' => Race::class,
                'reference_id' => $race->id,
            ]);

        expect($choice->target_type)->toBe('language');
        expect($choice->target_slug)->toBe('common');
        expect($choice->isUnrestricted())->toBeFalse();
    });

    it('scopes by choice type', function () {
        $race = Race::factory()->create();

        EntityChoice::factory()->languageChoice()->create([
            'reference_type' => Race::class,
            'reference_id' => $race->id,
        ]);
        EntityChoice::factory()->skillChoice()->create([
            'reference_type' => Race::class,
            'reference_id' => $race->id,
        ]);

        $languageChoices = EntityChoice::forReference(Race::class, $race->id)
            ->ofType('language')
            ->get();

        expect($languageChoices)->toHaveCount(1);
        expect($languageChoices->first()->choice_type)->toBe('language');
    });

    it('scopes cantrips vs spells', function () {
        $race = Race::factory()->create();

        EntityChoice::factory()->cantripChoice()->create([
            'reference_type' => Race::class,
            'reference_id' => $race->id,
        ]);
        EntityChoice::factory()->spellChoice(3)->create([
            'reference_type' => Race::class,
            'reference_id' => $race->id,
        ]);

        $cantrips = EntityChoice::forReference(Race::class, $race->id)->cantrips()->get();
        $spells = EntityChoice::forReference(Race::class, $race->id)->spells()->get();

        expect($cantrips)->toHaveCount(1);
        expect($spells)->toHaveCount(1);
    });
});
```

**Commit:** `test: add EntityChoice model tests`

---

### Task 7.2: Unit Tests for EntityChoiceQueryService

**Files:**
- Create: `tests/Unit/Services/EntityChoiceQueryServiceTest.php`

**Commit:** `test: add EntityChoiceQueryService tests`

---

### Task 7.3: Update Existing Handler Tests

Update tests for each handler to work with the new `entity_choices` structure.

**Commit:** `test: update choice handler tests for entity_choices`

---

## Phase 8: Quality Gates

### Task 8.1: Run Pint

```bash
sail composer pint
```

**Commit:** `style: apply Pint formatting`

---

### Task 8.2: Run Test Suites

```bash
sail artisan test --testsuite=Unit-Pure
sail artisan test --testsuite=Unit-DB
sail artisan test --testsuite=Feature-DB
```

Fix any failures before proceeding.

---

### Task 8.3: Fresh Import and Verify

```bash
sail artisan migrate:fresh
sail artisan import:all
sail artisan tinker --execute="echo 'EntityChoices: ' . App\Models\EntityChoice::count();"
```

Verify choices were imported correctly.

---

## Phase 9: Documentation

### Task 9.1: Update CHANGELOG.md

Add under `[Unreleased]`:

```markdown
### Changed
- Consolidated all character creation choices into unified `entity_choices` table
- Removed `is_choice`, `choice_group`, `choice_option` columns from entity_* tables
- Removed `equipment_choice_items` table

### Migration Notes
- Requires `migrate:fresh` and full re-import
- Choice handlers now query `entity_choices` instead of individual entity tables
```

**Commit:** `docs: update CHANGELOG for entity_choices consolidation`

---

### Task 9.2: Update PROJECT-STATUS.md

Add EntityChoice to model counts.

**Commit:** `docs: update PROJECT-STATUS with EntityChoice`

---

## Summary

| Phase | Tasks | Purpose |
|-------|-------|---------|
| 1 | Branch + Issue | Setup |
| 2 | Migration + Model + Factory | Create new `entity_choices` table |
| 3 | Drop columns migration + Model updates | Remove choice fields from old tables |
| 4 | Add relationships | Connect parent models to choices |
| 5 | Update parsers | Write to new table on import |
| 6 | Update handlers | Read from new table |
| 7 | Tests | Verify everything works |
| 8 | Quality gates | Pint + Tests + Import verification |
| 9 | Documentation | CHANGELOG + PROJECT-STATUS |

**Estimated commits:** ~25
**Re-import required:** Yes (no data migration)
