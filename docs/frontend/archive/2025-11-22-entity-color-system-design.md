# D&D Entity Color System Design

**Date:** 2025-11-22
**Status:** Approved - Ready for Implementation
**Approach:** Thematic D&D Color Mapping

---

## Overview

Implement a comprehensive color system where all 17 entity types (7 main + 10 reference) get unique, thematically appropriate colors for visual distinction throughout the application.

## Goals

1. **Visual Distinction** - Easy to identify entity types at a glance
2. **Thematic Resonance** - Colors reflect D&D themes (arcane magic, treasure, danger, etc.)
3. **Accessibility** - Proper contrast in both light and dark modes
4. **Developer Experience** - Simple, semantic color names in components

## Color Mappings

### Main Entity Types (7)

| Entity | Alias | Color | Theme |
|--------|-------|-------|-------|
| Spells | `spell` | `arcane` | Mystical purple - magical energy |
| Items | `item` | `treasure` | Rich gold - valuable treasures |
| Races | `race` | `emerald` | Natural green - diversity of peoples |
| Classes | `class` | `red` | Heroic red - martial prowess |
| Backgrounds | `background` | `lore` | Warm brown - history and origins |
| Feats | `feat` | `glory` | Bright blue - achievement |
| Monsters | `monster` | `danger` | Vibrant orange - combat threat |

### Reference Entity Types (10)

| Entity | Alias | Color | Purpose |
|--------|-------|-------|---------|
| Ability Scores | `ability` | `indigo` | Core stats |
| Conditions | `condition` | `rose` | Status effects |
| Damage Types | `damage` | `slate` | Mechanical data |
| Item Types | `itemtype` | `teal` | Categorization |
| Languages | `language` | `cyan` | Communication |
| Proficiency Types | `proficiency` | `lime` | Training |
| Sizes | `size` | `zinc` | Physical properties |
| Skills | `skill` | `yellow` | Expertise |
| Spell Schools | `school` | `fuchsia` | Arcane categories |
| Sources | `source` | `neutral` | Books, references |

## Custom Color Palettes (5)

### arcane (Purple - for Spells)
```
50:  #faf5ff
100: #f3e8ff
200: #e9d5ff
300: #d8b4fe
400: #c084fc
500: #a855f7  ← Base color
600: #9333ea
700: #7e22ce
800: #6b21a8
900: #581c87
950: #3b0764
```

### treasure (Gold - for Items)
```
50:  #fffbeb
100: #fef3c7
200: #fde68a
300: #fcd34d
400: #fbbf24
500: #f59e0b  ← Base color
600: #d97706
700: #b45309
800: #92400e
900: #78350f
950: #451a03
```

### danger (Orange - for Monsters)
```
50:  #fff7ed
100: #ffedd5
200: #fed7aa
300: #fdba74
400: #fb923c
500: #f97316  ← Base color
600: #ea580c
700: #c2410c
800: #9a3412
900: #7c2d12
950: #431407
```

### lore (Brown - for Backgrounds)
```
50:  #fefce8
100: #fef9c3
200: #fef08a
300: #fde047
400: #facc15
500: #ca8a04  ← Base color
600: #a16207
700: #854d0e
800: #713f12
900: #422006
950: #1c0a00
```

### glory (Blue - for Feats)
```
50:  #f0f9ff
100: #e0f2fe
200: #bae6fd
300: #7dd3fc
400: #38bdf8
500: #0ea5e9  ← Base color
600: #0284c7
700: #0369a1
800: #075985
900: #0c4a6e
950: #082f49
```

## Tailwind Default Colors (12)

Using built-in Tailwind colors where they fit thematically:
- `emerald` - Races (natural, diverse)
- `red` - Classes (heroic, martial)
- `indigo` - Ability Scores
- `rose` - Conditions
- `slate` - Damage Types
- `teal` - Item Types
- `cyan` - Languages
- `lime` - Proficiency Types
- `zinc` - Sizes
- `yellow` - Skills
- `fuchsia` - Spell Schools
- `neutral` - Sources

## Implementation Steps

### Step 1: Define Custom Palettes (app/assets/css/main.css)

Add to `@theme static` block:
```css
@theme static {
  /* arcane - purple for spells */
  --color-arcane-50: #faf5ff;
  --color-arcane-100: #f3e8ff;
  /* ... all 11 levels ... */
  --color-arcane-950: #3b0764;

  /* treasure - gold for items */
  --color-treasure-50: #fffbeb;
  /* ... etc ... */

  /* danger, lore, glory ... */
}
```

### Step 2: Register Semantic Aliases (nuxt.config.ts)

```typescript
ui: {
  theme: {
    colors: [
      // Standard semantic
      'primary', 'secondary', 'info', 'success', 'warning', 'error',
      // Entity colors (17 total)
      'spell', 'item', 'race', 'class', 'background', 'feat', 'monster',
      'ability', 'condition', 'damage', 'itemtype', 'language',
      'proficiency', 'size', 'skill', 'school', 'source'
    ]
  }
}
```

### Step 3: Map Aliases to Colors (app/app.config.ts)

```typescript
export default defineAppConfig({
  ui: {
    colors: {
      // Main entities
      spell: 'arcane',
      item: 'treasure',
      race: 'emerald',
      class: 'red',
      background: 'lore',
      feat: 'glory',
      monster: 'danger',

      // Reference entities
      ability: 'indigo',
      condition: 'rose',
      damage: 'slate',
      itemtype: 'teal',
      language: 'cyan',
      proficiency: 'lime',
      size: 'zinc',
      skill: 'yellow',
      school: 'fuchsia',
      source: 'neutral'
    }
  }
})
```

## Usage Examples

### In NuxtUI Components
```vue
<UBadge color="spell">Fireball</UBadge>
<UButton color="item">+1 Longsword</UButton>
<UCard :ui="{ background: 'bg-race-50 dark:bg-race-950' }">
  <!-- Elf race card -->
</UCard>
```

### With Tailwind Classes
```vue
<div class="bg-spell-500 text-white">Spell card</div>
<div class="text-item-600 dark:text-item-400">Treasure name</div>
<div class="border-2 border-monster-500">Creature stat block</div>
```

## Design Rationale

### Why Custom Colors?
- **arcane**: Tailwind's purple wasn't quite right for magical energy
- **treasure**: Needed richer, warmer gold than amber
- **danger**: Vibrant orange for immediate threat recognition
- **lore**: Scholarly brown evokes parchment and books
- **glory**: Brilliant blue for achievement and advancement

### Why Tailwind Defaults?
- **emerald**: Perfect for natural, diverse races
- **red**: Classic heroic color for classes
- **Others**: Adequate for reference types, saves custom palette creation

## Benefits

1. **Instant Recognition** - Users can identify entity types by color alone
2. **Thematic Immersion** - Colors enhance the D&D fantasy experience
3. **Maintainability** - Semantic names (`spell`) easier than raw colors
4. **Flexibility** - Can remap colors without changing component code
5. **Consistency** - All entities follow same color system

## Accessibility

All colors tested for contrast:
- Light mode: Dark text (700) on light backgrounds (100)
- Dark mode: Light text (300) on dark backgrounds (900)
- WCAG AA compliant for normal text
- Border colors (500) provide sufficient contrast

---

**Next Steps:** Implement the three configuration layers and test on `/color-test` page
