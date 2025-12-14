# Implementation Plan: Fix Character Choice Issues (#490, #491, #492)

**Date:** 2025-12-11
**Issues:** #490 (summary counts), #491 (duplicate fighting_style), #492 (ability score double-count)
**Branch:** `feature/issue-490-491-492-choice-fixes`
**Runner:** Sail

---

## Overview

Three related fixes for character choice handling:
1. **#492**: Expose `base_ability_scores` in API response
2. **#491**: Remove duplicate `FightingStyleChoiceHandler`
3. **#490**: Include all choice types in summary `pending_choices`

---

## Phase 1: Setup

### Task 1.1: Create Feature Branch
```bash
git checkout -b feature/issue-490-491-492-choice-fixes
```

---

## Phase 2: Fix #492 - Expose Base Ability Scores

### Task 2.1: Write Failing Test for Base Ability Scores

**File:** `tests/Feature/Api/CharacterResourceTest.php`

Add test verifying `base_ability_scores` is returned alongside `ability_scores`:

```php
it('returns both base and final ability scores', function () {
    // Create a Human (gets +1 to all stats)
    $race = Race::factory()->create(['slug' => 'human']);

    // Add racial modifier for +1 STR
    $race->modifiers()->create([
        'modifier_type' => 'ability_score',
        'modifier_subtype' => 'STR',
        'modifier_value' => '1',
        'is_choice' => false,
    ]);

    $character = Character::factory()->create([
        'race_slug' => $race->full_slug,
        'strength' => 10,  // base
    ]);

    $response = $this->getJson("/api/v1/characters/{$character->id}");

    $response->assertOk()
        ->assertJsonPath('data.ability_scores.STR', 11)      // final (10 + 1)
        ->assertJsonPath('data.base_ability_scores.STR', 10); // base
});
```

**Verify:** Test fails (base_ability_scores not present)

### Task 2.2: Add Base Ability Scores to CharacterResource

**File:** `app/Http/Resources/CharacterResource.php`

Update `getAbilityScoresData()` method (~line 75-90):

```php
private function getAbilityScoresData(): array
{
    $finalScores = $this->resource->getFinalAbilityScoresArray();
    $baseScores = $this->resource->getAbilityScoresArray();
    $modifiers = $this->calculateModifiers($finalScores);

    return [
        'ability_scores' => $this->formatAbilityScores($finalScores),
        'base_ability_scores' => $this->formatAbilityScores($baseScores),
        'modifiers' => $this->formatModifiers($modifiers),
    ];
}
```

**Verify:** Test passes

### Task 2.3: Update OpenAPI Documentation

**File:** `app/Http/Controllers/Api/CharacterController.php`

Update PHPDoc for show method to document new field.

**Verify:** `sail artisan scramble:analyze` succeeds

---

## Phase 3: Fix #491 - Remove Duplicate Fighting Style Handler

### Task 3.1: Write Test for Single Fighting Style Choice

**File:** `tests/Feature/Api/CharacterPendingChoicesTest.php`

Add test verifying Fighter gets exactly ONE fighting style choice:

```php
it('returns single fighting style choice for fighter', function () {
    $fighter = CharacterClass::where('slug', 'fighter')->first();
    $character = Character::factory()->create([
        'class_slug' => $fighter->full_slug,
        'level' => 1,
    ]);

    $response = $this->getJson("/api/v1/characters/{$character->id}/pending-choices");

    $response->assertOk();

    $choices = $response->json('data.choices');
    $fightingStyleChoices = collect($choices)->filter(function ($choice) {
        return $choice['type'] === 'fighting_style'
            || ($choice['type'] === 'optional_feature' && $choice['subtype'] === 'fighting_style');
    });

    expect($fightingStyleChoices)->toHaveCount(1);
    expect($fightingStyleChoices->first()['type'])->toBe('optional_feature');
    expect($fightingStyleChoices->first()['subtype'])->toBe('fighting_style');
});
```

**Verify:** Test fails (returns 2 choices)

### Task 3.2: Remove FightingStyleChoiceHandler Registration

**File:** `app/Providers/AppServiceProvider.php`

Remove line ~56:
```php
// DELETE THIS LINE:
$service->registerHandler(new FightingStyleChoiceHandler());
```

### Task 3.3: Delete FightingStyleChoiceHandler Class

**File:** `app/Services/ChoiceHandlers/FightingStyleChoiceHandler.php`

Delete entire file.

### Task 3.4: Update/Remove FightingStyleChoiceHandler Tests

**File:** `tests/Unit/Services/ChoiceHandlers/FightingStyleChoiceHandlerTest.php`

Delete entire file (handler no longer exists).

**Verify:**
- New test from 3.1 passes
- All other tests pass
- No references to FightingStyleChoiceHandler remain

### Task 3.5: Verify Summary Shows Correct Count

```bash
# Manual verification
curl -s "http://localhost:8080/api/v1/characters/{fighter_id}/pending-choices" | jq '.data.summary.total_pending'
# Should return 1, not 2
```

---

## Phase 4: Fix #490 - Dynamic Summary Counts

