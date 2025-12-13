# Character Export/Import Design

**Issue:** [#549](https://github.com/dfox288/ledger-of-heroes/issues/549)
**Created:** 2025-12-13

## Overview

Add UI for character export and import functionality. Backend endpoints already exist:
- `GET /api/v1/characters/{id}/export` - Returns portable JSON format (v1.1)
- `POST /api/v1/characters/import` - Creates character from JSON

## Export Feature

### Location
Character sheet Header.vue → Actions dropdown menu

### User Flow
1. User opens character sheet (draft or complete)
2. Clicks "Actions" dropdown
3. Selects "Export Character"
4. Browser downloads JSON file: `{publicId}-{YYYY-MM-DD-HHmm}.json`
5. Toast: "Character exported"

### Technical Implementation
- Add "Export Character" to `actionMenuItems` in Header.vue (always available, own section)
- Emit `export` event to parent page
- Parent calls `apiFetch('/characters/${id}/export')`
- Use `URL.createObjectURL()` + hidden `<a>` download pattern
- No new component needed

### Nitro Route
```
server/api/characters/[id]/export.get.ts
```

### Error Handling
- Network error → Toast: "Failed to export character"
- 404 → Toast: "Character not found"

## Import Feature

### Location
Character list page (`/characters/index.vue`) header, next to "Create Character" button

### User Flow
1. User visits character list page
2. Sees "Import Character" button (ghost variant)
3. Clicks button → Modal opens with tabs: "Upload File" | "Paste JSON"
4. **Upload tab:** File dropzone accepting `.json`, shows filename when selected
5. **Paste tab:** Textarea with placeholder
6. User provides JSON via either method
7. Clicks "Import" button
8. Success: Modal closes, toast "Character imported!", navigates to new character
9. Error: Inline error message in modal

### New Component
```
app/components/character/ImportModal.vue
```
- Uses `UModal` with `UTabs` inside
- Props: `open` (v-model)
- Emits: `import` with parsed JSON payload
- Client-side validation: checks `format_version` and `character` keys

### Nitro Route
```
server/api/characters/import.post.ts
```

### Error Handling
- Invalid JSON parse → "Invalid JSON format"
- Missing `format_version` → "Missing format version"
- Missing `character` data → "Missing character data"
- Backend 422 → Show backend's message
- Network error → "Failed to import. Please try again."

## API Response Shapes

### Export Response
```json
{
  "data": {
    "format_version": "1.1",
    "exported_at": "2025-12-13T08:50:20+00:00",
    "character": {
      "public_id": "arcane-grove-10QL",
      "name": "Character Name",
      "race": "phb:human",
      ...
    }
  }
}
```

### Import Request
```json
{
  "format_version": "1.1",
  "character": { /* exported character object */ }
}
```

### Import Response (Success)
```json
{
  "data": {
    "id": 123,
    "public_id": "new-phoenix-XyZ1",
    "name": "Imported Character"
  }
}
```

## Files to Create/Modify

### New Files
- `server/api/characters/[id]/export.get.ts`
- `server/api/characters/import.post.ts`
- `app/components/character/ImportModal.vue`
- `tests/components/character/ImportModal.test.ts`

### Modified Files
- `app/components/character/sheet/Header.vue` - Add export action
- `app/pages/characters/[publicId]/index.vue` - Add handleExport
- `app/pages/characters/index.vue` - Add import button + modal
- `tests/components/character/sheet/Header.test.ts` - Test export action

## Test Coverage

| Component | Tests |
|-----------|-------|
| `ImportModal.vue` | Renders tabs, file upload, paste, validates JSON, emits, shows errors |
| `Header.vue` | Export action in dropdown, emits export event |
| Character list page | Import button visible, modal opens |
| Character sheet page | handleExport triggers download |
