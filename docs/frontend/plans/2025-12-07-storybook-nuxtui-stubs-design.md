# Storybook NuxtUI Stubs Design

**Date:** 2025-12-07
**Status:** Approved
**Issue:** [#13](https://github.com/dfox288/dnd-rulebook-project/issues/13)

---

## Problem Statement

Storybook runs outside the Nuxt build context, so NuxtUI 4.x component styles don't load. Components render but appear unstyled. The previous attempt used inline stub components in `preview.ts`, but these were incomplete and didn't support all props/variants.

## Goal

Document **custom D&D components** (SpellCard, MonsterCard, wizard steps, etc.) in Storybook. NuxtUI primitives (UCard, UBadge) can use well-designed stubs since they're just containers for the custom content.

## Approach: Enhanced Stubs

Create a comprehensive NuxtUI stub library that:
- Matches NuxtUI appearance well enough for documentation
- Supports the D&D color palette (17 custom colors)
- Works in light and dark mode
- Has proper slot support

---

## NuxtUI Component Inventory

| Tier | Component | Usage Count | Priority |
|------|-----------|-------------|----------|
| Critical | UBadge | 185 | Must have full color/size/variant support |
| Critical | UIcon | 173 | Icon name display or placeholder |
| Critical | UCard | 74 | Full slot support (header, footer, default) |
| Critical | UButton | 70 | Colors, icons, loading states |
| Important | UInput | 18 | Basic input styling |
| Important | UAlert | 15 | Alert box with colors |
| Important | UAccordion | 13 | Expandable sections |
| Important | USelectMenu | 9 | Dropdown select |
| Secondary | UModal | 6 | Modal dialog |
| Secondary | UDropdownMenu | 5 | Dropdown menu |
| Secondary | USkeleton | 4 | Loading placeholder |
| Secondary | UFormField | 2 | Form wrapper |
| Secondary | UCheckbox | 2 | Checkbox input |
| Secondary | UButtonGroup | 2 | Button grouping |
| Rare | UTabs | 1 | Tab navigation |
| Rare | UPagination | 1 | Page navigation |
| Rare | UToggle | 1 | Toggle switch |
| Rare | USelect | 1 | Select input |

**Also needed:** `NuxtLink` (router link stub)

---

## Directory Structure

```
.storybook/
├── main.ts                 # Existing - no changes
├── preview.ts              # Update to import stubs
├── preview.css             # Existing - no changes
└── nuxtui-stubs/
    ├── index.ts            # Auto-registers all stubs globally
    ├── colors.ts           # Color mapping utility
    │
    ├── # Tier 1 - Critical
    ├── UCard.vue
    ├── UBadge.vue
    ├── UButton.vue
    ├── UIcon.vue
    │
    ├── # Tier 2 - Important
    ├── UInput.vue
    ├── UAlert.vue
    ├── UAccordion.vue
    ├── USelectMenu.vue
    │
    ├── # Tier 3 - Secondary
    ├── UModal.vue
    ├── UDropdownMenu.vue
    ├── USkeleton.vue
    ├── UFormField.vue
    ├── UCheckbox.vue
    ├── UButtonGroup.vue
    │
    ├── # Tier 4 - Rare
    ├── UTabs.vue
    ├── UPagination.vue
    ├── UToggle.vue
    ├── USelect.vue
    │
    └── NuxtLink.vue        # Router link stub
```

---

## Color System

### Color Mapping (colors.ts)

```typescript
// Maps NuxtUI color props to Tailwind color classes
export const colorMap: Record<string, string> = {
  // Semantic colors
  primary: 'rose',
  secondary: 'gray',
  info: 'blue',
  success: 'green',
  warning: 'yellow',
  error: 'red',
  neutral: 'gray',

  // D&D Entity colors (from app.config.ts → theme.css)
  spell: 'arcane',
  item: 'treasure',
  monster: 'danger',
  race: 'emerald',
  class: 'red',
  background: 'lore',
  feat: 'glory',

  // Reference entity colors
  ability: 'indigo',
  condition: 'pink',
  damage: 'slate',
  itemtype: 'teal',
  language: 'cyan',
  proficiency: 'lime',
  size: 'zinc',
  skill: 'yellow',
  school: 'fuchsia',
  source: 'neutral'
}

export function getColorClasses(color: string, variant: string = 'subtle') {
  const mapped = colorMap[color] || color

  switch (variant) {
    case 'solid':
      return `bg-${mapped}-600 text-white dark:bg-${mapped}-500`
    case 'outline':
      return `border border-${mapped}-500 text-${mapped}-600 dark:text-${mapped}-400`
    case 'subtle':
    default:
      return `bg-${mapped}-100 text-${mapped}-700 dark:bg-${mapped}-900/50 dark:text-${mapped}-300`
  }
}
```

### Why This Works

The `theme.css` file defines custom color palettes using `@theme static`:
- `--color-arcane-50` through `--color-arcane-950` (spell purple)
- `--color-treasure-50` through `--color-treasure-950` (item gold)
- etc.

Tailwind v4 recognizes these and generates utility classes like `bg-arcane-100`, `text-treasure-700`.

---

## Component Stub Specifications

### UCard

**Props:** `class`
**Slots:** `default`, `header`, `footer`

```vue
<template>
  <div
    class="bg-white dark:bg-gray-900 rounded-xl border border-gray-200 dark:border-gray-800 shadow-sm"
    :class="$attrs.class"
  >
    <div v-if="$slots.header" class="px-6 py-4 border-b border-gray-200 dark:border-gray-800">
      <slot name="header" />
    </div>
    <div class="p-6">
      <slot />
    </div>
    <div v-if="$slots.footer" class="px-6 py-4 border-t border-gray-200 dark:border-gray-800">
      <slot name="footer" />
    </div>
  </div>
</template>
```

### UBadge

**Props:** `color`, `variant`, `size`
**Slots:** `default`

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { getColorClasses } from './colors'

const props = withDefaults(defineProps<{
  color?: string
  variant?: 'solid' | 'subtle' | 'outline'
  size?: 'xs' | 'sm' | 'md' | 'lg'
}>(), {
  color: 'primary',
  variant: 'subtle',
  size: 'md'
})

const sizeClasses = computed(() => ({
  xs: 'text-xs px-1.5 py-0.5',
  sm: 'text-xs px-2 py-0.5',
  md: 'text-sm px-2.5 py-1',
  lg: 'text-base px-3 py-1.5'
}[props.size]))

const colorClasses = computed(() => getColorClasses(props.color, props.variant))
</script>

<template>
  <span
    class="inline-flex items-center font-medium rounded-md"
    :class="[sizeClasses, colorClasses]"
  >
    <slot />
  </span>
</template>
```

### UButton

**Props:** `color`, `variant`, `size`, `icon`, `trailing-icon`, `loading`, `disabled`
**Slots:** `default`, `leading`, `trailing`

### UIcon

**Props:** `name`

Display icon name as text or use a simple placeholder SVG.

```vue
<template>
  <span class="inline-flex items-center justify-center" :class="$attrs.class">
    <!-- Simple placeholder or icon name -->
    <svg class="w-full h-full" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
      <circle cx="12" cy="12" r="10" />
    </svg>
  </span>
</template>
```

### UAccordion

**Props:** `items`, `default-open`

Simple disclosure component with expand/collapse.

### Other Components

Lower-tier components can use minimal implementations that render basic structure without full interactivity.

---

## Preview.ts Updates

```typescript
// .storybook/preview.ts
import type { Preview } from '@storybook/vue3'
import { setup } from '@storybook/vue3'

import './preview.css'

// Import all NuxtUI stubs
import { registerNuxtUIStubs } from './nuxtui-stubs'

setup((app) => {
  registerNuxtUIStubs(app)
})

const preview: Preview = {
  parameters: {
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/i
      }
    },
    backgrounds: {
      default: 'dark',
      values: [
        { name: 'light', value: '#faf5ff' },  // purple-50 from theme
        { name: 'dark', value: '#0c0a09' }    // dark from theme
      ]
    }
  },
  decorators: [
    (story) => ({
      components: { story },
      template: '<div class="p-8 min-h-screen"><story /></div>'
    })
  ]
}

