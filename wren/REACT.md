# 14 — React Bindings

Wren's React bindings provide a declarative API similar to React Three Fiber (R3F), built on a custom `react-reconciler` host config that maps JSX elements directly to Wren's scene graph.

## Architecture

```
                React Tree                         Wren Scene Graph
                ──────────                         ────────────────
                <WrenCanvas>                       Renderer + Scene
                  <ambientLight>          →          AmbientLight
                  <directionalLight>      →          DirectionalLightNode
                  <group>                 →          GroupNode
                    <mesh>                →            MeshNode
                      <boxGeometry>       →              geometry = createBoxGeometry()
                      <lambertMaterial>   →              material = createLambertMaterial()
                    <mesh>                →            MeshNode
                      ...
```

The reconciler translates React tree operations (create, update, delete, reorder) into imperative Wren scene graph mutations.

## Package Structure

```ts
// wren-react/index.ts
export { WrenCanvas } from './canvas'
export { useFrame, useWren, useLoader, useOverlay } from './hooks'
export type { WrenCanvasProps } from './canvas'
// All Wren scene types are available as JSX intrinsic elements
```

## Custom Reconciler

### Host Config

```ts
import ReactReconciler from 'react-reconciler'

const hostConfig: ReactReconciler.HostConfig<
  string,           // Type (element name)
  Props,            // Props
  WrenInstance,     // Container (scene root)
  WrenInstance,     // Instance
  never,            // TextInstance (not supported)
  never,            // SuspenseInstance
  never,            // HydratableInstance
  WrenInstance,     // PublicInstance
  object,           // HostContext
  UpdatePayload,    // UpdatePayload
  never,            // ChildSet
  number,           // TimeoutHandle
  number            // NoTimeout
> = {
  supportsMutation: true,
  isPrimaryRenderer: false,

  createInstance(type, props, rootContainer) {
    return createElement(type, props, rootContainer)
  },

  appendInitialChild(parent, child) {
    appendToParent(parent, child)
  },

  appendChild(parent, child) {
    appendToParent(parent, child)
  },

  removeChild(parent, child) {
    removeFromParent(parent, child)
    disposeInstance(child)
  },

  insertBefore(parent, child, beforeChild) {
    insertChildBefore(parent, child, beforeChild)
  },

  commitUpdate(instance, payload, type, oldProps, newProps) {
    applyProps(instance, newProps, oldProps)
  },

  prepareForCommit() {
    return null
  },

  resetAfterCommit(container) {
    // Trigger a re-render of the 3D scene
    container.__wrenNeedsRender = true
  },

  // Text is not supported in the 3D scene
  createTextInstance() {
    throw new Error('Text is not supported in Wren scenes. Wrap text in an <htmlOverlay>.')
  },

  shouldSetTextContent() {
    return false
  },

  getRootHostContext() {
    return {}
  },

  getChildHostContext(parentContext) {
    return parentContext
  },

  finalizeInitialChildren() {
    return false
  },

  prepareUpdate(instance, type, oldProps, newProps) {
    return diffProps(oldProps, newProps)
  },

  getPublicInstance(instance) {
    return instance
  },

  scheduleTimeout: setTimeout,
  cancelTimeout: clearTimeout,
  noTimeout: -1,
  supportsPersistence: false,
  supportsHydration: false,

  getCurrentEventPriority() {
    return 0b0000000000000000000000000010000 // DefaultEventPriority
  },
}

const reconciler = ReactReconciler(hostConfig)
```

### Element Catalog

JSX element types map to Wren constructors:

```ts
const ELEMENT_MAP: Record<string, (props: any) => WrenInstance> = {
  // Scene graph
  group: (props) => createGroupNode(props.name),
  scene: (props) => createScene(),

  // Meshes
  mesh: (props) => createMeshNode(),
  skinnedMesh: (props) => createSkinnedMeshNode(),

  // Geometries (attach to parent mesh)
  planeGeometry: (props) => createPlaneGeometry(props),
  boxGeometry: (props) => createBoxGeometry(props),
  sphereGeometry: (props) => createSphereGeometry(props),
  coneGeometry: (props) => createConeGeometry(props),
  cylinderGeometry: (props) => createCylinderGeometry(props),
  capsuleGeometry: (props) => createCapsuleGeometry(props),
  circleGeometry: (props) => createCircleGeometry(props),

  // Materials (attach to parent mesh)
  basicMaterial: (props) => createBasicMaterial(props),
  lambertMaterial: (props) => createLambertMaterial(props),

  // Lights
  ambientLight: (props) => createAmbientLight(props),
  directionalLight: (props) => createDirectionalLightNode(props),

  // Camera
  perspectiveCamera: (props) => createPerspectiveCamera(props),

  // HTML overlay
  htmlOverlay: (props) => createHtmlOverlayHandle(props),
}
```

