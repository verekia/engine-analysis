# Rabbit — A Minimal High-Performance 3D Engine

## Vision

Rabbit is a minimal 3D game engine for the web that combines the developer-friendly API of three.js with the raw performance needed for mobile games. It targets **2000+ draw calls at 60fps on recent phones** without instancing, supports both WebGL2 and WebGPU, and ships with React bindings inspired by React Three Fiber.

The name comes from the rabbit — small, fast, and agile. Like its namesake, the engine is lean and quick.

## Core Principles

1. **Performance is not optional.** Every API decision is measured against draw call throughput. Zero allocations in the hot path. Minimal JS overhead per draw call.
2. **Simplicity over generality.** We support exactly the features listed below, implemented well. No plugin architecture, no extensible material system, no kitchen sink.
3. **Transparency that works.** Order-independent transparency via Weighted Blended OIT. No sorting headaches.
4. **Mobile-first.** If it runs well on a phone, it runs well everywhere.
5. **Z-up, right-handed.** Consistent coordinate system throughout.

## Feature Set

| Feature | Approach |
|---|---|
| GPU backends | WebGPU primary, WebGL2 fallback, thin HAL abstraction |
| Materials | Basic (unlit) and Lambert (diffuse) with vertex colors + material index system |
| Textures | Color maps, AO maps, KTX2 with Basis Universal transcoder |
| Model loading | glTF 2.0 with Draco mesh compression |
| Bloom | Per-vertex emissive → Unreal-style progressive down/up-sample bloom |
| Anti-aliasing | Hardware MSAA (4x default) |
| Raycasting | SAH-based BVH matching three-mesh-bvh performance |
| Animation | GPU-skinned skeletal meshes with crossfade blending |
| Culling | Per-object frustum culling with AABB |
| Transparency | Weighted Blended OIT — single pass, no sorting |
| Controls | Orbit camera controls |
| Geometries | Plane, box, sphere, cone, cylinder, capsule, circle |
| Lighting | Directional light + ambient light |
| Shadows | Cascaded Shadow Maps (3 cascades, PCF filtered) |
| Coordinate system | Z-up, right-handed |
| Scene graph | Parent/child hierarchy, Groups, bone attachment |
| HTML overlay | DOM elements projected to 3D scene positions |
| React | Full React bindings (reconciler-based, R3F-style) |

## Technical Decisions Summary

| Decision | Choice | Rationale |
|---|---|---|
| HAL design | WebGPU-modeled descriptors, mapped to WebGL2 | WebGPU's stateless model is cleaner; WebGL2 adapter maps state changes |
| Draw call perf | Material sorting + VAOs (GL) + render bundles (GPU) | Minimizes state changes, the #1 cost of draw calls |
| Transparency | Weighted Blended OIT (McGuire & Bavoil 2013) | Order-independent, single-pass, works in both backends |
| BVH | SAH split, iterative traversal, lazy build | Matches three-mesh-bvh quality; built on first raycast |
| Shadow maps | 3-cascade CSM with texel-snapped stable cascades | Smooth shadows for 200m worlds without shimmer |
| Bloom | Progressive down/up-sample (Unreal/Jimenez 2014) | High quality at low cost; driven by vertex emissive |
| Skinning | GPU skinning via uniform buffer of mat4x3 bones | Fast, works in both backends |
| Material indices | Vertex attribute → uniform array lookup | Single mesh, multiple material appearances |
| Coordinate system | Z-up, right-handed; Y-up glTF converted on import | Game-friendly coords; conversion at load boundary |
| React bindings | Custom reconciler via react-reconciler | Direct scene graph manipulation, no intermediate layer |

## Package Structure

```
rabbit/
├── src/
│   ├── core/           # Engine, Renderer, Scene, Camera
│   ├── hal/            # Hardware Abstraction Layer (WebGL2 + WebGPU)
│   ├── math/           # Vec3, Mat4, Quat, AABB, Frustum, Ray
│   ├── scene/          # Node, Group, Mesh, SkinnedMesh, Light, Bone
│   ├── materials/      # BasicMaterial, LambertMaterial, shader chunks
│   ├── geometries/     # Plane, Box, Sphere, Cone, Cylinder, Capsule, Circle
│   ├── animation/      # Skeleton, AnimationClip, AnimationMixer
│   ├── loaders/        # GLTFLoader, DracoDecoder, KTX2Loader, TextureLoader
│   ├── raycasting/     # BVH, Raycaster
│   ├── postfx/         # Bloom, OIT resolve, tone mapping
│   ├── controls/       # OrbitControls
│   ├── overlay/        # HtmlOverlay
│   └── index.ts        # Public API exports
├── react/
│   ├── src/
│   │   ├── reconciler.ts
│   │   ├── Canvas.tsx
│   │   ├── components.ts   # <mesh>, <group>, etc.
│   │   ├── hooks.ts        # useFrame, useLoader, useRabbit
│   │   └── Html.tsx
│   └── index.ts
├── package.json
└── tsconfig.json
```

## Documentation Index

Detailed architecture documents in `docs/`:

- [Architecture Overview](docs/ARCHITECTURE.md) — system layers, data flow, module boundaries
- [Rendering Pipeline](docs/RENDERING.md) — HAL, render loop, state management, MSAA
- [Materials & Shaders](docs/MATERIALS.md) — material system, shader generation, vertex colors, material indices
- [Scene Graph](docs/SCENE-GRAPH.md) — node hierarchy, transforms, groups, bone attachment
- [Asset Loading](docs/ASSETS.md) — glTF, Draco, KTX2/Basis, texture pipeline
- [Animation](docs/ANIMATION.md) — skeletal animation, GPU skinning, crossfade
- [Lighting & Shadows](docs/LIGHTING-SHADOWS.md) — directional/ambient lights, cascaded shadow maps
- [Culling & Raycasting](docs/CULLING-RAYCASTING.md) — frustum culling, SAH BVH, raycaster
- [Transparency & Bloom](docs/TRANSPARENCY-BLOOM.md) — Weighted Blended OIT, Unreal bloom
- [Interaction](docs/INTERACTION.md) — orbit controls, HTML overlay
- [React Bindings](docs/REACT.md) — reconciler, components, hooks
- [Performance](docs/PERFORMANCE.md) — draw call optimization, memory, mobile
- [API Reference](docs/API.md) — public API, code style, usage examples

## Code Style

- TypeScript throughout, strict mode
- No semicolons
- `const` arrow functions preferred over `function` declarations
- Minimal abstractions — complexity only when it earns its keep
- All public APIs documented with TSDoc comments
