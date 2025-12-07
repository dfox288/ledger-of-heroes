# Handover: Storybook Migration Attempt - Blocked by Compatibility Issues

**Date:** 2025-11-23
**Status:** ❌ BLOCKED - Migration Not Feasible
**Attempted:** Migration from standalone Storybook 8.6.14 to @nuxtjs/storybook module
**Result:** Both Storybook 8.x and 9.x are incompatible with Nuxt 4 + Vite 7

---

## 🚨 Critical Finding

**Neither the current Storybook 8.6.14 setup NOR the @nuxtjs/storybook@9.0.1 module are compatible with Nuxt 4.2.1 + Vite 7.2.4.**

The current Storybook 8.6.14 installation only works because it was installed with `--legacy-peer-deps`, which bypasses npm's peer dependency resolution and creates a potentially unstable setup.

---

## 📋 What Was Attempted

### Goal
Migrate from standalone Storybook 8.6.14 to `@nuxtjs/storybook` module for:
- Real NuxtUI components (no stubs needed)
- Full Nuxt context (composables, auto-imports)
- Automatic integration with `nuxt dev`
- Perfect visual fidelity

### Steps Taken
1. ✅ Created backup commit (df4e71a)
2. ✅ Researched @nuxtjs/storybook documentation
3. ✅ Uninstalled Storybook 8.6.14 packages
4. ❌ Attempted to install @nuxtjs/storybook@9.0.1 + storybook@~9.0.5
5. ❌ Hit peer dependency conflict with Vite 7
6. ❌ Tried storybook-nuxt init command
7. ❌ Hit same Vite compatibility issues
8. ✅ Rolled back to working state

---

## ⚠️ Compatibility Matrix

| Package | Version Required | Nuxt 4 Provides | Status |
|---------|------------------|-----------------|--------|
| **Storybook 8.6.14** | Vite 4-6 | Vite 7.2.4 | ❌ Incompatible |
| **@nuxtjs/storybook@9.0.1** | Vite 5-6 | Vite 7.2.4 | ❌ Incompatible |
| **@storybook-vue/nuxt@9.0.1** | Vite 5-7 | Vite 7.2.4 | ⚠️ Claims support, but nested deps fail |

### The Problem

```
Nuxt 4.2.1 → requires Vite 7.2.4 (peer dependency)
  ├─ @nuxt/devtools@3.1.0 → requires vite@>=6.0
  ├─ @nuxt/vite-builder → requires vite@^7.2.1
  └─ 15+ other modules → require Vite 7.x

Storybook 8.6.14 → requires Vite 4.0-6.0 (peer dependency)
  ├─ @storybook/builder-vite@8.6.14 → requires vite@^4.0.0 || ^5.0.0 || ^6.0.0
  └─ @storybook/vue3-vite@8.6.14 → requires vite@^4.0.0 || ^5.0.0 || ^6.0.0

@nuxtjs/storybook@9.0.1 → requires Vite 5.0-7.0 (peer dependency)
  └─ @storybook-vue/nuxt@9.0.1 → requires vite@^5.2.0 || ^6.0.0 || ^7.0.0
      ├─ @storybook/builder-vite@9.1.2 → requires vite@^9.1.2 (ERROR!)
      └─ @storybook/vue3-vite@9.1.2 → requires vite@^9.1.2 (ERROR!)
```

**Result:** npm cannot resolve these conflicts without `--force` or `--legacy-peer-deps`, which creates unstable installations.

---

## 🔍 Detailed Findings

### Issue 1: Storybook 8 + Vite 7 Incompatibility

**Error:**
```
npm error Could not resolve dependency:
npm error peer vite@"^4.0.0 || ^5.0.0 || ^6.0.0" from @storybook/builder-vite@8.6.14
npm error Found: vite@7.2.4
```

**Cause:** Storybook 8.6.14 was released before Vite 7 and explicitly requires Vite 6 or earlier.

