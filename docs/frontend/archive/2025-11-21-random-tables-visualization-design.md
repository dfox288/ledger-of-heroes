# Random Tables Visualization Design

**Date:** 2025-11-21
**Status:** Approved - Ready for Implementation
**Feature:** Display D&D random tables for background traits

---

## Overview

Add visualization for random tables (e.g., "Entertainer Routines", "Personality Traits") that are nested within background trait data. Tables will be displayed inline with their parent trait using a new reusable component.

### Problem Statement

Background entities have rich random table data (dice type, roll ranges, results) that is currently not displayed in the UI. This data appears in the API response under `traits[].random_tables[]` but is ignored by the current implementation.

Example from "Entertainer" background:
- **Entertainer Routines** (d10): Actor, Dancer, Fire-eater, etc.
- **Personality Trait** (d8): Multiple character flavor options
- **Ideal** (d6): Alignment-based motivations
- **Bond** (d6): Character connections
- **Flaw** (d6): Character weaknesses

### User Value

- Players can quickly reference random tables when creating characters
- DMs can use tables during gameplay for NPC generation
- Maintains D&D 5e sourcebook authenticity
- Improves background page completeness (showing all API data)

---

## Design Decisions

### 1. Display Pattern: Embedded Within Traits

**Decision:** Display random tables inline immediately after their parent trait's description.

**Rationale:**
- Keeps contextual relationship clear (which trait owns which table)
- Matches user mental model (tables are part of the trait)
- No need to cross-reference between sections
- Simpler implementation (no data aggregation needed)

**Rejected Alternatives:**
- ❌ Dedicated "Random Tables" section - loses context
- ❌ Hybrid with preview links - added complexity, potential redundancy

### 2. Visual Style: HTML Tables with Borders

**Decision:** Use traditional `<table>` elements with header row and bordered cells.

**Rationale:**
- Most familiar format for tabular data
- Clear visual structure (Roll | Result columns)
- Matches D&D sourcebook presentation
- Accessible (semantic HTML, screen reader friendly)

**Styling Approach:**
- Tailwind CSS utility classes
- NuxtUI color palette for consistency
- Dark mode support
- Hover states for better UX
- Responsive (tables work well on mobile due to narrow "Roll" column)

**Rejected Alternatives:**
- ❌ Card-based list - less suitable for tabular data
- ❌ Compact grid - doesn't match existing patterns

### 3. Component Structure: Single Reusable Component

**Decision:** Create `UiAccordionRandomTablesList.vue` as a new reusable component.

**Rationale:**
- Follows existing accordion slot component patterns
- Could be reused for other entities (classes, races) if they add random tables
- Clean separation of concerns
- Easy to test in isolation

---

## Architecture

### Component Hierarchy

```
BackgroundDetailPage
└── UiAccordionTraitsList (modified)
    ├── Trait Display (existing)
    └── UiAccordionRandomTablesList (NEW)
        └── HTML Tables
```

### Data Flow

1. Background detail page fetches data via API (existing)
2. Passes `traits[]` array to `UiAccordionTraitsList` (existing)
3. Component loops through traits (existing)
4. For each trait, displays description (existing)
5. **NEW:** If `trait.random_tables.length > 0`, render `<UiAccordionRandomTablesList>`
6. New component renders each table as HTML table

### Files to Create

- `app/components/ui/accordion/UiAccordionRandomTablesList.vue` - New component
- `tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts` - New tests

### Files to Modify

- `app/components/ui/accordion/UiAccordionTraitsList.vue` - Add random tables integration
- `tests/components/ui/accordion/UiAccordionTraitsList.test.ts` - Add integration tests

---

## Component Design: UiAccordionRandomTablesList.vue

### TypeScript Interface

```typescript
interface RandomTableEntry {
  id: number
  roll_min: number
  roll_max: number
  result_text: string
  sort_order: number
}

interface RandomTable {
  id: number
  table_name: string
  dice_type: string
  description?: string | null
  entries: RandomTableEntry[]
}

interface Props {
  tables: RandomTable[]
  borderColor?: string  // Inherit from parent trait's border color
}
```

### Template Structure

