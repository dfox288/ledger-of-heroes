# Handover: Migration to @nuxtjs/storybook Module

**Date:** 2025-11-23
**Status:** ❌ **BLOCKED** - See `HANDOVER-2025-11-23-STORYBOOK-MIGRATION-BLOCKER.md`
**Current:** Standalone Storybook 8.6.14 (with --legacy-peer-deps)
**Target:** @nuxtjs/storybook (Nuxt-integrated) - NOT COMPATIBLE WITH NUXT 4 + VITE 7

---

---

## ⚠️ MIGRATION BLOCKED - READ THIS FIRST

**This migration cannot proceed due to fundamental compatibility issues between Storybook and Nuxt 4 + Vite 7.**

For complete details, see: **`HANDOVER-2025-11-23-STORYBOOK-MIGRATION-BLOCKER.md`**

**TL;DR:**
- Neither Storybook 8.6.14 NOR @nuxtjs/storybook@9.0.1 are compatible with Vite 7
- Current setup only works with `--legacy-peer-deps` workaround
- Migration postponed until Storybook releases Vite 7 support

---

## 🎯 Why Migrate? (Original Rationale - For Future Reference)

### Current Limitations (Standalone Storybook)
- ❌ **No real NuxtUI components** - Using stubs with Tailwind approximations
- ❌ **Missing component styles** - NuxtUI v4's `@source` directive doesn't work outside Nuxt
- ❌ **Manual configuration** - Separate Storybook config, preview setup, CSS management
- ❌ **No Nuxt context** - Can't use composables, `#imports`, `#build` aliases
- ⚠️ **Visual inaccuracy** - Stubs don't match real components perfectly

### Benefits of @nuxtjs/storybook
- ✅ **Real NuxtUI components** - Full styling, interactions, and behavior
- ✅ **Automatic setup** - Integrates into Nuxt's build pipeline
- ✅ **Nuxt context** - Access to all composables, auto-imports, and modules
- ✅ **Perfect fidelity** - Components look exactly like production
- ✅ **Devtools integration** - View stories in Nuxt Devtools panel
- ✅ **Auto-start** - Runs alongside `npm run dev`

---

## 📦 Current Setup (To Be Replaced)

### Files Created
```
.storybook/
├── main.ts              # Storybook config (Vue3-Vite framework)
├── preview.ts           # Global decorators + component stubs
└── preview.css          # Tailwind + NuxtUI CSS variables + theme

app/assets/css/
└── theme.css            # Shared theme (KEEP - used by both)

app/components/ui/
├── BackLink.stories.ts           # 4 variants
├── list/
│   ├── PageHeader.stories.ts     # 6 variants
│   ├── SkeletonCards.stories.ts  # 5 variants
│   ├── EmptyState.stories.ts     # 5 variants
│   └── ResultsCount.stories.ts   # 7 variants
```

### Package Dependencies (Current)
```json
{
  "@storybook/addon-essentials": "^8.6.14",
  "@storybook/addon-links": "^8.6.14",
  "@storybook/vue3": "^8.6.14",
  "@storybook/vue3-vite": "^8.6.14",
  "storybook": "^8.6.14"
}
```

### Scripts (Current)
```json
{
  "storybook": "storybook dev -p 6006 --no-open",
  "build-storybook": "storybook build"
}
```

---

## 🚀 Migration Plan

### Phase 1: Install @nuxtjs/storybook

```bash
# 1. Remove current Storybook packages
docker compose exec nuxt npm uninstall @storybook/addon-essentials @storybook/addon-links @storybook/vue3 @storybook/vue3-vite storybook

# 2. Initialize @nuxtjs/storybook (use Storybook 9.1.16)
docker compose exec nuxt npx storybook@9.1.16 init

# OR install manually:
docker compose exec nuxt npm install --save-dev @nuxtjs/storybook
```

### Phase 2: Configure Nuxt Module

