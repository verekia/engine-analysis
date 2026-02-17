# React Bindings

## Overview

Mantis provides a React binding layer inspired by React Three Fiber (R3F). It
uses a custom `react-reconciler` to map JSX elements to Mantis scene graph
objects, enabling declarative 3D scene construction with automatic lifecycle
management.

## Architecture

```
┌──────────────────────────────────────────┐
│            React Component Tree           │
│  <Canvas> → <group> → <mesh> → ...       │
├──────────────────────────────────────────┤
│          Custom React Reconciler          │
│  createInstance · appendChild · removeChild│
│  commitUpdate · prepareUpdate             │
├──────────────────────────────────────────┤
│           Mantis Scene Graph              │
│  Engine → Scene → Node/Mesh/Group/Light   │
└──────────────────────────────────────────┘
```

The reconciler translates React's component lifecycle into Mantis scene graph
operations:

| React Operation | Mantis Operation |
|---|---|
| Mount `<mesh>` | `scene.createMesh(geometry, material)` |
| Mount `<group>` | `scene.createGroup()` |
| Mount as child | `parent.add(child)` |
| Unmount | `node.dispose()` (frees GPU resources) |
| Update props | `node.position = [...]`, `material.color = [...]` |

## Canvas Component

The root component that creates the Mantis engine and provides context:

```tsx
import { Canvas } from 'mantis/react'

const App = () => (
  <Canvas
    antialias={true}
    shadows={true}
    bloom={true}
    camera={{ fov: 60, near: 0.1, far: 500, position: [10, -20, 15] }}
    style={{ width: '100%', height: '100vh' }}
  >
    <Scene />
  </Canvas>
)
```

`<Canvas>` handles:
- Creating the `<canvas>` DOM element
- Initializing the Mantis engine (auto-detecting WebGPU/WebGL2)
- Setting up the render loop via `requestAnimationFrame`
- Providing engine context to all children
- Resizing on container size changes (via ResizeObserver)
- Cleanup on unmount

### Canvas Props

```typescript
interface CanvasProps {
  antialias?: boolean          // default true, 4× MSAA
  shadows?: boolean            // default false
  bloom?: boolean              // default false
  bloomIntensity?: number      // default 0.5
  camera?: CameraConfig        // default perspective camera
  style?: React.CSSProperties  // applied to container div
  className?: string
  onCreated?: (engine: Engine) => void  // called after engine init
  frameloop?: 'always' | 'demand'      // default 'always'
}
```

## Scene Elements

JSX elements map to Mantis scene objects:

### `<mesh>`

```tsx
<mesh position={[0, 0, 5]} rotation={[0, 0, 0, 1]} scale={[2, 2, 2]}>
  <sphereGeometry args={{ radius: 1, widthSegments: 32 }} />
  <lambertMaterial color={[0.8, 0.2, 0.1]} shadows />
</mesh>
```

Props:
- `position`: `[x, y, z]` — maps to `node.position`
- `rotation`: `[x, y, z, w]` quaternion — maps to `node.rotation`
- `scale`: `[x, y, z]` or `number` (uniform) — maps to `node.scale`
- `visible`: `boolean`
- `castShadow`: `boolean`
- `name`: `string`
- `onClick`, `onPointerOver`, `onPointerOut`: event handlers (via raycasting)

### `<group>`

```tsx
<group position={[10, 0, 0]} rotation={[0, 0, 0.38, 0.92]}>
  <mesh>...</mesh>
  <mesh>...</mesh>
</group>
```

### `<directionalLight>`

```tsx
<directionalLight
  direction={[-0.5, -0.3, -0.8]}
  color={[1, 0.95, 0.9]}
  intensity={1.0}
  castShadow
/>
```

### `<ambientLight>`

```tsx
<ambientLight color={[0.3, 0.3, 0.4]} intensity={1.0} />
```

### Geometry Elements

Attached as children of `<mesh>`:

```tsx
<planeGeometry args={{ width: 10, height: 10 }} />
<boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
<sphereGeometry args={{ radius: 0.5, widthSegments: 32 }} />
<coneGeometry args={{ radius: 0.5, height: 1 }} />
<cylinderGeometry args={{ radiusTop: 0.5, radiusBottom: 0.5, height: 1 }} />
<capsuleGeometry args={{ radius: 0.25, height: 1 }} />
<circleGeometry args={{ radius: 0.5, segments: 32 }} />
```

### Material Elements

Attached as children of `<mesh>`:

```tsx
<basicMaterial color={[1, 0.5, 0]} transparent opacity={0.5} />
<lambertMaterial
  color={[1, 1, 1]}
  shadows
  palette={[
    { color: [1, 1, 1] },
    { color: [0, 0.8, 0.7], emissive: [0, 0.8, 0.7], emissiveIntensity: 0.7 },
  ]}
/>
```

## Hooks

### `useFrame`

Register a callback that runs every frame before rendering:

```tsx
import { useFrame } from 'mantis/react'

const RotatingCube = () => {
  const meshRef = useRef<Mesh>(null)

  useFrame((dt, elapsed) => {
    if (meshRef.current) {
      meshRef.current.rotation = quatFromAxisAngle([0, 0, 1], elapsed * 0.5)
    }
  })

  return (
    <mesh ref={meshRef}>
      <boxGeometry />
      <lambertMaterial color={[0.5, 0.8, 1.0]} />
    </mesh>
  )
}
```

The callback receives `(deltaTime, elapsedTime)`. Callbacks are called in
registration order before the render pass.

### `useEngine`

Access the Mantis engine instance:

```tsx
const engine = useEngine()
engine.bloomIntensity = 0.8
```

### `useScene`

Access the scene for imperative operations:

```tsx
const scene = useScene()
const allMeshes = scene.findByName('Enemy')
```

