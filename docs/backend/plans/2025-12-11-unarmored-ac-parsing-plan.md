# Unarmored AC Parsing Implementation Plan

**Issue:** #455 - Parse unarmored AC calculations from feat/race text
**Branch:** `feature/issue-455-unarmored-ac-parsing`

## Pre-Flight

- [ ] Runner: Sail (`sail artisan`, `sail composer`)
- [ ] Create feature branch from main
- [ ] Verify tests pass before starting

```bash
git checkout -b feature/issue-455-unarmored-ac-parsing
docker compose exec php ./vendor/bin/pest --testsuite=Unit-Pure
```

---

## Task 1: Parser Trait with Unit Tests (TDD)

**Files:**
- `app/Services/Parsers/Concerns/ParsesUnarmoredAc.php`
- `tests/Unit/Parsers/ParsesUnarmoredAcTest.php`

**Steps:**
1. Write failing test: Dragon Hide pattern `"calculate your AC as 13 + your Dexterity modifier"`
2. Write failing test: Lizardfolk pattern `"your AC is 13 + your Dexterity modifier"`
3. Write failing test: Loxodon pattern `"your AC is 12 + your Constitution modifier"`
4. Write failing test: Tortle pattern `"base AC of 17"` (no ability modifier)
5. Write failing test: Shield allowance detection `"You can use a shield"`
6. Write failing test: Armor replacement detection `"You can't wear light, medium, or heavy armor"`
7. Write failing test: Returns null when no AC pattern found
8. Implement `parseUnarmoredAc()` to pass all tests

**Parser signature:**
```php
trait ParsesUnarmoredAc
{
    /**
     * Parse unarmored AC calculation from description text.
     *
     * @return array{base_ac: int, ability_code: string|null, allows_shield: bool, replaces_armor: bool}|null
     */
    protected function parseUnarmoredAc(string $text): ?array
}
```

**Expected return structure:**
```php
// Dragon Hide
['base_ac' => 13, 'ability_code' => 'DEX', 'allows_shield' => true, 'replaces_armor' => false]

// Tortle
['base_ac' => 17, 'ability_code' => null, 'allows_shield' => true, 'replaces_armor' => true]

// Loxodon
['base_ac' => 12, 'ability_code' => 'CON', 'allows_shield' => true, 'replaces_armor' => false]
```

**Regex patterns to implement:**
```php
// Pattern 1: "AC is X + your {Ability} modifier" or "calculate your AC as X + your {Ability} modifier"
/(?:AC\s+(?:is|as)|calculate your AC as)\s+(\d+)\s*\+\s*your\s+(Dexterity|Constitution)\s+modifier/i

// Pattern 2: "base AC of X" (flat AC, no ability)
/base\s+AC\s+of\s+(\d+)/i

// Shield detection
/(?:can\s+use|using)\s+a\s+shield/i

// Armor replacement detection
/(?:can't|cannot)\s+wear\s+(?:light,?\s*)?(?:medium,?\s*)?(?:or\s+)?(?:heavy\s+)?armor/i
```

**Verify:**
```bash
docker compose exec php ./vendor/bin/pest tests/Unit/Parsers/ParsesUnarmoredAcTest.php
```

---

## Task 2: Add Parser Trait to FeatXmlParser

**Files:**
- `app/Services/Parsers/FeatXmlParser.php`

**Steps:**
1. Add `use ParsesUnarmoredAc` to trait list
2. Add `'unarmored_ac' => $this->parseUnarmoredAc($description)` to `parseFeat()` return array

**Verify:**
```bash
docker compose exec php ./vendor/bin/pest tests/Unit/Parsers/FeatXmlParserTest.php
```

---

## Task 3: Add Parser Trait to RaceXmlParser

**Files:**
- `app/Services/Parsers/RaceXmlParser.php`

**Steps:**
1. Add `use ParsesUnarmoredAc` to trait list
2. Call `parseUnarmoredAc()` on trait text during parsing
3. Add `'unarmored_ac' => $unarmoredAc` to parsed race data

**Note:** Race traits are parsed individually. The parser should check each trait's text.

**Verify:**
```bash
docker compose exec php ./vendor/bin/pest tests/Unit/Parsers/RaceXmlParserTest.php
```

---

## Task 4: FeatImporter Integration with Tests (TDD)

**Files:**
- `app/Services/Importers/FeatImporter.php`
- `tests/Feature/Importers/FeatImporterTest.php`

**Steps:**
1. Write failing test: Import Dragon Hide feat creates `ac_unarmored` modifier
2. Write failing test: Modifier has correct `ability_score_id` (DEX)
3. Write failing test: Modifier has correct `condition` string
4. Implement `importUnarmoredAc()` method in FeatImporter
5. Call it from `importEntity()` after other modifiers

