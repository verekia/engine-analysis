# Performance — Strategies, Budgets, Benchmarks

## Overview

Fennec's primary performance goal is **2000 draw calls at 60fps on recent mobile phones**. This means the entire frame must complete within **16.6ms**, with the JS-side draw call submission taking no more than ~6-8ms to leave headroom for GPU work, post-processing, and browser overhead.

## Why Three.js Is Slow (and How Fennec Differs)

Understanding three.js performance bottlenecks informs every Fennec design decision:

| Three.js Problem | Cost | Fennec Solution |
|---|---|---|
| Heavy `Object3D` overhead (~40 properties) | Memory + GC | Minimal node (~15 properties) |
| `Matrix4.compose()` every frame for every object | CPU | Dirty flags, only recompute changed |
| Shader program switches not sorted | GPU stalls | Sort draw calls by shader ID |
| Material uniform uploads per-object | CPU + GPU | Shared UBOs, sorted by material |
| `WebGLRenderer.render()` traverses scene tree | CPU | Flat pre-built render list |
| No state caching (redundant GL calls) | GPU driver | Full state tracking, skip redundant |
| Transparent objects require sorting | CPU | Weighted Blended OIT, no sort |
| `Raycaster` does brute-force triangle tests | CPU | SAH BVH, flat array layout |
| Geometry attributes re-bound every draw | GPU | VAOs, vertex layout caching |
| GC pressure from temp Vector3/Matrix4 | GC pauses | Pre-allocated typed arrays, zero alloc hot path |

## Frame Budget Breakdown (16.6ms target)

```
Phase                          | Budget  | Notes
-------------------------------|---------|-------------------------------
JS: Animation update           | 0.3ms   | CPU skinning matrices
JS: Scene graph update         | 0.2ms   | Dirty-flag traversal
JS: Frustum culling            | 0.1ms   | AABB tests, 2000 objects
JS: Sort + render list build   | 0.2ms   | Radix sort by sort key
JS: Draw call submission       | 3.0ms   | 2000 calls × 1.5μs each
JS: Total                      | ~4.0ms  |
                               |         |
GPU: Shadow maps (3 cascades)  | 1.5ms   | Depth-only, culled per cascade
GPU: Opaque pass (2000 draws)  | 4.0ms   | Sorted, minimal state changes
GPU: OIT pass (100 transparent)| 0.5ms   | Unsorted, 2 render targets
GPU: Post-processing           | 1.0ms   | MSAA resolve + bloom
GPU: Total                     | ~7.0ms  |
                               |         |
Browser overhead               | ~2.0ms  | Compositing, rAF scheduling
                               |         |
TOTAL                          | ~13ms   | 3.6ms headroom
```

## Draw Call Submission Performance

The critical path is submitting 2000 draw calls with minimal JS overhead. Target: **< 2μs per draw call** on the JS side.

### State Sorting

Draw calls are sorted by a single numeric key to minimize GPU state changes:

```
Sort key (48 bits):
┌────────┬──────────┬────────────┬────────────┐
│ 8 bits │ 16 bits  │  12 bits   │  12 bits   │
│ render │ shader   │  material  │  geometry  │
│ order  │ ID       │  ID        │  ID        │
└────────┴──────────┴────────────┴────────────┘
```

Sorting by this key automatically groups draw calls by:
1. Opaque vs transparent (render order)
2. Same shader program (most expensive switch)
3. Same material (texture/uniform binds)
4. Same geometry (VAO/vertex buffer binds)

### Radix Sort

For 2000 items, radix sort outperforms comparison sort. We use a 2-pass radix sort on the 48-bit key:

```typescript
// Pre-allocated buffers (reused every frame)
const _sortKeys = new Float64Array(MAX_OBJECTS)    // Sort keys
const _sortIndices = new Uint16Array(MAX_OBJECTS)  // Object indices
const _tempKeys = new Float64Array(MAX_OBJECTS)
const _tempIndices = new Uint16Array(MAX_OBJECTS)

const radixSort = (keys: Float64Array, indices: Uint16Array, count: number) => {
  // 2-pass radix sort (24 bits per pass)
  // Pass 1: sort by lower 24 bits
  countingSort(keys, indices, _tempKeys, _tempIndices, count, 0x00FFFFFF, 0)
  // Pass 2: sort by upper 24 bits
  countingSort(_tempKeys, _tempIndices, keys, indices, count, 0x00FFFFFF, 24)
}
```

