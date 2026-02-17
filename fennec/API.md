# API — Design Philosophy and Usage Examples

## Design Philosophy

Fennec's API is designed around these principles:

1. **Performance by default** — No instancing needed for most use cases. Raw draw call throughput is the priority. Every object created, every function called in the hot path is justified.
2. **Minimal surface area** — Ship only what game developers actually need. No kitchen sink.
3. **Transparency that works** — Weighted Blended Order-Independent Transparency eliminates sorting headaches.
4. **Z-up, right-handed** — Matches Blender and most game conventions. GLTF Y-up is converted on import.
5. **TypeScript-first** — No semicolons, `const` arrow functions, strict types throughout.

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

## Core API

### Engine Creation

```typescript
import { createEngine, Scene, Camera } from 'fennec-core'

const engine = createEngine({
  canvas: document.getElementById('canvas'),
  antialias: true,        // Enable MSAA (default: true)
  sampleCount: 4,         // 1, 2, or 4 (default: 4)
  powerPreference: 'high-performance',  // GPU hint
})

const scene = new Scene()
const camera = new Camera({
  fov: 60,
  aspect: canvas.width / canvas.height,
  near: 0.1,
  far: 1000,
})
camera.position = [0, -10, 5]
camera.lookAt([0, 0, 0])

engine.render(scene, camera)
```

### Creating Meshes

```typescript
import { Mesh, BoxGeometry, LambertMaterial } from 'fennec-core'

const box = new Mesh(
  new BoxGeometry({ width: 1, height: 1, depth: 1 }),
  new LambertMaterial({ color: 0x44aa88 })
)
box.position = [0, 0, 1]
box.rotation = quat.fromAxisAngle([0, 0, 1], Math.PI / 4)  // Rotate around Z
box.scale = [1, 1, 2]
scene.add(box)
```

### Loading GLTF Assets

```typescript
import { loadGLTF } from 'fennec-loaders'

// Basic loading
const world = await loadGLTF('/world.glb')
scene.add(world)

// With Draco compression
const compressed = await loadGLTF('/compressed.glb', { draco: true })
scene.add(compressed)

// Material index system for low-poly worlds
const world = await loadGLTF('/world.glb')
world.setMaterialIndex(0, { color: 0xffffff })  // White walls
world.setMaterialIndex(1, { color: 0x000000 })  // Black windows
world.setMaterialIndex(2, {
  color: 0x00ccaa,
  emissive: 0x00ccaa,
  emissiveIntensity: 0.7,  // Glowing neon
})
scene.add(world)
```

### Lighting

```typescript
import { AmbientLight, DirectionalLight } from 'fennec-core'

// Ambient light (global fill)
const ambient = new AmbientLight({
  color: [0.5, 0.6, 0.7],  // Cool sky color
  intensity: 0.3,
})
scene.add(ambient)

// Directional light (sun) with shadows
const sun = new DirectionalLight({
  direction: vec3.normalize([1, 1, -2]),  // Top-right, slightly down
  color: [1.0, 0.95, 0.9],  // Warm sunlight
  intensity: 1.2,
  castShadow: true,
  shadowConfig: {
    cascades: 3,
    mapSize: 1024,
    far: 200,
  },
})
scene.add(sun)
scene.shadowLight = sun  // Mark as the shadow-casting light
```

### Transparency

```typescript
// Transparent materials use OIT (no sorting needed!)
const glass = new LambertMaterial({
  color: 0x88ccff,
  opacity: 0.3,
  transparent: true,
  receiveShadow: true,
})

const window = new Mesh(new BoxGeometry({ width: 2, height: 3, depth: 0.1 }), glass)
scene.add(window)
```

### Animation

```typescript
import { AnimationMixer } from 'fennec-animation'

const character = await loadGLTF('/character.glb')
scene.add(character)

const mixer = new AnimationMixer(character)
const idleAction = mixer.clipAction('Idle')
const runAction = mixer.clipAction('Run')

idleAction.play()

// Crossfade to run animation
idleAction.crossFadeTo(runAction, 0.3)

// Update in render loop
const clock = { delta: 0, elapsed: 0 }
function animate() {
  const now = performance.now() / 1000
  clock.delta = now - clock.elapsed
  clock.elapsed = now

  mixer.update(clock.delta)
  engine.render(scene, camera)
  requestAnimationFrame(animate)
}
animate()
```

### Raycasting & Picking

