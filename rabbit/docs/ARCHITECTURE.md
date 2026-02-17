# Architecture Overview

## System Layers

Rabbit is organized into five layers, where each layer depends only on layers below it:

```
┌─────────────────────────────────────────────────────┐
│  React Bindings    (react/)                         │
│  Reconciler, <Canvas>, components, hooks            │
├─────────────────────────────────────────────────────┤
│  High-Level API    (core/, controls/, overlay/)     │
│  Engine, Scene, Camera, OrbitControls, HtmlOverlay  │
├─────────────────────────────────────────────────────┤
│  Features          (scene/, materials/, animation/, │
│                     loaders/, raycasting/, postfx/) │
│  Nodes, Materials, Geometries, Loaders, BVH, Bloom │
├─────────────────────────────────────────────────────┤
│  Renderer          (hal/)                           │
│  HAL Device, Buffers, Textures, Pipelines, Passes  │
├─────────────────────────────────────────────────────┤
│  Math              (math/)                          │
│  Vec3, Mat4, Quat, AABB, Frustum, Ray, Plane       │
└─────────────────────────────────────────────────────┘
```

## Module Boundaries

### `math/`

Pure math types with no dependencies. All types are mutable for performance (methods mutate in-place and return `this` for chaining). No heap allocations in math operations — callers pass pre-allocated outputs.

Key types:
- `Vec3` — 3-component vector (backed by `Float32Array(3)`)
- `Mat4` — 4×4 matrix (backed by `Float32Array(16)`, column-major)
- `Quat` — quaternion (backed by `Float32Array(4)`)
- `AABB` — axis-aligned bounding box (min Vec3, max Vec3)
- `Frustum` — 6 planes extracted from view-projection matrix
- `Ray` — origin + direction for raycasting
- `MathPlane` — plane defined by normal + distance

The coordinate system is **Z-up, right-handed** throughout. The `Mat4` class provides `lookAt`, `perspective`, and `orthographic` methods assuming Z-up.

### `hal/`

Hardware Abstraction Layer. Provides a unified interface over WebGL2 and WebGPU. The interface is modeled after WebGPU's descriptor-based design because it is cleaner and maps naturally to both backends.

Key abstractions:
- `HalDevice` — creation of GPU resources, capability queries
- `HalBuffer` — vertex, index, and uniform buffers
- `HalTexture` — 2D textures, render targets, depth textures, texture arrays
- `HalSampler` — sampling state (filtering, wrapping, comparison)
- `HalShader` — compiled shader programs
- `HalPipeline` — full render pipeline state (shader + vertex layout + blend + depth/stencil)
- `HalBindGroup` — grouped resource bindings (uniforms, textures, samplers)
- `HalRenderPass` — render pass with attachments (color, depth, resolve for MSAA)
- `HalCommandEncoder` — command recording (WebGPU) / immediate execution (WebGL2)

The two backend implementations:
- `WebGL2Device` — maps HAL calls to WebGL2 state changes. Caches state to avoid redundant `gl.*` calls. Uses VAOs, UBOs, and `OES_draw_buffers_indexed` where available.
- `WebGPUDevice` — thin wrapper around native WebGPU. Supports render bundles for cached draw call sequences.

### `scene/`

Scene graph nodes and transform hierarchy.

- `Node` — base class with local/world transforms, parent/children, dirty flags
- `Group` — container node (no renderable)
- `Mesh` — geometry + material reference, AABB for culling
- `SkinnedMesh` — extends Mesh with skeleton binding
- `Bone` — node within a skeleton hierarchy
- `DirectionalLight` — direction + color + intensity + shadow config
- `AmbientLight` — color + intensity
- `Camera` — perspective or orthographic projection

### `materials/`

Material definitions and shader generation.

- `BasicMaterial` — unlit, supports color map, vertex colors, material index colors
- `LambertMaterial` — diffuse lighting (N·L), supports color map, AO map, vertex colors, material index colors, emissive
- Shader chunks are composed via `#define` flags based on material features
- A `ShaderCache` deduplicates compiled shader variants

### `geometries/`

Parametric geometry generators. Each returns a `Geometry` object (vertex + index buffers).

`Plane`, `Box`, `Sphere`, `Cone`, `Cylinder`, `Capsule`, `Circle` — all generated in Z-up orientation.

### `animation/`

Skeletal animation system.

- `AnimationClip` — keyframe tracks (position, rotation, scale per bone)
- `AnimationAction` — playback state for one clip (time, speed, weight, loop mode)
- `AnimationMixer` — blends multiple actions, drives skeleton updates
- Crossfade: blend weights interpolated over a duration between two actions

### `loaders/`

Asset loading pipeline.

