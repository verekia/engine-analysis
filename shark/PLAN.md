# Shark - Minimal High-Performance 3D Engine

## Overview

Shark is a minimal 3D game engine for the web that prioritizes raw draw call throughput and developer ergonomics. It targets **2000+ draw calls at 60fps on recent mobile devices** through a WebGPU-first architecture with a complete WebGL2 fallback path. The API surface is intentionally small — similar in feel to Three.js but with none of the performance baggage.

## Coordinate System

**Z-up, right-handed:**
- **X** = right
- **Y** = forward
- **Z** = up

All math, cameras, controls, loaders, and shaders use this convention. GLTF assets (Y-up) are automatically converted on import.

## Core Design Principles

1. **Zero-allocation render loop** — no objects created per frame, no GC pressure
2. **Flat typed arrays** — transforms, bone matrices, and BVH nodes stored in contiguous `Float32Array` buffers
3. **Sort-once, draw-many** — draw calls sorted by pipeline → material → geometry; re-sorted only when the render list changes
4. **WebGPU render bundles** — pre-recorded command buffers replayed each frame for static geometry; only animated/dynamic objects re-record
5. **Minimal shader variants** — Basic (unlit) and Lambert (diffuse) with compile-time feature flags, not runtime branching
6. **No instancing required** — the engine is fast enough with individual draw calls that instancing is an optimization you add later, not a prerequisite

## Architecture Overview

```
shark/
├── core/           # Math, typed array pools, constants
├── gpu/            # GPU Abstraction Layer (GAL)
│   ├── webgl2/     # WebGL2 backend
│   └── webgpu/     # WebGPU backend
├── scene/          # Scene graph, nodes, groups, frustum culling
├── render/         # Render loop, draw call sorting, render lists
├── materials/      # Basic, Lambert, shader generation
├── geometry/       # Parametric shapes, vertex formats
├── animation/      # Skeletal animation, cross-fading, bone attachment
├── lighting/       # Directional, ambient, CSM shadows
├── postprocess/    # MSAA, bloom (Unreal-style), OIT transparency
├── raycasting/     # SAH-BVH, ray-triangle intersection
├── loaders/        # GLTF + Draco, KTX2 + Basis Universal
├── controls/       # Orbit controls
├── overlay/        # HTML overlay (DOM at 3D positions)
└── react/          # React reconciler bindings (R3F-style)
```

## Technical Stack

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary GPU API | WebGPU | Render bundles, pipeline state objects, bind groups — designed for high draw call throughput |
| Fallback GPU API | WebGL2 | Universal support; VAOs + UBOs + draw call sorting close the gap |
| Shader language | WGSL + GLSL 300 es | Dual-source shaders via a small template preprocessor; core math identical |
| GLTF decoding | Custom parser + Draco WASM | Minimal dependency, direct buffer extraction |
| Texture compression | KTX2 + Basis Universal WASM | Transcodes to device-optimal format (BC/ETC/ASTC) at load time |
| BVH | SAH-BVH in flat Float32Array | Cache-friendly traversal, matches three-mesh-bvh performance |
| Transparency | Weighted Blended OIT | No sorting, no artifacts with intersecting geometry, single pass |
| Shadows | 3-cascade CSM | Smooth shadows across 200×200m worlds with texel-snapped projections |
| Bloom | Unreal-style threshold + downsample/blur chain | Per-vertex bloom driven by material-index emissive values |
| Animation | Sampled keyframes + slerp | Cross-fade blending between two active clips |
| React | Custom reconciler via `react-reconciler` | Declarative scene graph, `useFrame`, automatic disposal |
| Language | TypeScript, no semicolons | `const` arrow functions preferred over `function` declarations |

## Performance Budget (16.6ms frame on mobile)

| Phase | Budget | Notes |
|-------|--------|-------|
| Scene graph update | ~0.5ms | Dirty-flag transform propagation |
| Frustum culling | ~0.3ms | AABB vs 6 frustum planes |
| Draw call sorting | ~0.2ms | Radix sort by composite key (only when list changes) |
| Command recording | ~1.0ms | WebGPU: replay render bundles; WebGL2: sorted GL calls |
| Shadow pass (3 cascades) | ~3.0ms | Depth-only, simplified shaders |
| Main pass (2000 calls) | ~4.0ms | Pre-sorted, minimal state changes |
| Post-processing | ~2.0ms | MSAA resolve + bloom chain + OIT composite |
| Animation eval | ~1.0ms | Bone matrix computation for active skinned meshes |
| **Total** | **~12.0ms** | **~4.6ms headroom** |

