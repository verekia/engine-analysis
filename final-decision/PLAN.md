# PLAN — Final Decision

## Design Philosophy

**Performance is architecture, not optimization.** Every structural decision traces back to the hard target: **2000 draw calls at 60fps on recent mobile phones**.

## Key Decisions

### Full-Featured v1

**Decision: Ship all required features in v1** (8/9 implementations agree)

Bonobo's approach of deferring shadows, animation, glTF, and scene hierarchy to v2 is rejected. The requirements are clear: CSM shadows, skeletal animation, glTF+Draco+KTX2, scene hierarchy, React bindings, and HTML overlay are all v1 requirements.

### Monolithic Single Package

**Decision: Single package with internal modules** (5/9: Hyena, Rabbit, Shark, Wren, Bonobo)

Multi-package monorepos (Fennec, Caracal, Lynx, Mantis) add dependency management complexity that is not justified for an engine this focused — two materials, one light type, seven primitives. A single package with clear internal module boundaries provides:

- Simpler distribution and versioning (one `npm install`, one version number)
- Easier onboarding (one import path)
- No cross-package version drift
- Internal tree-shaking still works with modern bundlers

React bindings ship as a **separate package** (`engine-react`) since they add `react` and `react-reconciler` as peer dependencies and are optional.

### Object Graph with SoA Internals

**Decision: OOP public API with flat typed arrays internally** (6/9: Caracal, Fennec, Hyena, Lynx, Rabbit, Wren)

Pure SoA (Bonobo, Mantis, Shark) is maximally cache-friendly but hostile to users. An object-oriented public API (nodes, meshes, materials as objects with properties) backed by contiguous Float32Arrays for the renderer gives both:

- Familiar, debuggable API for developers coming from Three.js
- Cache-friendly iteration and GPU-uploadable buffers for the renderer
- Dirty flags on objects translate to SoA updates only when needed

### No Automatic Instancing

**Decision: Skip instancing — optimize individual draw calls instead** (8/9 agree)

The prompt explicitly states "no instancing." When per-draw JS overhead is optimized to <2-3μs, 2000 unique draws is feasible without instancing. Instancing adds complexity (instance buffers, batching logic, material limits) without meaningful benefit at this scale. Games with thousands of identical objects can add manual instancing later if needed.

### WebGPU-First with WebGL2 Fallback

**Decision: Model the GPU abstraction after WebGPU** (universal agreement)

WebGPU is the better API. The abstraction layer follows WebGPU concepts (immutable pipelines, bind groups, explicit command recording). WebGL2 is the translation layer, not the design target.

### Z-Up, Right-Handed Coordinate System

**Decision: Z-up right-handed** (universal agreement)

- X = right, Y = forward, Z = up
- Matches Blender convention
- glTF Y-up assets converted at import time

### Code Style

**Decision: TypeScript, no semicolons, const arrow functions** (universal agreement)

- TypeScript strict mode throughout
- No semicolons
- `const` arrow functions preferred over `function` declarations

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

## v1 Feature Scope

All of the following are in-scope for v1:

- WebGL2 and WebGPU backends
- Basic and Lambert materials
- Material index system with per-vertex color/emissive (32-entry palette)
- Vertex colors
- glTF 2.0 with Draco mesh compression
- KTX2/Basis Universal texture compression
- Color textures, AO textures
- Per-vertex bloom (Unreal-style)
- 4x MSAA
- SAH-based BVH raycasting (three-mesh-bvh quality)
- GPU skinned meshes with skeletal animation and crossfade
- Frustum culling
- Weighted Blended OIT transparency
- Orbit controls
- Parametric geometry primitives (plane, box, sphere, cone, cylinder, capsule, circle)
- Directional light + ambient light
- 3-cascade CSM for 200×200m low-poly worlds
- Parent/child scene graph with Groups
- Bone attachment for static meshes
- HTML overlay system
- React bindings (R3F-style, separate package)
- ACES filmic tone mapping

## Out of Scope for v1

- Point lights, spot lights
- PBR materials (Lambert is sufficient for stylized/low-poly)
- Morph targets / blend shapes
- Physics integration
- Text rendering
- Particle systems
- Post-processing beyond bloom + tone mapping
- PCSS / VSM / ESM shadow techniques
- Multiple shadow-casting lights

## Core Technical Stack

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Language | TypeScript strict | Full type safety, prompt requirement |
| GPU primary | WebGPU | Lower overhead, better API |
| GPU fallback | WebGL2 | Compatibility |
| Coordinate system | Z-up RH | Prompt requirement, Blender compat |
| Materials | Basic + Lambert | Sufficient for stylized/low-poly |
| Transparency | WBOIT | No sorting, single-pass |
| Shadows | 3-cascade CSM | Smooth shadows for 200m worlds |
| Bloom | Unreal-style progressive | Per-vertex emissive control |
| BVH | SAH flat array | three-mesh-bvh quality target |
| Animation | GPU skinning + crossfade | glTF skeletal animation |
| Assets | glTF 2.0 + Draco + KTX2 | Standard pipeline |
| React | Custom reconciler (separate pkg) | R3F-style declarative scene |
| Math | Custom, Float32Array | Zero-alloc, Z-up baked in |
| Sort | Radix sort on 64-bit keys | O(n) draw call ordering |
| Uniforms | Dynamic offsets on shared buffer | Standard, both backends |
