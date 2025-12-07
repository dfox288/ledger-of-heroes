# Storybook NuxtUI Stubs Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a comprehensive NuxtUI stub library so custom D&D components render properly in Storybook.

**Architecture:** Vue SFC stubs that mimic NuxtUI component appearance using Tailwind classes. A color mapping utility translates D&D semantic colors (spell, monster, item) to CSS classes that reference the existing theme.css custom color palettes.

**Tech Stack:** Vue 3, TypeScript, Tailwind CSS v4, Storybook 8.6

**Design Doc:** `2025-12-07-storybook-nuxtui-stubs-design.md`

---

## Task 1: Create Color Mapping Utility

**Files:**
- Create: `.storybook/nuxtui-stubs/colors.ts`

**Step 1: Create the colors utility**

```typescript
// .storybook/nuxtui-stubs/colors.ts

/**
 * Maps NuxtUI color props to Tailwind color class prefixes.
 * Custom D&D colors (arcane, treasure, etc.) are defined in app/assets/css/theme.css
 */
export const colorMap: Record<string, string> = {
  // Semantic colors (NuxtUI defaults)
  primary: 'rose',
  secondary: 'gray',
  info: 'blue',
  success: 'green',
  warning: 'amber',
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

  // Reference entity colors (standard Tailwind)
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

/**
 * Get the mapped color name for Tailwind classes
 */
export function getMappedColor(color: string): string {
  return colorMap[color] || color
}

/**
 * Generate Tailwind classes for badge/button colors based on variant
 */
export function getColorClasses(color: string, variant: 'solid' | 'subtle' | 'outline' | 'ghost' = 'subtle'): string {
  const mapped = getMappedColor(color)

  switch (variant) {
    case 'solid':
      return `bg-${mapped}-600 text-white dark:bg-${mapped}-500`
    case 'outline':
      return `border border-${mapped}-500 text-${mapped}-600 dark:border-${mapped}-400 dark:text-${mapped}-400 bg-transparent`
    case 'ghost':
      return `text-${mapped}-600 dark:text-${mapped}-400 bg-transparent hover:bg-${mapped}-100 dark:hover:bg-${mapped}-900/50`
    case 'subtle':
    default:
      return `bg-${mapped}-100 text-${mapped}-700 dark:bg-${mapped}-900/50 dark:text-${mapped}-300`
  }
}

/**
 * Generate Tailwind classes for button hover states
 */
export function getButtonClasses(color: string, variant: 'solid' | 'subtle' | 'outline' | 'ghost' = 'solid'): string {
  const mapped = getMappedColor(color)

  switch (variant) {
    case 'solid':
      return `bg-${mapped}-600 hover:bg-${mapped}-700 text-white dark:bg-${mapped}-500 dark:hover:bg-${mapped}-600`
    case 'outline':
      return `border border-${mapped}-500 text-${mapped}-600 hover:bg-${mapped}-50 dark:border-${mapped}-400 dark:text-${mapped}-400 dark:hover:bg-${mapped}-900/30`
    case 'ghost':
      return `text-${mapped}-600 hover:bg-${mapped}-100 dark:text-${mapped}-400 dark:hover:bg-${mapped}-900/50`
    case 'subtle':
    default:
      return `bg-${mapped}-100 hover:bg-${mapped}-200 text-${mapped}-700 dark:bg-${mapped}-900/50 dark:hover:bg-${mapped}-900/70 dark:text-${mapped}-300`
  }
}
```

**Step 2: Verify file created**

Run: `cat .storybook/nuxtui-stubs/colors.ts | head -20`
Expected: Shows the colorMap export

**Step 3: Commit**

```bash
git add .storybook/nuxtui-stubs/colors.ts
git commit -m "feat(storybook): add color mapping utility for NuxtUI stubs"
```

---

## Task 2: Create UCard Stub

**Files:**
- Create: `.storybook/nuxtui-stubs/UCard.vue`

**Step 1: Create UCard stub with full slot support**

```vue
<!-- .storybook/nuxtui-stubs/UCard.vue -->
<script setup lang="ts">
/**
 * UCard stub for Storybook
 * Supports: default slot, header slot, footer slot
 * Mimics NuxtUI UCard appearance with Tailwind classes
 */
defineOptions({ name: 'UCard' })
</script>

<template>
  <div
    class="bg-white dark:bg-gray-900 rounded-xl border border-gray-200 dark:border-gray-800 shadow-sm overflow-hidden"
    v-bind="$attrs"
  >
    <!-- Header slot -->
    <div
      v-if="$slots.header"
      class="px-6 py-4 border-b border-gray-200 dark:border-gray-800"
    >
      <slot name="header" />
    </div>

    <!-- Default slot (body) -->
    <div class="p-6">
      <slot />
    </div>

    <!-- Footer slot -->
    <div
      v-if="$slots.footer"
      class="px-6 py-4 border-t border-gray-200 dark:border-gray-800 bg-gray-50 dark:bg-gray-800/50"
    >
      <slot name="footer" />
    </div>
  </div>
</template>
```

**Step 2: Commit**

```bash
git add .storybook/nuxtui-stubs/UCard.vue
git commit -m "feat(storybook): add UCard stub with slot support"
```

---

## Task 3: Create UBadge Stub

**Files:**
- Create: `.storybook/nuxtui-stubs/UBadge.vue`

**Step 1: Create UBadge stub with color/size/variant support**

```vue
<!-- .storybook/nuxtui-stubs/UBadge.vue -->
<script setup lang="ts">
import { computed } from 'vue'
import { getColorClasses } from './colors'

/**
 * UBadge stub for Storybook
 * Supports: color, variant, size props
 * Uses color mapping for D&D custom colors (spell, monster, item, etc.)
 */
defineOptions({ name: 'UBadge' })

const props = withDefaults(defineProps<{
  color?: string
  variant?: 'solid' | 'subtle' | 'outline'
  size?: 'xs' | 'sm' | 'md' | 'lg'
}>(), {
  color: 'primary',
  variant: 'subtle',
  size: 'md'
})

const sizeClasses = computed(() => {
  const sizes: Record<string, string> = {
    xs: 'text-xs px-1.5 py-0.5',
    sm: 'text-xs px-2 py-0.5',
    md: 'text-sm px-2.5 py-1',
    lg: 'text-base px-3 py-1.5'
  }
  return sizes[props.size] || sizes.md
})

const colorClasses = computed(() => getColorClasses(props.color, props.variant))
</script>

<template>
  <span
    class="inline-flex items-center font-medium rounded-md whitespace-nowrap"
    :class="[sizeClasses, colorClasses]"
    v-bind="$attrs"
  >
    <slot />
  </span>
</template>
```

