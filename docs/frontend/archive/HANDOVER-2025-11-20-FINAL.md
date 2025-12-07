# Final Handover - Nuxt 4 Fresh Installation Complete
**Date:** 2025-11-20
**Status:** ✅ Clean Nuxt 4.2.1 installation with Docker running successfully
**Next Step:** Implement search feature

---

## 🎯 Current State

### ✅ What's Working
- **Docker containers running** on ports 3000 (Nuxt) and 8081 (Nginx)
- **Fresh Nuxt 4.2.1** installation with proper configuration
- **NuxtUI 4.2.0** installed and configured
- **Dev server** running without errors
- **TypeScript** strict mode enabled
- **ESLint** and test utils configured

### 📦 Installed Dependencies
```json
{
  "dependencies": {
    "@nuxt/eslint": "^1.10.0",
    "@nuxt/test-utils": "^3.20.1",
    "@nuxt/ui": "^4.2.0",
    "eslint": "^9.39.1",
    "nuxt": "^4.2.1",
    "typescript": "^5.9.3",
    "vue": "^3.5.24",
    "vue-router": "^4.6.3"
  }
}
```

### 🐳 Docker Status
```bash
# Containers running
- dnd-frontend-nuxt (port 3000)
- dnd-frontend-nginx (port 8081)

# Access URLs
http://localhost:3000  # Direct Nuxt
http://localhost:8081  # Via Nginx
```

---

## 🔑 Key Lessons Learned

### 1. **Nuxt 4 Directory Structure (CRITICAL)**

**All application code MUST be inside `app/` directory:**

```
frontend/
├── app/              # ⚠️ REQUIRED for Nuxt 4
│   ├── components/   # Vue components
│   ├── composables/  # Vue composables
│   ├── layouts/      # Page layouts
│   ├── pages/        # File-based routing
│   └── app.vue       # Root component
├── public/           # Static assets
├── types/            # TypeScript types
├── nuxt.config.ts    # Configuration
└── package.json      # Dependencies
```

**What stays at root:**
- Configuration files (nuxt.config.ts, tsconfig.json, etc.)
- package.json
- Docker files
- Documentation (docs/)
- Type definitions (types/)
- Tests (tests/)

### 2. **@vueuse/core Dependency Conflict**

**Problem:** NuxtUI 4.2.0 has version conflicts with @vueuse/core.

**Solutions (in order of preference):**
1. ✅ **Don't install @vueuse/core** - Use native JavaScript implementations
2. Use `package.json` overrides to force single version
3. Replace VueUse utilities with simple native code

**Example - Native Debounce:**
```typescript
let debounceTimeout: ReturnType<typeof setTimeout> | null = null
const debouncedSearch = (query: string) => {
  if (debounceTimeout) clearTimeout(debounceTimeout)
  debounceTimeout = setTimeout(async () => {
    // Your logic here
  }, 300)
}
```

### 3. **Docker Entrypoint File Required**

The `docker/node/entrypoint-dev.sh` file is required and must be executable:
```bash
#!/bin/sh
set -e

echo "🚀 Starting Nuxt development server..."

if [ ! -d "node_modules" ]; then
  echo "📦 Installing dependencies..."
  npm install
fi

echo "🔥 Starting npm run dev..."
npm run dev
```

### 4. **AI Assistant Context (llms.txt)**

**Nuxt provides LLM-friendly documentation:**
- Quick: `https://nuxt.com/llms.txt` (~5K tokens)
- Full: `https://nuxt.com/llms-full.txt` (1M+ tokens)

**Usage:**
- Ask Claude Code to fetch it when needed for best practices
- In Cursor/Windsurf: Type `@https://nuxt.com/llms.txt`
- Ensures AI guidance aligns with official Nuxt 4 patterns

---

## 🚀 Next Steps: Implement Search Feature

### Phase 1: Setup (30 minutes)

1. **Create type definitions** (`types/search.ts`)
2. **Add runtime config** to `nuxt.config.ts`
3. **Create `.env` file** with API URLs

### Phase 2: Backend Integration (45 minutes)

1. **Create `app/composables/useSearch.ts`**
   - Use `$fetch` for API calls
   - Handle loading/error states
   - Return typed SearchResult

2. **Create type definitions** for:
   - SearchResult
   - Spell, Item, Race, Class, Background, Feat

### Phase 3: UI Components (1 hour)

1. **Create `app/components/SearchInput.vue`**
   - Native debounce (300ms)
   - Dropdown autocomplete
   - Navigate to /search on Enter

2. **Create `app/components/SearchResultCard.vue`**
   - Entity-agnostic display
   - Type-specific badges
   - Metadata rendering

3. **Create `app/pages/index.vue`** (Homepage)
   - Hero section with search
   - Quick link cards
   - Stats footer

4. **Create `app/pages/search.vue`** (Results page)
   - URL-based search (?q=query)
   - Entity type filters
   - Grouped results