- `GLTFLoader` — parses glTF 2.0 JSON + binary, builds scene graph. Converts Y-up to Z-up on import.
- `DracoDecoder` — WASM-based Draco decompression (loaded lazily)
- `KTX2Loader` — KTX2 container parsing + Basis Universal WASM transcoder
- `TextureLoader` — loads images via `createImageBitmap`, creates HAL textures

### `raycasting/`

- `BVH` — SAH-based bounding volume hierarchy for triangle meshes. Built lazily on first query.
- `Raycaster` — scene-level raycasting: tests ray against node AABBs, then against mesh BVHs for intersected nodes

### `postfx/`

Post-processing pipeline.

- `BloomPass` — Unreal-style progressive downsample/upsample bloom
- `OITResolvePass` — composites weighted blended OIT buffers
- `ToneMappingPass` — ACES filmic tone mapping + final output

### `core/`

Top-level orchestration.

- `Engine` — initializes HAL device, manages render loop, resize handling
- `Renderer` — the main render pipeline: culling → shadow pass → opaque pass → transparent pass → post-processing

### `controls/`

- `OrbitControls` — pointer/touch-driven orbit, pan, zoom with damping

### `overlay/`

- `HtmlOverlay` — manages a DOM container over the canvas; projects 3D positions to 2D screen coordinates and positions HTML elements accordingly

## Data Flow Per Frame

```
1. Engine.tick(deltaTime)
   │
   ├─ 2. AnimationMixer.update(dt) for each mixer
   │     └─ Updates bone world matrices
   │
   ├─ 3. Scene graph world matrix update (dirty-flag propagation)
   │
   ├─ 4. Renderer.render(scene, camera)
   │     │
   │     ├─ 5. Extract frustum from camera VP matrix
   │     │
   │     ├─ 6. Traverse scene, frustum cull, collect render lists
   │     │     ├─ opaqueList (sorted: by pipeline ID, then front-to-back)
   │     │     └─ transparentList (unsorted — OIT handles order)
   │     │
   │     ├─ 7. Shadow pass: render opaqueList into CSM atlas
   │     │     └─ For each cascade: set light VP, render depth-only
   │     │
   │     ├─ 8. Opaque pass → MSAA render target
   │     │     └─ Bind pipeline, bind group, draw — for each item
   │     │
   │     ├─ 9. Transparent pass → OIT accumulation + revealage targets
   │     │     └─ Additive blend (accum), zero-blend (revealage)
   │     │
   │     ├─ 10. MSAA resolve (opaque target → resolve target)
   │     │
   │     ├─ 11. OIT composite: blend transparent over opaque
   │     │
   │     ├─ 12. Bloom pass
   │     │     ├─ Brightness threshold extract
   │     │     ├─ Progressive downsample (mip chain)
   │     │     ├─ Progressive upsample with tent filter
   │     │     └─ Additive composite onto scene
   │     │
   │     └─ 13. Tone mapping → screen
   │
   ├─ 14. HtmlOverlay.update(camera) — reproject DOM positions
   │
   └─ 15. OrbitControls.update(dt) — apply damping
```

## Key Design Decisions

### Why not ECS?

ECS (Entity Component System) is popular in engines, but adds complexity without benefit for our scope. A traditional scene graph with typed nodes is simpler, more debuggable, and sufficient for the feature set. The performance bottleneck is GPU draw calls, not JS-side entity iteration.

### Why uniform arrays for material indices?

The `_materialindex` vertex attribute indexes into a small uniform array of material properties. Alternatives considered:
- **1D texture lookup**: works but adds texture fetch latency in the shader and requires texture updates
- **Vertex color baked per material**: loses the ability to change colors at runtime
- **Separate draw calls per material index**: defeats the purpose of single-mesh models

A uniform array (max 32 entries) is fast to update, fast to read in shaders, and works identically in WebGL2 and WebGPU.

### Why Weighted Blended OIT over sorted transparency?

Sorted transparency (three.js approach) requires:
1. Sorting transparent objects back-to-front every frame (CPU cost)
2. Does not handle intersecting transparent geometry
3. Produces artifacts with overlapping transparent objects at similar depths

Weighted Blended OIT (McGuire & Bavoil 2013):
1. No sorting — order-independent
2. Handles intersecting geometry correctly
3. Single extra pass with two render targets
4. Works in WebGL2 (via `EXT_color_buffer_float`) and WebGPU natively
5. Only limitation: not physically accurate for strongly colored glass stacking, which is acceptable for game use

### Why Z-up right-handed?

Most game engines (Unreal, Source, many custom engines) use Z-up because:
- The ground plane is naturally X/Y, height is Z
- Map editors and level design tools align with this convention
- More intuitive for game developers

glTF uses Y-up, so we apply a 90° rotation around X at import time. This is a one-time cost at load.
