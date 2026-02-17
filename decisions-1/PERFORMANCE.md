# PERFORMANCE - Design Decisions

## Target

**Decision: 2000 draw calls at 60fps on recent mobile phones (16.6ms frame budget)**

- Sources: All 9 implementations agree
- Per-draw JS overhead target: <2-3μs
- This means the total JS draw submission cost for 2000 draws is ~4-6ms, leaving ~10ms for GPU work

## No Automatic Instancing

**Decision: Skip instancing — optimize individual draw calls instead**

- Sources: 8/9 implementations agree (all except Bonobo)
- Per-draw overhead optimized to <2-3μs makes 2000 unique draws feasible
- Instancing adds complexity (instance buffers, batching logic, material limits) without meaningful benefit when individual draws are cheap enough
- Games with 1000 identical trees can add manual instancing later if needed

## Draw Call Sorting

**Decision: Radix sort on 64-bit composite key**

- Sources: Caracal, Fennec, Hyena, Mantis, Rabbit, Wren (6/9 use radix sort)
- Rejected: `Array.prototype.sort` (Bonobo, Lynx) — O(n log n) vs O(n), though still fast enough

Sort key structure:
```
[layer 2b] [pass 2b] [transparent 1b] [pipeline 11b] [material 16b] [depth 32b]
```

Sort priority:
1. **Pipeline/shader** (most expensive state change)
2. **Material/textures** (moderate cost)
3. **Depth** (front-to-back for opaque → early-Z rejection)

Results:
- Without sorting: ~2000 state changes
- With sorting: ~10-200 state changes
- Reduction: 90-99%

## Frustum Culling

**Decision: Brute-force AABB-frustum test for <5000 objects; BVH-accelerated optional**

- Sources: Brute force from Bonobo, Caracal, Rabbit, Shark, Wren (5/9); BVH from Fennec, Hyena, Lynx, Mantis (4/9)

For 2000 objects, brute-force culling (6 plane tests per object) completes in ~0.1-0.2ms. Adding BVH would save ~0.05ms at the cost of maintenance complexity.

BVH-accelerated culling is available for scenes exceeding 5000 objects, using the scene-level BVH that already exists for raycasting.

Typical culling ratio: 30-70% of objects culled per frame.

## Zero Per-Frame Allocation

**Decision: Pre-allocate everything, never allocate in render loop (universal agreement)**

- Pre-allocated typed arrays at startup
- Module-level scratch variables for math temporaries
- Frame-scoped pool with reset for temporary vectors
- Pre-allocated draw command arrays
- No `new`, no object creation, no array creation in hot paths
- Eliminates GC pressure entirely — no frame spikes from garbage collection

## Dirty Flag System

**Decision: Two-level dirty flags for minimal matrix recomputation (universal agreement)**

- Only recompute local matrices for nodes whose position/rotation/scale changed
- Only recompute world matrices for nodes whose parent chain is dirty
- Typical: 2-5% of objects move per frame → 95-98% of matrix work skipped

## Uniform Upload Strategy

**Decision: Triple-buffered ring buffer with dynamic offsets**

- Sources: Mantis, Shark, Wren (ring buffer); all 9 (dynamic offsets)

```
Ring buffer: 4MB total, 3 frame regions (~1.3MB each)
Per-object: 256 bytes (world matrix + material data, aligned)
Capacity: ~5000 objects per frame region
```

- Zero per-object buffer allocation
- GPU never stalls on previous frames (triple buffered)
- Per-object switching uses dynamic offsets only — no bind group creation

## WebGPU Render Bundles

**Decision: Available for static geometry, not primary optimization**

- Sources: Caracal, Hyena, Lynx, Mantis, Rabbit (5/9) use render bundles

Record render bundles for static objects (e.g., 1800 static environment meshes). Replay is near-zero JS cost. Invalidate on scene structure change. Dynamic objects (200 animated characters) use normal draw submission.

Not the primary optimization — sort-based rendering already achieves the target. Render bundles are an additional optimization for games with mostly static environments.

## Shader Variant Pre-compilation

**Decision: Pre-compile common variants during loading**

- Sources: Caracal, Fennec, Hyena, Mantis, Rabbit (5/9)

Pre-warm the most common shader combinations during asset loading:
- Lambert opaque
- Lambert opaque + shadow
- Lambert opaque + shadow + texture
- Lambert opaque + shadow + vertex colors
- Lambert transparent
- Basic opaque
- Shadow depth-only