```vue
<template>
  <div class="space-y-6 pl-4">
    <div v-for="table in tables" :key="table.id" class="space-y-2">
      <!-- Table Name and Dice Type -->
      <h4 class="text-base font-semibold text-gray-900 dark:text-gray-100">
        {{ table.table_name }}
        <span class="text-sm font-normal text-gray-600 dark:text-gray-400">
          ({{ table.dice_type }})
        </span>
      </h4>

      <!-- Optional Description -->
      <p v-if="table.description" class="text-sm text-gray-700 dark:text-gray-300">
        {{ table.description }}
      </p>

      <!-- HTML Table -->
      <table class="min-w-full border border-gray-200 dark:border-gray-700 rounded-lg overflow-hidden">
        <thead class="bg-gray-50 dark:bg-gray-800">
          <tr>
            <th class="px-4 py-2 text-left text-xs font-semibold uppercase tracking-wider text-gray-700 dark:text-gray-300 w-24">
              Roll
            </th>
            <th class="px-4 py-2 text-left text-xs font-semibold uppercase tracking-wider text-gray-700 dark:text-gray-300">
              Result
            </th>
          </tr>
        </thead>
        <tbody class="divide-y divide-gray-200 dark:divide-gray-700 bg-white dark:bg-gray-900">
          <tr
            v-for="entry in table.entries"
            :key="entry.id"
            class="hover:bg-gray-50 dark:hover:bg-gray-800/50 transition-colors"
          >
            <td class="px-4 py-3 text-sm font-medium text-gray-900 dark:text-gray-100">
              {{ formatRollRange(entry.roll_min, entry.roll_max) }}
            </td>
            <td class="px-4 py-3 text-sm text-gray-700 dark:text-gray-300">
              {{ entry.result_text }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>
```

### Script Logic

```typescript
<script setup lang="ts">
const props = withDefaults(defineProps<Props>(), {
  borderColor: 'primary-500'
})

const formatRollRange = (min: number, max: number): string => {
  return min === max ? `${min}` : `${min}-${max}`
}
</script>
```

### Styling Details

**Key Design Elements:**
- `space-y-6` - Vertical spacing between multiple tables
- `pl-4` - Left padding to align with trait description content
- `w-24` - Fixed width for "Roll" column (prevents layout shift)
- `rounded-lg` - Rounded table corners for polish
- `divide-y` - Horizontal dividers between rows (cleaner than full borders)
- `hover:bg-gray-50` - Row hover effect for better UX
- `uppercase tracking-wider` - Header styling matches D&D book style

**Dark Mode:**
- Background: `bg-white` / `dark:bg-gray-900`
- Borders: `border-gray-200` / `dark:border-gray-700`
- Text: `text-gray-900` / `dark:text-gray-100`
- Hover: `hover:bg-gray-50` / `dark:hover:bg-gray-800/50`

---

## Integration: UiAccordionTraitsList.vue Updates

### TypeScript Interface Update

```typescript
interface Trait {
  id: number
  name: string
  description: string
  level?: number
  category?: string
  feature_name?: string
  random_tables?: RandomTable[]  // NEW
}
```

### Template Modification

```vue
<template>
  <div class="p-4 space-y-4">
    <div v-for="trait in traits" :key="trait.id" class="space-y-3">
      <!-- Existing trait display -->
      <div class="border-l-4 pl-4 py-2" :class="`border-${borderColor}`">
        <div class="flex items-center gap-2 mb-1">
          <UBadge v-if="showLevel && trait.level" color="info" variant="soft" size="xs">
            Level {{ trait.level }}
          </UBadge>
          <span class="font-semibold text-gray-900 dark:text-gray-100">
            {{ trait.feature_name || trait.name }}
          </span>
          <UBadge v-if="showCategory && trait.category" color="purple" variant="soft" size="xs">
            {{ trait.category }}
          </UBadge>
        </div>
        <div v-if="trait.description" class="text-sm text-gray-700 dark:text-gray-300 whitespace-pre-line">
          {{ trait.description }}
        </div>
      </div>

      <!-- NEW: Random tables display -->
      <UiAccordionRandomTablesList
        v-if="trait.random_tables && trait.random_tables.length > 0"
        :tables="trait.random_tables"
        :borderColor="borderColor"
      />
    </div>
  </div>
</template>
```

**Why `space-y-3`?**
- Changed from `space-y-4` to group trait description + tables closer together
- Creates visual hierarchy: traits are separate, but tables belong to their trait

---

## Testing Strategy

### Unit Tests: UiAccordionRandomTablesList.test.ts

**Test Coverage (13 tests total):**

1. ✅ **Renders table name and dice type**
   - Verify heading displays "Entertainer Routines (d10)"