### Attach Semantics

Children that are geometries or materials are "attached" to their parent mesh's properties rather than added as scene graph children:

```ts
const appendToParent = (parent: WrenInstance, child: WrenInstance): void => {
  if (child.__wrenType === 'geometry') {
    parent.geometry = child
  } else if (child.__wrenType === 'material') {
    parent.material = child
  } else {
    // Regular scene graph child
    addChild(parent, child)
  }
}
```

This means:
```tsx
<mesh>
  <boxGeometry width={2} height={2} depth={2} />
  <lambertMaterial color={[1, 0, 0]} />
</mesh>
```

Creates a mesh with the box geometry and lambert material attached, not as children in the scene graph.

### Props Diffing and Application

```ts
const applyProps = (instance: WrenInstance, newProps: Props, oldProps: Props): void => {
  for (const key in newProps) {
    if (key === 'children') continue
    if (newProps[key] === oldProps[key]) continue

    switch (key) {
      case 'position':
        setPosition(instance, newProps.position[0], newProps.position[1], newProps.position[2])
        break
      case 'rotation':
        // Accept euler [x,y,z] or quaternion [x,y,z,w]
        if (newProps.rotation.length === 3) {
          setRotationFromEuler(instance, ...newProps.rotation)
        } else {
          setRotation(instance, ...newProps.rotation)
        }
        break
      case 'scale':
        if (typeof newProps.scale === 'number') {
          setScale(instance, newProps.scale, newProps.scale, newProps.scale)
        } else {
          setScale(instance, ...newProps.scale)
        }
        break
      case 'visible':
        instance.visible = newProps.visible
        break
      case 'color':
        if (instance.__wrenType === 'material') {
          instance.color.set(newProps.color)
        }
        break
      // ... other property handlers
      default:
        // Direct property assignment for unrecognized keys
        if (key in instance) {
          (instance as any)[key] = newProps[key]
        }
    }
  }
}
```

## WrenCanvas Component

The root component that creates the renderer, scene, and render loop:

```tsx
interface WrenCanvasProps {
  children: React.ReactNode
  camera?: { fov?: number, near?: number, far?: number, position?: [number, number, number] }
  style?: React.CSSProperties
  className?: string
  onCreated?: (state: WrenState) => void
  shadows?: boolean | DirectionalShadowConfig
  antialias?: boolean
  bloom?: boolean | BloomConfig
  preferBackend?: 'webgpu' | 'webgl2'
}

const WrenCanvas: React.FC<WrenCanvasProps> = (props) => {
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const containerRef = useRef<HTMLDivElement>(null)
  const stateRef = useRef<WrenState>(null)

  useEffect(() => {
    const canvas = canvasRef.current!
    const init = async () => {
      const device = await createDevice(canvas, {
        preferBackend: props.preferBackend,
        antialias: props.antialias ?? true,
      })

      const renderer = createRenderer(device, {
        shadows: props.shadows,
        bloom: props.bloom,
      })

      const scene = createScene()
      const camera = createPerspectiveCamera({
        fov: props.camera?.fov ?? 60,
        near: props.camera?.near ?? 0.1,
        far: props.camera?.far ?? 1000,
      })
      if (props.camera?.position) {
        setPosition(camera, ...props.camera.position)
      }

      const state: WrenState = { device, renderer, scene, camera, canvas }
      stateRef.current = state

      // Mount React tree into scene
      const container = reconciler.createContainer(scene, 0, null, false, null, '', null, null)
      reconciler.updateContainer(
        <wrenContext.Provider value={state}>
          {props.children}
        </wrenContext.Provider>,
        container, null, null
      )

      // Render loop
      const frameCallbacks: Set<FrameCallback> = new Set()
      const loop = () => {
        const dt = clock.getDelta()
        frameCallbacks.forEach(cb => cb(state, dt))
        renderer.render(scene, camera)
        requestAnimationFrame(loop)
      }
      requestAnimationFrame(loop)

      props.onCreated?.(state)
    }

    init()
    return () => stateRef.current?.renderer.dispose()
  }, [])

  return (
    <div ref={containerRef} style={{ position: 'relative', ...props.style }} className={props.className}>
      <canvas ref={canvasRef} style={{ display: 'block', width: '100%', height: '100%' }} />
    </div>
  )
}
```

