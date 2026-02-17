# Architecture

## Module Structure

Mantis is organized as a layered set of modules with strict dependency ordering.
No circular dependencies exist. Each module is a directory under `src/` exporting
a public API through an `index.ts` barrel file.

```
src/
├── math/           # Vec3, Vec4, Mat4, Quat, AABB, Frustum, Pool
├── gal/            # GPU Abstraction Layer (interfaces + backends)
│   ├── types.ts    # Shared types: Pipeline, Buffer, Texture, BindGroup, RenderPass
│   ├── webgpu/     # WebGPU backend implementation
│   └── webgl2/     # WebGL2 backend implementation
├── renderer/       # Command buffer, draw sorting, ring buffer, frame orchestration
├── scene/          # Scene graph, Node, Mesh, Group, Camera, Light
├── materials/      # BasicMaterial, LambertMaterial, shader variants, material index
├── geometry/       # Parametric generators: Plane, Box, Sphere, Cone, Cylinder, Capsule, Circle
├── animation/      # Skeleton, AnimationClip, AnimationMixer, crossfade
├── shadows/        # CascadedShadowMap, cascade splitting, atlas management
├── postfx/         # Bloom pipeline, OIT composite, final blit
├── bvh/            # BVH build (SAH), traversal, scene BVH, mesh BVH
├── assets/         # GLTFLoader, DracoDecoder (worker), BasisTranscoder (worker)
├── controls/       # OrbitControls
├── overlay/        # HtmlOverlay, projection, occlusion
├── react/          # React reconciler, components, hooks
└── index.ts        # Public API barrel
```

## Dependency Graph

Dependencies flow strictly downward. A module may only import from modules
listed below it.

```
Level 0:  math
Level 1:  gal                         (depends on: math)
Level 2:  renderer                    (depends on: math, gal)
Level 3:  scene, materials, geometry  (depends on: math, gal, renderer)
Level 4:  animation, shadows          (depends on: math, scene, materials, renderer)
Level 5:  postfx, bvh                 (depends on: math, gal, renderer, scene)
Level 6:  assets                      (depends on: math, scene, materials, geometry, animation, bvh)
Level 7:  controls, overlay           (depends on: math, scene, renderer)
Level 8:  react                       (depends on: all of the above)
```

No module at level N imports from level N or higher. This prevents cycles and
enables tree-shaking — an app that does not use React bindings or the HTML
overlay pays zero cost for those modules.

## Engine Lifecycle

The `Engine` class is the root coordinator. It owns the GAL device, the
renderer, the scene, and all subsystems. Its lifecycle:

```typescript
// 1. Creation — auto-detects WebGPU or falls back to WebGL2
const engine = await createEngine({
  canvas: document.getElementById('canvas') as HTMLCanvasElement,
  antialias: true,     // 4× MSAA
  shadows: true,       // enable CSM
  bloom: true,         // enable bloom post-process
})

// 2. Frame loop — driven by requestAnimationFrame
engine.onFrame((dt, elapsed) => {
  // user logic here
})
engine.start()

// 3. Disposal
engine.dispose()
```

### Per-Frame Execution Order

Every frame executes this exact sequence:

```
1. Time        — compute delta, advance elapsed
2. User logic  — call onFrame callbacks
3. Animation   — advance mixers, compute joint matrices
4. Scene       — propagate dirty flags, recompute world matrices (breadth-first)
5. Culling     — extract frustum planes, test AABBs (BVH-accelerated)
6. Sort        — build 64-bit draw keys, radix sort
7. Upload      — write per-frame UBO (camera, lights), write ring buffer (per-object)
8. Shadow pass — render 3 cascade depth maps
9. Opaque pass — execute sorted opaque draw commands
10. OIT pass   — render transparent objects to accumulation + revealage targets
11. Resolve    — resolve MSAA
12. Post-FX    — composite OIT, bloom (downsample → upsample), final blit
13. Overlay    — update HTML element positions
14. Pool reset — reset scratch math pool write pointers
```

