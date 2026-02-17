# PERFORMANCE — Final Decision

## Target

**2000 draw calls at 60fps on recent mobile phones** (universal agreement)

16.6ms frame budget. Per-draw JS overhead target: <2-3μs.

## Performance Wins (Ranked by Impact)

### 1. Draw Call Sorting (Highest Impact)

**Decision: 64-bit radix sort by pipeline > material > depth** (Mantis, Shark, Wren approach)

State changes are the dominant CPU-side cost:
- Pipeline (shader) switches: ~50μs each on mobile
- Material/texture binds: ~10μs each
- Buffer offset changes: ~1μs each

With 2000 draws and 20 unique pipelines, unsorted submission causes ~200 pipeline switches. Sorted: ~20. That is ~9ms saved on mobile.

State change reduction with sorting: **90-99%** (from ~2000 to ~10-200).

### 2. Frustum Culling

**Decision: AABB P-vertex test against 6 frustum planes** (universal agreement)

Culling invisible objects prevents wasted GPU work. Typical culling ratio: 30-70% of objects invisible per frame.

For 2000 objects, brute-force AABB testing: ~0.1-0.2ms. BVH-accelerated culling available for >5000 objects via the scene BVH.

### 3. WebGPU-First

**Decision: WebGPU primary, WebGL2 fallback** (universal agreement)

WebGPU has lower driver overhead: immutable pipelines (no per-draw state validation), render bundles for static geometry, better multi-threaded potential.

On devices with WebGPU, draw call submission is ~30-50% cheaper than WebGL2.

### 4. Dirty Flags for Incremental Updates

**Decision: Update world matrices only for dirty nodes** (universal agreement)

Typical frame: 2-5% of nodes have moved. Dirty flags skip 95%+ of matrix recomputations. ~40× fewer computations than "update everything."

### 5. State Cache (WebGL2)

**Decision: Full GL state tracking** (8/9 agree)

Check state before every GL call, skip if unchanged. Eliminates 40-60% of redundant API calls when draws are sorted. Essential for WebGL2 performance parity with WebGPU.

### 6. Zero Per-Frame Allocations

**Decision: No `new`, no array creation, no closures in hot paths** (universal agreement)

- Pre-allocate all math scratch variables at module scope
- Frame-scoped pool with pointer reset (not deallocation)
- Pre-allocate sort key arrays, draw command buffers, traversal stacks
- Reuse typed arrays across frames
- Target: zero GC collections triggered by the render loop

### 7. Shader Warm-up

**Decision: Pre-compile common shader variants during loading** (Caracal approach)

Pre-compile during asset loading to prevent frame hitches:
- Lambert opaque + shadow + texture
- Lambert opaque + shadow + vertex colors
- Lambert transparent
- Basic opaque
- Shadow depth-only

## No Automatic Instancing

**Decision: Skip instancing** (8/9 agree)

When per-draw overhead is <2-3μs, 2000 unique draws is feasible without instancing. Instancing adds complexity (instance buffers, batching logic, material limits) without meaningful benefit at this scale.

## Uniform Upload Strategy

**Decision: Dynamic offsets on a shared buffer** (universal agreement)

Per-object data (world matrix, material overrides): 128-256 bytes per object.
Total per-frame upload: 2000 × 256 = ~512KB via single buffer write.
Dynamic offsets avoid per-object bind group creation.

## Render Bundles (WebGPU)

**Decision: Available for static geometry, not primary optimization** (6/9 approach)

Typical split: 1800 static + 200 dynamic objects. Static objects recorded into a render bundle, replayed with near-zero JS cost. Dynamic objects use normal draw submission.

Not the primary strategy — sort-based rendering achieves the target on both backends.

## Data Layout

**Decision: SoA for hot data, OOP wrappers for public API** (universal agreement)

