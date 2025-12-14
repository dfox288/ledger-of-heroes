# Implementation Plan: Character Wizard Flow Chaos Testing

**Goal:** Create an artisan command that simulates the frontend character wizard, with emphasis on **switching** between options mid-flow to find bugs in cascade/reset logic.

**Key Insight:** Linear flows work. The bugs hide in backtracking—when users change race after selecting languages, or switch classes after adding spells.

---

## Architecture Overview

```
app/Console/Commands/
└── TestWizardFlowCommand.php          # Artisan command entry point

app/Services/WizardFlowTesting/
├── FlowGenerator.php                  # Generates flow sequences (linear, chaos, parameterized)
├── FlowExecutor.php                   # Executes flows via HTTP calls
├── CharacterRandomizer.php            # Picks random valid entities from DB
├── StateSnapshot.php                  # Captures full character state at any point
├── SwitchValidator.php                # Validates state correctness after switches
└── ReportGenerator.php                # Outputs JSON report

storage/wizard-flow-reports/           # JSON reports stored here
```

---

## Command Interface

```bash
# Run 10 random chaos flows
php artisan test:wizard-flow --count=10 --chaos

# Run with specific switch pattern (reproducible)
php artisan test:wizard-flow --switches=race,background,race

# Test all races with random flow
php artisan test:wizard-flow --all-races --chaos

# Test equipment mode switches
php artisan test:wizard-flow --equipment-modes --chaos

# Test spellcaster vs martial class switches
php artisan test:wizard-flow --class-switch-types=spellcaster,martial

# Use specific random seed (reproducible)
php artisan test:wizard-flow --seed=12345 --chaos
```

**GitHub Issue:** #443

---

## Frontend Wizard Flow (Actual Order)

Based on frontend analysis (`useCharacterWizard.ts`), the wizard has **15 steps** with dynamic visibility:

| Step | Name | Visibility | API Calls |
|------|------|------------|-----------|
| 1 | Sourcebooks | Always | `GET /sources` |
| 2 | Race | Always | `GET /races`, `POST /characters` (creates character!), `GET /characters/{id}/stats` |
| 3 | Subrace | If race has subraces | `PATCH /characters/{id}`, `GET /stats`, `GET /summary` |
| 4 | Size | If race grants size choice | `POST /choices/{id}` |
| 5 | Class | Always | `GET /classes`, `POST /characters/{id}/classes` or `PUT .../classes/{slug}` |
| 6 | Subclass | If subclass_level === 1 | `PUT /classes/{slug}/subclass` |
| 7 | Background | Always | `GET /backgrounds`, `PATCH /characters/{id}` |
| 8 | Feats | If race grants feat or pending feat choices | `GET /pending-choices?type=feat`, `POST /choices/{id}` |
| 9 | Abilities | Always | `PATCH /characters/{id}` with scores, `POST /choices/{id}` for bonuses |
| 10 | Proficiencies | If choices exist | `GET /pending-choices?type=proficiency`, `POST /choices/{id}` |
| 11 | Languages | If choices exist | `GET /pending-choices?type=language`, `POST /choices/{id}` |
| 12 | Equipment | Always | `GET /pending-choices?type=equipment`, `POST /choices/{id}` |
| 13 | Spells | If spellcaster | `GET /pending-choices?type=spell`, `POST /choices/{id}` |
| 14 | Details | Always | `PATCH /characters/{id}` (name, alignment), `POST /notes` |
| 15 | Review | Always | `GET /characters/{id}`, `GET /stats`, `GET /proficiencies`, etc. |

**Key Insight:** Character is created on **Race step** (step 2), not at the start!

### Switch Points (Where Bugs Live)

| After Step | Switch | Expected Cascade |
|------------|--------|------------------|
| 3 (Subrace) | Race | Clear subrace, languages, spells, features, recalc ability bonuses |
| 5 (Class) | Race | Clear racial data + possibly class proficiency choices that depended on race |
| 6 (Subclass) | Class | Clear subclass, class spells, class features, class proficiencies |
| 7 (Background) | Race/Class | Clear all downstream choices |
| 9 (Abilities) | Background | Clear background languages, proficiencies, equipment |
| 11 (Languages) | Race | Clear racial languages, keep background languages as pending |
| 13 (Spells) | Class | Clear all class spells, recalc spell slots |

