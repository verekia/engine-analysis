# React Bindings — fennec-react

## Overview

`fennec-react` provides React bindings for Fennec, following the patterns established by React Three Fiber (R3F). A custom React reconciler manages the Fennec scene graph, allowing developers to describe 3D scenes declaratively with JSX. The bindings include hooks for animation loops, asset loading, and engine access.

## Architecture

```
React Component Tree                 Fennec Scene Graph
─────────────────────                ───────────────────
<Canvas>                      →      Engine + Scene
  <ambientLight>              →        AmbientLight
  <directionalLight>          →        DirectionalLight
  <group>                     →        Group
    <mesh>                    →          Mesh
      <boxGeometry>           →            BoxGeometry
      <lambertMaterial>       →            LambertMaterial
    </mesh>
  </group>
</Canvas>
```

The reconciler translates React's create/update/delete operations into Fennec scene graph mutations (add, remove, set property).

## Canvas Component

The root component that creates the engine, canvas, and render loop:

```typescript
interface CanvasProps {
  children: React.ReactNode
  style?: React.CSSProperties
  className?: string

  // Engine options
  antialias?: boolean           // MSAA (default: true)
  sampleCount?: 1 | 2 | 4      // MSAA samples (default: 4)
  shadows?: boolean | ShadowConfig
  background?: number           // Clear color (default: 0x000000)
  dpr?: number | [number, number]  // Device pixel ratio (default: [1, 2])
  preferWebGPU?: boolean        // Default: true

  // Render loop
  frameloop?: 'always' | 'demand'  // Default: 'always'

  // Callbacks
  onCreated?: (state: FennecState) => void
}

const Canvas: React.FC<CanvasProps> = ({ children, ...props }) => {
  const containerRef = useRef<HTMLDivElement>(null)
  const stateRef = useRef<FennecState | null>(null)

  useEffect(() => {
    const container = containerRef.current!
    const canvas = document.createElement('canvas')
    container.appendChild(canvas)

    // Initialize engine
    createEngine({ canvas, ...props }).then((engine) => {
      const state: FennecState = {
        engine,
        scene: new Scene(),
        camera: new PerspectiveCamera(),
        // ...
      }
      stateRef.current = state
      props.onCreated?.(state)

      // Start render loop
      startRenderLoop(state)
    })

    return () => {
      stateRef.current?.engine.dispose()
    }
  }, [])

  return (
    <div ref={containerRef} style={{ position: 'relative', width: '100%', height: '100%', ...props.style }}>
      {stateRef.current && (
        <FennecContext.Provider value={stateRef.current}>
          <ReconcilerRoot state={stateRef.current}>
            {children}
          </ReconcilerRoot>
        </FennecContext.Provider>
      )}
    </div>
  )
}
```

## Custom Reconciler

The reconciler uses `react-reconciler` to bridge React and Fennec:

```typescript
import Reconciler from 'react-reconciler'

const reconciler = Reconciler({
  supportsMutation: true,
  supportsPersistence: false,

  createInstance(type: string, props: Record<string, unknown>) {
    // Map JSX element type to Fennec object
    return createFennecElement(type, props)
  },

  appendChild(parent: SceneNode, child: SceneNode) {
    parent.add(child)
  },

  removeChild(parent: SceneNode, child: SceneNode) {
    parent.remove(child)
    child.dispose?.()
  },

  commitUpdate(instance: SceneNode, _updatePayload, _type, oldProps, newProps) {
    // Diff props and apply changes
    applyProps(instance, newProps, oldProps)
  },

  // ... other reconciler methods
})
```

### Element Type Mapping

```typescript
const ELEMENT_MAP: Record<string, () => SceneNode> = {
  // Scene objects
  'group': () => new Group(),
  'mesh': () => new Mesh(),
  'skinnedMesh': () => new SkinnedMesh(),

  // Geometries
  'boxGeometry': (props) => new BoxGeometry(props.args),
  'sphereGeometry': (props) => new SphereGeometry(props.args),
  'planeGeometry': (props) => new PlaneGeometry(props.args),
  'coneGeometry': (props) => new ConeGeometry(props.args),
  'cylinderGeometry': (props) => new CylinderGeometry(props.args),
  'capsuleGeometry': (props) => new CapsuleGeometry(props.args),
  'circleGeometry': (props) => new CircleGeometry(props.args),

  // Materials
  'basicMaterial': (props) => new BasicMaterial(props),
  'lambertMaterial': (props) => new LambertMaterial(props),

  // Lights
  'ambientLight': (props) => new AmbientLight(props),
  'directionalLight': (props) => new DirectionalLight(props),

  // Camera
  'perspectiveCamera': (props) => new PerspectiveCamera(props),
}
```

