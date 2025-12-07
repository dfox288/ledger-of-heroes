# Accordion Table Normalization Design

**Date**: 2025-11-23
**Status**: Approved
**Type**: Refactoring

## Problem Statement

Three accordion components display tabular data with inconsistent styling, creating visual inconsistencies across entity detail pages:

1. **UiAccordionClassCounters** - `px-4 py-3` padding, `bg-gray-50` header, mobile-responsive
2. **UiAccordionLevelProgression** - `px-3 py-2` padding, `bg-gray-800/50` header, desktop-only
3. **UiAccordionRandomTablesList** - `px-4 py-2` padding, `bg-gray-50` header, desktop-only, weird `pl-4` indent

**Additional Issues**:
- Only 1 of 3 tables has mobile-responsive layout
- 377 total lines of duplicated table markup code
- File organization inconsistency: 4 legacy components in `/ui/`, 15 in `/ui/accordion/`

## Goals

- ✅ Consistent table padding, colors, and widths across all table components
- ✅ All tables have mobile-responsive variants
- ✅ Reduced code duplication through shared base component
- ✅ Organized file structure (all accordions in `/ui/accordion/`)

## Constraints

- Must maintain backward compatibility (used across 6+ entity pages)
- TDD mandate: Tests first for any new components
- Must work in both light and dark modes
- Should be mobile-responsive

## Architecture

### Core Component: UiAccordionDataTable.vue

**Location**: `app/components/ui/accordion/UiAccordionDataTable.vue`

**Purpose**: Generic table renderer for accordion content with consistent styling and built-in mobile responsiveness.

**Props API**:
```typescript
interface Column {
  key: string           // Row data property key
  label: string         // Column header text
  align?: 'left' | 'center' | 'right'
  width?: string        // Optional width class (e.g., 'w-24')
  cellClass?: string    // Custom cell classes
}

interface Props {
  columns: Column[]
  rows: Record<string, any>[]
  mobileLayout?: 'stacked' | 'cards'  // Default: 'cards'
  striped?: boolean     // Zebra striping (default: true)
}
```

**Rendering Strategy**:
- **Desktop** (`md:` and up): Standard HTML `<table>` with consistent styling
- **Mobile** (`< md`): Responsive card layout (auto-switches at breakpoint)

**Standardized Styling**:
- Padding: `px-4 py-3` (from ClassCounters—most generous, best readability)
- Header: `bg-gray-50 dark:bg-gray-800`
- Body: `bg-white dark:bg-gray-900`
- Borders: `divide-gray-200 dark:divide-gray-700`
- Striping: Alternate row colors for readability

**Custom Cell Rendering**: Slots for badges, links, formatted values
```vue
<template #cell-{columnKey}="{ value, row }">
  <!-- Custom rendering -->
</template>
```

## Refactoring Strategy

### 1. UiAccordionClassCounters

**Current**: `app/components/ui/UiAccordionClassCounters.vue` (103 lines)
**After**: `app/components/ui/accordion/UiAccordionClassCounters.vue` (~50 lines)

**Changes**:
- Define columns: Level, Counter, Value, Reset Timing
- Use `#cell-reset_timing` slot for UBadge rendering
- Remove all custom table HTML and mobile layout code
- Keep sorting logic for counters by level
- Move from `/ui/` to `/ui/accordion/`

**Domain Logic Preserved**:
```typescript
// Sorting counters by level
const sortedCounters = computed(() => {
  return [...props.counters].sort((a, b) => a.level - b.level)
})
```

### 2. UiAccordionLevelProgression

**Current**: `app/components/ui/accordion/UiAccordionLevelProgression.vue` (143 lines)
**After**: `app/components/ui/accordion/UiAccordionLevelProgression.vue` (~80 lines)

**Changes**:
- Keep dynamic column calculation (hasCantrips, visibleSpellLevels)
- Generate column array programmatically
- Use slots for spell slot cells (the "—" for null values)
- Remove all custom table HTML
- **New feature**: Automatically gets mobile card layout

**Domain Logic Preserved**:
```typescript
// Dynamic column visibility
const hasCantrips = computed(() =>
  props.levelProgression.some(level => level.cantrips_known !== null && level.cantrips_known !== 0)
)

const visibleSpellLevels = computed(() => {
  const levels = []
  for (let i = 1; i <= 9; i++) {
    if (showSpellLevel(i)) levels.push(i)
  }
  return levels
})
```

### 3. UiAccordionRandomTablesList

**Current**: `app/components/ui/accordion/UiAccordionRandomTablesList.vue` (134 lines)
**After**: `app/components/ui/accordion/UiAccordionRandomTablesList.vue` (~70 lines)

**Changes**:
- Iterate over tables (outer loop remains)
- For each table, build columns dynamically (Roll + parsed pipe-delimited columns)
- Remove `pl-4` container indent (inconsistent with other components)
- Keep pipe-parsing logic for multi-column results
- Remove all custom table HTML

