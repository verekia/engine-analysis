# Architecture

## Overview

Fennec uses a layered architecture where each layer depends only on layers below it. The hot rendering path is designed as a flat, data-oriented pipeline that minimizes JavaScript overhead and GC pressure.

```
┌─────────────────────────────────────────────┐
│              fennec-react                    │  React reconciler, hooks, JSX
├─────────────────────────────────────────────┤
│  fennec-controls  │  fennec-overlay          │  Input handling, HTML overlay
├───────────────────┴─────────────────────────┤
│  fennec-loaders   │  fennec-animation        │  Asset loading, skeletal anim
├───────────────────┴─────────────────────────┤
│              fennec-spatial                   │  BVH, raycasting, culling
├─────────────────────────────────────────────┤
│              fennec-materials                 │  Shaders, materials, textures
├─────────────────────────────────────────────┤
│              fennec-core                      │  Renderer, scene graph, math
│  ┌────────────────┬────────────────────┐    │
│  │  WebGPU Backend │  WebGL2 Backend    │    │  GPU Abstraction Layer
│  └────────────────┴────────────────────┘    │
└─────────────────────────────────────────────┘
```

## Module Dependency Graph

```
fennec-core          ← no deps (ships own math)
fennec-materials     ← fennec-core
fennec-spatial       ← fennec-core
fennec-loaders       ← fennec-core, fennec-materials, fennec-spatial
fennec-animation     ← fennec-core
fennec-controls      ← fennec-core
fennec-overlay       ← fennec-core
fennec-react         ← all of the above
```

## Core Data Flow (Per Frame)

```
1. Input Phase
   ├── OrbitControls update camera from pointer/touch
   ├── AnimationMixer advance time, compute bone matrices
   └── User callbacks (useFrame hooks)

2. Scene Graph Update Phase
   ├── Walk dirty nodes, recompute world matrices
   ├── Update skinned mesh bone texture/UBO
   └── Update bounding volumes for moved objects

3. Culling Phase
   ├── Frustum cull all root-level objects (AABB vs frustum planes)
   └── Output: visible object list

4. Sort Phase
   ├── Separate opaque vs transparent
   ├── Opaque: sort by (shaderID, materialID, geometryID) — front to back
   └── Transparent: no sort needed (OIT)

5. Shadow Phase
   ├── For each CSM cascade:
   │   ├── Compute cascade frustum + light matrix
   │   ├── Cull objects against cascade frustum
   │   └── Render depth-only pass
   └── Output: shadow map atlas

6. Render Phase
   ├── Bind shared UBO (camera, lights, shadow matrices)
   ├── Opaque pass (sorted draw calls)
   ├── OIT accumulation pass (transparent objects)
   ├── OIT composite pass
   └── Output: color + depth in MSAA render target

7. Post-Processing Phase
   ├── MSAA resolve
   ├── Bloom threshold extract
   ├── Bloom blur (Dual Kawase, 5-6 iterations)
   ├── Bloom composite
   └── Final blit to canvas

8. Overlay Phase
   └── Project 3D positions → screen coords, update DOM elements
```

## Coordinate System

**Z-up, right-handed.** X = right, Y = forward, Z = up.

```
        Z (up)
        │
        │
        │
        └───── Y (forward)
       /
      /
     X (right)
```

All internal math, scene graph transforms, and physics use this convention. GLTF files (Y-up) are converted during import by swapping Y↔Z and adjusting rotations.

## Math Library

Fennec ships its own minimal math library rather than depending on gl-matrix or similar. This avoids:
- Extra dependency weight
- Impedance mismatch with our coordinate system
- Allocation patterns we can't control

Types:
- `Vec2`, `Vec3`, `Vec4` — backed by `Float32Array(2|3|4)`
- `Mat3`, `Mat4` — backed by `Float32Array(9|16)`, column-major
- `Quat` — backed by `Float32Array(4)`, `(x, y, z, w)` layout
- `AABB` — `{ min: Vec3, max: Vec3 }`
- `Ray` — `{ origin: Vec3, direction: Vec3 }`
- `Frustum` — 6 planes as `Vec4` (nx, ny, nz, d)

All math operations support both allocation-free (output parameter) and convenience (returns new) forms:

```typescript
// Allocation-free (hot path)
vec3.add(out, a, b)

// Convenience (setup code)
const result = Vec3.add(a, b)
```

## Object Identity & Incremental IDs

Every engine object gets a monotonically increasing integer ID. This is used for:
- Sort keys (bit-pack shaderID + materialID + geometryID into a single uint64-safe number)
- Fast Map/Set lookups (numeric keys)
- Cache invalidation

```typescript
let _nextId = 0
const generateId = (): number => _nextId++
```

## Memory Management Strategy

1. **Pre-allocated typed arrays** for all per-frame data (transform arrays, sort keys, visible lists)
2. **Object pools** for frequently created/destroyed objects (particles, projectiles)
3. **No closures in hot paths** — all callbacks are method references
4. **Flat arrays over linked structures** — BVH nodes, render lists, animation tracks
5. **Double-buffered uniform data** — write to staging, upload once per frame

## Error Handling

- Debug mode (`FENNEC_DEBUG`): verbose error messages, WebGL/WebGPU validation, shader compilation errors with source context
- Production mode: silent fallbacks, no string allocations in hot paths
- `FENNEC_DEBUG` is stripped via dead code elimination in production builds

## Build & Distribution

- Bundled with Rollup (ESM + CJS)
- Tree-shakeable: each package is independently importable
- WebGPU backend is dynamically imported (not loaded if WebGL2 is used)
- Draco and Basis WASM decoders loaded on-demand via Web Workers
- Total core bundle target: < 40KB gzipped (without loaders/workers)
