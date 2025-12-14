# API Health Check Suite Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a smoke test suite that verifies all GET endpoints return 200 with valid JSON structure.

**Architecture:** Parse `api.json` dynamically to discover endpoints, create minimal fixtures via factories and LookupSeeder, substitute path parameters, verify response structure.

**Tech Stack:** Pest 3.x, Laravel factories, LookupSeeder, RefreshDatabase

---

## Task 1: Create OpenApiEndpointExtractor

**Files:**
- Create: `tests/Support/OpenApiEndpointExtractor.php`

**Step 1: Write the failing test**

Create `tests/Unit/Support/OpenApiEndpointExtractorTest.php`:

```php
<?php

use Tests\Support\OpenApiEndpointExtractor;

it('extracts GET endpoints from api.json', function () {
    $endpoints = OpenApiEndpointExtractor::getTestableEndpoints();

    expect($endpoints)->toBeArray();
    expect($endpoints)->not->toBeEmpty();

    // Should have the spells index endpoint
    expect($endpoints)->toHaveKey('GET /v1/spells');
    expect($endpoints['GET /v1/spells'])->toMatchArray([
        'path' => '/v1/spells',
        'params' => [],
        'paginated' => true,
    ]);
});

it('extracts single-parameter endpoints', function () {
    $endpoints = OpenApiEndpointExtractor::getTestableEndpoints();

    expect($endpoints)->toHaveKey('GET /v1/spells/{spell}');
    expect($endpoints['GET /v1/spells/{spell}']['params'])->toBe(['spell']);
});

it('excludes multi-parameter endpoints', function () {
    $endpoints = OpenApiEndpointExtractor::getTestableEndpoints();

    // Should not include nested routes with multiple params
    $multiParamEndpoints = array_filter($endpoints, fn ($e) => count($e['params']) > 1);
    expect($multiParamEndpoints)->toBeEmpty();
});

it('excludes auth-required character endpoints', function () {
    $endpoints = OpenApiEndpointExtractor::getTestableEndpoints();

    // Character endpoints require auth - skip them for health check
    expect($endpoints)->not->toHaveKey('GET /v1/characters');
    expect($endpoints)->not->toHaveKey('GET /v1/characters/{character}');
});
```

**Step 2: Run test to verify it fails**

```bash
docker compose exec php ./vendor/bin/pest tests/Unit/Support/OpenApiEndpointExtractorTest.php
```

Expected: FAIL - class not found

**Step 3: Create Support directory and implement extractor**

```bash
mkdir -p tests/Support
```

Create `tests/Support/OpenApiEndpointExtractor.php`:

```php
<?php

namespace Tests\Support;

class OpenApiEndpointExtractor
{
    private static ?array $cachedEndpoints = null;

    /**
     * Endpoints that require authentication - skip for health checks.
     */
    private const SKIP_PREFIXES = [
        '/v1/characters',
        '/v1/auth',
    ];

    /**
     * Get all testable GET endpoints from api.json.
     *
     * @return array<string, array{path: string, params: array<string>, paginated: bool}>
     */
    public static function getTestableEndpoints(): array
    {
        if (self::$cachedEndpoints !== null) {
            return self::$cachedEndpoints;
        }

        $specPath = base_path('api.json');
        if (! file_exists($specPath)) {
            throw new \RuntimeException('api.json not found at: '.$specPath);
        }

        $spec = json_decode(file_get_contents($specPath), true);
        $endpoints = [];

        foreach ($spec['paths'] ?? [] as $path => $methods) {
            // Only GET methods
            if (! isset($methods['get'])) {
                continue;
            }

            // Skip auth-required endpoints
            if (self::shouldSkipPath($path)) {
                continue;
            }

            // Extract path parameters
            preg_match_all('/\{(\w+)\}/', $path, $matches);
            $params = $matches[1] ?? [];

            // Skip multi-parameter routes
            if (count($params) > 1) {
                continue;
            }

            // Determine if paginated (has meta in 200 response schema)
            $paginated = self::isPaginated($methods['get']);

            $key = "GET {$path}";
            $endpoints[$key] = [
                'path' => $path,
                'params' => $params,
                'paginated' => $paginated,
            ];
        }

        // Sort by path for consistent ordering
        ksort($endpoints);

        self::$cachedEndpoints = $endpoints;

        return $endpoints;
    }

    /**
     * Get parameter-to-fixture mapping for path substitution.
     *
     * @return array<string, string> Maps param name to fixture key
     */
    public static function getParamFixtureMap(): array
    {
        return [
            'spell' => 'spell',
            'monster' => 'monster',
            'class' => 'class',
            'race' => 'race',
            'background' => 'background',
            'feat' => 'feat',
            'item' => 'item',
            'optionalFeature' => 'optionalFeature',
            'abilityScore' => 'abilityScore',
            'alignment' => 'alignment',
            'condition' => 'condition',
            'damageType' => 'damageType',
            'itemProperty' => 'itemProperty',
            'itemType' => 'itemType',
            'language' => 'language',
            'proficiencyType' => 'proficiencyType',
            'size' => 'size',
            'skill' => 'skill',
            'source' => 'source',
            'spellSchool' => 'spellSchool',
        ];
    }

    private static function shouldSkipPath(string $path): bool
    {
        foreach (self::SKIP_PREFIXES as $prefix) {
            if (str_starts_with($path, $prefix)) {
                return true;
            }
        }

        return false;
    }

    private static function isPaginated(array $operation): bool
    {
        $schema = $operation['responses']['200']['content']['application/json']['schema'] ?? [];

        // Check if schema has 'meta' property (Laravel pagination)
        if (isset($schema['properties']['meta'])) {
            return true;
        }

        // Check if it references a paginated response
        if (isset($schema['$ref']) && str_contains($schema['$ref'], 'Paginated')) {
            return true;
        }

        return false;
    }

    /**
     * Clear the cached endpoints (useful for testing).
     */
    public static function clearCache(): void
    {
        self::$cachedEndpoints = null;
    }
}
```

