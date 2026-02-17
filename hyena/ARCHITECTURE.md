# Architecture

## System Overview

Hyena is structured as a set of loosely coupled modules connected through a central `Engine` coordinator. The engine owns the render loop, the GPU device, and the frame timing. Everything else — scene, materials, animations, controls — plugs into the engine but does not depend on each other directly.

```
┌─────────────────────────────────────────────────────────────────────┐
│                            Engine                                   │
│  ┌─────────┐  ┌──────────┐  ┌───────────┐  ┌───────────────────┐  │
│  │  Clock   │  │  Input   │  │  Renderer │  │  AnimationSystem  │  │
│  └─────────┘  └──────────┘  └─────┬─────┘  └───────────────────┘  │
│                                   │                                 │
│              ┌────────────────────┼────────────────────┐            │
│              │                    │                    │            │
│        ┌─────▼─────┐    ┌────────▼────────┐   ┌──────▼──────┐     │
│        │  ShadowPass│    │  GeometryPass   │   │  PostFXPass │     │
│        │  (CSM)     │    │  (Opaque + OIT) │   │(Bloom,MSAA) │     │
│        └───────────┘    └─────────────────┘   └─────────────┘     │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │                      Device (GPU Abstraction)              │    │
│  │   ┌──────────────┐              ┌───────────────┐          │    │
│  │   │  WebGPUDevice │              │  WebGL2Device  │          │    │
│  │   └──────────────┘              └───────────────┘          │    │
│  └────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                            Scene                               │
│  ┌────────┐ ┌──────┐ ┌──────────────┐ ┌───────┐ ┌──────────┐ │
│  │ Object3D│ │ Mesh │ │ SkinnedMesh  │ │ Light │ │  Group   │ │
│  └────────┘ └──────┘ └──────────────┘ └───────┘ └──────────┘ │
│  ┌──────────────┐ ┌────────────┐ ┌──────────────────────┐     │
│  │  Geometries   │ │  Materials │ │  BVH (Spatial Index) │     │
│  └──────────────┘ └────────────┘ └──────────────────────┘     │
└────────────────────────────────────────────────────────────────┘
```

## Module Dependency Graph

```
math (zero dependencies)
  ↑
backend/Device (depends on: math)
  ↑
core/Renderer (depends on: backend, math)
  ↑
scene/* (depends on: math)
  ↑
materials/* (depends on: math, backend)
  ↑
geometries/* (depends on: math)
  ↑
animation/* (depends on: math, scene)
  ↑
shadows/* (depends on: backend, math, scene)
  ↑
postfx/* (depends on: backend, math)
  ↑
bvh/* (depends on: math, scene)
  ↑
loaders/* (depends on: scene, materials, geometries, animation)
  ↑
controls/* (depends on: math, scene)
  ↑
overlay/* (depends on: math, scene, core)
  ↑
react/* (depends on: everything above)
```

## Module Descriptions

### `math/`
Zero-dependency math library. `Vec3`, `Vec4`, `Mat4`, `Quat`, `AABB`, `Frustum`, `Ray`. All operate on `Float32Array` backing stores. A global `Pool` provides scratch temporaries for zero-allocation operations.

### `backend/`
GPU abstraction layer. Defines the `Device` interface with two implementations:

- **`WebGPUDevice`**: Uses `GPUDevice`, `GPURenderPipeline`, `GPUBindGroup`, `GPURenderBundle`.
- **`WebGL2Device`**: Uses `WebGL2RenderingContext`, VAOs, UBOs, FBOs.

The `Device` exposes:
- `createBuffer(data, usage)` → `GpuBuffer`
- `createTexture(desc)` → `GpuTexture`
- `createPipeline(desc)` → `GpuPipeline`
- `createBindGroup(layout, entries)` → `GpuBindGroup`
- `createRenderTarget(desc)` → `GpuRenderTarget`
- `beginFrame()` / `endFrame()`
- `draw(pipeline, bindGroups, vertexBuffers, indexBuffer, count)`

### `core/`
- **`Engine`**: Top-level coordinator. Owns the `Device`, `Renderer`, `Clock`, and the `requestAnimationFrame` loop. Provides `engine.run(scene)` to start and `engine.stop()` to halt.
- **`Renderer`**: Orchestrates render passes. Walks the scene's render list, executes shadow pass, geometry pass (opaque then transparent OIT), post-processing pass.
- **`Clock`**: Frame timing. Provides `deltaTime`, `elapsedTime`, `frameCount`. Fixed-precision to avoid floating-point drift.

### `scene/`
- **`Object3D`**: Base class. Has `position` (Vec3), `rotation` (Quat), `scale` (Vec3). Computes `localMatrix` and `worldMatrix`. Dirty flag system. Parent/children tree. Each Object3D has an index into the global transforms SoA array.
- **`Group`**: Empty Object3D container for organizing hierarchy.
- **`Mesh`**: Object3D + Geometry + Material. Holds a `boundingBox` for frustum culling.
- **`SkinnedMesh`**: Mesh + Skeleton. References a `Skeleton` which holds the bone hierarchy and joint matrices.
- **`Scene`**: Root Object3D. Owns the flat render list, light list, and scene-level BVH.

