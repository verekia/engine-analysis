# ARCHITECTURE - Design Decisions

## Layered Architecture

**Decision: Strict layered architecture with one-way dependencies**

- Sources: All 9 implementations agree

```
React Bindings (top, leaf module)
    ↓
High-Level API / Controls / Overlay
    ↓
Rendering Pipeline / Features
    ↓
GPU Abstraction Layer (GAL)
    ↓
Math (bottom, zero dependencies)
```

Lower layers never depend on upper layers. Math depends on nothing. The GPU layer depends only on math and native APIs.

## Module Organization

**Decision: Monolithic single package with internal modules**

- Sources: Hyena, Rabbit, Shark, Wren, Bonobo (5/9)
- Rejected: Multi-package monorepo (Fennec, Caracal, Lynx, Mantis)

```
engine/
├── src/
│   ├── math/
│   ├── gal/          # GPU Abstraction Layer
│   ├── renderer/
│   ├── scene/
│   ├── materials/
│   ├── geometry/
│   ├── animation/
│   ├── loaders/
│   ├── spatial/      # BVH, raycasting, frustum culling
│   ├── controls/
│   ├── overlay/
│   └── react/        # Separate entry point
└── package.json
```

Rationale: Single package is simpler to distribute and version. Internal module boundaries still enforce separation of concerns. React bindings exported via a separate entry point (`engine/react`) so they're tree-shaken away for non-React users.

## Data Storage Philosophy

**Decision: Object-oriented public API with SoA internals**

- Sources: Caracal, Fennec, Hyena, Lynx, Rabbit, Wren (6/9)
- Rejected: Pure flat SoA (Bonobo, Mantis, Shark) — less flexible, harder to use

```typescript
// Public API: familiar object-oriented
const mesh = createMesh(geometry, material)
mesh.position.set(1, 0, 3)
mesh.rotation = quatFromEuler(0, 0, Math.PI / 4)

// Internal: flat arrays for rendering hot path
worldMatrices: Float32Array(activeNodeCount * 16)
```

Rationale: Familiar API for developers coming from Three.js. Internally, the renderer accesses flat typed arrays directly for cache-friendly iteration and zero-copy GPU upload. The wrapper objects keep the two representations in sync via dirty flags.

## Scene Graph Update Strategy

**Decision: Deferred dirty propagation with depth-first update**

- Sources: Combines Bonobo/Fennec deferred marking with majority depth-first update

```typescript
// Setting a property just marks dirty (O(1))
node.position.set(1, 2, 3)  // sets _dirtyLocal = true

// Once per frame, before rendering:
propagateDirtyFlags(root)     // mark children dirty if parent dirty
updateWorldMatrices(root)     // recompute only dirty nodes (depth-first)
```

Uses two-level dirty flags (Caracal, Fennec, Shark):
- `_dirtyLocal`: local matrix needs recomputation from position/rotation/scale
- `_dirtyWorld`: world matrix needs recomputation from parent

Early-exit optimization: skip propagation to children if they're already marked dirty.

## Render List Management

**Decision: Rebuild every frame**

- Sources: Bonobo, Fennec, Hyena, Rabbit, Wren (5/9)
- Rejected: Persistent sorted list (Caracal, Lynx, Mantis, Shark)

```typescript
// Each frame:
const visibleList = frustumCull(scene, camera)  // collect visible meshes
radixSort(visibleList, sortKeyOf)               // sort by composite key
buildDrawCommands(visibleList)                  // emit draw calls
```

Rationale: Simpler, always correct, easy to debug. For 2000 objects, culling + radix sort completes in <0.3ms. The reduced complexity outweighs the marginal performance gain of persistent lists.

## Uniform Buffer Strategy

**Decision: Triple-buffered ring buffer with dynamic offsets**

- Sources: Mantis, Shark, Wren (3/9 for ring buffer); all 9 for dynamic offsets

```typescript
const RING_SIZE = 4 * 1024 * 1024  // 4MB
const FRAME_COUNT = 3               // Triple buffering

// Each frame writes to a new region
const offset = (frameIndex % FRAME_COUNT) * (RING_SIZE / FRAME_COUNT)

// Per-object: just advance offset
for (const object of visibleObjects) {
  writeAt(offset + currentOffset, object.uniforms)
  setBindGroup(2, sharedObjectBindGroup, [currentOffset])
  currentOffset += alignedUniformSize
}
```

Rationale: Zero per-object buffer allocation. GPU never stalls on previous frames. Single large allocation amortized over entire frame. Dynamic offsets minimize bind group switching.

## Frame Lifecycle

**Decision: Standard frame lifecycle (universal pattern)**

```
1. Clock/time update
2. User callbacks (onFrame / useFrame)
3. Animation system update (sample keyframes, compute bone matrices)
4. Dirty flag propagation
5. World matrix recomputation (dirty nodes only)
6. Frustum culling
7. Build render list + radix sort
8. Upload per-frame uniforms (camera, lights, shadow matrices)
9. Execute render passes:
   a. Shadow maps (3 CSM cascades, depth-only)
   b. Opaque geometry (sorted, MSAA, MRT for emissive)
   c. Transparent geometry (OIT accumulation)
   d. MSAA resolve
   e. OIT composite
   f. Bloom (downsample → upsample)
   g. Final blit (tone mapping → screen)
10. HTML overlay position update
11. Reset scratch pools / advance ring buffer frame
```

## GPU Abstraction Interface

**Decision: WebGPU-modeled abstraction named "GAL" (GPU Abstraction Layer)**

- Sources: GAL naming from Caracal, Fennec, Lynx, Mantis (4/9 — most common)
- Interface follows universal pattern (~95% identical across all 9)

```typescript
interface GALDevice {
  readonly backend: 'webgpu' | 'webgl2'

  createBuffer(desc: BufferDescriptor): GALBuffer
  createTexture(desc: TextureDescriptor): GALTexture
  createSampler(desc: SamplerDescriptor): GALSampler
  createShaderModule(desc: ShaderDescriptor): GALShaderModule
  createPipeline(desc: PipelineDescriptor): GALPipeline
  createBindGroup(desc: BindGroupDescriptor): GALBindGroup

  beginFrame(): GALCommandEncoder
  submit(): void
}
```

Key concepts:
- Immutable pipelines (full render state baked at creation)
- Explicit bind groups (resources grouped by update frequency)
- Command recording (explicit begin/end passes, even on WebGL2)

## Threading Model

**Decision: Main thread rendering + Web Workers for asset decoding**

- Sources: All 9 implementations agree

Main thread: scene graph, render loop, user callbacks, GPU submission
Web Workers: Draco mesh decoding, Basis texture transcoding, optional BVH construction

Communication via `postMessage` with `Transferable` typed arrays for zero-copy transfer.

## Memory Management

**Decision: Pre-allocated pools, zero allocations in render loop**

- Sources: All 9 implementations agree
- Module-level scratch variables for math temporaries
- Frame-scoped pool with reset for temporary vectors (Hyena, Mantis, Lynx approach)
- Pre-allocated command arrays
- No `new` calls in hot paths
- GPU resources live until explicitly destroyed via `dispose()`