### Task 4.1: Write Failing Test for Fighting Style in Summary

**File:** `tests/Feature/Api/CharacterSummaryTest.php`

Add test verifying `fighting_style` appears in pending_choices:

```php
it('includes fighting_style in pending_choices', function () {
    $fighter = CharacterClass::where('slug', 'fighter')->first();
    $character = Character::factory()->create([
        'class_slug' => $fighter->full_slug,
        'level' => 1,
    ]);

    $response = $this->getJson("/api/v1/characters/{$character->id}/summary");

    $response->assertOk()
        ->assertJsonStructure([
            'data' => [
                'pending_choices' => [
                    'fighting_style',
                ],
            ],
        ])
        ->assertJsonPath('data.pending_choices.fighting_style', 1);
});

it('includes expertise in pending_choices', function () {
    $rogue = CharacterClass::where('slug', 'rogue')->first();
    $character = Character::factory()->create([
        'class_slug' => $rogue->full_slug,
        'level' => 1,
    ]);

    $response = $this->getJson("/api/v1/characters/{$character->id}/summary");

    $response->assertOk()
        ->assertJsonStructure([
            'data' => [
                'pending_choices' => [
                    'expertise',
                ],
            ],
        ]);
});
```

**Verify:** Tests fail (fields not present)

### Task 4.2: Refactor CharacterSummaryDTO to Use CharacterChoiceService

**File:** `app/DTOs/CharacterSummaryDTO.php`

Replace `calculatePendingChoices()` method (~line 100-149):

```php
private static function calculatePendingChoices(Character $character): array
{
    $choiceService = app(CharacterChoiceService::class);
    $summary = $choiceService->getSummary($character);

    // Get counts by type from the service
    $byType = $summary['by_type'] ?? [];

    // Return all types with their counts (default to 0 for missing types)
    return [
        'proficiencies' => $byType['proficiency'] ?? 0,
        'languages' => $byType['language'] ?? 0,
        'spells' => $byType['spell'] ?? 0,
        'optional_features' => $byType['optional_feature'] ?? 0,
        'asi' => $byType['asi'] ?? 0,
        'size' => $byType['size'] ?? 0,
        'feats' => $byType['feat'] ?? 0,
        'fighting_style' => $byType['fighting_style'] ?? 0,
        'expertise' => $byType['expertise'] ?? 0,
        'equipment' => $byType['equipment'] ?? 0,
        'subclass' => $byType['subclass'] ?? 0,
    ];
}
```

**Note:** Also add `use App\Services\CharacterChoiceService;` at top of file.

**Verify:** Tests pass

### Task 4.3: Update CharacterSummaryResource PHPDoc

**File:** `app/Http/Resources/CharacterSummaryResource.php`

Update docblock to include new fields in response shape.

### Task 4.4: Update Existing Summary Tests

**File:** `tests/Feature/Api/CharacterSummaryTest.php`

Update existing tests to expect new fields in response structure.

**Verify:** All summary tests pass

---

## Phase 5: Quality Gates

### Task 5.1: Run Pint
```bash
sail pint
```

### Task 5.2: Run Relevant Test Suites
```bash
sail test --testsuite=Unit-DB
sail test --testsuite=Feature-DB
```

### Task 5.3: Verify No Regressions
```bash
sail test --testsuite=Feature-Search
```

---

## Phase 6: Finalize

### Task 6.1: Update CHANGELOG.md

Add under `[Unreleased]`:
```markdown
### Fixed
- Character ability scores no longer double-count racial bonuses when editing (#492)
- Fighter pending choices returns single fighting style choice instead of duplicate (#491)

### Changed
- Character summary endpoint now includes all choice types in pending_choices (#490)
- Added `base_ability_scores` field to character API response (#492)
```

### Task 6.2: Commit Changes
```bash
git add -A
git commit -m "fix(#490,#491,#492): Character choice handling improvements

- Add base_ability_scores to character API response (#492)
- Remove duplicate FightingStyleChoiceHandler (#491)
- Include all choice types in summary pending_choices (#490)"
```

### Task 6.3: Create Pull Request
```bash
gh pr create --title "fix(#490,#491,#492): Character choice handling improvements" --body "## Summary
- Fixes ability score double-counting on character edit (#492)
- Removes duplicate fighting style choice for Fighters (#491)
- Adds fighting_style and expertise to summary pending_choices (#490)

## Test Plan
- [x] CharacterResource returns base_ability_scores
- [x] Fighter gets single fighting_style choice
- [x] Summary includes fighting_style and expertise counts
- [x] All test suites pass

Closes #490
Closes #491
Closes #492"
```

---

## Verification Checklist

- [ ] Base ability scores returned in character response
- [ ] Fighter pending-choices returns 1 fighting style choice (not 2)
- [ ] Summary pending_choices includes fighting_style
- [ ] Summary pending_choices includes expertise
- [ ] All Unit-DB tests pass
- [ ] All Feature-DB tests pass
- [ ] All Feature-Search tests pass
- [ ] Pint formatting clean
- [ ] CHANGELOG updated
- [ ] PR created with issue references