Steps 1–7 are CPU-bound. Steps 8–12 are GPU-bound. The CPU work for frame N+1
can overlap with GPU work for frame N via command buffer submission.

## Module Communication

Modules communicate through three mechanisms:

### 1. Direct method calls (synchronous, same frame)

The renderer calls scene graph methods to read transforms. The scene graph
calls material methods to read pipeline states. These are plain function calls
with no event system overhead.

### 2. Typed events (asynchronous, deferred)

A minimal typed event emitter for things that do not need same-frame response:

```typescript
type EngineEvents = {
  resize: { width: number, height: number }
  contextLost: void
  contextRestored: void
}

// Usage
engine.on('resize', ({ width, height }) => { /* ... */ })
```

### 3. Shared typed arrays (zero-copy data flow)

The scene graph stores transforms in contiguous `Float32Array` buffers. The
renderer reads these directly when uploading to the ring buffer. No intermediate
objects are created.

```
Scene SoA arrays ──→ Renderer ring buffer ──→ GPU uniform buffer
       (read)              (write)              (bind + draw)
```

## Backend Selection

On `createEngine`, Mantis probes for WebGPU support:

```typescript
const createEngine = async (opts: EngineOptions): Promise<Engine> => {
  let device: GALDevice

  if (navigator.gpu) {
    const adapter = await navigator.gpu.requestAdapter()
    if (adapter) {
      const gpuDevice = await adapter.requestDevice()
      device = new WebGPUDevice(gpuDevice, opts.canvas)
    }
  }

  if (!device) {
    const gl = opts.canvas.getContext('webgl2')
    if (!gl) throw new Error('Neither WebGPU nor WebGL2 is supported')
    device = new WebGL2Device(gl)
  }

  return new Engine(device, opts)
}
```

The GAL device interface is identical for both backends. All downstream code
interacts only with the GAL types — `Pipeline`, `Buffer`, `Texture`,
`BindGroup`, `RenderPass` — never with raw WebGPU or WebGL2 objects.

## Memory Model

### Allocation Strategy

Mantis allocates all long-lived buffers at initialization or object creation.
The render loop itself allocates **zero** heap objects. This eliminates GC
pressure during gameplay.

| Data | Allocation | Lifetime |
|---|---|---|
| SoA transform arrays | On scene creation, grown via doubling | Scene lifetime |
| Vertex/index buffers | On geometry creation | Geometry lifetime |
| Uniform ring buffer | On engine creation (4 MB) | Engine lifetime |
| Scratch math pool | On engine creation (fixed) | Engine lifetime, reset each frame |
| Command buffer array | On engine creation, grown via doubling | Engine lifetime |
| Sort key array | On engine creation, grown via doubling | Engine lifetime |

### Ring Buffer Design

The uniform ring buffer is the key to efficient per-object data upload:

```
┌──────────────────────────────────────────────────────────┐
│                    4 MB GPU Buffer                        │
├─────────┬─────────┬─────────┬───────────────────────────┤
│ Frame 0 │ Frame 1 │ Frame 2 │       (unused)            │
│ objects │ objects │ objects │                             │
├─────────┴─────────┴─────────┴───────────────────────────┤
│  ↑ writePtr                                              │
└──────────────────────────────────────────────────────────┘
```

Each frame, the renderer writes all per-object uniforms (model matrix + material
data) sequentially. A write pointer advances through the buffer. When it reaches
the end, it wraps to the beginning (ring behavior). Three frames of data are
kept alive simultaneously to avoid stalling on in-flight GPU work.

## Threading Model

- **Main thread**: Scene graph, render loop, user callbacks, input events
- **Worker pool** (2 workers): Draco decoding, Basis transcoding

Workers use `Transferable` typed arrays for zero-copy data return. The main
thread never blocks on worker results — asset loading is fully async via
Promises.

## Error Handling

- GPU context loss triggers `contextLost` event and pauses the render loop
- On context restore, all GPU resources are recreated from retained CPU-side data
- GLTF/texture loading errors reject the loader Promise with descriptive errors
- Invalid material/geometry configurations throw during creation, not during rendering
