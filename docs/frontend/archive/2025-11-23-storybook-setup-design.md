# Storybook Setup - Design Document

**Date:** 2025-11-23
**Status:** Approved
**Approach:** Vanilla Storybook (Vue 3 + Vite)

---

## Overview

Add Storybook 8 to the D&D 5e Compendium frontend for visual component documentation, interactive playground, and design system showcase.

### Goals

1. **Component Discovery** - Make the 33+ reusable components easy to find and understand
2. **Interactive Documentation** - Allow props/variants exploration without running the full app
3. **Design System** - Document colors, typography, spacing, and patterns
4. **Developer Onboarding** - Help new developers understand component APIs and usage patterns

### Non-Goals

- Full Nuxt runtime integration (composables, server features)
- E2E testing (Playwright handles that)
- Replace existing Vitest unit tests

---

## Architecture Decision: Vanilla Storybook

### Why NOT @nuxtjs/storybook Module?

- **Nuxt 4 support unclear** - Official docs mention Nuxt 3, no explicit Nuxt 4 confirmation
- **Additional complexity** - Module layer adds moving parts
- **Our components are portable** - Minimal Nuxt dependencies, mostly presentational

### Why Vanilla Storybook (Vue 3 + Vite)?

- ✅ **Battle-tested** - Storybook 8 + Vue 3 + Vite is production-ready
- ✅ **Future-proof** - Independent of Nuxt release cycle
- ✅ **Better docs** - Standard Storybook documentation applies
- ✅ **Simpler** - Just Vue components, explicit imports
- ✅ **Docker-friendly** - Easy to run in container environment

---

## Technical Design

### File Structure

```
frontend/
├── .storybook/
│   ├── main.ts              # Storybook configuration (framework, addons, aliases)
│   ├── preview.ts           # Global decorators (NuxtUI provider, theme)
│   ├── preview-head.html    # Custom fonts, meta tags
│   └── manager.ts           # Storybook UI customization (optional)
├── app/
│   └── components/
│       └── ui/
│           ├── BackLink.vue
│           ├── BackLink.stories.ts    # ← Co-located stories
│           ├── SourceDisplay.vue
│           ├── SourceDisplay.stories.ts
│           └── ...
├── docs/
│   ├── components/
│   │   └── README.md        # Component library overview
│   └── storybook/
│       ├── getting-started.mdx
│       ├── design-system.mdx
│       ├── list-page-pattern.mdx
│       └── detail-page-pattern.mdx
├── tests/
│   └── fixtures/
│       ├── spells.ts        # Mock spell data for stories
│       ├── items.ts
│       └── ...
└── package.json
```

### Integration Challenges & Solutions

#### Challenge 1: Path Aliases (`~/components`, `~/types`)

**Solution:** Configure Vite in `.storybook/main.ts`:

```typescript
import path from 'path'

export default {
  viteFinal: async (config) => {
    config.resolve.alias = {
      ...config.resolve.alias,
      '~': path.resolve(__dirname, '../app'),
      '@': path.resolve(__dirname, '../app')
    }
    return config
  }
}
```

#### Challenge 2: NuxtUI Components (`<UCard>`, `<UBadge>`)

**Solution:** Global decorator wrapping all stories in `<UProvider>`:

```typescript
// .storybook/preview.ts
import { setup } from '@storybook/vue3'

setup((app) => {
  // Register NuxtUI components globally
  // This makes <UCard>, <UBadge>, etc. available
})

export const decorators = [
  (story) => ({
    components: { story },
    template: '<div class="p-4"><story /></div>'
  })
]
```

#### Challenge 3: Auto-imports

**Current:** Nuxt auto-imports `ref`, `computed`, composables
**Storybook:** Requires explicit imports

**Solution:** Accept this trade-off. Explicit imports in stories are actually **better for documentation** - readers see exactly what's being used.

```typescript
// Story file
import { ref } from 'vue'
import BackLink from './BackLink.vue'
```

#### Challenge 4: Tailwind Theme

**Solution:** Import app's Tailwind config in `.storybook/preview.ts`:

```typescript
import '../app/assets/css/main.css'
```

This ensures Storybook components use the same colors, spacing, and typography as the live app.

---

## Component Priority

### Tier 1: Core UI Components (5 components)
**Why first:** Highest visibility, used on every list page

1. `UiListPageHeader` - Title, count, description, loading states
2. `UiListSkeletonCards` - Animated loading placeholders
3. `UiListEmptyState` - Empty results with filter clearing
4. `UiListResultsCount` - Pagination info (1-24 of 150)
5. `BackLink` - Breadcrumb navigation

