# Animated Fantasy Background Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add canvas-based animated background with mystical swirls and D&D runes to create atmospheric fantasy ambiance.

**Architecture:** Single Vue component (`AnimatedBackground.vue`) manages canvas element and lifecycle. Composable (`useAnimatedBackground.ts`) contains particle system classes (Swirl, Rune) and animation loop. Component integrates into `app.vue` with ClientOnly wrapper for SSR compatibility.

**Tech Stack:** Vue 3 Composition API, TypeScript, Canvas 2D API, Vitest + @nuxt/test-utils, Tailwind CSS

---

## Prerequisites

- Backend running: `cd ../importer && docker compose up -d`
- Frontend running: `docker compose up -d`
- Node modules installed: `docker compose exec nuxt npm install`

---

## Task 1: Create Swirl Particle Class (TDD)

**Files:**
- Create: `app/composables/useAnimatedBackground.ts`
- Test: `tests/composables/useAnimatedBackground.test.ts`

**Step 1: Write the failing test**

Create `tests/composables/useAnimatedBackground.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'

describe('useAnimatedBackground', () => {
  describe('Swirl class', () => {
    it('initializes with random position and velocity', () => {
      // We'll export Swirl class for testing
      const { Swirl } = await import('~/composables/useAnimatedBackground')

      const swirl = new Swirl(1920, 1080)

      expect(swirl.x).toBeGreaterThanOrEqual(0)
      expect(swirl.x).toBeLessThanOrEqual(1920)
      expect(swirl.y).toBeGreaterThanOrEqual(0)
      expect(swirl.y).toBeLessThanOrEqual(1080)
      expect(swirl.vx).toBeGreaterThanOrEqual(-30)
      expect(swirl.vx).toBeLessThanOrEqual(30)
      expect(swirl.size).toBeGreaterThanOrEqual(20)
      expect(swirl.size).toBeLessThanOrEqual(60)
    })

    it('updates position based on velocity and deltaTime', () => {
      const { Swirl } = await import('~/composables/useAnimatedBackground')

      const swirl = new Swirl(1920, 1080)
      const initialX = swirl.x
      const initialY = swirl.y

      swirl.update(1000) // 1 second

      // Position should have changed
      expect(swirl.x).not.toBe(initialX)
      expect(swirl.y).not.toBe(initialY)
    })

    it('wraps around screen edges', () => {
      const { Swirl } = await import('~/composables/useAnimatedBackground')

      const swirl = new Swirl(100, 100)
      swirl.x = 110 // Beyond right edge
      swirl.y = 110 // Beyond bottom edge

      swirl.update(16)

      expect(swirl.x).toBeLessThanOrEqual(100)
      expect(swirl.y).toBeLessThanOrEqual(100)
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: FAIL with "Cannot find module '~/composables/useAnimatedBackground'"

**Step 3: Write minimal implementation**

Create `app/composables/useAnimatedBackground.ts`:

```typescript
export class Swirl {
  x: number
  y: number
  vx: number
  vy: number
  size: number
  phase: number
  opacity: number
  private width: number
  private height: number

  constructor(width: number, height: number) {
    this.width = width
    this.height = height

    // Random position
    this.x = Math.random() * width
    this.y = Math.random() * height

    // Random velocity (-30 to 30 pixels/second)
    this.vx = (Math.random() - 0.5) * 60
    this.vy = (Math.random() - 0.5) * 60

    // Random size (20-60px radius)
    this.size = 20 + Math.random() * 40

    // Random phase for sine wave
    this.phase = Math.random() * Math.PI * 2

    // Random opacity (0.08-0.15)
    this.opacity = 0.08 + Math.random() * 0.07
  }

  update(deltaTime: number): void {
    // Convert deltaTime from milliseconds to seconds
    const dt = deltaTime / 1000

    // Update position based on velocity
    this.x += this.vx * dt
    this.y += this.vy * dt

    // Add sine wave drift for organic movement
    this.phase += dt * 0.5
    this.x += Math.sin(this.phase) * 0.5
    this.y += Math.cos(this.phase) * 0.5

    // Wrap around edges
    if (this.x > this.width) this.x = 0
    if (this.x < 0) this.x = this.width
    if (this.y > this.height) this.y = 0
    if (this.y < 0) this.y = this.height
  }

