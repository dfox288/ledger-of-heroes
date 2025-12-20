# Search Service Architecture

This document explains the internal architecture of search services for developers working on the backend.

For API consumer documentation (endpoints, filter syntax), see [SEARCH.md](./SEARCH.md).

## Overview

Search services handle Meilisearch queries and return hydrated Eloquent models with relationships. The `AbstractSearchService` base class provides common implementation to avoid code duplication.

```
Controller → SearchService → Meilisearch → Hydrate Models → API Resource
                   ↓
         AbstractSearchService (base class)
```

## Architecture Components

| Component | Purpose |
|-----------|---------|
| `AbstractSearchService` | Base class with common search logic |
| `{Entity}SearchService` | Entity-specific service (extends base) |
| `{Entity}SearchDTO` | Typed data transfer object for search params |
| `GlobalSearchService` | Cross-entity search (separate concern) |

## AbstractSearchService

Base class providing:

- `searchWithMeilisearch()` - Main search with pagination, filtering, result hydration
- `buildDatabaseQuery()` - Eloquent query for non-search pagination
- `buildScoutQuery()` - Scout builder for search
- `applySorting()` - Apply sort to Eloquent query
- `getDefaultRelationships()` - Backward compatibility alias

### Required Methods to Implement

```php
// 1. Return the model class
protected function getModelClass(): string
{
    return Spell::class;
}

// 2. Relationships for list endpoints (lightweight)
public function getIndexRelationships(): array
{
    return ['spellSchool', 'sources.source', 'classes'];
}

// 3. Relationships for detail endpoints (comprehensive)
public function getShowRelationships(): array
{
    return [...$this->getIndexRelationships(), 'tags', 'monsters', 'dataTables.entries'];
}
```

## Current State

### Services Extending AbstractSearchService (Correct)

| Service | Model |
|---------|-------|
| `SpellSearchService` | Spell |
| `BackgroundSearchService` | Background |
| `FeatSearchService` | Feat |
| `RaceSearchService` | Race |

### Services NOT Extending (Need Refactoring - Issue #814)

| Service | Model | Status |
|---------|-------|--------|
| `ItemSearchService` | Item | Duplicates base logic |
| `MonsterSearchService` | Monster | Duplicates base logic |
| `ClassSearchService` | CharacterClass | Duplicates base logic |
| `OptionalFeatureSearchService` | OptionalFeature | Duplicates base logic |

### Separate Concerns (Don't Extend)

| Service | Purpose |
|---------|---------|
| `GlobalSearchService` | Cross-entity search across all models |

## Adding a New Searchable Entity

### Step 1: Make Model Searchable

```php
// app/Models/NewEntity.php
use Laravel\Scout\Searchable;

class NewEntity extends Model
{
    use Searchable;

    public function toSearchableArray(): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'slug' => $this->slug,
            // Add denormalized fields for filtering
            'source_codes' => $this->sources->pluck('source.code')->toArray(),
        ];
    }

    public static function searchableOptions(): array
    {
        return [
            'filterableAttributes' => ['source_codes'],
            'sortableAttributes' => ['name'],
            'searchableAttributes' => ['name'],
        ];
    }
}
```

### Step 2: Create Search DTO

```php
// app/DTOs/NewEntitySearchDTO.php
final readonly class NewEntitySearchDTO
{
    public function __construct(
        public ?string $searchQuery = null,
        public ?string $meilisearchFilter = null,
        public int $page = 1,
        public int $perPage = 24,
        public string $sortBy = 'name',
        public string $sortDirection = 'asc',
    ) {}

    public static function fromRequest(Request $request): self
    {
        return new self(
            searchQuery: $request->input('q'),
            meilisearchFilter: $request->input('filter'),
            page: (int) $request->input('page', 1),
            perPage: (int) $request->input('per_page', 24),
            sortBy: $request->input('sort', 'name'),
            sortDirection: $request->input('direction', 'asc'),
        );
    }
}
```

### Step 3: Create Search Service

