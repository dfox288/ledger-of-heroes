# Design: Reference Pages Batch 2 (6 New Reference Types)

**Date:** 2025-11-21
**Status:** Approved
**Type:** Feature Addition

---

## Overview

Add 6 new reference pages to the D&D 5e Compendium frontend, following the established pattern from the first batch of reference pages (Sources, Languages, Sizes, Damage Types).

**New Reference Types:**
1. Ability Scores
2. Spell Schools
3. Item Types
4. Proficiency Types
5. Skills
6. Conditions

---

## Goals

- Provide comprehensive D&D 5e game mechanics reference
- Maintain visual consistency with existing reference pages
- Follow TDD mandate from CLAUDE.md
- Create clean, testable components
- Ensure all pages work in Docker environment

---

## Success Criteria

- [ ] All 6 pages render with HTTP 200 status
- [ ] Search functionality works on all pages
- [ ] Cards display all API fields appropriately
- [ ] Navigation dropdown includes new items (alphabetically sorted)
- [ ] Tests written FIRST for all components (TDD)
- [ ] All tests passing
- [ ] Each entity committed separately after completion
- [ ] Manual browser verification (light + dark mode)

---

## Architecture

### Component Structure

Each reference type gets:

1. **Card Component:** `app/components/{entity}/{Entity}Card.vue`
2. **List Page:** `app/pages/{entity}/index.vue`
3. **API Proxy:** `server/api/{entity}/index.get.ts`
4. **Tests:** `tests/components/{entity}/{Entity}Card.test.ts`
5. **Navigation Entry:** Updated in navigation component

### API Endpoints (Backend)

All endpoints verified and working:

- `GET /v1/ability-scores` → `{id, code, name}`
- `GET /v1/spell-schools` → `{id, code, name, description?}`
- `GET /v1/item-types` → `{id, code, name, description}`
- `GET /v1/proficiency-types` → `{id, name, category, subcategory?}`
- `GET /v1/skills` → `{id, name, ability_score: {id, code, name}}`
- `GET /v1/conditions` → `{id, name, slug, description}`

---

## Component Design Specifications

### 1. Ability Scores (`/ability-scores`)

**API Data:** `{id, code, name}`

**Card Layout:**
```
┌─────────────────────┐
│ [STR] Badge (lg)    │
│                     │
│ Strength            │
│                     │
│ [Ability Score] xs  │
└─────────────────────┘
```

**Features:**
- Code badge (large, neutral color)
- Full name as title
- Category badge at bottom
- Neutral gray theme

**TypeScript Interface:**
```typescript
interface AbilityScore {
  id: number
  code: string
  name: string
}
```

---

### 2. Spell Schools (`/spell-schools`)

**API Data:** `{id, code, name, description?}`

**Card Layout:**
```
┌─────────────────────┐
│ [A] Badge (lg)      │
│                     │
│ Abjuration          │
│                     │
│ Description text... │
│ (2 lines max)       │
│                     │
│ [Spell School] xs   │
└─────────────────────┘
```

**Features:**
- Code badge (large, neutral)
- School name as title
- Optional description (truncated to 2 lines with `line-clamp-2`)
- Category badge at bottom
- Neutral gray theme

**TypeScript Interface:**
```typescript
interface SpellSchool {
  id: number
  code: string
  name: string
  description?: string
}
```

---

### 3. Item Types (`/item-types`)

**API Data:** `{id, code, name, description}`

**Card Layout:**
```
┌─────────────────────┐
│ [A] Badge (lg)      │
│                     │
│ Ammunition          │
│                     │
│ Arrows, bolts...    │
│ (2 lines max)       │
│                     │
│ [Item Type] xs      │
└─────────────────────┘
```

**Features:**
- Code badge (large, neutral)
- Type name as title
- Description (truncated to 2 lines with `line-clamp-2`)
- Category badge at bottom
- Neutral gray theme

**TypeScript Interface:**
```typescript
interface ItemType {
  id: number
  code: string
  name: string
  description: string
}
```

---

### 4. Proficiency Types (`/proficiency-types`)

**API Data:** `{id, name, category, subcategory?}`

**Card Layout:**
```
┌─────────────────────┐
│ Light Armor         │
│                     │
│ [armor] Badge       │
│ [subcategory]       │
│ (if present)        │
│                     │
│ [Proficiency Type]  │
└─────────────────────┘
```

**Features:**
- Proficiency name as title
- Category badge (neutral)
- Optional subcategory badge (neutral, soft variant)
- Category badge at bottom
- Neutral gray theme

