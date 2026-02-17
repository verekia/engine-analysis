# PERFORMANCE.md - Performance Strategy

## Target

**2000 draw calls at 60fps on recent mobile phones** (universal agreement)

This is the hard constraint that drives every architectural decision. 60fps = 16.6ms per frame. Budget breakdown:

| Category | Budget |
|----------|--------|
| CPU: Scene update + culling + sorting | ~2ms |
| CPU: Animation + user callbacks | ~2ms |
| CPU: Draw call submission | ~2ms |
| GPU: Shadow passes (3 cascades) | ~1.5ms |
| GPU: Opaque pass | ~4ms |
| GPU: Transparent pass (OIT) | ~0.6ms |
| GPU: Post-processing (bloom + tone map) | ~0.8ms |
| Headroom | ~2.7ms |

## Performance Wins (Ranked by Impact)

### 1. Draw Call Sorting (Highest Impact)

**Decision: 64-bit radix sort by pipeline > material > depth** (Mantis, Shark, Wren approach)

State changes are the dominant CPU-side cost. Sorting draws to minimize state switches provides the largest single performance win:

- Pipeline (shader) switches: ~50us each on mobile
- Material/texture binds: ~10us each
- Buffer offset changes: ~1us each

With 2000 draws and 20 unique pipelines, unsorted submission causes ~200 pipeline switches. Sorted: ~20. That is ~9ms saved on mobile.

### 2. Frustum Culling

**Decision: AABB test against 6 frustum planes, BVH-accelerated for large scenes**

Culling invisible objects prevents wasted GPU work. For a scene where the camera sees 50% of objects, culling saves 50% of draw calls. Cost: 0.1ms for 2000 AABBs (brute force) or 0.03ms (BVH-accelerated).

### 3. WebGPU-First

**Decision: WebGPU primary, WebGL2 fallback** (universal agreement)

WebGPU has lower driver overhead than WebGL2:
- Immutable pipelines (no per-draw state validation)
- Render bundles for static geometry
- Compute shaders for culling and mipmap generation
- Better multi-threaded command recording potential

On devices with WebGPU, draw call submission is ~30-50% cheaper than WebGL2.

### 4. Dirty Flags for Incremental Updates

**Decision: Update world matrices only for dirty nodes** (universal agreement)

Typical frame: 2-5% of nodes have moved. Dirty flags skip 95%+ of matrix recomputations. Cost of flagging: O(subtree depth) per change. Cost of update: proportional to dirty node count, not total node count.

### 5. State Cache (WebGL2)

**Decision: Full GL state tracking to eliminate redundant calls** (8/9 implementations)

Before every GL call, check if the state is already set. Eliminates 40-60% of redundant API calls when draws are sorted. Essential for WebGL2 performance parity with WebGPU.

### 6. Zero Per-Frame Allocations

**Decision: No `new`, no array creation, no closures in hot paths** (universal agreement)

JavaScript GC pauses are unpredictable and can cause frame spikes:
- Pre-allocate all math scratch variables
- Pre-allocate sort key arrays, draw command buffers, traversal stacks
- Frame-scoped scratch pool with pointer reset (not deallocation)
- Reuse typed arrays across frames

Target: zero GC collections triggered by the render loop.

### 7. Shader Warm-up

**Decision: Pre-compile common shader variants during loading** (Caracal approach)

WebGL shader compilation is synchronous and can block the main thread for 50-200ms per variant. Pre-compiling during the asset loading phase prevents frame hitches during gameplay.

## Memory Budget

| Resource | Budget |
|----------|--------|
| JavaScript heap | < 50MB |
| GPU buffers (geometry) | < 100MB |
| GPU textures | < 200MB |
| Shadow maps (3 x 1024) | ~12MB |
| Bloom chain (1080p, RGBA16F) | ~5MB |
| OIT targets (1080p) | ~18MB |
| MSAA buffers (4x, 1080p) | ~32MB |
| **Total GPU memory** | **< 400MB** |

## Instancing

**Decision: Automatic instancing for meshes with same geometry + material** (universal agreement)

When multiple meshes share the same geometry and material, batch them into a single instanced draw call. Per-instance data (world matrix, color, flags) is uploaded to an instance buffer.

The sort key groups instances together naturally (same pipeline + same material = adjacent in sorted order). After sorting, consecutive draws with the same geometry+material are merged into instanced draws.

### Instance Buffer Layout

```
Per instance (96 bytes):
  mat4 worldMatrix (64 bytes)
  vec4 instanceColor (16 bytes)
  uint flags (4 bytes)
  padding (12 bytes)
```

## Profiling

### Frame Statistics API

```typescript
const stats = engine.getStats()
// Returns:
{
  fps: number,
  frameTime: number,       // Total frame time (ms)
  cpuTime: number,         // JavaScript time (ms)
  gpuTime: number,         // GPU time if available (ms)
  drawCalls: number,       // Actual GPU draw calls submitted
  triangles: number,       // Triangles rendered
  visibleObjects: number,  // Objects passing frustum culling
  culledObjects: number,   // Objects culled
  stateChanges: number,    // Pipeline/material switches
  textureBinds: number,    // Texture unit changes
}
```

### GPU Timing

**WebGPU**: Use `GPUCommandEncoder.writeTimestamp()` for per-pass GPU timing.
**WebGL2**: Use `EXT_disjoint_timer_query_webgl2` extension (not universally available).

## Scaling Strategies

### For Scenes > 2000 Objects

- BVH-accelerated frustum culling reduces visible set
- Instancing reduces draw call count
- Level-of-detail (future: not v1) reduces triangle count
- Object pooling for dynamic spawn/despawn

### For Mobile

- Reduce shadow map resolution (512x512 instead of 1024)
- Reduce bloom levels (3 instead of 5)
- Disable MSAA on very low-end devices
- Use RGBA8 bloom chain instead of RGBA16F

### For Desktop

- 2048x2048 shadow maps
- 4x MSAA
- Full bloom pipeline
- Higher geometry detail levels (future: LOD)

## WebGPU-Specific Optimizations

- **Render bundles**: Cache draw commands for static geometry
- **Compute shaders**: GPU-side frustum culling (move AABB tests to GPU)
- **Indirect draw**: `drawIndirect` / `drawIndexedIndirect` for GPU-driven rendering
- **Timestamp queries**: Per-pass GPU profiling

These are performance improvements that layer on top of the baseline architecture. The engine works correctly without them; they are progressive enhancements.

## Anti-Patterns Avoided

- No per-frame `new` allocations in render loop
- No `Array.sort` on hot paths (use radix sort)
- No Map/Set iteration in hot paths (use flat arrays)
- No closure creation in hot paths
- No string concatenation for shader variants (use bitmask)
- No `gl.getError()` in production mode
- No `readPixels` unless explicitly requested (occlusion testing)
- No synchronous WASM calls on main thread (use workers)
