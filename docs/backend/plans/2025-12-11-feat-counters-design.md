# Feat Counters Implementation Plan

**Issue:** #449 - Parse limited-use ability counters from XML
**Date:** 2025-12-11
**Status:** Ready for implementation

## Overview

Parse limited-use counters from feat text (e.g., "You have 3 luck points") and store them in the existing `class_counters` table, extending it to support both class features and feats.

## Environment

- **Runner:** Sail (`docker compose exec php ...`)
- **Branch:** `feature/issue-449-feat-counters`
- **Database:** MySQL (prod), SQLite (tests)

---

## Phase 1: Database Migration

### Task 1.1: Extend class_counters table for feats

**File:** `database/migrations/2025_12_11_000001_add_feat_id_to_class_counters.php`

```php
Schema::table('class_counters', function (Blueprint $table) {
    // Make class_id nullable (was required)
    $table->unsignedBigInteger('class_id')->nullable()->change();

    // Add feat_id as alternative foreign key
    $table->foreignId('feat_id')->nullable()->after('class_id')
        ->constrained()->cascadeOnDelete();

    // Index for feat lookups
    $table->index('feat_id');
});

// Note: Check constraint (exactly one of class_id/feat_id) enforced at application level
// MySQL < 8.0.16 doesn't enforce CHECK constraints
```

**Verification:**
```bash
docker compose exec php php artisan migrate
docker compose exec php php artisan tinker --execute="echo implode(', ', Schema::getColumnListing('class_counters'));"
# Should include: feat_id
```

### Task 1.2: Update ClassCounter model

**File:** `app/Models/ClassCounter.php`

Add:
- `feat_id` to `$fillable`
- `feat_id` to `$casts` as integer
- `feat()` BelongsTo relationship

```php
protected $fillable = [
    'class_id',
    'feat_id',  // NEW
    'level',
    'counter_name',
    'counter_value',
    'reset_timing',
];

protected $casts = [
    'class_id' => 'integer',
    'feat_id' => 'integer',  // NEW
    'level' => 'integer',
    'counter_value' => 'integer',
];

public function feat(): BelongsTo
{
    return $this->belongsTo(Feat::class);
}
```

---

## Phase 2: Parser (TDD)

### Task 2.1: Write failing parser tests

**File:** `tests/Unit/Services/Parsers/FeatXmlParserUsageLimitsTest.php`

Test cases:
1. Parse "You have 3 luck points" → `base_uses: 3`
2. Parse "You have inexhaustible luck" with "3 luck points" → `base_uses: 3`
3. Parse feat with "once per long rest" but no explicit count → `base_uses: 1`
4. Parse feat with "proficiency bonus" times → `uses_formula: 'proficiency'`
5. Parse feat with no usage limits → `base_uses: null`
6. Verify `resets_on` is also captured (already works)

**Sample test:**
```php
it('parses luck points from Lucky feat', function () {
    $xml = <<<XML
    <compendium>
        <feat>
            <name>Lucky</name>
            <text>You have inexhaustible luck. You have 3 luck points. Whenever you make an attack roll... You regain your expended luck points when you finish a long rest.</text>
        </feat>
    </compendium>
    XML;

    $parser = new FeatXmlParser();
    $feats = $parser->parse($xml);

    expect($feats[0]['base_uses'])->toBe(3);
    expect($feats[0]['resets_on'])->toBe(ResetTiming::LONG_REST);
});
```

**Run tests (should FAIL):**
```bash
docker compose exec php ./vendor/bin/pest tests/Unit/Services/Parsers/FeatXmlParserUsageLimitsTest.php
```

### Task 2.2: Create ParsesUsageLimits trait

**File:** `app/Services/Parsers/Concerns/ParsesUsageLimits.php`

Patterns to match:
| Pattern | Example | Result |
|---------|---------|--------|
| `You have (\d+) (\w+ points)` | "You have 3 luck points" | `base_uses: 3` |
| `(\d+) times` | "use this feature 2 times" | `base_uses: 2` |
| `once per (short\|long) rest` | "once per long rest" | `base_uses: 1` |
| `(once\|twice\|three times)` | "twice before" | `base_uses: 2` |
| `proficiency bonus` times | "a number of times equal to your proficiency bonus" | `uses_formula: 'proficiency'` |

