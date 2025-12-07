# Design Document: Animated Fantasy Background

**Date:** 2025-11-22
**Feature:** Canvas-based animated background with mystical swirls and D&D runes
**Status:** Approved for implementation

---

## Overview

Add an atmospheric animated background to the D&D 5e Compendium application featuring mystical energy swirls and subtle runic symbols. The animation creates a fantasy screensaver-like ambiance while maintaining content readability and performance.

---

## Goals

### Primary Goals
- **Atmospheric Enhancement:** Create immersive D&D fantasy atmosphere without distracting from content
- **Performance-Conscious:** Smooth 30 FPS animation with minimal CPU/battery impact
- **Accessible:** Respect user accessibility preferences (prefers-reduced-motion)
- **Mode-Aware:** Adapt color palette to light/dark mode seamlessly

### Success Criteria
- Animation visible but subtle (15-25% effective opacity)
- No impact on text readability or UI interactions
- <5% CPU usage on modern devices while idle
- Zero hydration errors (SSR compatible)
- All existing tests continue passing (645/645)
- New tests for animation component and composable

---

## User Experience

### Visual Design

**Light Mode:**
- Mystical swirls in purple/blue tones (violet-500 range)
- Runes in deep indigo (indigo-600)
- Subtle, ethereal, complements gray-50 background

**Dark Mode:**
- Brighter swirls in purple/cyan (violet-400, cyan-400)
- Magical glow effect against gray-950 background
- More vibrant to stand out in dark theme

**Opacity Levels:**
- Swirls: 0.08-0.15 opacity (very subtle flowing energy)
- Runes: 0.03-0.12 opacity (ghostly, barely visible)
- Combined atmospheric effect: 15-25% presence

### Animation Behavior

**Swirls (Mystical Energy Currents):**
- ~40 particles flowing across screen
- Organic movement with sine wave drift
- Size variation: 20-60px radius
- Speed: 10-30 pixels/second
- Wraps around edges (infinite scroll)
- Bezier curve rendering with gradient fills

**Runes (Arcane Symbols):**
- ~6 runic symbols scattered across viewport
- Static position, slow rotation (0.1-0.3°/second)
- Fade in over 8-12 seconds, fade out over 8-12 seconds
- Reposition randomly when fully faded
- Unicode rune characters (Norse runes + geometric symbols)

**Interaction:**
- No user interaction (pointer-events: none)
- Purely decorative (aria-hidden: true)
- Consistent across all pages (no page-specific variations)

---

## Technical Architecture

### Component Structure

```
app/components/AnimatedBackground.vue
├── Canvas element (fixed, full viewport, z-index: -1)
├── Lifecycle management (mount/unmount)
├── Window resize handling
└── Color mode integration

app/composables/useAnimatedBackground.ts
├── Particle system classes (Swirl, Rune)
├── Animation loop (requestAnimationFrame)
├── Performance monitoring
└── Accessibility checks
```

### Implementation Details

#### AnimatedBackground.vue

**Responsibilities:**
- Render canvas element with correct positioning
- Check `prefers-reduced-motion` media query (skip rendering if true)
- Initialize animation engine on mount
- Handle window resize events
- Pass dark/light mode to composable
- Clean up on unmount

**Template:**
```vue
<template>
  <canvas
    ref="canvasRef"
    class="fixed inset-0 -z-10 pointer-events-none"
    aria-hidden="true"
  />
</template>
```

**Key Attributes:**
- `fixed inset-0`: Full viewport coverage
- `-z-10`: Behind all content (Tailwind utility)
- `pointer-events-none`: Clicks pass through to content
- `aria-hidden="true"`: Decorative, not meaningful to screen readers

#### useAnimatedBackground.ts

**Particle Classes:**

```typescript
class Swirl {
  x: number              // Position X
  y: number              // Position Y
  vx: number             // Velocity X (10-30 px/s)
  vy: number             // Velocity Y (10-30 px/s)
  size: number           // Radius (20-60px)
  phase: number          // Sine wave phase offset
  opacity: number        // 0.08-0.15

  update(deltaTime: number): void
  draw(ctx: CanvasRenderingContext2D, colors: ColorPalette): void
}

class Rune {
  x: number              // Position X
  y: number              // Position Y
  symbol: string         // Unicode rune character
  size: number           // Font size (40-80px)
  opacity: number        // 0-0.12 (fades in/out)
  rotation: number       // Current rotation angle
  fadeDirection: number  // 1 (fading in) or -1 (fading out)

  update(deltaTime: number): void
  draw(ctx: CanvasRenderingContext2D, colors: ColorPalette): void
}
```