**Domain Logic Preserved**:
```typescript
// Parse pipe-delimited result_text into columns
const parseColumns = (resultText: string | null): string[] => {
  if (!resultText) return ['']
  return resultText.split('|').map(col => col.trim())
}
```

## File Organization Cleanup

**Move 4 components** from `/ui/` to `/ui/accordion/`:

1. `UiAccordionClassCounters.vue`
2. `UiAccordionConditions.vue`
3. `UiAccordionItemDetail.vue`
4. `UiAccordionPrerequisites.vue`

**Method**: Use `git mv` to preserve history

**Impact**: Zero breaking changes—Nuxt's auto-import resolves `<UiAccordionClassCounters>` correctly regardless of file location.

## Testing Strategy

### New Component Tests

**File**: `tests/components/ui/accordion/UiAccordionDataTable.test.ts`

**Test Cases**:
- ✅ Renders table with basic columns and rows
- ✅ Desktop table structure (thead, tbody, correct cell count)
- ✅ Mobile card layout rendering
- ✅ Striped rows (zebra striping)
- ✅ Custom cell slots work correctly
- ✅ Column alignment (left/center/right)
- ✅ Empty state (no rows)
- ✅ Dark mode classes applied

### Refactored Component Tests

**Update existing test files**:
- `UiAccordionClassCounters.test.ts` - Verify sorting, badges render
- `UiAccordionLevelProgression.test.ts` - Verify dynamic columns, spell slots
- `UiAccordionRandomTablesList.test.ts` - Verify pipe parsing, roll ranges

### Integration Tests

Run full test suite to ensure no regressions in parent pages:
- `app/pages/classes/[slug].vue`
- `app/pages/monsters/[slug].vue`
- Others using refactored components

## Implementation Order

### Phase 1: Foundation (TDD)
1. Write tests for UiAccordionDataTable (RED)
2. Implement UiAccordionDataTable (GREEN)
3. Refactor and polish (REFACTOR)

### Phase 2: Refactor Components (TDD for each)
1. Update UiAccordionClassCounters tests to expect new structure (RED)
2. Refactor UiAccordionClassCounters to use base table (GREEN)
3. Repeat for LevelProgression and RandomTablesList

### Phase 3: File Organization
1. Use `git mv` to move 4 components to `/ui/accordion/`
2. Verify auto-import still works
3. Run full test suite

### Phase 4: Verification & Documentation
1. Manual browser testing (all entity detail pages)
2. Test mobile responsive behavior
3. Update CHANGELOG.md
4. Commit all changes

## Success Metrics

### Before
- ❌ 3 different table padding schemes
- ❌ 2 different header colors
- ❌ Only 1 of 3 tables has mobile support
- ❌ 377 total lines of table markup code
- ❌ Inconsistent file organization (4 in `/ui/`, 15 in `/ui/accordion/`)

### After
- ✅ Single consistent padding: `px-4 py-3`
- ✅ Single header color: `bg-gray-50 dark:bg-gray-800`
- ✅ All 3 tables have mobile support
- ✅ ~200 lines of table markup code (47% reduction)
- ✅ All 19 accordion components in `/ui/accordion/`
- ✅ Reusable base component for future table needs

## Design Decisions

### Why Keep Domain-Specific Components?

We're keeping `UiAccordionClassCounters`, `UiAccordionLevelProgression`, etc. as thin wrappers because:

1. **Encapsulation of Domain Logic**: Each has specific sorting, filtering, or formatting logic
2. **Clean Parent Components**: Parents use simple, typed props instead of configuring base table
3. **Type Safety**: TypeScript types for domain entities (ClassCounter[], LevelProgression[])

**Think of it as**:
- `UiAccordionDataTable` = Generic table renderer (like `<table>` on steroids)
- `UiAccordionClassCounters` = Domain component that knows how to display class counters

### Standardized Padding Choice

Selected `px-4 py-3` (from ClassCounters) because:
- Most generous spacing (best readability)
- Already in production and looks good
- Avoids making tables feel cramped

### Mobile Layout Strategy

Using card layout instead of horizontal scrolling because:
- Better mobile UX (no pinch-to-zoom needed)
- Follows existing ClassCounters pattern (proven in production)
- Maintains readability on small screens

## Migration Path

**Backward Compatibility**: All refactored components maintain the same props interface—parent components require zero changes.

**Rollback Plan**: If issues arise, `git revert` the refactoring commits. Base component is additive, so it can be removed without affecting existing code.

## Future Opportunities

After this refactoring, we could:
- Use `UiAccordionDataTable` for new tabular data needs
- Consider consolidating key-value display patterns (separate effort)
- Explore badge list pattern extraction (separate effort)

These are out of scope for this design but become easier with the base component in place.