```typescript
// Internal: contiguous arrays for cache-friendly iteration
worldMatrices: Float32Array(maxObjects * 16)
aabbPool: Float32Array(maxObjects * 6)
visibilityFlags: Uint8Array(maxObjects)
sortKeys: BigUint64Array(maxObjects)

// Public: wrapper objects reference into arrays by index
mesh.position.set(1, 2, 3)  // Writes to positions[nodeIndex * 3 + 0..2]
```

## Render List Strategy

**Decision: Rebuild every frame** (5/9 agree)

For 2000 objects: collect visible meshes + radix sort completes in <0.3ms total. Always correct, simple to debug.

## Mobile-Specific Optimizations

**Decision: Tile-based GPU awareness** (4/9: Caracal, Hyena, Mantis, Wren)

- MSAA nearly free on tile-based mobile GPUs (on-chip resolve)
- Minimize render target switches (each forces expensive tile flush)
- Half-resolution bloom (reduces memory bandwidth, dominant mobile cost)
- 1024px shadow maps by default (configurable to 2048 for desktop)
- 3×3 Poisson PCF (9 samples, configurable)

## Frame Budget Breakdown

**CPU budget (~6ms):**

| Phase | Cost |
|-------|------|
| Matrix updates (dirty only) | 0.2-0.5ms |
| Frustum culling | 0.1-0.2ms |
| Sort + render list | 0.05-0.2ms |
| Draw submission (2000 @ 2-3μs) | 4.0-6.0ms |
| Animation update | 0.2-0.5ms |
| Miscellaneous (overlays, controls) | 0.2-0.5ms |

**GPU budget (~10ms on mobile):**

| Phase | Cost |
|-------|------|
| Shadow maps (3 CSM, 1024) | ~1.5ms |
| Opaque geometry (2000 draws) | 3.0-5.0ms |
| MSAA resolve | 0.2-0.5ms |
| OIT composite | 0.3-0.5ms |
| Bloom + tone mapping | 0.6-1.1ms |

Note: JS and GPU overlap (JS prepares frame N+1 while GPU renders frame N). Effective frame time: max(JS, GPU) ≈ 6-10ms → 60fps achievable.

## Memory Budget

| Resource | Budget |
|----------|--------|
| JavaScript heap | < 50MB |
| GPU buffers (geometry) | < 100MB |
| GPU textures | < 200MB |
| Shadow maps (3 × 1024) | ~12MB |
| Bloom chain (1080p, RGBA16F) | ~5MB |
| OIT targets (1080p) | ~18MB |
| MSAA buffers (4x, 1080p) | ~32MB |
| **Total GPU memory** | **< 400MB** |

## Frame Statistics API

```typescript
const stats = engine.getStats()
{
  fps: number,
  frameTime: number,        // Total frame time (ms)
  jsTime: number,           // CPU time (ms)
  gpuTime: number,          // GPU time if available (ms)
  drawCalls: number,
  triangles: number,
  stateChanges: number,
  pipelineSwitches: number,
  culledObjects: number,
  visibleObjects: number,
  shadowDrawCalls: number,
}
```

GPU timing via WebGPU timestamp queries or WebGL2 `EXT_disjoint_timer_query_webgl2`.

## Comparison to Three.js

| Metric | This Engine | Three.js |
|--------|-------------|----------|
| Per-draw overhead | 2-3μs | ~15-50μs (5-20× slower) |
| GC pressure | Zero | High (frame spikes) |
| Matrix updates | Dirty-only | Every object every frame |
| State changes | Radix sort minimizes | No sorting |
| Uniform upload | Single UBO + offsets | Individual uniform calls |

## Anti-Patterns Avoided

- No per-frame `new` allocations in render loop
- No `Array.sort` on hot paths (use radix sort)
- No Map/Set iteration in hot paths (use flat arrays)
- No closure creation in hot paths
- No string concatenation for shader variants (use bitmask)
- No `gl.getError()` in production mode
- No `readPixels` unless explicitly requested
- No synchronous WASM calls on main thread (use workers)