**Step 4: Run test to verify it passes**

```bash
docker compose exec php ./vendor/bin/pest tests/Unit/Support/OpenApiEndpointExtractorTest.php
```

Expected: PASS

**Step 5: Commit**

```bash
git add tests/Support/OpenApiEndpointExtractor.php tests/Unit/Support/OpenApiEndpointExtractorTest.php
git commit -m "feat: add OpenApiEndpointExtractor for health check suite"
```

---

## Task 2: Create CreatesHealthCheckFixtures Trait

**Files:**
- Create: `tests/Support/Traits/CreatesHealthCheckFixtures.php`
- Create: `tests/Unit/Support/CreatesHealthCheckFixturesTest.php`

**Step 1: Write the failing test**

Create `tests/Unit/Support/CreatesHealthCheckFixturesTest.php`:

```php
<?php

use App\Models\Spell;
use App\Models\Monster;
use App\Models\CharacterClass;
use Tests\Support\Traits\CreatesHealthCheckFixtures;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);
uses(CreatesHealthCheckFixtures::class);

beforeEach(function () {
    $this->seed(\Database\Seeders\LookupSeeder::class);
    $this->setUpHealthCheckFixtures();
});

it('creates spell fixture', function () {
    expect($this->fixtures)->toHaveKey('spell');
    expect($this->fixtures['spell'])->toBeInstanceOf(Spell::class);
    expect($this->fixtures['spell']->slug)->not->toBeEmpty();
});

it('creates monster fixture with alignment', function () {
    expect($this->fixtures)->toHaveKey('monster');
    expect($this->fixtures['monster'])->toBeInstanceOf(Monster::class);
    expect($this->fixtures['monster']->alignment)->not->toBeEmpty();
});

it('creates class fixture', function () {
    expect($this->fixtures)->toHaveKey('class');
    expect($this->fixtures['class'])->toBeInstanceOf(CharacterClass::class);
});

it('provides lookup fixtures from seeder', function () {
    expect($this->fixtures)->toHaveKey('abilityScore');
    expect($this->fixtures)->toHaveKey('spellSchool');
    expect($this->fixtures)->toHaveKey('damageType');
});

it('substitutes path parameters correctly', function () {
    $path = '/v1/spells/{spell}';
    $params = ['spell'];

    $result = $this->substitutePathParams($path, $params);

    expect($result)->toBe('/v1/spells/'.$this->fixtures['spell']->slug);
});
```

**Step 2: Run test to verify it fails**

```bash
docker compose exec php ./vendor/bin/pest tests/Unit/Support/CreatesHealthCheckFixturesTest.php
```

Expected: FAIL - trait not found

**Step 3: Create directory and implement trait**

```bash
mkdir -p tests/Support/Traits
```

Create `tests/Support/Traits/CreatesHealthCheckFixtures.php`:

```php
<?php

namespace Tests\Support\Traits;

use App\Models\AbilityScore;
use App\Models\Background;
use App\Models\CharacterClass;
use App\Models\Condition;
use App\Models\DamageType;
use App\Models\Feat;
use App\Models\Item;
use App\Models\ItemProperty;
use App\Models\ItemType;
use App\Models\Language;
use App\Models\Monster;
use App\Models\OptionalFeature;
use App\Models\ProficiencyType;
use App\Models\Race;
use App\Models\Size;
use App\Models\Skill;
use App\Models\Source;
use App\Models\Spell;
use App\Models\SpellSchool;

trait CreatesHealthCheckFixtures
{
    /**
     * Fixtures created for health check tests.
     *
     * @var array<string, mixed>
     */
    protected array $fixtures = [];

    /**
     * Set up minimal fixtures for health check tests.
     * Call this in beforeEach() after seeding lookups.
     */
    protected function setUpHealthCheckFixtures(): void
    {
        // Entity fixtures (created via factories)
        $this->fixtures['spell'] = Spell::factory()->create();
        $this->fixtures['monster'] = Monster::factory()->create(['alignment' => 'Lawful Good']);
        $this->fixtures['class'] = CharacterClass::factory()->create();
        $this->fixtures['race'] = Race::factory()->create();
        $this->fixtures['background'] = Background::factory()->create();
        $this->fixtures['feat'] = Feat::factory()->create();
        $this->fixtures['item'] = Item::factory()->create();
        $this->fixtures['optionalFeature'] = OptionalFeature::factory()->create();

        // Lookup fixtures (from LookupSeeder - just grab first record)
        $this->fixtures['abilityScore'] = AbilityScore::first();
        $this->fixtures['condition'] = Condition::first();
        $this->fixtures['damageType'] = DamageType::first();
        $this->fixtures['itemProperty'] = ItemProperty::first();
        $this->fixtures['itemType'] = ItemType::first();
        $this->fixtures['language'] = Language::first();
        $this->fixtures['proficiencyType'] = ProficiencyType::first();
        $this->fixtures['size'] = Size::first();
        $this->fixtures['skill'] = Skill::first();
        $this->fixtures['source'] = Source::first();
        $this->fixtures['spellSchool'] = SpellSchool::first();

        // Derived lookups (from entity attributes)
        $this->fixtures['alignment'] = (object) ['slug' => 'lawful-good'];
    }

    /**
     * Substitute path parameters with fixture values.
     *
     * @param  string  $path  The path with {param} placeholders
     * @param  array<string>  $params  Parameter names to substitute
     * @return string The path with actual values
     */
    protected function substitutePathParams(string $path, array $params): string
    {
        foreach ($params as $param) {
            if (! isset($this->fixtures[$param])) {
                throw new \RuntimeException("No fixture found for parameter: {$param}");
            }

            $fixture = $this->fixtures[$param];
            $value = is_object($fixture) ? ($fixture->slug ?? $fixture->id) : $fixture;

            $path = str_replace("{{$param}}", $value, $path);
        }

        return $path;
    }
}
```

**Step 4: Run test to verify it passes**

```bash
docker compose exec php ./vendor/bin/pest tests/Unit/Support/CreatesHealthCheckFixturesTest.php
```

Expected: PASS

**Step 5: Commit**

```bash
git add tests/Support/Traits/CreatesHealthCheckFixtures.php tests/Unit/Support/CreatesHealthCheckFixturesTest.php
git commit -m "feat: add CreatesHealthCheckFixtures trait"
```

---

## Task 3: Create ApiEndpointHealthTest

**Files:**
- Create: `tests/Feature/HealthCheck/ApiEndpointHealthTest.php`

**Step 1: Create test directory**

```bash
mkdir -p tests/Feature/HealthCheck
```

**Step 2: Write the health check test**

Create `tests/Feature/HealthCheck/ApiEndpointHealthTest.php`:

```php
<?php

namespace Tests\Feature\HealthCheck;

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\Support\OpenApiEndpointExtractor;
use Tests\Support\Traits\CreatesHealthCheckFixtures;
use Tests\TestCase;

/**
 * API Health Check Suite
 *
 * Smoke tests all GET endpoints discovered from api.json.
 * Verifies endpoints return 200 with valid JSON structure.
 *
 * Run: docker compose exec php ./vendor/bin/pest --testsuite=Health-Check
 */
class ApiEndpointHealthTest extends TestCase
{
    use CreatesHealthCheckFixtures;
    use RefreshDatabase;

    protected $seeder = \Database\Seeders\LookupSeeder::class;

    protected function setUp(): void
    {
        parent::setUp();
        $this->setUpHealthCheckFixtures();
    }

    /**
     * @dataProvider endpointProvider
     */
    public function test_endpoint_returns_valid_response(string $key, array $endpoint): void
    {
        $path = $this->substitutePathParams($endpoint['path'], $endpoint['params']);

        $response = $this->getJson("/api{$path}");

        // Must return 200
        $this->assertEquals(
            200,
            $response->status(),
            "Endpoint {$key} returned status {$response->status()}: ".$response->getContent()
        );

        // Must be valid JSON with 'data' key
        $json = $response->json();
        $this->assertIsArray($json, "Endpoint {$key} did not return JSON array");
        $this->assertArrayHasKey('data', $json, "Endpoint {$key} missing 'data' key");

        // Paginated endpoints must have 'meta' key
        if ($endpoint['paginated']) {
            $this->assertArrayHasKey('meta', $json, "Paginated endpoint {$key} missing 'meta' key");
        }
    }

    public static function endpointProvider(): array
    {
        // Static provider - load endpoints once
        return array_map(
            fn ($endpoint, $key) => [$key, $endpoint],
            OpenApiEndpointExtractor::getTestableEndpoints(),
            array_keys(OpenApiEndpointExtractor::getTestableEndpoints())
        );
    }
}
```