export default preview
```

---

## Implementation Phases

### Phase 1: Foundation (30 min)
- [ ] Create `.storybook/nuxtui-stubs/` directory
- [ ] Create `colors.ts` with color mapping
- [ ] Create `index.ts` with registration function

### Phase 2: Critical Stubs (2 hours)
- [ ] UCard.vue - full slot support
- [ ] UBadge.vue - all colors, variants, sizes
- [ ] UButton.vue - colors, icons, states
- [ ] UIcon.vue - placeholder or icon display

### Phase 3: Important Stubs (2 hours)
- [ ] UInput.vue
- [ ] UAlert.vue
- [ ] UAccordion.vue
- [ ] USelectMenu.vue

### Phase 4: Secondary Stubs (1 hour)
- [ ] UModal.vue
- [ ] UDropdownMenu.vue
- [ ] USkeleton.vue
- [ ] UFormField.vue
- [ ] UCheckbox.vue
- [ ] UButtonGroup.vue

### Phase 5: Rare Stubs (30 min)
- [ ] UTabs.vue
- [ ] UPagination.vue
- [ ] UToggle.vue
- [ ] USelect.vue
- [ ] NuxtLink.vue

### Phase 6: Integration (30 min)
- [ ] Update preview.ts
- [ ] Remove old inline stubs
- [ ] Test existing stories

### Phase 7: New Stories (1 hour)
- [ ] SpellCard.stories.ts
- [ ] MonsterCard.stories.ts
- [ ] ClassCard.stories.ts

### Phase 8: Verification (30 min)
- [ ] Run Storybook: `npm run storybook`
- [ ] Compare key components against real app
- [ ] Fix any styling issues

---

## Success Criteria

1. Existing 5 stories render correctly with new stubs
2. SpellCard story shows proper D&D purple badges
3. MonsterCard story shows icons and orange badges
4. Light/dark mode toggle works
5. No console errors in Storybook

---

## Future Considerations

- **Icon library:** Consider integrating a subset of Heroicons for better icon display
- **Storybook 9:** When @nuxtjs/storybook supports Nuxt 4, could migrate to that
- **Visual regression:** Could add Chromatic for automated visual testing

---

## References

- [Storybook Styling Docs](https://storybook.js.org/docs/configure/styling-and-css)
- [Nuxt Storybook Module](https://storybook.nuxtjs.org/)
- [NuxtUI Components](https://ui.nuxt.com/docs/getting-started/theme/components)
- [GitHub Issue #18670](https://github.com/storybookjs/storybook/issues/18670) - Vite CSS variables
