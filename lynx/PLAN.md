# Lynx — A Minimal High-Performance 3D Engine

## Vision

Lynx is a minimal, high-performance 3D game engine for the web. It targets the same developer ergonomics as three.js — simple scene graph, declarative materials, easy asset loading — but is architecturally designed for raw performance from the ground up. The goal is 2000+ draw calls at 60fps on recent mobile phones without instancing.

## Core Design Principles

1. **Zero-overhead abstraction**: The GPU abstraction layer adds no measurable cost over raw WebGPU/WebGL2 calls.
2. **Static by default, dynamic by opt-in**: Render bundles (WebGPU) and cached command sequences (WebGL2) make static scenes nearly free. Dynamic objects opt into per-frame updates.
3. **Flat data, deep trees**: The scene graph is a comfortable tree of nodes for authoring, but rendering operates on a flat, sorted command list rebuilt only when the graph changes.
4. **Allocate nothing per frame**: All per-frame data uses pre-allocated typed array pools. Zero GC pressure during steady-state rendering.
5. **Correct transparency without sorting**: Weighted Blended Order-Independent Transparency eliminates the sorting headaches of traditional alpha blending.

## Technical Decisions Summary

| Decision | Choice | Rationale |
|---|---|---|
| GPU abstraction | Thin layer modeled after WebGPU | WebGPU is the better API; WebGL2 backend translates |
| Draw call perf | Sort by pipeline→material→depth, render bundles (WebGPU), VAO caching (WebGL2) | Minimizes state changes, the #1 bottleneck |
| Uniform strategy | UBOs with dynamic offsets | One large UBO per frame, offset per object — fewer binds |
| Transparency | Weighted Blended OIT (McGuire & Bavoil 2013) | Order-independent, works in both backends, no sorting |
| Bloom | Unreal-style dual Kawase / progressive downscale-upscale chain | High quality, efficient, works with per-vertex emissive |
| Shadows | 3-cascade CSM with PCF 5×5, stable snapping | Smooth shadows across 200×200m, no shimmering |
| BVH | SAH-based, flat array layout, stackless traversal | Matches three-mesh-bvh perf, cache-friendly |
| Animation | Bone matrices via UBO (≤128 bones) or data texture | Efficient GPU upload, supports crossfade blending |
| Coordinate system | Z-up, right-handed | Matches requirement; GLTF Y-up converted on import |
| Textures | KTX2 + Basis Universal transcoder, runtime format selection | BC7/ASTC/ETC2 based on device capabilities |
| GLTF decoding | Draco WASM decoder in Web Worker | Non-blocking mesh decompression |
| MSAA | Hardware 4× MSAA, resolve before post-processing | Simple, efficient, well-supported |
| React bindings | Custom reconciler via react-reconciler | R3F-style declarative API without R3F dependency |
| HTML overlay | 3D→2D projection, CSS transform positioning | Lightweight, no extra render pass |
| Math | Custom minimal library, Z-up RH, column-major matrices | No external dependency, tuned for engine needs |

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    React Bindings                       │
│              <Canvas> <Mesh> <Group> ...                │
├─────────────────────────────────────────────────────────┤
│                    HTML Overlay                         │
│           3D→screen projection, DOM sync               │
├─────────────────────────────────────────────────────────┤
│                   Lynx Core API                         │
│    Scene · Camera · Mesh · Light · Controls · Loader    │
├──────────────┬──────────────────────────────────────────┤
│ Scene Graph  │           Render Pipeline                │
│ Node tree,   │  Command sort, frustum cull,             │
│ transforms,  │  shadow pass, opaque pass,               │
│ dirty flags  │  transparent pass (OIT),                 │
│              │  post-process (bloom, tone map, resolve)  │
├──────────────┴──────────────┬───────────────────────────┤
│    Animation System         │   Asset Pipeline          │
│  Skeleton, clips,           │  GLTF parser, Draco       │
│  mixer, crossfade           │  worker, KTX2/Basis       │
├─────────────────────────────┤  worker, texture mgr      │
│    Raycasting / BVH         │                           │
│  SAH build, flat array,     │                           │
│  stackless traversal        │                           │
├─────────────────────────────┴───────────────────────────┤
│              GPU Abstraction Layer (GAL)                 │
│        Device · Buffer · Texture · Pipeline             │
│        BindGroup · RenderPass · CommandEncoder          │
├──────────────────────┬──────────────────────────────────┤
│    WebGPU Backend    │       WebGL2 Backend             │
│  Native API calls    │  State-tracked translation       │
└──────────────────────┴──────────────────────────────────┘
```

## Module Map

| Module | File(s) | Description |
|---|---|---|
| GPU Abstraction | `src/gal/` | Backend-agnostic GPU interface |
| WebGPU Backend | `src/gal/webgpu/` | WebGPU implementation |
| WebGL2 Backend | `src/gal/webgl2/` | WebGL2 implementation |
| Scene Graph | `src/scene/` | Node, Mesh, Group, Camera, Light |
| Materials | `src/materials/` | BasicMaterial, LambertMaterial, shader gen |
| Geometry | `src/geometry/` | Parametric shapes + BufferGeometry |
| Renderer | `src/renderer/` | Render pipeline, command sorting, passes |
| Lighting | `src/lighting/` | Directional, ambient, CSM |
| Animation | `src/animation/` | Skeleton, AnimationClip, Mixer |
| Assets | `src/assets/` | GLTF loader, Draco worker, KTX2 worker |
| Raycasting | `src/raycasting/` | BVH build + query |
| Post-processing | `src/postprocess/` | Bloom, tone mapping, MSAA resolve |
| Transparency | `src/transparency/` | Weighted Blended OIT pass |
| Controls | `src/controls/` | OrbitControls |
| HTML Overlay | `src/overlay/` | HtmlOverlay manager |
| Math | `src/math/` | Vec3, Mat4, Quat, AABB, Frustum, Ray |
| React | `packages/lynx-react/` | React reconciler + hooks |

## Documentation Files

Each aspect of the architecture is described in detail in the following files:

- [ARCHITECTURE.md](./ARCHITECTURE.md) — Overall system architecture and data flow
- [GPU-ABSTRACTION.md](./GPU-ABSTRACTION.md) — WebGL2/WebGPU abstraction layer design
- [RENDERER.md](./RENDERER.md) — Render pipeline, draw call optimization, command sorting
- [SCENE-GRAPH.md](./SCENE-GRAPH.md) — Node hierarchy, transforms, frustum culling, groups
- [MATERIALS.md](./MATERIALS.md) — Material system, material index, vertex colors, shaders
- [GEOMETRY.md](./GEOMETRY.md) — Parametric geometries
- [LIGHTING-SHADOWS.md](./LIGHTING-SHADOWS.md) — Directional/ambient lights, cascading shadow maps
- [ANIMATION.md](./ANIMATION.md) — Skeletal animation, skinned meshes, crossfade, bone attachment
- [ASSETS.md](./ASSETS.md) — GLTF loading, Draco decoding, KTX2/Basis transcoding
- [TRANSPARENCY.md](./TRANSPARENCY.md) — Weighted Blended OIT system
- [BLOOM-POSTPROCESS.md](./BLOOM-POSTPROCESS.md) — Per-vertex bloom, post-processing pipeline, MSAA
- [RAYCASTING.md](./RAYCASTING.md) — BVH construction and ray queries
- [CONTROLS.md](./CONTROLS.md) — Orbit controls
- [HTML-OVERLAY.md](./HTML-OVERLAY.md) — DOM element overlay system
- [REACT.md](./REACT.md) — React bindings and reconciler
- [MATH.md](./MATH.md) — Math library (Z-up, right-handed)
- [PERFORMANCE.md](./PERFORMANCE.md) — Performance strategy and mobile optimization

## Code Style

- TypeScript, strict mode
- No semicolons
- `const` arrow functions preferred over `function` declarations
- Minimal comments — code should be self-documenting
- JSDoc on public API only
- No classes for data (use plain objects + typed arrays)
- Classes for stateful systems (Renderer, Scene, AnimationMixer)

## Package Structure

```
lynx/
├── packages/
│   ├── lynx/              # Core engine
│   │   ├── src/
│   │   │   ├── gal/       # GPU abstraction layer
│   │   │   ├── scene/     # Scene graph
│   │   │   ├── materials/ # Materials + shaders
│   │   │   ├── geometry/  # Parametric geometry
│   │   │   ├── renderer/  # Render pipeline
│   │   │   ├── lighting/  # Lights + shadows
│   │   │   ├── animation/ # Skeletal animation
│   │   │   ├── assets/    # GLTF, Draco, KTX2
│   │   │   ├── raycasting/# BVH + raycasting
│   │   │   ├── postprocess/# Bloom, tonemap
│   │   │   ├── transparency/# OIT
│   │   │   ├── controls/  # Orbit controls
│   │   │   ├── overlay/   # HTML overlay
│   │   │   └── math/      # Vectors, matrices, etc.
│   │   └── package.json
│   └── lynx-react/        # React bindings
│       ├── src/
│       └── package.json
└── examples/
    ├── basic/
    ├── animation/
    ├── shadows/
    └── react/
```
