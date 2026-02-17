# React Bindings

## Philosophy

Bonobo uses a **thin React layer** with no custom reconciler, no fake Three.js objects. Components call `world.spawn()` / `world.despawn()` via `useEffect`. Props sync via refs.

## Why No Custom Reconciler?

A custom React reconciler (like R3F's) adds thousands of lines of code to map React's internal fiber tree to engine objects. It's powerful but complex and fragile across React versions.

`useEffect` + refs achieves the same result for our use case:

- Mount → spawn entity
- Prop change → update entity
- Unmount → despawn entity

It's 50 lines per component instead of a 2000-line reconciler. The tradeoff is you lose automatic deep prop spreading (`<mesh position-x={5} />`), but explicit props are clearer anyway.

## Core Components

### `<Canvas>`

Root component that creates the World and starts the render loop:

```tsx
function Canvas({ children, ...props }: CanvasProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const worldRef = useRef<World>(null)

  useEffect(() => {
    const world = new World(canvasRef.current!)
    worldRef.current = world
    // Start render loop
    const loop = () => { world.render(); requestAnimationFrame(loop) }
    requestAnimationFrame(loop)
    return () => world.destroy()
  }, [])

  return (
    <WorldContext.Provider value={worldRef}>
      <canvas ref={canvasRef} {...props} />
      {children}
    </WorldContext.Provider>
  )
}
```

### `<Mesh>`

Spawns an entity with geometry and material:

```tsx
function Mesh({ geometry, material, position, rotation, scale, color }: MeshProps) {
  const world = useWorld()
  const entityRef = useRef<Entity>(null)

  useEffect(() => {
    entityRef.current = world.spawn({ geometry, material })
    return () => entityRef.current?.destroy()
  }, [geometry, material])

  // Sync transforms without re-spawning
  useEffect(() => {
    if (position) entityRef.current?.setPosition(...position)
  }, [position?.[0], position?.[1], position?.[2]])

  useEffect(() => {
    if (rotation) entityRef.current?.setRotation(...rotation)
  }, [rotation?.[0], rotation?.[1], rotation?.[2]])

  useEffect(() => {
    if (scale) entityRef.current?.setScale(...scale)
  }, [scale?.[0], scale?.[1], scale?.[2]])

  useEffect(() => {
    if (color) entityRef.current?.setColor(...color)
  }, [color?.[0], color?.[1], color?.[2], color?.[3]])

  return null // No DOM output
}
```

### `<Camera>`

Sets the camera parameters:

```tsx
function Camera({ eye, target, up, fov, near, far }: CameraProps) {
  const world = useWorld()

  useEffect(() => {
    world.setCamera({ eye, target, up, fov, near, far })
  }, [eye, target, up, fov, near, far])

  return null
}
```

### `<Light>`

Sets the light parameters:

```tsx
function Light({ direction, color, ambientColor }: LightProps) {
  const world = useWorld()

  useEffect(() => {
    world.setLight({ direction, color, ambientColor })
  }, [direction, color, ambientColor])

  return null
}
```

## Context and Hooks

### WorldContext

```ts
const WorldContext = createContext<World | null>(null)

function useWorld(): World {
  const world = useContext(WorldContext)
  if (!world) throw new Error('useWorld must be used within <Canvas>')
  return world
}
```

## Usage Example

```tsx
import { Canvas, Mesh, Camera, Light } from 'bonobo/react'

function App() {
  const [rotation, setRotation] = useState([0, 0, 0])

  useEffect(() => {
    const interval = setInterval(() => {
      setRotation(([x, y, z]) => [x, y + 0.01, z])
    }, 16)
    return () => clearInterval(interval)
  }, [])

  return (
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
        rotation={rotation}
        scale={[1, 1, 1]}
        color={[1, 0, 0, 1]}
      />
    </Canvas>
  )
}
```

## Comparison to React Three Fiber

| Feature | R3F | Bonobo |
|---------|-----|--------|
| Custom reconciler | Yes (2000+ lines) | No |
| JSX for scene graph | Full scene graph | Flat entities only |
| Prop spreading | `<mesh position-x={5} />` | `<Mesh position={[5,0,0]} />` |
| Underlying engine | Three.js | Custom WebGPU engine |
| Performance | Object-heavy | Flat typed arrays |
| Complexity | High (reconciler + Three.js) | Low (useEffect + World) |

## Advanced Hooks (Future)

### useFrame

For per-frame updates:

```tsx
function RotatingBox() {
  const entityRef = useRef<Entity>(null)

  useFrame((delta) => {
    if (entityRef.current) {
      const [x, y, z] = entityRef.current.getRotation()
      entityRef.current.setRotation(x, y + delta, z)
    }
  })

  return <Mesh ref={entityRef} geometry={boxGeo} material={mat} />
}
```

### useRaycast

For mouse interaction:

```tsx
function ClickableBox() {
  const onClick = () => console.log('Clicked!')
  useRaycast(entityRef, onClick)

  return <Mesh ref={entityRef} geometry={boxGeo} material={mat} />
}
```

## Performance Benefits

- No reconciler overhead
- Direct entity updates (no React re-renders for transform changes)
- Minimal React component tree (just declarative spawning)
- Engine runs independently of React render cycle

## TypeScript Support

Full TypeScript support with strict typing:

```ts
interface MeshProps {
  geometry: Geometry
  material: Material
  position?: [number, number, number]
  rotation?: [number, number, number]
  scale?: [number, number, number]
  color?: [number, number, number, number]
}
```

## Future Extensions

- `<Group>` for hierarchical transforms (when scene graph is added)
- `<InstancedMesh>` for manual instancing control
- `<Sprite>` for billboards
- `<SkinnedMesh>` for animated characters (when skeletal animation is added)
