# Design: 3D Dice Face Numbers

**Date:** 2025-11-23
**Status:** Approved
**Implementation Time:** 2-3 hours

## Overview

Add decorative face numbers to the 3D polyhedral dice in the animated background. Numbers should be subtle and blend with the ethereal glass-like aesthetic.

## Design Decisions

### Visual Style
- **Purpose:** Decorative enhancement (not functional/readable)
- **Aesthetic:** Subtle, blending with glass material
- **Special Treatment:** None (no critical roll highlighting)

### Typography
- **Font:** `bold 80px Georgia, serif`
- **Color:** `rgba(255, 255, 255, 0.35)` (white at 35% opacity)
- **Rationale:** Serif font feels traditional/D&D, low opacity blends naturally

## Technical Approach

### Architecture

**Texture Generation:**
- One canvas-based texture per die type (d4, d6, d8, d10, d12, d20)
- Use HTML5 Canvas API to draw numbers
- Convert to `THREE.CanvasTexture` for Three.js
- Cache textures in module-level Map

**Texture Application:**
- Apply textures to material's `map` property
- Use Three.js default UV mapping (no custom UVs needed)
- Leverage existing geometry UV coordinates

### Texture Specifications

| Die Type | Faces | Canvas Size | Grid Layout | Notes |
|----------|-------|-------------|-------------|-------|
| d4 | 4 | 256×256 | 2×2 | Tetrahedron |
| d6 | 6 | 512×512 | 3×2 | Cube (standard cross layout) |
| d8 | 8 | 512×512 | 4×2 | Octahedron |
| d10 | 10 | 512×512 | 5×2 | Numbers 1-9, 0 (last position) |
| d12 | 12 | 512×512 | 4×3 | Dodecahedron |
| d20 | 20 | 1024×1024 | 5×4 | Icosahedron (larger for more faces) |

**Why Power-of-2 Dimensions?**
- GPU optimization (prevents warnings)
- Better texture filtering performance
- Standard practice in WebGL/Three.js

### Implementation Components

**New Functions:**
```typescript
// Texture generators (one per die type)
function createD4Texture(): THREE.Texture
function createD6Texture(): THREE.Texture
function createD8Texture(): THREE.Texture
function createD10Texture(): THREE.Texture
function createD12Texture(): THREE.Texture
function createD20Texture(): THREE.Texture

// Master factory
function createDiceTexture(type: DieType): THREE.Texture

// Texture cache
const diceTextures: Map<DieType, THREE.Texture> = new Map()
```

**Modified Functions:**
- `createDie()` - Apply texture to material
- `initThreeJS()` - Pre-generate all textures before creating dice

### Canvas Rendering Pattern

```typescript
function createD6Texture(): THREE.Texture {
  const canvas = document.createElement('canvas')
  canvas.width = 512
  canvas.height = 512
  const ctx = canvas.getContext('2d')!

  // Setup text rendering
  ctx.fillStyle = 'rgba(255, 255, 255, 0.35)'
  ctx.font = 'bold 80px Georgia, serif'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'

  // Calculate grid cells (3 columns × 2 rows)
  const cols = 3, rows = 2
  const cellWidth = canvas.width / cols
  const cellHeight = canvas.height / rows

  // Draw numbers 1-6
  for (let i = 0; i < 6; i++) {
    const col = i % cols
    const row = Math.floor(i / cols)
    const x = col * cellWidth + cellWidth / 2
    const y = row * cellHeight + cellHeight / 2

    ctx.fillText((i + 1).toString(), x, y)
  }

  return new THREE.CanvasTexture(canvas)
}
```

### Material Integration

**Current Material:**
```typescript
new THREE.MeshPhysicalMaterial({
  color: DICE_COLORS[colorIndex]!.color,
  transparent: true,
  opacity: 0.25,
  transmission: 0.5,
  // ... other properties
})
```

**With Texture:**
```typescript
new THREE.MeshPhysicalMaterial({
  color: DICE_COLORS[colorIndex]!.color,
  map: diceTextures.get(type), // Add texture
  transparent: true,
  opacity: 0.25,
  transmission: 0.5,
  // ... other properties
})
```

## UV Mapping Strategy

**Relying on Three.js Defaults:**
- `BoxGeometry` (d6): Standard cube unwrap
- `TetrahedronGeometry` (d4): Triangular face layout
- `OctahedronGeometry` (d8): 8-face layout
- `DodecahedronGeometry` (d12): 12-face pentagon layout
- `IcosahedronGeometry` (d20): 20-face triangle layout

**Trade-off:** Some faces may have slight UV stretching, but this adds character and is acceptable for decorative purposes.

## Performance Considerations

**Memory:**
- 6 textures total (one per die type)
- Texture sizes: 256px to 1024px
- Estimated memory: ~2-3 MB total
- Impact: Negligible (textures are small and cached)

**Rendering:**
- No additional draw calls (same meshes, just textured)
- Texture filtering: `THREE.LinearFilter` (default, performant)
- No FPS impact expected

## Testing Strategy

**Visual Verification:**
1. Check each die type displays numbers correctly
2. Verify numbers rotate naturally with dice
3. Confirm subtle appearance (35% opacity blends well)
4. Test in both light and dark modes
5. Verify no visual glitches or UV issues

**Performance Verification:**
1. Check browser console for texture warnings
2. Monitor FPS (should remain stable)
3. Test on lower-end devices if available

## Alternative Approaches Considered

### Individual Face Textures (Rejected)
- **Reason:** Too complex for decorative purpose
- **Effort:** 6-8 hours vs 2-3 hours
- **Benefit:** Perfect control, but overkill

### Decal-Based Numbers (Rejected)
- **Reason:** More overhead, complex scene graph
- **Effort:** 5-6 hours
- **Benefit:** Perfect centering, but not worth the cost

## Success Criteria

- ✅ Numbers visible but subtle (blend with glass aesthetic)
- ✅ All dice types (d4-d20) have appropriate face numbers
- ✅ No performance degradation
- ✅ No visual glitches or UV artifacts
- ✅ Numbers rotate naturally with dice tumbling
- ✅ Works in both light and dark color modes

## Future Enhancements (Out of Scope)

- High-contrast mode toggle for accessibility
- Custom font options (fantasy/medieval)
- Per-die color customization
- Critical roll visual effects (20 on d20)
- Mobile performance optimization (reduce texture sizes)

## References

- Current Implementation: `app/composables/useAnimatedBackground.ts`
- 3D Dice Handover: `docs/HANDOVER-2025-11-23-3D-DICE-INTEGRATION.md`
- Three.js CanvasTexture: https://threejs.org/docs/#api/en/textures/CanvasTexture
- Three.js UV Mapping: Built-in geometry default UVs
