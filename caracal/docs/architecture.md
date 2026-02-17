# Architecture

## High-Level Design

Caracal is organized as a set of loosely coupled modules that communicate through well-defined interfaces. The engine avoids deep inheritance hierarchies in favor of composition and flat data layouts optimized for cache performance.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Application Layer                        │
│  (User code, React bindings, OrbitControls, HTMLOverlay)        │
├─────────────────────────────────────────────────────────────────┤
│                        Engine (Orchestrator)                    │
│  Creates backend, manages render loop, exposes public API       │
├──────────┬──────────┬──────────┬──────────┬─────────────────────┤
│  Scene   │ Renderer │ Loaders  │Animation │  Post-Processing    │
│  Graph   │          │          │  System  │                     │
├──────────┴──────────┴──────────┴──────────┴─────────────────────┤
│                    Backend Abstraction (GPUDevice)               │
├─────────────────────────────┬───────────────────────────────────┤
│       WebGL2Backend         │         WebGPUBackend             │
└─────────────────────────────┴───────────────────────────────────┘
```

## Module Dependency Graph

```
math ◄─── core ◄─── renderer ◄─── postprocess
  ▲         ▲          ▲              ▲
  │         │          │              │
  │     materials   loaders      bloom/oit
  │         ▲          ▲
  │         │          │
  │      geometry   animation
  │         ▲
  │         │
  └──── bvh/raycaster
```

Arrows point from dependent → dependency. `math` has zero dependencies. `core` depends only on `math`. Everything flows upward.

## Core Abstractions

### Engine

The top-level entry point. Creates the GPU backend, canvas, and render loop. Manages the frame lifecycle.

```typescript
// Pseudocode — final API in api-reference.md
const engine = createEngine({
  canvas: document.getElementById('game'),
  backend: 'auto', // 'webgpu' | 'webgl2' | 'auto'
  antialias: true,  // 4x MSAA
  pixelRatio: window.devicePixelRatio,
})
```

The engine auto-detects WebGPU support and falls back to WebGL 2. It owns:
- The `GPUDevice` (backend abstraction)
- The `Renderer` (draw call submission)
- The `PostProcessor` (bloom, OIT composite, tone mapping)
- The frame loop (`requestAnimationFrame` with delta tracking)

### Scene

A scene is a flat container that holds the root of the scene graph and maintains derived render lists. It does **not** own the camera — cameras are nodes in the graph that the renderer references.

### Node

The base unit of the scene graph. Every entity (mesh, light, camera, group, bone) is a `Node`. Nodes hold:
- Local transform (position, rotation, scale)
- World matrix (computed from parent chain)
- AABB (for culling)
- Optional components (geometry, material, skeleton, light data)

### GPUDevice

The backend abstraction interface. See [renderer.md](renderer.md) for full detail.

## Data Flow Per Frame

```
1. engine.tick(deltaTime)
   │
   ├─ 2. animationMixer.update(dt)
   │     └─ Update bone matrices for all active skeletons
   │
   ├─ 3. scene.updateWorldMatrices()
   │     └─ Walk dirty nodes, recompute world matrices
   │     └─ Update world AABBs
   │
   ├─ 4. renderer.render(scene, camera)
   │     │
   │     ├─ 4a. Frustum cull → build render list
   │     │
   │     ├─ 4b. Sort render list (opaque: state sort, transparent: none needed for OIT)
   │     │
   │     ├─ 4c. Shadow pass: render CSM cascades
   │     │
   │     ├─ 4d. Opaque pass: draw all opaque meshes
   │     │
   │     ├─ 4e. Transparent pass (OIT): draw all transparent meshes → accum + revealage
   │     │
   │     └─ 4f. OIT composite pass
   │
   └─ 5. postProcessor.apply()
         ├─ Bloom extraction + blur + composite
         └─ Tone mapping → final output
