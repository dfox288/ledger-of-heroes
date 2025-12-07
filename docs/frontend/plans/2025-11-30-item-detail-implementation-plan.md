# Item Detail Page Implementation Plan

**Design Doc:** `2025-11-30-item-detail-page-redesign.md`
**Approach:** TDD - Tests first, then implementation

## Phase 1: Composable Foundation

### Task 1.1: Create `useItemDetail` composable
**File:** `app/composables/useItemDetail.ts`
**Tests:** `tests/composables/useItemDetail.test.ts`

```typescript
// Test cases:
- detects melee weapon (item_type.code === 'M')
- detects ranged weapon (item_type.code === 'R')
- detects ammunition (item_type.code === 'A')
- detects light armor (item_type.code === 'LA')
- detects medium armor (item_type.code === 'MA')
- detects heavy armor (item_type.code === 'HA')
- detects shield (item_type.code === 'S')
- detects charged item (charges_max not null)
- detects magic item (is_magic === true)
- detects item with spells (spells.length > 0)
- parses attunement text from detail field
- groups spells by charge cost
- formats damage display (with versatile)
- formats AC display
- formats cost in gp/cp
- returns category color based on item type
```

**Implementation:**
- Wrap `useEntityDetail` for data fetching
- Add computed properties for type detection
- Parse `detail` field for attunement requirements (regex)
- Group spells by `charges_cost` field

---

## Phase 2: Hero & Quick Info Components

### Task 2.1: Create `ItemHero` component
**File:** `app/components/item/ItemHero.vue`
**Tests:** `tests/components/item/ItemHero.test.ts`

```typescript
// Test cases:
- renders item name
- renders item type badge with correct color
- renders rarity badge with correct color
- renders magic bonus badge when present (+1, +2, +3)
- renders magic sparkle indicator when is_magic
- renders attunement badge when requires_attunement
- renders image when available
- handles missing image gracefully
- applies correct layout (2/3 text, 1/3 image)
```

**Props:**
```typescript
interface Props {
  item: Item
  imagePath: string | null
}
```

### Task 2.2: Create `ItemQuickInfo` component
**File:** `app/components/item/ItemQuickInfo.vue`
**Tests:** `tests/components/item/ItemQuickInfo.test.ts`

```typescript
// Test cases:
- renders cost when present (formatted)
- renders weight when present
- renders source with page number
- handles missing cost gracefully
- handles missing weight gracefully
- displays multiple sources
```

**Props:**
```typescript
interface Props {
  cost: string | null      // Pre-formatted "15 gp"
  weight: string | null    // "3 lb"
  sources: Source[]
}
```

---

## Phase 3: Combat Stats Components

### Task 3.1: Create `ItemWeaponStats` component
**File:** `app/components/item/ItemWeaponStats.vue`
**Tests:** `tests/components/item/ItemWeaponStats.test.ts`

```typescript
// Test cases:
- renders damage dice
- renders damage type name
- renders versatile damage in parentheses when present
- renders range for ranged weapons (normal/long)
- shows "Melee" for melee weapons without range
- renders property badges
- property badges show tooltip on hover
- handles weapon with no properties
- handles ammunition (shows quantity context)
```

**Props:**
```typescript
interface Props {
  damageDice: string | null
  versatileDamage: string | null
  damageType: DamageType | null
  rangeNormal: number | null
  rangeLong: number | null
  properties: Property[]
  proficiencyCategory: string | null
}
```

### Task 3.2: Create `ItemArmorStats` component
**File:** `app/components/item/ItemArmorStats.vue`
**Tests:** `tests/components/item/ItemArmorStats.test.ts`

```typescript
// Test cases:
- renders armor class value
- renders strength requirement when present
- shows "None" for STR when no requirement
- renders stealth disadvantage indicator
- shows "Normal" stealth when no disadvantage
- renders dex modifier info (from modifiers)
- parses dex modifier condition correctly
- handles shield (+2 AC bonus display)
- shows speed penalty warning when STR requirement exists
```