**Update `nuxt.config.ts`:**
```typescript
export default defineNuxtConfig({
  modules: [
    '@nuxt/eslint',
    '@nuxt/ui',
    '@nuxt/test-utils',
    '@nuxt/image',
    '@nuxt/fonts',
    '@nuxtjs/storybook' // Add this
  ],

  storybook: {
    // Storybook configuration
    url: 'http://localhost:6006',
    port: 6006
  }
})
```

### Phase 3: Migrate Stories

**Story format stays mostly the same**, but remove component stubs:

**Before (with stubs):**
```typescript
import type { Meta, StoryObj } from '@storybook/vue3'
import BackLink from './BackLink.vue'

const NuxtLinkStub = { /* ... */ }
const UButtonStub = { /* ... */ }

const meta: Meta<typeof BackLink> = {
  title: 'UI/Navigation/BackLink',
  component: BackLink,
  render: (args) => ({
    components: { BackLink, NuxtLink: NuxtLinkStub, UButton: UButtonStub },
    // ...
  })
}
```

**After (with real components):**
```typescript
import type { Meta, StoryObj } from '@storybook/vue3'
import BackLink from './BackLink.vue'

const meta: Meta<typeof BackLink> = {
  title: 'UI/Navigation/BackLink',
  component: BackLink
  // No stubs needed - real NuxtUI components available!
}
```

### Phase 4: Clean Up

```bash
# Remove old Storybook config (after confirming migration works)
rm -rf .storybook/

# Keep theme.css (still used by main app)
# Keep all *.stories.ts files (just update them to remove stubs)
```

### Phase 5: Update Scripts

**package.json:**
```json
{
  "scripts": {
    "dev": "nuxt dev",  // Storybook auto-starts with this
    "storybook": "nuxt storybook dev",  // Explicit Storybook-only mode
    "build-storybook": "nuxt storybook build"
  }
}
```

---

## 🔄 Migration Steps (Detailed)

### Step 1: Backup Current Work
```bash
git add .
git commit -m "chore: Checkpoint before @nuxtjs/storybook migration"
```

### Step 2: Install Module
```bash
docker compose exec nuxt npx storybook@9.1.16 init
```

This will:
- Install `@nuxtjs/storybook` package
- Add module to `nuxt.config.ts`
- Create initial configuration

### Step 3: Test One Story
Update ONE story file (e.g., `PageHeader.stories.ts`) to remove stubs and test if it works with real components.