```

## Directory Structure

```
src/
├── core/
│   ├── Engine.ts            # Engine creation, frame loop
│   ├── Scene.ts             # Scene container, render list
│   ├── Node.ts              # Base node class
│   ├── Group.ts             # Grouping node (no geometry/material)
│   ├── Mesh.ts              # Node with geometry + material
│   ├── Camera.ts            # Perspective/orthographic camera
│   └── types.ts             # Shared type definitions
│
├── math/
│   ├── Vec3.ts              # 3D vector (Z-up aware)
│   ├── Vec4.ts              # 4D vector / homogeneous coords
│   ├── Mat4.ts              # 4×4 matrix (column-major)
│   ├── Quat.ts              # Quaternion
│   ├── Euler.ts             # Euler angles (Z-up: ZYX order)
│   ├── AABB.ts              # Axis-aligned bounding box
│   ├── Frustum.ts           # View frustum (6 planes)
│   ├── Ray.ts               # Ray for raycasting
│   ├── Sphere.ts            # Bounding sphere
│   └── common.ts            # clamp, lerp, DEG2RAD, etc.
│
├── backend/
│   ├── GPUDevice.ts         # Abstract interface
│   ├── GPUTypes.ts          # Shared enums/types
│   ├── WebGL2Backend.ts     # WebGL 2 implementation
│   └── WebGPUBackend.ts     # WebGPU implementation
│
├── renderer/
│   ├── Renderer.ts          # Main render orchestrator
│   ├── RenderList.ts        # Culled + sorted draw call list
│   ├── StateSort.ts         # Sort key generation + sorting
│   ├── ShadowPass.ts        # CSM rendering
│   ├── OpaquePass.ts        # Opaque draw submission
│   ├── TransparentPass.ts   # OIT accumulation pass
│   └── OITComposite.ts      # OIT resolve/composite pass
│
├── materials/
│   ├── Material.ts          # Base material
│   ├── BasicMaterial.ts     # Unlit material
│   ├── LambertMaterial.ts   # Diffuse-lit material
│   ├── ShaderLib.ts         # Shader source generation (GLSL + WGSL)
│   └── MaterialIndex.ts     # Per-index color/emissive mapping
│
├── geometry/
│   ├── Geometry.ts          # Geometry container
│   ├── BufferAttribute.ts   # Typed array attribute wrapper
│   ├── PlaneGeometry.ts     # Parametric plane
│   ├── BoxGeometry.ts       # Parametric box
│   ├── SphereGeometry.ts    # Parametric UV sphere
│   ├── ConeGeometry.ts      # Parametric cone
│   ├── CylinderGeometry.ts  # Parametric cylinder
│   ├── CapsuleGeometry.ts   # Parametric capsule
│   ├── CircleGeometry.ts    # Parametric circle/disc
│   └── generators.ts        # Shared generation utilities
│
├── animation/
│   ├── Skeleton.ts          # Skeleton definition
│   ├── Bone.ts              # Bone node
│   ├── AnimationClip.ts     # Keyframe animation data
│   ├── AnimationMixer.ts    # Playback + crossfade controller
│   └── BoneAttachment.ts    # Attach static meshes to bones
│
├── bvh/
│   ├── BVH.ts               # SAH-BVH construction
│   ├── BVHTraversal.ts      # Flat stackless ray traversal
│   └── Raycaster.ts         # High-level raycasting API
│
├── loaders/
│   ├── GLTFLoader.ts        # glTF 2.0 parser + scene builder
│   ├── DracoDecoder.ts      # Web Worker Draco WASM decoder
│   ├── KTX2Loader.ts        # KTX2 container parser
│   └── BasisTranscoder.ts   # Basis Universal WASM transcoder
│
├── controls/
│   └── OrbitControls.ts     # Orbit camera controls
│
├── overlay/
│   └── HTMLOverlay.ts       # DOM overlay at 3D positions
│
├── postprocess/
│   ├── PostProcessor.ts     # Post-processing pipeline manager
│   ├── BloomPass.ts         # Selective Unreal-style bloom
│   └── ToneMapPass.ts       # ACES tone mapping
│
└── index.ts                 # Public API barrel export

react/
├── reconciler.ts            # React reconciler (react-reconciler)
├── Canvas.ts                # <Canvas> root component
├── components.ts            # <mesh>, <group>, <light>, etc.
├── hooks.ts                 # useFrame, useEngine, useLoader, etc.
└── index.ts                 # React API barrel export
```

## Design Principles

### 1. No Allocations in the Hot Path

Every `Vec3`, `Mat4`, `Quat` used during rendering is pre-allocated in a pool or on the module. The render loop never calls `new`. Temporary math results use a thread-local scratch pool:

```typescript
// Internal scratch vectors — never exposed, never GC'd
const _v0 = new Vec3()
const _v1 = new Vec3()
const _m0 = new Mat4()
```

### 2. Data-Oriented Where It Matters

While the public API is object-oriented (nodes, materials, meshes), internal hot-path data is stored in flat typed arrays:

- **World matrices**: `Float32Array(nodeCount * 16)` — contiguous, cache-friendly
- **AABB bounds**: `Float32Array(nodeCount * 6)` — min/max packed
- **Sort keys**: `Uint32Array(drawCallCount)` — single array radix sort

### 3. Minimal Abstraction Over GPU APIs

The backend abstraction is intentionally thin — it wraps resource creation and command encoding but does **not** attempt to unify the command model. The renderer has small, clearly marked branches for backend-specific optimizations:

```typescript
if (device.backend === 'webgpu') {
  // Use pre-recorded render bundle for static meshes
  pass.executeBundles([staticBundle])
} else {
  // WebGL: use WEBGL_multi_draw if available
  submitDrawCalls(renderList)
}
```

### 4. Components Over Inheritance

Nodes are composed from data, not deep class trees:

```
Node
├── .transform     → TransformData (position, rotation, scale, worldMatrix)
├── .geometry?     → Geometry (buffer attributes, index buffer)
├── .material?     → Material (shader, uniforms, textures)
├── .skeleton?     → Skeleton (bones, inverse bind matrices)
├── .lightData?    → LightData (type, color, intensity, shadow config)
└── .children      → Node[]
```

A `Mesh` is just a `Node` that always has `.geometry` and `.material`. A `Light` is a `Node` with `.lightData`. This keeps the base `Node` class small and avoids empty fields.

### 5. Explicit Over Magic

Unlike Three.js, which auto-detects many things at render time (needsUpdate flags, auto-uploading textures, auto-compiling shaders), Caracal is explicit:

- Shaders compile at material creation time, not first render
- Textures upload when you call `texture.upload(device)`, not when first referenced
- World matrices update in a single batch pass, not lazily per-access

This eliminates hidden frame spikes and makes performance predictable.