### Equipment Mode Considerations

Equipment step (12) behavior depends on `equipment_mode`:
- `gold`: No equipment choices, just gold amount
- `equipment`: Class + background equipment choices

Switching class/background after equipment selection should:
- `gold` mode: No change (gold is fixed)
- `equipment` mode: Reset equipment choices for that source

---

## Implementation Tasks

### Task 1: Create FlowGenerator

**File:** `app/Services/WizardFlowTesting/FlowGenerator.php`

Generates sequences of actions:

```php
class FlowGenerator
{
    public function linear(): array
    {
        return [
            ['action' => 'create'],
            ['action' => 'set_race', 'target' => 'random'],
            ['action' => 'set_background', 'target' => 'random'],
            ['action' => 'set_ability_scores'],
            ['action' => 'add_class', 'target' => 'random'],
            ['action' => 'resolve_choices'],
            ['action' => 'add_spells'],  // if spellcaster
            ['action' => 'validate'],
        ];
    }

    public function chaos(int $switchCount = 3): array
    {
        $flow = $this->linear();
        // Insert random switches at random points
        return $this->injectSwitches($flow, $switchCount);
    }

    public function withSwitches(array $switchSequence): array
    {
        // e.g., ['race', 'background', 'race']
        // Build flow with specific switch points
    }
}
```

### Task 2: Create CharacterRandomizer

**File:** `app/Services/WizardFlowTesting/CharacterRandomizer.php`

Picks random valid entities from DB:

```php
class CharacterRandomizer
{
    public function __construct(private int $seed)
    {
        mt_srand($seed);
    }

    public function randomRace(): Race
    {
        return Race::inRandomOrder()->first();
    }

    public function randomBackground(): Background
    {
        return Background::inRandomOrder()->first();
    }

    public function randomClass(bool $spellcaster = null): CharacterClass
    {
        $query = CharacterClass::query();
        if ($spellcaster === true) {
            $query->whereNotNull('spellcasting_ability');
        } elseif ($spellcaster === false) {
            $query->whereNull('spellcasting_ability');
        }
        return $query->inRandomOrder()->first();
    }

    public function randomAbilityScores(): array
    {
        // Standard array shuffled
        return [...];
    }

    public function generatePublicId(): string
    {
        // adjective-noun-4char
    }

    public function differentRace(Race $current): Race
    {
        return Race::where('id', '!=', $current->id)
            ->inRandomOrder()->first();
    }
}
```

### Task 3: Create StateSnapshot

**File:** `app/Services/WizardFlowTesting/StateSnapshot.php`

Captures full character state:

```php
class StateSnapshot
{
    public static function capture(int $characterId): array
    {
        // Hit GET /characters/{id} and capture full response
        // Also hit /pending-choices, /spells, /equipment, /languages, /proficiencies

        return [
            'timestamp' => now()->toIso8601String(),
            'character' => $characterResponse,
            'pending_choices' => $choicesResponse,
            'spells' => $spellsResponse,
            'equipment' => $equipmentResponse,
            'languages' => $languagesResponse,
            'proficiencies' => $proficienciesResponse,
        ];
    }
}
```

### Task 4: Create FlowExecutor

**File:** `app/Services/WizardFlowTesting/FlowExecutor.php`

Executes flow steps via HTTP:

```php
class FlowExecutor
{
    public function execute(array $flow, CharacterRandomizer $randomizer): FlowResult
    {
        $result = new FlowResult();
        $characterId = null;

        foreach ($flow as $step) {
            $snapshotBefore = $characterId ? StateSnapshot::capture($characterId) : null;

            try {
                $response = $this->executeStep($step, $characterId, $randomizer);

                if ($step['action'] === 'create') {
                    $characterId = $response['data']['id'];
                }

                $snapshotAfter = StateSnapshot::capture($characterId);

                // If this was a switch, validate the cascade
                if (str_starts_with($step['action'], 'switch_')) {
                    $validation = SwitchValidator::validate(
                        $step['action'],
                        $snapshotBefore,
                        $snapshotAfter
                    );

                    if (!$validation->passed) {
                        $result->addFailure($step, $validation);
                    }
                }

                $result->addStep($step, $snapshotAfter);

            } catch (Exception $e) {
                $result->addError($step, $e);
                break;
            }
        }

        return $result;
    }

    private function executeStep(array $step, ?int $characterId, CharacterRandomizer $rand): array
    {
        return match ($step['action']) {
            'create' => $this->createCharacter($rand),
            'set_race' => $this->setRace($characterId, $rand->randomRace()),
            'set_background' => $this->setBackground($characterId, $rand->randomBackground()),
            'set_ability_scores' => $this->setAbilityScores($characterId, $rand->randomAbilityScores()),
            'add_class' => $this->addClass($characterId, $rand->randomClass()),
            'resolve_choices' => $this->resolveAllChoices($characterId, $rand),
            'add_spells' => $this->addSpells($characterId, $rand),
            'switch_race' => $this->switchRace($characterId, $rand),
            'switch_background' => $this->switchBackground($characterId, $rand),
            'switch_class' => $this->switchClass($characterId, $rand),
            'validate' => $this->validate($characterId),
        };
    }
}
```

### Task 5: Create SwitchValidator

**File:** `app/Services/WizardFlowTesting/SwitchValidator.php`

**This is the critical piece.** Validates cascade behavior:

```php
class SwitchValidator
{
    public static function validate(
        string $switchType,
        array $before,
        array $after
    ): ValidationResult {
        return match ($switchType) {
            'switch_race' => self::validateRaceSwitch($before, $after),
            'switch_background' => self::validateBackgroundSwitch($before, $after),
            'switch_class' => self::validateClassSwitch($before, $after),
        };
    }

    private static function validateRaceSwitch(array $before, array $after): ValidationResult
    {
        $errors = [];

        // Race should be different
        if ($before['character']['data']['race']['slug'] === $after['character']['data']['race']['slug']) {
            $errors[] = 'Race did not change';
        }

        // Racial languages should be cleared (only keep Common and chosen ones reset)
        $beforeLanguages = collect($before['languages']['data'])->pluck('slug')->toArray();
        $afterLanguages = collect($after['languages']['data'])->pluck('slug')->toArray();

        // Old racial languages should not persist
        // (Need to check which were racial vs background vs chosen)

        // Racial spells should be cleared
        $beforeRacialSpells = collect($before['spells']['data'])
            ->where('source', 'race')->pluck('slug')->toArray();
        $afterRacialSpells = collect($after['spells']['data'])
            ->where('source', 'race')->pluck('slug')->toArray();

        if (count($afterRacialSpells) > 0) {
            $errors[] = 'Racial spells not cleared after race switch: ' . implode(', ', $afterRacialSpells);
        }

        // Speed should update
        // Size should update
        // Ability bonuses should recalculate

        // Pending choices should reflect new race
        $afterChoices = collect($after['pending_choices']['data']);
        // Should have new racial choices pending

        return new ValidationResult(empty($errors), $errors);
    }

    private static function validateBackgroundSwitch(array $before, array $after): ValidationResult
    {
        $errors = [];

        // Background languages should reset
        // Background proficiencies should reset
        // Background equipment should reset (if equipment mode is 'background')
        // Pending choices should reflect new background

        return new ValidationResult(empty($errors), $errors);
    }

    private static function validateClassSwitch(array $before, array $after): ValidationResult
    {
        $errors = [];

        // Class spells should be cleared
        $beforeClassSpells = collect($before['spells']['data'])
            ->where('source', 'class')->pluck('slug')->toArray();
        $afterClassSpells = collect($after['spells']['data'])
            ->where('source', 'class')->pluck('slug')->toArray();

        if (count($afterClassSpells) > 0) {
            $errors[] = 'Class spells not cleared after class switch: ' . implode(', ', $afterClassSpells);
        }

        // Class proficiencies should reset
        // Class features should clear
        // Class equipment should reset (if equipment mode is 'class')
        // Hit die should change
        // Spellcasting ability should change (or disappear)

        return new ValidationResult(empty($errors), $errors);
    }
}
```

