# Entity Color Rollout Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Roll out entity-specific colors across all 18 card components, search components, and update all tests to use semantic entity colors instead of generic colors.

**Architecture:** Direct template updates replacing hardcoded generic colors (`primary`, `info`, `warning`) with semantic entity colors (`spell`, `item`, `race`). No composables needed - NuxtUI resolves entity colors via the three-layer config system (nuxt.config.ts → app.config.ts → main.css).

**Tech Stack:** Vue 3, NuxtUI 4, Vitest, @nuxt/test-utils

---

## Phase A: Main Entity Cards (7 components)

### Task 1: SpellCard Component

**Files:**
- Modify: `app/components/spell/SpellCard.vue`
- Test: `tests/components/spell/SpellCard.test.ts` (if exists)

**Step 1: Read SpellCard component**

Read: `app/components/spell/SpellCard.vue`
Purpose: Identify all color usages

**Step 2: Update SpellCard template**

Replace generic colors with entity color:
- Find all `color="primary"`, `color="info"`, `color="warning"`, `color="error"`
- Replace with `color="spell"`
- Remove any color mapping functions like `getLevelColor()`, `getSchoolColor()` from script section
- Keep spell school colors dynamic if they provide sub-categorization value

Expected changes:
```vue
<!-- BEFORE -->
<UBadge :color="getLevelColor(spell.level)">{{ levelText }}</UBadge>

<!-- AFTER -->
<UBadge color="spell">{{ levelText }}</UBadge>
```

**Step 3: Update border classes (if present)**

Replace generic borders with entity-colored borders:
```vue
<!-- BEFORE -->
<UCard class="border border-gray-200 dark:border-gray-700">

<!-- AFTER -->
<UCard class="border-2 border-spell-300 dark:border-spell-700 hover:border-spell-500">
```

**Step 4: Check if tests exist**

Run: `ls tests/components/spell/SpellCard.test.ts`
- If exists → Continue to Step 5
- If not exists → Skip to Step 7

**Step 5: Update SpellCard tests (if they exist)**

Read: `tests/components/spell/SpellCard.test.ts`
Update color assertions:
```typescript
// BEFORE
expect(badge.attributes('color')).toBe('primary')

// AFTER
expect(badge.attributes('color')).toBe('spell')
```

**Step 6: Run tests**

Run: `docker compose exec nuxt npm run test -- SpellCard.test.ts`
Expected: All tests pass

**Step 7: Manual verification**

Run: `curl -s http://localhost:3000/spells -o /dev/null -w "HTTP %{http_code}\n"`
Expected: HTTP 200

Open browser: `http://localhost:3000/spells`
Verify:
- Spell cards show purple (arcane) color
- Badges use purple theme
- Borders are purple (if updated)
- Dark mode works correctly

**Step 8: Commit**

```bash
git add app/components/spell/SpellCard.vue tests/components/spell/
git commit -m "refactor: Apply spell entity colors to SpellCard

- Replace generic color mapping with semantic 'spell' color
- Remove getLevelColor() function (simplified)
- Update tests to expect 'spell' color
- Add entity-colored borders
- Verified in browser: purple theme applied correctly

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 2: ItemCard Component

**Files:**
- Modify: `app/components/item/ItemCard.vue`
- Test: `tests/components/item/ItemCard.test.ts` (if exists)

**Step 1: Read ItemCard component**

Read: `app/components/item/ItemCard.vue`
Purpose: Identify all color usages

**Step 2: Update ItemCard template**

Replace generic colors with `color="item"`:
- Find `color="primary"`, `color="warning"`, `color="info"`, etc.
- Replace with `color="item"`
- Keep rarity colors (`rarityColor`) if they provide valuable sub-categorization
- Remove `getItemTypeColor()` if it just maps to generic colors

**Step 3: Update border classes**

```vue
<UCard class="border-2 border-item-300 dark:border-item-700 hover:border-item-500">
```

**Step 4: Update tests if they exist**

Read: `tests/components/item/ItemCard.test.ts` (if exists)
Update assertions: `expect(color).toBe('item')`

**Step 5: Run tests**

Run: `docker compose exec nuxt npm run test -- ItemCard.test.ts`
Expected: All tests pass (or skip if no tests)

**Step 6: Manual verification**

Open: `http://localhost:3000/items`
Verify:
- Item cards show gold (treasure) color
- Borders are golden
- Dark mode works

**Step 7: Commit**

