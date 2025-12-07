# Design: Class Detail Page Enhancement

**Date:** 2025-11-26
**Status:** Approved for Implementation
**Scope:** Moderate Enhancement

---

## Overview

Enhance the Classes detail page (`app/pages/classes/[slug].vue`) to be more visually appealing and game-useful. The centerpiece is a new **Class Progression Table** showing what features and resources you gain at each level.

---

## Goals

1. **Add Class Progression Table** - The most requested feature for class pages
2. **Improve visual hierarchy** - Icons on accordion headers, better section organization
3. **Game-ready information** - Hit Points card, categorized proficiencies
4. **Better subclass navigation** - Card grid instead of nested accordions

---

## Non-Goals

- Full page redesign with tabs (future consideration)
- Backend API changes (noted for future)
- Subclass detail page improvements (separate effort)

---

## Design

### New Page Structure (Top to Bottom)

1. **Breadcrumb** - Existing, no change
2. **Header** - Existing with minor badge tweaks
3. **Quick Stats + Image** - Existing 2/3 + 1/3 grid layout
4. **NEW: Class Progression Table** - Prominent, always visible
5. **Description Card** - Flavor text from `entity.description`
6. **NEW: Hit Points Card** - HP calculation display
7. **Enhanced Accordion** with icons:
   - Proficiencies (categorized display)
   - Starting Equipment
   - Features (with level badges)
   - Subclasses (card grid)
   - Class Counters (if not shown in progression table)
   - Spell Slot Progression (for casters)
   - Source

### Removed/Changed

- "Additional Class Traits" merged into Features
- Tags section removed (low value)
- First trait no longer used as description

---

## New Components

### 1. `UiClassProgressionTable.vue`

**Purpose:** Display level-by-level progression of features and counters.

**Location:** `app/components/ui/class/UiClassProgressionTable.vue`

**Props:**
```typescript
interface Props {
  features: ClassFeature[]
  counters: ClassCounter[]
  maxLevel?: number // Default: 20
}
```

**Table Structure:**
| Level | Prof. Bonus | Features | [Counter Columns...] |
|-------|-------------|----------|----------------------|
| 1 | +2 | Expertise, Sneak Attack, Thieves' Cant | 1d6 |
| 2 | +2 | Cunning Action | 1d6 |
| 3 | +2 | Roguish Archetype | 2d6 |
| ... | ... | ... | ... |

**Implementation Notes:**
- Proficiency bonus calculated: `Math.ceil(level / 4) + 1`
- Counter columns added dynamically based on counter types present
- Features grouped by level, displayed as comma-separated list
- Use `UTable` with `sticky` header for scrolling
- Mobile: Simplified view (Level + Features only, hide Prof. Bonus and counters)
- Entity color accent on header row

**Data Transformation:**
```typescript
// Group features by level
const featuresByLevel = computed(() => {
  const grouped = new Map<number, string[]>()
  for (const feature of props.features) {
    const names = grouped.get(feature.level) || []
    names.push(feature.feature_name)
    grouped.set(feature.level, names)
  }
  return grouped
})

// Get counter value at specific level
const getCounterAtLevel = (counterName: string, level: number) => {
  // Find the highest counter entry at or below this level
  const entries = props.counters
    .filter(c => c.counter_name === counterName && c.level <= level)
    .sort((a, b) => b.level - a.level)
  return entries[0]?.counter_value || null
}
```

---

### 2. `UiHitPointsCard.vue`

**Purpose:** Display hit point calculation in game-ready format.

**Location:** `app/components/ui/class/UiHitPointsCard.vue`

**Props:**
```typescript
interface Props {
  hitDie: number // e.g., 8, 10, 12
}
```

**Display:**
```
┌─────────────────────────────────────────────────────┐
│ ❤️ Hit Points                                       │
├─────────────────────────────────────────────────────┤
│ Hit Dice        │ 1d8 per rogue level              │
│ HP at 1st Level │ 8 + Constitution modifier        │
│ HP at Higher    │ 1d8 (or 5) + Constitution        │
│ Levels          │ modifier per rogue level after 1 │
└─────────────────────────────────────────────────────┘
```

**Implementation:** Simple UCard with grid layout, uses entity color accent.

---

### 3. `UiProficienciesCard.vue`

**Purpose:** Display proficiencies in categorized format.

**Location:** `app/components/ui/class/UiProficienciesCard.vue`

**Props:**
```typescript
interface Props {
  proficiencies: string[]
}
```

**Categories:**
- Armor
- Weapons
- Tools
- Saving Throws
- Skills

**Implementation Notes:**
- Parse proficiency strings to categorize (regex-based)
- Display as definition list or two-column grid
- Handle "Choose X from..." skill selections

**Future Backend Enhancement:** API could return pre-categorized proficiencies:
```json
{
  "proficiencies": {
    "armor": ["Light armor"],
    "weapons": ["Simple weapons", "hand crossbows", ...],
    "tools": ["Thieves' tools"],
    "saving_throws": ["Dexterity", "Intelligence"],
    "skills": {
      "choose": 4,
      "from": ["Acrobatics", "Athletics", ...]
    }
  }
}
```

---

### 4. `UiSubclassCards.vue`

**Purpose:** Display subclasses as a navigable card grid.

**Location:** `app/components/ui/class/UiSubclassCards.vue`

**Props:**
```typescript
interface Props {
  subclasses: Subclass[]
  basePath: string // e.g., "/classes"
}
```

**Card Display:**
```
┌──────────────────┐
│ Arcane Trickster │
│ ───────────────  │
│ PHB p.97         │
│ 5 features       │
│ [View →]         │
└──────────────────┘
```

