# Slug-Based Character References Migration Guide

This guide documents the migration from ID-based to slug-based character references (Issue #288).

## Overview

Character data now uses **slugs** instead of database IDs for entity references. This enables:
- **Portability**: Characters can be exported and imported between different database instances
- **Survivability**: Character data survives sourcebook reimports that reassign IDs
- **Shareability**: Characters can be shared via JSON files

## What Changed

### Request Format

**Before (ID-based):**
```json
{
  "race_id": 42,
  "background_id": 15,
  "class_id": 7
}
```

**After (Slug-based):**
```json
{
  "race": "phb:high-elf",
  "background": "phb:sage",
  "class_slug": "phb:wizard"
}
```

### Response Format

**Before:**
```json
{
  "race_id": 42,
  "race": { "id": 42, "name": "High Elf" }
}
```

**After:**
```json
{
  "race_slug": "phb:high-elf",
  "race": { "id": 42, "name": "High Elf", "full_slug": "phb:high-elf" },
  "race_is_dangling": false
}
```

### Full Slug Format

All entities now have a `full_slug` in the format `{source_code}:{slug}`:
- `phb:high-elf` - Race from Player's Handbook
- `xge:shadow-blade` - Spell from Xanathar's Guide
- `core:common` - Universal lookups (languages, skills, conditions)

## New API Endpoints

### Character Export
```
GET /api/v1/characters/{character}/export
```

Exports a character as portable JSON:
```json
{
  "format_version": "1.0",
  "exported_at": "2025-01-07T12:00:00Z",
  "character": {
    "public_id": "brave-wizard-x7k2",
    "name": "Gandalf",
    "race": "phb:human",
    "background": "phb:sage",
    "classes": [
      {"class": "phb:wizard", "level": 5}
    ],
    "spells": [...],
    "equipment": [...]
  }
}
```

### Character Import
```
POST /api/v1/characters/import
Content-Type: application/json

{...export data...}
```

Response:
```json
{
  "success": true,
  "character": { "public_id": "brave-wizard-x7k2", "name": "Gandalf" },
  "warnings": [
    "Spell 'phb:wish' not found in database - reference preserved"
  ]
}
```

### Character Validation
```
GET /api/v1/characters/{character}/validate
```

Checks for dangling references:
```json
{
  "valid": true,
  "dangling_references": {},
  "summary": {
    "total_references": 25,
    "valid_references": 25,
    "dangling_count": 0
  }
}
```

## Dangling References

Characters can now reference entities that don't exist in the database. This is intentional:
- Enables importing characters with content from other sources
- API gracefully handles missing entities with `is_dangling: true` flag
- Use the validation endpoint to detect dangling references

## Frontend Migration Steps

### 1. Update Entity Selection
Replace `*_id` fields with slug-based references:

```typescript
// Before
const createCharacter = (data: {
  race_id: number;
  class_id: number;
}) => {...};

// After
const createCharacter = (data: {
  race: string;  // "phb:high-elf"
  class_slug: string;  // "phb:wizard"
}) => {...};
```

### 2. Use full_slug from Responses
Entity responses now include `full_slug`:

```typescript
// Store the full_slug for references
const selectedRace = raceResponse.full_slug; // "phb:high-elf"
```

### 3. Handle Dangling References
Check `is_dangling` flags in responses:

```typescript
if (character.race_is_dangling) {
  // Show warning: Race not found in database
}
```

### 4. Implement Export/Import
Add character sharing:

```typescript
// Export
const exportData = await api.get(`/characters/${id}/export`);
downloadAsJson(exportData, `${character.name}.json`);

// Import
const result = await api.post('/characters/import', exportData);
if (result.warnings.length > 0) {
  showWarnings(result.warnings);
}
```

## Database Changes

### New Columns
- `full_slug` added to: races, classes, backgrounds, spells, items, feats, languages, skills, conditions, optional_features
- Character tables now use `*_slug` columns instead of `*_id` foreign keys

### Migration Safety
- Existing characters are automatically migrated
- ID-based lookups still work for backwards compatibility
- Run validation endpoint to check for issues after reimport

## Rollback Considerations

If you need to rollback:
1. The `*_slug` columns reference entities by `full_slug`
2. Dangling references will remain (entities not in DB)
3. Run validation to identify affected characters
