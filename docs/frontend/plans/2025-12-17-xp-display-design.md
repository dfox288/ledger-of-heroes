# XP Progress Display Design

**Issue:** #653
**Date:** 2025-12-17
**Status:** Approved

## Overview

Add an XP progress bar below the character name/class info in the Header component. Clicking opens an edit modal in play mode. Hidden at max level.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Location | Header, below name/class line | Natural reading flow, doesn't crowd existing elements |
| Visualization | Progress bar + text | Clear visual indicator, shows current and target |
| Max level behavior | Hidden | Level 20 characters don't need XP tracking |
| Editing | Modal in play mode | Consistent with HP editing pattern |

## Components

### 1. `CharacterSheetXpBar.vue` (new)

Location: `app/components/character/sheet/XpBar.vue`

Props:
- `xpData`: XP response from API
- `isPlayMode`: boolean
- `characterId`: string

Emits:
- `edit`: when clicked in play mode

Behavior:
- Displays progress bar + "6,500 / 14,000 XP" text
- Hidden when `is_max_level: true`
- Clickable only in play mode

### 2. `XpEditModal.vue` (new)

Location: `app/components/character/sheet/XpEditModal.vue`

Props:
- `open`: boolean (v-model)
- `currentXp`: number
- `characterId`: string

Emits:
- `saved`: after successful update

Behavior:
- Single number input, pre-filled with current XP
- Validates non-negative integer
- Calls POST endpoint on save
- Shows toast on success

### 3. `Header.vue` (modify)

Changes:
- Add XpBar below the race/class/background line
- Accept `xpData` and `isPlayMode` props
- Emit `edit-xp` event for modal control

### 4. Character Page (modify)

Location: `app/pages/characters/[publicId]/index.vue`

Changes:
- Use `useCharacterXp(publicId)` composable
- Pass XP data to Header
- Handle modal open/close state

## Data Flow

### API Endpoints

```
GET /api/v1/characters/{id}/xp
Response: {
  "data": {
    "experience_points": 6500,
    "level": 5,
    "next_level_xp": 14000,
    "xp_to_next_level": 7500,
    "xp_progress_percent": 46.67,
    "is_max_level": false
  }
}

POST /api/v1/characters/{id}/xp
Body: { "experience_points": 7000 }
Response: Same shape as GET
```

### Nitro Routes

- `server/api/characters/[id]/xp.get.ts`
- `server/api/characters/[id]/xp.post.ts`

### Composable: `useCharacterXp`

```typescript
function useCharacterXp(characterId: MaybeRef<string>) {
  // Returns:
  // - xpData: Ref<XpData | null>
  // - pending: Ref<boolean>
  // - error: Ref<Error | null>
  // - updateXp(newValue: number): Promise<void>
  // - refresh(): Promise<void>
}
```

## UI/UX Details

### Progress Bar
- Component: NuxtUI `UProgress`
- Color: `primary` (neutral blue)
- Size: `sm`
- Width: Matches name/class text width

### Text Format
- Format: "6,500 / 14,000 XP"
- Style: `text-sm text-gray-500 dark:text-gray-400`
- Numbers comma-formatted

### Interaction
- Play mode: cursor pointer, hover highlight, click opens modal
- Non-play mode: display only, no cursor change

### Edit Modal
- Title: "Edit Experience Points"
- Single number input
- Cancel / Save buttons
- Success toast: "Experience points updated"

## Testing Strategy

### Unit Tests

**CharacterSheetXpBar.test.ts:**
- Renders progress bar with correct percentage
- Shows formatted XP text
- Hidden when `is_max_level: true`
- Clickable in play mode only
- Emits `edit` event on click

**XpEditModal.test.ts:**
- Opens with current XP value
- Validates input (non-negative integer)
- Calls API on save
- Shows toast on success
- Closes on cancel

**useCharacterXp.test.ts:**
- Fetches XP data on init
- Returns correct reactive state
- `updateXp()` calls POST and refreshes

### Integration Tests
- Header shows XP bar when data loads
- Edit flow: click -> modal -> save -> updated display

## Files Summary

### New Files
- `app/components/character/sheet/XpBar.vue`
- `app/components/character/sheet/XpEditModal.vue`
- `app/composables/useCharacterXp.ts`
- `server/api/characters/[id]/xp.get.ts`
- `server/api/characters/[id]/xp.post.ts`
- `tests/components/character/sheet/XpBar.test.ts`
- `tests/components/character/sheet/XpEditModal.test.ts`
- `tests/composables/useCharacterXp.test.ts`

### Modified Files
- `app/components/character/sheet/Header.vue`
- `app/pages/characters/[publicId]/index.vue`
