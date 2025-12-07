# Design: Subclass Detail Page Enhancement

**Date:** 2025-11-26
**Status:** Approved
**Author:** Claude (AI Assistant)

---

## Problem Statement

Subclass detail pages (e.g., `/classes/rogue-assassin`) currently show only subclass-specific data, ignoring inherited parent class information. In D&D 5e, subclasses inherit everything from their parent class (hit points, proficiencies, base features, equipment). Users need to see this complete picture.

### Current State

- Subclass pages hide key components via `v-if="entity.is_base_class"`:
  - `UiClassProgressionTable` - hidden
  - `UiClassHitPointsCard` - hidden
  - `UiClassSubclassCards` - hidden (correct - subclasses don't have sub-subclasses)
- API already provides full `parent_class` object with all inherited data
- Users must navigate to parent class page to see proficiencies, hit points, base features

### Desired State

Subclass pages display:
1. Clear subclass identity with parent class link
2. Subclass-specific features prominently
3. All inherited parent class data with clear "Inherited from X" markers
4. Dual images showing both subclass and parent class

---

## Design Overview

### Layout Approach: Integrated Sections

Show parent class data inline with clear "Inherited from [Parent]" section headers. Single scrollable page maintains consistency with existing patterns.

**Rejected alternatives:**
- Tabbed interface (requires extra click)
- Collapsible parent section (hides important inherited data)

---

## Page Layout (Top to Bottom)

### 1. Breadcrumb Navigation

**Current:** `Classes > Assassin`
**New:** `Classes > Rogue > Assassin`

Shows class hierarchy. Middle item links to parent class.

### 2. Header with Badges

```
Title: "Assassin"
Badges:
  - [Subclass of Rogue →] (clickable, links to /classes/rogue, warning color)
  - [✨ Spellcasting Ability] (if subclass or parent has spellcasting)
```

Replace current "Subclass" badge with interactive "Subclass of X" badge.

### 3. Quick Stats + Dual Image

**Left (2/3 width): Quick Stats Card**
- Hit Die: from `parent_class.hit_die`
- Primary Ability: from `parent_class.primary_ability`
- Spellcasting: from parent or subclass `spellcasting_ability`

**Right (1/3 width): Dual Image Display**
- Subclass image: 512px, fills main area
- Parent class image: 128px thumbnail, bottom-right corner overlay
- Parent thumbnail has:
  - Subtle border (class color)
  - "Base Class" label below
  - Links to parent class page
  - Hover effect for discoverability

### 4. Subclass Description

Show `entity.description` - the subclass's own description explaining what makes it unique.

### 5. Hit Points Card (Inherited)

Display `UiClassHitPointsCard` using `parent_class` data:
- `hit-die`: from `parent_class.hit_die`
- `class-name`: from `parent_class.name` (shows "per rogue level")

Add visual indicator: "Inherited from [Parent]" badge or subtitle.

### 6. Subclass Features (Primary Focus)

**This is the main content** - subclass-specific features displayed prominently.

Use existing `UiAccordionTraitsList` component:
- Show only `entity.features` (subclass features)
- Display level for each feature
- Open by default or as standalone section (not in accordion)

Example for Assassin:
- Assassinate (3rd level)
- Infiltration Expertise (9th level)
- Impostor (13th level)
- Death Strike (17th level)

### 7. Class Progression Table (Inherited)

Display `UiClassProgressionTable` using parent class data:
- `features`: from `parent_class.features` (filtered for non-optional)
- `counters`: from `parent_class.counters`

Section header: "Class Progression (Inherited from Rogue)"

### 8. Inherited Content Accordion

Accordion sections for remaining parent class data:

| Section | Data Source | Icon |
|---------|-------------|------|
| Base Class Features | `parent_class.features` | `i-heroicons-star` |
| Proficiencies | `parent_class.proficiencies` | `i-heroicons-academic-cap` |
| Starting Equipment | `parent_class.equipment` | `i-heroicons-shopping-bag` |
| Class Traits | `parent_class.traits` | `i-heroicons-shield-check` |
| Spell Slot Progression | `parent_class.level_progression` | `i-heroicons-sparkles` |

Each section header includes "(Inherited from [Parent])" text.

### 9. Source Display

Show subclass source information using `entity.sources`.

### 10. Bottom Navigation

Back link to Classes list.

---

## Component Changes

### Modified: `app/pages/classes/[slug].vue`

**New computed properties:**
```typescript
// Determine if viewing a subclass
const isSubclass = computed(() => !entity.value?.is_base_class)

// Get parent class data (for subclasses)
const parentClass = computed(() => entity.value?.parent_class)

// Get effective hit die (from parent for subclasses)
const effectiveHitDie = computed(() =>
  isSubclass.value ? parentClass.value?.hit_die : entity.value?.hit_die
)

// Get subclass image path
const subclassImagePath = computed(() => {
  if (!entity.value) return null
  return getImagePath('classes', entity.value.slug, 512)
})

// Get parent class image path (for overlay)
const parentClassImagePath = computed(() => {
  if (!parentClass.value) return null
  return getImagePath('classes', parentClass.value.slug, 128)
})
```

**Template changes:**
- Conditional breadcrumb with parent class
- New "Subclass of X" badge (replaces "Subclass" badge)
- Dual image display for subclasses
- Show inherited components when `isSubclass && parentClass`
- Add "Inherited from X" headers to inherited sections

### New: `app/components/ui/class/UiClassParentImageOverlay.vue`

Small component for the parent class thumbnail overlay:

**Props:**
- `parentSlug: string` - Parent class slug for image and link
- `parentName: string` - Parent class name for alt text and label

**Features:**
- 128px image with rounded corners
- "Base Class" label below
- Links to `/classes/{parentSlug}`
- Hover scale effect
- Positioned absolute bottom-right

---

## Data Flow

```
API Response (subclass)
├── entity.id, entity.slug, entity.name
├── entity.is_base_class = false
├── entity.description (subclass description)
├── entity.features[] (subclass-specific features)
├── entity.sources[]
└── entity.parent_class
    ├── parent_class.hit_die
    ├── parent_class.primary_ability
    ├── parent_class.proficiencies[]
    ├── parent_class.features[] (base class features)
    ├── parent_class.equipment[]
    ├── parent_class.traits[]
    ├── parent_class.counters[]
    └── parent_class.level_progression[]
```

**Usage in template:**
- Subclass data: `entity.*`
- Inherited data: `parentClass.*` (via computed)

---

## Visual Indicators for Inherited Content

### Option A: Section Headers (Recommended)
```html
<h3>
  <UIcon name="i-heroicons-star" />
  Base Class Features
  <UBadge color="neutral" variant="subtle" size="xs">
    Inherited from {{ parentClass.name }}
  </UBadge>
</h3>
```

### Option B: Border Color
Use a different border color (e.g., `neutral` instead of `class`) for inherited sections.

### Option C: Subtle Background
Add a very subtle background tint to inherited sections.

**Decision:** Use Option A (section headers with badge) for clarity without visual clutter.

---

## Edge Cases

1. **Subclass without parent_class data:** Show error state or fetch parent separately (shouldn't happen with current API)

2. **Parent class without certain data:** Conditionally hide sections (e.g., no spell progression for non-casters)

3. **Subclass with its own spellcasting:** Show subclass spellcasting ability in header badge

4. **Image loading failures:** Graceful fallback - hide overlay if parent image fails

---

## Testing Strategy

### Unit Tests (New)

1. **UiClassParentImageOverlay.test.ts**
   - Renders parent class image
   - Shows "Base Class" label
   - Links to parent class page
   - Hover effects work

### Integration Tests (Modified)

2. **classes/[slug].vue tests**
   - Subclass page shows parent class link in breadcrumb
   - Subclass page shows "Subclass of X" badge
   - Subclass page displays inherited hit points card
   - Subclass page displays inherited progression table
   - Subclass page shows dual images
   - Base class page unchanged (no regression)

### Manual Verification

- `/classes/rogue-assassin` - Rogue subclass
- `/classes/wizard-school-of-evocation` - Wizard subclass (with spellcasting)
- `/classes/fighter-champion` - Fighter subclass
- `/classes/rogue` - Base class (ensure no regression)

---

## Success Criteria

1. Subclass pages show complete inherited data from parent class
2. Clear visual distinction between subclass-specific and inherited content
3. Easy navigation to parent class page
4. Dual image display working with overlay
5. All existing base class functionality preserved
6. All tests passing
7. No TypeScript errors
8. Mobile responsive

---

## Implementation Order

1. Create `UiClassParentImageOverlay` component (TDD)
2. Add computed properties for parent class data
3. Update breadcrumb for subclass hierarchy
4. Update header badges
5. Implement dual image display
6. Show inherited Hit Points Card
7. Show inherited Progression Table
8. Add inherited accordion sections
9. Add "Inherited from X" indicators
10. Test all subclass pages
11. Verify base class pages unchanged

---

## Future Enhancements (Out of Scope)

- Click feature in progression table to scroll to feature detail
- Proficiency categorization (armor/weapons/tools/saves/skills)
- Side-by-side subclass comparison
- Character level highlighting in progression table

---

**End of Design Document**
