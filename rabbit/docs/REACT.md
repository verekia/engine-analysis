# React Bindings

## Overview

Rabbit's React bindings provide a declarative API for building 3D scenes, inspired by React Three Fiber (R3F). The bindings use a custom React reconciler that directly manipulates the Rabbit scene graph, with no intermediate representation.

The package is published separately as `@rabbit/react`.

## Architecture

```
JSX Tree                          Rabbit Scene Graph
─────────                         ──────────────────
<Canvas>                          Engine + Renderer
  <ambientLight />        ──▶     AmbientLight
  <directionalLight />    ──▶     DirectionalLight
  <group>                 ──▶     Group
    <mesh>                ──▶       Mesh
      <boxGeometry />     ──▶         .geometry = new BoxGeometry(...)
      <lambertMaterial /> ──▶         .material = new LambertMaterial(...)
    </mesh>
  </group>
</Canvas>

React Reconciler bridges JSX operations → scene graph mutations
```

## Canvas Component

The root component that initializes the engine and provides context:

```tsx
interface CanvasProps {
  children: React.ReactNode
  style?: React.CSSProperties
  className?: string

  // Engine config
  backend?: 'webgpu' | 'webgl2' | 'auto'   // default 'auto'
  msaa?: 1 | 2 | 4                          // default 4
  pixelRatio?: number                        // default window.devicePixelRatio

  // Camera defaults
  camera?: {
    fov?: number                             // default 60
    near?: number                            // default 0.1
    far?: number                             // default 1000
    position?: [number, number, number]      // default [5, 5, 5]
  }

  // Bloom
  bloom?: BloomConfig | false

  // Callbacks
  onCreated?: (state: RabbitState) => void
}

const Canvas: React.FC<CanvasProps> = ({ children, ...props }) => {
  const containerRef = useRef<HTMLDivElement>(null)
  const stateRef = useRef<RabbitState | null>(null)

  useEffect(() => {
    const container = containerRef.current!
    const canvas = document.createElement('canvas')
    container.appendChild(canvas)

    // Initialize engine
    const engine = new Engine(canvas, {
      backend: props.backend ?? 'auto',
      msaa: props.msaa ?? 4,
      pixelRatio: props.pixelRatio,
    })

    const scene = new Scene()
    const camera = new Camera()
    camera.fov = (props.camera?.fov ?? 60) * (Math.PI / 180)
    if (props.camera?.position) {
      camera.position.set(...props.camera.position)
    }

    const state: RabbitState = {
      engine,
      scene,
      camera,
      canvas,
      gl: engine.device,
      overlay: new HtmlOverlay(canvas),
      raycaster: new Raycaster(),
      size: { width: canvas.width, height: canvas.height },
      clock: { elapsed: 0, delta: 0 },
      frameCallbacks: new Set(),
    }

    stateRef.current = state

    // Mount React tree into the scene
    const root = createReconcilerRoot(scene, state)
    root.render(
      <RabbitContext.Provider value={state}>
        {children}
      </RabbitContext.Provider>
    )

    // Start render loop
    engine.onTick((dt) => {
      state.clock.delta = dt
      state.clock.elapsed += dt

      // Run frame callbacks
      for (const cb of state.frameCallbacks) {
        cb(state, dt)
      }

      // Render
      engine.renderer.render(scene, camera)
      state.overlay.update(camera, canvas.width, canvas.height)
    })

    props.onCreated?.(state)

    return () => {
      root.unmount()
      engine.destroy()
    }
  }, [])

  return <div ref={containerRef} style={props.style} className={props.className} />
}
```

## React Reconciler

The reconciler maps React lifecycle events to scene graph operations:

```typescript
import Reconciler from 'react-reconciler'

const reconciler = Reconciler({
  supportsMutation: true,
  supportsPersistence: false,

  createInstance(type: string, props: any, root: any) {
    // Map JSX element type to Rabbit object
    switch (type) {
      case 'group':
        return applyProps(new Group(), props)
      case 'mesh':
        return applyProps(new Mesh(), props)
      case 'skinnedMesh':
        return applyProps(new SkinnedMesh(), props)
      case 'bone':
        return applyProps(new Bone(), props)
      case 'ambientLight':
        return applyProps(new AmbientLight(), props)
      case 'directionalLight':
        return applyProps(new DirectionalLight(), props)

      // Geometries — attached to parent mesh
      case 'boxGeometry':
        return { __type: 'geometry', value: new BoxGeometry(props) }
      case 'sphereGeometry':
        return { __type: 'geometry', value: new SphereGeometry(props) }
      case 'planeGeometry':
        return { __type: 'geometry', value: new PlaneGeometry(props) }
      case 'cylinderGeometry':
        return { __type: 'geometry', value: new CylinderGeometry(props) }
      case 'coneGeometry':
        return { __type: 'geometry', value: new ConeGeometry(props) }
      case 'capsuleGeometry':
        return { __type: 'geometry', value: new CapsuleGeometry(props) }
      case 'circleGeometry':
        return { __type: 'geometry', value: new CircleGeometry(props) }

      // Materials — attached to parent mesh
      case 'basicMaterial':
        return { __type: 'material', value: new BasicMaterial(props) }
      case 'lambertMaterial':
        return { __type: 'material', value: new LambertMaterial(props) }

      default:
        throw new Error(`Unknown element type: ${type}`)
    }
  },

  appendChild(parent: any, child: any) {
    if (child.__type === 'geometry' && parent instanceof Mesh) {
      parent.geometry = child.value
    } else if (child.__type === 'material' && parent instanceof Mesh) {
      parent.material = child.value
    } else if (parent instanceof Node && child instanceof Node) {
      parent.add(child)
    }
  },

  removeChild(parent: any, child: any) {
    if (child instanceof Node && parent instanceof Node) {
      parent.remove(child)
    }
  },

  commitUpdate(instance: any, updatePayload: any, type: string, oldProps: any, newProps: any) {
    if (instance instanceof Node) {
      applyProps(instance, newProps, oldProps)
    } else if (instance.__type === 'geometry') {
      // Rebuild geometry with new props
      instance.value.dispose()
      instance.value = createGeometry(type, newProps)
      // Re-attach to parent mesh
      if (instance.__parentMesh) {
        instance.__parentMesh.geometry = instance.value
      }
    } else if (instance.__type === 'material') {
      applyProps(instance.value, newProps, oldProps)
    }
  },

  // ... other required reconciler methods (mostly no-ops)
})
```

### Property Application

The `applyProps` function maps JSX props to Rabbit object properties:

```typescript
const applyProps = (instance: any, props: any, oldProps?: any) => {
  for (const [key, value] of Object.entries(props)) {
    if (key === 'children') continue
    if (key === 'ref') continue
    if (oldProps && oldProps[key] === value) continue // no change

    switch (key) {
      case 'position':
        instance.position.set(...(value as [number, number, number]))
        break
      case 'rotation':
        // Accept euler angles [x, y, z] for convenience
        instance.setRotationFromEuler(...(value as [number, number, number]))
        break
      case 'scale':
        if (typeof value === 'number') {
          instance.scale.set(value, value, value)
        } else {
          instance.scale.set(...(value as [number, number, number]))
        }
        break
      case 'visible':
        instance.visible = value
        break
      case 'name':
        instance.name = value
        break
      default:
        // Direct property assignment for material/light props
        if (key in instance) {
          instance[key] = value
        }
        break
    }
  }
  return instance
}
```

## Hooks

### useFrame

Register a callback that runs every frame. Automatically cleaned up on unmount.

```typescript
const useFrame = (callback: (state: RabbitState, delta: number) => void) => {
  const state = useContext(RabbitContext)
  const callbackRef = useRef(callback)
  callbackRef.current = callback

  useEffect(() => {
    const wrappedCb = (s: RabbitState, dt: number) => callbackRef.current(s, dt)
    state.frameCallbacks.add(wrappedCb)
    return () => { state.frameCallbacks.delete(wrappedCb) }
  }, [state])
}
```

### useRabbit

Access the engine state from any component:

```typescript
const useRabbit = (): RabbitState => {
  return useContext(RabbitContext)
}
```

### useLoader

Async asset loading with Suspense support:

```typescript
const useLoader = <T>(
  loader: { load: (url: string, ...args: any[]) => Promise<T> },
  url: string,
  ...args: any[]
): T => {
  const state = useContext(RabbitContext)

  // Suspense-compatible: throw a promise to suspend
  const cacheKey = `${loader.constructor.name}:${url}`
  const cache = state._loaderCache

  if (cache.has(cacheKey)) {
    const entry = cache.get(cacheKey)!
    if (entry.status === 'resolved') return entry.value as T
    if (entry.status === 'rejected') throw entry.error
    throw entry.promise // suspend
  }

  const promise = loader.load(url, ...args).then(
    (value) => { entry.status = 'resolved'; entry.value = value },
    (error) => { entry.status = 'rejected'; entry.error = error },
  )

  const entry = { status: 'pending' as const, promise, value: null, error: null }
  cache.set(cacheKey, entry)
  throw promise // suspend
}
```

### useGLTF

Convenience wrapper for glTF loading:

```typescript
const useGLTF = (url: string): GLTFResult => {
  const { engine } = useRabbit()
  return useLoader(engine.gltfLoader, url)
}
```

## Components

### Primitive

Mount a pre-built scene graph (e.g., from glTF) into the React tree:

```tsx
const Primitive: React.FC<{ object: Node } & NodeProps> = ({ object, ...props }) => {
  const groupRef = useRef<Group>(null)

  useEffect(() => {
    if (groupRef.current) {
      groupRef.current.add(object)
      return () => { groupRef.current!.remove(object) }
    }
  }, [object])

  return <group ref={groupRef} {...props} />
}
```

### Html

Render DOM content at a 3D position:

```tsx
interface HtmlProps {
  children: React.ReactNode
  position?: [number, number, number]
  center?: boolean
  occlude?: boolean
  style?: React.CSSProperties
  className?: string
  pointerEvents?: boolean
}

const Html: React.FC<HtmlProps> = ({
  children,
  position = [0, 0, 0],
  center = true,
  occlude = false,
  pointerEvents = false,
  style,
  className,
}) => {
  const { overlay } = useRabbit()
  const idRef = useRef(nextId++)
  const containerRef = useRef<HTMLDivElement | null>(null)
  const portalRef = useRef<HTMLDivElement | null>(null)

  useEffect(() => {
    const container = document.createElement('div')
    container.className = className ?? ''
    Object.assign(container.style, style)
    containerRef.current = container

    overlay.add(idRef.current, container, new Vec3(...position), {
      center,
      occlude,
      pointerEvents,
    })

    return () => {
      overlay.remove(idRef.current)
    }
  }, [])

  // Update position when prop changes
  useEffect(() => {
    const entry = overlay.getEntry(idRef.current)
    if (entry) {
      entry.worldPosition.set(...position)
    }
  }, [position[0], position[1], position[2]])

  // Render children into the overlay container via portal
  return containerRef.current
    ? ReactDOM.createPortal(children, containerRef.current)
    : null
}
```

### OrbitControls Component

```tsx
const OrbitControls: React.FC<OrbitControlsConfig> = (props) => {
  const { camera, canvas } = useRabbit()
  const controlsRef = useRef<OrbitControlsImpl | null>(null)

  useEffect(() => {
    controlsRef.current = new OrbitControlsImpl(camera, canvas, props)
    return () => { controlsRef.current!.dispose() }
  }, [camera, canvas])

  useFrame((_, dt) => {
    controlsRef.current?.update(dt)
  })

  return null // no DOM output
}
```

## Usage Example

```tsx
import { Canvas, useFrame, useGLTF, Html, OrbitControls } from '@rabbit/react'

const Character = ({ url }: { url: string }) => {
  const gltf = useGLTF(url)
  const meshRef = useRef<Mesh>(null)

  useFrame((_, dt) => {
    if (meshRef.current) {
      meshRef.current.rotation.z += dt * 0.5
    }
  })

  return (
    <group>
      <primitive ref={meshRef} object={gltf.scene} />
      <Html position={[0, 0, 2.5]} center>
        <div className="label">Player 1</div>
      </Html>
    </group>
  )
}

const App = () => (
  <Canvas
    camera={{ position: [10, 10, 8], fov: 50 }}
    bloom={{ intensity: 0.5, threshold: 1.0 }}
  >
    <ambientLight intensity={0.2} />
    <directionalLight
      position={[20, 20, 30]}
      intensity={1.2}
      castShadow
    />
    <mesh position={[0, 0, 0]}>
      <boxGeometry width={2} height={2} depth={2} />
      <lambertMaterial
        color={[0.8, 0.2, 0.1]}
        materialIndexEntries={[
          { color: [1, 1, 1] },
          { color: [0, 0.8, 0.7], emissive: [0, 0.8, 0.7], emissiveIntensity: 0.7 },
        ]}
      />
    </mesh>
    <Suspense fallback={null}>
      <Character url="/models/character.glb" />
    </Suspense>
    <OrbitControls />
  </Canvas>
)
```

## Type Declarations

For TypeScript JSX support, we extend the JSX namespace:

```typescript
declare global {
  namespace JSX {
    interface IntrinsicElements {
      group: NodeProps
      mesh: NodeProps & { geometry?: Geometry, material?: Material }
      skinnedMesh: NodeProps
      bone: NodeProps & { boneIndex?: number }
      ambientLight: NodeProps & { intensity?: number, color?: [number, number, number] }
      directionalLight: NodeProps & DirectionalLightProps
      boxGeometry: BoxGeometryProps
      sphereGeometry: SphereGeometryProps
      planeGeometry: PlaneGeometryProps
      cylinderGeometry: CylinderGeometryProps
      coneGeometry: ConeGeometryProps
      capsuleGeometry: CapsuleGeometryProps
      circleGeometry: CircleGeometryProps
      basicMaterial: BasicMaterialProps
      lambertMaterial: LambertMaterialProps
    }
  }
}

interface NodeProps {
  ref?: React.Ref<Node>
  position?: [number, number, number]
  rotation?: [number, number, number]
  scale?: number | [number, number, number]
  visible?: boolean
  name?: string
  children?: React.ReactNode
}
```