```php
trait ParsesUsageLimits
{
    use ConvertsWordNumbers;

    protected function parseBaseUses(string $text): ?int
    {
        // Pattern 1: "You have N points"
        if (preg_match('/you have (\d+) \w+ points?/i', $text, $match)) {
            return (int) $match[1];
        }

        // Pattern 2: "N times" (with word numbers)
        if (preg_match('/\b(one|two|three|four|five|\d+) times?\b/i', $text, $match)) {
            return $this->wordToNumber($match[1]) ?? (int) $match[1];
        }

        // Pattern 3: "once per rest" implies 1 use
        if (preg_match('/once per (short|long) rest/i', $text)) {
            return 1;
        }

        return null;
    }

    protected function parseUsesFormula(string $text): ?string
    {
        if (preg_match('/number of times equal to your proficiency bonus/i', $text)) {
            return 'proficiency';
        }

        if (preg_match('/times equal to .* modifier/i', $text)) {
            // Could extract ability, but keep simple for now
            return 'ability_modifier';
        }

        return null;
    }
}
```

### Task 2.3: Integrate into FeatXmlParser

**File:** `app/Services/Parsers/FeatXmlParser.php`

Add trait use and update `parseFeat()`:

```php
use ParsesUsageLimits;

private function parseFeat(SimpleXMLElement $element): array
{
    // ... existing code ...

    return [
        // ... existing fields ...
        'resets_on' => $this->parseResetTiming($description),
        'base_uses' => $this->parseBaseUses($description),        // NEW
        'uses_formula' => $this->parseUsesFormula($description),  // NEW
    ];
}
```

**Run tests (should PASS):**
```bash
docker compose exec php ./vendor/bin/pest tests/Unit/Services/Parsers/FeatXmlParserUsageLimitsTest.php
```

---

## Phase 3: Importer Integration (TDD)

### Task 3.1: Write failing importer test

**File:** `tests/Feature/Importers/FeatImporterCounterTest.php`

```php
it('creates counter record for Lucky feat', function () {
    // Import Lucky feat
    $importer = app(FeatImporter::class);
    $importer->import();

    $feat = Feat::where('name', 'Lucky')->first();

    $counter = ClassCounter::where('feat_id', $feat->id)->first();

    expect($counter)->not->toBeNull();
    expect($counter->counter_name)->toBe('Luck Points');
    expect($counter->counter_value)->toBe(3);
    expect($counter->reset_timing)->toBe('L');
    expect($counter->level)->toBe(1);
});
```

### Task 3.2: Update FeatImporter to store counters

**File:** `app/Services/Importers/FeatImporter.php`

Add counter creation after feat is saved:

```php
protected function importFeat(array $featData): void
{
    $feat = Feat::updateOrCreate(
        ['slug' => Str::slug($featData['name'])],
        [/* existing fields */]
    );

    // NEW: Create counter if feat has uses
    if ($featData['base_uses'] !== null || $featData['uses_formula'] !== null) {
        $this->createFeatCounter($feat, $featData);
    }

    // ... existing relationship imports ...
}

private function createFeatCounter(Feat $feat, array $featData): void
{
    ClassCounter::updateOrCreate(
        [
            'feat_id' => $feat->id,
            'counter_name' => $this->deriveCounterName($feat, $featData),
        ],
        [
            'class_id' => null,
            'level' => 1,  // Feats don't have level progression
            'counter_value' => $featData['base_uses'] ?? 1,
            'reset_timing' => $featData['resets_on']?->value[0] ?? null,  // 'S', 'L', 'D'
        ]
    );
}

private function deriveCounterName(Feat $feat, array $featData): string
{
    // Try to extract from text pattern (e.g., "luck points" → "Luck Points")
    // Fallback to "{FeatName} Uses"
    return $feat->name . ' Uses';
}
```

**Run tests:**
```bash
docker compose exec php ./vendor/bin/pest tests/Feature/Importers/FeatImporterCounterTest.php
```

---

## Phase 4: Service Layer Update

### Task 4.1: Update FeatureUseService for feats

**File:** `app/Services/FeatureUseService.php`

The service currently only handles `ClassFeature`. Need to extend for feat-based features.

Update `initializeUsesForFeature()` to check feat counters:

