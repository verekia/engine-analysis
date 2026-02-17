# PLAN.md - Architecture Decisions

## Design Philosophy

**Performance is architecture, not optimization.** The engine should be fast by default through structural decisions, not through after-the-fact tweaks. Every design choice traces back to the target: **2000 draw calls at 60fps on recent mobile phones**.

## Key Decisions

### Monolithic Single Package

**Decision: Single package with internal modules** (Hyena, Rabbit, Shark, Wren approach)

Multi-package monorepos (Fennec, Caracal, Lynx, Mantis) add dependency management complexity that is not justified for an engine this focused. A single package with clear internal module boundaries provides:

- Simpler distribution and versioning
- Easier onboarding (one install, one import)
- Internal tree-shaking still works with modern bundlers via side-effect-free modules
- No cross-package version drift

React bindings ship as a separate package (`engine-react`) since they add `react-reconciler` as a peer dependency and are optional.

### Object Graph with SoA Internals

**Decision: OOP public API with flat typed arrays internally** (Caracal, Fennec, Hyena, Lynx, Rabbit, Wren approach)

Pure SoA (Bonobo, Mantis, Shark) is maximal for cache performance but hostile to users. An object-oriented public API (nodes, meshes, materials as objects) backed by contiguous Float32Arrays for the renderer gives both:

- Familiar, debuggable API for developers
- Cache-friendly iteration and GPU-uploadable buffers for the renderer
- Dirty flags on objects translate to SoA updates only when needed

### Full Feature Set from v1

**Decision: Ship all required features in v1** (8/9 implementations agree)

Bonobo's approach of deferring shadows, animation, glTF, and scene hierarchy to v2 leaves too many gaps. The prompt requirements are clear: CSM shadows, skeletal animation, glTF+Draco+KTX2, scene hierarchy, React bindings, and HTML overlay are all v1 requirements. Shipping without them is not an option.

### WebGPU-First with WebGL2 Fallback

**Decision: Model the GPU abstraction after WebGPU** (universal agreement)

WebGPU is the better API. The abstraction layer should follow WebGPU concepts (immutable pipelines, bind groups, explicit command recording). WebGL2 is the translation layer, not the design target.

### TypeScript, No Semicolons, Arrow Functions

**Decision: Follow the prompt's style requirements exactly** (universal agreement)

- TypeScript strict mode throughout
- No semicolons
- `const` arrow functions preferred over `function` blocks
- Minimal, well-documented code

### Z-Up, Right-Handed Coordinate System

**Decision: Z-up right-handed** (universal agreement)

- X = right, Y = forward, Z = up
- Matches Blender convention
- glTF Y-up assets converted at import time

## Package Structure

```
engine/
  src/
    math/           # Vec3, Mat4, Quat, AABB, Frustum, Ray
    device/         # GPU abstraction (Device, Buffer, Texture, Pipeline, BindGroup)
    renderer/       # Render loop, draw call sorting, state management
    scene/          # Scene graph, nodes, transforms, dirty flags
    materials/      # BasicMaterial, LambertMaterial, shader variants
    geometry/       # Parametric primitives, buffer layouts
    animation/      # Skeleton, AnimationMixer, crossfade, bone attachment
    loaders/        # glTF, Draco, KTX2/Basis
    lighting/       # Directional light, ambient light, CSM shadows
    spatial/        # BVH, raycasting, frustum culling
    postfx/         # Bloom, MSAA, OIT composite, tone mapping
    controls/       # OrbitControls
    overlay/        # HTML overlay system
  package.json

engine-react/
  src/
    reconciler.ts   # Custom react-reconciler host config
    Canvas.tsx      # Root component
    hooks.ts        # useFrame, useEngine, useLoader, useAnimations
    Html.tsx         # HTML overlay React component
    components.ts   # JSX element type definitions
  package.json      # peer depends on engine + react
```

## Core Technical Stack

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Language | TypeScript | Full type safety, prompt requirement |
| GPU primary | WebGPU | Lower overhead, better API |
| GPU fallback | WebGL2 | Compatibility |
| Coordinate system | Z-up RH | Prompt requirement, Blender compat |
| Materials | Basic + Lambert | Sufficient for stylized/low-poly |
| Transparency | WBOIT | No sorting, no headaches |
| Shadows | 3-cascade CSM | Smooth shadows for 200m worlds |
| Bloom | Unreal-style progressive | Per-vertex emissive control |
| BVH | SAH flat array | three-mesh-bvh quality target |
| Animation | GPU skinning + crossfade | glTF skeletal animation |
| Assets | glTF 2.0 + Draco + KTX2 | Standard pipeline |
| React | Custom reconciler | R3F-style declarative scene |
| Math | Custom, Float32Array | Zero-alloc, Z-up baked in |
| Sort | Radix sort | O(n) draw call ordering |
| Uniforms | Dynamic offsets | Standard, both backends |