## Hooks

### useFrame

Register a callback that runs every frame, outside of React's reconciliation cycle:

```ts
const useFrame = (callback: (state: WrenState, delta: number) => void): void => {
  const state = useContext(wrenContext)
  const callbackRef = useRef(callback)
  callbackRef.current = callback

  useEffect(() => {
    const cb = (s: WrenState, dt: number) => callbackRef.current(s, dt)
    state.frameCallbacks.add(cb)
    return () => { state.frameCallbacks.delete(cb) }
  }, [state])
}
```

### useWren

Access the Wren state (device, renderer, scene, camera):

```ts
const useWren = (): WrenState => {
  return useContext(wrenContext)
}
```

### useLoader

Load assets with React Suspense integration:

```ts
const useLoader = <T>(
  loader: (url: string, device: WrenDevice) => Promise<T>,
  url: string
): T => {
  const { device } = useWren()
  // Suspense-compatible: throws promise on first call, returns result on second
  const cached = loaderCache.get(url)
  if (cached) return cached as T
  throw loader(url, device).then(result => {
    loaderCache.set(url, result)
  })
}

// Usage:
const Model = ({ url }: { url: string }) => {
  const gltf = useLoader(loadGLTF, url)
  return <primitive object={gltf.scenes[0]} />
}
```

### useOverlay

Create an HTML overlay element from within a React component:

```ts
const useOverlay = (options: OverlayOptions): RefObject<HTMLDivElement> => {
  const ref = useRef<HTMLDivElement>(null)
  const { overlay } = useWren()

  useEffect(() => {
    if (!ref.current) return
    const handle = overlay.add(ref.current, options)
    return () => overlay.remove(handle)
  }, [overlay, options])

  return ref
}
```

## Usage Example

```tsx
import { WrenCanvas, useFrame, useLoader } from 'wren-react'

const Scene = () => {
  const meshRef = useRef()

  useFrame((state, dt) => {
    meshRef.current.rotation[2] += dt  // Rotate around Z (up)
    markDirty(meshRef.current)
  })

  return (
    <>
      <ambientLight color={[1, 1, 1]} intensity={0.3} />
      <directionalLight
        color={[1, 0.95, 0.9]}
        intensity={1.0}
        position={[10, 10, 20]}
        castShadow
      />
      <mesh ref={meshRef} position={[0, 0, 1]} castShadow>
        <boxGeometry width={2} height={2} depth={2} />
        <lambertMaterial color={[0.2, 0.6, 1.0]} />
      </mesh>
      <mesh position={[0, 0, 0]} receiveShadow>
        <planeGeometry width={20} height={20} />
        <lambertMaterial color={[0.8, 0.8, 0.8]} />
      </mesh>
    </>
  )
}

const App = () => (
  <WrenCanvas
    camera={{ position: [5, 5, 5], fov: 60 }}
    shadows
    bloom={{ intensity: 0.5 }}
  >
    <Scene />
  </WrenCanvas>
)
```

## TypeScript JSX Types

```ts
// wren-react/jsx.d.ts
declare namespace JSX {
  interface IntrinsicElements {
    group: GroupProps
    mesh: MeshProps
    skinnedMesh: SkinnedMeshProps

    planeGeometry: PlaneGeometryProps
    boxGeometry: BoxGeometryProps
    sphereGeometry: SphereGeometryProps
    coneGeometry: ConeGeometryProps
    cylinderGeometry: CylinderGeometryProps
    capsuleGeometry: CapsuleGeometryProps
    circleGeometry: CircleGeometryProps

    basicMaterial: BasicMaterialProps
    lambertMaterial: LambertMaterialProps

    ambientLight: AmbientLightProps
    directionalLight: DirectionalLightProps

    perspectiveCamera: PerspectiveCameraProps

    htmlOverlay: HtmlOverlayProps

    primitive: { object: SceneNode }
  }
}
```

## Event System

Pointer events on 3D objects are implemented via raycasting:

```tsx
<mesh
  onClick={(event: WrenPointerEvent) => {
    console.log('Clicked at:', event.point)
  }}
  onPointerOver={(event) => {
    document.body.style.cursor = 'pointer'
  }}
  onPointerOut={() => {
    document.body.style.cursor = 'default'
  }}
>
  <boxGeometry />
  <lambertMaterial color={[1, 0, 0]} />
</mesh>
```

The canvas listens for DOM pointer events, unprojects the mouse position into a ray, raycasts against all meshes with event handlers, and dispatches synthetic events to the closest hit.
