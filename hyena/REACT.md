# React Bindings

## Overview

`@hyena/react` provides React bindings similar to React Three Fiber (R3F). It uses a custom `react-reconciler` to map JSX elements directly to Hyena scene objects, enabling declarative 3D scene construction with React's component model.

## Core Architecture

```
React Tree                          Hyena Scene Graph
──────────                          ──────────────────
<Canvas>                            Engine + Scene
  <group position={[0,0,1]}>       → Group (pos: 0,0,1)
    <mesh>                          → Mesh
      <boxGeometry />               →   BoxGeometry
      <lambertMaterial />           →   LambertMaterial
    </mesh>
    <mesh>                          → Mesh
      <sphereGeometry />            →   SphereGeometry
      <basicMaterial />             →   BasicMaterial
    </mesh>
  </group>
  <ambientLight />                  → AmbientLight
  <directionalLight />              → DirectionalLight
</Canvas>
```

## Reconciler

The reconciler implements `react-reconciler`'s host config interface to bridge React's virtual DOM to Hyena's scene graph:

```typescript
import Reconciler from 'react-reconciler'

const reconciler = Reconciler({
  // Create a Hyena object from a JSX element type
  createInstance(type: string, props: any) {
    switch (type) {
      case 'mesh': return new Mesh()
      case 'group': return new Group()
      case 'boxGeometry': return new BoxGeometry(props)
      case 'sphereGeometry': return new SphereGeometry(props)
      case 'lambertMaterial': return new LambertMaterial(props)
      case 'basicMaterial': return new BasicMaterial(props)
      case 'ambientLight': return new AmbientLight(props)
      case 'directionalLight': return new DirectionalLight(props)
      case 'skinnedMesh': return new SkinnedMesh()
      // ... etc
    }
  },

  // Attach child to parent in the scene graph
  appendChild(parent: Object3D, child: any) {
    if (child instanceof Geometry) {
      (parent as Mesh).geometry = child
    } else if (child instanceof Material) {
      (parent as Mesh).material = child
    } else if (child instanceof Object3D) {
      parent.add(child)
    }
  },

  // Remove child from parent
  removeChild(parent: Object3D, child: any) {
    if (child instanceof Object3D) {
      parent.remove(child)
    }
  },

  // Update props on an existing instance
  commitUpdate(instance: any, updatePayload: any, type: string, oldProps: any, newProps: any) {
    applyProps(instance, newProps)
  },

  // Cleanup on unmount
  removeChildFromContainer(container: Scene, child: any) {
    if (child instanceof Object3D) {
      container.remove(child)
      child.dispose()
    }
  },

  // ... other host config methods
  supportsMutation: true,
  isPrimaryRenderer: false,
})
```

### Property Application

Props are applied to Hyena objects via a generic `applyProps` function:

```typescript
const applyProps = (instance: any, props: Record<string, any>) => {
  for (const [key, value] of Object.entries(props)) {
    if (key === 'children') continue
    if (key === 'ref') continue

    // Handle shorthand array props → Vec3/Quat
    if (key === 'position' && Array.isArray(value)) {
      instance.position.set(value[0], value[1], value[2])
    } else if (key === 'scale' && Array.isArray(value)) {
      instance.scale.set(value[0], value[1], value[2])
    } else if (key === 'rotation' && Array.isArray(value)) {
      // Euler angles [x, y, z] → quaternion
      instance.rotation.setFromEuler(value[0], value[1], value[2])
    } else if (key.startsWith('on')) {
      // Event handlers (onClick, onPointerOver, etc.)
      instance[key] = value
    } else {
      instance[key] = value
    }
  }
}
```

## `<Canvas>` Component

The root component that creates the Hyena engine and provides context:

```typescript
interface CanvasProps {
  children: React.ReactNode
  style?: React.CSSProperties
  className?: string
  camera?: { position?: [number, number, number], fov?: number, near?: number, far?: number }
  shadows?: boolean | ShadowOptions
  bloom?: boolean | BloomOptions
  backgroundColor?: [number, number, number]
  onCreated?: (state: EngineState) => void
}

const Canvas: React.FC<CanvasProps> = ({ children, ...props }) => {
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const [engine, setEngine] = useState<Engine | null>(null)

  useEffect(() => {
    const init = async () => {
      const engine = await Engine.create({
        canvas: canvasRef.current!,
        shadows: props.shadows,
        bloom: props.bloom,
      })
      setEngine(engine)
      props.onCreated?.({ engine, scene: engine.scene, camera: engine.camera })
    }
    init()
    return () => engine?.destroy()
  }, [])

  return (
    <div style={{ position: 'relative', width: '100%', height: '100%', ...props.style }}>
      <canvas ref={canvasRef} style={{ width: '100%', height: '100%' }} />
      {engine && (
        <engineContext.Provider value={engine}>
          <ReconcilerRoot engine={engine}>
            {children}
          </ReconcilerRoot>
        </engineContext.Provider>
      )}
    </div>
  )
}
```

