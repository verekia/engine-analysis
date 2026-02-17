# PERFORMANCE.md - Decisions

## Decision: 2000 Draws at 60fps, Radix Sort, No Instancing, SoA + Dirty Flags

### Hard Performance Target

Universal agreement: **2000 draw calls at 60fps on mobile** (16.6ms frame budget).

### No Instancing

**Chosen**: No instancing (8/9 implementations)
**Rejected**: Automatic instancing (Bonobo only)

Rationale from 8/9 implementations: when per-draw JS overhead is optimized to <2-3 microseconds, 2000 unique draws is feasible without instancing. Instancing adds complexity (instance buffers, material limits, instance data management) for a benefit that only materializes with >5000 identical objects, which is uncommon in the target use case.

### Per-Draw Overhead Target: <2-3 Microseconds

For 2000 draws at <3us each: ~6ms JS overhead, leaving ~10ms for GPU work.

Achieved via:
- Radix sort (minimizes state changes)
- Full state cache on WebGL2 (skip redundant GL calls)
- Dynamic UBO offsets (single buffer, offset changes per draw)
- Pre-compiled pipelines (no runtime state assembly)

### Sort Algorithm: Radix Sort

**Chosen**: Radix sort (6/9: Caracal, Fennec, Hyena, Mantis, Rabbit, Wren)
**Rejected**: `Array.sort` (2/9) - O(n log n) vs O(n), unpredictable timing

For 2000 keys, radix sort completes in ~0.05-0.1ms. Two passes on 32-bit words for 64-bit keys.

State change reduction: 90-99% (from ~2000 state changes unsorted to ~10-200 sorted).

### Data Layout: SoA for Hot Data

**Chosen**: Structure of Arrays for transform data and render state (5/9: Bonobo, Hyena, Lynx, Mantis, Wren)

```typescript
// Contiguous typed arrays for cache-friendly access
worldMatrices: Float32Array(maxObjects * 16)
aabbPool: Float32Array(maxObjects * 6)
visibilityFlags: Uint8Array(maxObjects)
sortKeys: BigUint64Array(maxObjects)
```

OOP wrappers for the public API reference into these arrays by index.

### Frustum Culling: AABB P-Vertex Test

Universal agreement: frustum culling via Gribb-Hartmann plane extraction + p-vertex/n-vertex AABB test.

Typical culling: 30-70% of objects invisible. For 2000 objects with brute force: ~0.1-0.2ms.

**BVH-accelerated culling** available but not the default strategy. Brute force is sufficient for <5000 objects. The scene BVH is built regardless (for raycasting), so frustum culling can optionally use it.

### Dirty Flag System

Universal agreement: only recompute what changed.

- **Transform dirty**: Only recompute world matrices for moved objects (~2-5% per frame)
- **Material dirty**: Only rebuild material bind groups when material properties change
- **Geometry dirty**: Only re-upload geometry buffers when vertex data changes

Typical frame: ~40x fewer matrix computations than "update everything."

### Zero Per-Frame Allocation

Universal agreement. Strategies:

- Pre-allocated typed arrays at startup
- Module-level scratch variables for math temporaries
- Frame-scoped pool with reset (Mantis/Lynx approach)
- No `new` calls in the render loop
- No string concatenation or object spread in hot paths

### Uniform Upload: Dynamic Offsets

**Chosen**: Single large UBO per bind group, dynamic offsets per draw (6/9 implementations)

Per-object data (world matrix, material index, flags): 128-256 bytes per object.
Total per-frame upload: 2000 * 256 = ~512KB via single `writeBuffer`.

### Render Bundles: Optional WebGPU Enhancement

Available for static geometry on WebGPU (5/9 mention this). Not the primary strategy.

Typical split: 1800 static + 200 dynamic objects. Static objects recorded into a render bundle, replayed with near-zero JS cost.

### Frame Budget Distribution

Target for 2000 draws at 60fps on mobile:

**CPU budget (~6ms):**
| Phase | Cost |
|-------|------|
| Matrix updates (dirty only) | 0.2-0.5ms |
| Frustum culling | 0.1-0.2ms |
| Sort + render list | 0.05-0.2ms |
| Draw submission | 4-5ms |
| Animation | 0.2-0.5ms |
| Misc (overlays, controls) | 0.2-0.5ms |

**GPU budget (~10ms on mobile):**
| Phase | Cost |
|-------|------|
| Shadow maps (3 CSM, 1024) | 2.5ms |
| Opaque geometry | 3-5ms |
| Transparency (OIT) | 0.5-1ms |
| Post-processing (bloom) | 1-2ms |
| MSAA resolve | 0.2-0.5ms |

### Mobile-Specific Optimizations

**Chosen**: Tile-based GPU awareness (4/9: Caracal, Hyena, Mantis, Wren)

- MSAA nearly free on tile-based mobile GPUs (on-chip resolve)
- Minimize render target switches (forces expensive tile flush)
- Half-resolution bloom (reduces memory bandwidth)
- Proper load/store actions for render passes

### Shader Compilation Strategy

**Chosen**: Lazy compilation with cache (6/9 implementations)

Shader variants compiled on first use and cached by feature bitmask. Optionally pre-compile during loading via `engine.precompileShaders()`.

### Stats API

```typescript
interface FrameStats {
  fps: number
  frameTime: number       // Total frame time (ms)
  jsTime: number          // JS-side time (ms)
  gpuTime: number         // GPU time (ms, when available via timestamp queries)
  drawCalls: number
  triangles: number
  stateChanges: number
  culledObjects: number
  visibleObjects: number
}

engine.onFrameStats((stats) => { /* update debug UI */ })
```

GPU timing via WebGPU timestamp queries or WebGL2 `EXT_disjoint_timer_query_webgl2`.

### Profiling Hooks

- Per-phase timing (shadow, opaque, transparent, post)
- State change counter
- Culling effectiveness (visible/total ratio)
- Memory usage estimates
