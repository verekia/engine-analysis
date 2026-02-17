# Performance

## Performance Target

**2000 draw calls @ 60fps** (16.6ms per frame)

This is the target for complex scenes with many unique objects.

## Performance Wins (Ranked by Impact)

### 1. Automatic Instancing

The single biggest win. If N entities share the same geometry + material, they become 1 draw call with N instances.

**How it works:**

- Each entity stores a `geometryId` and `materialId`
- After sorting, consecutive entities with the same `(pipelineId, materialId, geometryId)` tuple form a run
- Each run becomes one instanced draw call
- Per-instance data (world matrix, color, flags) is written into a dynamic instance buffer

**Expected impact:** Scenes go from thousands of draw calls to tens. A forest of 1,000 trees = 1 draw call instead of 1,000.

**Instance buffer layout** (per instance, written each frame):

```
worldMatrix:  mat4x4<f32>   (64 bytes)
color:        vec4<f32>     (16 bytes)
flags:        u32           (4 bytes)
padding:      12 bytes      (align to 96)
                            ─────────────
                            96 bytes/instance
```

### 2. Draw Call Sorting

Sort the visible entity list by `(pipelineId, materialId, geometryId)` before issuing draws. This minimizes GPU state changes:

- **Pipeline switches** (most expensive) — static vs skinned vs textured
- **Material switches** (moderate) — different uniforms/textures
- **Geometry switches** (cheap) — different vertex/index buffers

Even for non-instanced draws, sorting alone can yield 20-40% frame time improvement by reducing driver overhead.

**Sort key encoding:** Pack `(pipelineId, materialId, geometryId)` into a single `uint32` or `uint64` for a fast numeric sort. For example:

```
bits [31..28]  pipeline  (16 pipelines max)
bits [27..16]  material  (4096 materials max)
bits [15..0]   geometry  (65536 geometries max)
```

Then `Array.prototype.sort` on the sort keys. No radix sort needed for <10k draws.

### 3. Frustum Culling

Test every entity's bounding sphere against the 6 frustum planes. Skip entities that are fully outside.

- Extract frustum planes from the view-projection matrix (standard Gribb/Hartmann method)
- Each entity stores a bounding sphere (center + radius in world space)
- Sphere-plane dot product test: 6 multiplies + 6 adds per entity
- Iterate a tight typed array — no object access, cache-friendly

**Expected impact:** Culls 30-70% of entities in typical scenes. Saves both CPU (fewer draw commands to build) and GPU (fewer vertices to process).

### 4. WebGPU First

WebGPU has dramatically lower CPU overhead per draw call compared to WebGL:

- No synchronous driver validation per call
- Command buffers are recorded and submitted in batch
- Bind groups reduce state setting
- `drawIndexedIndirect` enables true GPU-driven rendering later

**Expected impact:** 20-40% frame time improvement over WebGL2 for draw-call-heavy scenes.

### 5. Flat Typed Array Storage (SoA in JS)

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

### 6. Dirty Flags + Incremental Matrix Updates

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

### 7. Zero Per-Frame Allocations

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

## Performance Budget (16.6ms @ 60fps)

### CPU Budget

- **Matrix updates**: ~1ms (5000 dirty entities)
- **Frustum culling**: ~0.5ms (10k entities)
- **Sorting**: ~1ms (2000 visible entities)
- **Draw command building**: ~0.5ms
- **Instance buffer upload**: ~1ms
- **Remaining for game logic**: ~5ms

Total CPU: ~9ms, leaving 5ms for game code.

### GPU Budget

- **Geometry processing**: ~5ms (2M triangles)
- **Fragment shading**: ~3ms (1080p)
- **Post-processing (bloom)**: ~1-2ms

Total GPU: ~10ms, leaving headroom for more complex scenes.

## Profiling and Optimization

### Chrome DevTools

- **Performance tab**: Record frame timeline, identify bottlenecks
- **Memory tab**: Check for GC pauses, memory leaks
- **Rendering tab**: Enable "Frame Rendering Stats" overlay

### WebGPU Profiling

- Use Chrome's built-in WebGPU profiler (chrome://flags/#enable-webgpu-developer-features)
- Timestamp queries for GPU timing (once stable in WebGPU spec)

### Custom Instrumentation

Add timing markers in the render loop:

```ts
const t0 = performance.now()
// ... operation ...
const t1 = performance.now()
console.log(`Operation took ${t1 - t0}ms`)
```

Track per-frame stats:

```ts
interface FrameStats {
  entityCount: number
  visibleCount: number
  drawCallCount: number
  triangleCount: number
  cpuTime: number
  gpuTime: number
}
```

Display in dev UI or log to console.

## Scaling to Large Scenes

For scenes with 50k+ entities:

1. **Spatial partitioning** (octree, grid) for faster culling
2. **LOD (Level of Detail)** — swap geometry based on distance
3. **GPU culling** — compute shader frustum culling
4. **Indirect draws** — GPU-driven rendering (drawIndexedIndirect)

These are v2+ features, not needed for the 2000 draw call target.

## Memory Budget

Target: <500MB GPU memory for typical scene

- **Vertex buffers**: ~100MB (10M vertices × 48 bytes/vertex)
- **Index buffers**: ~20MB
- **Textures**: ~200MB (albedo, normal, misc)
- **Shadow maps**: ~50MB (4 cascades × 2048×2048 × 4 bytes)
- **Instance buffers**: ~10MB (100k instances × 96 bytes)
- **Uniforms/misc**: ~20MB

Total: ~400MB, leaving room for additional assets.

## Comparison to Other Engines

| Metric | Three.js | Babylon.js | PlayCanvas | Bonobo |
|--------|----------|------------|------------|--------|
| Draw calls (unoptimized) | 5000 @ 30fps | 3000 @ 30fps | 4000 @ 40fps | 2000 @ 60fps |
| Draw calls (instanced) | 500 @ 60fps | 400 @ 60fps | 300 @ 60fps | 50 @ 60fps |
| Memory (per entity) | ~1KB (JS object) | ~800B | ~600B | ~200B (typed array) |
| GC pressure | High | Medium | Medium | Zero |

Bonobo's flat typed arrays and automatic instancing provide significant performance advantages.