**Step 2: Commit**

```bash
git add .storybook/nuxtui-stubs/UBadge.vue
git commit -m "feat(storybook): add UBadge stub with D&D color support"
```

---

## Task 4: Create UButton Stub

**Files:**
- Create: `.storybook/nuxtui-stubs/UButton.vue`

**Step 1: Create UButton stub**

```vue
<!-- .storybook/nuxtui-stubs/UButton.vue -->
<script setup lang="ts">
import { computed } from 'vue'
import { getButtonClasses } from './colors'

/**
 * UButton stub for Storybook
 * Supports: color, variant, size, icon, loading, disabled
 */
defineOptions({ name: 'UButton' })

const props = withDefaults(defineProps<{
  color?: string
  variant?: 'solid' | 'subtle' | 'outline' | 'ghost'
  size?: 'xs' | 'sm' | 'md' | 'lg' | 'xl'
  icon?: string
  trailingIcon?: string
  loading?: boolean
  disabled?: boolean
  to?: string
}>(), {
  color: 'primary',
  variant: 'solid',
  size: 'md'
})

const emit = defineEmits<{
  click: [event: MouseEvent]
}>()

const sizeClasses = computed(() => {
  const sizes: Record<string, string> = {
    xs: 'text-xs px-2 py-1 gap-1',
    sm: 'text-sm px-3 py-1.5 gap-1.5',
    md: 'text-sm px-4 py-2 gap-2',
    lg: 'text-base px-5 py-2.5 gap-2',
    xl: 'text-lg px-6 py-3 gap-2.5'
  }
  return sizes[props.size] || sizes.md
})

const colorClasses = computed(() => getButtonClasses(props.color, props.variant))

function handleClick(event: MouseEvent) {
  if (!props.disabled && !props.loading) {
    emit('click', event)
  }
}
</script>

<template>
  <component
    :is="to ? 'a' : 'button'"
    :href="to"
    :disabled="disabled || loading"
    class="inline-flex items-center justify-center font-semibold rounded-lg transition-all duration-200 focus:outline-none focus:ring-2 focus:ring-offset-2"
    :class="[
      sizeClasses,
      colorClasses,
      {
        'opacity-50 cursor-not-allowed': disabled,
        'cursor-wait': loading
      }
    ]"
    v-bind="$attrs"
    @click="handleClick"
  >
    <!-- Loading spinner -->
    <svg
      v-if="loading"
      class="animate-spin -ml-1 mr-2 h-4 w-4"
      fill="none"
      viewBox="0 0 24 24"
    >
      <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" />
      <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z" />
    </svg>

    <!-- Leading icon placeholder -->
    <span v-else-if="icon" class="w-4 h-4 flex items-center justify-center">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" class="w-full h-full">
        <circle cx="12" cy="12" r="3" />
      </svg>
    </span>

    <!-- Leading slot -->
    <slot name="leading" />

    <!-- Default slot -->
    <slot />

    <!-- Trailing slot -->
    <slot name="trailing" />

    <!-- Trailing icon placeholder -->
    <span v-if="trailingIcon" class="w-4 h-4 flex items-center justify-center">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" class="w-full h-full">
        <path d="M9 5l7 7-7 7" />
      </svg>
    </span>
  </component>
</template>
```

**Step 2: Commit**

```bash
git add .storybook/nuxtui-stubs/UButton.vue
git commit -m "feat(storybook): add UButton stub with loading and icon support"
```

---

## Task 5: Create UIcon Stub

**Files:**
- Create: `.storybook/nuxtui-stubs/UIcon.vue`

**Step 1: Create UIcon stub with common icon shapes**