**Step 3: Run test to see initial results**

```bash
docker compose exec php ./vendor/bin/pest tests/Feature/HealthCheck/ApiEndpointHealthTest.php --stop-on-failure
```

Expected: Some tests may fail if fixtures are missing or paths need adjustment.

**Step 4: Fix any fixture issues discovered**

Review failures and update `CreatesHealthCheckFixtures` trait as needed.

**Step 5: Commit when passing**

```bash
git add tests/Feature/HealthCheck/ApiEndpointHealthTest.php
git commit -m "feat: add API endpoint health check test"
```

---

## Task 4: Add Health-Check Test Suite to phpunit.xml

**Files:**
- Modify: `phpunit.xml`

**Step 1: Add the new test suite**

Add after the `Importers` testsuite (around line 182):

```xml
        <!-- Health-Check: Smoke test all GET endpoints (approx 10-15s) -->
        <testsuite name="Health-Check">
            <directory>tests/Feature/HealthCheck</directory>
        </testsuite>
```

**Step 2: Run the suite by name**

```bash
docker compose exec php ./vendor/bin/pest --testsuite=Health-Check
```

Expected: All health check tests pass.

**Step 3: Commit**

```bash
git add phpunit.xml
git commit -m "feat: add Health-Check test suite to phpunit.xml"
```

---

## Task 5: Handle Edge Cases and Refine

**Files:**
- Modify: `tests/Support/OpenApiEndpointExtractor.php`
- Modify: `tests/Support/Traits/CreatesHealthCheckFixtures.php`

**Step 1: Run full suite and identify failures**

```bash
docker compose exec php ./vendor/bin/pest --testsuite=Health-Check 2>&1 | tee tests/results/health-check-output.log
```

**Step 2: Review and fix common issues**

Common issues to address:
- Missing fixtures for certain param types
- Routes that need specific data relationships
- Derived lookups needing parent records

**Step 3: Add any missing parameter mappings**

Review the extractor's `getParamFixtureMap()` and fixture trait for completeness.

**Step 4: Run final validation**

```bash
docker compose exec php ./vendor/bin/pest --testsuite=Health-Check
```

Expected: All tests pass.

**Step 5: Commit refinements**

```bash
git add tests/Support/
git commit -m "fix: refine health check fixtures for edge cases"
```

---

## Task 6: Final Validation and Documentation

**Step 1: Run all test suites to ensure no regressions**

```bash
docker compose exec php ./vendor/bin/pest --testsuite=Unit-Pure
docker compose exec php ./vendor/bin/pest --testsuite=Unit-DB
docker compose exec php ./vendor/bin/pest --testsuite=Feature-DB
docker compose exec php ./vendor/bin/pest --testsuite=Health-Check
```

**Step 2: Format code**

```bash
docker compose exec php ./vendor/bin/pint
```

**Step 3: Final commit**

```bash
git add -A
git commit -m "chore: format health check suite files"
```

**Step 4: Update CHANGELOG.md**

Add under `[Unreleased]`:

```markdown
### Added
- API Health Check test suite - smoke tests all GET endpoints from api.json
  - `tests/Feature/HealthCheck/ApiEndpointHealthTest.php` - main test
  - `tests/Support/OpenApiEndpointExtractor.php` - dynamic endpoint discovery
  - `tests/Support/Traits/CreatesHealthCheckFixtures.php` - minimal fixtures
  - Run with: `./vendor/bin/pest --testsuite=Health-Check`
```

**Step 5: Commit changelog**

```bash
git add CHANGELOG.md
git commit -m "docs: add health check suite to changelog"
```

---

## Summary

| Task | Files | Purpose |
|------|-------|---------|
| 1 | `OpenApiEndpointExtractor.php` | Parse api.json, extract GET endpoints |
| 2 | `CreatesHealthCheckFixtures.php` | Create minimal test fixtures |
| 3 | `ApiEndpointHealthTest.php` | Main health check test |
| 4 | `phpunit.xml` | Register new test suite |
| 5 | Support files | Handle edge cases |
| 6 | Validation | Ensure all suites pass |

**Expected results:**
- ~60-70 GET endpoints tested
- Runtime: ~10-15 seconds
- No Meilisearch dependency