**TypeScript Interface:**
```typescript
interface ProficiencyType {
  id: number
  name: string
  category: string
  subcategory?: string
}
```

---

### 5. Skills (`/skills`)

**API Data:** `{id, name, ability_score: {id, code, name}}`

**Card Layout:**
```
┌─────────────────────┐
│ Acrobatics          │
│                     │
│ [DEX] Badge (blue)  │
│ Dexterity           │
│                     │
│ [Skill] xs          │
└─────────────────────┘
```

**Features:**
- Skill name as title
- Ability score code badge (info/blue color)
- Ability score full name (small text)
- Category badge at bottom
- Neutral gray theme (card), info color (ability badge)

**TypeScript Interface:**
```typescript
interface Skill {
  id: number
  name: string
  ability_score: {
    id: number
    code: string
    name: string
  }
}
```

**Nested Data Handling:**
- Always use optional chaining: `skill.ability_score?.code`
- Show fallback if ability_score is missing

---

### 6. Conditions (`/conditions`)

**API Data:** `{id, name, slug, description}`

**Card Layout:**
```
┌─────────────────────┐
│ Blinded             │
│                     │
│ A blinded creature  │
│ can't see and auto- │
│ matically fails...  │
│ (3 lines max)       │
│                     │
│ [Condition] xs      │
└─────────────────────┘
```

**Features:**
- Condition name as title
- Description (truncated to 3 lines with `line-clamp-3`)
- Category badge at bottom
- Neutral gray theme

**Note:** 3 lines instead of 2 because condition descriptions contain important game rules.

**TypeScript Interface:**
```typescript
interface Condition {
  id: number
  name: string
  slug: string
  description: string
}
```

---

## Page Structure (Shared Pattern)

All 6 pages follow this structure:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const { apiFetch } = useApi()
const searchQuery = ref('')

const queryParams = computed(() => {
  const params: Record<string, any> = {}
  if (searchQuery.value.trim()) {
    params.q = searchQuery.value.trim()
  }
  return params
})

const { data: response, pending: loading, error, refresh } = await useAsyncData(
  'entity-list',
  async () => {
    const response = await apiFetch('/entity-endpoint', {
      query: queryParams.value
    })
    return response
  },
  { watch: [queryParams] }
)

const items = computed(() => response.value?.data || [])
const totalResults = computed(() => items.value.length)

useSeoMeta({
  title: 'Entity Name - D&D 5e Compendium',
  description: 'Browse all D&D 5e entity items.',
})

useHead({
  title: 'Entity Name - D&D 5e Compendium',
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-7xl">
    <UiListPageHeader
      title="Entity Name"
      :total="totalResults"
      description="Browse D&D 5e entity items"
      :loading="loading"
    />

    <div class="mb-6">
      <UInput
        v-model="searchQuery"
        icon="i-heroicons-magnifying-glass"
        size="lg"
        placeholder="Search..."
        :ui="{ icon: { trailing: { pointer: '' } } }"
      >
        <template v-if="searchQuery" #trailing>
          <UButton
            color="gray"
            variant="link"
            icon="i-heroicons-x-mark-20-solid"
            :padded="false"
            @click="searchQuery = ''"
          />
        </template>
      </UInput>
    </div>

    <UiListSkeletonCards v-if="loading" />

    <UiListErrorState
      v-else-if="error"
      :error="error"
      entity-name="Entity Name"
      @retry="refresh"
    />

    <UiListEmptyState
      v-else-if="items.length === 0"
      entity-name="entity name"
      :has-filters="!!searchQuery"
      @clear-filters="searchQuery = ''"
    />

    <div v-else>
      <UiListResultsCount
        :from="1"
        :to="totalResults"
        :total="totalResults"
        entity-name="entity name"
      />

      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 mb-8">
        <EntityCard
          v-for="item in items"
          :key="item.id"
          :entity="item"
        />
      </div>
    </div>

    <UiBackLink />
    <JsonDebugPanel :data="{ items, total: totalResults }" title="Entity Data" />
  </div>
</template>
```

**Key Features:**
- Real-time search filtering
- Loading/error/empty states
- No pagination (small datasets)
- Responsive grid (1/2/3 columns)
- JSON debug panel
- Back link navigation

---

## API Proxy Structure

All 6 proxies follow this simple pattern:

```typescript
// server/api/{entity}/index.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const query = getQuery(event)
  const data = await $fetch(`${config.apiBaseServer}/{entity-endpoint}`, { query })
  return data
})
```

**Purpose:**
- Forwards requests to Laravel backend
- Uses SSR-safe URL (`apiBaseServer` = `host.docker.internal:8080`)
- Passes search query parameters through
- Returns raw API response

---

## Navigation Updates

Add to existing "Reference" dropdown menu, maintaining alphabetical order:

**Current Navigation:**
```
Reference ▼
  - Damage Types
  - Languages
  - Sizes
  - Sources