### Property Application

Props are applied to Fennec objects using a smart diffing system:

```typescript
const applyProps = (instance: any, newProps: Record<string, any>, oldProps: Record<string, any> = {}) => {
  for (const [key, value] of Object.entries(newProps)) {
    if (key === 'children' || key === 'args' || key === 'ref') continue
    if (value === oldProps[key]) continue

    // Handle nested properties: position={[1, 2, 3]} → instance.position.set(1, 2, 3)
    if (key === 'position' || key === 'scale') {
      if (Array.isArray(value)) {
        instance[key][0] = value[0]
        instance[key][1] = value[1]
        instance[key][2] = value[2]
        instance.markDirty()
      }
    } else if (key === 'rotation') {
      // Expect euler angles [x, y, z] or quaternion [x, y, z, w]
      if (value.length === 3) {
        quat.fromEuler(instance.rotation, value[0], value[1], value[2])
      } else {
        quat.set(instance.rotation, value[0], value[1], value[2], value[3])
      }
      instance.markDirty()
    } else {
      // Direct property set
      instance[key] = value
    }
  }
}
```

## Hooks

### useFrame

Register a callback that runs every frame. Similar to R3F's `useFrame`:

```typescript
type FrameCallback = (state: FennecState, delta: number) => void

const useFrame = (callback: FrameCallback, priority?: number) => {
  const state = useContext(FennecContext)
  const callbackRef = useRef(callback)
  callbackRef.current = callback

  useEffect(() => {
    const sub = state.subscribe((s, delta) => callbackRef.current(s, delta), priority)
    return () => sub.unsubscribe()
  }, [state, priority])
}
```

Usage:

```tsx
const RotatingBox = () => {
  const meshRef = useRef<Mesh>(null)

  useFrame((_, delta) => {
    meshRef.current!.rotation[2] += delta // Rotate around Z
    meshRef.current!.markDirty()
  })

  return (
    <mesh ref={meshRef}>
      <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
      <lambertMaterial color={0x44aa88} />
    </mesh>
  )
}
```

### useEngine

Access the engine state (camera, scene, engine instance, etc.):

```typescript
const useEngine = (): FennecState => {
  return useContext(FennecContext)
}

interface FennecState {
  engine: Engine
  scene: Scene
  camera: Camera
  canvas: HTMLCanvasElement
  viewport: { width: number, height: number }
  pointer: { x: number, y: number }  // Normalized device coords (-1 to 1)
  clock: { elapsed: number, delta: number }
}
```

### useLoader

Generic async loader hook with Suspense integration:

```typescript
const useLoader = <T,>(
  loader: (url: string, options?: any) => Promise<T>,
  url: string,
  options?: any,
): T => {
  const [resource] = useState(() => {
    // Use React Suspense protocol
    let result: T | undefined
    let error: Error | undefined
    let promise: Promise<void> | undefined

    const load = () => {
      promise = loader(url, options)
        .then(r => { result = r })
        .catch(e => { error = e })
    }
    load()

    return {
      read() {
        if (error) throw error
        if (result !== undefined) return result
        throw promise // Suspense will catch this
      }
    }
  })

  return resource.read()
}
```

### useGLTF

Convenience hook for loading GLTF models:

```typescript
const useGLTF = (url: string, options?: GLTFLoadOptions): GLTFResult => {
  return useLoader(loadGLTF, url, options)
}

// Usage with Suspense
const Character = () => {
  const { scene, animations } = useGLTF('/character.glb', { draco: true })

  return <primitive object={scene} />
}

const App = () => (
  <Canvas>
    <Suspense fallback={null}>
      <Character />
    </Suspense>
  </Canvas>
)
```

### useAnimations

Hook for managing skeletal animations:

```typescript
const useAnimations = (clips: AnimationClip[], skeleton: Skeleton) => {
  const mixer = useMemo(() => new AnimationMixer(skeleton), [skeleton])
  const actions = useMemo(() => {
    const map: Record<string, AnimationAction> = {}
    for (const clip of clips) {
      map[clip.name] = mixer.createAction(clip)
    }
    return map
  }, [clips, mixer])

  // Auto-update mixer in render loop
  useFrame((_, delta) => {
    mixer.update(delta)
  })

  return { mixer, actions }
}

// Usage
const Character = () => {
  const { scene, animations, skeletons } = useGLTF('/character.glb')
  const { actions } = useAnimations(animations, skeletons[0])

  useEffect(() => {
    actions.idle?.play()
  }, [actions])

  return <primitive object={scene} />
}
```

## Special Components

### `<primitive>`

Injects a pre-existing Fennec object into the scene graph (bypass reconciler creation):