### `useLoader`

Load assets with React Suspense integration:

```tsx
import { useLoader, GLTFLoader } from 'mantis/react'

const Character = () => {
  const gltf = useLoader(GLTFLoader, '/models/character.glb')

  return <primitive object={gltf.scene} position={[0, 0, 0]} />
}

// Wrapped in Suspense
const App = () => (
  <Canvas>
    <Suspense fallback={null}>
      <Character />
    </Suspense>
  </Canvas>
)
```

`useLoader` caches results — loading the same URL twice returns the cached
result immediately.

### `useRaycast`

Perform raycasting from the camera through a screen position:

```tsx
const raycast = useRaycast()

const handleClick = (event: React.MouseEvent) => {
  const hit = raycast(event.clientX, event.clientY)
  if (hit) {
    console.log('Hit:', hit.object.name, 'at', hit.point)
  }
}
```

## Pointer Events

Meshes with `onClick`, `onPointerOver`, or `onPointerOut` props automatically
participate in raycasting. Only meshes with event handlers are tested — no
performance cost for non-interactive meshes.

```tsx
const InteractiveCube = () => {
  const [hovered, setHovered] = useState(false)

  return (
    <mesh
      onClick={(event) => console.log('Clicked!', event.point)}
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
    >
      <boxGeometry />
      <lambertMaterial color={hovered ? [1, 0.5, 0] : [0.5, 0.5, 0.5]} />
    </mesh>
  )
}
```

Event object:
```typescript
interface MantisPointerEvent {
  object: Mesh          // the mesh that was hit
  point: Vec3           // world-space hit position
  normal: Vec3          // world-space surface normal
  distance: number      // distance from camera
  faceIndex: number     // triangle index
  uv: Vec2              // UV at hit point
  nativeEvent: MouseEvent | TouchEvent
}
```

Raycasting for pointer events runs once per pointer move/click, testing only
meshes with registered handlers. The scene BVH accelerates this.

## `<primitive>`

Mount a pre-built Mantis object (e.g., from a loaded glTF) into the React tree:

```tsx
const gltf = useLoader(GLTFLoader, '/models/tree.glb')

// Attach the loaded scene graph as-is
<primitive object={gltf.scene} position={[5, 10, 0]} />
```

`<primitive>` wraps an existing Mantis node without recreating it. Props like
`position`, `rotation`, `scale` are applied to the root node.

## `<Html>`

Render DOM content at a 3D position (see [HTML-OVERLAY.md](./HTML-OVERLAY.md)):

```tsx
import { Html } from 'mantis/react'

<mesh position={[0, 0, 10]}>
  <sphereGeometry args={{ radius: 0.5 }} />
  <basicMaterial color={[1, 0, 0]} />
  <Html position={[0, 0, 1.5]} center>
    <div className="label">Enemy HP: 100</div>
  </Html>
</mesh>
```

## Reconciler Implementation

The reconciler is built with `react-reconciler`:

```typescript
import Reconciler from 'react-reconciler'

const reconciler = Reconciler({
  supportsMutation: true,
  supportsPersistence: false,

  createInstance(type, props, rootContainer) {
    switch (type) {
      case 'mesh':
        return rootContainer.scene.createMesh()
      case 'group':
        return rootContainer.scene.createGroup()
      case 'directionalLight':
        return rootContainer.scene.createDirectionalLight(props)
      case 'ambientLight':
        return rootContainer.scene.createAmbientLight(props)
      case 'sphereGeometry':
        return { type: 'geometry', data: createSphere(props.args) }
      case 'lambertMaterial':
        return { type: 'material', data: new LambertMaterial(props) }
      // ... other element types
    }
  },

  appendChild(parent, child) {
    if (child.type === 'geometry') {
      parent.geometry = new Geometry(parent.engine.device, child.data)
    } else if (child.type === 'material') {
      parent.material = child.data
    } else {
      parent.add(child)
    }
  },

  removeChild(parent, child) {
    if (child.dispose) child.dispose()
    parent.remove?.(child)
  },

  commitUpdate(instance, updatePayload, type, oldProps, newProps) {
    applyProps(instance, newProps)
  },

  // ... other required methods
})
```

### Property Application

```typescript
const applyProps = (instance: any, props: Record<string, any>) => {
  for (const [key, value] of Object.entries(props)) {
    switch (key) {
      case 'position':
        instance.position = value
        break
      case 'rotation':
        instance.rotation = value
        break
      case 'scale':
        if (typeof value === 'number') {
          instance.scale = [value, value, value]
        } else {
          instance.scale = value
        }
        break
      case 'visible':
        instance.visible = value
        break
      case 'color':
        if (instance.color !== undefined) instance.color = value
        break
      // ... other props
      case 'children':
      case 'ref':
        break // handled by React
      default:
        if (key in instance) instance[key] = value
    }
  }
}
```

## Automatic Disposal

When a component unmounts, the reconciler calls `dispose()` on the
corresponding Mantis object. This frees:
- GPU buffers (vertex, index)
- GPU textures
- Compiled shader pipelines (reference counted — only freed when no material
  uses them)
- Scene graph node slot (returned to free list)

Materials and geometries used by multiple meshes are reference counted. A
shared geometry is only freed when the last mesh using it unmounts.

## Performance Considerations

- **Render loop runs outside React** — `requestAnimationFrame` drives Mantis
  directly. React re-renders only affect the scene graph, not the render loop.
- **Prop updates are batched** — Multiple prop changes in a single React render
  cycle produce one `commitUpdate` call per component.
- **useFrame callbacks avoid React** — Frame logic runs directly on Mantis
  objects via refs, bypassing React's reconciliation.
- **Suspense for loading** — Asset loading suspends the React subtree, not the
  render loop. The rest of the scene continues rendering while models load.