**Animation Loop:**

```typescript
- Target: 30 FPS (16.67ms budget, throttled to 33.33ms)
- Delta time tracking for frame-independent animation
- Visibility API: pause when tab hidden
- Clear canvas each frame
- Update all particles (position, opacity, rotation)
- Draw all particles
- requestAnimationFrame for next frame
```

**Performance Optimizations:**
1. **FPS Throttling:** 30 FPS instead of 60 FPS (50% reduction in processing)
2. **Visibility API:** Complete pause when tab not active (battery friendly)
3. **Reduced Motion:** Skip initialization if `prefers-reduced-motion: reduce`
4. **Simple Particle Count:** 46 total particles (40 swirls + 6 runes)
5. **No Image Assets:** Pure geometric/text rendering (low memory)
6. **Delta Time Compensation:** Consistent speed across varying frame rates

**Rune Character Set:**

```typescript
const RUNE_SYMBOLS = [
  'ᚠ', 'ᚢ', 'ᚦ', 'ᚨ', 'ᚱ', 'ᚲ',  // Norse runes (fantasy aesthetic)
  '⚔', '✦', '◈', '⬡', '⬢', '⬣'     // Geometric D&D symbols
]
```

**Color Palettes:**

```typescript
interface ColorPalette {
  swirlColor: string
  runeColor: string
}

const LIGHT_MODE_COLORS = {
  swirlColor: 'rgba(139, 92, 246, {opacity})',  // violet-500
  runeColor: 'rgba(79, 70, 229, {opacity})'      // indigo-600
}

const DARK_MODE_COLORS = {
  swirlColor: 'rgba(167, 139, 250, {opacity})',  // violet-400
  runeColor: 'rgba(34, 211, 238, {opacity})'     // cyan-400
}
```

---

## Integration

### app.vue Modification

Add `<AnimatedBackground />` component inside main container:

```vue
<template>
  <UApp>
    <div class="min-h-screen bg-gray-50 dark:bg-gray-950 text-gray-900 dark:text-gray-100">
      <!-- NEW: Animated background -->
      <ClientOnly>
        <AnimatedBackground />
      </ClientOnly>

      <!-- Existing navigation and content -->
      <nav>...</nav>
      <UMain>...</UMain>
    </div>
  </UApp>
</template>
```

**Why ClientOnly:**
- Canvas API only works in browser (not during SSR)
- Prevents hydration mismatches
- Graceful degradation (no animation during SSR, appears on hydration)

---

## Accessibility

### Reduced Motion Support

**Detection:**
```typescript
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches
```

**Behavior:**
- If `true`: Skip animation initialization entirely (canvas stays blank)
- If `false`: Render animation normally
- Respects OS-level accessibility settings

### Screen Reader Behavior

- Canvas marked `aria-hidden="true"` (decorative only)
- No semantic content in animation
- Zero impact on keyboard navigation or focus management

### Manual Toggle

Not implemented in v1. Users who need to disable animations can:
1. Enable "Reduce motion" in OS accessibility settings
2. Use browser extensions to disable animations
3. Future enhancement: Add manual toggle near dark mode button

---

## Testing Strategy

### Unit Tests

**AnimatedBackground.test.ts:**
- ✅ Renders canvas element
- ✅ Sets canvas to full viewport size
- ✅ Positions canvas behind content (z-index: -1)
- ✅ Makes canvas non-interactive (pointer-events: none)
- ✅ Stops animation when unmounted
- ✅ Skips rendering when prefers-reduced-motion is true
- ✅ Updates colors when dark mode changes

**useAnimatedBackground.test.ts:**
- ✅ Initializes particles on setup
- ✅ Starts animation loop
- ✅ Pauses animation when document hidden
- ✅ Resumes animation when document visible
- ✅ Cleans up on destroy
- ✅ Uses correct color palette for light mode
- ✅ Uses correct color palette for dark mode

### Integration Tests

**app.test.ts:**
- ✅ Renders AnimatedBackground component
- ✅ No hydration errors (ClientOnly wrapper)
- ✅ Canvas exists in DOM after mount

### Manual Browser Testing

**Visual verification:**
- [ ] Open app, see subtle swirls + runes
- [ ] Toggle dark mode, colors change appropriately
- [ ] Switch tabs, animation pauses (verify in DevTools Performance tab)
- [ ] Enable "Reduce motion" in OS, verify no animation
- [ ] Test on mobile (performance acceptable)
- [ ] Verify text remains readable on all pages
- [ ] Check light/dark mode contrast