5. **Create `app/layouts/default.vue`**
   - Header with search
   - Dark mode toggle
   - Footer

### Phase 4: Testing & Verification

1. **Manual browser testing**
   - Homepage loads
   - Search dropdown works
   - Full results page works
   - Dark mode toggles

2. **Backend API verification**
   ```bash
   # Test search endpoint
   curl "http://localhost:8080/api/v1/search?q=fire"
   ```

---

## 📝 Implementation Checklist

### Before Starting
- [ ] Verify backend API is running (`http://localhost:8080/api/v1`)
- [ ] Verify frontend Docker containers are up
- [ ] Verify Nuxt dev server is accessible (`http://localhost:3000`)

### Configuration Files
- [ ] Add runtime config to `nuxt.config.ts`
- [ ] Create `.env` file with API URLs
- [ ] Create `types/search.ts` with all interfaces

### Composables
- [ ] Create `app/composables/useSearch.ts`
- [ ] Test composable with manual API calls

### Components
- [ ] Create `app/components/SearchInput.vue` (no @vueuse/core!)
- [ ] Create `app/components/SearchResultCard.vue`

### Pages
- [ ] Create `app/pages/index.vue` (homepage)
- [ ] Create `app/pages/search.vue` (results)

### Layouts
- [ ] Create `app/layouts/default.vue`

### Testing
- [ ] Test homepage loads
- [ ] Test instant search dropdown
- [ ] Test full search results page
- [ ] Test dark mode
- [ ] Test in browser console (no errors)

---

## 🐛 Common Issues & Solutions

### Issue: Pages not loading
**Cause:** Files not in `app/` directory
**Fix:** Ensure all pages are in `app/pages/`

### Issue: Components not found
**Cause:** Components not in `app/components/`
**Fix:** Move components into `app/` structure

### Issue: @vueuse/core errors
**Cause:** Version conflict with NuxtUI
**Fix:** Remove @vueuse/core, use native implementations

### Issue: Docker container not starting
**Cause:** Missing or incorrect entrypoint script
**Fix:** Check `docker/node/entrypoint-dev.sh` exists and is executable

### Issue: 500 errors in browser
**Cause:** Module dependency issues
**Fix:**
```bash
docker compose exec nuxt rm -rf .nuxt .output
docker compose restart nuxt
```

---

## 📚 Reference Documentation

### Official Nuxt Docs
- **Nuxt 4.x:** https://nuxt.com/docs/4.x/getting-started/introduction
- **NuxtUI:** https://ui.nuxt.com/docs/getting-started
- **llms.txt:** https://nuxt.com/llms.txt

### Backend API
- **API Docs:** http://localhost:8080/docs/api
- **OpenAPI Spec:** http://localhost:8080/docs/api.json
- **Search Endpoint:** http://localhost:8080/api/v1/search

### Project Documentation
- **Backend CLAUDE.md:** `../importer/CLAUDE.md`
- **Frontend CLAUDE.md:** `./CLAUDE.md`
- **Fresh Install Guide:** `./docs/HANDOVER-2025-11-20-FRESH-INSTALL-GUIDE.md`

---

## ⚡ Quick Start Commands

### Docker
```bash
# Start containers
docker compose up -d

# View logs
docker compose logs -f nuxt

# Restart containers
docker compose restart

# Stop containers
docker compose down

# Execute commands in container
docker compose exec nuxt npm install
docker compose exec nuxt npm run dev
```

### Development
```bash
# Install dependencies
npm install

# Start dev server
npm run dev

# Run tests
npm run test

# Lint code
npm run lint

# Type check
npm run typecheck
```

---

## 🎯 Success Criteria

The search feature will be complete when:

- [ ] ✅ Homepage loads at `http://localhost:3000`
- [ ] ✅ Search input is visible in header
- [ ] ✅ Typing shows instant results after 300ms
- [ ] ✅ Clicking result navigates to detail page (will 404 until built)
- [ ] ✅ Pressing Enter navigates to `/search?q=query`
- [ ] ✅ Search results page shows grouped results
- [ ] ✅ Entity type filters work
- [ ] ✅ Dark mode toggle works
- [ ] ✅ No console errors
- [ ] ✅ Responsive on mobile/desktop

---

## 💡 Recommendations

1. **Start simple** - Get basic search working before adding features
2. **Use llms.txt** - Reference official docs when unsure
3. **Test incrementally** - Verify each component as you build
4. **Avoid @vueuse/core** - Use native implementations to avoid conflicts
5. **Follow Nuxt patterns** - Auto-imports, file-based routing, composables
6. **Check backend first** - Verify API returns data before debugging frontend

---

**Current Environment:**
- Node: 22-alpine (in Docker)
- Nuxt: 4.2.1
- NuxtUI: 4.2.0
- TypeScript: 5.9.3
- Vue: 3.5.24

**Ready to start implementing the search feature!** 🚀