Avoids frame spikes from on-demand compilation during gameplay.

## Mobile-Specific Optimizations

**Decision: Tile-based GPU awareness**

- Sources: Caracal, Hyena, Mantis, Wren (4/9)

- **MSAA nearly free on mobile**: on-chip resolve, no extra bandwidth
- **Minimize render target switches**: each switch forces tile flush (expensive)
- **Half-resolution bloom**: reduces memory bandwidth (dominant cost on mobile)
- **1024px shadow maps**: configurable down from 2048 for mobile
- **3×3 PCF shadows**: 9 samples vs 25 for mobile (configurable)

## Data Layout

**Decision: SoA (Structure of Arrays) internally, OOP wrapper for public API**

- Sources: SoA from Bonobo, Hyena, Lynx, Mantis, Wren (5/9); OOP wrapper from Fennec, Caracal, Rabbit (3/9)

Internally, transforms stored as flat typed arrays for cache-friendly batch operations:
```typescript
// Internal
positions: Float32Array(maxNodes * 3)
worldMatrices: Float32Array(maxNodes * 16)
```

Public API wraps with familiar object interface:
```typescript
// Public
mesh.position.set(1, 2, 3)  // writes to positions[nodeIndex * 3 + 0..2]
```

## Render List Strategy

**Decision: Rebuild every frame**

- Sources: Bonobo, Caracal, Fennec, Lynx, Mantis, Rabbit, Wren (7/9)
- Rejected: Persistent sorted list (Hyena, Shark) — marginal gain, more complex

For 2000 objects: collect visible meshes + radix sort completes in <0.3ms total. Always correct, simple to debug.

## State Change Reduction (with sorting)

Typical breakdown for 2000 sorted draws:
- Pipeline switches: 4-10
- Material switches: 8-200
- Texture switches: 12-500
- Geometry switches: 50-500
- Dynamic offset only: 1500-1900

Most draws only require changing the dynamic UBO offset — the cheapest possible state change.

## Frame Statistics API

**Decision: Comprehensive stats for profiling**

- Sources: 7/9 implementations provide stats

```typescript
const stats = engine.getStats()
{
  fps: number,
  frameTime: number,        // ms
  jsTime: number,           // ms (CPU)
  gpuTime: number,          // ms (when available via timestamp queries)
  drawCalls: number,
  triangles: number,
  stateChanges: number,
  pipelineSwitches: number,
  culled: number,
  visible: number,
  shadowDrawCalls: number,
}
```

GPU timing available via WebGPU timestamp queries or WebGL2 `EXT_disjoint_timer_query_webgl2`.

## Frame Budget Breakdown

```
JavaScript (CPU):
  Matrix updates (dirty only):      0.2-0.5ms
  Frustum culling:                   0.1-0.2ms
  Draw list sorting (radix):         0.05-0.1ms
  Uniform writes (ring buffer):      0.2-0.5ms
  Draw submission (2000 @ 2-3μs):    4.0-6.0ms
  Animation update:                  0.1-0.5ms
  Miscellaneous:                     0.2-0.5ms
  ─────────────────────────────────────────────
  JS Total:                          ~5.0-8.0ms

GPU:
  Shadow maps (3 CSM, depth-only):   1.5-3.0ms
  Opaque pass (2000 draws):          3.0-5.0ms
  MSAA resolve:                      0.2-0.5ms
  OIT composite:                     0.3-0.5ms
  Bloom (5 down + 5 up):             0.5-1.5ms
  Tone mapping + blit:               0.1-0.2ms
  ─────────────────────────────────────────────
  GPU Total:                         ~5.5-10.5ms

Note: JS and GPU overlap (JS prepares frame N+1 while GPU renders frame N)
Effective frame time: max(JS, GPU) ≈ 5-10ms → 60fps achievable
```

## Comparison to Three.js

- Per-draw overhead: 2-3μs vs Three.js ~15-50μs (5-20× improvement)
- GC pressure: zero vs Three.js high (eliminates GC frame spikes)
- Matrix updates: dirty-only vs Three.js all objects every frame (40× improvement typical)
- State changes: radix sort minimizes vs Three.js no sorting
- Uniform upload: single UBO with offsets vs Three.js individual uniform calls
