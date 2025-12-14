# Feature Uses Tracking Implementation Plan

**Issue:** #256 - Feature Uses Tracking: Limited-Use Abilities
**Date:** 2025-12-10
**Status:** Ready for Implementation

## Overview

Wire up existing infrastructure to track and manage limited-use class features (Rage, Ki, Action Surge, etc.). Most components already exist - this is primarily a connection task.

## Existing Infrastructure (No Changes Needed)

| Component | Location | Status |
|-----------|----------|--------|
| `class_counters` table | Migration exists | 103 counters imported |
| `character_features.uses_remaining/max_uses` | Migration exists | Columns present |
| `feature_selections.uses_remaining/max_uses` | Migration exists | Columns present |
| `class_features.resets_on` | Migration exists | 206 features have data |
| `HasLimitedUses` trait | `app/Models/Concerns/` | Complete |
| `ResetTiming` enum | `app/Enums/` | Complete |
| `RestService` | `app/Services/` | Identifies features |
| `RestController` | `app/Http/Controllers/Api/` | Endpoints exist |

## Implementation Tasks

### Phase 1: Core Service (FeatureUseService)

**File:** `app/Services/FeatureUseService.php` (new)

```php
<?php

namespace App\Services;

use App\Enums\ResetTiming;
use App\Models\Character;
use App\Models\CharacterFeature;
use App\Models\ClassCounter;
use App\Models\ClassFeature;
use Illuminate\Support\Collection;

class FeatureUseService
{
    /**
     * Get all features with limited uses for a character.
     * Includes current uses, max uses, and reset timing.
     *
     * @return Collection<int, array{
     *     id: int,
     *     feature_name: string,
     *     feature_slug: string,
     *     source: string,
     *     uses_remaining: int,
     *     max_uses: int,
     *     resets_on: string|null
     * }>
     */
    public function getFeaturesWithUses(Character $character): Collection;

    /**
     * Use a feature (decrement uses_remaining).
     * Returns false if no uses remaining.
     */
    public function useFeature(Character $character, int $characterFeatureId): bool;

    /**
     * Reset a specific feature's uses to max.
     */
    public function resetFeature(Character $character, int $characterFeatureId): void;

    /**
     * Reset all features matching a recharge type.
     * Called by RestService during short/long rest.
     */
    public function resetByRechargeType(Character $character, ResetTiming ...$types): int;

    /**
     * Initialize max_uses for a character feature based on class counters.
     * Called when feature is granted (CharacterFeatureService::createFeatureIfNotExists).
     */
    public function initializeUsesForFeature(
        CharacterFeature $characterFeature,
        int $classLevel
    ): void;

    /**
     * Recalculate max_uses for all features (on level up).
     * Some features scale with level (Ki, Rage, etc.)
     */
    public function recalculateMaxUses(Character $character): void;

    /**
     * Look up the counter value for a feature at a given level.
     */
    private function getCounterValueForFeature(
        ClassFeature $feature,
        int $level
    ): ?int;
}
```

**Key Logic:**

```php
public function getCounterValueForFeature(ClassFeature $feature, int $level): ?int
{
    // Find counter by matching feature_name to counter_name
    // Get the highest level counter <= character's class level
    return ClassCounter::where('class_id', $feature->class_id)
        ->where('counter_name', $feature->feature_name)
        ->where('level', '<=', $level)
        ->orderByDesc('level')
        ->value('counter_value');
}
```

---

### Phase 2: Wire Up CharacterFeatureService

**File:** `app/Services/CharacterFeatureService.php` (modify)

**Changes:**

1. Inject `FeatureUseService` in constructor
2. In `createFeatureIfNotExists()`, after creating the feature:
   ```php
   // Initialize uses if this feature has a counter
   $this->featureUseService->initializeUsesForFeature(
       $characterFeature,
       $levelAcquired
   );
   ```

---

### Phase 3: Wire Up RestService

**File:** `app/Services/RestService.php` (modify)

**Changes:**

1. Inject `FeatureUseService` in constructor
2. In `shortRest()`, after identifying features:
   ```php
   // Actually reset the uses
   $resetCount = $this->featureUseService->resetByRechargeType(
       $character,
       ResetTiming::SHORT_REST
   );
   ```