### Task 6: Create ReportGenerator

**File:** `app/Services/WizardFlowTesting/ReportGenerator.php`

```php
class ReportGenerator
{
    public function generate(array $results, int $seed): array
    {
        $report = [
            'run_id' => Str::uuid()->toString(),
            'timestamp' => now()->toIso8601String(),
            'seed' => $seed,
            'iterations' => count($results),
            'results' => [],
            'summary' => [
                'total' => count($results),
                'passed' => 0,
                'failed' => 0,
                'errors' => 0,
                'failure_patterns' => [],
            ],
        ];

        foreach ($results as $result) {
            $report['results'][] = $result->toArray();

            if ($result->passed) {
                $report['summary']['passed']++;
            } elseif ($result->hasError) {
                $report['summary']['errors']++;
            } else {
                $report['summary']['failed']++;
                foreach ($result->failures as $failure) {
                    $pattern = $failure['pattern'] ?? 'unknown';
                    $report['summary']['failure_patterns'][$pattern] =
                        ($report['summary']['failure_patterns'][$pattern] ?? 0) + 1;
                }
            }
        }

        return $report;
    }

    public function save(array $report): string
    {
        $path = storage_path("wizard-flow-reports/{$report['run_id']}.json");
        file_put_contents($path, json_encode($report, JSON_PRETTY_PRINT));
        return $path;
    }
}
```

### Task 7: Create Artisan Command

**File:** `app/Console/Commands/TestWizardFlowCommand.php`

```php
class TestWizardFlowCommand extends Command
{
    protected $signature = 'test:wizard-flow
        {--count=1 : Number of iterations}
        {--chaos : Enable random switches}
        {--switches= : Specific switch sequence (comma-separated)}
        {--seed= : Random seed for reproducibility}
        {--all-races : Test every race}
        {--equipment-modes : Test all equipment modes}
        {--class-switch-types= : Test switching between class types}
        {--keep-characters : Keep test characters (default)}
        {--cleanup : Delete test characters after run}';

    protected $description = 'Test character wizard flow with switch/backtrack chaos testing';

    public function handle(): int
    {
        $seed = $this->option('seed') ?? random_int(1, 999999);
        $count = (int) $this->option('count');

        $this->info("Running wizard flow test with seed: {$seed}");

        $generator = new FlowGenerator();
        $executor = new FlowExecutor();
        $reporter = new ReportGenerator();

        $results = [];

        $this->withProgressBar(range(1, $count), function ($i) use (&$results, $generator, $executor, $seed) {
            $randomizer = new CharacterRandomizer($seed + $i);

            $flow = $this->option('chaos')
                ? $generator->chaos()
                : ($this->option('switches')
                    ? $generator->withSwitches(explode(',', $this->option('switches')))
                    : $generator->linear());

            $result = $executor->execute($flow, $randomizer);
            $results[] = $result;
        });

        $this->newLine(2);

        $report = $reporter->generate($results, $seed);
        $path = $reporter->save($report);

        $this->info("Report saved to: {$path}");
        $this->table(
            ['Metric', 'Value'],
            [
                ['Total', $report['summary']['total']],
                ['Passed', $report['summary']['passed']],
                ['Failed', $report['summary']['failed']],
                ['Errors', $report['summary']['errors']],
            ]
        );

        if ($report['summary']['failed'] > 0) {
            $this->error('Failure patterns:');
            foreach ($report['summary']['failure_patterns'] as $pattern => $count) {
                $this->line("  - {$pattern}: {$count}");
            }
            return Command::FAILURE;
        }

        return Command::SUCCESS;
    }
}
```