### Tier 2: Card Components (4 components)
**Why second:** Visual variety, show design system in action

6. `SpellCard` - Complex (level, school, badges, sources)
7. `ItemCard` - Rarity colors, magic/attunement
8. `MonsterCard` - CR badges, type display
9. `SourceCard` - Simplest card (baseline example)

### Tier 3: Display Components (3 components)
**Why third:** Reusable patterns used across entities

10. `UiSourceDisplay` - Source citation formatting
11. `UiModifiersDisplay` - Advantage/disadvantage badges
12. `UiTagsDisplay` - Tag pill display

### Tier 4: Accordion Components (3 components)
**Why fourth:** Complex nested data, good examples

13. `UiAccordionBadgeList` - Badge collections
14. `UiAccordionBulletList` - Bullet point lists
15. `UiAccordionDamageEffects` - Spell effects with scaling

**Total: 15 components** for initial launch
**Remaining: 18+ components** to add incrementally

---

## Story Structure

### Component Story Format (CSF) 3.0

```typescript
// BackLink.stories.ts
import type { Meta, StoryObj } from '@storybook/vue3'
import BackLink from './BackLink.vue'

const meta: Meta<typeof BackLink> = {
  title: 'UI/Navigation/BackLink',        // Sidebar organization
  component: BackLink,
  tags: ['autodocs'],                     // Auto-generate prop docs
  argTypes: {
    to: {
      control: 'text',
      description: 'Destination route path'
    },
    label: {
      control: 'text',
      description: 'Link text (optional)'
    }
  }
}

export default meta
type Story = StoryObj<typeof BackLink>

// Stories = component variants
export const Default: Story = {
  args: {
    to: '/spells',
    label: 'Back to Spells'
  }
}

export const HomePage: Story = {
  args: {
    to: '/',
    label: 'Home'
  }
}

export const WithoutLabel: Story = {
  args: {
    to: '/items'
    // label omitted - tests optional prop
  }
}
```

### Story Naming Convention

- **File:** `ComponentName.stories.ts` (co-located with component)
- **Title:** `Category/Subcategory/ComponentName`
  - `UI/Navigation/BackLink`
  - `UI/List/PageHeader`
  - `Cards/Entity/SpellCard`
  - `UI/Accordion/BadgeList`
- **Exports:** PascalCase describing variant (`Default`, `WithIcon`, `Loading`, `Error`)

---

## Documentation Levels

### Level 1: JSDoc Comments (Component Props)

Add TypeScript-friendly documentation to all props:

```vue
<script setup lang="ts">
interface Props {
  /**
   * The destination route path
   * @example '/spells'
   * @example '/items/longsword'
   */
  to: string

  /**
   * Link text to display
   * @default Derived from route (e.g., "Back to Spells")
   */
  label?: string

  /**
   * Show home icon for root navigation
   * @default false
   */
  showHomeIcon?: boolean
}

defineProps<Props>()
</script>
```

**Benefits:**
- Shows in IDE autocomplete
- Appears in Storybook auto-docs
- Type-safe documentation

### Level 2: Story Descriptions (Usage Guidance)

Add context to specific variants:

```typescript
export const WithHomeIcon: Story = {
  args: { to: '/', showHomeIcon: true },
  parameters: {
    docs: {
      description: {
        story: 'Use the home icon for root-level navigation to make it visually distinct from entity back links.'
      }
    }
  }
}
```

### Level 3: MDX Documentation (Patterns & Systems)

Create comprehensive guides:

**`docs/components/README.md`** - Component library overview
- Architecture explanation
- When to use which component
- How to create new components

**`docs/storybook/design-system.mdx`** - Visual design system
- Color palette with swatches
- Typography scale with examples
- Spacing system
- Icon usage

**`docs/storybook/list-page-pattern.mdx`** - List page implementation
- Step-by-step guide
- Required components
- Filter integration
- Example code

**`docs/storybook/detail-page-pattern.mdx`** - Detail page implementation
- Layout structure
- Accordion usage
- Image integration
- Example code

---

## Mock Data Strategy

### Create Fixture Files

```typescript
// tests/fixtures/spells.ts
import type { Spell } from '~/types'

export const mockFireball: Spell = {
  id: 1,
  name: 'Fireball',
  slug: 'fireball',
  level: 3,
  school: { id: 5, name: 'Evocation' },
  description: 'A bright streak flashes...',
  // ... complete realistic data
}

export const mockCantrip: Spell = {
  id: 2,
  name: 'Fire Bolt',
  level: 0,
  // ...
}
```