```php
public function initializeUsesForFeature(CharacterFeature $characterFeature, int $classLevel): void
{
    $feature = $characterFeature->feature;

    // Handle ClassFeature (existing)
    if ($feature instanceof ClassFeature) {
        $counterValue = $this->getCounterValueForClassFeature($feature, $classLevel);
        // ... existing logic ...
        return;
    }

    // Handle Feat (NEW)
    if ($feature instanceof Feat) {
        $counterValue = $this->getCounterValueForFeat($feature);
        if ($counterValue !== null) {
            $characterFeature->update([
                'max_uses' => $counterValue,
                'uses_remaining' => $counterValue,
            ]);
        }
    }
}

private function getCounterValueForFeat(Feat $feat): ?int
{
    return ClassCounter::where('feat_id', $feat->id)
        ->value('counter_value');
}
```

### Task 4.2: Update getFeaturesWithUses for feats

Ensure feat-based features appear in the API response with correct `resets_on`.

---

## Phase 5: Add Feat relationship to ClassCounter

### Task 5.1: Add counters relationship to Feat model

**File:** `app/Models/Feat.php`

```php
public function counters(): HasMany
{
    return $this->hasMany(ClassCounter::class);
}
```

---

## Phase 6: Quality Gates

### Task 6.1: Run all test suites

```bash
# Unit tests (parsers)
docker compose exec php ./vendor/bin/pest --testsuite=Unit-Pure

# DB tests (models, importers)
docker compose exec php ./vendor/bin/pest --testsuite=Unit-DB

# Feature tests (API)
docker compose exec php ./vendor/bin/pest --testsuite=Feature-DB
```

### Task 6.2: Format with Pint

```bash
docker compose exec php ./vendor/bin/pint
```

### Task 6.3: Verify Lucky feat in API

```bash
# Re-import feats
docker compose exec php php artisan import:feats

# Check counter exists
docker compose exec php php artisan tinker --execute="
\$feat = App\Models\Feat::where('name', 'Lucky')->first();
echo 'Feat: ' . \$feat->name . PHP_EOL;
echo 'Counter: ' . App\Models\ClassCounter::where('feat_id', \$feat->id)->first()?->toJson();
"
```

---

## Phase 7: Documentation & Commit

### Task 7.1: Update CHANGELOG.md

```markdown
### Added

- **Issue #449**: Parse limited-use ability counters from feat text
  - Extended `class_counters` table to support feat-based counters via `feat_id`
  - New `ParsesUsageLimits` trait extracts use counts from feat descriptions
  - Lucky feat now shows 3 luck points with long rest reset
  - `FeatureUseService` now initializes max_uses for feat-based features
```

### Task 7.2: Commit and push

```bash
git add -A
git commit -m "feat(#449): Parse limited-use ability counters from feat text"
git push -u origin feature/issue-449-feat-counters
```

### Task 7.3: Create PR

```bash
gh pr create --title "feat(#449): Parse limited-use ability counters from feat text" --body "$(cat <<'EOF'
## Summary
- Extended `class_counters` table to support feat-based counters via `feat_id`
- New `ParsesUsageLimits` trait extracts use counts from feat descriptions
- Lucky feat now shows 3 luck points with long rest reset

## Test plan
- [ ] Unit tests for parser patterns pass
- [ ] Integration tests for importer pass
- [ ] Lucky feat has counter after import
- [ ] All existing tests pass

Closes #449
EOF
)"
```

---

## File Summary

| File | Action |
|------|--------|
| `database/migrations/2025_12_11_*_add_feat_id_to_class_counters.php` | Create |
| `app/Models/ClassCounter.php` | Modify |
| `app/Models/Feat.php` | Modify |
| `app/Services/Parsers/Concerns/ParsesUsageLimits.php` | Create |
| `app/Services/Parsers/FeatXmlParser.php` | Modify |
| `app/Services/Importers/FeatImporter.php` | Modify |
| `app/Services/FeatureUseService.php` | Modify |
| `tests/Unit/Services/Parsers/FeatXmlParserUsageLimitsTest.php` | Create |
| `tests/Feature/Importers/FeatImporterCounterTest.php` | Create |
| `CHANGELOG.md` | Modify |

---

## Estimated Scope

- **New files:** 3
- **Modified files:** 7
- **Test files:** 2
- **Migration:** 1
