# Performance Strategy

## Target

**2000+ draw calls at 60 fps on recent mobile phones** (e.g., iPhone 14, Pixel 7, Samsung S23).

This is the central design constraint. Every architectural decision in Kestrel is informed by this target.

## Why Three.js is Slow

To understand Kestrel's approach, we need to understand where three.js spends its time:

1. **Per-object JavaScript overhead** (~50% of frame time for 2000 objects):
   - `updateWorldMatrix()` creates temporary `Matrix4` objects
   - `onBeforeRender` callbacks, layer checks, visibility checks
   - Uniform upload: each draw call sets 20+ individual `gl.uniform*` calls
   - Material property validation on every frame

2. **GPU state changes** (~20%):
   - No sorting by material — random state change order
   - Individual texture binds, blend state changes, shader switches

3. **Garbage collection** (~10%):
   - Temporary `Vector3`, `Matrix4`, `Color` objects created per frame
   - Closure allocations in callbacks

4. **Render list rebuild** (~10%):
   - Every frame: collect objects, sort, filter

Kestrel eliminates each of these.

## Strategy 1: Zero-Allocation Render Loop

**No JavaScript objects are created during rendering.** The entire render loop operates on pre-allocated typed arrays and pools.

### Math Pool

```typescript
// Pre-allocated scratch registers
const pool = {
  vec3: Array.from({ length: 32 }, () => new Float32Array(3)),
  vec4: Array.from({ length: 16 }, () => new Float32Array(4)),
  mat4: Array.from({ length: 16 }, () => new Float32Array(16)),
  quat: Array.from({ length: 8 },  () => new Float32Array(4)),
  _vec3Index: 0,
  _vec4Index: 0,
  _mat4Index: 0,
  _quatIndex: 0,
}

// Usage — grab a temporary, use it, done. Index resets each frame.
const tmpVec = pool.vec3[pool._vec3Index++]
vec3.subtract(tmpVec, a, b)
```

At the start of each frame, all pool indices reset to 0. No allocation, no deallocation, no GC.

### SoA Transform Updates

World matrix updates operate on contiguous `Float32Array` slices. No `Matrix4` class instances. See [SCENE-GRAPH.md](./SCENE-GRAPH.md) for details.

## Strategy 2: Dirty Flag System

Only recompute what changed:

- **Transform dirty**: Only recompute `localMatrix` and `worldMatrix` for objects whose position/rotation/scale changed.
- **Render list dirty**: Only re-sort when materials or objects change (not every frame).
- **Bounding box dirty**: Only recompute world AABB when world matrix changes.
- **BVH dirty**: Only refit BVH nodes whose child AABBs changed.

In a typical game frame with 2000 objects, maybe 50 move. That's 50 matrix recomputations instead of 2000.

## Strategy 3: Draw Call Sorting

Draw calls are sorted by a 64-bit key to minimize GPU state changes:

```
Sort key layout:
[8 bits: pipeline] [8 bits: material] [16 bits: texture hash] [32 bits: depth]
```

This groups draws by:
1. **Pipeline** (shader variant) — most expensive to switch
2. **Material** (uniform buffer) — moderate cost to switch
3. **Texture** (bind group) — moderate cost to switch
4. **Depth** (front-to-back) — maximizes early-z rejection

Result: for 2000 draw calls, if there are 5 unique pipelines and 20 unique materials, we get ~5 pipeline switches and ~20 material switches instead of potentially 2000 of each.

### Sorting Algorithm

Radix sort on the 64-bit key. O(n) time complexity. For 2000 items, this takes <0.05ms.

## Strategy 4: Uniform Upload Batching

### WebGPU: Dynamic Bind Group Offsets

Per-object uniforms (world matrix, material color) are written to a single large uniform buffer. Each draw call uses a dynamic offset:

```
// Upload all object uniforms once
const uniformData = new Float32Array(2000 * 20)  // 20 floats per object
for i in 0..2000:
  writeObjectUniforms(uniformData, i * 20, objects[i])
device.writeBuffer(uniformBuffer, uniformData)

// Draw loop — just set offset
for i in 0..2000:
  setBindGroup(1, objectBindGroup, [i * 256])  // dynamic offset
  drawIndexed(...)
```

Cost: 1 buffer upload + 2000 offset changes. Versus three.js: 2000 × ~20 individual uniform calls.

### WebGL2: UBO with Dynamic Offset

Same strategy using `gl.bindBufferRange()` with different offsets into a single UBO:

```
gl.bindBufferRange(GL.UNIFORM_BUFFER, bindingPoint, ubo, offset, size)
```

## Strategy 5: Render List Persistence

The render list is not rebuilt every frame. It's a persistent sorted array:

```
Frame 1: Build full render list, sort (O(n log n))
Frame 2: Nothing changed → reuse list (O(0))
Frame 3: 1 object added → insert in sorted position (O(log n))
Frame 4: 1 material changed → re-sort one entry (O(log n))
Frame N: Nothing changed → reuse list (O(0))
```

Typical per-frame cost: O(1) to O(log n) instead of O(n log n).

## Strategy 6: WebGPU Render Bundles

For static geometry (which is the majority in most game scenes), WebGPU render bundles eliminate virtually all per-draw JS overhead:

```typescript
// Record once
const bundle = device.createRenderBundle(encoder => {
  for (const item of staticRenderList) {
    encoder.setPipeline(item.pipeline)
    encoder.setBindGroup(0, frameBindGroup)
    encoder.setBindGroup(1, item.materialBindGroup)
    encoder.setBindGroup(2, item.objectBindGroup)
    encoder.setVertexBuffer(0, item.vertexBuffer)
    encoder.setIndexBuffer(item.indexBuffer, 'uint16')
    encoder.drawIndexed(item.indexCount)
  }
})

// Replay every frame — near-zero JS cost
renderPass.executeBundles([bundle])
```

The bundle is invalidated only when static objects are added/removed. For a scene with 1800 static objects and 200 dynamic ones, the JS overhead drops to handling just the 200 dynamic draw calls.

On WebGL2, there's no equivalent. The WebGL2 path relies on strategies 1–5 to stay fast.

## Strategy 7: Frustum Culling with BVH

The BVH allows frustum culling to be O(log n) instead of O(n):

```
Without BVH: test 2000 AABBs against 6 planes = 12000 dot products
With BVH:    traverse tree, skip entire branches = ~200 dot products typical
```

See [BVH-RAYCASTING.md](./BVH-RAYCASTING.md) for BVH details.

## Strategy 8: Minimal Shader Variants

Each unique combination of features compiles to a dedicated shader with no branching. This is critical for mobile GPUs (which handle branches poorly):

```wgsl
// BAD: dynamic branching per fragment
if (hasTexture) { color *= textureSample(...); }
if (hasShadow) { shadow = sampleShadow(...); }

// GOOD: compiled-out at pipeline creation
// Variant: lambert_tex_shadow
color *= textureSample(...);
shadow = sampleShadow(...);
```

Kestrel generates all needed variants at material creation time and caches them.

## Performance Budget Breakdown (2000 draw calls, 60 fps target)

```
Total frame budget:             16.6 ms
├── JavaScript (target: ≤4ms)
│   ├── Transform updates:       0.3 ms (dirty flags, ~50 changed)
│   ├── Animation updates:       0.5 ms (~10 skinned meshes)
│   ├── Frustum culling:         0.2 ms (BVH query)
│   ├── Render list maintenance: 0.1 ms (persistent, minimal changes)
│   ├── Uniform buffer writes:   0.8 ms (batch upload)
│   ├── Draw call submission:    1.5 ms (2000 calls, sorted)
│   └── Misc (controls, etc):   0.3 ms
│
├── GPU (target: ≤10ms)
│   ├── Shadow pass:             2.0 ms (depth-only, 3 cascades)
│   ├── Geometry pass (opaque):  4.0 ms (2000 draw calls, sorted)
│   ├── Geometry pass (OIT):     1.0 ms (transparent objects)
│   ├── MSAA resolve:            0.5 ms
│   ├── Bloom:                   1.5 ms (downsample + upsample chain)
│   └── Final composite:         0.3 ms
│
└── Headroom:                    2.6 ms
```

## Comparison with Three.js

| Metric | Three.js (2000 calls) | Kestrel (2000 calls) |
|---|---|---|
| JS frame time | ~12ms | ~4ms |
| State changes | ~2000 | ~50 |
| GC pressure | High | Zero |
| Uniform uploads | 40000 individual calls | 1 batch upload |
| Matrix updates | 2000/frame (all) | ~50/frame (dirty only) |
| Render list sort | Every frame | Only on change |
| WebGPU bundles | N/A | Yes (static) |

## Mobile-Specific Optimizations

- **Tile-based GPU awareness**: MSAA is nearly free on mobile tile-based renderers (Adreno, Mali, Apple GPU) when using the load/store architecture correctly. Kestrel configures render passes with `loadOp: 'clear'` and `storeOp: 'discard'` on depth where possible.
- **Half-resolution bloom**: The bloom chain operates at half resolution, which is fine for the soft glow effect and dramatically reduces fragment shader pressure.
- **Texture compression**: KTX2/Basis transcodes to ASTC on mobile (native format), avoiding costly runtime decompression.
- **Shader simplicity**: Lambert shading (not PBR) means fewer texture samples and fewer ALU operations per fragment — exactly what mobile GPUs need.
