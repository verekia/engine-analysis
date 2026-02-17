# API Design

## API Philosophy

Bonobo's API is designed to be:

1. **Explicit and predictable** — No magic, no surprises
2. **Type-safe** — Full TypeScript support with strict types
3. **Zero-cost abstractions** — High-level API compiles to tight loops
4. **React-friendly** — Declarative components for scene construction
5. **Debuggable** — Inspect internal state directly (typed arrays in devtools)

## Core API

### World

The main engine class:

```ts
class World {
  constructor(canvas: HTMLCanvasElement, options?: WorldOptions)

  spawn(desc: EntityDesc): Entity
  despawn(entity: Entity): void

  setCamera(camera: CameraDesc): void
  setLight(light: LightDesc): void

  render(): void
  destroy(): void
}

interface WorldOptions {
  maxEntities?: number      // Default: 50000
  preferWebGPU?: boolean    // Default: true
}
```

### Entity

Integer ID with setter methods:

```ts
interface Entity {
  id: number

  setPosition(x: number, y: number, z: number): void
  setRotation(x: number, y: number, z: number): void
  setScale(x: number, y: number, z: number): void
  setColor(r: number, g: number, b: number, a: number): void
  setFlag(flag: number): void

  getPosition(): [number, number, number]
  getRotation(): [number, number, number]
  getScale(): [number, number, number]
  getColor(): [number, number, number, number]

  destroy(): void
}
```

### Geometry API

```ts
// Primitive generators
createBox(width: number, height: number, depth: number): Geometry
createSphere(radius: number, widthSegments: number, heightSegments: number): Geometry
createPlane(width: number, height: number): Geometry
createCone(radius: number, height: number, radialSegments: number): Geometry
createCylinder(radiusTop: number, radiusBottom: number, height: number, radialSegments: number): Geometry
createCapsule(radius: number, height: number, radialSegments: number): Geometry
createCircle(radius: number, segments: number): Geometry

// Custom geometry
createGeometry(vertices: Float32Array, indices: Uint16Array | Uint32Array): Geometry
```

### Material API

```ts
// Simple material creation
createMaterial(type: 'static' | 'skinned' | 'textured', options?: MaterialOptions): Material

interface MaterialOptions {
  texture?: GPUTexture       // For textured materials
  sampler?: GPUSampler
}
```

## React API

### Components

```tsx
<Canvas width={800} height={600}>
  <Camera
    eye={[0, -10, 5]}
    target={[0, 0, 0]}
    up={[0, 0, 1]}
    fov={Math.PI / 4}
    near={0.1}
    far={1000}
  />

  <Light
    direction={[0, 1, -1]}
    color={[1, 1, 1]}
    ambientColor={[0.2, 0.2, 0.2]}
  />

  <Mesh
    geometry={boxGeometry}
    material={redMaterial}
    position={[0, 0, 0]}
    rotation={[0, 0, 0]}
    scale={[1, 1, 1]}
    color={[1, 0, 0, 1]}
  />
</Canvas>
```

### Hooks

```ts
// Access the World instance
const world = useWorld()

// Future: per-frame updates
useFrame((delta: number) => {
  // Called every frame
})

// Future: raycasting interaction
useRaycast(entityRef, (hit: RaycastHit) => {
  console.log('Clicked!', hit)
})
```

## Usage Examples

### Basic Scene (Vanilla)

```ts
import { World, createBox, createMaterial } from 'bonobo'

const canvas = document.querySelector('canvas')!
const world = new World(canvas)

// Create resources
const boxGeo = createBox(1, 1, 1)
const redMat = createMaterial('static')

// Set up camera
world.setCamera({
  eye: [0, -10, 5],
  target: [0, 0, 0],
  up: [0, 0, 1],
  fov: Math.PI / 4,
  near: 0.1,
  far: 1000,
})

// Set up light
world.setLight({
  direction: [0, 1, -1],
  color: [1, 1, 1],
  ambientColor: [0.2, 0.2, 0.2],
})

// Spawn entities
const box1 = world.spawn({
  geometry: boxGeo,
  material: redMat,
  position: [0, 0, 0],
})

const box2 = world.spawn({
  geometry: boxGeo,
  material: redMat,
  position: [2, 0, 0],
})

// Render loop
function loop() {
  world.render()
  requestAnimationFrame(loop)
}
requestAnimationFrame(loop)
```

