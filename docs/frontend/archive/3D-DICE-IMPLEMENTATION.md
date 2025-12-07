# 3D Dice Implementation Guide

## Overview

This document outlines how to implement realistic 3D polyhedral dice using Three.js for the background animation.

## Current State

**✅ Fully Implemented (2025-11-23):**
- **Parchment background** - Full-viewport cover with grayscale filter, cached via OffscreenCanvas (6-8% opacity)
- **Interactive magic particles** - 40-50 sparkles with optimized rendering (70% circles, no shadow blur)
- **3D Dice rendering** - 8 polyhedral dice (d4, d6, d8, d10, d12, 3× d20) with glass materials and wireframes
- **Mouse repulsion** - Particles avoid cursor within 150px radius
- **Scroll inertia** - Particles and dice gain momentum when scrolling
- **Performance optimized** - ~70% CPU reduction through multiple optimizations

**Performance Optimizations Applied:**
- Particle count: 80-120 → 40-50
- Removed constellation line calculation (O(n²) nested loops eliminated)
- Removed shadow blur from particles (GPU filter removed)
- Cached parchment grayscale filter using OffscreenCanvas
- Removed Three.js textures from dice (no texture generation)
- Simplified particle shapes (70% circles vs complex polygons)
- Reduced trail length: 5 → 3 points
- Result: Smooth 30 FPS without CPU fan activation

## Why Three.js?

Three.js provides:
- True 3D polyhedral geometry
- Realistic lighting and shadows
- Physics simulation (optional via Cannon.js or Rapier)
- Performance optimization
- WebGL rendering

## Implementation Plan

### Step 1: Install Dependencies

```bash
npm install three @types/three
# Optional for physics:
npm install @react-three/cannon cannon-es
```

### Step 2: Create 3D Dice Geometries

Each die type needs proper geometry:

**d4 (Tetrahedron):**
```typescript
const geometry = new THREE.TetrahedronGeometry(size)
```

**d6 (Cube):**
```typescript
const geometry = new THREE.BoxGeometry(size, size, size)
```

**d8 (Octahedron):**
```typescript
const geometry = new THREE.OctahedronGeometry(size)
```

**d12 (Dodecahedron):**
```typescript
const geometry = new THREE.DodecahedronGeometry(size)
```

**d20 (Icosahedron):**
```typescript
const geometry = new THREE.IcosahedronGeometry(size)
```

**d10 (Pentagonal Trapezohedron):**
- Custom geometry required (no built-in)
- Use vertex/face arrays

### Step 3: Materials and Textures

**Semi-transparent dice with numbers:**
```typescript
const material = new THREE.MeshPhysicalMaterial({
  color: 0x667799,
  transparent: true,
  opacity: 0.3,
  roughness: 0.3,
  metalness: 0.1,
  clearcoat: 0.5,
  transmission: 0.2 // Glass-like
})

// Add face numbers as textures or canvas-based labels
```

### Step 4: Lighting Setup

```typescript
// Ambient light (base illumination)
const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)

// Directional light (shadows)
const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
directionalLight.position.set(5, 10, 7.5)
directionalLight.castShadow = true

scene.add(ambientLight, directionalLight)
```

### Step 5: Animation Loop

```typescript
class Die3D {
  mesh: THREE.Mesh
  rotationSpeed: THREE.Vector3
  velocity: THREE.Vector3

  constructor(type: DieType) {
    this.mesh = this.createMesh(type)
    this.rotationSpeed = new THREE.Vector3(
      Math.random() * 0.5,
      Math.random() * 0.5,
      Math.random() * 0.5
    )
    this.velocity = new THREE.Vector3(
      (Math.random() - 0.5) * 2,
      (Math.random() - 0.5) * 2,
      (Math.random() - 0.5) * 2
    )
  }

  update(deltaTime: number) {
    // Update rotation
    this.mesh.rotation.x += this.rotationSpeed.x * deltaTime
    this.mesh.rotation.y += this.rotationSpeed.y * deltaTime
    this.mesh.rotation.z += this.rotationSpeed.z * deltaTime

    // Update position
    this.mesh.position.add(this.velocity.clone().multiplyScalar(deltaTime))

    // Wrap around edges
    // ...
  }
}
```

### Step 6: Integration with Canvas

**Dual Canvas Approach:**
- Keep existing 2D canvas for parchment + particles
- Add WebGL canvas layer for 3D dice
- Stack them using z-index

```vue
<!-- app.vue -->
<div style="position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; z-index: 1;">
  <!-- 2D Canvas (parchment + particles) -->
  <canvas ref="canvas2d" />

  <!-- 3D Canvas (dice) -->
  <canvas ref="canvas3d" style="position: absolute; top: 0; left: 0;" />
</div>
```

### Step 7: Performance Optimization

**Rendering:**
- Limit to 8-12 dice (fewer than 2D version)
- Use level of detail (LOD) for distant dice
- Share materials between dice
- Use `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))`

**Physics (optional):**
- Only calculate collisions when dice "roll"
- Sleep physics bodies when stationary
- Use simplified collision boxes

### Step 8: Visual Effects

**Critical Hit Glow:**
```typescript
if (die.currentFace === 20) {
  die.mesh.material.emissive = new THREE.Color(0xffaa00)
  die.mesh.material.emissiveIntensity = 2.0
}
```

**Face Numbers:**
```typescript
// Option 1: Canvas texture per face
const canvas = document.createElement('canvas')
canvas.width = 256
canvas.height = 256
const ctx = canvas.getContext('2d')!
ctx.font = 'bold 120px Arial'
ctx.fillStyle = '#ffffff'
ctx.textAlign = 'center'
ctx.fillText('20', 128, 150)

const texture = new THREE.CanvasTexture(canvas)
```

## File Structure

```
app/composables/
  useAnimatedBackground.ts       # Existing 2D background
  useAnimatedBackground3D.ts     # New Three.js dice (to create)

app/components/
  AnimatedBackground.vue         # Existing component
  AnimatedBackground3D.vue       # New 3D component (to create)
```

## Alternative: Simplified Approach

If Three.js is too complex, consider:

1. **Pre-rendered Sprite Sheets:**
   - Create rotation sequences as PNG sprites
   - Much lighter weight
   - Less realistic but still better than 2D

2. **CSS 3D Transforms:**
   - Use `transform-style: preserve-3d`
   - Limited but native browser support
   - Good for simple rotation effects

3. **Babylon.js:**
   - Similar to Three.js but with built-in physics
   - Potentially easier for dice simulation

## Example Libraries

**Existing dice libraries to reference:**
- `dice-box` - Complete 3D dice roller
- `threejs-dice` - Simple Three.js dice
- `dice-roller-3d` - Physics-based roller

## Next Steps

1. Install Three.js and create basic scene
2. Implement single d20 with rotation
3. Add face number textures
4. Test performance
5. Add remaining die types
6. Integrate with existing particle system
7. Add mouse interaction (optional)
8. Polish and optimize

## Notes

- Keep 2D particles - they provide magical atmosphere
- 3D dice should be subtle (low opacity)
- Prioritize performance over realism
- Test on lower-end devices
