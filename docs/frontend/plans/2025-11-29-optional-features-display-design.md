# Optional Features Display Design

**Date**: 2025-11-29
**Status**: Approved
**Feature**: Display Eldritch Invocations, Artificer Infusions, Elemental Disciplines, etc.

---

## Overview

Add display of `optional_features` (Eldritch Invocations, Infusions, Elemental Disciplines, Metamagic, etc.) across the three class detail views in a D&D-appropriate manner.

### Data Available from API

```typescript
interface OptionalFeatureResource {
  id: number
  slug: string
  name: string
  feature_type: string           // e.g., "eldritch_invocation"
  feature_type_label: string     // e.g., "Eldritch Invocation"
  level_requirement: number | null
  prerequisite_text: string | null
  description: string
  casting_time: string | null    // For spell-like features
  range: string | null
  duration: string | null
  resource_type: string | null
  resource_cost: number | null
  has_spell_mechanics: boolean
  sources?: EntitySourceResource[]
}
```

### Classes with Optional Features

| Class/Subclass | Feature Type | Count |
|----------------|--------------|-------|
| Warlock | Eldritch Invocations | 54 |
| Monk (Four Elements) | Elemental Disciplines | 17 |
| Artificer | Infusions | 16 |
| Sorcerer | Metamagic | TBD |
| Fighter (Battle Master) | Maneuvers | TBD |

---

## Design by View

### Overview View: Teaser Card

Simple card showing available class options with count and link to Journey view.

**Component**: `ClassOverviewOptionsCard`

**Display**:
```
┌─────────────────────────────────────────────────┐
│ 🔮 Class Options                                │
│                                                 │
│ Eldritch Invocations          54 choices        │
│ Customize your warlock with mystic powers       │
│                                                 │
│                        [View in Journey →]      │
└─────────────────────────────────────────────────┘
```

**Behavior**:
- Only shows if `optional_features.length > 0`
- Groups by `feature_type_label` if multiple types exist
- Links to Journey view for full exploration

---

### Journey View: "Available Options" Section

Separate section after features at each level showing what options unlock or are available.

**Component**: `ClassJourneyOptionsSection`

**Display**:
```
Level 2 ────────────────────────────────────
  ▪ Eldritch Invocations
    You learn two invocations of your choice...

  ┌─────────────────────────────────────────────┐
  │ 📜 Available Options (32)                   │
  │                                             │
  │ ▼ Eldritch Invocations                     │
  │   ┌────────────────────────────────────┐   │
  │   │ No Prerequisites                   │   │
  │   │ ├─ Armor of Shadows               │   │
  │   │ ├─ Beast Speech                   │   │
  │   │ └─ Devil's Sight                  │   │
  │   │                                    │   │
  │   │ Requires Eldritch Blast           │   │
  │   │ ├─ Agonizing Blast                │   │
  │   │ └─ Repelling Blast                │   │
  │   │                                    │   │
  │   │ Requires Pact Boon                 │   │
  │   │ ├─ Improved Pact Weapon (Blade)   │   │
  │   │ └─ Voice of the Chain... (Chain)  │   │
  │   └────────────────────────────────────┘   │
  └─────────────────────────────────────────────┘
```

**Behavior**:
- Shows at levels where options unlock (level 2 for Warlock, level 3 for Four Elements)
- Groups options by prerequisite for easy scanning
- Collapsible to avoid overwhelming with 54+ options
- At higher levels, shows "Unlocks at Level X" for newly available options

**Prerequisite Grouping Logic**:
1. "No Prerequisites" — `level_requirement === null && prerequisite_text === null`
2. "Requires [X]" — Group by `prerequisite_text` value
3. Within groups, sort alphabetically by name

---

### Reference View: Full Alphabetical List

Accordion section after "Class Features" with complete alphabetical listing.

**Component**: Extends existing accordion pattern

**Display**:
```
▼ Eldritch Invocations (54)
  ─────────────────────────────────────────────
  A
  ├─ Agonizing Blast
  │  Prerequisite: Eldritch Blast cantrip
  │  When you cast eldritch blast, add your
  │  Charisma modifier to the damage...
  │  Source: PHB p.110
  │
  ├─ Armor of Shadows
  │  You can cast mage armor on yourself...
  │  Source: PHB p.110
```

**Behavior**:
- Alphabetically sorted
- Shows prerequisite if present
- Shows level requirement if present
- Expandable descriptions (truncate to ~150 chars with "Show more")

