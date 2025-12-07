# Entity Images Feature Design

**Date:** 2025-11-22
**Status:** Approved
**Entities:** Races (initial), Classes/Backgrounds/Spells (future)

## Overview

Add generated AI images to entity detail pages and list cards using pre-optimized image variants (256px, 512px, original). Images are optional, gracefully degrade when missing, and use a configurable provider system.

## Goals

1. **Detail Pages:** CV-style header with image on right (1/3 width, 512px variant)
2. **List Cards:** Subtle background images with opacity (10% base, 20% hover, 256px variant)
3. **Reusability:** Components work across all entity types (races, classes, backgrounds, spells, items)
4. **Performance:** Use pre-optimized images from image-generator conversions
5. **Flexibility:** Configurable image provider via environment variable

## Architecture

### Environment Configuration

**New Variable:**
```bash
NUXT_PUBLIC_IMAGE_PROVIDER=stability-ai
```

**Runtime Config Addition (nuxt.config.ts):**
```typescript
public: {
  imageProvider: process.env.NUXT_PUBLIC_IMAGE_PROVIDER || 'stability-ai',
  // ... existing config
}
```

**Why Public:**
- Client-side image path construction
- Non-sensitive configuration
- Enables provider switching without code changes

---

### Composable: useEntityImage

**File:** `app/composables/useEntityImage.ts`

**Signature:**
```typescript
function useEntityImage(
  slug: string,
  entity: 'races' | 'classes' | 'backgrounds' | 'spells' | 'items',
  size?: '256' | '512' | 'original'
): string | null
```

**Path Resolution:**
- `256`: `/images/generated/conversions/256/{entity}/{provider}/{slug}.png`
- `512`: `/images/generated/conversions/512/{entity}/{provider}/{slug}.png`
- `original`: `/images/generated/{entity}/{provider}/{slug}.png`
- Invalid inputs: `null`

**Example:**
```typescript
useEntityImage('dragonborn', 'races', '512')
// → '/images/generated/conversions/512/races/stability-ai/dragonborn.png'
```

**Error Handling:**
- Returns `null` for invalid slug/entity/size
- Components use `v-if` for graceful degradation
- No runtime errors, no broken images

---

### Component: UiEntityHeaderWithImage

**File:** `app/components/ui/UiEntityHeaderWithImage.vue`

**Purpose:** Replace `UiDetailPageHeader` with image-aware version

**Layout:**
```
Desktop (≥768px):
┌──────────────────────────────────────────────┐
│  [Header Content - 2/3]    │  [Image - 1/3] │
│  - Title                    │   Aspect 1:1   │
│  - Badges                   │   512px        │
│  - Description/Stats        │   Lazy loaded  │
└──────────────────────────────────────────────┘

Mobile (<768px):
┌──────────────────┐
│  [Image - top]   │
│  Smaller, square │
├──────────────────┤
│  [Header Content]│
│  Full width      │
└──────────────────┘
```

**Props:**
```typescript
interface Props {
  title: string
  badges?: Array<{ label: string, color: string, variant: string, size: string }>
  imagePath?: string | null
  imageAlt?: string
}
```

**Key Features:**
- Flexbox responsive layout: `flex flex-col md:flex-row gap-6`
- Image section: `w-full md:w-1/3 aspect-square`
- Content section: `flex-1`
- Conditional rendering: `v-if="imagePath"` (no image = no layout shift)
- NuxtImg with `loading="lazy"` and `fit="cover"`
- Rounded corners, subtle shadow

---

### Component Update: RaceCard

**File:** `app/components/race/RaceCard.vue` (modify existing)

**Background Image Layer:**
```vue
<NuxtLink :to="`/races/${race.slug}`" class="block h-full group">
  <UCard class="relative overflow-hidden hover:shadow-lg transition-shadow h-full border">
    <!-- Background Image (absolute positioned) -->
    <div
      v-if="backgroundImage"
      class="absolute inset-0 bg-cover bg-center opacity-10 transition-opacity duration-300 group-hover:opacity-20"
      :style="{ backgroundImage: `url(${backgroundImage})` }"
    />

    <!-- Content (relative z-10) -->
    <div class="relative z-10">
      <!-- Existing card content unchanged -->
    </div>
  </UCard>
</NuxtLink>
```