```bash
git add app/components/item/ tests/components/item/
git commit -m "refactor: Apply item entity colors to ItemCard

- Use semantic 'item' color (treasure gold)
- Simplify color logic
- Update tests
- Add entity-colored borders

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 3: RaceCard Component

**Files:**
- Modify: `app/components/race/RaceCard.vue`
- Test: `tests/components/race/RaceCard.test.ts` (if exists)

**Step 1: Read RaceCard**

Read: `app/components/race/RaceCard.vue`

**Step 2: Update template**

Replace with `color="race"` (maps to emerald green)

**Step 3: Update borders**

```vue
<UCard class="border-2 border-race-300 dark:border-race-700 hover:border-race-500">
```

**Step 4: Update tests**

Update to expect `color="race"`

**Step 5: Run tests**

Run: `docker compose exec nuxt npm run test -- RaceCard.test.ts`

**Step 6: Verify**

Open: `http://localhost:3000/races`
Check: Emerald green theme

**Step 7: Commit**

```bash
git add app/components/race/ tests/components/race/
git commit -m "refactor: Apply race entity colors to RaceCard

- Use semantic 'race' color (emerald green)
- Update tests
- Add entity-colored borders

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 4: ClassCard Component

**Files:**
- Modify: `app/components/class/ClassCard.vue`
- Test: `tests/components/class/ClassCard.test.ts` (if exists)

**Step 1: Read ClassCard**

Read: `app/components/class/ClassCard.vue`

**Step 2: Update template**

Replace with `color="class"` (maps to red)

**Step 3: Update borders**

```vue
<UCard class="border-2 border-class-300 dark:border-class-700 hover:border-class-500">
```

**Step 4: Update tests**

Update to expect `color="class"`

**Step 5: Run tests**

Run: `docker compose exec nuxt npm run test -- ClassCard.test.ts`

**Step 6: Verify**

Open: `http://localhost:3000/classes`
Check: Red theme

**Step 7: Commit**

```bash
git add app/components/class/ tests/components/class/
git commit -m "refactor: Apply class entity colors to ClassCard

- Use semantic 'class' color (heroic red)
- Update tests
- Add entity-colored borders

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 5: BackgroundCard Component

**Files:**
- Modify: `app/components/background/BackgroundCard.vue`
- Test: `tests/components/background/BackgroundCard.test.ts` (if exists)

**Step 1: Read BackgroundCard**

Read: `app/components/background/BackgroundCard.vue`

**Step 2: Update template**

Replace with `color="background"` (maps to lore yellow-brown)

**Step 3: Update borders**

```vue
<UCard class="border-2 border-background-300 dark:border-background-700 hover:border-background-500">
```

**Step 4: Update tests**

Update to expect `color="background"`

**Step 5: Run tests**

Run: `docker compose exec nuxt npm run test -- BackgroundCard.test.ts`

**Step 6: Verify**

Open: `http://localhost:3000/backgrounds`
Check: Yellow-brown (lore) theme

**Step 7: Commit**

```bash
git add app/components/background/ tests/components/background/
git commit -m "refactor: Apply background entity colors to BackgroundCard

- Use semantic 'background' color (lore yellow-brown)
- Update tests
- Add entity-colored borders

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 6: FeatCard Component

**Files:**
- Modify: `app/components/feat/FeatCard.vue`
- Test: `tests/components/feat/FeatCard.test.ts` (if exists)

**Step 1: Read FeatCard**

Read: `app/components/feat/FeatCard.vue`

**Step 2: Update template**

Replace with `color="feat"` (maps to glory blue)

**Step 3: Update borders**

```vue
<UCard class="border-2 border-feat-300 dark:border-feat-700 hover:border-feat-500">
```

**Step 4: Update tests**

Update to expect `color="feat"`

**Step 5: Run tests**

Run: `docker compose exec nuxt npm run test -- FeatCard.test.ts`

**Step 6: Verify**

Open: `http://localhost:3000/feats`
Check: Blue (glory) theme

**Step 7: Commit**

```bash
git add app/components/feat/ tests/components/feat/
git commit -m "refactor: Apply feat entity colors to FeatCard

- Use semantic 'feat' color (glory blue)
- Update tests
- Add entity-colored borders

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 7: MonsterCard Component

**Files:**
- Modify: `app/components/monster/MonsterCard.vue`
- Test: `tests/components/monster/MonsterCard.test.ts`

**Step 1: Read MonsterCard**

Read: `app/components/monster/MonsterCard.vue`

**Step 2: Update template**

Replace with `color="monster"` (maps to danger orange)

**Step 3: Update borders**

```vue
<UCard class="border-2 border-monster-300 dark:border-monster-700 hover:border-monster-500">
```

**Step 4: Update tests**

Read: `tests/components/monster/MonsterCard.test.ts`
Update to expect `color="monster"`

**Step 5: Run tests**

Run: `docker compose exec nuxt npm run test -- MonsterCard.test.ts`

**Step 6: Verify**

Open: `http://localhost:3000/monsters`
Check: Orange (danger) theme

