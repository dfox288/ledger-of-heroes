# Background Detail Page Redesign

**Issue:** #51
**Date:** 2025-11-30
**Status:** Design Approved

## Overview

Redesign the background detail page to better showcase all available API data, particularly the **Feature** (signature ability) and **Suggested Characteristics** (rollable tables) that are currently hidden or ignored.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Page structure | Single page (Quick Reference) | Backgrounds are simpler than classes; don't need multi-view |
| Characteristics tables | Static display only | Reference-focused, no interactive dice rolling |
| Feature prominence | Hero section after quick stats | This is what makes each background unique |
| Table layout | 2x2 grid (Personality, Ideal, Bond, Flaw) | Visual balance, easy scanning |

## API Data Audit

The `/backgrounds/{slug}` endpoint provides rich data that's underutilized:

### Currently Displayed
- `name`, `slug` - Header
- `proficiencies` - Skill names (but not ability scores)
- `languages` - Badge list
- `equipment` - Equipment list
- `sources` - Source display
- `traits` - Generic accordion (loses structure)

### Currently Hidden/Ignored
- **`traits[].category: "feature"`** - The signature background ability!
- **`traits[].data_tables`** - Full rollable tables with entries:
  - d8 Personality Traits
  - d6 Ideals (with alignment tags)
  - d6 Bonds
  - d6 Flaws
- **`proficiencies[].skill.ability_score`** - Which ability the skill uses

## Page Layout

```
┌─────────────────────────────────────────────────────────┐
│ Backgrounds > {name}                       (breadcrumb) │
│                                                         │
│ {NAME}                                [Background] {src}│
├─────────────────────────────────────────────────────────┤
│                                                         │
│ ┌──────────────────────────┐  ┌──────────────────────┐ │
│ │ 🎓 SKILLS                │  │ 🗣️ LANGUAGES         │ │
│ │ {skill} ({ability})      │  │ {languages}          │ │
│ │ {skill} ({ability})      │  │                      │ │
│ ├──────────────────────────┤  ├──────────────────────┤ │
│ │ 🔧 TOOLS (if any)        │  │ 💰 STARTING WEALTH   │ │
│ │ {tool proficiencies}     │  │ {gold} gp            │ │
│ └──────────────────────────┘  └──────────────────────┘ │
│                                                         │
├─────────────────────────────────────────────────────────┤
│ ⭐ FEATURE                                              │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│ {Feature Name}                                          │
│                                                         │
│ {Feature description - full text}                       │
│                                                         │
├─────────────────────────────────────────────────────────┤
│ DESCRIPTION                                             │
│                                                         │
│ {Background description from traits[category=null]}     │
│                                                         │
├─────────────────────────────────────────────────────────┤
│ 🎭 SUGGESTED CHARACTERISTICS                            │
│                                                         │
│ ┌─────────────────────────┐ ┌─────────────────────────┐ │
│ │ d8 PERSONALITY TRAIT    │ │ d6 IDEAL                │ │
│ │ ───────────────────     │ │ ───────────────────     │ │
│ │ 1. {entry}              │ │ 1. {entry} (Alignment)  │ │
│ │ 2. {entry}              │ │ 2. {entry} (Alignment)  │ │
│ │ ...                     │ │ ...                     │ │
│ └─────────────────────────┘ └─────────────────────────┘ │
│                                                         │
│ ┌─────────────────────────┐ ┌─────────────────────────┐ │
│ │ d6 BOND                 │ │ d6 FLAW                 │ │
│ │ ───────────────────     │ │ ───────────────────     │ │
│ │ 1. {entry}              │ │ 1. {entry}              │ │
│ │ ...                     │ │ ...                     │ │
│ └─────────────────────────┘ └─────────────────────────┘ │
│                                                         │
├─────────────────────────────────────────────────────────┤
│ ▸ Starting Equipment (accordion - detailed item list)   │
│ ▸ Source Information (accordion)                        │
│                                                         │
├─────────────────────────────────────────────────────────┤
│ ← Back to Backgrounds                                   │
└─────────────────────────────────────────────────────────┘
```

## Component Structure

### New Components

```
app/components/background/
├── BackgroundQuickStats.vue        # 2x2 grid: Skills, Languages, Tools, Gold
├── BackgroundFeatureCard.vue       # Hero section for the feature
└── BackgroundCharacteristicsGrid.vue  # 2x2 grid of rollable tables
```

### Component Details

#### BackgroundQuickStats.vue
- Props: `proficiencies`, `languages`, `equipment`
- Extracts skill proficiencies with ability score codes
- Shows "X of your choice" for language choices
- Calculates starting gold from equipment

#### BackgroundFeatureCard.vue
- Props: `feature` (single trait object with category="feature")
- Prominent card with star icon
- Full feature description displayed

#### BackgroundCharacteristicsGrid.vue
- Props: `dataTables` (array of table objects)
- 2x2 responsive grid (stacks on mobile)
- Each table shows:
  - Table name with dice type (e.g., "d8 Personality Trait")
  - Numbered entries (1-8 or 1-6)
  - Alignment tags for Ideals

## Data Extraction Logic

```typescript
// In useBackgroundDetail composable or page

// Extract feature trait
const feature = computed(() =>
  entity.value?.traits?.find(t => t.category === 'feature')
)

// Extract description trait
const description = computed(() =>
  entity.value?.traits?.find(t => t.category === null && t.name === 'Description')
)

// Extract characteristics trait (has data_tables)
const characteristics = computed(() =>
  entity.value?.traits?.find(t => t.category === 'characteristics')
)

// Get the 4 rollable tables
const dataTables = computed(() =>
  characteristics.value?.data_tables ?? []
)

// Skills with ability scores
const skillsWithAbilities = computed(() =>
  entity.value?.proficiencies
    ?.filter(p => p.proficiency_type === 'skill' && p.skill)
    .map(p => ({
      name: p.skill.name,
      abilityCode: p.skill.ability_score?.code
    })) ?? []
)
```

## Responsive Behavior

| Breakpoint | Quick Stats | Characteristics |
|------------|-------------|-----------------|
| Mobile (<640px) | 1 column stack | 1 column stack |
| Tablet (640-1024px) | 2x2 grid | 2x2 grid |
| Desktop (>1024px) | 2x2 grid | 2x2 grid |

## Test Plan

1. **BackgroundQuickStats tests**
   - Displays skills with ability score codes
   - Handles language choices ("2 of your choice")
   - Shows tool proficiencies when present
   - Calculates gold correctly

2. **BackgroundFeatureCard tests**
   - Renders feature name and description
   - Handles missing feature gracefully

3. **BackgroundCharacteristicsGrid tests**
   - Renders all 4 table types
   - Shows dice type correctly
   - Displays all entries with roll numbers
   - Handles Ideal alignment tags

4. **Page integration tests**
   - Extracts feature from traits correctly
   - Extracts characteristics from traits correctly
   - All sections render in correct order

## Implementation Order

1. Create `useBackgroundDetail` composable (data extraction)
2. Create `BackgroundQuickStats` component (TDD)
3. Create `BackgroundFeatureCard` component (TDD)
4. Create `BackgroundCharacteristicsGrid` component (TDD)
5. Refactor `pages/backgrounds/[slug].vue` to use new components
6. Update test mock data with full background structure
7. Browser verification (light/dark mode, mobile)