```vue
<!-- .storybook/nuxtui-stubs/UIcon.vue -->
<script setup lang="ts">
import { computed } from 'vue'

/**
 * UIcon stub for Storybook
 * Shows placeholder SVG icons based on common icon name patterns
 */
defineOptions({ name: 'UIcon', inheritAttrs: false })

const props = defineProps<{
  name?: string
}>()

// Map common icon names to simple SVG paths
const iconPaths = computed(() => {
  const name = props.name?.toLowerCase() || ''

  // Search/magnifying glass
  if (name.includes('search') || name.includes('magnify')) {
    return 'M21 21l-5.2-5.2M17 10a7 7 0 1 1-14 0 7 7 0 0 1 14 0z'
  }
  // Close/X
  if (name.includes('close') || name.includes('x-mark') || name.includes('times')) {
    return 'M6 6l12 12M6 18L18 6'
  }
  // Check/checkmark
  if (name.includes('check')) {
    return 'M5 13l4 4L19 7'
  }
  // Arrow right/chevron right
  if (name.includes('arrow-right') || name.includes('chevron-right')) {
    return 'M9 5l7 7-7 7'
  }
  // Arrow left/chevron left
  if (name.includes('arrow-left') || name.includes('chevron-left')) {
    return 'M15 19l-7-7 7-7'
  }
  // Arrow up/chevron up
  if (name.includes('arrow-up') || name.includes('chevron-up')) {
    return 'M5 15l7-7 7 7'
  }
  // Arrow down/chevron down
  if (name.includes('arrow-down') || name.includes('chevron-down')) {
    return 'M19 9l-7 7-7-7'
  }
  // Plus/add
  if (name.includes('plus') || name.includes('add')) {
    return 'M12 5v14M5 12h14'
  }
  // Minus
  if (name.includes('minus')) {
    return 'M5 12h14'
  }
  // Edit/pencil
  if (name.includes('edit') || name.includes('pencil')) {
    return 'M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7M18.5 2.5a2.12 2.12 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z'
  }
  // Trash/delete
  if (name.includes('trash') || name.includes('delete')) {
    return 'M3 6h18M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2m3 0v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6h14'
  }
  // User/person
  if (name.includes('user') || name.includes('person')) {
    return 'M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2M12 11a4 4 0 1 0 0-8 4 4 0 0 0 0 8z'
  }
  // Heart
  if (name.includes('heart')) {
    return 'M20.8 4.6a5.5 5.5 0 0 0-7.8 0L12 5.7l-1-1.1a5.5 5.5 0 0 0-7.8 7.8l1 1.1L12 21.4l7.8-7.9 1-1.1a5.5 5.5 0 0 0 0-7.8z'
  }
  // Star
  if (name.includes('star')) {
    return 'M12 2l3.1 6.3 6.9 1-5 4.9 1.2 6.8-6.2-3.3-6.2 3.3 1.2-6.8-5-4.9 6.9-1L12 2z'
  }
  // Settings/cog
  if (name.includes('setting') || name.includes('cog') || name.includes('gear')) {
    return 'M12 15a3 3 0 1 0 0-6 3 3 0 0 0 0 6zM19.4 15a1.6 1.6 0 0 0 .3 1.8l.1.1a2 2 0 1 1-2.9 2.9l-.1-.1a1.6 1.6 0 0 0-1.8-.3 1.6 1.6 0 0 0-1 1.5V21a2 2 0 1 1-4 0v-.1a1.6 1.6 0 0 0-1-1.5 1.6 1.6 0 0 0-1.8.3l-.1.1a2 2 0 1 1-2.9-2.9l.1-.1a1.6 1.6 0 0 0 .3-1.8 1.6 1.6 0 0 0-1.5-1H3a2 2 0 1 1 0-4h.1a1.6 1.6 0 0 0 1.5-1 1.6 1.6 0 0 0-.3-1.8l-.1-.1a2 2 0 1 1 2.9-2.9l.1.1a1.6 1.6 0 0 0 1.8.3h.1a1.6 1.6 0 0 0 1-1.5V3a2 2 0 1 1 4 0v.1a1.6 1.6 0 0 0 1 1.5 1.6 1.6 0 0 0 1.8-.3l.1-.1a2 2 0 1 1 2.9 2.9l-.1.1a1.6 1.6 0 0 0-.3 1.8v.1a1.6 1.6 0 0 0 1.5 1H21a2 2 0 1 1 0 4h-.1a1.6 1.6 0 0 0-1.5 1z'
  }
  // Info/information
  if (name.includes('info')) {
    return 'M12 22c5.5 0 10-4.5 10-10S17.5 2 12 2 2 6.5 2 12s4.5 10 10 10zM12 16v-4M12 8h.01'
  }
  // Warning/alert
  if (name.includes('warning') || name.includes('alert') || name.includes('exclamation')) {
    return 'M12 9v4M12 17h.01M10.3 3.2L1.8 18a2 2 0 0 0 1.7 3h17a2 2 0 0 0 1.7-3L13.7 3.2a2 2 0 0 0-3.4 0z'
  }
  // Menu/hamburger
  if (name.includes('menu') || name.includes('bars')) {
    return 'M3 12h18M3 6h18M3 18h18'
  }
  // External link
  if (name.includes('external') || name.includes('arrow-top-right')) {
    return 'M18 13v6a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h6M15 3h6v6M10 14L21 3'
  }
  // Book/documentation
  if (name.includes('book') || name.includes('document')) {
    return 'M4 19.5A2.5 2.5 0 0 1 6.5 17H20M4 19.5A2.5 2.5 0 0 0 6.5 22H20V2H6.5A2.5 2.5 0 0 0 4 4.5v15z'
  }
  // Shield (for AC, defense)
  if (name.includes('shield')) {
    return 'M12 22s8-4 8-10V5l-8-3-8 3v7c0 6 8 10 8 10z'
  }
  // Sword/weapon
  if (name.includes('sword') || name.includes('weapon')) {
    return 'M14.5 17.5L3 6V3h3l11.5 11.5M13 19l6-6M16 16l4 4M19 21l2-2'
  }
  // Dice (D&D!)
  if (name.includes('dice') || name.includes('d20')) {
    return 'M12 2L2 7l10 5 10-5-10-5zM2 17l10 5 10-5M2 12l10 5 10-5'
  }

  // Default: simple circle
  return 'M12 12m-10 0a10 10 0 1 0 20 0 10 10 0 1 0-20 0'
})
</script>

<template>
  <span
    class="inline-flex items-center justify-center shrink-0"
    :class="$attrs.class"
  >
    <svg
      class="w-full h-full"
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      stroke-width="2"
      stroke-linecap="round"
      stroke-linejoin="round"
    >
      <path :d="iconPaths" />
    </svg>
  </span>
</template>
```

**Step 2: Commit**

```bash
git add .storybook/nuxtui-stubs/UIcon.vue
git commit -m "feat(storybook): add UIcon stub with common icon patterns"
```

---

## Task 6: Create UInput Stub

**Files:**
- Create: `.storybook/nuxtui-stubs/UInput.vue`

**Step 1: Create UInput stub**

```vue
<!-- .storybook/nuxtui-stubs/UInput.vue -->
<script setup lang="ts">
/**
 * UInput stub for Storybook
 * Supports: modelValue, placeholder, type, disabled, icon
 */
defineOptions({ name: 'UInput' })

const props = withDefaults(defineProps<{
  modelValue?: string | number
  placeholder?: string
  type?: string
  disabled?: boolean
  icon?: string
  size?: 'xs' | 'sm' | 'md' | 'lg' | 'xl'
}>(), {
  type: 'text',
  size: 'md'
})

const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

const sizeClasses: Record<string, string> = {
  xs: 'text-xs px-2 py-1',
  sm: 'text-sm px-2.5 py-1.5',
  md: 'text-sm px-3 py-2',
  lg: 'text-base px-4 py-2.5',
  xl: 'text-lg px-5 py-3'
}

function handleInput(event: Event) {
  const target = event.target as HTMLInputElement
  emit('update:modelValue', target.value)
}
</script>

<template>
  <div class="relative">
    <!-- Leading icon -->
    <span
      v-if="icon"
      class="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-gray-400"
    >
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" class="w-full h-full">
        <circle cx="11" cy="11" r="8" />
        <path d="m21 21-4.35-4.35" />
      </svg>
    </span>

    <input
      :type="type"
      :value="modelValue"
      :placeholder="placeholder"
      :disabled="disabled"
      class="w-full rounded-lg border border-gray-300 dark:border-gray-700 bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100 placeholder-gray-400 dark:placeholder-gray-500 focus:border-rose-500 dark:focus:border-rose-400 focus:ring-2 focus:ring-rose-500/20 dark:focus:ring-rose-400/20 focus:outline-none transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
      :class="[
        sizeClasses[size],
        icon ? 'pl-10' : ''
      ]"
      v-bind="$attrs"
      @input="handleInput"
    >
  </div>
</template>
```

**Step 2: Commit**

```bash
git add .storybook/nuxtui-stubs/UInput.vue
git commit -m "feat(storybook): add UInput stub"
```

---

## Task 7: Create UAlert Stub

**Files:**
- Create: `.storybook/nuxtui-stubs/UAlert.vue`

**Step 1: Create UAlert stub**