### Use in Stories

```typescript
import { mockFireball, mockCantrip } from '@/tests/fixtures/spells'

export const LevelThreeSpell: Story = {
  args: {
    spell: mockFireball
  }
}

export const Cantrip: Story = {
  args: {
    spell: mockCantrip
  }
}
```

**Benefits:**
- Realistic examples
- Reusable across stories and tests
- Single source of truth for mock data

---

## Development Workflow

### Running Storybook

```bash
# Start Storybook dev server
npm run storybook

# Build static Storybook for deployment
npm run build-storybook

# Run alongside Nuxt dev server
npm run dev  # Terminal 1 (http://localhost:3000)
npm run storybook  # Terminal 2 (http://localhost:6006)
```

### Docker Integration

```bash
# Inside Docker container
docker compose exec nuxt npm run storybook

# Access at http://localhost:6006
```

### Adding New Stories

1. Create `ComponentName.stories.ts` next to component
2. Import component and types
3. Define meta with title and controls
4. Export 2-4 story variants
5. Add JSDoc to component props
6. Verify in Storybook UI

---

## Implementation Plan

### Phase A: Foundation (30-45 min)

1. Install Storybook via `npx storybook@latest init`
2. Configure `.storybook/main.ts` with path aliases
3. Set up `.storybook/preview.ts` with NuxtUI and Tailwind
4. Create first story (`BackLink.stories.ts`) to validate setup
5. Run `npm run storybook` and verify component renders

**Success criteria:** BackLink component visible in Storybook with working controls

### Phase B: Core Stories (1.5-2 hours)

1. Create Tier 1 stories (5 list components)
2. Create Tier 2 stories (4 card components)
3. Add mock data fixtures
4. Test all interactive controls
5. Verify dark mode works

**Success criteria:** 9 components documented with multiple variants each

### Phase C: Documentation (1 hour)

1. Add JSDoc comments to all 15 components
2. Write `docs/components/README.md`
3. Create `docs/storybook/design-system.mdx`
4. Create `docs/storybook/list-page-pattern.mdx`

**Success criteria:** Searchable documentation with live examples

### Phase D: Remaining Stories (30-45 min)

1. Create Tier 3 stories (3 display components)
2. Create Tier 4 stories (3 accordion components)
3. Final browser testing
4. Update CHANGELOG.md

**Success criteria:** 15 components fully documented in Storybook

### Phase E: Commit (5 min)

1. Run full test suite to ensure no regressions
2. Commit with comprehensive message
3. Update CURRENT_STATUS.md

---

## Success Criteria

✅ **Storybook runs** at `http://localhost:6006`
✅ **15 components documented** with interactive controls
✅ **NuxtUI theme works** (components styled correctly)
✅ **Dark mode functions** (toggle between light/dark)
✅ **Path aliases resolve** (`~/components`, `~/types`)
✅ **JSDoc appears** in auto-generated prop tables
✅ **Mock data realistic** (actual D&D entities)
✅ **Documentation searchable** (design system, patterns)
✅ **No test regressions** (731/731 tests still passing)
✅ **Docker compatible** (runs in container)

---

## Future Enhancements

### Phase 2 (Later)
- Add remaining 18+ components
- Create MDX stories for complex workflows
- Add Storybook addons (a11y, interactions)
- Set up Chromatic for visual regression testing
- Deploy Storybook to GitHub Pages or Netlify

### Phase 3 (Optional)
- Add Storybook interactions (play functions)
- Create component templates/generators
- Add design tokens addon
- Integrate with Figma designs

---

## Appendix: Storybook Addons

**Included by default:**
- `@storybook/addon-essentials` - Controls, docs, viewport, backgrounds
- `@storybook/addon-links` - Navigation between stories

**Recommended for future:**
- `@storybook/addon-a11y` - Accessibility testing
- `@storybook/addon-interactions` - User interaction testing
- `storybook-dark-mode` - Enhanced dark mode toggle

---

## Resources

- [Storybook Vue 3 + Vite Docs](https://storybook.js.org/docs/get-started/frameworks/vue3-vite)
- [Component Story Format 3.0](https://storybook.js.org/docs/api/csf)
- [Writing Stories](https://storybook.js.org/docs/writing-stories)
- [Writing Docs](https://storybook.js.org/docs/writing-docs)

---

**Status:** Ready for implementation
**Estimated time:** 4-5 hours
**Next step:** Phase A - Foundation setup