### `materials/`
- **`BasicMaterial`**: Unlit. Color × texture. Supports vertex colors and material index mapping.
- **`LambertMaterial`**: Diffuse N·L lighting. Color, AO texture, color texture. Supports vertex colors, material index, and emissive per index.
- Materials compile to GPU pipelines on first use and cache them.

### `geometries/`
Parametric generators producing indexed vertex buffers:
`PlaneGeometry`, `BoxGeometry`, `SphereGeometry`, `ConeGeometry`, `CylinderGeometry`, `CapsuleGeometry`, `CircleGeometry`.

Each outputs: `positions` (Float32Array), `normals` (Float32Array), `uvs` (Float32Array), `indices` (Uint16Array or Uint32Array).

### `animation/`
- **`AnimationClip`**: A set of channels, each targeting a bone's position, rotation, or scale over time.
- **`AnimationMixer`**: Manages active animations on a skeleton. Supports playing, stopping, and crossfading between two clips.
- **`Skeleton`**: Bone hierarchy. Computes joint matrices each frame for GPU skinning.

### `lights/`
- **`AmbientLight`**: Flat color added to all fragments.
- **`DirectionalLight`**: Infinite directional light. Direction vector + color + intensity. Drives the CSM shadow system.

### `shadows/`
- **`CascadingShadowMap`**: 3 cascades. Computes cascade splits, renders depth from light's perspective into a shadow atlas, provides shadow sampling data to the geometry pass.

### `postfx/`
- **`BloomPass`**: Threshold extraction → downsample chain → upsample chain → additive composite.
- **`MSAAResolve`**: Resolves MSAA render target to single-sample for post-processing.
- **`OITComposite`**: Composites weighted-blended OIT accumulation/revealage onto the opaque buffer.

### `bvh/`
- **`BVH`**: SAH-based Bounding Volume Hierarchy. Flat array layout. Supports `raycast(ray)`, `frustumQuery(frustum)`, `nearestPoint(point)`. Built from mesh AABBs. Supports incremental refit for moving objects.

### `loaders/`
- **`GLTFLoader`**: Parses `.gltf`/`.glb` files. Extracts meshes, materials, skeletons, animations. Applies Y→Z up rotation. Reads `_materialindex` vertex attribute.
- **`DracoDecoder`**: WASM-based Draco mesh decompression in a Web Worker.
- **`BasisDecoder`**: WASM-based Basis Universal / KTX2 texture transcoding in a Web Worker. Selects optimal compressed format per GPU (BC7, ETC2, ASTC).

### `controls/`
- **`OrbitControls`**: Pointer/touch-driven orbit, pan, zoom around a target point. Damping, min/max distance, angle limits.

### `overlay/`
- **`HtmlOverlay`**: Manages a pool of absolutely-positioned DOM elements. Each frame, projects their 3D `worldPosition` to screen coordinates via the camera's view-projection matrix and applies a CSS `transform: translate(x, y)`. Optional depth-based occlusion.

### `react/`
- **Reconciler**: Custom `react-reconciler` host config that maps JSX elements to Hyena scene objects.
- **`<Canvas>`**: Root component. Creates `Engine`, provides context.
- **Hooks**: `useFrame(callback)`, `useEngine()`, `useScene()`, `useLoader(loader, url)`.
- **Automatic disposal**: Geometries, materials, textures are disposed when their component unmounts.

## Initialization Flow

```
1. Engine.create({ canvas })
   ├─ Detect WebGPU support
   ├─ Create WebGPUDevice or WebGL2Device
   ├─ Create Renderer (shadow pass, geometry pass, post-fx pass)
   ├─ Create Clock
   └─ Return Engine instance

2. engine.run(scene)
   ├─ Build initial render list from scene graph
   ├─ Build scene BVH from mesh AABBs
   ├─ Start requestAnimationFrame loop:
   │   ├─ clock.tick()
   │   ├─ scene.updateWorldMatrices()    // dirty flags only
   │   ├─ animationSystem.update(dt)     // advance animations
   │   ├─ controls.update(dt)            // orbit controls
   │   ├─ frustumCull(scene, camera)     // mark visible objects
   │   ├─ renderer.render(scene, camera) // execute render passes
   │   └─ overlay.update(camera)         // project HTML elements
   └─ Loop continues until engine.stop()
```

## Threading Model

The main thread runs the render loop and scene logic. Heavy work is offloaded to Web Workers:

- **Draco decoding**: A dedicated worker loads the Draco WASM module and decodes compressed mesh buffers. Results are transferred back via `Transferable` arrays (zero-copy).
- **Basis transcoding**: A dedicated worker transcodes KTX2/Basis textures to the optimal GPU compressed format. Results transferred as `ArrayBuffer`.
- **BVH construction** (optional): For very large scenes, the initial BVH build can be offloaded to a worker.

All worker communication uses `postMessage` with `Transferable` objects to avoid copying large buffers.