```tsx
const Model = () => {
  const { scene } = useGLTF('/model.glb')
  return <primitive object={scene} position={[0, 0, 0]} />
}
```

### `<Html>`

Renders DOM elements at 3D positions using the overlay system:

```tsx
interface HtmlProps {
  children: React.ReactNode
  position?: [number, number, number]   // World position
  center?: boolean                       // Center the element
  occlude?: boolean                      // Hide behind geometry
  style?: React.CSSProperties
  className?: string
  as?: string                            // Container element type (default: 'div')
  portal?: React.RefObject<HTMLElement>  // Render into a different container
}

const Html: React.FC<HtmlProps> = ({ children, position, ...props }) => {
  const state = useEngine()
  const groupRef = useRef<Group>(null)
  const overlayRef = useRef<OverlayItem | null>(null)

  useEffect(() => {
    const el = document.createElement(props.as ?? 'div')
    el.style.position = 'absolute'
    el.style.left = '0'
    el.style.top = '0'
    if (props.center) {
      el.style.transform = 'translate(-50%, -50%)'
    }

    const overlayItem = state.overlay.add({
      worldPosition: groupRef.current!.getWorldPosition(),
      element: el,
      occlude: props.occlude,
    })
    overlayRef.current = overlayItem

    return () => {
      state.overlay.remove(overlayItem)
    }
  }, [])

  // Update world position each frame
  useFrame(() => {
    if (overlayRef.current && groupRef.current) {
      vec3.copy(overlayRef.current.worldPosition, groupRef.current.getWorldPosition())
    }
  })

  // Render React children into the overlay DOM element via portal
  return (
    <>
      <group ref={groupRef} position={position} />
      {overlayRef.current &&
        ReactDOM.createPortal(children, overlayRef.current.element)}
    </>
  )
}
```

Usage:

```tsx
const HealthBar = ({ hp, maxHp }) => (
  <Html position={[0, 0, 2.5]} center>
    <div className="health-bar">
      <div className="health-fill" style={{ width: `${(hp / maxHp) * 100}%` }} />
    </div>
  </Html>
)
```

### `<orbitControls>`

Declarative orbit controls:

```tsx
<Canvas>
  <orbitControls
    target={[0, 0, 0]}
    maxDistance={100}
    minPolarAngle={0.1}
    maxPolarAngle={Math.PI / 2}
    enableDamping
  />
  {/* scene content */}
</Canvas>
```

## Render Loop Integration

The render loop is managed by the `<Canvas>` component and calls registered `useFrame` callbacks in priority order:

```typescript
const startRenderLoop = (state: FennecState) => {
  let lastTime = performance.now()

  const loop = (time: number) => {
    const delta = (time - lastTime) / 1000
    lastTime = time

    state.clock.elapsed += delta
    state.clock.delta = delta

    // Call useFrame subscribers (sorted by priority)
    for (const sub of state._frameSubscribers) {
      sub.callback(state, delta)
    }

    // Render
    state.engine.render(state.scene, state.camera)

    // Update overlays
    state.overlay?.update(state.camera, state.viewport.width, state.viewport.height)

    state._rafId = requestAnimationFrame(loop)
  }

  state._rafId = requestAnimationFrame(loop)
}
```

## Events

Pointer events on meshes via raycasting:

```tsx
<mesh
  onClick={(event) => console.log('clicked', event.point)}
  onPointerOver={(event) => console.log('hovered')}
  onPointerOut={() => console.log('unhovered')}
>
  <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
  <lambertMaterial color={0x44aa88} />
</mesh>
```

The event system performs a single raycast per pointer event against all meshes with event handlers. This uses the BVH for efficient hit testing.

## TypeScript Types

Full type definitions for all JSX elements:

```typescript
declare global {
  namespace JSX {
    interface IntrinsicElements {
      group: GroupProps
      mesh: MeshProps
      skinnedMesh: SkinnedMeshProps
      boxGeometry: { args?: BoxGeometryOptions }
      sphereGeometry: { args?: SphereGeometryOptions }
      planeGeometry: { args?: PlaneGeometryOptions }
      coneGeometry: { args?: ConeGeometryOptions }
      cylinderGeometry: { args?: CylinderGeometryOptions }
      capsuleGeometry: { args?: CapsuleGeometryOptions }
      circleGeometry: { args?: CircleGeometryOptions }
      basicMaterial: BasicMaterialOptions
      lambertMaterial: LambertMaterialOptions
      ambientLight: AmbientLightProps
      directionalLight: DirectionalLightProps
      perspectiveCamera: CameraProps
      orbitControls: OrbitControlsOptions
      primitive: { object: SceneNode } & Partial<SceneNodeProps>
    }
  }
}
```