3. In `longRest()`, after identifying features:
   ```php
   // Reset all features (long rest encompasses all timings)
   $resetCount = $this->featureUseService->resetByRechargeType(
       $character,
       ResetTiming::SHORT_REST,
       ResetTiming::LONG_REST,
       ResetTiming::DAWN
   );
   ```

---

### Phase 4: API Endpoint for Using Features

**File:** `app/Http/Controllers/Api/FeatureUseController.php` (new)

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Character;
use App\Services\FeatureUseService;
use Illuminate\Http\JsonResponse;

class FeatureUseController extends Controller
{
    public function __construct(
        private readonly FeatureUseService $featureUseService
    ) {}

    /**
     * Use a feature (decrement uses).
     *
     * POST /api/v1/characters/{character}/features/{characterFeatureId}/use
     */
    public function use(Character $character, int $characterFeatureId): JsonResponse
    {
        $success = $this->featureUseService->useFeature($character, $characterFeatureId);

        if (! $success) {
            return response()->json([
                'message' => 'No uses remaining for this feature',
            ], 422);
        }

        $feature = $character->features()->find($characterFeatureId);

        return response()->json([
            'success' => true,
            'uses_remaining' => $feature->uses_remaining,
            'max_uses' => $feature->max_uses,
        ]);
    }