**Importer method:**
```php
private function importUnarmoredAc(Feat $feat, ?array $unarmoredAc): void
{
    if ($unarmoredAc === null) {
        return;
    }

    // Build condition string
    $conditions = [];
    $conditions[] = 'allows_shield: ' . ($unarmoredAc['allows_shield'] ? 'true' : 'false');
    $conditions[] = 'replaces_armor: ' . ($unarmoredAc['replaces_armor'] ? 'true' : 'false');
    $condition = implode('; ', $conditions);

    // Look up ability_score_id if ability_code provided
    $abilityScoreId = null;
    if ($unarmoredAc['ability_code']) {
        $abilityScore = AbilityScore::where('code', $unarmoredAc['ability_code'])->first();
        $abilityScoreId = $abilityScore?->id;
    }

    $this->importModifier($feat, 'ac_unarmored', [
        'value' => (string) $unarmoredAc['base_ac'],
        'ability_score_id' => $abilityScoreId,
        'condition' => $condition,
    ]);
}
```

**Verify:**
```bash
docker compose exec php ./vendor/bin/pest tests/Feature/Importers/FeatImporterTest.php --filter=unarmored
```

---

## Task 5: RaceImporter Integration with Tests (TDD)

**Files:**
- `app/Services/Importers/RaceImporter.php`
- `tests/Feature/Importers/RaceImporterTest.php`

**Steps:**
1. Write failing test: Import Lizardfolk race creates `ac_unarmored` modifier
2. Write failing test: Import Tortle race creates modifier with `ability_score_id: null`
3. Write failing test: Import Loxodon race creates modifier with CON ability
4. Implement `importUnarmoredAc()` method in RaceImporter (similar to FeatImporter)
5. Call it during trait import

**Verify:**
```bash
docker compose exec php ./vendor/bin/pest tests/Feature/Importers/RaceImporterTest.php --filter=unarmored
```

---

## Task 6: Quality Gates

**Steps:**
1. Run Pint: `docker compose exec php ./vendor/bin/pint`
2. Run Unit-Pure suite: `docker compose exec php ./vendor/bin/pest --testsuite=Unit-Pure`
3. Run Unit-DB suite: `docker compose exec php ./vendor/bin/pest --testsuite=Unit-DB`
4. Run Importers suite: `docker compose exec php ./vendor/bin/pest --testsuite=Importers`

---

## Task 7: Re-import Affected Data

**Steps:**
1. Re-import feats to populate new modifiers
2. Re-import races to populate new modifiers
3. Verify data in database

```bash
docker compose exec php php artisan import:feats
docker compose exec php php artisan import:races
```

**Manual verification:**
```bash
docker compose exec php php artisan tinker --execute="
    \App\Models\Modifier::where('modifier_category', 'ac_unarmored')
        ->with('reference')
        ->get()
        ->each(fn(\$m) => dump(\$m->reference->name . ': ' . \$m->value . ' + ' . \$m->abilityScore?->code));
"
```

---

## Task 8: Update CHANGELOG & Commit

**Steps:**
1. Update CHANGELOG.md under [Unreleased]
2. Commit all changes
3. Push to feature branch
4. Create PR

**CHANGELOG entry:**
```markdown
### Added
- Parse unarmored AC calculations from feat/race description text
- New `ac_unarmored` modifier category with base AC, ability modifier, and condition flags
- Affected entities: Dragon Hide feat, Lizardfolk, Loxodon, Tortle, Locathah races
```

```bash
git add -A
git commit -m "feat(#455): Parse unarmored AC calculations from feat/race text"
git push -u origin feature/issue-455-unarmored-ac-parsing
gh pr create --title "feat(#455): Parse unarmored AC calculations" --body "Closes #455"
```

---

## Verification Checklist

- [ ] Unit tests for parser pass (all text patterns)
- [ ] Integration tests for FeatImporter pass
- [ ] Integration tests for RaceImporter pass
- [ ] All existing tests still pass
- [ ] Pint shows no issues
- [ ] Dragon Hide feat has `ac_unarmored` modifier after import
- [ ] Lizardfolk race has `ac_unarmored` modifier after import
- [ ] Tortle race has `ac_unarmored` modifier with `ability_score_id: null`
- [ ] Loxodon race has `ac_unarmored` modifier with CON ability
- [ ] CHANGELOG.md updated
- [ ] PR created

---

## Known Affected Entities

| Entity | Type | Base AC | Ability | Allows Shield | Replaces Armor |
|--------|------|---------|---------|---------------|----------------|
| Dragon Hide | Feat | 13 | DEX | Yes | No |
| Lizardfolk | Race | 13 | DEX | Yes | No |
| Loxodon | Race | 12 | CON | Yes | No |
| Tortle | Race | 17 | None | Yes | Yes |
| Locathah | Race | 12 | DEX | Yes | No |

Additional races may be discovered during implementation.