```php
// app/Services/NewEntitySearchService.php
final class NewEntitySearchService extends AbstractSearchService
{
    private const INDEX_RELATIONSHIPS = [
        'sources.source',
    ];

    private const SHOW_RELATIONSHIPS = [
        ...self::INDEX_RELATIONSHIPS,
        'relatedEntities',
        'dataTables.entries',
    ];

    protected function getModelClass(): string
    {
        return NewEntity::class;
    }

    public function getIndexRelationships(): array
    {
        return self::INDEX_RELATIONSHIPS;
    }

    public function getShowRelationships(): array
    {
        return self::SHOW_RELATIONSHIPS;
    }
}
```

### Step 4: Use in Controller

```php
// app/Http/Controllers/Api/NewEntityController.php
public function index(NewEntityIndexRequest $request, Client $client): AnonymousResourceCollection
{
    $dto = NewEntitySearchDTO::fromRequest($request);

    if ($dto->searchQuery || $dto->meilisearchFilter) {
        $results = $this->service->searchWithMeilisearch($dto, $client);
    } else {
        $results = $this->service->buildDatabaseQuery($dto)->paginate($dto->perPage);
    }

    return NewEntityResource::collection($results);
}
```

### Step 5: Configure Index and Import

```bash
# Add to search:configure-indexes command, then run:
just artisan search:configure-indexes

# Import to Meilisearch
just artisan scout:import "App\\Models\\NewEntity"
```

## Best Practices

### Relationship Separation

**Index relationships** should be minimal for list performance:
```php
private const INDEX_RELATIONSHIPS = [
    'primaryRelation',      // Always needed
    'sources.source',       // Common filter
];
```

**Show relationships** can be comprehensive:
```php
private const SHOW_RELATIONSHIPS = [
    ...self::INDEX_RELATIONSHIPS,
    'dataTables.entries',   // Heavy but needed for detail
    'relatedItems',         // Only on detail page
];
```

### DTO Requirements

The DTO must have these properties (checked by base class):
- `searchQuery` - Full-text search string
- `meilisearchFilter` - Raw Meilisearch filter expression
- `page` - Current page number
- `perPage` - Items per page
- `sortBy` - Sort field name
- `sortDirection` - 'asc' or 'desc'

### Error Handling

The base class wraps Meilisearch exceptions in `InvalidFilterSyntaxException`:
```php
try {
    $results = $client->index($indexName)->search($query, $params);
} catch (\MeiliSearch\Exceptions\ApiException $e) {
    throw new InvalidFilterSyntaxException($filter, $e->getMessage());
}
```

## Anti-Patterns

### Duplicating Base Class Logic

```php
// BAD - Don't copy searchWithMeilisearch() implementation
class ItemSearchService
{
    public function searchWithMeilisearch($dto, $client)
    {
        // 50 lines of duplicated code...
    }
}

// GOOD - Extend and inherit
class ItemSearchService extends AbstractSearchService
{
    // Only define relationships and model class
}
```

### Eloquent Filtering in Services

```php
// BAD - Using Eloquent for user-facing filters
$spells = Spell::whereHas('classes', fn($q) => $q->where('slug', 'wizard'))->get();

// GOOD - Use Meilisearch filter syntax
$results = $service->searchWithMeilisearch($dto, $client);
// With filter: "class_slugs IN [wizard]"
```

### Hardcoded Relationships

```php
// BAD - Relationships duplicated across methods
public function search() {
    return Item::with(['itemType', 'sources.source'])->get();
}
public function show($id) {
    return Item::with(['itemType', 'sources.source'])->find($id); // Duplicated!
}

// GOOD - Use constants
private const INDEX_RELATIONSHIPS = ['itemType', 'sources.source'];
```

## References

- [SEARCH.md](./SEARCH.md) - API consumer documentation
- [MEILISEARCH-FILTERS.md](./MEILISEARCH-FILTERS.md) - Filter syntax reference
- [05-code-patterns.md](../../.claude/rules/05-code-patterns.md) - Gold standards
- Issue #814 - Consolidate remaining services to extend AbstractSearchService
