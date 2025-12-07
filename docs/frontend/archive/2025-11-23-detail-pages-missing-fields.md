# Detail Pages Missing Fields - Design Document

**Date:** 2025-11-23
**Status:** Approved
**Approach:** Component-First Architecture

---

## Problem Statement

Comprehensive audit of all 7 detail pages revealed missing API fields and inconsistencies:

### Missing Fields by Entity

- **Classes (85% coverage):**
  - Missing `counters` (ClassCounterResource[]) - Resource tracking like Rage, Ki Points
  - Missing `spells` relationship - Not needed inline, separate endpoint exists

- **Feats (80% coverage):**
  - Missing `proficiencies` (ProficiencyResource[]) - Proficiency grants
  - Missing `conditions` (EntityConditionResource[]) - Condition-related mechanics

- **Monsters (95% coverage):**
  - Missing `conditions` (EntityConditionResource[]) - Immunities/vulnerabilities

- **Items (95% coverage):**
  - Missing `detail` (string|null) - Flavor text and usage notes
  - Missing `prerequisites` (EntityPrerequisiteResource[]) - Usage requirements

### Critical Inconsistencies

1. **Conditions Display:** Beautifully implemented for Races, completely absent from Monsters/Feats
2. **Image Components:** Mix of `UiDetailEntityImage`, `UCard` wrappers, and raw `<NuxtImg>` tags
3. **Proficiencies:** Displayed for Races/Classes/Backgrounds/Items but missing from Feats

---

## Architecture Decision

**Chosen Approach:** Component-First (Approach 1)

### Rationale

1. **Consistency with existing codebase** - Already uses component-based architecture
2. **Conditions appear in 3 entities** - Component ensures uniform display
3. **TDD mandate** - Components are easier to test in isolation
4. **Maintainability** - Centralized logic, single source of truth
5. **Future-proof** - New entities can immediately use these components

### Trade-offs Accepted

- ⚠️ ~20% more upfront time creating components vs. inline implementation
- ✅ Saved back on first bug fix or feature addition
- ✅ Better code quality and test coverage

---

## Component Architecture

### New Components

```
app/components/ui/
├── UiAccordionConditions.vue          - Conditions display (Races, Feats, Monsters)
├── UiAccordionClassCounters.vue       - Class resource tracking table
├── UiAccordionItemDetail.vue          - Item detail field display
└── UiAccordionPrerequisites.vue       - Prerequisites display (Items)

tests/components/ui/
├── UiAccordionConditions.test.ts
├── UiAccordionClassCounters.test.ts
├── UiAccordionItemDetail.test.ts
└── UiAccordionPrerequisites.test.ts
```

### Updated Pages

```
app/pages/
├── classes/[slug].vue     - Add counters accordion
├── feats/[slug].vue       - Add proficiencies & conditions accordions
├── monsters/[slug].vue    - Add conditions accordion
├── items/[slug].vue       - Add detail & prerequisites accordions
└── backgrounds/[slug].vue - Standardize image component
```

---

## Component Specifications

### 1. UiAccordionConditions

**Purpose:** Display entity conditions with effect type and descriptions.

**Props:**
```typescript
interface Props {
  conditions: EntityConditionResource[]
  entityType?: 'race' | 'feat' | 'monster'  // Optional semantic coloring
}
```

**Features:**
- Badge showing condition name (color-coded by effect type)
- Effect type display: immunity, resistance, vulnerability, inflicts
- Condition description from the condition resource
- Optional entity-specific description from pivot table
- Grid layout for multiple conditions
- Empty state handling

**Based on:** `app/pages/races/[slug].vue` lines 287-320

**Usage:**
```vue
<UiAccordionConditions
  :conditions="monster.conditions"
  entity-type="monster"
/>
```

---

### 2. UiAccordionClassCounters

**Purpose:** Display class resource counters across 20 levels.

**Props:**
```typescript
interface Props {
  counters: ClassCounterResource[]
}
```

**Interface:**
```typescript
interface ClassCounterResource {
  id: number
  level: number
  counter_name: string
  counter_value: number
  reset_timing: 'Short Rest' | 'Long Rest' | 'Does Not Reset'
}
```

**Features:**
- Table format: Level | Counter Name | Value | Reset Timing
- Grouped by counter type (all "Rage" rows together)
- Color-coded reset timing:
  - Short Rest = blue (`info`)
  - Long Rest = purple (`primary`)
  - Does Not Reset = neutral
- Responsive: stacks on mobile, table on desktop
- Shows all 20 levels as decided
- Empty state if no counters