### Minimal Per-Draw Overhead

Each draw call in the hot loop does exactly:

```typescript
// Per draw call (1.5μs budget)
const submitDraw = (item: RenderItem) => {
  // 1. Bind material (only if changed)
  if (item.materialId !== _currentMaterial) {
    bindMaterial(item.material)           // ~0.3μs
    _currentMaterial = item.materialId
  }

  // 2. Bind geometry (only if changed)
  if (item.geometryId !== _currentGeometry) {
    backend.setVertexBuffer(0, item.geometry._positionBuffer)
    backend.setIndexBuffer(item.geometry._indexBuffer, 'uint16')
    _currentGeometry = item.geometryId    // ~0.2μs
  }

  // 3. Upload object uniforms (always)
  backend.updateUniformBuffer(objectUBO, item._uniformData, 0)  // ~0.5μs
  backend.setBindGroup(3, item._bindGroup)

  // 4. Draw
  backend.drawIndexed(item.geometry.indexCount, 1, 0)  // ~0.3μs
}
```

### WebGPU Render Bundles

For static scenes (objects that don't move), WebGPU render bundles pre-record the entire draw sequence. Replaying a bundle has near-zero JS cost:

```typescript
// Record once (or when scene changes)
const bundle = recordRenderBundle(staticObjects)

// Per frame: ~0.01ms for any number of static objects
renderPass.executeBundles([bundle])

// Only dynamic objects need per-frame draw call submission
for (const item of dynamicObjects) {
  submitDraw(item)
}
```

## Memory Layout

### Avoiding GC Pressure

The #1 cause of frame drops in JS 3D engines is garbage collection pauses. Fennec eliminates GC pressure in the hot path:

```typescript
// BAD (three.js pattern): creates new Vector3 every call
const getWorldPosition = () => new Vector3().setFromMatrixPosition(this.matrixWorld)

// GOOD (Fennec pattern): write to pre-allocated output
const _tempVec3 = vec3.create()  // Module-level, allocated once
const getWorldPosition = (node: SceneNode, out: Vec3 = _tempVec3): Vec3 => {
  out[0] = node.worldMatrix[12]
  out[1] = node.worldMatrix[13]
  out[2] = node.worldMatrix[14]
  return out
}
```

### Pre-Allocated Frame Data

All per-frame arrays are pre-allocated at engine creation and reused:

```typescript
interface FrameData {
  visibleObjects: SceneNode[]       // Pre-allocated to MAX_OBJECTS
  visibleCount: number
  opaqueList: RenderItem[]          // Pre-allocated
  opaqueCount: number
  transparentList: RenderItem[]     // Pre-allocated
  transparentCount: number
  sortKeys: Float64Array            // Pre-allocated
  sortIndices: Uint16Array          // Pre-allocated
  shadowVisible: SceneNode[][]      // One per cascade
}
```

### Typed Array Backing

All math types are backed by typed arrays, not plain objects:

```typescript
// BAD: plain object (GC-tracked, slow property access)
{ x: 1, y: 2, z: 3 }

// GOOD: typed array (no GC overhead, SIMD-friendly, direct GPU upload)
new Float32Array([1, 2, 3])
```

## Uniform Buffer Strategy

### Shared UBOs

Camera and light data is uploaded once per frame to shared UBOs, not per-draw-call:

```
UBO Slot 0: Frame Uniforms (bound once per frame)
┌──────────────────────────────┐
│ mat4 viewProjection    (64B) │
│ mat4 view              (64B) │
│ vec4 cameraPosition    (16B) │
│ vec4 viewport          (16B) │
│ float time              (4B) │
│ float deltaTime         (4B) │
│ padding                 (8B) │
│ Total: 176B                  │
└──────────────────────────────┘

UBO Slot 1: Light Uniforms (bound once per frame)
┌──────────────────────────────┐
│ vec4 dirLightDirection (16B) │
│ vec4 dirLightColor     (16B) │
│ vec4 ambientColor      (16B) │
│ mat4 cascadeVPs[3]    (192B) │
│ vec4 cascadeSplits     (16B) │
│ float shadowBias        (4B) │
│ padding                (12B) │
│ Total: 272B                  │
└──────────────────────────────┘

UBO Slot 3: Object Uniforms (updated per draw call)
┌──────────────────────────────┐
│ mat4 modelMatrix       (64B) │
│ mat4 normalMatrix      (64B) │
│ Total: 128B                  │
└──────────────────────────────┘
```

The object UBO is small (128B) and uses `bufferSubData` (WebGL2) or `writeBuffer` (WebGPU) for fast updates.

## Benchmarking Methodology

### Standard Benchmark Scene

```
2000 unique meshes (cubes):
  - 500 with Material A (shader 1)
  - 500 with Material B (shader 1, different textures)
  - 500 with Material C (shader 2)
  - 500 with Material D (shader 2, different textures)
  - 100 transparent meshes (OIT)
  - 3 CSM cascades, 1 directional light
  - Bloom enabled
  - MSAA 4x
  - Target: 60fps (16.6ms frame time)
```

### Metrics to Track

| Metric | Target | How |
|--------|--------|-----|
| Frame time | < 16.6ms | `performance.now()` around frame |
| JS time | < 6ms | `performance.now()` around JS code |
| GPU time | < 8ms | WebGPU timestamp queries / EXT_disjoint_timer_query |
| Draw calls | 2000 | Count in render loop |
| State changes | < 50 | Count shader/material/geometry switches |
| Triangles | 100K+ | Sum of visible geometry |
| GC pauses | 0 per 60 frames | `PerformanceObserver` for GC events |
| Memory | < 50MB | `performance.memory` (Chrome) |

### Device Targets

| Tier | Device Example | Target FPS | Notes |
|------|----------------|-----------|-------|
| High | iPhone 15 Pro, Pixel 8 | 60fps, 2000 draws | Full quality |
| Medium | iPhone 12, Pixel 6 | 60fps, 1500 draws | Reduce shadow resolution |
| Low | iPhone SE 2, Pixel 4a | 30fps, 1000 draws | Reduce MSAA, bloom levels |

## Optimization Techniques Summary

### CPU-Side

1. **Dirty flags** — Only recompute transforms for moved objects
2. **Flat render lists** — Pre-allocated arrays, no tree traversal per frame
3. **State sorting** — Minimize GPU state switches via sort key
4. **Radix sort** — O(n) sorting for render list
5. **Zero allocation** — All temp math uses pre-allocated typed arrays
6. **Binary search** — Animation keyframe lookup
7. **Frustum culling** — Skip invisible objects entirely

### GPU-Side

1. **Shared UBOs** — Camera/light data uploaded once, not per-draw
2. **VAO caching** — Geometry state bound in one call
3. **MSAA** — Hardware anti-aliasing (not post-process)
4. **Weighted Blended OIT** — No transparent sorting
5. **Dual Kawase bloom** — Fewer passes than Gaussian
6. **Shadow atlas** — Single texture for all cascades
7. **Render bundles** — WebGPU pre-recorded commands for static scenes
8. **Compressed textures** — KTX2/Basis reduces memory bandwidth

### What We Intentionally Don't Do

- **No instancing** — By keeping per-draw overhead < 2μs, 2000 unique draws is feasible without instancing. Instancing adds complexity (instance buffers, material limits) that isn't needed for the target use case.
- **No LOD** — For 200m low-poly worlds, LOD adds complexity without meaningful benefit.
- **No occlusion culling** — Frustum culling is sufficient for open low-poly worlds. Occlusion culling (Hi-Z, software rasterizer) is expensive to implement and maintain.
- **No texture atlasing** — KTX2 compression reduces texture memory pressure enough. Atlasing complicates UV coordinates.
- **No multi-threaded rendering** — Web Workers for asset loading, but the render loop stays on the main thread for simplicity and predictability.