```typescript
import { createRaycaster } from 'fennec-spatial'

const raycaster = createRaycaster()

canvas.addEventListener('click', (e) => {
  const rect = canvas.getBoundingClientRect()
  const x = ((e.clientX - rect.left) / rect.width) * 2 - 1
  const y = -((e.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera([x, y], camera)
  const hits = raycaster.intersectScene(scene)

  if (hits.length > 0) {
    console.log('Clicked on:', hits[0].object)
    console.log('Hit point:', hits[0].point)
    console.log('Distance:', hits[0].distance)
  }
})
```

### Orbit Controls

```typescript
import { createOrbitControls } from 'fennec-controls'

const controls = createOrbitControls(camera, canvas, {
  target: [0, 0, 0],
  distance: 10,
  minDistance: 2,
  maxDistance: 50,
  enableDamping: true,
})

// Update in render loop
function animate() {
  controls.update(clock.delta)
  engine.render(scene, camera)
  requestAnimationFrame(animate)
}
```

### HTML Overlay

```typescript
import { createOverlayManager } from 'fennec-overlay'

const overlay = createOverlayManager(canvasContainer)

// Add a label above a character
const label = document.createElement('div')
label.textContent = 'Player 1'
label.className = 'player-label'

const overlayItem = overlay.add({
  worldPosition: character.position,
  element: label,
  offsetY: -40,  // 40px above the character
  occlude: true,  // Hide when behind objects
})

// Update in render loop
function animate() {
  overlay.update(camera, canvas.width, canvas.height)
  engine.render(scene, camera)
  requestAnimationFrame(animate)
}
```

## React API

Fennec provides React bindings similar to React Three Fiber:

```tsx
import { Canvas, useFrame } from 'fennec-react'
import { useGLTF } from 'fennec-react/loaders'

const Box = () => {
  const ref = useRef()
  useFrame((_, delta) => {
    ref.current.rotation.z += delta
  })

  return (
    <mesh ref={ref} position={[0, 0, 1]}>
      <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
      <lambertMaterial color={0x44aa88} />
    </mesh>
  )
}

const Scene = () => {
  const world = useGLTF('/world.glb')

  return (
    <group>
      <primitive object={world} />
      <Box />
    </group>
  )
}

const App = () => (
  <Canvas
    shadows
    antialias
    camera={{ position: [0, -10, 5], fov: 60 }}
  >
    <ambientLight intensity={0.3} color={[0.5, 0.6, 0.7]} />
    <directionalLight
      position={[10, 10, 20]}
      intensity={1.2}
      castShadow
      shadowConfig={{ cascades: 3, mapSize: 1024 }}
    />
    <Scene />
  </Canvas>
)
```

### React Hooks

```tsx
import { useFrame, useThree } from 'fennec-react'

// Run code every frame
useFrame((state, delta) => {
  console.log('Frame time:', delta)
  console.log('Elapsed:', state.clock.elapsed)
})

// Access engine internals
const { scene, camera, engine } = useThree()
```

### Material Index in React

```tsx
const World = () => {
  const world = useGLTF('/world.glb')

  useEffect(() => {
    world.setMaterialIndex(0, { color: 0xffffff })
    world.setMaterialIndex(1, { color: 0x000000 })
    world.setMaterialIndex(2, {
      color: 0x00ccaa,
      emissive: 0x00ccaa,
      emissiveIntensity: 0.7,
    })
  }, [world])

  return <primitive object={world} />
}
```

## Performance API

### Debug Info

```typescript
const stats = engine.getStats()
console.log('Draw calls:', stats.drawCalls)
console.log('Triangles:', stats.triangles)
console.log('Frame time:', stats.frameTime, 'ms')
```

### Performance Budgets

Fennec is designed to hit these targets on recent mobile devices:

- **2000 draw calls @ 60fps** (main goal)
- **16ms frame budget** (60fps)
- **Shadow rendering**: < 2ms
- **Post-processing**: < 2ms
- **Culling + sorting**: < 1ms
- **Rendering**: ~10-12ms

## TypeScript Types

Fennec is fully typed. All APIs have complete TypeScript definitions:

```typescript
import type {
  Engine,
  Scene,
  Camera,
  Mesh,
  Material,
  Geometry,
  Vec3,
  Mat4,
  Quat,
} from 'fennec-core'
```

## Error Handling

Debug mode provides detailed error messages:

```typescript
// Development (verbose errors)
const engine = createEngine({ canvas, debug: true })

// Production (silent fallbacks)
const engine = createEngine({ canvas, debug: false })
```

All shader compilation errors include source context in debug mode.