**Pattern:** Similar to `UiAccordionLevelProgression`

**Usage:**
```vue
<UiAccordionClassCounters :counters="entity.counters" />
```

---

### 3. UiAccordionItemDetail

**Purpose:** Display item detail field (flavor text, usage notes).

**Props:**
```typescript
interface Props {
  detail: string | null
}
```

**Features:**
- Simple styled text display
- Italic styling to differentiate from description
- Whitespace-preserved for multi-line content
- Only renders if detail is not null

**Usage:**
```vue
<UiAccordionItemDetail
  v-if="item.detail"
  :detail="item.detail"
/>
```

---

### 4. UiAccordionPrerequisites

**Purpose:** Display item prerequisites (beyond attunement boolean).

**Props:**
```typescript
interface Props {
  prerequisites: EntityPrerequisiteResource[]
}
```

**Interface:**
```typescript
interface EntityPrerequisiteResource {
  id: number
  prerequisite_type: string | null
  prerequisite_id: number | null
  minimum_value: number | null
  description: string | null
  group_id: number
  ability_score?: AbilityScoreResource
  race?: RaceResource
  skill?: SkillResource
  proficiency_type?: ProficiencyTypeResource
}
```

**Features:**
- Bullet list of prerequisites
- Smart formatting:
  - Ability scores: "Strength 13 or higher"
  - Race: "Must be a Dwarf"
  - Proficiency: "Must be proficient with martial weapons"
  - Custom: Shows description field
- Grouped by `group_id`:
  - OR requirements within same group
  - AND requirements between groups
- Empty state handling

**Based on:** `app/pages/feats/[slug].vue` lines 65-94 (adapted)

**Usage:**
```vue
<UiAccordionPrerequisites
  :prerequisites="item.prerequisites"
/>
```

---

## Data Flow

```
API Response
    ↓
useEntityDetail composable
    ↓
Page Component (e.g., monsters/[slug].vue)
    ↓
UI Component (e.g., UiAccordionConditions)
    ↓
Rendered Display
```

**Example for Conditions:**

1. API returns:
```json
{
  "conditions": [
    {
      "id": 1,
      "condition_id": 5,
      "condition": {
        "id": 5,
        "name": "Poisoned",
        "description": "A poisoned creature has disadvantage..."
      },
      "effect_type": "immunity",
      "description": null
    }
  ]
}
```

2. Page receives full entity object via `useEntityDetail`
3. Page passes `monster.conditions` to component
4. Component renders badges, descriptions, effect types

---

## Integration Strategy

### Implementation Order (TDD)

**Phase 1: Component Creation**
1. Write test for `UiAccordionConditions` (RED)
2. Implement component (GREEN)
3. Refactor and test edge cases
4. Repeat for remaining 3 components

**Phase 2: Page Integration**
1. Update Classes page - add counters accordion
2. Update Feats page - add proficiencies & conditions
3. Update Monsters page - add conditions accordion
4. Update Items page - add detail & prerequisites
5. Test each page with `mountSuspended`
6. Browser verification (light/dark mode, responsive)

**Phase 3: Standardization**
1. Update Backgrounds page to use `UiDetailStandaloneImage`
2. Remove extra `UCard` wrapper from Monsters image
3. Ensure consistent spacing/layout across all pages

---

## Testing Strategy

### Component Tests (Vitest + @nuxt/test-utils)

**UiAccordionConditions.test.ts:**
- ✅ Renders multiple conditions
- ✅ Shows effect type badge with correct color
- ✅ Displays condition description
- ✅ Shows entity-specific description if provided
- ✅ Empty state when conditions array is empty
- ✅ Handles null/undefined gracefully

**UiAccordionClassCounters.test.ts:**
- ✅ Renders table with all 20 levels
- ✅ Groups counters by name
- ✅ Shows reset timing with color coding
- ✅ Responsive layout (test at mobile/desktop breakpoints)
- ✅ Empty state when no counters
- ✅ Sorts by level ascending

**UiAccordionItemDetail.test.ts:**
- ✅ Renders detail text
- ✅ Preserves whitespace
- ✅ Applies italic styling
- ✅ Does not render if detail is null

**UiAccordionPrerequisites.test.ts:**
- ✅ Renders ability score requirements
- ✅ Renders race requirements
- ✅ Renders proficiency requirements
- ✅ Groups OR requirements (same group_id)
- ✅ Shows AND between groups
- ✅ Shows custom descriptions
- ✅ Empty state

### Page Tests