### Step 4: Verify Real NuxtUI Works
- Start dev server: `docker compose exec nuxt npm run dev`
- Open Storybook (should auto-open or go to http://localhost:6006)
- Check if PageHeader story renders with real components
- Verify NuxtUI styling is present

### Step 5: Migrate Remaining Stories
If Step 4 works, update the other 4 story files to remove stubs.

### Step 6: Remove Old Config
Once all stories work:
```bash
rm -rf .storybook/
git add .
git commit -m "refactor: Migrate to @nuxtjs/storybook module"
```

---

## 📝 Story Migration Checklist

For each story file, update:

- [ ] **Remove component stubs** from story file
- [ ] **Remove custom render function** (if it only provided stubs)
- [ ] **Test story renders** with real components
- [ ] **Verify styling** matches production
- [ ] **Check dark mode** toggle still works
- [ ] **Test interactive controls** in Storybook

Example migration for EmptyState.stories.ts:

**Delete these lines:**
```typescript
// Remove all stub definitions
const UCardStub = { /* ... */ }
const UIconStub = { /* ... */ }
const UButtonStub = { /* ... */ }

// Remove custom render if it only provides stubs
render: (args) => ({
  components: { UiListEmptyState, UCard: UCardStub, ... }
})
```

**Keep:**
```typescript
// Keep all story definitions
export const Default: Story = { args: { entityName: 'spells' } }
export const WithFilters: Story = { args: { ... } }
// etc.
```

---

## ⚠️ Potential Issues & Solutions

### Issue 1: Port Conflict
**Problem:** Port 6006 already in use
**Solution:**
```typescript
// nuxt.config.ts
storybook: {
  port: 6007  // Use different port
}
```

### Issue 2: Stories Not Auto-Detected
**Problem:** Stories don't appear in Storybook
**Solution:**
```typescript
// nuxt.config.ts
storybook: {
  stories: [
    './app/components/**/*.stories.@(js|ts|tsx)'
  ]
}
```

### Issue 3: NuxtUI Theme Not Applied
**Problem:** Components render but look unstyled
**Solution:** Check that `@nuxt/ui` module is loaded before `@nuxtjs/storybook` in `modules` array.

### Issue 4: Build Errors
**Problem:** `#imports` or `#build` not found
**Solution:** Ensure you're running Storybook through Nuxt (`nuxt dev` or `nuxt storybook dev`), not standalone Storybook CLI.

---

## 🎨 Expected Improvements

### Visual Fidelity
**Before (stubs):**
- Approximate styling with Tailwind classes
- No hover states, focus rings, or animations
- Colors might not match exactly
- No support for NuxtUI variants/sizes

**After (@nuxtjs/storybook):**
- Pixel-perfect match to production
- Full hover, focus, and active states
- Exact NuxtUI theme colors
- All variants, sizes, and props work correctly

### Developer Experience
**Before:**
- Manual Storybook restart: `docker compose exec nuxt npm run storybook`
- Separate from Nuxt dev server
- Stubs need manual updates when NuxtUI changes

**After:**
- Auto-starts with `npm run dev`
- Integrated into Nuxt Devtools
- Always in sync with NuxtUI (no stubs to maintain)

---

## 📚 Resources

- **@nuxtjs/storybook Docs:** https://storybook.nuxtjs.org/
- **LLM Context:** https://storybook.nuxtjs.org/llms.txt
- **Storybook 9 Changelog:** https://github.com/storybookjs/storybook/blob/main/CHANGELOG.md
- **NuxtUI Docs:** https://ui.nuxt.com/docs

---

## ✅ Success Criteria

Migration is complete when:
- [ ] `@nuxtjs/storybook` module installed and configured
- [ ] All 5 story files migrated (stubs removed)
- [ ] Storybook accessible at http://localhost:6006
- [ ] Real NuxtUI components render correctly
- [ ] Dark mode works
- [ ] All controls/interactions functional
- [ ] Old `.storybook/` directory removed
- [ ] Everything committed

---

## 💡 Decision: To Migrate or Not?

### Reasons TO Migrate
1. **Accuracy** - Get real component styling, not approximations
2. **Maintainability** - No stubs to keep in sync with NuxtUI updates
3. **Features** - Full access to Nuxt context (composables, auto-imports)
4. **Integration** - Storybook in Devtools, auto-start with dev server

### Reasons NOT TO Migrate (Yet)
1. **Working Solution** - Current stubs are functional for documentation
2. **Migration Effort** - Need to update 5 story files + config
3. **Risk** - Module might have bugs or compatibility issues
4. **Learning Curve** - Team needs to learn new module's quirks

### Recommendation
**Migrate when:**
- You need to add Tier 2+ stories (card components with complex styling)
- Visual fidelity becomes important (e.g., before launch)
- You're adding many more components to Storybook

**Stay with current approach if:**
- Current 5 stories meet your documentation needs
- You're focused on building features, not perfecting Storybook
- You want to wait for @nuxtjs/storybook to mature more

---

## 📦 Rollback Plan

If migration fails:
```bash
# 1. Revert to last commit
git reset --hard HEAD~1

# 2. Reinstall old Storybook packages
docker compose exec nuxt npm install --save-dev @storybook/vue3-vite@^8.6.14 storybook@^8.6.14

# 3. Restart Storybook
docker compose exec nuxt npm run storybook
```

---

**Status:** Documentation complete. Ready for migration when needed.
**Next Step:** Decide whether to migrate now or continue with current approach.