**Performance verification:**
- [ ] CPU usage <5% while idle (DevTools Performance)
- [ ] No scroll jank while animation running
- [ ] Tab switching pauses animation immediately
- [ ] FPS stable at 30 (not fluctuating wildly)

---

## Performance Budget

### Target Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| FPS | 30 (±3) | DevTools Performance panel |
| CPU (idle) | <5% | DevTools Performance panel |
| Memory | <10MB | DevTools Memory profiler |
| Frame time | ~33ms | DevTools Performance panel |
| Startup impact | <50ms | Component mount timing |

### Degradation Strategy

If performance issues detected:
1. **First:** Reduce particle count (40 swirls → 20 swirls)
2. **Second:** Disable runes (simpler swirls only)
3. **Third:** Skip animation on mobile devices entirely
4. **Last resort:** Add manual toggle to disable

---

## File Structure

### New Files

```
app/components/AnimatedBackground.vue          (~40 lines)
app/composables/useAnimatedBackground.ts       (~250 lines)
tests/components/AnimatedBackground.test.ts    (~80 lines)
tests/composables/useAnimatedBackground.test.ts (~100 lines)
docs/plans/2025-11-22-animated-background-design.md (this file)
docs/plans/2025-11-22-animated-background-implementation.md
```

### Modified Files

```
app/app.vue                                    (+3 lines: import + component)
CHANGELOG.md                                   (+1 entry)
docs/CURRENT_STATUS.md                         (update feature list)
```

### Estimated Totals

- ~470 lines of new code
- ~10 new tests
- Zero changes to existing components (isolated feature)
- 1-2 hour implementation time (with TDD)

---

## Future Enhancements

### Potential v2 Features

1. **Manual Toggle:** Add button near dark mode toggle to disable animation
2. **Performance Auto-Adjust:** Detect slow devices, reduce particle count automatically
3. **Page-Specific Themes:** More intense swirls on spell pages, subtle on reference pages
4. **Interaction Effects:** Mouse movement influences swirl direction (subtle parallax)
5. **Season/Event Themes:** Special particle effects for holidays or game releases

### Extensibility

The particle system architecture supports easy addition of new effect types:
- Add new class (e.g., `class Sparkle`)
- Instantiate in composable
- Include in update/draw loops
- No changes to component layer

---

## Alternatives Considered

### Multi-Layer Canvas System
- **Rejected:** Added complexity (3 components) without clear benefit
- **Reason:** Single canvas simpler to test and maintain

### CSS-Only Animation
- **Rejected:** Cannot achieve particle effects or runes
- **Reason:** CSS gradients too limited for desired visual effects

### WebGL/Three.js
- **Rejected:** Massive overkill for 2D particle system
- **Reason:** Canvas 2D API sufficient, lighter bundle size

### Always-On Animation (No Pause)
- **Rejected:** Battery drain on laptops/mobile
- **Reason:** Visibility API pause improves battery life significantly

---

## Risk Assessment

### Low Risk
- ✅ Isolated feature (no changes to existing components)
- ✅ ClientOnly wrapper prevents SSR issues
- ✅ Simple particle count (predictable performance)

### Medium Risk
- ⚠️ Performance on older devices (mitigation: prefers-reduced-motion)
- ⚠️ Mobile battery impact (mitigation: visibility API pause)

### Mitigations
- Comprehensive performance testing before merge
- Manual toggle as escape hatch (future v2 if needed)
- Feature can be disabled by removing single component

---

## Success Metrics

### Qualitative
- Users report app feels more immersive
- No complaints about readability or distraction
- Positive feedback on fantasy aesthetic

### Quantitative
- Zero increase in bug reports
- No performance regressions (Lighthouse scores maintained)
- 100% test pass rate maintained (645+ tests)
- <5% CPU usage verified on target devices

---

## Conclusion

The animated fantasy background feature adds atmospheric D&D immersion through a performance-conscious canvas particle system. The single-component architecture ensures maintainability while delivering mystical swirls and subtle runes that enhance the user experience without compromising readability or accessibility.

**Design Status:** ✅ Approved for implementation
**Next Step:** Create implementation plan and set up development worktree

---

**Document Version:** 1.0
**Author:** Claude Code
**Approved By:** User
**Implementation Start:** 2025-11-22