```vue
<!-- .storybook/nuxtui-stubs/UAlert.vue -->
<script setup lang="ts">
import { computed } from 'vue'
import { getColorClasses } from './colors'

/**
 * UAlert stub for Storybook
 * Supports: color, variant, title, description, icon
 */
defineOptions({ name: 'UAlert' })

const props = withDefaults(defineProps<{
  color?: string
  variant?: 'solid' | 'subtle' | 'outline'
  title?: string
  description?: string
  icon?: string
}>(), {
  color: 'primary',
  variant: 'subtle'
})

const colorClasses = computed(() => getColorClasses(props.color, props.variant))
</script>

<template>
  <div
    class="rounded-lg p-4"
    :class="colorClasses"
    role="alert"
    v-bind="$attrs"
  >
    <div class="flex gap-3">
      <!-- Icon placeholder -->
      <span v-if="icon" class="w-5 h-5 shrink-0 mt-0.5">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" class="w-full h-full">
          <circle cx="12" cy="12" r="10" />
          <path d="M12 16v-4M12 8h.01" />
        </svg>
      </span>

      <div class="flex-1">
        <p v-if="title" class="font-semibold">{{ title }}</p>
        <p v-if="description" class="text-sm mt-1 opacity-90">{{ description }}</p>
        <slot />
      </div>
    </div>
  </div>
</template>
```

**Step 2: Commit**

```bash
git add .storybook/nuxtui-stubs/UAlert.vue
git commit -m "feat(storybook): add UAlert stub"
```

---

## Task 8: Create UAccordion Stub

**Files:**
- Create: `.storybook/nuxtui-stubs/UAccordion.vue`

**Step 1: Create UAccordion stub**

```vue
<!-- .storybook/nuxtui-stubs/UAccordion.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

/**
 * UAccordion stub for Storybook
 * Supports: items array with label/content/slot, defaultOpen
 */
defineOptions({ name: 'UAccordion' })

interface AccordionItem {
  label?: string
  icon?: string
  content?: string
  slot?: string
  defaultOpen?: boolean
  disabled?: boolean
}

const props = withDefaults(defineProps<{
  items?: AccordionItem[]
  defaultOpen?: boolean | string | string[]
  multiple?: boolean
}>(), {
  items: () => [],
  multiple: false
})

// Track which items are open
const openItems = ref<Set<number>>(new Set())

// Initialize default open items
if (props.defaultOpen === true && props.items.length > 0) {
  openItems.value.add(0)
} else if (Array.isArray(props.defaultOpen)) {
  props.defaultOpen.forEach((_, i) => openItems.value.add(i))
}

// Also check individual item defaultOpen
props.items.forEach((item, i) => {
  if (item.defaultOpen) openItems.value.add(i)
})

function toggleItem(index: number) {
  if (props.items[index]?.disabled) return

  if (openItems.value.has(index)) {
    openItems.value.delete(index)
  } else {
    if (!props.multiple) {
      openItems.value.clear()
    }
    openItems.value.add(index)
  }
}

function isOpen(index: number): boolean {
  return openItems.value.has(index)
}
</script>

<template>
  <div class="divide-y divide-gray-200 dark:divide-gray-800 border border-gray-200 dark:border-gray-800 rounded-lg overflow-hidden" v-bind="$attrs">
    <div
      v-for="(item, index) in items"
      :key="index"
    >
      <!-- Header -->
      <button
        type="button"
        class="w-full flex items-center justify-between px-4 py-3 text-left bg-white dark:bg-gray-900 hover:bg-gray-50 dark:hover:bg-gray-800 transition-colors"
        :class="{ 'opacity-50 cursor-not-allowed': item.disabled }"
        @click="toggleItem(index)"
      >
        <div class="flex items-center gap-3">
          <!-- Icon placeholder -->
          <span v-if="item.icon" class="w-5 h-5 text-gray-500">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" class="w-full h-full">
              <circle cx="12" cy="12" r="10" />
            </svg>
          </span>
          <span class="font-medium text-gray-900 dark:text-gray-100">{{ item.label }}</span>
        </div>

        <!-- Chevron -->
        <svg
          class="w-5 h-5 text-gray-400 transition-transform duration-200"
          :class="{ 'rotate-180': isOpen(index) }"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <path d="M19 9l-7 7-7-7" />
        </svg>
      </button>

      <!-- Content -->
      <div
        v-if="isOpen(index)"
        class="px-4 py-3 bg-gray-50 dark:bg-gray-800/50 text-gray-700 dark:text-gray-300"
      >
        <slot :name="item.slot || `item-${index}`" :item="item">
          {{ item.content }}
        </slot>
      </div>
    </div>
  </div>
</template>
```

**Step 2: Commit**

```bash
git add .storybook/nuxtui-stubs/UAccordion.vue
git commit -m "feat(storybook): add UAccordion stub"
```

---

## Task 9: Create Remaining Stubs (Batch)

**Files:**
- Create: `.storybook/nuxtui-stubs/USelectMenu.vue`
- Create: `.storybook/nuxtui-stubs/UModal.vue`
- Create: `.storybook/nuxtui-stubs/UDropdownMenu.vue`
- Create: `.storybook/nuxtui-stubs/USkeleton.vue`
- Create: `.storybook/nuxtui-stubs/UFormField.vue`
- Create: `.storybook/nuxtui-stubs/UCheckbox.vue`
- Create: `.storybook/nuxtui-stubs/UButtonGroup.vue`
- Create: `.storybook/nuxtui-stubs/UTabs.vue`
- Create: `.storybook/nuxtui-stubs/UPagination.vue`
- Create: `.storybook/nuxtui-stubs/UToggle.vue`
- Create: `.storybook/nuxtui-stubs/USelect.vue`
- Create: `.storybook/nuxtui-stubs/NuxtLink.vue`

**Step 1: Create USelectMenu stub**

