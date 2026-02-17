# ARCHITECTURE.md - System Architecture

## Layered Architecture

Strict one-way dependency flow with no circular dependencies:

```
React Bindings          (engine-react package)
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

Lower layers never import from upper layers. Math depends on nothing. React is a leaf module that depends on everything but nothing depends on it.

## Data Storage: OOP Surface, SoA Core

**Decision: Object-oriented public API backed by Structure-of-Arrays internally** (Caracal, Fennec, Hyena, Lynx, Rabbit, Wren approach over Bonobo's pure SoA)

### Public API (Object Graph)

```typescript
const mesh = createMesh(geometry, material)
mesh.position.set(1, 0, 3)
scene.add(mesh)
group.add(mesh)
```

Users interact with objects that have properties. Setting `position` marks the node dirty. This is intuitive, debuggable, and familiar to anyone coming from Three.js.

### Internal Representation (Flat Arrays)

```typescript
// Renderer-facing data in contiguous typed arrays
worldMatrices: Float32Array   // All world matrices packed for GPU upload
sortKeys: BigUint64Array      // 64-bit sort keys for draw call ordering
visibilityFlags: Uint8Array   // Frustum culling results
```

The renderer never walks the object graph. It iterates flat arrays. Objects maintain an index into these arrays, and dirty flags trigger selective updates.

### Why Not Pure SoA (Bonobo)?

Pure SoA is maximally cache-friendly but makes the public API unergonomic. Users must reference entities by integer IDs. Parent-child relationships require separate index arrays. Debugging is harder (inspect a Float32Array vs inspect an object with named properties). The overhead of thin object wrappers is negligible compared to the DX improvement.

### Why Not Pure OOP?

Per-node matrix storage scattered across heap objects is cache-hostile for batch operations (frustum culling 2000 nodes, uploading 2000 matrices). Keeping renderer-critical data in contiguous arrays maintains performance where it matters.

## Frame Lifecycle

Every frame follows this sequence (universal agreement across all 9 implementations):

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
13. Bloom (downsample chain -> upsample chain -> composite)
14. Tone mapping + final blit to screen
15. HTML overlay update (project 3D positions to screen)
16. Reset scratch pools
```

## GPU Abstraction Layer

**Decision: Name it "Device"** (Hyena, Shark approach - direct, no unnecessary acronyms)

The abstraction is modeled after WebGPU's API. WebGL2 implements the same interface via state caching and emulation.

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

WebGL2 backend wraps this with:
- **Full state caching** (8/9 implementations agree this is necessary): track current program, VAO, FBO, bound textures, UBOs, blend/depth/cull state. Skip redundant GL calls.
- **Pipeline = GL state bundle**: `useProgram` + blend + depth + cull state applied together.
- **BindGroup = UBO bindings + texture units**: Applied via `bindBufferRange` and `activeTexture`/`bindTexture`.
- **VAO caching**: per geometry+pipeline pair, lazily created.

## Module Communication

- **Direct function calls** between layers (no event bus, no pub/sub for hot paths)
- **Typed events** for infrequent operations (asset loaded, resize, context lost)
- **Shared typed arrays** for renderer-facing data (world matrices, sort keys, bone matrices)

## Memory Management

- **Pre-allocated pools** for math scratch variables (module-level `_tempVec3`, `_tempMat4`)
- **Frame-scoped scratch pool** with reset at frame end (Mantis, Lynx approach)
- **Zero allocations in the render loop** - no `new`, no array creation, no closures in hot paths
- **GPU resources live until explicitly disposed** - no automatic GC of WebGL/WebGPU objects
- **Geometric buffer growth** for dynamic arrays (double capacity when exceeded)

## Threading Model

- **Main thread**: Scene graph, render loop, user callbacks, GPU command submission
- **Web Workers**: Draco mesh decoding, Basis texture transcoding, optional BVH construction
- **Communication**: `postMessage` with `Transferable` typed arrays (zero-copy)
- **Worker pool**: 2-4 workers scaled to `navigator.hardwareConcurrency` for parallel asset decoding

The render loop stays on the main thread. OffscreenCanvas would prevent HTML overlay compositing, and the synchronization overhead with workers is not justified given the 16.6ms frame budget.

## Error Handling

- **Debug mode** (`createEngine(canvas, { debug: true })`): verbose validation errors, shader compilation diagnostics, missing attribute warnings
- **Production mode**: silent failures with console.warn for recoverable issues, throws for unrecoverable issues (no WebGL2/WebGPU support)
- **Backend fallback**: Try WebGPU first, fall back to WebGL2 automatically, throw if neither available