    /**
     * Reset a feature's uses (for manual override/DM fiat).
     *
     * POST /api/v1/characters/{character}/features/{characterFeatureId}/reset
     */
    public function reset(Character $character, int $characterFeatureId): JsonResponse
    {
        $this->featureUseService->resetFeature($character, $characterFeatureId);

        $feature = $character->features()->find($characterFeatureId);

        return response()->json([
            'success' => true,
            'uses_remaining' => $feature->uses_remaining,
            'max_uses' => $feature->max_uses,
        ]);
    }
}
```

**Routes:** `routes/api.php`

```php
Route::post('characters/{character}/features/{characterFeatureId}/use', [FeatureUseController::class, 'use']);
Route::post('characters/{character}/features/{characterFeatureId}/reset', [FeatureUseController::class, 'reset']);
```

---

### Phase 5: Populate features_with_uses in CharacterSummaryDTO

**File:** `app/DTOs/CharacterSummaryDTO.php` (modify)

**Changes:**

1. Inject `FeatureUseService` in `fromCharacter()`
2. Replace placeholder in `getResources()`:
   ```php
   // Features with uses
   $featuresWithUses = $featureUseService->getFeaturesWithUses($character)
       ->map(fn ($f) => [
           'id' => $f['id'],
           'feature_name' => $f['feature_name'],
           'uses_remaining' => $f['uses_remaining'],
           'max_uses' => $f['max_uses'],
           'resets_on' => $f['resets_on'],
           'source' => $f['source'],
       ])
       ->values()
       ->all();
   ```

---

### Phase 6: Handle Level-Up Recalculation

**File:** `app/Services/LevelUpService.php` (modify, if exists) or create observer

When a character levels up, recalculate max_uses for scaling features:

```php
$this->featureUseService->recalculateMaxUses($character);
```

This handles:
- Ki points (monk level)
- Rage (barbarian level breakpoints)
- Action Surge (fighter 17)
- Etc.

---

## Test Plan

### Unit Tests: `tests/Unit/Services/FeatureUseServiceTest.php`

| Test | Description |
|------|-------------|
| `it_gets_features_with_uses_for_character` | Returns collection with correct structure |
| `it_excludes_features_without_limited_uses` | Only includes features with max_uses set |
| `it_uses_feature_decrements_remaining` | uses_remaining decreases by 1 |
| `it_returns_false_when_no_uses_remaining` | Cannot use exhausted feature |
| `it_resets_feature_to_max_uses` | resetFeature restores uses_remaining |
| `it_resets_by_short_rest_timing` | Only resets SHORT_REST features |
| `it_resets_by_long_rest_timing` | Resets SHORT/LONG/DAWN features |
| `it_initializes_uses_from_counter` | Looks up ClassCounter for max_uses |
| `it_handles_features_without_counters` | No error for features without counters |
| `it_recalculates_on_level_up` | max_uses increases when counter scales |

### Feature Tests: `tests/Feature/Api/FeatureUseApiTest.php`

| Test | Description |
|------|-------------|
| `it_uses_a_feature` | POST /features/{id}/use returns updated uses |
| `it_returns_422_when_feature_exhausted` | Cannot use feature with 0 remaining |
| `it_resets_a_feature` | POST /features/{id}/reset restores uses |
| `it_returns_404_for_invalid_feature` | Feature must belong to character |
| `it_returns_404_for_invalid_character` | Character must exist |

### Integration Tests: `tests/Feature/Api/RestControllerTest.php` (extend)

| Test | Description |
|------|-------------|
| `short_rest_resets_feature_uses` | uses_remaining restored for SHORT_REST features |
| `long_rest_resets_all_feature_uses` | All limited features reset |
| `short_rest_does_not_reset_long_rest_features` | Rage (LONG_REST) not reset on short rest |

### Integration Tests: `tests/Feature/Api/CharacterSummaryTest.php` (extend)

| Test | Description |
|------|-------------|
| `it_returns_features_with_uses_in_resources` | Replaces empty array test |

---

## Files to Create/Modify

| Action | File |
|--------|------|
| **Create** | `app/Services/FeatureUseService.php` |
| **Create** | `app/Http/Controllers/Api/FeatureUseController.php` |
| **Create** | `tests/Unit/Services/FeatureUseServiceTest.php` |
| **Create** | `tests/Feature/Api/FeatureUseApiTest.php` |
| **Modify** | `app/Services/CharacterFeatureService.php` |
| **Modify** | `app/Services/RestService.php` |
| **Modify** | `app/DTOs/CharacterSummaryDTO.php` |
| **Modify** | `routes/api.php` |
| **Modify** | `tests/Feature/Api/RestControllerTest.php` |
| **Modify** | `tests/Feature/Api/CharacterSummaryTest.php` |

---

## Execution Order

1. **FeatureUseService** - Core logic (TDD: write tests first)
2. **FeatureUseController + Routes** - API endpoints (TDD)
3. **CharacterFeatureService** - Wire up initialization
4. **RestService** - Wire up reset on rest
5. **CharacterSummaryDTO** - Populate features_with_uses
6. **Level-up handling** - Recalculate on level change
7. **Quality gates** - Pint, full test suite

---

## Counter-to-Feature Matching Strategy

The `class_counters.counter_name` closely matches `class_features.feature_name`:

| Counter Name | Feature Name | Match |
|--------------|--------------|-------|
| `Action Surge` | `Action Surge` | Exact |
| `Ki` | `Ki` | Exact |
| `Rage` | `Rage` | Exact |
| `Channel Divinity` | `Channel Divinity` | Exact |
| `Second Wind` | `Second Wind` | Exact |
| `Wild Shape` | `Wild Shape` | Exact |

**Matching Algorithm:**
1. Exact match on `counter_name = feature_name`
2. Fall back to `class_id` constraint (counter belongs to same class as feature)
3. Get highest-level counter where `counter.level <= character_class_level`

---

## Edge Cases

1. **Feature without counter** - Some features have `resets_on` but no counter (unlimited uses per rest). Leave `max_uses = null`.

2. **Dynamic ability-based uses** - Bardic Inspiration (CHA mod), Divine Sense (1 + CHA mod). These need formula evaluation, not counter lookup. **Defer to Phase 2** - hardcode common formulas if time permits.

3. **Multiclass** - Each class has its own counters. A Fighter 2/Monk 3 has:
   - Action Surge (1 use) from Fighter
   - Ki (3 points) from Monk

4. **Subclass counters** - Some counters are subclass-specific (e.g., "Accursed Specter" for Hexblade). The `class_id` on the counter points to the subclass, which matches the feature's `class_id`.

---

## Acceptance Criteria

- [ ] Can view all limited-use features with remaining uses in summary
- [ ] Can expend a feature use via API
- [ ] Short rest resets short-rest features
- [ ] Long rest resets all features
- [ ] Features get max_uses initialized when granted
- [ ] Level-up recalculates max_uses for scaling features
- [ ] Error when trying to use exhausted feature
- [ ] All tests pass
- [ ] Pint formatting passes