Update existing page component tests:
- ✅ Verify new accordion sections appear when data exists
- ✅ Verify accordion sections hidden when data missing
- ✅ Data flows correctly from API mock to component
- ✅ Conditional rendering works

---

## Error Handling

### Missing Data
- All accordion sections use conditional rendering:
  ```vue
  v-if="entity.conditions && entity.conditions.length > 0"
  ```
- Components handle empty arrays gracefully
- No accordion item appears if data missing (existing pattern)

### Malformed Data
- Optional chaining: `condition?.name`, `prerequisite?.ability_score?.code`
- Fallback text: `'Unknown'` or empty string
- TypeScript compile-time type safety
- Runtime type validation in components

---

## Styling Consistency

### Following Existing Patterns

**Accordion Sections:**
- Use `UAccordion` with `type="multiple"`
- Default closed (`defaultOpen: false`)
- Slot-based content

**Cards:**
- Use `UCard` with `#header` template slot
- Consistent header styling: `text-xl font-semibold`

**Badges:**
- Use `UBadge` with semantic colors from `utils/badgeColors.ts`
- Size: `default` or `sm`
- Variant: `soft` or `subtle`

**Spacing:**
- Major sections: `space-y-8`
- Within components: `space-y-3`
- Grid gaps: `gap-3` or `gap-4`

**Colors:**
- Primary: Classes, spells, magic
- Warning: Feats, legendary monsters
- Info: Informational badges
- Neutral: Secondary information
- Success: Backgrounds
- Error: High CR monsters

### New Color Utilities (if needed)

**For Class Counters:**
```typescript
// app/utils/badgeColors.ts
export function getResetTimingColor(timing: string): BadgeColor {
  switch (timing) {
    case 'Short Rest': return 'info'
    case 'Long Rest': return 'primary'
    case 'Does Not Reset': return 'neutral'
    default: return 'neutral'
  }
}
```

**For Condition Effects:**
```typescript
// app/utils/badgeColors.ts
export function getConditionEffectColor(effectType: string): BadgeColor {
  switch (effectType) {
    case 'immunity': return 'success'
    case 'resistance': return 'info'
    case 'vulnerability': return 'error'
    case 'inflicts': return 'warning'
    default: return 'neutral'
  }
}
```

---

## Success Criteria

### Functional Requirements
- ✅ All missing API fields displayed on detail pages
- ✅ Conditions displayed consistently across Races, Feats, Monsters
- ✅ Class counters show all 20 levels in table format
- ✅ Feat proficiencies displayed
- ✅ Item detail and prerequisites displayed
- ✅ Image components standardized

### Quality Requirements
- ✅ All new components have test coverage >90%
- ✅ All tests pass (`npm run test`)
- ✅ TypeScript compiles without errors (`npm run typecheck`)
- ✅ ESLint passes (`npm run lint`)
- ✅ Browser verification (Chrome/Firefox, light/dark mode)
- ✅ Mobile responsive (375px, 768px, 1440px)
- ✅ Keyboard accessible
- ✅ Work committed immediately after completion

---

## Rollout Plan

### Step 1: Create Components (TDD)
1. `UiAccordionConditions` + test
2. `UiAccordionClassCounters` + test
3. `UiAccordionItemDetail` + test
4. `UiAccordionPrerequisites` + test

### Step 2: Integrate into Pages
1. Classes page (counters)
2. Feats page (proficiencies + conditions)
3. Monsters page (conditions)
4. Items page (detail + prerequisites)

### Step 3: Standardization
1. Backgrounds page (image component)
2. Monsters page (remove extra UCard)

### Step 4: Verification
1. Run full test suite
2. Browser testing (all breakpoints)
3. Dark mode verification
4. Accessibility check

### Step 5: Documentation
1. Update CHANGELOG.md
2. Commit with detailed message

---

## Future Considerations

### Potential Enhancements (Not in Scope)
- Class spells inline display (currently handled by separate endpoint)
- Expandable condition descriptions (hover tooltips)
- Counter value calculator (e.g., "at your level: X")
- Prerequisites validation display (show if player meets requirements)

### Technical Debt Addressed
- ✅ Conditions display inconsistency resolved
- ✅ Image component standardization
- ✅ Missing API fields now utilized
- ✅ Test coverage improved

---

## References

- **Audit Report:** Internal analysis (2025-11-23)
- **API Schema:** `app/types/api/generated.ts`
- **Existing Components:** `app/components/ui/`
- **CLAUDE.md:** TDD mandate, component patterns
- **NuxtUI Docs:** https://ui.nuxt.com/docs
