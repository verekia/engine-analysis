# Caracal — A Minimal, High-Performance 3D Engine

Caracal is a minimal 3D game engine for the web that prioritizes raw performance and developer ergonomics. It targets stylized, low-poly 3D games and can sustain **2000+ draw calls at 60 fps on modern phones** without instancing.

## Vision

Three.js proved that a friendly API makes 3D accessible on the web. But its design predates modern GPU APIs and carries significant overhead per draw call. Caracal takes the lessons learned from Three.js's developer experience and rebuilds on a foundation designed for performance from day one — supporting both WebGL 2 and WebGPU with a thin, zero-overhead abstraction layer.

## Core Requirements

| Requirement | Approach |
|---|---|
| WebGL 2 + WebGPU | Thin backend abstraction; shared render graph |
| Materials | Basic (unlit) and Lambert (diffuse) |
| glTF + Draco | Streaming loader with Web Worker Draco decoder |
| Textures | Color maps, AO maps, KTX2/Basis Universal transcoding |
| Per-vertex bloom | `_materialindex` attribute → per-index color/emissive lookup; selective Unreal-style bloom |
| MSAA | 4× MSAA via renderbuffer (WebGL) / multisample texture (WebGPU) |
| Raycasting + BVH | SAH-built flat BVH; Möller–Trumbore intersection |
| Skeletal animation | GPU skinning, animation mixer with linear crossfade |
| Frustum culling | AABB-based, updated from world-matrix dirty flags |
| Transparency | Weighted Blended OIT (single-pass, no sorting) |
| 2000 draw calls @ 60 fps | State sorting, UBOs, render bundles (WebGPU), multi-draw (WebGL) |
| Orbit controls | Spherical coordinates, touch + pointer, damping |
| Parametric geometry | Plane, box, sphere, cone, cylinder, capsule, circle |
| Lighting | Directional + ambient |
| Shadows | 3-cascade CSM with PCF filtering |
| Coordinate system | Z-up, right-handed (X right, Y forward, Z up) |
| Scene graph | Parent/child hierarchy, groups, bone attachment |
| HTML overlay | World-to-screen projection, DOM overlay container |
| React bindings | Custom reconciler (R3F-style declarative API) |
| Code style | TypeScript, no semicolons, `const` arrow functions |

## Key Technical Decisions

### 1. Backend Abstraction — "GPU Device" Layer

Rather than abstracting at the draw-call level (which adds overhead), Caracal abstracts at the **resource level**: buffers, textures, shaders, render passes. Each backend (`WebGL2Backend`, `WebGPUBackend`) implements a `GPUDevice` interface. The render loop issues commands through this interface, and each backend translates them to native API calls with near-zero overhead.

**Rationale:** Abstracting at the resource level keeps the hot path thin. The renderer can still use backend-specific optimizations (render bundles for WebGPU, multi-draw for WebGL 2) because the render loop has explicit backend branches only where it matters.

### 2. Transparency — Weighted Blended OIT

Three.js's transparency problems stem from relying on painter's algorithm with depth sorting, which breaks for intersecting or complex geometry. Caracal uses **Weighted Blended Order-Independent Transparency (WBOIT)** (McGuire & Bavoil 2013):

- Single render pass for all transparent objects (no sorting)
- Two render targets: accumulation (RGBA16F) and revealage (R8)
- One full-screen composite pass
- Works correctly for overlapping/intersecting transparent geometry

**Rationale:** WBOIT is the best balance of visual quality, correctness, and performance for game use cases. It eliminates all sorting artifacts with minimal GPU cost.

### 3. BVH — Flat SAH-BVH with Stackless Traversal

For raycasting at the quality level of three-mesh-bvh:

- **SAH (Surface Area Heuristic)** construction for optimal tree quality
- **Flat array layout** (32 bytes per node) for cache-friendly traversal
- **Stackless traversal** using skip pointers for minimal memory allocation
- Supports both triangle-level and object-level BVH

**Rationale:** SAH produces the best ray-traversal performance. Flat arrays avoid pointer chasing and GC pressure. Stackless traversal removes the need for a traversal stack allocation.

### 4. Cascading Shadow Maps — 3 Cascades, Stable Projection

For a 200×200 m low-poly world:

- 3 cascades with practical split scheme (λ=0.5 blend of logarithmic and uniform)
- Texel-snapped projection to eliminate shadow edge swimming
- 2048×2048 shadow atlas (3 × 1024 regions or single 2048 with viewports)
- 5×5 PCF kernel with rotated Poisson disk for smooth penumbras

### 5. Performance Architecture

The core performance strategy:

1. **State sorting**: Draw calls sorted by shader → material → front-to-back distance
2. **UBOs/Bind groups**: Camera, lights, shadow data uploaded once per frame
3. **Zero GC in render loop**: All temp math objects pre-allocated; no `new` in hot path
4. **WebGPU render bundles**: Pre-encoded draw commands for static geometry
5. **`WEBGL_multi_draw`**: Batch multiple draw calls into single driver call
6. **Flat world matrix array**: Contiguous Float32Array for all transforms
7. **Frustum culling**: AABB test before any draw call submission

### 6. Coordinate System — Z-Up, Right-Handed

```
    Z (up)
    |
    |
    +------ X (right)
   /
  Y (forward)
```

All math (matrices, quaternions, Euler angles) operates in this convention. glTF import automatically converts from Y-up.

## Project Structure

```
caracal/
├── src/
│   ├── core/            # Engine, Scene, Node, Group, Camera
│   ├── math/            # Vec3, Mat4, Quat, AABB, Frustum, Ray
│   ├── backend/         # GPUDevice interface, WebGL2Backend, WebGPUBackend
│   ├── renderer/        # RenderLoop, RenderList, StateSort, ShadowPass, OITPass
│   ├── materials/       # Material, BasicMaterial, LambertMaterial, ShaderLib
│   ├── geometry/        # Geometry, BufferAttribute, parametric generators
│   ├── animation/       # Skeleton, Bone, AnimationClip, AnimationMixer
│   ├── bvh/             # BVH construction, traversal, Raycaster
│   ├── loaders/         # GLTFLoader, DracoDecoder, KTX2Loader, BasisTranscoder
│   ├── controls/        # OrbitControls
│   ├── overlay/         # HTMLOverlay, world-to-screen projection
│   └── postprocess/     # BloomPass, CompositePass, ToneMapPass
├── react/               # React reconciler, components, hooks
├── docs/                # Architecture documentation
└── examples/            # Demo scenes
```

## Dependencies

| Dependency | Purpose | Size |
|---|---|---|
| `draco3d` | Draco mesh decompression (WASM) | ~300 KB |
| `basis_universal` | Basis/KTX2 texture transcoding (WASM) | ~200 KB |
| `react-reconciler` | React bindings (peer dep) | 0 (peer) |

No other runtime dependencies. All math, BVH, rendering, animation — built from scratch.

## Documentation Index

| Document | Description |
|---|---|
| [Architecture](docs/architecture.md) | High-level module design, data flow, directory structure |
| [Renderer](docs/renderer.md) | WebGL/WebGPU abstraction, render pipeline, state management |
| [Scene Graph](docs/scene-graph.md) | Nodes, groups, transforms, dirty propagation, Z-up math |
| [Materials](docs/materials.md) | Material system, shader generation, material index system |
| [Geometry](docs/geometry.md) | Buffer layout, parametric generators, custom attributes |
| [Animation](docs/animation.md) | Skeletal animation, skinning, crossfade, bone attachment |
| [Lighting & Shadows](docs/lighting-shadows.md) | Directional/ambient lights, cascading shadow maps |
| [Post-Processing](docs/post-processing.md) | Selective bloom, tone mapping, MSAA |
| [Transparency](docs/transparency.md) | Weighted Blended OIT implementation |
| [Raycasting & BVH](docs/raycasting-bvh.md) | SAH-BVH construction, flat traversal, raycasting API |
| [Asset Loading](docs/asset-loading.md) | glTF, Draco, KTX2/Basis pipeline |
| [HTML Overlay](docs/html-overlay.md) | DOM elements at 3D positions |
| [React Bindings](docs/react-bindings.md) | Reconciler, components, hooks |
| [Performance](docs/performance.md) | Optimization strategies, benchmarks, mobile targets |
| [API Reference](docs/api-reference.md) | Public API surface and usage examples |