2. ✅ **Displays table description when present**
   - Verify optional description paragraph appears

3. ✅ **Hides description when null/undefined**
   - Verify no empty paragraph element

4. ✅ **Renders table header correctly**
   - Verify "Roll" and "Result" columns

5. ✅ **Renders all table entries**
   - Verify all 10 entries appear for Entertainer Routines

6. ✅ **Formats single roll correctly**
   - Verify "1" displays as "1" (not "1-1")

7. ✅ **Formats roll range correctly**
   - Verify entries with roll_min=1, roll_max=3 display as "1-3"

8. ✅ **Handles multiple tables with proper spacing**
   - Verify space-y-6 applies between tables

9. ✅ **Handles empty tables array gracefully**
   - Verify nothing renders when tables=[]

10. ✅ **Applies border color from prop**
    - Verify borderColor prop is accepted (even if unused in v1)

11. ✅ **Renders result_text correctly**
    - Verify "Actor", "Dancer", etc. appear in Result column

12. ✅ **Maintains sort order**
    - Verify entries render in sort_order sequence

13. ✅ **Component mounts without errors**
    - Basic smoke test

### Integration Tests: UiAccordionTraitsList.test.ts (3 new tests)

14. ✅ **Renders random tables when trait has them**
    - Pass trait with random_tables, verify component appears

15. ✅ **Passes border color to random tables component**
    - Verify borderColor prop is forwarded

16. ✅ **Doesn't render random tables component when array is empty**
    - Pass trait with random_tables=[], verify component doesn't render

### Manual Browser Testing

**Test Plan:**
1. Navigate to `http://localhost:3000/backgrounds/entertainer`
2. Verify 5 random tables render:
   - Entertainer Routines (d10)
   - Personality Trait (d8)
   - Ideal (d6)
   - Bond (d6)
   - Flaw (d6)
3. Check table formatting:
   - Roll ranges display correctly (1, 2, 3 vs 1-3)
   - Result text appears
   - Tables have proper borders
4. Test responsive behavior:
   - Mobile (375px) - tables should remain readable
   - Tablet (768px)
   - Desktop (1440px)
5. Test dark mode:
   - Toggle dark mode
   - Verify all table elements have proper contrast
   - Check hover states
6. Test other backgrounds:
   - Check backgrounds without random tables (ensure no errors)
   - Check backgrounds with different table structures

---

## Implementation Workflow (TDD)

### Phase 1: Write Tests (RED)

```bash
# Create test file
touch tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts

# Write all 13 test cases
# Run tests - verify they FAIL
docker compose exec nuxt npm run test -- UiAccordionRandomTablesList.test.ts
```

