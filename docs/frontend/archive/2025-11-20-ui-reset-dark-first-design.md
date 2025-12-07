# UI Reset - Dark-First Design

**Date:** 2025-11-20
**Status:** Implementation Ready
**Approach:** Complete configuration reset with dark-first design

---

## Problem Statement

Current UI has critical issues:
- Fonts not rendering correctly
- Icons not scaled properly
- No contrast (completely white page)
- Dark mode toggle not working
- Configuration conflicts between NuxtUI, Tailwind, and custom configs

## Solution Overview

**Complete reset approach:** Remove all configuration overrides, use NuxtUI defaults, rebuild layout from scratch with pure HTML + Tailwind.

**Design Philosophy:** Dark-first design (dark mode as primary, light mode as fallback)

---

## Architecture Decisions

### 1. Configuration Reset

**Remove all custom configurations:**
- ✅ Delete `app.config.ts` - eliminates NuxtUI color overrides
- ✅ Delete `tailwind.config.ts` - let NuxtUI handle Tailwind entirely
- ✅ Reset `main.css` - single line: `@import "tailwindcss"`

**Why:** Configuration conflicts are causing the issues. NuxtUI's defaults are excellent and well-tested. Our overrides were fighting with the framework.

### 2. Layout Structure

**Replace UApp/UHeader/UFooter with custom HTML + Tailwind:**

```vue
<template>
  <div class="min-h-screen bg-gray-50 dark:bg-gray-950">
    <nav class="border-b border-gray-200 dark:border-gray-800 bg-white dark:bg-gray-900">
      <!-- Navigation content -->
    </nav>

    <main class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
      <NuxtPage />
    </main>
  </div>
</template>
```

**Why:**
- Maximum control over layout
- Eliminates UApp wrapper issues
- Simple, predictable structure
- Easy to debug

### 3. Dark Mode Implementation

**Use NuxtUI's built-in color mode:**
- `useColorMode()` composable
- `UColorModeButton` component for toggle
- Automatic localStorage persistence
- Works with Tailwind's `dark:` classes

**Color Scheme:**
- **Dark mode (primary):** `gray-950` background, `gray-900` surfaces, `gray-800` borders
- **Light mode (fallback):** `gray-50` background, `white` surfaces, `gray-200` borders

**Why:** Tailwind's gray scale provides proper contrast ratios automatically. Dark mode feels premium and reduces eye strain.

### 4. Navigation Design

**Structure:**
- Left: Site title (clickable home link)
- Center: Entity type links (6 buttons)
- Right: Dark mode toggle

**Components:**
- Use `UButton` for navigation (consistent hover states, accessibility)
- Simple, flat design
- Mobile-responsive (horizontal on desktop, can stack on mobile)

### 5. Component Preservation

**Keep existing components unchanged:**
- ✅ `SearchInput.vue` - Already has proper dark mode classes
- ✅ `SearchResultCard.vue` - Already has proper dark mode classes
- ✅ `useSearch.ts` - No changes needed
- ✅ All page components - Will work once foundation is fixed

**Why:** These components are already well-built with proper `dark:` classes throughout. No need to touch them.

---

## Implementation Steps

### Step 1: Clean Slate
1. Delete `app.config.ts`
2. Delete `tailwind.config.ts`
3. Reset `app/assets/css/main.css` to single import
4. Backup old `app.vue` (rename to `app.vue.backup`)

### Step 2: Rebuild app.vue
1. Create minimal layout structure
2. Add navigation bar with proper dark mode classes
3. Add main content area
4. Test that page loads without errors

### Step 3: Add Navigation
1. Add site title link
2. Add 6 entity type buttons (Spells, Items, Races, Classes, Backgrounds, Feats)
3. Add dark mode toggle (UColorModeButton)
4. Test navigation works

### Step 4: Verify Dark Mode
1. Test toggle button works
2. Verify dark mode persists on page reload
3. Check all colors have proper contrast
4. Test in both light and dark modes

### Step 5: Test Existing Components
1. Navigate to search page
2. Test SearchInput component works
3. Test SearchResultCard rendering
4. Verify all entity list pages work
5. Verify all entity detail pages work

---

## Expected Outcomes

**Fixed Issues:**
- ✅ Proper font rendering (system fonts via Tailwind defaults)
- ✅ Correct icon scaling (NuxtUI icon system)
- ✅ Proper contrast (Tailwind gray scale)
- ✅ Working dark mode toggle (useColorMode + UColorModeButton)
- ✅ No configuration conflicts (minimal config)

**Preserved Functionality:**
- ✅ All search features work
- ✅ All entity pages work
- ✅ All custom components work
- ✅ All composables work

---

## Visual Design Specification

### Dark Mode (Primary)
- Background: `bg-gray-950` (#030712) - Almost black
- Surface: `bg-gray-900` (#111827) - Elevated elements
- Border: `border-gray-800` (#1F2937) - Subtle divisions
- Text Primary: `text-gray-100` (#F3F4F6) - High contrast
- Text Secondary: `text-gray-400` (#9CA3AF) - Lower emphasis

### Light Mode (Fallback)
- Background: `bg-gray-50` (#F9FAFB) - Very light gray
- Surface: `bg-white` (#FFFFFF) - Pure white
- Border: `border-gray-200` (#E5E7EB) - Subtle divisions
- Text Primary: `text-gray-900` (#111827) - High contrast
- Text Secondary: `text-gray-600` (#4B5563) - Lower emphasis

### Interactive Elements
- Links: Use `text-blue-600 dark:text-blue-400` for primary actions
- Hover: `hover:bg-gray-100 dark:hover:bg-gray-800` for buttons
- Focus: Tailwind's default focus rings (accessible)

---

## Testing Checklist

**After Implementation:**
- [ ] Page loads without errors (no 500)
- [ ] Fonts render correctly (system fonts)
- [ ] Icons scale properly
- [ ] Background has proper color (not pure white)
- [ ] Dark mode toggle works
- [ ] Dark mode persists on reload
- [ ] All navigation links work
- [ ] SearchInput component works
- [ ] SearchResultCard component works
- [ ] All entity pages load
- [ ] Mobile responsive design works
- [ ] No console errors

---

## Risk Mitigation

**Backup Strategy:**
- Old `app.vue` saved as `app.vue.backup`
- Old configs saved as `.ts.backup` files
- Can revert quickly if needed

**Rollback Plan:**
If implementation fails:
1. Restore `app.vue.backup` to `app.vue`
2. Restore config files
3. Restart dev server
4. Investigate specific issue

---

## Success Criteria

**Must Have:**
- ✅ Dark mode works perfectly
- ✅ Page has proper contrast and colors
- ✅ All existing functionality preserved
- ✅ No configuration errors

**Nice to Have:**
- Clean, minimal code
- Easy to understand layout
- Good performance
- Accessible navigation

---

**Status:** Ready for implementation
