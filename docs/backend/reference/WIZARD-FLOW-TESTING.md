# Wizard Flow Chaos Testing

**Purpose:** Simulates the frontend character wizard to find bugs, especially in **switch/backtrack** scenarios where users change race/class/background mid-flow.

**Location:** `app/Services/WizardFlowTesting/` | `app/Console/Commands/TestWizardFlowCommand.php`

**GitHub Issue:** #443

## Quick Start

```bash
# Run 10 linear flows (no switches)
docker compose exec php php artisan test:wizard-flow --count=10

# Enable chaos mode (random switches)
docker compose exec php php artisan test:wizard-flow --count=10 --chaos

# Specific switch pattern (reproducible)
docker compose exec php php artisan test:wizard-flow --switches=race,background,race

# With seed for reproducibility
docker compose exec php php artisan test:wizard-flow --chaos --seed=12345

# View previous reports
docker compose exec php php artisan test:wizard-flow --list-reports
docker compose exec php php artisan test:wizard-flow --show-report=<run_id>
```

## Test Modes

| Mode | Description |
|------|-------------|
| Linear | Full wizard flow without switches (baseline) |
| Chaos | Random switches inserted at random points |
| Parameterized | Specific switch sequence for reproducing bugs |

## Switch Validation

After each switch, validates cascade behavior:
- **Race switch:** Racial spells/features/languages cleared, speed/size updated
- **Background switch:** Background features/proficiencies cleared
- **Class switch:** Class spells/features cleared, hit die updated

## Reports

JSON reports saved to `storage/wizard-flow-reports/` with:
- Full snapshots before/after switches
- Failure patterns identified
- Seed for reproducibility

## Known Limitations

- Equipment category choices (e.g., "any martial weapon") are skipped
- Some race/background combos have data issues (language count mismatches)

## When to Use

- After changing choice handler logic
- After modifying cascade/cleanup behavior
- Before major releases to verify wizard stability
- When investigating user-reported wizard bugs