```vue
<!-- .storybook/nuxtui-stubs/USelectMenu.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

defineOptions({ name: 'USelectMenu' })

const props = withDefaults(defineProps<{
  modelValue?: any
  options?: Array<{ label: string; value: any; disabled?: boolean } | string>
  placeholder?: string
  disabled?: boolean
  multiple?: boolean
}>(), {
  options: () => [],
  placeholder: 'Select...'
})

const emit = defineEmits<{
  'update:modelValue': [value: any]
}>()

const isOpen = ref(false)

const normalizedOptions = computed(() => {
  return props.options.map(opt =>
    typeof opt === 'string' ? { label: opt, value: opt } : opt
  )
})

const displayValue = computed(() => {
  if (!props.modelValue) return props.placeholder
  const selected = normalizedOptions.value.find(o => o.value === props.modelValue)
  return selected?.label || props.modelValue
})

function selectOption(option: { label: string; value: any }) {
  emit('update:modelValue', option.value)
  isOpen.value = false
}
</script>

<template>
  <div class="relative" v-bind="$attrs">
    <button
      type="button"
      class="w-full flex items-center justify-between px-3 py-2 rounded-lg border border-gray-300 dark:border-gray-700 bg-white dark:bg-gray-900 text-left"
      :class="{ 'opacity-50 cursor-not-allowed': disabled }"
      :disabled="disabled"
      @click="isOpen = !isOpen"
    >
      <span :class="modelValue ? 'text-gray-900 dark:text-gray-100' : 'text-gray-400'">
        {{ displayValue }}
      </span>
      <svg class="w-4 h-4 text-gray-400" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <path d="M19 9l-7 7-7-7" />
      </svg>
    </button>

    <div
      v-if="isOpen"
      class="absolute z-50 w-full mt-1 py-1 bg-white dark:bg-gray-900 border border-gray-200 dark:border-gray-700 rounded-lg shadow-lg max-h-60 overflow-auto"
    >
      <button
        v-for="option in normalizedOptions"
        :key="option.value"
        type="button"
        class="w-full px-3 py-2 text-left hover:bg-gray-100 dark:hover:bg-gray-800"
        :class="{
          'bg-rose-50 dark:bg-rose-900/20': option.value === modelValue,
          'opacity-50 cursor-not-allowed': option.disabled
        }"
        :disabled="option.disabled"
        @click="selectOption(option)"
      >
        {{ option.label }}
      </button>
    </div>
  </div>
</template>
```

**Step 2: Create UModal stub**

```vue
<!-- .storybook/nuxtui-stubs/UModal.vue -->
<script setup lang="ts">
defineOptions({ name: 'UModal' })

const props = withDefaults(defineProps<{
  modelValue?: boolean
  title?: string
  description?: string
}>(), {
  modelValue: false
})

const emit = defineEmits<{
  'update:modelValue': [value: boolean]
  close: []
}>()

function close() {
  emit('update:modelValue', false)
  emit('close')
}
</script>

<template>
  <Teleport to="body">
    <div
      v-if="modelValue"
      class="fixed inset-0 z-50 flex items-center justify-center p-4"
    >
      <!-- Backdrop -->
      <div
        class="absolute inset-0 bg-black/50"
        @click="close"
      />

      <!-- Modal -->
      <div
        class="relative bg-white dark:bg-gray-900 rounded-xl shadow-xl max-w-lg w-full max-h-[90vh] overflow-auto"
        v-bind="$attrs"
      >
        <!-- Header -->
        <div v-if="title || $slots.header" class="px-6 py-4 border-b border-gray-200 dark:border-gray-800">
          <slot name="header">
            <h3 class="text-lg font-semibold text-gray-900 dark:text-gray-100">{{ title }}</h3>
            <p v-if="description" class="mt-1 text-sm text-gray-500">{{ description }}</p>
          </slot>
        </div>

        <!-- Body -->
        <div class="p-6">
          <slot />
        </div>

        <!-- Footer -->
        <div v-if="$slots.footer" class="px-6 py-4 border-t border-gray-200 dark:border-gray-800 bg-gray-50 dark:bg-gray-800/50">
          <slot name="footer" />
        </div>

        <!-- Close button -->
        <button
          type="button"
          class="absolute top-4 right-4 text-gray-400 hover:text-gray-600 dark:hover:text-gray-300"
          @click="close"
        >
          <svg class="w-5 h-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M6 6l12 12M6 18L18 6" />
          </svg>
        </button>
      </div>
    </div>
  </Teleport>
</template>
```

**Step 3: Create UDropdownMenu stub**

```vue
<!-- .storybook/nuxtui-stubs/UDropdownMenu.vue -->
<script setup lang="ts">
import { ref } from 'vue'

defineOptions({ name: 'UDropdownMenu' })

defineProps<{
  items?: Array<Array<{ label: string; icon?: string; click?: () => void }>>
}>()

const isOpen = ref(false)
</script>

<template>
  <div class="relative inline-block" v-bind="$attrs">
    <div @click="isOpen = !isOpen">
      <slot />
    </div>

    <div
      v-if="isOpen"
      class="absolute right-0 z-50 mt-2 py-1 bg-white dark:bg-gray-900 border border-gray-200 dark:border-gray-700 rounded-lg shadow-lg min-w-[160px]"
    >
      <template v-for="(group, groupIndex) in items" :key="groupIndex">
        <div v-if="groupIndex > 0" class="my-1 border-t border-gray-200 dark:border-gray-700" />
        <button
          v-for="item in group"
          :key="item.label"
          type="button"
          class="w-full px-4 py-2 text-left text-sm hover:bg-gray-100 dark:hover:bg-gray-800 flex items-center gap-2"
          @click="item.click?.(); isOpen = false"
        >
          <span v-if="item.icon" class="w-4 h-4 text-gray-400">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" class="w-full h-full">
              <circle cx="12" cy="12" r="3" />
            </svg>
          </span>
          {{ item.label }}
        </button>
      </template>
    </div>
  </div>
</template>
```

**Step 4: Create USkeleton stub**

```vue
<!-- .storybook/nuxtui-stubs/USkeleton.vue -->
<script setup lang="ts">
defineOptions({ name: 'USkeleton' })
</script>

<template>
  <div
    class="animate-pulse bg-gray-200 dark:bg-gray-700 rounded"
    v-bind="$attrs"
  />
</template>
```

**Step 5: Create UFormField stub**

```vue
<!-- .storybook/nuxtui-stubs/UFormField.vue -->
<script setup lang="ts">
defineOptions({ name: 'UFormField' })

defineProps<{
  label?: string
  description?: string
  error?: string
  required?: boolean
}>()
</script>

<template>
  <div class="space-y-1" v-bind="$attrs">
    <label v-if="label" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
      {{ label }}
      <span v-if="required" class="text-red-500">*</span>
    </label>
    <slot />
    <p v-if="description" class="text-xs text-gray-500">{{ description }}</p>
    <p v-if="error" class="text-xs text-red-500">{{ error }}</p>
  </div>
</template>
```

**Step 6: Create UCheckbox stub**

```vue
<!-- .storybook/nuxtui-stubs/UCheckbox.vue -->
<script setup lang="ts">
defineOptions({ name: 'UCheckbox' })

const props = defineProps<{
  modelValue?: boolean
  label?: string
  disabled?: boolean
}>()

const emit = defineEmits<{
  'update:modelValue': [value: boolean]
}>()
</script>

<template>
  <label class="inline-flex items-center gap-2 cursor-pointer" :class="{ 'opacity-50 cursor-not-allowed': disabled }">
    <input
      type="checkbox"
      :checked="modelValue"
      :disabled="disabled"
      class="w-4 h-4 rounded border-gray-300 dark:border-gray-600 text-rose-600 focus:ring-rose-500"
      @change="emit('update:modelValue', ($event.target as HTMLInputElement).checked)"
    >
    <span v-if="label" class="text-sm text-gray-700 dark:text-gray-300">{{ label }}</span>
    <slot />
  </label>
</template>
```