**Expected Result:** All tests fail (component doesn't exist yet)

### Phase 2: Minimal Implementation (GREEN)

```bash
# Create component file
touch app/components/ui/accordion/UiAccordionRandomTablesList.vue

# Implement component (minimal code to pass tests)
# Run tests - verify they PASS
docker compose exec nuxt npm run test -- UiAccordionRandomTablesList.test.ts
```

**Expected Result:** All 13 tests pass

### Phase 3: Integration

```bash
# Update UiAccordionTraitsList.vue
# - Add TypeScript interface
# - Add template integration
# - Add new tests

# Run full test suite
docker compose exec nuxt npm run test
```

**Expected Result:** All tests pass, no regressions

### Phase 4: Manual Verification

```bash
# Start dev server (if not running)
docker compose up -d

# Open browser
open http://localhost:3000/backgrounds/entertainer

# Verify visually
# - Tables render correctly
# - Styling looks good
# - Dark mode works
# - Responsive behavior
```

**Expected Result:** Tables display as designed, no visual issues

### Phase 5: Commit

```bash
git add app/components/ui/accordion/UiAccordionRandomTablesList.vue
git add tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts
git add app/components/ui/accordion/UiAccordionTraitsList.vue
git add tests/components/ui/accordion/UiAccordionTraitsList.test.ts

git commit -m "feat: Add random tables visualization for backgrounds

- Created UiAccordionRandomTablesList component with TDD
- Added 13 unit tests (all passing)
- Integrated into UiAccordionTraitsList
- Tables display inline with trait descriptions
- Supports roll ranges (e.g., 1-3 vs single rolls)
- Full dark mode and responsive support
- Verified with Entertainer background (5 tables)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Edge Cases & Considerations

### Data Quality

**Empty entries array:**
- Component should handle `entries: []` gracefully
- Display table header but empty tbody (or hide entire table?)
- Decision: Hide entire table if `entries.length === 0`

**Missing fields:**
- `description` is optional (already handled with `v-if`)
- `roll_min` / `roll_max` should always exist (backend guarantees)
- `result_text` should always exist (backend guarantees)

**Sort order:**
- API provides `sort_order` field
- Entries should already be sorted by backend
- Frontend renders in array order (trusts backend sorting)

### Performance

**Number of tables:**
- Entertainer has 5 tables (~30 total entries)
- Rendering performance is negligible for this data size
- No virtualization needed

**Future optimization:**
- If backgrounds add 20+ tables, consider lazy loading
- Current implementation is sufficient for known data

### Accessibility

**Semantic HTML:**
- Using proper `<table>`, `<thead>`, `<tbody>` elements
- Screen readers will announce "table with 2 columns, X rows"

**Keyboard Navigation:**
- Tables are naturally focusable and navigable
- No interactive elements inside tables (no additional accessibility concerns)

**Color Contrast:**
- Text colors meet WCAG AA standards
- Tested with dark mode

### Browser Compatibility

**Table rendering:**
- Standard HTML tables (universal support)
- Tailwind CSS utilities (modern browser support)
- No advanced CSS features required

**Testing targets:**
- Chrome/Edge (Chromium)
- Firefox
- Safari
- Mobile browsers (iOS Safari, Chrome Mobile)

---

## Future Enhancements (Out of Scope)

**Not included in this implementation:**

1. **Interactive dice rolling**
   - Click a table row to "roll" and highlight random result
   - Would require additional state management and animations

2. **Table filtering/search**
   - Filter table entries by keyword
   - Not needed for current data sizes (max 10 entries per table)

3. **Export/print functionality**
   - Print-friendly table formatting
   - CSV export
   - Not requested, can add later if needed

4. **Roll history**
   - Track previous rolls
   - Would require client-side storage

5. **Custom table theming**
   - Per-table border colors
   - Currently inherits from parent trait's borderColor
   - Can enhance if needed

---

## Success Criteria

**Definition of Done:**

- [ ] ✅ `UiAccordionRandomTablesList.vue` component created
- [ ] ✅ All 13 unit tests written and passing
- [ ] ✅ Integration tests written and passing
- [ ] ✅ Full test suite passes (no regressions)
- [ ] ✅ Manual browser testing completed
- [ ] ✅ Dark mode verified
- [ ] ✅ Responsive behavior verified
- [ ] ✅ TypeScript compiles with no errors
- [ ] ✅ ESLint passes with no warnings
- [ ] ✅ Work committed with descriptive message
- [ ] ✅ Feature works on Entertainer background
- [ ] ✅ Feature handles backgrounds without random tables

**User Acceptance:**
- Players can view all random tables for backgrounds
- Tables are clearly formatted and easy to read
- Dark mode works correctly
- Mobile users can read tables comfortably

---

## API Data Reference

**Example API Response (Entertainer background):**

```json
{
  "traits": [
    {
      "id": 833,
      "name": "Entertainer Routines",
      "category": "flavor",
      "description": "A good entertainer is versatile...",
      "random_tables": [
        {
          "id": 168,
          "table_name": "Entertainer Routines",
          "dice_type": "d10",
          "description": null,
          "entries": [
            {
              "id": 725,
              "roll_min": 1,
              "roll_max": 1,
              "result_text": "Actor",
              "sort_order": 0
            },
            {
              "id": 726,
              "roll_min": 2,
              "roll_max": 2,
              "result_text": "Dancer",
              "sort_order": 1
            }
            // ... more entries
          ]
        }
      ]
    }
  ]
}
```

**Key Fields:**
- `table_name` - Display as heading (e.g., "Entertainer Routines")
- `dice_type` - Display in parentheses (e.g., "(d10)")
- `description` - Optional explanatory text
- `entries[]` - Array of table rows
  - `roll_min` / `roll_max` - Roll range (format as "1" or "1-3")
  - `result_text` - The table result
  - `sort_order` - Backend-provided ordering

---

## Conclusion

This design provides a clean, accessible, and maintainable solution for displaying D&D random tables within background traits. The implementation follows existing component patterns, uses TDD methodology, and integrates seamlessly with the current UI architecture.

**Next Steps:**
1. Set up git worktree for isolated development
2. Create detailed implementation plan
3. Begin TDD implementation (tests first!)