**Step 7: Commit**

```bash
git add app/components/monster/ tests/components/monster/
git commit -m "refactor: Apply monster entity colors to MonsterCard

- Use semantic 'monster' color (danger orange)
- Update tests
- Add entity-colored borders

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase B: Reference Entity Cards (10 components)

### Task 8: AbilityScoreCard

**Files:**
- Modify: `app/components/ability-score/AbilityScoreCard.vue`

**Steps:**
1. Read component
2. Replace colors with `color="ability"` (indigo)
3. Add borders: `border-ability-300/700/500`
4. Update tests if they exist
5. Verify: `http://localhost:3000/ability-scores`
6. Commit: "refactor: Apply ability entity colors to AbilityScoreCard"

---

### Task 9: ConditionCard

**Files:**
- Modify: `app/components/condition/ConditionCard.vue`

**Steps:**
1. Read component
2. Replace with `color="condition"` (rose)
3. Add borders: `border-condition-300/700/500`
4. Update tests
5. Verify: `http://localhost:3000/conditions`
6. Commit: "refactor: Apply condition entity colors to ConditionCard"

---

### Task 10: DamageTypeCard

**Files:**
- Modify: `app/components/damage-type/DamageTypeCard.vue`

**Steps:**
1. Read component
2. Replace with `color="damage"` (slate)
3. Add borders: `border-damage-300/700/500`
4. Update tests
5. Verify: `http://localhost:3000/damage-types`
6. Commit: "refactor: Apply damage entity colors to DamageTypeCard"

---

### Task 11: ItemTypeCard

**Files:**
- Modify: `app/components/item-type/ItemTypeCard.vue`

**Steps:**
1. Read component
2. Replace with `color="itemtype"` (teal)
3. Add borders: `border-itemtype-300/700/500`
4. Update tests
5. Verify: `http://localhost:3000/item-types`
6. Commit: "refactor: Apply itemtype entity colors to ItemTypeCard"

---

### Task 12: LanguageCard

**Files:**
- Modify: `app/components/language/LanguageCard.vue`

**Steps:**
1. Read component
2. Replace with `color="language"` (cyan)
3. Add borders: `border-language-300/700/500`
4. Update tests
5. Verify: `http://localhost:3000/languages`
6. Commit: "refactor: Apply language entity colors to LanguageCard"

---

### Task 13: ProficiencyTypeCard

**Files:**
- Modify: `app/components/proficiency-type/ProficiencyTypeCard.vue`

**Steps:**
1. Read component
2. Replace with `color="proficiency"` (lime)
3. Add borders: `border-proficiency-300/700/500`
4. Update tests
5. Verify: `http://localhost:3000/proficiency-types`
6. Commit: "refactor: Apply proficiency entity colors to ProficiencyTypeCard"

---

### Task 14: SizeCard

**Files:**
- Modify: `app/components/size/SizeCard.vue`

**Steps:**
1. Read component
2. Replace with `color="size"` (zinc)
3. Add borders: `border-size-300/700/500`
4. Update tests
5. Verify: `http://localhost:3000/sizes`
6. Commit: "refactor: Apply size entity colors to SizeCard"

---

### Task 15: SkillCard

**Files:**
- Modify: `app/components/skill/SkillCard.vue`
- Test: `tests/components/skill/SkillCard.test.ts`

**Steps:**
1. Read component
2. Replace with `color="skill"` (yellow)
3. Add borders: `border-skill-300/700/500`
4. Read and update tests
5. Run: `npm run test -- SkillCard.test.ts`
6. Verify: `http://localhost:3000/skills`
7. Commit: "refactor: Apply skill entity colors to SkillCard"

---

### Task 16: SpellSchoolCard

**Files:**
- Modify: `app/components/spell-school/SpellSchoolCard.vue`

**Steps:**
1. Read component
2. Replace with `color="school"` (fuchsia)
3. Add borders: `border-school-300/700/500`
4. Update tests
5. Verify: `http://localhost:3000/spell-schools`
6. Commit: "refactor: Apply school entity colors to SpellSchoolCard"