## Hooks

### `useFrame(callback, priority?)`

Register a callback to run every frame. Callbacks are sorted by priority (lower = earlier).

```typescript
const useFrame = (callback: (state: FrameState, delta: number) => void, priority = 0) => {
  const engine = useContext(engineContext)

  useEffect(() => {
    const entry = { callback, priority }
    engine.addFrameCallback(entry)
    return () => engine.removeFrameCallback(entry)
  }, [callback, priority])
}

// Usage
const Spinner = () => {
  const ref = useRef<Mesh>(null)
  useFrame((_, dt) => {
    ref.current!.rotation.z += dt * 2
  })
  return (
    <mesh ref={ref}>
      <boxGeometry width={1} height={1} depth={1} />
      <lambertMaterial color={[1, 0.5, 0]} />
    </mesh>
  )
}
```

### `useEngine()`

Access the Engine instance:

```typescript
const useEngine = () => useContext(engineContext)
```

### `useScene()`

Access the Scene:

```typescript
const useScene = () => useContext(engineContext).scene
```

### `useLoader(loaderClass, url)`

Load an asset with Suspense support:

```typescript
const useLoader = <T>(
  LoaderClass: new () => { load: (url: string) => Promise<T> },
  url: string
): T => {
  const cached = loaderCache.get(url)
  if (cached) return cached as T
  throw loadAsset(LoaderClass, url)  // throw Promise for Suspense
}

// Usage
const Model = ({ url }: { url: string }) => {
  const gltf = useLoader(GLTFLoader, url)
  return <primitive object={gltf.scene} />
}

// With Suspense
const App = () => (
  <Canvas>
    <Suspense fallback={null}>
      <Model url="/models/character.glb" />
    </Suspense>
  </Canvas>
)
```

### `useRaycast()`

Raycasting from pointer events:

```typescript
const useRaycast = () => {
  const engine = useEngine()
  return useCallback((event: PointerEvent) => {
    const rect = engine.canvas.getBoundingClientRect()
    const x = ((event.clientX - rect.left) / rect.width) * 2 - 1
    const y = -((event.clientY - rect.top) / rect.height) * 2 + 1
    const ray = engine.camera.screenToRay(x, y)
    return engine.scene.raycast(ray)
  }, [engine])
}
```

## Pointer Events

Meshes can receive pointer events when they have event handlers:

```tsx
<mesh
  onClick={(event) => console.log('clicked', event.point)}
  onPointerOver={(event) => console.log('hover')}
  onPointerOut={() => console.log('unhover')}
>
  <boxGeometry width={1} height={1} depth={1} />
  <lambertMaterial color={[1, 0, 0]} />
</mesh>
```

The engine performs raycasting on pointer events and dispatches to the hit mesh. Only meshes with event handlers are included in the raycast test (performance optimization).

## `<primitive>` Element

For inserting pre-built Hyena objects (like GLTF scenes) directly:

```tsx
const gltf = useLoader(GLTFLoader, '/models/scene.glb')
return <primitive object={gltf.scene} position={[0, 0, 0]} />
```

The reconciler attaches the existing Object3D to the parent without creating a new instance.

## HTML Overlay Integration

```tsx
import { Html } from '@hyena/react'

const HealthBar = ({ mesh }: { mesh: Mesh }) => (
  <Html position={[0, 0, 2]} occlude>
    <div className="health-bar">
      <div className="fill" style={{ width: '75%' }} />
    </div>
  </Html>
)
```

See [HTML-OVERLAY.md](./HTML-OVERLAY.md) for the overlay system details.

## Automatic Disposal

When a component unmounts, its Hyena resources are automatically disposed:

```
Component unmounts
  → reconciler.removeChild()
    → parent.remove(child)
    → child.traverse(obj => {
        if (obj is Mesh):
          obj.geometry.dispose()  // if not shared
          obj.material.dispose()  // if not shared
      })
```

Shared resources (geometries, materials, textures used by multiple components) use reference counting and are only disposed when the last reference is removed.

## OrbitControls Integration

```tsx
import { OrbitControls } from '@hyena/react'

const App = () => (
  <Canvas>
    <OrbitControls damping={0.1} minDistance={2} maxDistance={50} />
    <mesh>
      <boxGeometry />
      <lambertMaterial />
    </mesh>
  </Canvas>
)
```

## TypeScript Support

All JSX elements are fully typed:

```typescript
declare global {
  namespace JSX {
    interface IntrinsicElements {
      mesh: MeshProps
      group: GroupProps
      boxGeometry: BoxGeometryProps
      sphereGeometry: SphereGeometryProps
      // ... etc
      lambertMaterial: LambertMaterialProps
      basicMaterial: BasicMaterialProps
      ambientLight: AmbientLightProps
      directionalLight: DirectionalLightProps
    }
  }
}
```

This provides autocomplete and type checking for all 3D element props in JSX.
