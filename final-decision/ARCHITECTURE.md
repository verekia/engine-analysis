# ARCHITECTURE — Final Decision

## Layered Architecture

**Decision: Strict layered architecture with one-way dependencies** (universal agreement)

```
React Bindings          (engine-react package, leaf)
    |
Controls / HTML Overlay (leaf modules)
    |
Animation / Loaders     (feature modules)
    |
Renderer / Post-FX      (rendering pipeline)
    |
Scene / Materials        (scene representation)
    |
Spatial / Lighting       (spatial queries, light data)
    |
Device                   (GPU abstraction layer)
    |
Math                     (zero dependencies)
```

Lower layers never import from upper layers. Math depends on nothing. React is a leaf module — nothing depends on it.

## Module Organization

**Decision: Monolithic single package with internal modules** (5/9: Hyena, Rabbit, Shark, Wren, Bonobo)

```
engine/
├── src/
│   ├── math/
│   ├── device/       # GPU Abstraction Layer
│   ├── renderer/
│   ├── scene/
│   ├── materials/
│   ├── geometry/
│   ├── animation/
│   ├── loaders/
│   ├── lighting/
│   ├── spatial/      # BVH, raycasting, frustum culling
│   ├── postfx/
│   ├── controls/
│   └── overlay/
└── package.json
```

React bindings export from a separate `engine-react` package with `react` and `react-reconciler` as peer dependencies.

## Data Storage Philosophy

**Decision: Object-oriented public API with SoA internals** (6/9: Caracal, Fennec, Hyena, Lynx, Rabbit, Wren)

```typescript
// Public API: familiar object-oriented
const mesh = createMesh(geometry, material)
mesh.position.set(1, 0, 3)
scene.add(mesh)

// Internal: renderer reads flat arrays directly
// worldMatrices: Float32Array - contiguous, GPU-uploadable
// sortKeys: BigUint64Array - 64-bit sort keys for draw call ordering
// visibilityFlags: Uint8Array - frustum culling results
```

Users interact with objects that have named properties. Setting `position` marks the node dirty. The renderer never walks the object graph — it iterates flat typed arrays. Objects maintain an index into these arrays, and dirty flags trigger selective updates.

### Why Not Pure SoA (Bonobo, Mantis, Shark)?

Pure SoA is maximally cache-friendly but makes the public API unergonomic: entities referenced by integer IDs, parent-child via index arrays, debugging requires inspecting raw Float32Arrays. The overhead of thin object wrappers is negligible compared to the DX improvement.

### Why Not Pure OOP?

Per-node matrix storage scattered across heap objects is cache-hostile for batch operations (frustum culling 2000 nodes, uploading 2000 matrices). Contiguous arrays maintain performance where it matters.

## Scene Graph Update Strategy

**Decision: Deferred dirty propagation with depth-first update** (Caracal, Fennec, Shark approach)

Two-level dirty flags:
- `_dirtyLocal`: local matrix needs recomputation from position/rotation/scale
- `_dirtyWorld`: world matrix needs recomputation from parent chain

```typescript
// Setting a property just marks dirty (O(1))
node.position.set(1, 2, 3)  // sets _dirtyLocal = true

// Once per frame, before rendering:
propagateDirtyFlags(root)     // mark children dirty if parent dirty
updateWorldMatrices(root)     // recompute only dirty nodes (depth-first)
```

Early-exit optimization: if a node is already marked `_dirtyWorld`, skip propagation to its subtree (prevents O(n²) cascading).

## Render List Management

**Decision: Rebuild every frame** (5/9: Bonobo, Fennec, Hyena, Rabbit, Wren)

```typescript
// Each frame:
const visibleList = frustumCull(scene, camera)  // collect visible meshes
radixSort(visibleList, sortKeyOf)               // sort by composite key
buildDrawCommands(visibleList)                  // emit draw calls
```

For 2000 objects, culling + radix sort completes in <0.3ms. Simpler, always correct, easy to debug. The reduced complexity outweighs the marginal performance gain of persistent sorted lists.

## Uniform Buffer Strategy

**Decision: Dynamic offsets on a shared buffer** (universal agreement for per-object data)

Three bind groups by update frequency:

| Slot | Name | Update Frequency | Contents |
|------|------|-----------------|----------|
| 0 | Per-frame | Once per frame | Camera VP, lights, shadow matrices |
| 1 | Per-material | Per material change | Textures, palette, material params |
| 2 | Per-object | Per draw call (offset only) | World matrix, bone matrices |

Per-object data uses dynamic offsets into a shared buffer. This avoids per-object bind group creation and minimizes binding overhead.