---

### Task 17: SourceCard

**Files:**
- Modify: `app/components/source/SourceCard.vue`

**Steps:**
1. Read component
2. Replace with `color="source"` (neutral)
3. Add borders: `border-source-300/700/500`
4. Update tests
5. Verify: `http://localhost:3000/sources`
6. Commit: "refactor: Apply source entity colors to SourceCard"

---

## Phase C: Search Components

### Task 18: SearchResultCard Component

**Files:**
- Modify: `app/components/SearchResultCard.vue`
- Test: `tests/components/SearchResultCard.test.ts`

**Step 1: Read SearchResultCard**

Read: `app/components/SearchResultCard.vue`
Purpose: Understand current color mapping

**Step 2: Simplify color logic**

**BEFORE:**
```vue
<script setup lang="ts">
const getBadgeColor = computed(() => {
  const colors: Record<string, ColorName> = {
    spell: 'primary',
    item: 'warning',
    race: 'info',
    class: 'error',
    background: 'success',
    feat: 'warning'
  }
  return colors[props.type] || 'neutral'
})
</script>

<template>
  <UBadge :color="getBadgeColor">{{ type }}</UBadge>
</template>
```

**AFTER:**
```vue
<script setup lang="ts">
// Remove getBadgeColor entirely!
</script>

<template>
  <!-- props.type is already 'spell', 'item', etc. -->
  <UBadge :color="type">{{ type }}</UBadge>
</template>
```

**Step 3: Update all color references**

Find all uses of `getBadgeColor` and replace with `:color="type"`

**Step 4: Update tests**

Read: `tests/components/SearchResultCard.test.ts`

Update test expectations:
```typescript
// Test for each entity type
it('displays spell badge with spell color', () => {
  const wrapper = mountSuspended(SearchResultCard, {
    props: { type: 'spell', result: mockSpell }
  })
  expect(wrapper.find('[data-test="type-badge"]').attributes('color')).toBe('spell')
})
```

**Step 5: Run tests**

Run: `docker compose exec nuxt npm run test -- SearchResultCard.test.ts`
Expected: All tests pass

**Step 6: Manual verification**

Open: `http://localhost:3000/search?q=fireball`
Verify:
- Spell results show purple badges
- Item results show gold badges
- Each entity type has correct color

**Step 7: Commit**

```bash
git add app/components/SearchResultCard.vue tests/components/SearchResultCard.test.ts
git commit -m "refactor: Simplify SearchResultCard to use entity colors

- Remove getBadgeColor mapping function
- Use props.type directly (already semantic color)
- Update tests to expect entity colors
- Verified in browser with search results

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 19: Search Page Filter Buttons

**Files:**
- Modify: `app/pages/search.vue`

**Step 1: Read search page**

Read: `app/pages/search.vue`
Focus on filter button section (around line 165-190)

**Step 2: Create entity color mapping**

Add to script section:
```vue
<script setup lang="ts">
// ... existing code ...

/**
 * Get entity color for filter buttons
 */
const getFilterColor = (value: string): string => {
  // Map plural entity names to entity colors
  const entityColors: Record<string, string> = {
    spells: 'spell',
    items: 'item',
    races: 'race',
    classes: 'class',
    backgrounds: 'background',
    feats: 'feat'
  }
  return entityColors[value] || 'neutral'
}
</script>
```

**Step 3: Update filter button colors**

**BEFORE:**
```vue
<UButton
  :variant="selectedTypes.includes(option.value) ? 'solid' : 'soft'"
  :color="selectedTypes.includes(option.value) ? 'primary' : 'neutral'"
>
  {{ option.label }}
</UButton>
```

**AFTER:**
```vue
<UButton
  :variant="selectedTypes.includes(option.value as EntityType) ? 'solid' : 'soft'"
  :color="selectedTypes.includes(option.value as EntityType) ? getFilterColor(option.value) : 'neutral'"
>
  {{ option.label }}
</UButton>
```

**Step 4: Manual verification**

Open: `http://localhost:3000/search?q=sword`
Verify:
- "All" button stays neutral
- "Spells" button shows purple when active
- "Items" button shows gold when active
- Each entity filter has correct color when selected

**Step 5: Commit**