## Detailed Architecture Documents

| Document | Contents |
|----------|----------|
| [RENDERER.md](./RENDERER.md) | GPU abstraction layer, WebGL2/WebGPU backends, command encoding, render bundles |
| [SCENE_GRAPH.md](./SCENE_GRAPH.md) | Node hierarchy, transforms, groups, dirty flags, frustum culling |
| [MATERIALS.md](./MATERIALS.md) | Shader system, Basic/Lambert materials, vertex colors, material-index system, emissive/bloom |
| [GEOMETRY.md](./GEOMETRY.md) | Vertex format, parametric shapes (plane, box, sphere, cone, cylinder, capsule, circle) |
| [ASSETS.md](./ASSETS.md) | GLTF loader, Draco decoding, KTX2/Basis transcoding, Y-up → Z-up conversion |
| [ANIMATION.md](./ANIMATION.md) | Skeletal animation, skinning, cross-fade blending, bone attachment |
| [LIGHTING.md](./LIGHTING.md) | Directional light, ambient light, 3-cascade shadow maps |
| [POST_PROCESSING.md](./POST_PROCESSING.md) | MSAA, Unreal-style bloom, weighted blended OIT transparency |
| [RAYCASTING.md](./RAYCASTING.md) | SAH-BVH construction, ray-AABB/ray-triangle intersection, traversal |
| [CONTROLS.md](./CONTROLS.md) | Orbit controls with damping, Z-up adaptation |
| [HTML_OVERLAY.md](./HTML_OVERLAY.md) | DOM element positioning at 3D coordinates, occlusion, CSS3D-style |
| [REACT.md](./REACT.md) | React reconciler, declarative components, hooks, event system |

## Package Structure

```
@shark3d/core        # Engine core (renderer, scene, materials, geometry, animation, lighting, etc.)
@shark3d/react       # React bindings
@shark3d/draco       # Draco WASM decoder (lazy-loaded)
@shark3d/basis       # Basis Universal WASM transcoder (lazy-loaded)
```

The core package is tree-shakeable. If you don't import `OrbitControls`, it's not in your bundle. WASM decoders are separate packages loaded on demand when a GLTF uses Draco compression or a texture uses KTX2/Basis.

## Example Usage (Imperative)

```ts
import { Engine, Scene, PerspectiveCamera, Mesh, BoxGeometry, LambertMaterial, DirectionalLight, AmbientLight } from '@shark3d/core'

const engine = await Engine.create({ canvas: document.getElementById('canvas') as HTMLCanvasElement })
const scene = new Scene()
const camera = new PerspectiveCamera({ fov: 60, near: 0.1, far: 500 })
camera.position.set(0, -5, 3)
camera.lookAt(0, 0, 0)

scene.add(new AmbientLight({ color: 0x404040 }))
scene.add(new DirectionalLight({ color: 0xffffff, intensity: 1, castShadow: true }))

const box = new Mesh(new BoxGeometry({ width: 1, height: 1, depth: 1 }), new LambertMaterial({ color: 0x00aaff }))
scene.add(box)

engine.onFrame(() => {
  box.rotation.z += 0.01
  engine.render(scene, camera)
})
```

## Example Usage (React)

```tsx
import { Canvas, useFrame } from '@shark3d/react'
import { useRef } from 'react'

const Box = () => {
  const ref = useRef()
  useFrame(() => { ref.current.rotation.z += 0.01 })
  return (
    <mesh ref={ref}>
      <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
      <lambertMaterial color={0x00aaff} />
    </mesh>
  )
}

const App = () => (
  <Canvas camera={{ fov: 60, position: [0, -5, 3] }}>
    <ambientLight color={0x404040} />
    <directionalLight color={0xffffff} intensity={1} castShadow />
    <Box />
  </Canvas>
)
```
