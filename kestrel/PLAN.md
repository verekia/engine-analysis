# Kestrel — Minimal High-Performance 3D Engine

## Vision

Kestrel is a minimal, high-performance 3D game engine for the web. It provides a developer experience reminiscent of three.js — intuitive scene graph, familiar material/geometry/mesh abstractions — but is architecturally designed from the ground up for raw performance. The target is **2000+ draw calls at 60 fps on recent mobile devices**, without relying on instancing.

## Core Technical Decisions

### 1. Dual Backend: WebGPU Primary, WebGL2 Fallback

- **WebGPU** is the primary target. It offers render bundles (pre-recorded command buffers), bind groups, compute shaders, and dramatically lower driver overhead.
- **WebGL2** is the mandatory fallback for devices and browsers without WebGPU support.
- A thin `Device` abstraction layer normalizes both APIs behind a unified interface. The abstraction is deliberately minimal — it doesn't try to be a general-purpose GPU abstraction, just enough for Kestrel's pipeline.
- Where WebGPU offers advanced features (render bundles, compute-based skinning, indirect draws), we use them. The WebGL2 path has equivalent but potentially slower implementations.

### 2. Coordinate System: Z-Up, Right-Handed

- World up is `+Z`, forward is `+Y`, right is `+X`.
- All math, camera defaults, gravity, and asset import pipelines respect this convention.
- GLTF assets (Y-up) are rotated on import.

### 3. Performance Philosophy

The fundamental insight is that three.js is slow not because of WebGL, but because of **per-frame JavaScript overhead**: uniform uploads, state validation, temporary object allocation, and GC pressure. Kestrel eliminates these:

- **Structure-of-Arrays (SoA)** for transforms: a single `Float32Array` holds all world matrices contiguously.
- **Dirty flags** propagate through the scene graph. Only changed matrices are recomputed.
- **Zero allocation in the render loop.** All math temporaries come from a pre-allocated pool.
- **Draw call sorting** by pipeline → material → texture minimizes GPU state changes.
- **Pre-compiled render lists** that are rebuilt only when the scene changes, not every frame.
- **WebGPU render bundles** for static geometry — the entire draw sequence is recorded once and replayed.

### 4. Transparency: Weighted Blended OIT

Order-Independent Transparency (McGuire & Bavoil 2013) eliminates the sorting nightmare. A single extra render pass writes accumulation and revealage to offscreen targets. The composite pass blends the result. No per-fragment sorting. No visual artifacts from wrong draw order. Works identically on both backends.

### 5. Bloom: Unreal-Style with Per-Vertex Control

The material index system (`_materialindex` vertex attribute) lets users assign emissive intensity per index. The bloom pipeline extracts bright fragments, performs a progressive downsample/upsample chain with tent filtering, and composites additively. Per-vertex emissive is written to an emissive buffer in the geometry pass.

### 6. Shadows: Cascading Shadow Maps

3 cascades with practical split scheme (logarithmic/linear blend). Single shadow atlas texture. PCF filtering for smooth results across a 200×200m world. Stable cascade boundaries to prevent shadow swimming.

### 7. BVH: SAH-Based, Flat Array Layout

Surface Area Heuristic BVH for both raycasting and frustum culling. Flat array layout (cache-friendly traversal). Supports incremental updates for moving objects. Performance target: match three-mesh-bvh.

### 8. Animation: Channel-Based with Simple Crossfade

Skeletal animation with per-bone TRS channels. GPU skinning in the vertex shader (or compute shader on WebGPU). Crossfade via weighted blending of two active animation states with configurable transition duration.

### 9. Asset Pipeline

- **GLTF 2.0** as the primary format.
- **Draco** decoding via WASM in a Web Worker.
- **KTX2/Basis Universal** transcoding via WASM in a Web Worker, selecting the optimal GPU format (BC, ETC, ASTC) per device.
- Material index attribute (`_materialindex`) is a custom GLTF extra or vertex attribute.

### 10. React Bindings

A `react-reconciler` based integration (like React Three Fiber) for declarative scene construction. Automatic resource disposal on unmount. Hooks for the render loop, pointer events, and controls.

### 11. HTML Overlay

DOM elements positioned at 3D world coordinates, projected to screen space each frame via CSS `transform`. Supports occlusion testing against the depth buffer.

## Project Structure