```bash
git add app/pages/search.vue
git commit -m "refactor: Apply entity colors to search filter buttons

- Add getFilterColor() mapping function
- Active filters show entity-specific colors
- Inactive filters remain neutral
- Verified in browser: filters visually match entity types

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase D: Final Verification & Cleanup

### Task 20: Run Full Test Suite

**Step 1: Run all tests**

Run: `docker compose exec nuxt npm run test`
Expected: All tests pass

**Step 2: Check for failures**

If any tests fail:
- Read the failing test file
- Update color assertions to use entity colors
- Re-run tests until all pass

**Step 3: Type checking**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No type errors

**Step 4: Commit any remaining test fixes**

```bash
git add tests/
git commit -m "test: Fix remaining color assertions for entity colors

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 21: Visual Verification Tour

**Step 1: Visit all main entity pages**

For each URL, verify colors are correct:
- http://localhost:3000/spells → Purple (arcane)
- http://localhost:3000/items → Gold (treasure)
- http://localhost:3000/races → Green (emerald)
- http://localhost:3000/classes → Red
- http://localhost:3000/backgrounds → Yellow-brown (lore)
- http://localhost:3000/feats → Blue (glory)
- http://localhost:3000/monsters → Orange (danger)

**Step 2: Visit reference entity pages**

Check colors:
- http://localhost:3000/ability-scores → Indigo
- http://localhost:3000/conditions → Rose
- http://localhost:3000/damage-types → Slate
- http://localhost:3000/item-types → Teal
- http://localhost:3000/languages → Cyan
- http://localhost:3000/proficiency-types → Lime
- http://localhost:3000/sizes → Zinc
- http://localhost:3000/skills → Yellow
- http://localhost:3000/spell-schools → Fuchsia
- http://localhost:3000/sources → Neutral

**Step 3: Test search functionality**

- Visit: http://localhost:3000/search?q=fireball
- Verify search results show correct entity colors
- Click each filter button, verify active state shows entity color

**Step 4: Test dark mode**

Toggle dark mode on all pages, verify:
- Colors remain visible and accessible
- Borders show correctly
- No overwhelming brightness

---

### Task 22: Update CHANGELOG

**Step 1: Read CHANGELOG**

Read: `CHANGELOG.md`

**Step 2: Add entry**

Add to `[Unreleased]` section under `### Changed`:

```markdown
### Changed
- Applied entity-specific colors across all card components and search interface (2025-11-22)
  - Main entities: Spells (purple), Items (gold), Races (green), Classes (red), Backgrounds (yellow-brown), Feats (blue), Monsters (orange)
  - Reference entities: Ability scores, conditions, damage types, item types, languages, proficiencies, sizes, skills, spell schools, sources
  - Search results and filter buttons now use entity colors for instant visual recognition
  - Simplified component code by removing generic color mapping functions
```

**Step 3: Commit CHANGELOG**

```bash
git add CHANGELOG.md
git commit -m "docs: Update CHANGELOG for entity color rollout

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Success Criteria

After completing all tasks:

- [x] All 18 entity cards use semantic entity colors
- [x] SearchResultCard uses dynamic entity colors
- [x] Search filter buttons show entity colors when active
- [x] All tests pass
- [x] Type checking passes
- [x] All pages verified in browser (light + dark mode)
- [x] CHANGELOG updated
- [x] Code is simpler (color mapping functions removed)
- [x] User can identify entity types by color instantly

---

## Rollback Strategy

If issues arise with a specific entity:

1. Identify the problematic commit
2. Revert that entity's commit: `git revert <commit-hash>`
3. Fix the color definition in `app/assets/css/main.css` if needed
4. Re-apply the entity color changes
5. Re-commit

Example:
```bash
# Revert spell colors
git revert abc123

# Fix arcane palette in main.css
# Re-apply spell colors to SpellCard
# Commit again
```

---

## Estimated Timeline

- **Phase A (7 main entities):** 2-3.5 hours
- **Phase B (10 reference entities):** 2.5-5 hours
- **Phase C (search components):** 30-60 minutes
- **Phase D (verification & cleanup):** 30-60 minutes

**Total: 6-11 hours** (can be split across multiple sessions)

---

## Notes for Engineer

- Follow TDD: Read component → Update → Test → Verify → Commit
- Commit after EACH entity (frequent commits!)
- Don't batch multiple entities into one commit
- Test in browser after each entity (catch issues early)
- Keep sub-category colors (spell schools, item rarity) dynamic
- Only entity TYPE badges get static entity colors
- DRY: If you see repeated patterns, note them but don't abstract prematurely
- YAGNI: Don't add features beyond color replacement
- Entity colors are registered in `nuxt.config.ts`, mapped in `app.config.ts`
- No composables needed - NuxtUI handles resolution automatically
