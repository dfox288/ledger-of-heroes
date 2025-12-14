# API Health Check Test Suite Design

**Date**: 2025-12-12
**Status**: Approved
**Purpose**: Smoke test + regression safety net for all GET API endpoints

## Overview

A lightweight test suite that dynamically discovers GET endpoints from `api.json` and verifies they return 200 with valid JSON structure. Catches route/controller/resource breakage without deep validation.

## Decisions

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Scope | GET endpoints only (~93) | 75% coverage, no auth/state complexity |
| Data | Minimal factories (RefreshDatabase) | Fast, self-contained, no Meilisearch |
| Discovery | Parse api.json dynamically | Auto-updates when spec changes |
| Path params | Create fixtures, substitute IDs | Tests full route→controller→resource flow |
| Nested routes | Single-param only, skip multi-param | Keeps complexity down, nested routes covered elsewhere |
| Validation | Status 200 + JSON + structure (data, meta) | Catches real issues without over-testing |

## Architecture

### File Structure

```
tests/
├── Feature/
│   └── HealthCheck/
│       └── ApiEndpointHealthTest.php
└── Support/
    ├── OpenApiEndpointExtractor.php
    └── Traits/
        └── CreatesHealthCheckFixtures.php
```

### OpenApiEndpointExtractor

Static helper class that:
- Parses `api.json` at test runtime
- Extracts all GET paths
- Filters to single-param or no-param routes
- Returns structured dataset for Pest

```php
// Output format
[
    'GET /v1/spells' => [
        'path' => '/v1/spells',
        'params' => [],
        'paginated' => true
    ],
    'GET /v1/spells/{spell}' => [
        'path' => '/v1/spells/{spell}',
        'params' => ['spell'],
        'paginated' => false
    ],
]
```

### CreatesHealthCheckFixtures Trait

Creates one record per entity type:

```php
protected static array $fixtures = [];

protected function setUpFixtures(): void
{
    self::$fixtures = [
        'spell' => Spell::factory()->create(),
        'monster' => Monster::factory()->create(),
        'class' => CharacterClass::factory()->create(),
        'race' => Race::factory()->create(),
        'background' => Background::factory()->create(),
        'feat' => Feat::factory()->create(),
        'item' => Item::factory()->create(),
        'abilityScore' => AbilityScore::factory()->create(),
        'alignment' => Alignment::factory()->create(),
        'damageType' => DamageType::factory()->create(),
        // ... ~15-20 entity types
    ];
}
```

### Parameter-to-Entity Mapping

```php
$paramEntityMap = [
    'spell' => Spell::class,
    'monster' => Monster::class,
    'class' => CharacterClass::class,
    'race' => Race::class,
    'background' => Background::class,
    'feat' => Feat::class,
    'item' => Item::class,
    'abilityScore' => AbilityScore::class,
    // ... maps route param names to model classes
];
```

### Test Implementation

```php
<?php

use Tests\Support\OpenApiEndpointExtractor;
use Tests\Support\Traits\CreatesHealthCheckFixtures;

uses(CreatesHealthCheckFixtures::class);

beforeEach(function () {
    $this->setUpFixtures();
});

it('returns 200 with valid structure for GET endpoints', function (array $endpoint) {
    $path = $this->substituteParams($endpoint['path'], $endpoint['params']);

    $response = $this->getJson("/api{$path}");

    expect($response->status())->toBe(200);
    expect($response->json())->toBeArray();
    expect($response->json())->toHaveKey('data');

    if ($endpoint['paginated']) {
        expect($response->json())->toHaveKey('meta');
    }
})->with(fn () => OpenApiEndpointExtractor::getTestableEndpoints());
```

## phpunit.xml Integration

```xml
<!-- Health-Check: Smoke test all GET endpoints (approx 10-15s) -->
<testsuite name="Health-Check">
    <directory>tests/Feature/HealthCheck</directory>
</testsuite>
```

## Usage

```bash
# Run health check suite
docker compose exec php ./vendor/bin/pest --testsuite=Health-Check

# CI quick check sequence
docker compose exec php ./vendor/bin/pest --testsuite=Unit-Pure
docker compose exec php ./vendor/bin/pest --testsuite=Health-Check
```

## Expected Results

- **Endpoints tested**: ~60-70 (after filtering multi-param routes)
- **Runtime**: ~10-15 seconds
- **Dependencies**: SQLite in-memory, no Meilisearch

```
PASS  Tests\Feature\HealthCheck\ApiEndpointHealthTest
✓ returns 200 with valid structure for GET endpoints with GET /v1/spells
✓ returns 200 with valid structure for GET endpoints with GET /v1/spells/{spell}
✓ returns 200 with valid structure for GET endpoints with GET /v1/monsters
...

Tests: 67 passed
Time: 12.34s
```

## Edge Cases

- **Auth-required endpoints**: Skip or expect 401 (character endpoints need auth)
- **Empty results**: Index endpoints with no matching data return `{"data": [], "meta": {...}}`
- **Missing fixtures**: Test fails clearly if param can't be substituted

## Future Enhancements (Not in Scope)

- POST/PUT/DELETE endpoint coverage
- Response schema validation against OpenAPI spec
- Performance regression detection (response time thresholds)
