# Performance Strategy

## Target: 2000 Draw Calls at 60fps on Mobile

The primary performance target is **2000 draw calls at 60fps on recent mobile phones** (e.g., iPhone 14+, Pixel 7+). This gives us a budget of **16.6ms per frame**, of which the JS overhead for draw submission must consume no more than ~6ms (the rest goes to GPU execution, compositing, and browser overhead).

Three.js typically falls apart at 500-1000 draw calls on mobile due to excessive per-draw JS overhead. Lynx achieves 2000+ by eliminating that overhead through several complementary strategies.

## Strategy 1: Minimize Per-Draw JS Work

### The Problem

In three.js, each draw call involves:
- Traversing the scene graph to find the object
- Computing the world matrix
- Setting uniforms (multiple `gl.uniform*` calls per object)
- Binding VAO, textures, shader program
- Calling `gl.drawElements`

This adds up to ~20-50 µs per draw call in JS alone. At 2000 calls: 40-100ms — far too slow.

### The Solution

Lynx pre-computes everything and minimizes per-draw JS work to ~2-3 µs:

```
Per-draw work in Lynx:
  1. Compare sort key with previous (1 branch)
  2. If pipeline changed: bind pipeline (rare — ~16 unique pipelines)
  3. If material changed: bind group offset (rare — ~100 unique materials)
  4. Set dynamic UBO offset (1 call)
  5. Call draw (1 call)
```

**Total per-draw JS: ~2-3 µs → 2000 draws = 4-6ms**

## Strategy 2: Sort and Batch by State

### Sort Key Encoding

Each render command has a 64-bit sort key:

```
Bits 63-60: Pipeline ID    (4 bits  → 16 pipelines max)
Bits 59-48: Material ID    (12 bits → 4096 materials max)
Bits 47-0:  Depth          (48 bits → front-to-back for opaque)
```

Sorting by this key (via radix sort — O(n), not O(n log n)) groups draws by pipeline first, then by material. This means:

- **Pipeline binds** (most expensive state change): only at pipeline boundaries (~5-10 per frame)
- **Material binds**: only at material boundaries (~100-500 per frame)
- **Per-object UBO offset**: every draw call, but it's just a single offset change

### Radix Sort Performance

Radix sort on 2000 64-bit keys takes ~0.1ms on mobile. This is faster than `Array.sort` for large n and produces a perfectly sorted command list.

## Strategy 3: Dynamic UBO with Offsets

Instead of uploading per-object uniforms individually, pack all object data into a single large Uniform Buffer Object:

```
UBO Layout (per frame):
┌────────────────────┐ offset 0
│ Object 0           │
│  worldMatrix (64B) │
│  materialData(64B) │
│  padding    (128B) │
├────────────────────┤ offset 256
│ Object 1           │
│  worldMatrix (64B) │
│  materialData(64B) │
│  padding    (128B) │
├────────────────────┤ offset 512
│ Object 2           │
│  ...               │
└────────────────────┘
```

Each object occupies exactly 256 bytes (the minimum UBO alignment on most GPUs). For 2000 objects: **512 KB per frame**.

On draw:
```typescript
// WebGPU: set bind group with dynamic offset
pass.setBindGroup(1, objectBindGroup, [objectIndex * 256])

// WebGL2: bind buffer range
gl.bindBufferRange(gl.UNIFORM_BUFFER, 1, ubo, objectIndex * 256, 256)
```

**One call per object** instead of multiple uniform uploads.

### Writing the UBO

```typescript
// Pre-allocated Float32Array for the entire UBO
const objectUboData = new Float32Array(2000 * 64) // 256 bytes = 64 floats per object

const writeObjectData = (index: number, worldMatrix: Float32Array, materialData: Float32Array) => {
    const offset = index * 64 // 64 floats = 256 bytes
    objectUboData.set(worldMatrix, offset)      // 16 floats = world matrix
    objectUboData.set(materialData, offset + 16) // material palette, etc.
}

// Upload entire UBO once per frame
device.writeBuffer(objectUbo, 0, objectUboData)
```

One buffer upload per frame for all objects. No per-object GL calls for uniforms.

## Strategy 4: WebGPU Render Bundles