```
kestrel/
├── src/
│   ├── core/           # Engine, Scene, Renderer
│   ├── math/           # Vec3, Vec4, Mat4, Quat, Pool
│   ├── backend/        # Device abstraction, WebGPU, WebGL2
│   ├── scene/          # Object3D, Group, Mesh, SkinnedMesh
│   ├── materials/      # BasicMaterial, LambertMaterial, MaterialIndex
│   ├── geometries/     # Plane, Box, Sphere, Cone, Cylinder, Capsule, Circle
│   ├── textures/       # Texture, KTX2Loader
│   ├── animation/      # AnimationClip, AnimationMixer, Crossfade
│   ├── lights/         # DirectionalLight, AmbientLight
│   ├── shadows/        # CascadingShadowMap
│   ├── postfx/         # Bloom, MSAA resolve, OIT composite
│   ├── bvh/            # BVH build, traverse, raycast
│   ├── loaders/        # GLTFLoader, DracoDecoder, BasisDecoder
│   ├── controls/       # OrbitControls
│   ├── overlay/        # HtmlOverlay
│   └── react/          # Reconciler, hooks, components
├── shaders/
│   ├── wgsl/           # WebGPU shaders
│   └── glsl/           # WebGL2 shaders
└── workers/
    ├── draco.worker.ts
    └── basis.worker.ts
```

## Architecture Documents

| Document | Description |
|---|---|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | High-level system architecture and module relationships |
| [RENDERING.md](./RENDERING.md) | Rendering pipeline, backend abstraction, MSAA, frame structure |
| [SCENE-GRAPH.md](./SCENE-GRAPH.md) | Scene graph, transforms, hierarchy, groups, frustum culling |
| [MATERIALS.md](./MATERIALS.md) | Material system, vertex colors, material index, emissive mapping |
| [ANIMATION.md](./ANIMATION.md) | Skeletal animation, skinned meshes, crossfade, bone attachment |
| [PERFORMANCE.md](./PERFORMANCE.md) | Performance strategy for 2000+ draw calls at 60 fps |
| [SHADOWS.md](./SHADOWS.md) | Cascading shadow maps implementation |
| [BLOOM.md](./BLOOM.md) | Unreal-style bloom with per-vertex emissive |
| [TRANSPARENCY.md](./TRANSPARENCY.md) | Weighted Blended Order-Independent Transparency |
| [BVH-RAYCASTING.md](./BVH-RAYCASTING.md) | BVH construction, raycasting, spatial queries |
| [GEOMETRY.md](./GEOMETRY.md) | Parametric primitives, BufferGeometry, vertex layout |
| [ASSETS.md](./ASSETS.md) | GLTF loading, Draco decoding, KTX2/Basis transcoding |
| [REACT.md](./REACT.md) | React bindings and reconciler |
| [HTML-OVERLAY.md](./HTML-OVERLAY.md) | DOM element overlay system |
| [MATH.md](./MATH.md) | Math library design and memory management |
| [CONTROLS.md](./CONTROLS.md) | Orbit controls |

## API Preview

```typescript
import { Engine, Scene, Mesh, BoxGeometry, LambertMaterial, DirectionalLight, AmbientLight } from 'kestrel'

const engine = await Engine.create({ canvas: document.getElementById('canvas') as HTMLCanvasElement })
const scene = new Scene()

const box = new Mesh(
  new BoxGeometry({ width: 1, height: 1, depth: 1 }),
  new LambertMaterial({ color: [0.2, 0.6, 1.0] })
)
box.position.set(0, 0, 0.5)
scene.add(box)

scene.add(new AmbientLight({ intensity: 0.3 }))
scene.add(new DirectionalLight({ direction: [1, 1, -1], intensity: 0.8 }))

engine.run(scene)
```

### React Usage

```tsx
import { Canvas, useFrame } from '@kestrel/react'
import { useRef } from 'react'

const Spinner = () => {
  const ref = useRef<Mesh>(null)
  useFrame((_, dt) => { ref.current!.rotation.z += dt })
  return (
    <mesh ref={ref}>
      <boxGeometry width={1} height={1} depth={1} />
      <lambertMaterial color={[0.2, 0.6, 1.0]} />
    </mesh>
  )
}

export const App = () => (
  <Canvas>
    <ambientLight intensity={0.3} />
    <directionalLight direction={[1, 1, -1]} intensity={0.8} />
    <Spinner />
  </Canvas>
)
```
