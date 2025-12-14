# Level-Up Flow Testing Design

**Date:** 2025-12-14
**Status:** Ready for implementation

## Overview

Create a chaos/stress testing tool for the character level-up workflow, similar to the existing wizard flow chaos test. The goal is to find bugs in the level-up process by exercising all code paths through randomized and happy-path testing.

## Command Structure

```bash
php artisan test:level-up-flow

Options:
  --character=ID      Use existing character (must be complete)
  --create            Create fresh character first (default if no --character)
  --target-level=N    Level to reach (default: 20)
  --chaos             Enable random multiclassing and choices
  --seed=N            Random seed for reproducibility
  --count=N           Number of iterations (default: 1)
  --force-class=slug  Force specific starting class
  --cleanup           Delete test characters after run
  --verbose-steps     Show each step as it executes
  --list-reports      List previous reports
  --show-report=ID    Show specific report
```

## Test Modes

### Linear Mode (default)
- Single class progression from level 1 to target
- Predictable choices: average HP, +2 to primary stat for ASI
- Validates happy path works correctly

### Chaos Mode
- Random multiclassing at random levels
- Random HP choices (roll/average/manual)
- Random ASI vs feat selection
- Random class to level when multiclassed
- Maximum bug discovery potential

## Prerequisites Fix

The existing wizard flow test has a gap: it doesn't fail if `is_complete` is false after all steps. This must be fixed first since level-up requires a complete character.

### New: CompletionValidator

```php
class CompletionValidator
{
    public function validate(array $snapshot): ValidationResult
    {
        $errors = [];

        $isComplete = $snapshot['derived']['is_complete'] ?? false;
        if (!$isComplete) {
            $errors[] = 'Character is not complete after wizard flow';

            $status = $snapshot['derived']['validation_status'] ?? [];
            foreach ($status as $field => $valid) {
                if (!$valid) {
                    $errors[] = "Missing: {$field}";
                }
            }
        }

        return new ValidationResult(empty($errors), $errors);
    }
}
```

## Execution Flow

### Character Creation (when --create)

1. Reuse `FlowGenerator` and `FlowExecutor` from WizardFlowTesting
2. Run linear wizard flow (no chaos during creation)
3. Verify `is_complete = true` before proceeding
4. Respect `--force-class` option

### Level-Up Loop

```
For each level from current to target:
  1. Select class to level (primary in linear, random in chaos)
  2. Call POST /characters/{id}/classes/{class}/level-up
  3. Resolve pending choices in order:
     a. HP choice (roll/average/manual)
     b. Subclass choice (if unlocked)
     c. ASI/Feat choice (if at ASI level)
     d. Spell choices (new spells known)
     e. Feature selections (invocations, etc.)
  4. Capture state snapshot
  5. Validate level-up succeeded
  6. In chaos mode: maybe add multiclass
  7. Record step result
```

## Validations After Each Level

| Check | Description |
|-------|-------------|
| Level incremented | `total_level` matches expected |
| HP increased | `max_hit_points` > before (minimum +1) |
| Class level incremented | Specific class level went up by 1 |
| Features granted | Class features for this level present |
| Subclass features | If subclass unlocked, features granted |
| Spell slots updated | Match expected for caster level |
| ASI applied | If ASI level, stats changed or feat added |
| No orphaned choices | All required pending choices resolved |

## File Structure

```
app/Services/
├── WizardFlowTesting/           # Existing
│   ├── FlowExecutor.php         # Modify: add completion validation
│   ├── CompletionValidator.php  # NEW
│   └── ...
│
└── LevelUpFlowTesting/          # NEW directory
    ├── LevelUpFlowExecutor.php  # Main orchestrator
    ├── LevelUpValidator.php     # Post-level validation
    ├── LevelUpFlowResult.php    # Result container
    ├── LevelUpStepResult.php    # Per-level result
    ├── LevelUpReportGenerator.php
    └── LevelUpStateSnapshot.php # Extended snapshot

app/Console/Commands/
└── TestLevelUpFlowCommand.php   # NEW command
```

## StateSnapshot Extensions

New derived fields needed for level-up validation:

```php
'total_level' => $character['total_level'],
'max_hp' => $character['max_hit_points'],
'class_levels' => $this->extractClassLevels($character),
'required_pending_count' => $this->countRequiredPending($pendingChoices),
'ability_score_totals' => $character['ability_scores'],
'feat_slugs' => $this->extractFeatSlugs($features),
```

## Reused Components

From WizardFlowTesting:
- `CharacterRandomizer` - random selections with seed
- `StateSnapshot` - base snapshot logic
- `ValidationResult` - result container
- `FlowGenerator` + `FlowExecutor` - for --create option
- `ReportGenerator` - JSON report structure (adapt for level-up)

## Implementation Order

1. Add `CompletionValidator` to WizardFlowTesting
2. Integrate into `FlowExecutor` validate step
3. Create `LevelUpFlowTesting/` directory structure
4. Implement `LevelUpStateSnapshot` (extend base)
5. Implement `LevelUpValidator`
6. Implement `LevelUpFlowExecutor`
7. Implement result classes
8. Implement `LevelUpReportGenerator`
9. Create `TestLevelUpFlowCommand`
10. Test with linear mode
11. Add chaos mode features
12. Comprehensive testing

## Success Criteria

- [ ] Wizard flow validates `is_complete` at end
- [ ] Level-up command can create fresh character and level to 20
- [ ] Linear mode: 100% pass rate for all classes
- [ ] Chaos mode: finds edge cases without false positives
- [ ] Reports saved with reproducible seeds
- [ ] Cleanup option removes test characters
