# React Bindings Architecture

## Overview

Shark provides React bindings in the `@shark3d/react` package, following the architecture pioneered by React Three Fiber (R3F). A custom React reconciler maps JSX elements to Shark scene graph nodes, enabling a fully declarative approach to 3D scene construction.

## Custom Reconciler

### Why a Custom Reconciler

React's built-in DOM reconciler only knows about HTML elements. To make `<mesh>`, `<group>`, `<boxGeometry>`, etc. work as first-class React elements, we implement a custom reconciler using the `react-reconciler` package.

The reconciler maps React lifecycle operations to scene graph mutations:

| React Operation | Scene Graph Effect |
|----------------|-------------------|
| `createElement('mesh')` | `new Mesh()` |
| `appendChild(parent, child)` | `parent.add(child)` |
| `removeChild(parent, child)` | `parent.remove(child); child.dispose()` |
| `commitUpdate(node, props)` | Apply changed properties |

### Reconciler Implementation

```typescript
import Reconciler from 'react-reconciler'

const hostConfig = {
  supportsMutation: true,
  supportsPersistence: false,

  createInstance(type: string, props: Props) {
    switch (type) {
      case 'mesh':              return new Mesh()
      case 'group':             return new Group()
      case 'skinnedMesh':       return new SkinnedMesh()
      case 'boxGeometry':       return new BoxGeometry(props.args)
      case 'sphereGeometry':    return new SphereGeometry(props.args)
      case 'planeGeometry':     return new PlaneGeometry(props.args)
      case 'coneGeometry':      return new ConeGeometry(props.args)
      case 'cylinderGeometry':  return new CylinderGeometry(props.args)
      case 'capsuleGeometry':   return new CapsuleGeometry(props.args)
      case 'circleGeometry':    return new CircleGeometry(props.args)
      case 'basicMaterial':     return new BasicMaterial(props)
      case 'lambertMaterial':   return new LambertMaterial(props)
      case 'ambientLight':      return new AmbientLight(props)
      case 'directionalLight':  return new DirectionalLight(props)
      case 'html':              return new HtmlAnchor(props)
      default:                  throw new Error(`Unknown element: ${type}`)
    }
  },

  appendChild(parent: any, child: any) {
    // Geometry and Material attach to parent Mesh specially
    if (child instanceof Geometry) {
      parent.geometry = child
      return
    }
    if (child instanceof Material) {
      parent.material = child
      return
    }
    parent.add(child)
  },

  removeChild(parent: any, child: any) {
    if (child instanceof Geometry || child instanceof Material) {
      child.dispose()
      return
    }
    parent.remove(child)
    child.dispose()
  },

  commitUpdate(instance: any, _payload: any, _type: string, oldProps: Props, newProps: Props) {
    applyProps(instance, newProps, oldProps)
  },

  // ... other required lifecycle methods
}

const reconciler = Reconciler(hostConfig)
```

### Property Application

```typescript
const applyProps = (instance: any, newProps: Props, oldProps: Props = {}) => {
  for (const [key, value] of Object.entries(newProps)) {
    if (key === 'children' || key === 'ref' || key === 'args') continue

    if (key === 'position' && Array.isArray(value)) {
      instance.position.set(value[0], value[1], value[2])
    } else if (key === 'rotation' && Array.isArray(value)) {
      instance.rotation.setFromEuler(value[0], value[1], value[2])
    } else if (key === 'scale') {
      if (typeof value === 'number') instance.scale.set(value, value, value)
      else instance.scale.set(value[0], value[1], value[2])
    } else if (key === 'color' && typeof value === 'number') {
      instance.color.setHex(value)
    } else if (key.startsWith('on')) {
      // Store event handlers for the pointer event system
      instance.__handlers = instance.__handlers ?? {}
      instance.__handlers[key] = value
    } else {
      instance[key] = value
    }
  }
}
```

## Canvas Component

The root `<Canvas>` component creates the engine, scene, camera, and render loop:

```tsx
interface CanvasProps {
  camera?: {
    fov?: number
    near?: number
    far?: number
    position?: [number, number, number]
  }
  style?: React.CSSProperties
  className?: string
  shadows?: boolean
  msaa?: boolean | number
  bloom?: BloomConfig | boolean
  onCreated?: (state: SharkState) => void
  children: React.ReactNode
}

const Canvas: React.FC<CanvasProps> = ({
  camera: cameraProps,
  children,
  shadows,
  msaa,
  bloom,
  onCreated,
  ...rest
}) => {
  const containerRef = useRef<HTMLDivElement>(null)
  const stateRef = useRef<SharkState>(null)

  useEffect(() => {
    const container = containerRef.current!

    // Create canvas
    const canvas = document.createElement('canvas')
    container.appendChild(canvas)

    // Create overlay div for HtmlAnchors
    const overlay = document.createElement('div')
    overlay.style.cssText =
      'position:absolute;top:0;left:0;width:100%;height:100%;pointer-events:none;overflow:hidden;'
    container.appendChild(overlay)

    // Initialize engine asynchronously
    Engine.create({ canvas, shadows, msaa, bloom }).then((engine) => {
      const scene = new Scene()
      const camera = new PerspectiveCamera(cameraProps)
      if (cameraProps?.position) camera.position.set(...cameraProps.position)
      camera.lookAt(0, 0, 0)

      const state: SharkState = { engine, scene, camera, canvas, overlay }
      stateRef.current = state

      // Mount React tree
      const root = reconciler.createContainer(scene, 0, null, false, null, '', console.error, null)
      reconciler.updateContainer(
        <SharkContext.Provider value={state}>{children}</SharkContext.Provider>,
        root, null, () => {},
      )

      // Render loop
      engine.onFrame((delta) => {
        frameCallbacks.forEach((cb) => cb(state, delta))
        engine.render(scene, camera)
        updateOverlays(scene, camera, canvas, overlay)
      })

      onCreated?.(state)
    })

    return () => {
      stateRef.current?.engine.dispose()
      container.innerHTML = ''
    }
  }, [])

  return (
    <div
      ref={containerRef}
      style={{ position: 'relative', width: '100%', height: '100%', ...rest.style }}
      className={rest.className}
    />
  )
}
```

## Context and Hooks

### SharkContext

```typescript
interface SharkState {
  engine: Engine
  scene: Scene
  camera: Camera
  canvas: HTMLCanvasElement
  overlay: HTMLDivElement
}

const SharkContext = createContext<SharkState | null>(null)
```

### useShark

Access engine state from any child component:

```typescript
const useShark = (): SharkState => {
  const state = useContext(SharkContext)
  if (!state) throw new Error('useShark must be used within <Canvas>')
  return state
}
```

### useFrame

Register a per-frame callback. Runs every frame before rendering, bypassing React entirely for performance:

```typescript
type FrameCallback = (state: SharkState, delta: number) => void
const frameCallbacks = new Set<FrameCallback>()

const useFrame = (callback: FrameCallback) => {
  const ref = useRef(callback)
  ref.current = callback  // Always use latest closure

  useEffect(() => {
    const cb: FrameCallback = (s, d) => ref.current(s, d)
    frameCallbacks.add(cb)
    return () => { frameCallbacks.delete(cb) }
  }, [])
}
```

### useLoader

Async resource loading with Suspense integration:

```typescript
const loaderCache = new Map<string, { promise?: Promise<any>, result?: any, error?: any }>()

const useLoader = <T,>(loader: { load: (url: string) => Promise<T> }, url: string): T => {
  const entry = loaderCache.get(url)
  if (entry?.result) return entry.result
  if (entry?.error) throw entry.error
  if (entry?.promise) throw entry.promise

  const promise = loader.load(url).then(
    (result) => { loaderCache.get(url)!.result = result },
    (error) => { loaderCache.get(url)!.error = error },
  )
  loaderCache.set(url, { promise })
  throw promise  // Suspense boundary catches this
}

// Usage:
const Model = ({ url }: { url: string }) => {
  const gltf = useLoader(GLTFLoader, url)
  return <primitive object={gltf.scene} />
}

// With Suspense:
<Suspense fallback={<LoadingSpinner />}>
  <Model url="/models/character.glb" />
</Suspense>
```

## Event System

### Pointer Events via Raycasting

DOM-style pointer events are mapped to 3D objects using raycasting:

```typescript
const setupEventSystem = (state: SharkState) => {
  const raycaster = new Raycaster()

  const handleEvent = (type: string, e: PointerEvent) => {
    const rect = state.canvas.getBoundingClientRect()
    const ndc: Vec2 = [
      ((e.clientX - rect.left) / rect.width) * 2 - 1,
      -((e.clientY - rect.top) / rect.height) * 2 + 1,
    ]

    raycaster.setFromCamera(ndc, state.camera)
    const hits = raycaster.intersectObjects(state.scene.children, true)

    for (const hit of hits) {
      let node: Node3D | null = hit.object
      while (node) {
        const handler = node.__handlers?.[type]
        if (handler) {
          handler({ ...hit, nativeEvent: e, stopPropagation: () => { node = null } })
          break
        }
        node = node.parent
      }
    }
  }

  state.canvas.addEventListener('click', (e) => handleEvent('onClick', e))
  state.canvas.addEventListener('pointerdown', (e) => handleEvent('onPointerDown', e))
  state.canvas.addEventListener('pointerup', (e) => handleEvent('onPointerUp', e))
  state.canvas.addEventListener('pointermove', (e) => handleEvent('onPointerMove', e))
}
```

### Usage

```tsx
const InteractiveBox = () => {
  const [hovered, setHovered] = useState(false)

  return (
    <mesh
      onClick={(e) => console.log('clicked at', e.point)}
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
    >
      <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
      <lambertMaterial color={hovered ? 0xff0000 : 0x00ff00} />
    </mesh>
  )
}
```

## Special Elements

### `<primitive>`

Mount an existing Shark object (e.g., GLTF result) into the React tree:

```tsx
const Model = () => {
  const gltf = useLoader(GLTFLoader, '/model.glb')
  return <primitive object={gltf.scene} position={[0, 0, 0]} />
}
```

The reconciler detects `primitive` and uses the provided object directly.

### `<Html>`

Renders DOM content at a 3D position using React portals:

```tsx
<Html position={[0, 0, 2]} center={[0.5, 0.5]}>
  <div className="tooltip">Hello world</div>
</Html>
```

Internally creates an `HtmlAnchor` in the scene graph and renders children via `ReactDOM.createPortal` into the anchor's DOM element.

## Component Library

All Shark scene graph types as lowercase JSX elements:

```tsx
// Scene graph
<group />
<mesh />
<skinnedMesh />

// Geometries (children of <mesh>)
<boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
<sphereGeometry args={{ radius: 0.5, widthSegments: 32 }} />
<planeGeometry args={{ width: 10, height: 10 }} />
<coneGeometry args={{ radius: 0.5, height: 1 }} />
<cylinderGeometry args={{ topRadius: 0.5, bottomRadius: 0.5, height: 1 }} />
<capsuleGeometry args={{ radius: 0.3, height: 1 }} />
<circleGeometry args={{ radius: 0.5 }} />

// Materials (children of <mesh>)
<basicMaterial color={0xff0000} transparent opacity={0.5} />
<lambertMaterial color={0x00ff00} receiveShadows />

// Lights
<ambientLight color={0x404040} intensity={0.3} />
<directionalLight color={0xffffff} intensity={1} position={[50, -50, 100]} castShadow />

// Overlay
<Html position={[0, 0, 2]} center={[0.5, 0.5]}>
  <div>Overlay content</div>
</Html>

// Controls
<OrbitControls target={[0, 0, 0]} enableDamping />
```

## Automatic Disposal

When components unmount, the reconciler's `removeChild`:
1. Removes the node from its scene graph parent
2. Disposes GPU resources (geometry buffers, textures)
3. Invalidates BVH caches
4. Removes HTML overlay elements

No manual cleanup needed — GPU memory is freed automatically.

## Performance Notes

1. **Reconciler overhead** is negligible for scene setup (rare). For per-frame updates, `useFrame` bypasses React entirely.

2. **Prop diffing**: Only changed props are applied, minimizing scene graph mutations.

3. **Direct access via refs** for imperative per-frame operations:
   ```tsx
   const ref = useRef<Mesh>()
   useFrame(() => { ref.current!.rotation.z += 0.01 })
   return <mesh ref={ref}>...</mesh>
   ```

4. **Memoization**: Use `React.memo` for expensive subtrees that change infrequently.