**Step 7: Create UButtonGroup stub**

```vue
<!-- .storybook/nuxtui-stubs/UButtonGroup.vue -->
<script setup lang="ts">
defineOptions({ name: 'UButtonGroup' })
</script>

<template>
  <div class="inline-flex rounded-lg overflow-hidden divide-x divide-gray-200 dark:divide-gray-700 border border-gray-200 dark:border-gray-700" v-bind="$attrs">
    <slot />
  </div>
</template>
```

**Step 8: Create UTabs stub**

```vue
<!-- .storybook/nuxtui-stubs/UTabs.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

defineOptions({ name: 'UTabs' })

const props = withDefaults(defineProps<{
  items?: Array<{ label: string; slot?: string; disabled?: boolean }>
  defaultIndex?: number
}>(), {
  items: () => [],
  defaultIndex: 0
})

const activeIndex = ref(props.defaultIndex)

function setActive(index: number) {
  if (!props.items[index]?.disabled) {
    activeIndex.value = index
  }
}
</script>

<template>
  <div v-bind="$attrs">
    <!-- Tab headers -->
    <div class="flex border-b border-gray-200 dark:border-gray-700">
      <button
        v-for="(item, index) in items"
        :key="index"
        type="button"
        class="px-4 py-2 text-sm font-medium border-b-2 -mb-px transition-colors"
        :class="[
          activeIndex === index
            ? 'border-rose-500 text-rose-600 dark:text-rose-400'
            : 'border-transparent text-gray-500 hover:text-gray-700 dark:hover:text-gray-300',
          { 'opacity-50 cursor-not-allowed': item.disabled }
        ]"
        @click="setActive(index)"
      >
        {{ item.label }}
      </button>
    </div>

    <!-- Tab content -->
    <div class="py-4">
      <slot :name="items[activeIndex]?.slot || `tab-${activeIndex}`" />
    </div>
  </div>
</template>
```

**Step 9: Create UPagination stub**

```vue
<!-- .storybook/nuxtui-stubs/UPagination.vue -->
<script setup lang="ts">
import { computed } from 'vue'

defineOptions({ name: 'UPagination' })

const props = withDefaults(defineProps<{
  modelValue?: number
  total?: number
  pageCount?: number
}>(), {
  modelValue: 1,
  total: 100,
  pageCount: 10
})

const emit = defineEmits<{
  'update:modelValue': [value: number]
}>()

const totalPages = computed(() => Math.ceil(props.total / props.pageCount))

function goToPage(page: number) {
  if (page >= 1 && page <= totalPages.value) {
    emit('update:modelValue', page)
  }
}
</script>

<template>
  <div class="flex items-center gap-1" v-bind="$attrs">
    <button
      type="button"
      class="px-3 py-1 rounded border border-gray-300 dark:border-gray-700 disabled:opacity-50"
      :disabled="modelValue <= 1"
      @click="goToPage(modelValue - 1)"
    >
      Prev
    </button>

    <span class="px-3 py-1 text-sm">
      {{ modelValue }} / {{ totalPages }}
    </span>

    <button
      type="button"
      class="px-3 py-1 rounded border border-gray-300 dark:border-gray-700 disabled:opacity-50"
      :disabled="modelValue >= totalPages"
      @click="goToPage(modelValue + 1)"
    >
      Next
    </button>
  </div>
</template>
```

**Step 10: Create UToggle stub**

```vue
<!-- .storybook/nuxtui-stubs/UToggle.vue -->
<script setup lang="ts">
defineOptions({ name: 'UToggle' })

const props = defineProps<{
  modelValue?: boolean
  disabled?: boolean
}>()

const emit = defineEmits<{
  'update:modelValue': [value: boolean]
}>()
</script>

<template>
  <button
    type="button"
    role="switch"
    :aria-checked="modelValue"
    class="relative inline-flex h-6 w-11 shrink-0 cursor-pointer rounded-full border-2 border-transparent transition-colors focus:outline-none focus:ring-2 focus:ring-rose-500 focus:ring-offset-2"
    :class="[
      modelValue ? 'bg-rose-600' : 'bg-gray-200 dark:bg-gray-700',
      { 'opacity-50 cursor-not-allowed': disabled }
    ]"
    :disabled="disabled"
    @click="emit('update:modelValue', !modelValue)"
  >
    <span
      class="pointer-events-none inline-block h-5 w-5 transform rounded-full bg-white shadow ring-0 transition"
      :class="modelValue ? 'translate-x-5' : 'translate-x-0'"
    />
  </button>
</template>
```

**Step 11: Create USelect stub**

```vue
<!-- .storybook/nuxtui-stubs/USelect.vue -->
<script setup lang="ts">
import { computed } from 'vue'

defineOptions({ name: 'USelect' })

const props = withDefaults(defineProps<{
  modelValue?: any
  options?: Array<{ label: string; value: any } | string>
  placeholder?: string
  disabled?: boolean
}>(), {
  options: () => [],
  placeholder: 'Select...'
})

const emit = defineEmits<{
  'update:modelValue': [value: any]
}>()

const normalizedOptions = computed(() => {
  return props.options.map(opt =>
    typeof opt === 'string' ? { label: opt, value: opt } : opt
  )
})
</script>

<template>
  <select
    :value="modelValue"
    :disabled="disabled"
    class="w-full px-3 py-2 rounded-lg border border-gray-300 dark:border-gray-700 bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100"
    v-bind="$attrs"
    @change="emit('update:modelValue', ($event.target as HTMLSelectElement).value)"
  >
    <option v-if="placeholder" value="" disabled>{{ placeholder }}</option>
    <option v-for="option in normalizedOptions" :key="option.value" :value="option.value">
      {{ option.label }}
    </option>
  </select>
</template>
```

**Step 12: Create NuxtLink stub**

```vue
<!-- .storybook/nuxtui-stubs/NuxtLink.vue -->
<script setup lang="ts">
defineOptions({ name: 'NuxtLink' })

defineProps<{
  to?: string | { name?: string; params?: Record<string, any> }
  href?: string
  external?: boolean
  target?: string
}>()
</script>

<template>
  <a
    :href="typeof to === 'string' ? to : href || '#'"
    :target="external ? '_blank' : target"
    :rel="external ? 'noopener noreferrer' : undefined"
    v-bind="$attrs"
  >
    <slot />
  </a>
</template>
```

