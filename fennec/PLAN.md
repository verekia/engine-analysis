# Fennec — Minimal High-Performance 3D Engine

Fennec is a minimal, high-performance 3D game engine for the web. It targets **2000+ draw calls at 60fps on recent mobile devices** while providing a developer experience inspired by three.js. It supports both WebGPU (primary) and WebGL2 (fallback), with React bindings similar to React Three Fiber.

## Design Philosophy

1. **Performance by default** — No instancing needed for most use cases. Raw draw call throughput is the priority. Every object created, every function called in the hot path is justified.
2. **Minimal surface area** — Ship only what game developers actually need. No kitchen sink.
3. **Transparency that works** — Weighted Blended Order-Independent Transparency eliminates sorting headaches.
4. **Z-up, right-handed** — Matches Blender and most game conventions. GLTF Y-up is converted on import.
5. **TypeScript-first** — No semicolons, `const` arrow functions, strict types throughout.

## Key Technical Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Primary backend | WebGPU | Render bundles, bind groups, compute shaders for BVH |
| Fallback backend | WebGL2 | UBOs + VAOs for acceptable perf, broad compat |
| Transparency | Weighted Blended OIT | No sorting, works with MSAA, minimal cost |
| BVH | SAH-split flat array BVH | Cache-friendly, matches three-mesh-bvh perf |
| Shadow technique | Cascaded Shadow Maps (3 cascades) | Smooth shadows for 200m worlds |
| Bloom | Dual Kawase blur (Unreal-style) | Faster than Gaussian, better quality |
| Animation blending | GPU skinning + CPU crossfade mixer | Skeletal blending with minimal overhead |
| Coordinate system | Z-up, right-handed | Game convention, convert GLTF on import |
| React bindings | Custom reconciler | Full R3F-style declarative API |
| Asset decoding | Web Workers (Draco WASM, Basis WASM) | Non-blocking main thread |
| Scene graph | Dirty-flag transform hierarchy | Minimal recomputation |

## Architecture Documents

| Document | Contents |
|---|---|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Core architecture, module structure, data flow |
| [RENDERER.md](./RENDERER.md) | GPU abstraction layer, WebGL2/WebGPU backends, render loop |
| [SCENE-GRAPH.md](./SCENE-GRAPH.md) | Transforms, groups, parent/child, bone attachment |
| [MATERIALS.md](./MATERIALS.md) | Shaders, Basic/Lambert materials, material index system, vertex colors |
| [POST-PROCESSING.md](./POST-PROCESSING.md) | Bloom, MSAA, Weighted Blended OIT transparency |
| [ASSETS.md](./ASSETS.md) | GLTF loader, Draco, KTX2/Basis, texture pipeline |
| [ANIMATION.md](./ANIMATION.md) | Skeletal animation, GPU skinning, crossfade mixer |
| [SPATIAL.md](./SPATIAL.md) | BVH, raycasting, frustum culling |
| [SHADOWS.md](./SHADOWS.md) | Cascaded shadow maps, PCF filtering |
| [GEOMETRY.md](./GEOMETRY.md) | Parametric geometries (plane, box, sphere, etc.) |
| [INTERACTION.md](./INTERACTION.md) | Orbit controls, HTML overlay system |
| [REACT.md](./REACT.md) | React bindings, reconciler, hooks |
| [PERFORMANCE.md](./PERFORMANCE.md) | Performance strategies, benchmarks, budgets |

## Package Structure

```
fennec/
├── packages/
│   ├── fennec-core/          # Engine core (renderer, scene, math)
│   ├── fennec-materials/     # Material & shader system
│   ├── fennec-loaders/       # GLTF, Draco, KTX2 loaders
│   ├── fennec-animation/     # Skeletal animation & mixer
│   ├── fennec-spatial/       # BVH, raycasting, culling
│   ├── fennec-controls/      # Orbit controls
│   ├── fennec-overlay/       # HTML overlay system
│   └── fennec-react/         # React bindings
├── examples/
└── benchmarks/
```

## API Preview

```typescript
import { createEngine, Scene, Mesh, BoxGeometry, LambertMaterial } from 'fennec-core'
import { loadGLTF } from 'fennec-loaders'

const engine = createEngine({ canvas, antialias: true })

const scene = new Scene()
const box = new Mesh(
  new BoxGeometry({ width: 1, height: 1, depth: 1 }),
  new LambertMaterial({ color: 0x44aa88 })
)
scene.add(box)

// Material index system
const world = await loadGLTF('/world.glb', { draco: true })
world.setMaterialIndex(0, { color: 0xffffff })
world.setMaterialIndex(1, { color: 0x000000 })
world.setMaterialIndex(2, { color: 0x00ccaa, emissive: 0x00ccaa, emissiveIntensity: 0.7 })
scene.add(world)

engine.render(scene, camera)
```

### React Bindings

```tsx
import { Canvas, useFrame } from 'fennec-react'
import { useGLTF } from 'fennec-react/loaders'

const Box = () => {
  const ref = useRef()
  useFrame((_, delta) => { ref.current.rotation.z += delta })
  return (
    <mesh ref={ref}>
      <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
      <lambertMaterial color={0x44aa88} />
    </mesh>
  )
}

const App = () => (
  <Canvas shadows antialias>
    <ambientLight intensity={0.3} />
    <directionalLight position={[10, 10, 20]} castShadow />
    <Box />
  </Canvas>
)
```