**Current Workaround:** Installed with `--legacy-peer-deps` (masks the problem, doesn't fix it).

### Issue 2: @nuxtjs/storybook + Nested Dependencies

**Error:**
```
npm error peer storybook@"~9.0.5" from @nuxtjs/storybook@9.0.1
npm error Found: storybook@9.1.16
```

**Cause:** The `@nuxtjs/storybook@9.0.1` package depends on `@storybook-vue/nuxt@9.0.1`, which bundles Storybook 9.1.2 dependencies that require newer Vite versions than what's actually compatible.

### Issue 3: Version Mismatch in Dependency Tree

@storybook-vue/nuxt@9.0.1 claims to support Vite 7, but its nested dependencies don't:
- @storybook/builder-vite@9.1.2 → requires storybook@^9.1.2
- But @nuxtjs/storybook@9.0.1 → requires storybook@~9.0.5

This creates a circular dependency conflict.

---

## 📊 Attempted Solutions

### Solution 1: Use @nuxtjs/storybook@9.0.1
**Command:**
```bash
npm install -D @nuxtjs/storybook@^9.0.1 storybook@~9.0.5
```

**Result:** ❌ Failed
```
npm error peer storybook@"^9.1.2" from @storybook/builder-vite@9.1.2
```

### Solution 2: Use storybook-nuxt init
**Command:**
```bash
npx storybook-nuxt init
```

**Result:** ❌ Failed - Tried to downgrade to incompatible Vite 5 versions

### Solution 3: Force Installation
**Command:**
```bash
npm install -D @nuxtjs/storybook --force
```

**Result:** ⚠️ Not attempted - Would create unstable installation, potentially breaking builds

---

## 🎯 Recommendations

### Short Term (Now - 3 months)
**✅ Keep current Storybook 8.6.14 standalone setup**

**Rationale:**
- It works (with `--legacy-peer-deps`)
- 5 working stories with comprehensive variants
- Component stubs are functional for documentation
- Visual approximation is "good enough" for current needs

**Maintenance:**
1. Always use `npm install --legacy-peer-deps` when installing dependencies
2. Document this requirement in package.json scripts
3. Do NOT upgrade Storybook until Vite 7 support is released

**To maintain:**
```json
{
  "scripts": {
    "install": "npm install --legacy-peer-deps",
    "storybook": "storybook dev -p 6006 --no-open"
  }
}
```

### Medium Term (3-6 months)
**⏳ Monitor Storybook releases for Vite 7 support**

**Watch for:**
1. **Storybook 9.1.x or 9.2.x** with explicit Vite 7 support
2. **@nuxtjs/storybook@10.x** that resolves dependency issues
3. Community forks like `@tbanys/storybook-vue-nuxt-v4`

**Resources to monitor:**
- [Storybook Release Notes](https://github.com/storybookjs/storybook/releases)
- [GitHub Issue #31608](https://github.com/storybookjs/storybook/issues/31608)
- [@nuxtjs/storybook Repository](https://github.com/nuxt-modules/storybook)

### Long Term (6+ months)
**🔄 Migrate to @nuxtjs/storybook when compatible**

**Prerequisites:**
- Storybook officially supports Vite 7.x
- @nuxtjs/storybook module is updated for Nuxt 4
- All peer dependencies resolve without `--force` or `--legacy-peer-deps`

**When ready, follow:** `docs/HANDOVER-2025-11-23-STORYBOOK-NUXT-MODULE.md`

---

## 💡 Alternative Approaches (If Visual Fidelity Becomes Critical)

### Option 1: Improve Component Stubs
**Effort:** Low | **Benefit:** Medium

Enhance existing stubs in `.storybook/preview.ts` to more closely match NuxtUI components:
- Extract exact styles from NuxtUI source
- Use NuxtUI's CSS variables
- Improve hover states, transitions, and focus rings

**Files to update:**
- `.storybook/preview.ts` (global stubs)
- `.storybook/preview.css` (theme variables)

### Option 2: Use Histoire (Storybook Alternative)
**Effort:** High | **Benefit:** High

[Histoire](https://histoire.dev/) is a Vite-native story tool that might have better compatibility:
- Native Vite plugin (no framework adapter needed)
- Lighter weight than Storybook
- Better Vue 3 / Nuxt integration

**To explore:**
```bash
npm install -D histoire @histoire/plugin-vue @histoire/plugin-nuxt
```

### Option 3: Custom Documentation Site
**Effort:** Very High | **Benefit:** Maximum Control

Build a custom Nuxt page at `/docs/components` that renders components with real NuxtUI:
- Full Nuxt context (composables, auto-imports)
- Real styling (no stubs)
- Live component playground using Nuxt pages

**Effort:** 8-16 hours | **Maintenance:** Low

---

## 📦 Current State After Investigation

### Files Modified
- ❌ None (rolled back via `git reset --hard df4e71a`)

### Dependencies
- ✅ Storybook 8.6.14 restored
- ✅ All packages working with `--legacy-peer-deps`

### Git Commits
- df4e71a - Checkpoint before @nuxtjs/storybook migration (backup)
- Next commit will document findings

---

## 🧪 Verification Steps

To confirm current setup still works:

```bash
# 1. Install dependencies
docker compose exec nuxt npm install --legacy-peer-deps

# 2. Start Storybook
docker compose exec nuxt npm run storybook

# 3. Access at http://localhost:6006
# 4. Verify all 5 story files render correctly
# 5. Test dark mode toggle
```

**Expected:** All stories render with component stubs, dark mode works.

---

## 📚 Related Documentation

- **Original Migration Plan:** `docs/HANDOVER-2025-11-23-STORYBOOK-NUXT-MODULE.md`
- **Storybook Setup:** `.storybook/main.ts`, `.storybook/preview.ts`
- **Component Stories:** `app/components/ui/**/*.stories.ts`
- **Current Status:** `docs/CURRENT_STATUS.md`

---

## 🔗 External Resources

**Storybook + Nuxt Compatibility:**
- [Storybook Issue #31608](https://github.com/storybookjs/storybook/issues/31608) - Storybook 9 Nuxt compatibility
- [@nuxtjs/storybook on npm](https://www.npmjs.com/package/@nuxtjs/storybook)
- [Storybook Nuxt Docs](https://storybook.nuxtjs.org/)

**Alternative: Nuxt 4 Compatible Fork:**
- [@tbanys/storybook-vue-nuxt-v4](https://www.npmjs.com/package/@tbanys/storybook-vue-nuxt-v4) - Community fork

**Vite Compatibility:**
- [Vite 7 Release Notes](https://vitejs.dev/blog/announcing-vite7)
- [Nuxt 4 Vite Requirements](https://nuxt.com/docs/4.x)

---

## ✅ Success Criteria (For Future Migration)

Migration will be considered ready when:
- [ ] Storybook officially supports Vite 7.x
- [ ] @nuxtjs/storybook installs without `--force` or `--legacy-peer-deps`
- [ ] npm audit shows no peer dependency warnings for Storybook packages
- [ ] All existing stories render with real NuxtUI components
- [ ] Dark mode toggle works
- [ ] Storybook integrates with `nuxt dev` command

---

**Status:** Current setup is stable and functional. Migration postponed until ecosystem compatibility improves.

**Next Agent:** Continue using standalone Storybook 8.6.14 with component stubs. Focus on building features, not perfecting Storybook integration.