```

**Updated Navigation:**
```
Reference ▼
  - Ability Scores (NEW)
  - Conditions (NEW)
  - Damage Types
  - Item Types (NEW)
  - Languages
  - Proficiency Types (NEW)
  - Sizes
  - Skills (NEW)
  - Spell Schools (NEW)
  - Sources
```

**File to Update:** Navigation component (likely `app/components/ui/Navigation.vue` or layout)

---

## Testing Strategy (TDD)

### Mandatory TDD Process

For each entity, follow this exact sequence:

#### 1. RED Phase - Write Failing Tests First

```typescript
// tests/components/{entity}/{Entity}Card.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import EntityCard from '~/components/{entity}/EntityCard.vue'

describe('EntityCard', () => {
  it('displays entity name', async () => {
    const wrapper = await mountSuspended(EntityCard, {
      props: { entity: { id: 1, name: 'Test' } }
    })
    expect(wrapper.text()).toContain('Test')
  })

  it('displays code badge when present', async () => { ... })
  it('displays category badge', async () => { ... })
  it('handles optional fields gracefully', async () => { ... })
  it('truncates long descriptions', async () => { ... })
})
```

**Run tests - verify they FAIL:**
```bash
docker compose exec nuxt npm test -- EntityCard.test.ts
# Expected: All tests fail (component doesn't exist yet)
```

#### 2. GREEN Phase - Implement Minimal Code to Pass

Create the card component with just enough code to make tests pass.

**Run tests - verify they PASS:**
```bash
docker compose exec nuxt npm test -- EntityCard.test.ts
# Expected: All tests pass
```

#### 3. Create Page + API Proxy

No tests needed for these (simple proxy + standard page pattern).

#### 4. Manual Browser Verification

```bash
# Start dev server (if not running)
docker compose exec nuxt npm run dev

# Test in browser
curl http://localhost:3000/{entity-slug}
# Expected: HTTP 200

# Verify search works
# Verify light/dark mode
# Verify responsive grid
```

#### 5. Commit

```bash
git add app/components/{entity}/
git add app/pages/{entity}/
git add server/api/{entity}/
git add tests/components/{entity}/

git commit -m "feat: Add {Entity} reference page with tests

- Created {Entity}Card component (X tests, all passing)
- Added /{entity-slug} page with search
- Created API proxy endpoint
- Verified in browser (HTTP 200)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

#### 6. Repeat for Next Entity

### Minimum Test Coverage Per Component

Each card component must have at least:

- ✅ Name/title display test
- ✅ Badge display test
- ✅ Optional field handling test (if applicable)
- ✅ Text truncation test (if descriptions present)
- ✅ Nested data test (if applicable, e.g., Skills)

**Target:** 5-8 tests per component

---

## Implementation Order

**Simplest → Most Complex:**

1. **Ability Scores** - Simplest (code + name only)
2. **Spell Schools** - Add optional description
3. **Item Types** - Similar to spell schools
4. **Proficiency Types** - Add category + optional subcategory
5. **Skills** - Add nested ability_score object
6. **Conditions** - Longest descriptions (3-line truncation)

**Rationale:** Learn patterns on simple components, apply to complex ones.

---

## Error Handling

All components must gracefully handle:

- ✅ Missing optional fields (`description`, `subcategory`)
- ✅ Empty arrays (no search results)
- ✅ API errors (via `UiListErrorState`)
- ✅ Loading states (via `UiListSkeletonCards`)
- ✅ Null/undefined nested data (e.g., `ability_score?.code`)

**Implementation:**
- Use optional chaining (`?.`) for all nested/optional properties
- Provide default values where needed
- Use `v-if` directives to conditionally render optional sections

---

## Design System Compliance

### Colors (NuxtUI v4)

- **Card Theme:** Neutral gray (all reference pages)
- **Code Badges:** Neutral solid (large size)
- **Category Badges:** Neutral soft (extra-small size)
- **Ability Score Badge (Skills only):** Info/blue (highlights relationship)
- **Subcategory Badge:** Neutral soft (distinguishes from category)