**Step 13: Commit all secondary stubs**

```bash
git add .storybook/nuxtui-stubs/
git commit -m "feat(storybook): add remaining NuxtUI component stubs"
```

---

## Task 10: Create Index File and Register Stubs

**Files:**
- Create: `.storybook/nuxtui-stubs/index.ts`
- Modify: `.storybook/preview.ts`

**Step 1: Create index.ts to export and register all stubs**

```typescript
// .storybook/nuxtui-stubs/index.ts

import type { App } from 'vue'

// Import all stub components
import UCard from './UCard.vue'
import UBadge from './UBadge.vue'
import UButton from './UButton.vue'
import UIcon from './UIcon.vue'
import UInput from './UInput.vue'
import UAlert from './UAlert.vue'
import UAccordion from './UAccordion.vue'
import USelectMenu from './USelectMenu.vue'
import UModal from './UModal.vue'
import UDropdownMenu from './UDropdownMenu.vue'
import USkeleton from './USkeleton.vue'
import UFormField from './UFormField.vue'
import UCheckbox from './UCheckbox.vue'
import UButtonGroup from './UButtonGroup.vue'
import UTabs from './UTabs.vue'
import UPagination from './UPagination.vue'
import UToggle from './UToggle.vue'
import USelect from './USelect.vue'
import NuxtLink from './NuxtLink.vue'

// Export all components
export {
  UCard,
  UBadge,
  UButton,
  UIcon,
  UInput,
  UAlert,
  UAccordion,
  USelectMenu,
  UModal,
  UDropdownMenu,
  USkeleton,
  UFormField,
  UCheckbox,
  UButtonGroup,
  UTabs,
  UPagination,
  UToggle,
  USelect,
  NuxtLink
}

// Component map for registration
const components = {
  UCard,
  UBadge,
  UButton,
  UIcon,
  UInput,
  UAlert,
  UAccordion,
  USelectMenu,
  UModal,
  UDropdownMenu,
  USkeleton,
  UFormField,
  UCheckbox,
  UButtonGroup,
  UTabs,
  UPagination,
  UToggle,
  USelect,
  NuxtLink
}

/**
 * Register all NuxtUI stub components globally on a Vue app instance
 */
export function registerNuxtUIStubs(app: App): void {
  Object.entries(components).forEach(([name, component]) => {
    app.component(name, component)
  })
}
```

**Step 2: Update preview.ts to use new stubs**

```typescript
// .storybook/preview.ts
import type { Preview } from '@storybook/vue3'
import { setup } from '@storybook/vue3'

// Import Storybook-specific CSS (includes Tailwind + theme)
import './preview.css'

// Import and register all NuxtUI stubs
import { registerNuxtUIStubs } from './nuxtui-stubs'

// Setup Vue app with global stubs
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
        { name: 'dark', value: '#0c0a09' }    // dark mode background
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

**Step 3: Commit**

```bash
git add .storybook/nuxtui-stubs/index.ts .storybook/preview.ts
git commit -m "feat(storybook): register NuxtUI stubs globally"
```

---

## Task 11: Test Existing Stories

**Step 1: Start Storybook**

Run: `docker compose exec nuxt npm run storybook`
Expected: Storybook starts on port 6006

**Step 2: Verify existing stories render**

Open: `http://localhost:6006`

Check these stories render without errors:
- UI/List/PageHeader - Should show title and count
- UI/List/ResultsCount - Should show result count text
- UI/List/EmptyState - Should show empty state message
- UI/BackLink - Should show back link with arrow
- UI/List/SkeletonCards - Should show loading skeleton

**Step 3: Fix any issues found**

If any story fails to render, debug and fix before proceeding.

**Step 4: Commit any fixes**

```bash
git add -A
git commit -m "fix(storybook): resolve rendering issues with existing stories"
```

---

## Task 12: Add SpellCard Story

**Files:**
- Create: `app/components/spell/Card.stories.ts`

**Step 1: Create SpellCard story**

```typescript
// app/components/spell/Card.stories.ts
import type { Meta, StoryObj } from '@storybook/vue3'
import SpellCard from './Card.vue'

/**
 * SpellCard displays a D&D 5e spell with level, school, casting time,
 * concentration indicator, and class availability.
 */
const meta: Meta<typeof SpellCard> = {
  title: 'Entities/Spell/SpellCard',
  component: SpellCard,
  tags: ['autodocs'],
  argTypes: {
    spell: {
      description: 'The spell data object'
    }
  }
}

export default meta
type Story = StoryObj<typeof SpellCard>

// Mock spell data matching API structure
const mockFireball = {
  id: 1,
  name: 'Fireball',
  slug: 'fireball',
  level: 3,
  school: { name: 'Evocation', slug: 'evocation' },
  casting_time: '1 action',
  range: '150 feet',
  duration: 'Instantaneous',
  concentration: false,
  ritual: false,
  components: ['V', 'S', 'M'],
  material: 'A tiny ball of bat guano and sulfur',
  description: 'A bright streak flashes from your pointing finger...',
  classes: [
    { name: 'Sorcerer', slug: 'sorcerer' },
    { name: 'Wizard', slug: 'wizard' }
  ],
  source: { name: "Player's Handbook", abbreviation: 'PHB' }
}

const mockHoldPerson = {
  id: 2,
  name: 'Hold Person',
  slug: 'hold-person',
  level: 2,
  school: { name: 'Enchantment', slug: 'enchantment' },
  casting_time: '1 action',
  range: '60 feet',
  duration: 'Concentration, up to 1 minute',
  concentration: true,
  ritual: false,
  components: ['V', 'S', 'M'],
  material: 'A small, straight piece of iron',
  description: 'Choose a humanoid that you can see within range...',
  classes: [
    { name: 'Bard', slug: 'bard' },
    { name: 'Cleric', slug: 'cleric' },
    { name: 'Druid', slug: 'druid' },
    { name: 'Sorcerer', slug: 'sorcerer' },
    { name: 'Warlock', slug: 'warlock' },
    { name: 'Wizard', slug: 'wizard' }
  ],
  source: { name: "Player's Handbook", abbreviation: 'PHB' }
}

const mockDetectMagic = {
  id: 3,
  name: 'Detect Magic',
  slug: 'detect-magic',
  level: 1,
  school: { name: 'Divination', slug: 'divination' },
  casting_time: '1 action',
  range: 'Self',
  duration: 'Concentration, up to 10 minutes',
  concentration: true,
  ritual: true,
  components: ['V', 'S'],
  description: 'For the duration, you sense the presence of magic...',
  classes: [
    { name: 'Bard', slug: 'bard' },
    { name: 'Cleric', slug: 'cleric' },
    { name: 'Druid', slug: 'druid' },
    { name: 'Paladin', slug: 'paladin' },
    { name: 'Ranger', slug: 'ranger' },
    { name: 'Sorcerer', slug: 'sorcerer' },
    { name: 'Wizard', slug: 'wizard' }
  ],
  source: { name: "Player's Handbook", abbreviation: 'PHB' }
}

const mockLight = {
  id: 4,
  name: 'Light',
  slug: 'light',
  level: 0,
  school: { name: 'Evocation', slug: 'evocation' },
  casting_time: '1 action',
  range: 'Touch',
  duration: '1 hour',
  concentration: false,
  ritual: false,
  components: ['V', 'M'],
  material: 'A firefly or phosphorescent moss',
  description: 'You touch one object that is no larger than 10 feet in any dimension...',
  classes: [
    { name: 'Bard', slug: 'bard' },
    { name: 'Cleric', slug: 'cleric' },
    { name: 'Sorcerer', slug: 'sorcerer' },
    { name: 'Wizard', slug: 'wizard' }
  ],
  source: { name: "Player's Handbook", abbreviation: 'PHB' }
}

/**
 * Standard spell card showing a 3rd level evocation spell
 */
export const Default: Story = {
  args: {
    spell: mockFireball
  }
}

/**
 * Spell requiring concentration - shows concentration badge
 */
export const Concentration: Story = {
  args: {
    spell: mockHoldPerson
  }
}

/**
 * Ritual spell - shows both concentration and ritual badges
 */
export const Ritual: Story = {
  args: {
    spell: mockDetectMagic
  }
}

/**
 * Cantrip (level 0) - shows "Cantrip" instead of level number
 */
export const Cantrip: Story = {
  args: {
    spell: mockLight
  }
}
```