---

## Component Architecture

### New Components

| Component | Location | Purpose |
|-----------|----------|---------|
| `ClassOverviewOptionsCard` | `app/components/class/overview/` | Teaser card for Overview |
| `ClassJourneyOptionsSection` | `app/components/class/journey/` | Level-specific options panel |
| `ClassOptionCard` | `app/components/class/` | Single option display |
| `ClassOptionsGroup` | `app/components/class/` | Group of options by prerequisite |

### Modified Files

| File | Changes |
|------|---------|
| `useClassDetail.ts` | Add `optionalFeatures`, `hasOptionalFeatures`, `optionalFeaturesByType` |
| `entities.ts` | Export `OptionalFeatureResource` type |
| `[slug]/index.vue` | Add `ClassOverviewOptionsCard` section |
| `[slug]/journey.vue` | Add options to `TimelineLevel` interface, render `ClassJourneyOptionsSection` |
| `[slug]/reference.vue` | Add accordion section for optional features |

---

## Data Layer Changes

### useClassDetail.ts additions

```typescript
import type { OptionalFeatureResource } from '~/types/api/entities'

// In composable return:

// Raw optional features array
const optionalFeatures = computed<OptionalFeatureResource[]>(
  () => entity.value?.optional_features ?? []
)

// Quick check for conditional rendering
const hasOptionalFeatures = computed(() => optionalFeatures.value.length > 0)

// Grouped by feature type (for multiple option types)
const optionalFeaturesByType = computed(() => {
  const grouped = new Map<string, OptionalFeatureResource[]>()
  for (const feature of optionalFeatures.value) {
    const type = feature.feature_type_label
    if (!grouped.has(type)) grouped.set(type, [])
    grouped.get(type)!.push(feature)
  }
  return grouped
})

// Get options available at a specific level
function getOptionsAvailableAtLevel(level: number): OptionalFeatureResource[] {
  return optionalFeatures.value.filter(f =>
    f.level_requirement === null || f.level_requirement <= level
  )
}

// Get options that unlock AT a specific level (new at this level)
function getOptionsUnlockingAtLevel(level: number): OptionalFeatureResource[] {
  return optionalFeatures.value.filter(f => f.level_requirement === level)
}
```

### entities.ts addition

```typescript
// Re-export for component usage
export type OptionalFeatureResource = components['schemas']['OptionalFeatureResource']
```

---

## Test Strategy

### Unit Tests (Composable)

```typescript
describe('useClassDetail - optional features', () => {
  it('returns empty array when no optional features', async () => {})
  it('returns optional features for Warlock', async () => {})
  it('groups features by type correctly', async () => {})
  it('filters options by level requirement', async () => {})
  it('identifies options unlocking at specific level', async () => {})
})
```

### Component Tests

```typescript
describe('ClassOverviewOptionsCard', () => {
  it('renders nothing when no optional features', async () => {})
  it('displays feature type label and count', async () => {})
  it('links to journey view', async () => {})
  it('handles multiple feature types', async () => {})
})

describe('ClassJourneyOptionsSection', () => {
  it('renders nothing when no options at level', async () => {})
  it('groups options by prerequisite', async () => {})
  it('shows "No Prerequisites" group first', async () => {})
  it('is collapsible', async () => {})
})

describe('ClassOptionCard', () => {
  it('displays option name', async () => {})
  it('shows prerequisite when present', async () => {})
  it('shows level requirement when present', async () => {})
  it('truncates long descriptions', async () => {})
  it('expands description on click', async () => {})
})
```

---

## Implementation Order

1. **Data Layer** — Type export + composable updates
2. **ClassOptionCard** — Reusable single option display
3. **ClassOptionsGroup** — Group by prerequisite
4. **ClassOverviewOptionsCard** — Overview teaser
5. **ClassJourneyOptionsSection** — Journey integration
6. **Reference accordion** — Full list in Reference view
7. **Integration tests** — End-to-end verification

---

## Success Criteria

- [ ] Warlock shows 54 Eldritch Invocations
- [ ] Four Elements Monk shows 17 Elemental Disciplines
- [ ] Artificer shows 16 Infusions
- [ ] Options grouped by prerequisite in Journey view
- [ ] Alphabetical listing in Reference view
- [ ] All tests pass
- [ ] TypeScript compiles without errors
- [ ] Works for both base classes and subclasses