  draw(ctx: CanvasRenderingContext2D, color: string): void {
    // Will implement in next task
  }
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: PASS (3 tests)

**Step 5: Commit**

```bash
git add app/composables/useAnimatedBackground.ts tests/composables/useAnimatedBackground.test.ts
git commit -m "feat: Add Swirl particle class with movement and wrapping

- Random position and velocity initialization
- Delta time-based movement with sine wave drift
- Screen edge wrapping for infinite scroll effect
- 3 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Add Swirl Drawing Method (TDD)

**Files:**
- Modify: `app/composables/useAnimatedBackground.ts`
- Modify: `tests/composables/useAnimatedBackground.test.ts`

**Step 1: Write the failing test**

Add to `tests/composables/useAnimatedBackground.test.ts`:

```typescript
it('draws swirl with gradient fill', () => {
  const { Swirl } = await import('~/composables/useAnimatedBackground')

  const canvas = document.createElement('canvas')
  canvas.width = 100
  canvas.height = 100
  const ctx = canvas.getContext('2d')!

  const swirl = new Swirl(100, 100)
  swirl.draw(ctx, 'rgba(139, 92, 246, OPACITY)')

  // Verify something was drawn (canvas not blank)
  const imageData = ctx.getImageData(0, 0, 100, 100)
  const hasPixels = imageData.data.some(value => value > 0)
  expect(hasPixels).toBe(true)
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: FAIL with "hasPixels is false" (draw method is empty)

**Step 3: Write implementation**

Update `draw` method in `app/composables/useAnimatedBackground.ts`:

```typescript
draw(ctx: CanvasRenderingContext2D, color: string): void {
  ctx.save()

  // Replace OPACITY placeholder with actual opacity
  const fillColor = color.replace('OPACITY', this.opacity.toString())

  // Create radial gradient for mystical glow effect
  const gradient = ctx.createRadialGradient(
    this.x, this.y, 0,
    this.x, this.y, this.size
  )
  gradient.addColorStop(0, fillColor)
  gradient.addColorStop(1, fillColor.replace(/[\d.]+\)$/, '0)')) // Fade to transparent

  // Draw circle with gradient
  ctx.fillStyle = gradient
  ctx.beginPath()
  ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2)
  ctx.fill()

  ctx.restore()
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: PASS (4 tests)

**Step 5: Commit**

```bash
git add app/composables/useAnimatedBackground.ts tests/composables/useAnimatedBackground.test.ts
git commit -m "feat: Add Swirl drawing with radial gradient

- Radial gradient creates mystical glow effect
- Opacity parameter support
- Gradient fades to transparent at edges
- 4 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Create Rune Particle Class (TDD)

**Files:**
- Modify: `app/composables/useAnimatedBackground.ts`
- Modify: `tests/composables/useAnimatedBackground.test.ts`

**Step 1: Write the failing test**

Add to `tests/composables/useAnimatedBackground.test.ts`:

```typescript
describe('Rune class', () => {
  it('initializes with random position and symbol', () => {
    const { Rune } = await import('~/composables/useAnimatedBackground')

    const rune = new Rune(1920, 1080)

    expect(rune.x).toBeGreaterThanOrEqual(0)
    expect(rune.x).toBeLessThanOrEqual(1920)
    expect(rune.y).toBeGreaterThanOrEqual(0)
    expect(rune.y).toBeLessThanOrEqual(1080)
    expect(rune.symbol).toBeDefined()
    expect(rune.symbol.length).toBeGreaterThan(0)
    expect(rune.size).toBeGreaterThanOrEqual(40)
    expect(rune.size).toBeLessThanOrEqual(80)
  })

  it('fades in and out over time', () => {
    const { Rune } = await import('~/composables/useAnimatedBackground')

    const rune = new Rune(100, 100)
    rune.opacity = 0
    rune.fadeDirection = 1 // Fading in

    rune.update(1000) // 1 second

    expect(rune.opacity).toBeGreaterThan(0)
  })

  it('rotates slowly over time', () => {
    const { Rune } = await import('~/composables/useAnimatedBackground')

    const rune = new Rune(100, 100)
    const initialRotation = rune.rotation

    rune.update(1000) // 1 second

    expect(rune.rotation).not.toBe(initialRotation)
  })

  it('repositions when fully faded out', () => {
    const { Rune } = await import('~/composables/useAnimatedBackground')

    const rune = new Rune(100, 100)
    rune.opacity = 0
    rune.fadeDirection = -1 // Fading out
    const initialX = rune.x
    const initialY = rune.y

    rune.update(16) // Trigger reposition check

    // When opacity is 0 and fading out, should flip to fading in
    expect(rune.fadeDirection).toBe(1)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: FAIL with "Cannot find export 'Rune'"

**Step 3: Write implementation**

Add to `app/composables/useAnimatedBackground.ts`:

```typescript
const RUNE_SYMBOLS = [
  'ᚠ', 'ᚢ', 'ᚦ', 'ᚨ', 'ᚱ', 'ᚲ',  // Norse runes
  '⚔', '✦', '◈', '⬡', '⬢', '⬣'     // Geometric symbols
]

export class Rune {
  x: number
  y: number
  symbol: string
  size: number
  opacity: number
  rotation: number
  fadeDirection: number
  private fadeSpeed: number
  private rotationSpeed: number
  private width: number
  private height: number

  constructor(width: number, height: number) {
    this.width = width
    this.height = height

    // Random position
    this.x = Math.random() * width
    this.y = Math.random() * height

    // Random symbol
    this.symbol = RUNE_SYMBOLS[Math.floor(Math.random() * RUNE_SYMBOLS.length)]

    // Random size (40-80px)
    this.size = 40 + Math.random() * 40

    // Start with random opacity (0-0.12)
    this.opacity = Math.random() * 0.12

    // Random initial rotation
    this.rotation = Math.random() * Math.PI * 2

    // Random fade direction (1 = in, -1 = out)
    this.fadeDirection = Math.random() > 0.5 ? 1 : -1

    // Fade speed (0.01-0.015 opacity change per second)
    this.fadeSpeed = 0.01 + Math.random() * 0.005

    // Rotation speed (0.1-0.3 degrees per second)
    this.rotationSpeed = (0.1 + Math.random() * 0.2) * (Math.PI / 180)
  }

  update(deltaTime: number): void {
    const dt = deltaTime / 1000

    // Update rotation
    this.rotation += this.rotationSpeed * dt

    // Update opacity (fade in/out)
    this.opacity += this.fadeDirection * this.fadeSpeed * dt

    // Clamp opacity and reverse direction at limits
    if (this.opacity >= 0.12) {
      this.opacity = 0.12
      this.fadeDirection = -1 // Start fading out
    } else if (this.opacity <= 0) {
      this.opacity = 0
      this.fadeDirection = 1 // Start fading in

      // Reposition when fully faded
      this.x = Math.random() * this.width
      this.y = Math.random() * this.height
      this.symbol = RUNE_SYMBOLS[Math.floor(Math.random() * RUNE_SYMBOLS.length)]
    }
  }

  draw(ctx: CanvasRenderingContext2D, color: string): void {
    // Will implement in next task
  }
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: PASS (8 tests)

**Step 5: Commit**

```bash
git add app/composables/useAnimatedBackground.ts tests/composables/useAnimatedBackground.test.ts
git commit -m "feat: Add Rune particle class with fade and rotation

- Random symbol selection from rune set
- Fade in/out behavior with opacity limits
- Slow rotation animation
- Repositions when fully faded out
- 8 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Add Rune Drawing Method (TDD)

**Files:**
- Modify: `app/composables/useAnimatedBackground.ts`
- Modify: `tests/composables/useAnimatedBackground.test.ts`

**Step 1: Write the failing test**

Add to `tests/composables/useAnimatedBackground.test.ts`:

```typescript
it('draws rune symbol with rotation', () => {
  const { Rune } = await import('~/composables/useAnimatedBackground')

  const canvas = document.createElement('canvas')
  canvas.width = 200
  canvas.height = 200
  const ctx = canvas.getContext('2d')!

  const rune = new Rune(200, 200)
  rune.opacity = 0.1 // Make visible
  rune.draw(ctx, 'rgba(79, 70, 229, OPACITY)')

  // Verify something was drawn
  const imageData = ctx.getImageData(0, 0, 200, 200)
  const hasPixels = imageData.data.some(value => value > 0)
  expect(hasPixels).toBe(true)
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: FAIL with "hasPixels is false"

**Step 3: Write implementation**

Update `draw` method in `Rune` class:

```typescript
draw(ctx: CanvasRenderingContext2D, color: string): void {
  if (this.opacity === 0) return // Skip if invisible

  ctx.save()

  // Move to rune position and rotate
  ctx.translate(this.x, this.y)
  ctx.rotate(this.rotation)

  // Replace OPACITY placeholder
  const fillColor = color.replace('OPACITY', this.opacity.toString())

  // Draw rune symbol
  ctx.fillStyle = fillColor
  ctx.font = `${this.size}px serif`
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillText(this.symbol, 0, 0)

  ctx.restore()
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: PASS (9 tests)

**Step 5: Commit**

```bash
git add app/composables/useAnimatedBackground.ts tests/composables/useAnimatedBackground.test.ts
git commit -m "feat: Add Rune drawing with rotation and opacity

- Rotated text rendering for rune symbols
- Opacity parameter support
- Skips drawing when invisible
- 9 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Create Animation Engine Composable (TDD)

**Files:**
- Modify: `app/composables/useAnimatedBackground.ts`
- Modify: `tests/composables/useAnimatedBackground.test.ts`

**Step 1: Write the failing test**

Add to `tests/composables/useAnimatedBackground.test.ts`:

```typescript
describe('useAnimatedBackground composable', () => {
  it('initializes particles', () => {
    const { useAnimatedBackground } = await import('~/composables/useAnimatedBackground')

    const canvas = document.createElement('canvas')
    canvas.width = 1920
    canvas.height = 1080
    const ctx = canvas.getContext('2d')!

    const { initialize } = useAnimatedBackground(canvas, false)
    initialize()

    // Should create particles (we'll add a getter for testing)
    const { getParticleCount } = useAnimatedBackground(canvas, false)
    expect(getParticleCount()).toBeGreaterThan(0)
  })

  it('starts animation loop', () => {
    const { useAnimatedBackground } = await import('~/composables/useAnimatedBackground')

    const canvas = document.createElement('canvas')
    const { initialize, start, isRunning } = useAnimatedBackground(canvas, false)

    initialize()
    start()

    expect(isRunning()).toBe(true)
  })

  it('stops animation loop', () => {
    const { useAnimatedBackground } = await import('~/composables/useAnimatedBackground')

    const canvas = document.createElement('canvas')
    const { initialize, start, stop, isRunning } = useAnimatedBackground(canvas, false)

    initialize()
    start()
    stop()

    expect(isRunning()).toBe(false)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: FAIL with "useAnimatedBackground is not a function"

**Step 3: Write implementation**

Add to `app/composables/useAnimatedBackground.ts`:

```typescript
interface ColorPalette {
  swirlColor: string
  runeColor: string
}

const LIGHT_MODE_COLORS: ColorPalette = {
  swirlColor: 'rgba(139, 92, 246, OPACITY)', // violet-500
  runeColor: 'rgba(79, 70, 229, OPACITY)'     // indigo-600
}

const DARK_MODE_COLORS: ColorPalette = {
  swirlColor: 'rgba(167, 139, 250, OPACITY)', // violet-400
  runeColor: 'rgba(34, 211, 238, OPACITY)'    // cyan-400
}

export function useAnimatedBackground(canvas: HTMLCanvasElement, isDark: boolean) {
  const ctx = canvas.getContext('2d')
  if (!ctx) throw new Error('Canvas context not available')

  let swirls: Swirl[] = []
  let runes: Rune[] = []
  let animationFrameId: number | null = null
  let lastTime = 0

  const colors = isDark ? DARK_MODE_COLORS : LIGHT_MODE_COLORS

  function initialize() {
    const width = canvas.width
    const height = canvas.height

    // Create 40 swirls
    swirls = Array.from({ length: 40 }, () => new Swirl(width, height))

    // Create 6 runes
    runes = Array.from({ length: 6 }, () => new Rune(width, height))
  }

  function animate(currentTime: number) {
    if (!lastTime) lastTime = currentTime
    const deltaTime = currentTime - lastTime
    lastTime = currentTime

    // Throttle to 30 FPS (33.33ms per frame)
    if (deltaTime < 33) {
      animationFrameId = requestAnimationFrame(animate)
      return
    }

    // Clear canvas
    ctx.clearRect(0, 0, canvas.width, canvas.height)

    // Update and draw swirls
    for (const swirl of swirls) {
      swirl.update(deltaTime)
      swirl.draw(ctx, colors.swirlColor)
    }

    // Update and draw runes
    for (const rune of runes) {
      rune.update(deltaTime)
      rune.draw(ctx, colors.runeColor)
    }

    // Continue animation
    animationFrameId = requestAnimationFrame(animate)
  }

  function start() {
    if (animationFrameId !== null) return // Already running
    lastTime = 0
    animationFrameId = requestAnimationFrame(animate)
  }

  function stop() {
    if (animationFrameId !== null) {
      cancelAnimationFrame(animationFrameId)
      animationFrameId = null
      lastTime = 0
    }
  }

  function isRunning() {
    return animationFrameId !== null
  }

  function getParticleCount() {
    return swirls.length + runes.length
  }

  return {
    initialize,
    start,
    stop,
    isRunning,
    getParticleCount
  }
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: PASS (12 tests)

**Step 5: Commit**

```bash
git add app/composables/useAnimatedBackground.ts tests/composables/useAnimatedBackground.test.ts
git commit -m "feat: Add animation engine composable

- Initializes 40 swirls and 6 runes
- 30 FPS throttled animation loop
- Start/stop controls
- Light/dark mode color palettes
- 12 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Add Visibility API and Reduced Motion Support (TDD)

**Files:**
- Modify: `app/composables/useAnimatedBackground.ts`
- Modify: `tests/composables/useAnimatedBackground.test.ts`

**Step 1: Write the failing test**

Add to `tests/composables/useAnimatedBackground.test.ts`:

```typescript
it('pauses animation when document becomes hidden', async () => {
  const { useAnimatedBackground } = await import('~/composables/useAnimatedBackground')

  const canvas = document.createElement('canvas')
  const { initialize, start, isRunning } = useAnimatedBackground(canvas, false)

  initialize()
  start()
  expect(isRunning()).toBe(true)

  // Simulate document visibility change
  Object.defineProperty(document, 'visibilityState', {
    writable: true,
    configurable: true,
    value: 'hidden'
  })
  document.dispatchEvent(new Event('visibilitychange'))

  // Wait for next tick
  await new Promise(resolve => setTimeout(resolve, 0))

  expect(isRunning()).toBe(false)
})

it('respects prefers-reduced-motion', () => {
  const { shouldAnimate } = await import('~/composables/useAnimatedBackground')

  // Mock matchMedia
  const mockMatchMedia = (matches: boolean) => ({
    matches,
    media: '(prefers-reduced-motion: reduce)',
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn()
  })

  window.matchMedia = vi.fn().mockReturnValue(mockMatchMedia(true))
  expect(shouldAnimate()).toBe(false)

  window.matchMedia = vi.fn().mockReturnValue(mockMatchMedia(false))
  expect(shouldAnimate()).toBe(true)
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: FAIL with "shouldAnimate is not a function"

**Step 3: Write implementation**

Add to `app/composables/useAnimatedBackground.ts`:

```typescript
export function shouldAnimate(): boolean {
  if (typeof window === 'undefined') return false

  const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)')
  return !mediaQuery.matches
}

export function useAnimatedBackground(canvas: HTMLCanvasElement, isDark: boolean) {
  const ctx = canvas.getContext('2d')
  if (!ctx) throw new Error('Canvas context not available')

  let swirls: Swirl[] = []
  let runes: Rune[] = []
  let animationFrameId: number | null = null
  let lastTime = 0

  const colors = isDark ? DARK_MODE_COLORS : LIGHT_MODE_COLORS

  // Visibility change handler
  function handleVisibilityChange() {
    if (document.visibilityState === 'hidden') {
      stop()
    } else {
      start()
    }
  }

  function initialize() {
    const width = canvas.width
    const height = canvas.height

    // Create 40 swirls
    swirls = Array.from({ length: 40 }, () => new Swirl(width, height))

    // Create 6 runes
    runes = Array.from({ length: 6 }, () => new Rune(width, height))

    // Listen for visibility changes
    if (typeof document !== 'undefined') {
      document.addEventListener('visibilitychange', handleVisibilityChange)
    }
  }

  function animate(currentTime: number) {
    if (!lastTime) lastTime = currentTime
    const deltaTime = currentTime - lastTime
    lastTime = currentTime

    // Throttle to 30 FPS (33.33ms per frame)
    if (deltaTime < 33) {
      animationFrameId = requestAnimationFrame(animate)
      return
    }

    // Clear canvas
    ctx.clearRect(0, 0, canvas.width, canvas.height)

    // Update and draw swirls
    for (const swirl of swirls) {
      swirl.update(deltaTime)
      swirl.draw(ctx, colors.swirlColor)
    }

    // Update and draw runes
    for (const rune of runes) {
      rune.update(deltaTime)
      rune.draw(ctx, colors.runeColor)
    }

    // Continue animation
    animationFrameId = requestAnimationFrame(animate)
  }

  function start() {
    if (animationFrameId !== null) return // Already running
    lastTime = 0
    animationFrameId = requestAnimationFrame(animate)
  }

  function stop() {
    if (animationFrameId !== null) {
      cancelAnimationFrame(animationFrameId)
      animationFrameId = null
      lastTime = 0
    }
  }

  function cleanup() {
    stop()
    if (typeof document !== 'undefined') {
      document.removeEventListener('visibilitychange', handleVisibilityChange)
    }
  }

  function isRunning() {
    return animationFrameId !== null
  }

  function getParticleCount() {
    return swirls.length + runes.length
  }

  return {
    initialize,
    start,
    stop,
    cleanup,
    isRunning,
    getParticleCount
  }
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- useAnimatedBackground`

Expected: PASS (14 tests)

**Step 5: Commit**

```bash
git add app/composables/useAnimatedBackground.ts tests/composables/useAnimatedBackground.test.ts
git commit -m "feat: Add visibility API and reduced motion support

- Pauses animation when tab hidden (battery friendly)
- Respects prefers-reduced-motion accessibility setting
- Cleanup method removes event listeners
- 14 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Create AnimatedBackground Component (TDD)

**Files:**
- Create: `app/components/AnimatedBackground.vue`
- Create: `tests/components/AnimatedBackground.test.ts`

**Step 1: Write the failing test**

Create `tests/components/AnimatedBackground.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import AnimatedBackground from '~/components/AnimatedBackground.vue'

describe('AnimatedBackground', () => {
  it('renders canvas element', async () => {
    const wrapper = await mountSuspended(AnimatedBackground)

    const canvas = wrapper.find('canvas')
    expect(canvas.exists()).toBe(true)
  })

  it('positions canvas behind content', async () => {
    const wrapper = await mountSuspended(AnimatedBackground)

    const canvas = wrapper.find('canvas')
    expect(canvas.classes()).toContain('fixed')
    expect(canvas.classes()).toContain('-z-10')
  })

  it('makes canvas non-interactive', async () => {
    const wrapper = await mountSuspended(AnimatedBackground)

    const canvas = wrapper.find('canvas')
    expect(canvas.classes()).toContain('pointer-events-none')
  })

  it('marks canvas as aria-hidden', async () => {
    const wrapper = await mountSuspended(AnimatedBackground)

    const canvas = wrapper.find('canvas')
    expect(canvas.attributes('aria-hidden')).toBe('true')
  })

  it('skips rendering when prefers-reduced-motion is true', async () => {
    // Mock shouldAnimate to return false
    vi.mock('~/composables/useAnimatedBackground', () => ({
      shouldAnimate: () => false,
      useAnimatedBackground: vi.fn()
    }))

    const wrapper = await mountSuspended(AnimatedBackground)

    // Canvas should not be rendered
    const canvas = wrapper.find('canvas')
    expect(canvas.exists()).toBe(false)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- AnimatedBackground`

Expected: FAIL with "Cannot find module '~/components/AnimatedBackground.vue'"

**Step 3: Write implementation**

Create `app/components/AnimatedBackground.vue`:

```vue
<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, watchEffect } from 'vue'
import { useColorMode } from '#imports'

const canvasRef = ref<HTMLCanvasElement | null>(null)
const colorMode = useColorMode()

let cleanup: (() => void) | null = null

// Check if animations should run
const animate = shouldAnimate()

// Handle window resize
function handleResize() {
  if (!canvasRef.value) return

  canvasRef.value.width = window.innerWidth
  canvasRef.value.height = window.innerHeight
}

onMounted(() => {
  if (!animate || !canvasRef.value) return

  // Set initial canvas size
  handleResize()

  // Listen for resize
  window.addEventListener('resize', handleResize)

  // Initialize animation
  const isDark = colorMode.value === 'dark'
  const { initialize, start, cleanup: cleanupFn } = useAnimatedBackground(canvasRef.value, isDark)

  initialize()
  start()
  cleanup = cleanupFn

  // Watch for color mode changes
  watchEffect(() => {
    if (!canvasRef.value) return

    // Reinitialize with new color mode
    if (cleanup) cleanup()

    const isDark = colorMode.value === 'dark'
    const { initialize, start, cleanup: cleanupFn } = useAnimatedBackground(canvasRef.value, isDark)

    initialize()
    start()
    cleanup = cleanupFn
  })
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize)
  if (cleanup) cleanup()
})
</script>

<template>
  <canvas
    v-if="animate"
    ref="canvasRef"
    class="fixed inset-0 -z-10 pointer-events-none"
    aria-hidden="true"
  />
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- AnimatedBackground`

Expected: PASS (5 tests)

**Step 5: Commit**

```bash
git add app/components/AnimatedBackground.vue tests/components/AnimatedBackground.test.ts
git commit -m "feat: Create AnimatedBackground component

- Canvas element with fixed positioning
- Window resize handling
- Color mode reactivity
- Prefers-reduced-motion support
- Lifecycle cleanup
- 5 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 8: Integrate Component into app.vue

**Files:**
- Modify: `app/app.vue`

**Step 1: Add component to app.vue**

Update `app/app.vue`:

```vue
<script setup lang="ts">
// Existing script content...
</script>

<template>
  <UApp>
    <div class="min-h-screen bg-gray-50 dark:bg-gray-950 text-gray-900 dark:text-gray-100">
      <!-- Animated background -->
      <ClientOnly>
        <AnimatedBackground />
      </ClientOnly>

      <!-- Navigation Bar -->
      <nav class="border-b border-gray-200 dark:border-gray-800 bg-white dark:bg-gray-900">
        <!-- Existing nav content... -->
      </nav>

      <!-- Main Content -->
      <UMain>
        <NuxtLayout>
          <NuxtPage />
        </NuxtLayout>
      </UMain>
    </div>
  </UApp>
</template>
```

**Step 2: Test in browser**

Run: `docker compose exec nuxt npm run dev`

Open: `http://localhost:3000`

Expected:
- See subtle purple/blue swirls floating across the background
- See faint runic symbols fading in/out
- Toggle dark mode → see colors change to brighter purple/cyan
- Switch tabs → animation should pause
- All existing functionality works normally

**Step 3: Verify full test suite**

Run: `docker compose exec nuxt npm run test`

Expected: All tests pass (645+ tests, including new ones)

**Step 4: Commit**

```bash
git add app/app.vue
git commit -m "feat: Integrate AnimatedBackground into app layout

- Added to app.vue with ClientOnly wrapper
- Positioned behind all content
- No impact on existing functionality
- Full test suite passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 9: Update Documentation

**Files:**
- Modify: `CHANGELOG.md`
- Modify: `docs/CURRENT_STATUS.md`

**Step 1: Update CHANGELOG.md**

Add to the `[Unreleased]` section in `CHANGELOG.md`:

```markdown
### Added
- Animated fantasy background with mystical swirls and D&D runes (2025-11-22)
  - Canvas-based particle system with 40 swirls and 6 runic symbols
  - 30 FPS performance-optimized animation
  - Respects prefers-reduced-motion accessibility setting
  - Pauses when tab not visible (battery friendly)
  - Light/dark mode color palettes
  - 19 new tests (composable + component)
```

**Step 2: Update CURRENT_STATUS.md**

Add to the "What's Complete and Working" section in `docs/CURRENT_STATUS.md`:

```markdown
### Animated Fantasy Background (NEW! ✨)
**Status:** ✅ Complete

**Visual Features:**
- 40 mystical energy swirls flowing across screen
- 6 D&D runic symbols fading in/out
- Organic movement with sine wave drift
- Light/dark mode color adaptation (purple/blue → purple/cyan)

**Performance:**
- 30 FPS throttled animation (battery efficient)
- Pauses when tab hidden (Visibility API)
- Respects prefers-reduced-motion accessibility setting
- <5% CPU usage on modern devices

**Technical:**
- Canvas 2D API with particle system
- `AnimatedBackground.vue` component + `useAnimatedBackground.ts` composable
- SSR compatible (ClientOnly wrapper)
- 19 new tests (all passing)

**Files:**
- Component: `app/components/AnimatedBackground.vue`
- Composable: `app/composables/useAnimatedBackground.ts`
- Tests: `tests/components/AnimatedBackground.test.ts` + `tests/composables/useAnimatedBackground.test.ts`
```

Update test count in header:
```markdown
**Test Coverage:** 664/664 tests passing (100% pass rate!) ✨
```

**Step 3: Commit documentation**

```bash
git add CHANGELOG.md docs/CURRENT_STATUS.md
git commit -m "docs: Update CHANGELOG and CURRENT_STATUS for animated background

- Added animated background to changelog
- Updated test count to 664
- Documented performance and accessibility features

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 10: Final Verification and Browser Testing

**Files:**
- N/A (manual testing)

**Step 1: Run full test suite**

Run: `docker compose exec nuxt npm run test`

Expected: 664/664 tests passing

**Step 2: Run ESLint**

Run: `docker compose exec nuxt npm run lint`

Expected: 0 errors

**Step 3: Run TypeScript type check**

Run: `docker compose exec nuxt npm run typecheck`

Expected: 13 errors (pre-existing, no new errors)

**Step 4: Manual browser testing checklist**

Test in browser (`http://localhost:3000`):

- [ ] Animation visible on load (subtle swirls + runes)
- [ ] Swirls moving smoothly (no jank)
- [ ] Runes fading in/out slowly
- [ ] Text remains readable on all pages
- [ ] Toggle dark mode → colors change immediately
- [ ] Switch browser tab → check DevTools (animation should pause)
- [ ] Return to tab → animation resumes
- [ ] Test all entity pages (spells, items, races, classes, backgrounds, feats)
- [ ] Test reference pages (no visual issues)
- [ ] Mobile responsive (test on narrow viewport)
- [ ] Enable "Reduce motion" in OS → animation should not appear
- [ ] Disable "Reduce motion" → animation appears again

**Step 5: Performance verification**

Open DevTools → Performance tab:

- [ ] Record 10 seconds of idle page
- [ ] CPU usage <5%
- [ ] FPS stable around 30
- [ ] No memory leaks (check Memory tab)

**Step 6: Create final verification commit**

```bash
git commit --allow-empty -m "test: Verify animated background feature complete

✅ 664/664 tests passing
✅ ESLint clean
✅ TypeScript 13 errors (unchanged)
✅ Browser tested on all pages
✅ Performance verified (<5% CPU, 30 FPS)
✅ Accessibility verified (prefers-reduced-motion)
✅ Dark/light mode transitions working

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Success Criteria

**All tasks complete when:**

- ✅ 664+ tests passing (100% pass rate)
- ✅ ESLint clean (0 errors)
- ✅ TypeScript stable (13 errors, no new ones)
- ✅ Animation visible and smooth in browser
- ✅ All pages tested (entity + reference)
- ✅ Dark mode toggle updates colors
- ✅ Tab switching pauses animation
- ✅ Prefers-reduced-motion respected
- ✅ Performance <5% CPU usage
- ✅ Documentation updated (CHANGELOG + CURRENT_STATUS)
- ✅ All commits created with proper messages

---

## Rollback Plan

If issues arise:

```bash
# View commits for this feature
git log --oneline --grep="animated background" --grep="Swirl" --grep="Rune"

# Revert all commits (replace <hash> with commit before feature)
git revert <hash>..HEAD

# Or hard reset (destructive)
git reset --hard <hash>
```

---

## Estimated Time

- Tasks 1-6: Composable + Tests (60-90 minutes)
- Task 7: Component + Tests (30 minutes)
- Task 8: Integration (15 minutes)
- Task 9: Documentation (15 minutes)
- Task 10: Testing (30 minutes)

**Total:** ~2.5-3 hours with TDD

---

**Implementation Status:** Ready to execute
**Created:** 2025-11-22
**Total Tasks:** 10
**Estimated Duration:** 2.5-3 hours