**Computed Property:**
```typescript
const backgroundImage = computed(() =>
  useEntityImage(props.race.slug, 'races', '256')
)
```

**Opacity Strategy:**
- Base: 10% (subtle, doesn't interfere)
- Hover: 20% (gentle enhancement via group-hover)
- Transition: 300ms smooth
- Text contrast: Maintained at all opacity levels

**Why CSS Background vs NuxtImg:**
- Easier opacity/blend mode control
- Better performance for decorative images
- No need for optimization (256px pre-optimized)
- Simpler z-index layering

---

### Integration: Race Detail Page

**File:** `app/pages/races/[slug].vue`

**Changes:**

1. **Add composable:**
```typescript
const imagePath = computed(() =>
  race.value ? useEntityImage(race.value.slug, 'races', '512') : null
)
```

2. **Replace header component:**
```vue
<!-- Before -->
<UiDetailPageHeader
  :title="race.name"
  :badges="[...]"
/>

<!-- After -->
<UiEntityHeaderWithImage
  :title="race.name"
  :badges="[...]"
  :image-path="imagePath"
  :image-alt="`${race.name} character portrait`"
/>
```

**No other changes needed** - rest of page remains identical.

---

### Integration: Race List Page

**File:** `app/pages/races/index.vue`

**Changes:** None required

RaceCard component handles background images internally. List page continues to use `<RaceCard>` as before.

---

## Testing Strategy (TDD)

### Test Files to Create

1. **`tests/composables/useEntityImage.test.ts`**
   - Path generation for all sizes (256, 512, original)
   - Provider from runtime config
   - Null returns for invalid inputs
   - All entity types (races, classes, etc.)

2. **`tests/components/ui/UiEntityHeaderWithImage.test.ts`**
   - Renders with image path
   - Renders without image path (no image section)
   - Props render correctly (title, badges)
   - Image alt text set correctly
   - Responsive layout classes present

3. **`tests/components/race/RaceCard.test.ts`** (update existing)
   - Background image div renders when image exists
   - Background image div hidden when null
   - Opacity classes applied correctly
   - Group hover classes present

### TDD Workflow

**RED-GREEN-REFACTOR:**
1. ✅ Write failing tests for composable
2. ✅ Implement composable to pass tests
3. ✅ Write failing tests for UiEntityHeaderWithImage
4. ✅ Implement component to pass tests
5. ✅ Update RaceCard tests
6. ✅ Modify RaceCard to pass tests
7. ✅ Update integration tests
8. ✅ Manual browser verification (all pages HTTP 200)

**Coverage Requirements:**
- Composable: 100% (simple path logic)
- Components: >90% (visual, edge cases)
- Integration: Happy path + no-image fallback

---

## File Structure

```
frontend/
├── app/
│   ├── composables/
│   │   └── useEntityImage.ts                    [NEW]
│   ├── components/
│   │   ├── ui/
│   │   │   └── UiEntityHeaderWithImage.vue      [NEW]
│   │   └── race/
│   │       └── RaceCard.vue                     [MODIFIED]
│   └── pages/
│       └── races/
│           ├── [slug].vue                       [MODIFIED]
│           └── index.vue                        [NO CHANGE]
├── tests/
│   ├── composables/
│   │   └── useEntityImage.test.ts               [NEW]
│   └── components/
│       ├── ui/
│       │   └── UiEntityHeaderWithImage.test.ts  [NEW]
│       └── race/
│           └── RaceCard.test.ts                 [UPDATED]
├── .env.example                                 [MODIFIED]
├── .env                                         [MODIFIED]
└── nuxt.config.ts                               [MODIFIED]
```

---

## Accessibility

**Images on Detail Pages:**
- Meaningful alt text: `"${entityName} character portrait"`
- Lazy loading (performance)
- No layout shift when image loads

**Background Images on Cards:**
- Purely decorative (CSS background)
- No alt text needed
- Text contrast maintained (10-20% opacity)
- Sufficient WCAG AA contrast ratios

**Keyboard Navigation:**
- All existing navigation works unchanged
- Images don't interfere with focus states

---

## Performance Considerations

**Optimizations Built-In:**
- Pre-optimized images (256px, 512px) from image-generator
- Lazy loading via NuxtImg on detail pages
- Small file sizes (~90-400KB for conversions)
- No runtime image processing needed
- CSS backgrounds for cards (lightweight)

**Expected Impact:**
- Detail page: +400KB max (512px image, lazy loaded)
- Card backgrounds: +90KB each (256px, only visible cards)
- No impact on Time to Interactive
- Minimal impact on LCP (images below fold mostly)

---

## Future Extensibility

**Easy Additions:**
1. **Other Entities:** Same pattern for classes/backgrounds/spells
   ```typescript
   useEntityImage(class.slug, 'classes', '512')
   ```

2. **Image Lightbox:** Click to view original high-res
   ```typescript
   useEntityImage(race.slug, 'races', 'original')
   ```

3. **Fallback Images:** Default placeholder when missing
   ```typescript
   imagePath.value || '/images/defaults/race-placeholder.png'
   ```

4. **Different Providers:** Change env variable only
   ```bash
   NUXT_PUBLIC_IMAGE_PROVIDER=dall-e
   ```

5. **WebP Support:** Add `.webp` variants to conversions
   ```typescript
   `/conversions/256/${entity}/${provider}/${slug}.webp`
   ```

**No component changes required** for any of the above!

---

## Dependencies

**Required:**
- ✅ @nuxt/image (already installed)
- ✅ Generated images mounted at `/images/generated/`
- ✅ Conversions folder with 256px and 512px variants

**Image Structure:**
```
/images/generated/
├── conversions/
│   ├── 256/
│   │   └── races/stability-ai/*.png
│   └── 512/
│       └── races/stability-ai/*.png
└── races/stability-ai/*.png (originals)
```

---

## Success Criteria

**Before marking complete:**
- [ ] All new tests pass (composable + components)
- [ ] Full test suite passes (no regressions)
- [ ] TypeScript compiles with no errors
- [ ] ESLint passes
- [ ] Race detail pages show images (512px, right side)
- [ ] Race cards show background images (256px, 10% opacity)
- [ ] Hover effect works (20% opacity)
- [ ] Missing images degrade gracefully (no broken images)
- [ ] Mobile responsive (image on top)
- [ ] Dark mode works
- [ ] All pages verified in browser (HTTP 200)
- [ ] CHANGELOG.md updated
- [ ] Work committed

---

## Implementation Order

**Following TDD:**

1. **Environment Setup**
   - Add `NUXT_PUBLIC_IMAGE_PROVIDER` to `.env` and `.env.example`
   - Update `nuxt.config.ts` runtime config
   - Commit

2. **Composable (TDD)**
   - Write tests for `useEntityImage`
   - Implement composable
   - All tests pass
   - Commit

3. **Header Component (TDD)**
   - Write tests for `UiEntityHeaderWithImage`
   - Implement component
   - All tests pass
   - Commit

4. **Card Background (TDD)**
   - Update tests for `RaceCard`
   - Modify component
   - All tests pass
   - Commit

5. **Integration**
   - Update race detail page
   - Manual browser testing
   - Update CHANGELOG
   - Commit

6. **Verification**
   - Full test suite
   - All pages work
   - Create handover document

---

## Notes

**Why Not Use NuxtImg for Card Backgrounds?**
- CSS backgrounds give better opacity/blend control
- Absolute positioning easier with divs
- Images already optimized (256px)
- No need for srcset/responsive variants
- Simpler z-index layering

**Provider Strategy:**
- Single provider per environment (configured via env)
- Future: Could extend to per-entity provider if needed
- Current images: `stability-ai` only

**Image Aspect Ratio:**
- All generated images are square (1:1)
- Header component enforces `aspect-square`
- Cards use `bg-cover` for any ratio

---

**Design approved. Ready for worktree setup and implementation planning.**