**Implementation:**
- 2-column grid on mobile, 3-4 columns on desktop
- Each card links to `/classes/{subclass-slug}`
- Show source abbreviation and feature count
- Hover effect with entity color

---

### 5. Enhanced Accordion Headers

**Approach:** Add icons to accordion items using the `icon` property.

**Icon Mapping:**
| Section | Icon | Color |
|---------|------|-------|
| Hit Points | `i-heroicons-heart` | error |
| Proficiencies | `i-heroicons-academic-cap` | info |
| Equipment | `i-heroicons-briefcase` | warning |
| Features | `i-heroicons-bolt` | class |
| Subclasses | `i-heroicons-users` | class |
| Counters | `i-heroicons-calculator` | success |
| Spell Progression | `i-heroicons-sparkles` | primary |
| Source | `i-heroicons-book-open` | neutral |

**Implementation:**
```typescript
const accordionItems = computed(() => [
  {
    label: `Features (${entity.value.features.length})`,
    icon: 'i-heroicons-bolt',
    slot: 'features'
  },
  // ...
])
```

---

## Page Layout Mockup

```
┌─────────────────────────────────────────────────────────────┐
│ ← Back to Classes                                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ROGUE                                    [Base Class]      │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────┐  ┌────────────────────────┐ │
│ │ Quick Stats                 │  │                        │ │
│ │ ❤️ Hit Die: 1d8             │  │      [Rogue Image]     │ │
│ │ ⭐ Primary: DEX             │  │                        │ │
│ └─────────────────────────────┘  └────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ CLASS PROGRESSION                                           │
│ ┌───────┬───────┬──────────────────────────┬──────────────┐ │
│ │ Level │ Prof. │ Features                 │ Sneak Attack │ │
│ ├───────┼───────┼──────────────────────────┼──────────────┤ │
│ │ 1     │ +2    │ Expertise, Sneak Attack  │ 1d6          │ │
│ │ 2     │ +2    │ Cunning Action           │ 1d6          │ │
│ │ 3     │ +2    │ Roguish Archetype        │ 2d6          │ │
│ │ ...   │ ...   │ ...                      │ ...          │ │
│ └───────┴───────┴──────────────────────────┴──────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ Description Card                                            │
│ "Signaling for her companions to wait, a halfling creeps    │
│ forward through the dungeon hall..."                        │
├─────────────────────────────────────────────────────────────┤
│ ❤️ Hit Points Card                                          │
│ Hit Dice: 1d8 | HP at 1st: 8 + CON | HP/Level: 1d8 (or 5)  │
├─────────────────────────────────────────────────────────────┤
│ ▶ 🎓 Proficiencies (5 categories)                          │
│ ▶ 💼 Starting Equipment                                     │
│ ▶ ⚡ Features (15)                                          │
│ ▶ 👥 Subclasses (4)                                         │
│ ▶ 📖 Source                                                 │
├─────────────────────────────────────────────────────────────┤
│ ← Back to Classes                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## Future Backend Enhancements

> **Note:** The current implementation will do data transformation on the frontend. However, several enhancements would benefit from backend support:

### 1. Pre-categorized Proficiencies
Current API returns flat string array. Backend could return:
```json
{
  "proficiencies_categorized": {
    "armor": ["Light armor"],
    "weapons": ["Simple weapons", "hand crossbows"],
    "tools": ["Thieves' tools"],
    "saving_throws": ["Dexterity", "Intelligence"],
    "skills": { "choose": 4, "from": ["Acrobatics", ...] }
  }
}
```

### 2. Pre-built Progression Table
Backend could return a ready-to-render progression:
```json
{
  "level_progression": [
    {
      "level": 1,
      "proficiency_bonus": 2,
      "features": ["Expertise", "Sneak Attack", "Thieves' Cant"],
      "counters": { "sneak_attack": "1d6" }
    },
    // ...
  ]
}
```

### 3. Counter Display Format
Backend could specify how counters should be displayed:
```json
{
  "counter_name": "Sneak Attack",
  "display_format": "{value}d6", // Template for display
  "progression": [
    { "level": 1, "value": 1 },
    { "level": 3, "value": 2 },
    // ...
  ]
}
```

### 4. Multiclassing Requirements
Backend could include multiclassing prerequisites:
```json
{
  "multiclass_requirements": {
    "ability_minimum": { "dexterity": 13 },
    "proficiencies_gained": ["Light armor", "Thieves' tools"]
  }
}
```

---

## Implementation Order

1. **UiClassProgressionTable** - Core feature, highest value
2. **Enhanced accordion icons** - Quick win, visual improvement
3. **UiHitPointsCard** - Simple, game-useful
4. **UiSubclassCards** - Better navigation
5. **UiProficienciesCard** - Nice to have, more complex parsing

---

## Testing Strategy

- Unit tests for data transformation functions (grouping features by level, counter interpolation)
- Component tests for each new component
- Visual regression testing for responsive behavior
- E2E test for page load and accordion interaction

---

## Success Criteria

- [ ] Class progression table displays correctly for all classes
- [ ] Counters (Sneak Attack, Ki, Rage, etc.) show in progression table
- [ ] Accordion headers have appropriate icons
- [ ] Hit Points card shows correct calculations
- [ ] Subclass cards link to correct detail pages
- [ ] Page remains performant (no layout shift, fast LCP)
- [ ] Dark mode works correctly
- [ ] Mobile responsive

---

## References

- Wikidot Rogue page: https://dnd5e.wikidot.com/rogue
- NuxtUI Accordion: https://ui.nuxt.com/components/accordion
- NuxtUI Table: https://ui.nuxt.com/components/table
- Current implementation: `app/pages/classes/[slug].vue`

---

**End of Design Document**