```typescript
// WebGPU: setBindGroup(2, objectBindGroup, [byteOffset])
// WebGL2: gl.bindBufferRange(GL.UNIFORM_BUFFER, 2, buffer, byteOffset, size)
```

The uniform buffer is sized for the maximum expected visible objects (e.g., 4096 × 256 bytes = 1MB). If the scene exceeds this, the buffer grows geometrically.

## Frame Lifecycle

**Decision: Standard frame lifecycle** (universal agreement)

```
1.  Time/clock update (deltaTime, elapsed)
2.  User callbacks (useFrame / onFrame)
3.  Animation system update (sample keyframes, blend, compute bone matrices)
4.  Dirty flag propagation (mark children of dirty nodes)
5.  World matrix recomputation (only dirty nodes)
6.  Frustum culling (test AABBs against 6 planes)
7.  Build draw command list (collect visible meshes)
8.  Sort draw commands (radix sort by 64-bit key)
9.  Upload per-frame uniforms (camera, lights, shadow matrices)
10. Execute render passes:
      a. Shadow pass (3 CSM cascades, depth-only)
      b. Opaque pass (sorted, MSAA, MRT for emissive)
      c. Transparent pass (OIT accumulation, depth read-only)
11. MSAA resolve
12. OIT composite (full-screen)
13. Bloom (downsample chain → upsample chain → composite)
14. Tone mapping + final blit to screen
15. HTML overlay update (project 3D positions to screen)
16. Reset scratch pools
```

## GPU Abstraction Layer

**Decision: WebGPU-modeled abstraction named "Device"** (simpler naming, follows WebGPU's own convention)

Both WebGPU and WebGL2 backends implement the same interface:

```typescript
interface Device {
  readonly backend: 'webgpu' | 'webgl2'

  createBuffer(desc: BufferDescriptor): Buffer
  createTexture(desc: TextureDescriptor): Texture
  createSampler(desc: SamplerDescriptor): Sampler
  createShaderModule(desc: ShaderDescriptor): ShaderModule
  createPipeline(desc: PipelineDescriptor): Pipeline
  createBindGroup(desc: BindGroupDescriptor): BindGroup

  beginFrame(): FrameEncoder
  submit(): void
}
```

Key concepts:
- **Immutable pipelines**: full render state baked at creation
- **Explicit bind groups**: resources grouped by update frequency
- **Command recording**: explicit begin/end passes, even on WebGL2

WebGL2 backend implements this with:
- **Full state caching** (8/9 agree): track current program, VAO, FBO, bound textures, UBOs, blend/depth/cull state. Skip redundant GL calls.
- **Pipeline = GL state bundle**: `useProgram` + blend + depth + cull state applied together
- **BindGroup = UBO bindings + texture units**: applied via `bindBufferRange` and `activeTexture`/`bindTexture`
- **VAO caching**: per geometry+pipeline pair, lazily created

## Module Communication

- **Direct function calls** between layers (no event bus, no pub/sub for hot paths)
- **Typed events** for infrequent operations (asset loaded, resize, context lost)
- **Shared typed arrays** for renderer-facing data (world matrices, sort keys, bone matrices)

## Memory Management

- **Pre-allocated pools** for math scratch variables (module-level `_tempVec3`, `_tempMat4`)
- **Frame-scoped scratch pool** with pointer reset at frame end (Mantis, Lynx approach)
- **Zero allocations in the render loop** — no `new`, no array creation, no closures in hot paths
- **GPU resources live until explicitly disposed** — no automatic GC of WebGL/WebGPU objects
- **Geometric buffer growth** for dynamic arrays (double capacity when exceeded)

## Threading Model

- **Main thread**: Scene graph, render loop, user callbacks, GPU command submission
- **Web Workers**: Draco mesh decoding, Basis texture transcoding, optional BVH construction
- **Communication**: `postMessage` with `Transferable` typed arrays (zero-copy)
- **Worker pool**: 2-4 workers scaled to `navigator.hardwareConcurrency`

The render loop stays on the main thread. OffscreenCanvas would prevent HTML overlay compositing, and the synchronization overhead is not justified given the 16.6ms frame budget.

## Error Handling

- **Debug mode** (`createEngine(canvas, { debug: true })`): verbose validation errors, shader compilation diagnostics, missing attribute warnings
- **Production mode**: silent failures with console.warn for recoverable issues, throws for unrecoverable issues (no WebGL2/WebGPU support)
- **Backend fallback**: try WebGPU first, fall back to WebGL2 automatically, throw if neither available