For static objects (non-animated meshes that don't change material), pre-record draw commands into a **Render Bundle**:

```typescript
const recordStaticBundle = (device: GalDevice, staticCommands: RenderCommand[]): GPURenderBundle => {
    const encoder = device.createRenderBundleEncoder({
        colorFormats: ['rgba16float'],
        depthStencilFormat: 'depth24plus',
        sampleCount: 4,
    })

    let currentPipeline = -1
    let currentMaterial = -1

    for (const cmd of staticCommands) {
        if (cmd.pipelineId !== currentPipeline) {
            encoder.setPipeline(pipelines[cmd.pipelineId])
            currentPipeline = cmd.pipelineId
        }
        if (cmd.materialId !== currentMaterial) {
            encoder.setBindGroup(0, materialBindGroups[cmd.materialId])
            currentMaterial = cmd.materialId
        }
        encoder.setBindGroup(1, objectBindGroup, [cmd.objectIndex * 256])
        encoder.setVertexBuffer(0, cmd.geometry.vertexBuffer)
        encoder.setIndexBuffer(cmd.geometry.indexBuffer, 'uint16')
        encoder.drawIndexed(cmd.geometry.indexCount)
    }

    return encoder.finish()
}

// In render loop — replay the bundle with near-zero JS overhead
renderPass.executeBundles([staticBundle])
```

Render bundles reduce JS draw overhead to **nearly zero** for static objects. Only dynamic (animated) objects need per-frame command recording.

### Invalidation

The static bundle is re-recorded only when:
- An object is added or removed from the scene
- An object's material changes
- An object's visibility changes

For a typical scene where most objects are static, this means the bundle is re-recorded rarely.

## Strategy 5: WebGL2 State Caching

For WebGL2 (no render bundles), minimize state changes by tracking current state:

```typescript
interface GL2StateCache {
    currentProgram: WebGLProgram | null
    currentVAO: WebGLVertexArrayObject | null
    currentTextures: (WebGLTexture | null)[]
    currentUboBindings: (WebGLBuffer | null)[]
    currentBlend: boolean
    currentDepthTest: boolean
    currentDepthWrite: boolean
    currentCullFace: number
}

const bindPipelineGL2 = (gl: WebGL2RenderingContext, cache: GL2StateCache, pipeline: GL2Pipeline) => {
    if (cache.currentProgram !== pipeline.program) {
        gl.useProgram(pipeline.program)
        cache.currentProgram = pipeline.program
    }
    if (cache.currentCullFace !== pipeline.cullFace) {
        gl.cullFace(pipeline.cullFace)
        cache.currentCullFace = pipeline.cullFace
    }
    // ... etc for blend, depth, stencil
}

const bindGeometryGL2 = (gl: WebGL2RenderingContext, cache: GL2StateCache, vao: WebGLVertexArrayObject) => {
    if (cache.currentVAO !== vao) {
        gl.bindVertexArray(vao)
        cache.currentVAO = vao
    }
}
```

Combined with sort-by-state, this reduces redundant GL calls by 80-90%.

## Strategy 6: Zero Per-Frame Allocation

JavaScript garbage collection pauses can cause frame drops. Lynx allocates nothing during steady-state rendering:

### Pre-allocated Pools

```typescript
// Typed array pool for temporary math operations
const tempVec3Pool = Array.from({ length: 16 }, () => new Float32Array(3))
const tempMat4Pool = Array.from({ length: 8 }, () => new Float32Array(16))
let tempVec3Index = 0
let tempMat4Index = 0

const getTempVec3 = (): Float32Array => tempVec3Pool[tempVec3Index++ & 15]
const getTempMat4 = (): Float32Array => tempMat4Pool[tempMat4Index++ & 7]

// Reset at start of each frame
const resetTempPools = () => {
    tempVec3Index = 0
    tempMat4Index = 0
}
```

### Flat Command List

```typescript
// Pre-allocated command array (never grows during frame)
const commandPool = new Array<RenderCommand>(4096)
let commandCount = 0

const resetCommands = () => { commandCount = 0 }
const addCommand = (cmd: RenderCommand) => { commandPool[commandCount++] = cmd }
```

### Sort Key Array

```typescript
// Pre-allocated sort key array for radix sort
const sortKeys = new BigUint64Array(4096)
const sortIndices = new Uint16Array(4096)
// Radix sort operates in-place on these arrays — no allocation
```

## Strategy 7: Efficient Frustum Culling

Frustum culling eliminates invisible objects before they reach the command list:

```
2000 scene objects → frustum cull → ~800 visible → sort → draw
```

The cull test per object is ~6 dot products (one per frustum plane) against the object's world AABB. With early-out on first failing plane, average cost is ~3-4 dot products.

**Cost: 2000 objects × ~50ns = ~0.1ms**

## Strategy 8: Dirty Flag Propagation

Only recompute world matrices when transforms change:

```typescript
interface Node {
    localDirty: boolean    // local TRS changed
    worldDirty: boolean    // this or any ancestor changed
    localMatrix: Float32Array
    worldMatrix: Float32Array
}

// Setting position/rotation/scale marks localDirty
// Parent worldDirty propagates to children
// Only dirty nodes recompute matrices in the update pass
```

For a scene with 2000 objects where 50 move per frame: **50 matrix recomputes instead of 2000**.

## Frame Budget Breakdown

Typical frame at 2000 draw calls on a mobile GPU:

| Phase | Cost (ms) | Notes |
|---|---|---|
| JS: Dirty check + matrix update | 0.2 | ~50 dirty nodes |
| JS: Frustum cull | 0.1 | 2000 AABB tests |
| JS: Command generation | 0.1 | Flat array write |
| JS: Radix sort | 0.1 | 2000 keys |
| JS: UBO data packing | 0.5 | 2000 × 256B write |
| JS: UBO upload | 0.3 | 512KB buffer write |
| JS: Draw submission | 4.0 | 2000 draws × 2µs |
| **JS Total** | **~5.3** | |
| GPU: Shadow maps (3 cascades) | 2.5 | ~500 casters per cascade |
| GPU: Opaque pass | 3.0 | 2000 draws, minimal overdraw |
| GPU: OIT pass | 0.5 | ~200 transparent objects |
| GPU: Post-process (bloom + tonemap) | 0.8 | 12 passes, small textures |
| GPU: MSAA resolve | 0.2 | Hardware resolve |
| **GPU Total** | **~7.0** | |
| Browser overhead | ~2.0 | Compositing, other tasks |
| **Frame Total** | **~14.3** | Under 16.6ms budget |

## Comparison with Three.js

| Metric | Three.js | Lynx | Improvement |
|---|---|---|---|
| Per-draw JS overhead | ~30 µs | ~2-3 µs | 10-15× |
| Uniform upload | Per-object calls | Single UBO | 1 call vs 2000 |
| State changes per frame | ~2000 | ~100-200 | 10-20× |
| Sort algorithm | `Array.sort` (O(n log n)) | Radix sort (O(n)) | ~3× for n=2000 |
| GC pressure per frame | ~50-200 KB | ~0 KB | Eliminates GC pauses |
| Transparency sorting | Per-object sort (broken) | OIT (correct, no sort) | Eliminates sort entirely |
| World matrix updates | All objects every frame | Only dirty objects | ~40× for mostly-static |
| Static objects | Re-submitted every frame | Render bundles (WebGPU) | Near-zero JS cost |

## Profiling API

```typescript
// Built-in frame stats
const stats = renderer.getStats()
// {
//     drawCalls: 1847,
//     triangles: 523000,
//     pipelineBinds: 8,
//     materialBinds: 127,
//     frustumCulled: 1153,
//     jsTime: 5.2,        // ms
//     gpuTime: 6.8,       // ms (WebGPU timestamp queries)
//     frameTime: 14.1,    // ms
// }
```

## Optimization Checklist for Users

1. **Share geometries** — Multiple meshes using the same geometry share VBOs and BVH
2. **Share materials** — Identical material configs should use the same material instance
3. **Use material index** — Instead of many materials, use a single mesh with `_materialindex` vertex attribute
4. **Mark static** — `mesh.static = true` enables render bundle caching (WebGPU)
5. **Avoid per-frame allocations** — In `useFrame` callbacks, pre-allocate vectors/matrices
6. **Use `intersectAny`** — For boolean raycasting (is there a wall between A and B?), use the any-hit variant
7. **LOD manually** — Replace distant meshes with simpler geometry by toggling visibility
8. **Texture sizes** — Use KTX2 with Basis transcoding for compressed GPU textures (4-8× less VRAM)
