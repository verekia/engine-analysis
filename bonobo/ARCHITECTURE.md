# Architecture

## High-Level Architecture

```
┌─ React Layer ──────────────────────────────────────────────────────┐
│                                                                    │
│  <Canvas>              Thin React wrapper. Components call         │
│    <Mesh />            world.spawn() / world.despawn() via         │
│    <Camera />          useEffect. Props sync via refs.             │
│    <Light />           No custom reconciler.                       │
│  </Canvas>                                                         │
│                                                                    │
└───────────────────────────┬────────────────────────────────────────┘
                            │
┌─ World ───────────────────┴────────────────────────────────────────┐
│                                                                    │
│  Entity storage      Flat typed arrays (SoA layout in JS)          │
│  Material registry   Map<materialId, MaterialDesc>                 │
│  Geometry registry   Map<geometryId, GPUBufferHandles>             │
│  Render list         Sorted draw commands, auto-instanced          │
│                                                                    │
└───────────────────────────┬────────────────────────────────────────┘
                            │
┌─ Render Loop ─────────────┴────────────────────────────────────────┐
│                                                                    │
│  1. Recompute dirty matrices (only flagged entities)               │
│  2. Frustum cull → visible list                                    │
│  3. Sort visible by pipeline → material → geometry                 │
│  4. Merge same-geometry+material runs into instanced draws         │
│  5. Upload instance buffers + submit GPU draw calls                │
│  6. Post-processing (bloom, etc.)                                  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

## Core Design Decisions

### Flat Typed Array Storage (SoA in JS)

All per-entity data lives in flat `Float32Array` / `Uint32Array` buffers:

```ts
// Pre-allocated for MAX_ENTITIES (e.g. 50,000)
positions:      Float32Array(MAX * 3)
rotations:      Float32Array(MAX * 3)     // Euler XYZ
scales:         Float32Array(MAX * 3)
worldMatrices:  Float32Array(MAX * 16)
colors:         Float32Array(MAX * 4)     // RGBA
flags:          Uint32Array(MAX)
boundingSpheres: Float32Array(MAX * 4)    // cx, cy, cz, radius
geometryIds:    Uint32Array(MAX)
materialIds:    Uint32Array(MAX)
sortKeys:       Uint32Array(MAX)
```

**Why this matters:**

- **Zero GC** — no per-entity objects to collect
- **Cache-friendly** — iterating `flags` to find dirty entities reads contiguous memory
- **GPU-uploadable** — `worldMatrices` can be copied directly into a GPU buffer
- **Debuggable** — it's just TypeScript, inspect in devtools

### Entity System

#### Spawn / Despawn

```ts
// Internally: finds a free slot, writes defaults into typed arrays
const entity = world.spawn({
  geometry: boxGeo,
  material: redMat,
  position: [0, 1, 0],
})

// Internally: marks slot as free, adds to freelist
world.despawn(entity)
```

Entities are integer IDs (indices into the typed arrays). A freelist tracks available slots to avoid fragmentation.

#### Entity Handle

```ts
interface Entity {
  id: number                    // index into SoA arrays
  setPosition(x, y, z): void   // writes to positions[], sets dirty flag
  setRotation(x, y, z): void
  setScale(x, y, z): void
  setColor(r, g, b, a): void
  setFlag(flag: number): void
  destroy(): void               // calls world.despawn
}
```

Each setter writes directly into the typed array at the correct offset and sets the dirty flag. No intermediate objects.

### Dirty Flags + Incremental Matrix Updates

Only recompute TRS → world matrix for entities that changed:

```ts
for (let i = 0; i < entityCount; i++) {
  if (flags[i] & DIRTY) {
    computeWorldMatrix(i)     // reads positions/rotations/scales, writes worldMatrices
    flags[i] &= ~DIRTY        // clear dirty bit
  }
}
```

Three.js calls `updateMatrixWorld()` on the entire scene tree every frame. With dirty flags, a static scene of 10,000 entities where 5 move = 5 matrix computations, not 10,000.

### Zero Per-Frame Allocations

All math uses pre-allocated scratch variables:

```ts
// Module-level scratch — never allocate in the render loop
const _mat4A = new Float32Array(16)
const _mat4B = new Float32Array(16)
const _vec3A = new Float32Array(3)
const _vec3B = new Float32Array(3)
const _frustumPlanes = new Float32Array(24) // 6 planes × 4 floats
```

No `new Vector3()`, no `new Matrix4()`, no temp arrays. At 144fps you get 6.9ms per frame — GC pauses of even 1-2ms are visible as hitches.