### Typography

- **Card Title:** `text-xl font-semibold`
- **Code Badge:** `size="lg"`
- **Category Badge:** `size="xs"`
- **Description Text:** `text-sm text-gray-600 dark:text-gray-400`
- **Nested Data (Skills):** `text-sm`

### Spacing

- **Card Content:** `space-y-3`
- **Grid Gap:** `gap-4`
- **Section Margins:** `mb-6`, `mb-8`

### Text Truncation

- **2 Lines:** `line-clamp-2` (Spell Schools, Item Types)
- **3 Lines:** `line-clamp-3` (Conditions)

**CSS Class:**
```html
<p class="text-sm text-gray-600 dark:text-gray-400 line-clamp-2">
  {{ description }}
</p>
```

---

## File Checklist

For each entity, create these files:

**Component Files:**
- [ ] `app/components/{entity}/{Entity}Card.vue`
- [ ] `app/pages/{entity}/index.vue`
- [ ] `server/api/{entity}/index.get.ts`

**Test Files:**
- [ ] `tests/components/{entity}/{Entity}Card.test.ts`

**Updated Files:**
- [ ] Navigation component (add to dropdown)

**Total:** 25 new files (4 files × 6 entities + 1 navigation update)

---

## Risks & Mitigations

### Risk 1: Component Auto-Import Issues

**Problem:** Nested components need folder prefix (e.g., `<AbilityScoreAbilityScoreCard>`)

**Mitigation:**
- Components in `components/{entity}/` folders auto-import with prefix
- Use `<AbilityScoreCard>` not `<Card>`
- Test early, verify in browser

### Risk 2: Optional Fields Causing Errors

**Problem:** Missing `description` or `subcategory` could break rendering

**Mitigation:**
- Use optional chaining (`?.`) everywhere
- Use `v-if` directives for optional sections
- Add tests for missing fields

### Risk 3: API Endpoint Mismatch

**Problem:** Frontend expects `/ability-scores` but backend uses different path

**Mitigation:**
- All endpoints already verified with curl tests
- API proxy handles path mapping
- Test in browser immediately after creating proxy

### Risk 4: Test Not Following TDD

**Problem:** Implementing component before writing tests

**Mitigation:**
- MUST write tests first (RED phase)
- MUST verify tests fail before implementing
- MUST commit tests with implementation
- Follow checklist strictly

---

## Success Metrics

### Functional Metrics

- ✅ All 6 pages return HTTP 200
- ✅ Search filters results correctly on all pages
- ✅ All API data fields displayed
- ✅ No console errors
- ✅ No hydration errors

### Quality Metrics

- ✅ 30-48 new tests (5-8 per component × 6)
- ✅ 100% test pass rate
- ✅ All components use TypeScript interfaces
- ✅ All components handle optional data
- ✅ TDD process followed for all components

### User Experience Metrics

- ✅ Cards visually consistent with existing reference pages
- ✅ Light/dark mode works correctly
- ✅ Responsive at 375px, 768px, 1440px
- ✅ Keyboard navigation works
- ✅ Loading/error/empty states clear and helpful

---

## Completion Checklist

- [ ] Ability Scores (tests + component + page + proxy + commit)
- [ ] Spell Schools (tests + component + page + proxy + commit)
- [ ] Item Types (tests + component + page + proxy + commit)
- [ ] Proficiency Types (tests + component + page + proxy + commit)
- [ ] Skills (tests + component + page + proxy + commit)
- [ ] Conditions (tests + component + page + proxy + commit)
- [ ] Navigation updated with all 6 new items
- [ ] All pages verified in browser (HTTP 200)
- [ ] All tests passing
- [ ] Documentation updated (CURRENT_STATUS.md, handover doc)
- [ ] Final commit with navigation updates

**Total Commits:** 7 (6 entities + 1 navigation/docs update)

---

## Next Steps After Completion

1. Update `docs/CURRENT_STATUS.md` to reflect 10 reference pages
2. Create handover document in `docs/HANDOVER-2025-11-21-REFERENCE-PAGES-BATCH-2.md`
3. Consider adding detail pages for reference items (future enhancement)
4. Consider adding cross-references (e.g., link from Spell to Spell School)

---

**Design Status:** ✅ APPROVED
**Ready for Implementation:** YES
**Estimated Time:** 2-3 hours
**Commits:** 7 total