---

## Validation Rules by Switch Type

### Race Switch → What Should Reset

| Data | Expected Behavior |
|------|-------------------|
| Race | Changed to new race |
| Racial languages | Cleared |
| Racial language choices | Reset to pending |
| Racial spells | Cleared |
| Racial features | Cleared |
| Speed | Updated to new race |
| Size | Updated to new race |
| Ability bonuses from race | Recalculated |
| Darkvision/senses | Updated |
| Pending choices | New race choices appear |

### Background Switch → What Should Reset

| Data | Expected Behavior |
|------|-------------------|
| Background | Changed to new background |
| Background languages | Cleared |
| Background proficiencies | Cleared |
| Background equipment | Cleared (if equipment_mode uses background) |
| Background features | Cleared |
| Pending choices | New background choices appear |

### Class Switch → What Should Reset

| Data | Expected Behavior |
|------|-------------------|
| Class | Changed to new class |
| Class spells | Cleared |
| Class proficiency choices | Reset |
| Class features | Cleared |
| Hit die type | Updated |
| Spellcasting ability | Updated or removed |
| Spell slots | Recalculated |
| Saving throw proficiencies | Updated |
| Class equipment | Cleared (if equipment_mode uses class) |
| Pending choices | New class choices appear |

---

## Equipment Mode Testing

Equipment mode affects what resets on switches:

| Mode | Background Switch | Class Switch |
|------|-------------------|--------------|
| `gold` | No equipment change | No equipment change |
| `equipment` | Background gear resets | Class gear resets |

Test both modes to verify correct cascade.

---

## JSON Report Structure

```json
{
  "run_id": "uuid",
  "timestamp": "2025-12-10T...",
  "seed": 12345,
  "iterations": 10,
  "results": [
    {
      "iteration": 1,
      "character_id": 123,
      "public_id": "bold-hero-x7k2",
      "flow": [
        {"action": "create", "status": "ok"},
        {"action": "set_race", "target": "phb:elf", "status": "ok"},
        {"action": "set_background", "target": "phb:sage", "status": "ok"},
        {"action": "resolve_choices", "resolved": 2, "status": "ok"},
        {"action": "switch_race", "from": "phb:elf", "to": "phb:dwarf", "status": "fail"}
      ],
      "status": "FAIL",
      "failure_point": "switch_race",
      "errors": [
        "Racial spells not cleared: ['phb:dancing-lights']",
        "Language 'Elvish' still present after switching from Elf"
      ],
      "snapshots": {
        "before_switch": { "...full state..." },
        "after_switch": { "...full state..." }
      }
    }
  ],
  "summary": {
    "total": 10,
    "passed": 7,
    "failed": 3,
    "errors": 0,
    "failure_patterns": {
      "racial_spells_not_cleared": 2,
      "language_not_reset": 1
    }
  }
}
```

---

## Implementation Order

1. **CharacterRandomizer** - Pick random entities
2. **StateSnapshot** - Capture character state
3. **FlowGenerator** - Generate linear flows first
4. **FlowExecutor** - Execute linear flows
5. **Artisan Command** - Basic `--count` working
6. **SwitchValidator** - Add validation logic
7. **FlowGenerator chaos mode** - Add switch injection
8. **ReportGenerator** - JSON output
9. **Equipment mode testing** - Add `--equipment-modes`
10. **Class type testing** - Add `--class-switch-types`

---

## Testing the Tester

Run against known-good scenarios:
1. Linear flow should pass 100%
2. Single race switch should validate correctly
3. Known bugs (if any) should be detected

---

## Success Criteria

- [ ] Can run 100 chaos iterations without crashing
- [ ] Detects when racial spells persist after race switch
- [ ] Detects when languages persist incorrectly
- [ ] Detects when class spells persist after class switch
- [ ] Detects equipment mode cascade issues
- [ ] JSON report clearly identifies failure patterns
- [ ] Seed makes runs reproducible