**Props:**
```typescript
interface Props {
  armorClass: number | null
  strengthRequirement: number | null
  stealthDisadvantage: boolean
  modifiers: Modifier[]
  isShield: boolean
}
```

---

## Phase 4: Magic Item Components

### Task 4.1: Create `ItemMagicSection` component
**File:** `app/components/item/ItemMagicSection.vue`
**Tests:** `tests/components/item/ItemMagicSection.test.ts`

```typescript
// Test cases:
- renders charges with max value
- renders recharge formula and timing
- handles missing recharge info gracefully
- renders attunement requirement text
- renders modifier bonuses (attack, damage, spell attack)
- groups modifiers by category
- handles item with charges but no attunement
- handles item with attunement but no charges
```

**Props:**
```typescript
interface Props {
  chargesMax: string | null
  rechargeFormula: string | null
  rechargeTiming: string | null
  attunementText: string | null
  modifiers: Modifier[]
}
```

### Task 4.2: Create `ItemSpellsTable` component
**File:** `app/components/item/ItemSpellsTable.vue`
**Tests:** `tests/components/item/ItemSpellsTable.test.ts`

```typescript
// Test cases:
- renders spells grouped by charge cost
- shows "Free" for spells with no charge cost
- sorts groups by charge cost (Free first, then ascending)
- renders spell names as links to spell pages
- handles empty spells array
- shows spell level in parentheses
- handles spells without linked spell data (name only)
```

**Props:**
```typescript
interface Props {
  spellsByChargeCost: Map<string, ItemSpell[]>
}
```

---

## Phase 5: Page Assembly

### Task 5.1: Rewrite `[slug].vue` page
**File:** `app/pages/items/[slug].vue`
**Tests:** `tests/pages/items/detail.test.ts`

```typescript
// Test cases:
- renders loading state
- renders error state
- renders breadcrumb
- renders ItemHero for all items
- renders ItemWeaponStats only for weapons
- renders ItemArmorStats only for armor
- renders ItemMagicSection for magic items with charges/attunement
- renders ItemSpellsTable when item has spells
- renders description
- renders ItemQuickInfo
- renders accordion with abilities when present
- renders accordion with data tables when present
- renders accordion with prerequisites when present
- renders accordion with saving throws when present
- complex item (Staff of Magi) shows all relevant sections
- simple item (Longsword) shows only weapon stats
- armor item (Plate) shows only armor stats
- potion shows only description and quick info
```

---

## Phase 6: Polish & Edge Cases

### Task 6.1: Handle edge cases
- Items with both weapon AND armor stats (if any exist)
- Items with empty descriptions
- Items with very long property lists
- Items with no image

### Task 6.2: Visual polish
- Consistent spacing between sections
- Badge colors match design system
- Dark mode compatibility
- Mobile responsiveness

### Task 6.3: Delete old code
- Remove unused accordion slots from old implementation
- Clean up any orphaned components

---

## Verification Checklist

After each phase:
- [ ] All new tests pass
- [ ] `npm run test:items` passes
- [ ] `npm run typecheck` passes
- [ ] `npm run lint:fix` passes
- [ ] Visual check in browser (Longsword, Plate Armor, Staff of the Magi)

Final verification:
- [ ] Full test suite passes
- [ ] All item types render correctly
- [ ] Mobile responsive
- [ ] Dark mode works
- [ ] No console errors

---

## Estimated Effort

| Phase | Tasks | Est. Time |
|-------|-------|-----------|
| Phase 1 | Composable | 30 min |
| Phase 2 | Hero + Quick Info | 30 min |
| Phase 3 | Combat Stats | 45 min |
| Phase 4 | Magic Components | 45 min |
| Phase 5 | Page Assembly | 30 min |
| Phase 6 | Polish | 30 min |
| **Total** | | **~3.5 hours** |