### Basic Scene (React)

```tsx
import { Canvas, Mesh, Camera, Light } from 'bonobo/react'

function App() {
  const boxGeo = createBox(1, 1, 1)
  const redMat = createMaterial('static')

  return (
    <Canvas width={800} height={600}>
      <Camera
        eye={[0, -10, 5]}
        target={[0, 0, 0]}
        up={[0, 0, 1]}
        fov={Math.PI / 4}
      />

      <Light
        direction={[0, 1, -1]}
        color={[1, 1, 1]}
        ambientColor={[0.2, 0.2, 0.2]}
      />

      <Mesh
        geometry={boxGeo}
        material={redMat}
        position={[0, 0, 0]}
        color={[1, 0, 0, 1]}
      />

      <Mesh
        geometry={boxGeo}
        material={redMat}
        position={[2, 0, 0]}
        color={[0, 1, 0, 1]}
      />
    </Canvas>
  )
}
```

### Animated Scene (React)

```tsx
function RotatingBox() {
  const [rotation, setRotation] = useState([0, 0, 0])

  useEffect(() => {
    const interval = setInterval(() => {
      setRotation(([x, y, z]) => [x, y + 0.01, z])
    }, 16)
    return () => clearInterval(interval)
  }, [])

  return (
    <Mesh
      geometry={boxGeo}
      material={redMat}
      rotation={rotation}
    />
  )
}
```

## Error Handling

Clear error messages for common mistakes:

```ts
// Missing geometry or material
world.spawn({ position: [0, 0, 0] })
// Error: Entity requires geometry and material

// Invalid typed array length
createGeometry(new Float32Array(10), indices)
// Error: Vertex data length must be multiple of vertex size (48 bytes)

// Out of entity slots
world.spawn({ geometry, material })
// Error: Max entities (50000) exceeded. Increase maxEntities option.
```

## TypeScript Types

Full TypeScript support with exported types:

```ts
export interface Geometry { /* ... */ }
export interface Material { /* ... */ }
export interface Entity { /* ... */ }
export interface World { /* ... */ }

export interface EntityDesc {
  geometry: Geometry
  material: Material
  position?: [number, number, number]
  rotation?: [number, number, number]
  scale?: [number, number, number]
  color?: [number, number, number, number]
}

export interface CameraDesc {
  eye: [number, number, number]
  target: [number, number, number]
  up: [number, number, number]
  fov: number
  near?: number
  far?: number
}

export interface LightDesc {
  direction: [number, number, number]
  color: [number, number, number]
  ambientColor: [number, number, number]
}
```

## Debugging API

Expose internal state for debugging:

```ts
world.getStats(): FrameStats

interface FrameStats {
  entityCount: number
  visibleCount: number
  drawCallCount: number
  triangleCount: number
  cpuTime: number
  gpuTime: number
}
```

Access typed arrays directly (read-only):

```ts
world.entities.positions      // Float32Array
world.entities.rotations      // Float32Array
world.entities.worldMatrices  // Float32Array
// etc.
```

Inspect in Chrome DevTools → Console:

```js
> world.entities.positions
Float32Array(150000) [0, 0, 0, 2, 0, 0, ...]
```

## Future API Extensions

### Queries (v2)

Entity queries for iteration:

```ts
world.query({ withGeometry: boxGeo }).forEach(entity => {
  // Process all box entities
})

world.queryInRadius([0, 0, 0], 10).forEach(entity => {
  // Process entities within radius
})
```

### Events (v2)

Event system for entity lifecycle:

```ts
world.on('entity:spawn', (entity) => console.log('Spawned', entity))
world.on('entity:despawn', (entity) => console.log('Despawned', entity))
```

### Commands (v2)

Deferred command buffer for thread-safe updates:

```ts
const commands = world.commands()
commands.spawn({ geometry, material })
commands.despawn(entity)
commands.flush()  // Apply all commands at once
```