**Step 2: Commit**

```bash
git add app/components/spell/Card.stories.ts
git commit -m "feat(storybook): add SpellCard stories"
```

---

## Task 13: Add MonsterCard Story

**Files:**
- Create: `app/components/monster/MonsterCard.stories.ts`

**Step 1: Create MonsterCard story**

```typescript
// app/components/monster/MonsterCard.stories.ts
import type { Meta, StoryObj } from '@storybook/vue3'
import MonsterCard from './MonsterCard.vue'

/**
 * MonsterCard displays a D&D 5e monster with CR, type, size,
 * hit points, armor class, and speeds.
 */
const meta: Meta<typeof MonsterCard> = {
  title: 'Entities/Monster/MonsterCard',
  component: MonsterCard,
  tags: ['autodocs']
}

export default meta
type Story = StoryObj<typeof MonsterCard>

const mockGoblin = {
  id: 1,
  name: 'Goblin',
  slug: 'goblin',
  challenge_rating: '1/4',
  challenge_rating_decimal: 0.25,
  xp: 50,
  type: 'Humanoid',
  subtype: 'goblinoid',
  size: { name: 'Small', abbreviation: 'S' },
  alignment: 'Neutral Evil',
  hit_points: 7,
  hit_dice: '2d6',
  armor_class: 15,
  armor_type: 'leather armor, shield',
  speeds: { walk: 30 },
  source: { name: "Monster Manual", abbreviation: 'MM' }
}

const mockAdultRedDragon = {
  id: 2,
  name: 'Adult Red Dragon',
  slug: 'adult-red-dragon',
  challenge_rating: '17',
  challenge_rating_decimal: 17,
  xp: 18000,
  type: 'Dragon',
  size: { name: 'Huge', abbreviation: 'H' },
  alignment: 'Chaotic Evil',
  hit_points: 256,
  hit_dice: '19d12+133',
  armor_class: 19,
  armor_type: 'natural armor',
  speeds: { walk: 40, climb: 40, fly: 80 },
  source: { name: "Monster Manual", abbreviation: 'MM' }
}

const mockAboleth = {
  id: 3,
  name: 'Aboleth',
  slug: 'aboleth',
  challenge_rating: '10',
  challenge_rating_decimal: 10,
  xp: 5900,
  type: 'Aberration',
  size: { name: 'Large', abbreviation: 'L' },
  alignment: 'Lawful Evil',
  hit_points: 135,
  hit_dice: '18d10+36',
  armor_class: 17,
  armor_type: 'natural armor',
  speeds: { walk: 10, swim: 40 },
  source: { name: "Monster Manual", abbreviation: 'MM' }
}

/**
 * Low CR humanoid monster
 */
export const Default: Story = {
  args: {
    monster: mockGoblin
  }
}

/**
 * High CR dragon with multiple movement types
 */
export const HighCR: Story = {
  args: {
    monster: mockAdultRedDragon
  }
}

/**
 * Aberration with swim speed
 */
export const Aberration: Story = {
  args: {
    monster: mockAboleth
  }
}
```

**Step 2: Commit**

```bash
git add app/components/monster/MonsterCard.stories.ts
git commit -m "feat(storybook): add MonsterCard stories"
```

---

## Task 14: Verify and Final Test

**Step 1: Restart Storybook**

Run: `docker compose exec nuxt npm run storybook`

**Step 2: Verify all stories**

Check in browser at `http://localhost:6006`:

- [ ] UI/List/* stories render correctly
- [ ] Entities/Spell/SpellCard shows purple badges (arcane color)
- [ ] Entities/Monster/MonsterCard shows orange badges (danger color)
- [ ] Light/dark mode backgrounds work via toolbar
- [ ] No console errors

**Step 3: Final commit**

```bash
git add -A
git commit -m "feat(storybook): complete NuxtUI stub library implementation

- Add 19 NuxtUI component stubs with D&D color support
- Add color mapping utility for custom entity colors
- Add SpellCard and MonsterCard stories
- Update preview.ts to use new stub system

Closes #13"
```

---

## Summary

| Task | Description | Est. Time |
|------|-------------|-----------|
| 1 | Color mapping utility | 10 min |
| 2 | UCard stub | 10 min |
| 3 | UBadge stub | 15 min |
| 4 | UButton stub | 15 min |
| 5 | UIcon stub | 20 min |
| 6 | UInput stub | 10 min |
| 7 | UAlert stub | 10 min |
| 8 | UAccordion stub | 15 min |
| 9 | Remaining stubs (batch) | 45 min |
| 10 | Index file + preview.ts | 15 min |
| 11 | Test existing stories | 15 min |
| 12 | SpellCard story | 15 min |
| 13 | MonsterCard story | 15 min |
| 14 | Final verification | 15 min |

**Total: ~3.5 hours**
