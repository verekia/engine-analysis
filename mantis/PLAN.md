# Mantis — A Minimal High-Performance 3D Engine

## Philosophy

Mantis is a 3D game engine for the web that pairs the developer ergonomics of
three.js with the raw throughput of a native engine. It achieves this not through
micro-optimizations bolted onto a legacy architecture, but through ground-up
design decisions that eliminate overhead at the structural level.

**Core tenets:**

1. **Performance by design** — SoA data layouts, zero-allocation render loops,
   command-buffer rendering, and a uniform ring buffer make 2000+ draw calls at
   60 fps on mobile phones a baseline, not a stretch goal.
2. **Minimal surface area** — Two material types (Basic, Lambert), two light
   types (ambient, directional), one shadow technique (CSM). Enough for
   beautiful low-poly and stylized games without the complexity tax of PBR.
3. **Transparency that works** — Weighted Blended Order-Independent Transparency
   eliminates sorting headaches entirely.
4. **Z-up, right-handed** — Consistent with Blender and Unreal. All systems,
   from math to controls to GLTF import, operate in Z-up natively.
5. **React-first bindings** — A React Three Fiber-style reconciler as a
   first-class citizen, not an afterthought.

---

## Technical Summary

| Area | Decision |
|---|---|
| **GPU backends** | WebGPU primary, WebGL2 fallback — thin abstraction modeled on WebGPU concepts |
| **Coordinate system** | Z-up, right-handed (X right, Y forward, Z up) |
| **Materials** | BasicMaterial (unlit), LambertMaterial (N·L + ambient + AO + shadows) |
| **Vertex coloring** | Per-vertex `_materialIndex` attribute → palette of colors + emissive per index |
| **Textures** | Color map, AO map, KTX2 via Basis Universal WASM transcoder |
| **GLTF** | glTF 2.0 with Draco (WASM worker), Y→Z-up baked at load time |
| **Anti-aliasing** | 4× MSAA resolved before post-processing |
| **Transparency** | Weighted Blended OIT (McGuire & Bavoil 2013), single-pass, no sorting |
| **Shadows** | 3-cascade CSM, single 3072×1024 atlas, PCF 3×3 Poisson, texel snapping |
| **Bloom** | Unreal-style: threshold → 5-level downsample (Karis) → 5-level tent upsample → composite |
| **Raycasting** | SAH BVH (12-bin), flat array nodes, scene-level + mesh-level |
| **Animation** | GPU skinning (≤4 weights), keyframe tracks, weighted crossfade blending |
| **Scene graph** | SoA transforms, dirty-flag propagation, breadth-first world matrix update |
| **Frustum culling** | World-space AABB vs 6 frustum planes, BVH-accelerated for large scenes |
| **Performance model** | Command buffer pattern, uniform ring buffer, radix-sorted draw keys, render bundles |
| **Geometries** | Plane, Box, Sphere, Cone, Cylinder, Capsule, Circle — all parametric |
| **Controls** | Orbit controls with inertia, Z-up aware (azimuth around Z, elevation toward Z) |
| **HTML overlay** | Project 3D → screen, CSS `transform` positioning, optional depth occlusion |
| **React** | Custom `react-reconciler`, `<Canvas>`, declarative scene, `useFrame`, Suspense loaders |
| **Math** | Zero-alloc Float32Array-backed Vec3/Vec4/Mat4/Quat with scratch pool |
| **Code style** | TypeScript, no semicolons, `const` arrow functions, minimal and documented |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    React Bindings                        │
│  <Canvas> · <Mesh> · <Group> · useFrame · useLoader     │
├─────────────────────────────────────────────────────────┤
│                    HTML Overlay                          │
│  Project 3D → 2D · CSS transform · Occlusion            │
├────────────────┬────────────────────────────────────────┤
│  Orbit Controls│         Asset Pipeline                  │
│  Damped inertia│  GLTF → Draco (worker) → BVH build     │
│  Z-up polar    │  KTX2 → Basis (worker) → GPU texture   │
├────────────────┴────────────────────────────────────────┤
│                Post-Processing Stack                     │
│  OIT Composite → Bloom (downsample/upsample) → Tonemap  │
├─────────────────────────────────────────────────────────┤
│           Renderer (Command Buffer + Sort)               │
│  Frustum cull → Sort draw keys → Build commands →        │
│  Upload ring buffer → Execute passes                     │
├──────────────────────────┬──────────────────────────────┤
│     Scene Graph          │      Animation System         │
│  SoA transforms          │  Skeletal (GPU skinning)      │
│  Dirty propagation       │  Crossfade blending           │
│  Parent/child + Groups   │  Bone attachment              │
├──────────────────────────┼──────────────────────────────┤
│     Material System      │       Shadow System           │
│  Basic + Lambert         │  3-cascade CSM                │
│  Material index palette  │  PCF Poisson filtering        │
│  Shader variant cache    │  Texel snap + cascade blend   │
├──────────────────────────┴──────────────────────────────┤
│              GPU Abstraction Layer (GAL)                  │
│  Pipeline · Buffer · Texture · BindGroup · RenderPass    │
├──────────────────────────┬──────────────────────────────┤
│    WebGPU Backend        │     WebGL2 Backend            │
│  Native pipelines        │  Program + VAO + state cache  │
│  Render bundles          │  UBO emulation                │
│  Dynamic offsets         │  bindBufferRange              │
└──────────────────────────┴──────────────────────────────┘
│                        Math                              │
│  Vec3 · Vec4 · Mat4 · Quat · AABB · Frustum · Pool      │
└─────────────────────────────────────────────────────────┘
```

---

## Document Index

| File | Contents |
|---|---|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Module structure, dependency graph, engine lifecycle |
| [RENDERER.md](./RENDERER.md) | GPU abstraction layer, WebGPU/WebGL2 backends, frame pipeline |
| [PERFORMANCE.md](./PERFORMANCE.md) | Performance strategy, budgets, command buffer, ring buffer, sorting |
| [MATERIALS.md](./MATERIALS.md) | Material system, material index palette, shader variants |
| [SCENE-GRAPH.md](./SCENE-GRAPH.md) | SoA transforms, hierarchy, groups, frustum culling |
| [ASSETS.md](./ASSETS.md) | GLTF loading, Draco decoding, KTX2/Basis transcoding |
| [ANIMATION.md](./ANIMATION.md) | Skeletal animation, crossfades, bone attachment |
| [SHADOWS.md](./SHADOWS.md) | Cascaded shadow maps, PCF filtering, atlas layout |
| [TRANSPARENCY.md](./TRANSPARENCY.md) | Weighted Blended OIT implementation |
| [RAYCASTING.md](./RAYCASTING.md) | BVH construction and traversal, raycasting API |
| [BLOOM.md](./BLOOM.md) | Unreal-style bloom pipeline, per-vertex emissive control |
| [GEOMETRY.md](./GEOMETRY.md) | Parametric geometry generators |
| [REACT.md](./REACT.md) | React reconciler, components, hooks |
| [HTML-OVERLAY.md](./HTML-OVERLAY.md) | DOM element projection and positioning |
| [MATH.md](./MATH.md) | Math library: vectors, matrices, quaternions, pools |
| [CONTROLS.md](./CONTROLS.md) | Orbit controls with Z-up polar coordinates |
| [API.md](./API.md) | Complete public API surface with usage examples |

---

## Performance Targets

| Metric | Target | Strategy |
|---|---|---|
| Draw calls at 60 fps (mobile) | 2000+ | Command buffer, ring buffer uniforms, radix sort, minimal state changes |
| JS overhead per draw call | < 5 µs | Pre-baked pipelines, flat command arrays, no allocation |
| Scene graph update (2000 nodes, 50 dirty) | < 0.3 ms | SoA layout, dirty flags, skip clean subtrees |
| Frustum cull (2000 objects) | < 0.15 ms | BVH-accelerated plane tests |
| Shadow map render (3 cascades) | < 3 ms GPU | Depth-only pass, front-to-back sort, early-Z |
| Bloom post-process | < 1.5 ms GPU | Half-res chain, 5 levels, optimized filters |
| BVH raycast (100K triangles) | < 0.1 ms | SAH BVH, flat array, stack traversal |
| GLTF load (1MB Draco) | < 200 ms | Worker-based, zero-copy transferable |

---

## Why Not Three.js?

Three.js is a remarkable library, but its architecture makes high draw-call
counts fundamentally expensive:

- **Per-draw uniform diffing** — Three.js checks every material property every
  frame to decide what to upload. Mantis pre-compiles materials into immutable
  pipeline states and uses a ring buffer for all dynamic uniforms.
- **Object allocation in render loop** — Three.js creates temporary vectors and
  matrices during rendering. Mantis uses a scratch pool reset each frame.
- **No draw call sorting** — Three.js renders objects in scene graph order.
  Mantis sorts by a 64-bit key (pipeline → material → texture → depth) via
  radix sort, minimizing GPU state changes.
- **Transparency sorting** — Three.js sorts transparent objects by centroid
  distance, which fails for intersecting or large objects. Mantis uses WBOIT
  which requires no sorting at all.
- **No UBOs** — Three.js uploads uniforms individually. Mantis batches all
  per-frame and per-object uniforms into UBOs with dynamic offsets.
