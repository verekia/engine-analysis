# React Bindings

## Overview

Caracal provides React bindings inspired by React Three Fiber (R3F), enabling declarative 3D scene construction with React components. The bindings use a custom React reconciler to map JSX elements directly to Caracal scene graph nodes.

## Architecture

```
React Component Tree              Caracal Scene Graph
─────────────────────             ─────────────────────

<Canvas>                          Engine
  <scene>                            Scene
    <ambientLight />                   AmbientLight
    <directionalLight />               DirectionalLight
    <group>                            Group
      <mesh>                             Mesh
        <boxGeometry />                    geometry: BoxGeometry
        <lambertMaterial />                material: LambertMaterial
      </mesh>
    </group>
  </scene>
</Canvas>

React Reconciler translates React operations
(create, update, delete) into scene graph mutations
```

## Custom Reconciler

The reconciler is built with `react-reconciler` and maps React lifecycle events to Caracal scene graph operations:

```typescript
import Reconciler from 'react-reconciler'

const reconciler = Reconciler({
  // Create a Caracal node from a JSX element type
  createInstance(type: string, props: any) {
    switch (type) {
      case 'mesh': return createMesh(null, null) // geometry/material set via children
      case 'group': return createGroup()
      case 'directionalLight': return createDirectionalLight(props)
      case 'ambientLight': return createAmbientLight(props)
      case 'perspectiveCamera': return createPerspectiveCamera(props)
      // Geometry and material are handled as "instances" attached to parent mesh
      default: throw new Error(`Unknown element type: ${type}`)
    }
  },

  // Append child to parent in the scene graph
  appendChild(parent: Node, child: Node | GeometryOrMaterial) {
    if (isGeometry(child)) {
      (parent as Mesh).geometry = child
    } else if (isMaterial(child)) {
      (parent as Mesh).material = child
    } else {
      parent.add(child)
    }
  },

  // Remove child from parent
  removeChild(parent: Node, child: Node | GeometryOrMaterial) {
    if (isGeometry(child)) {
      (parent as Mesh).geometry = null
    } else if (isMaterial(child)) {
      (parent as Mesh).material = null
    } else {
      parent.remove(child)
    }
  },

  // Update props on an existing node
  commitUpdate(instance: Node, updatePayload: any, type: string, oldProps: any, newProps: any) {
    applyProps(instance, newProps, oldProps)
  },

  // ... other reconciler methods (prepareUpdate, finalizeInitialChildren, etc.)
})
```

### Prop Application

Props are applied directly to Caracal node properties. The reconciler diffs props and applies only changes:

```typescript
const applyProps = (instance: any, newProps: Record<string, any>, oldProps: Record<string, any>) => {
  for (const [key, value] of Object.entries(newProps)) {
    if (key === 'children') continue
    if (value === oldProps[key]) continue

    // Handle nested properties: position={[1, 2, 3]} → node.position.set(1, 2, 3)
    if (key === 'position' && Array.isArray(value)) {
      instance.position.set(value[0], value[1], value[2])
    } else if (key === 'rotation' && Array.isArray(value)) {
      instance.rotation.setFromEuler(value[0], value[1], value[2])
    } else if (key === 'scale' && Array.isArray(value)) {
      if (typeof value === 'number') {
        instance.scale.set(value, value, value)
      } else {
        instance.scale.set(value[0], value[1], value[2])
      }
    } else {
      // Direct property assignment
      instance[key] = value
    }
  }
}
```

## `<Canvas>` Component

The root component that creates the engine and provides context:

```tsx
interface CanvasProps {
  children: React.ReactNode
  style?: React.CSSProperties
  className?: string
  backend?: 'auto' | 'webgl2' | 'webgpu'
  antialias?: boolean           // default: true
  pixelRatio?: number           // default: devicePixelRatio
  shadows?: boolean             // default: false
  bloom?: boolean | BloomConfig // default: false
  camera?: CameraConfig         // default: perspective, position [0, -10, 5]
  onCreated?: (state: EngineState) => void
}

const Canvas = ({ children, ...props }: CanvasProps) => {
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const [engine, setEngine] = useState<Engine | null>(null)

  useEffect(() => {
    const canvas = canvasRef.current!
    const eng = createEngine({ canvas, ...props })
    setEngine(eng)

    return () => eng.dispose()
  }, [])

  return (
    <div style={{ position: 'relative', width: '100%', height: '100%', ...props.style }}>
      <canvas ref={canvasRef} style={{ width: '100%', height: '100%' }} />
      {engine && (
        <EngineContext.Provider value={engine}>
          <SceneContainer engine={engine}>
            {children}
          </SceneContainer>
        </EngineContext.Provider>
      )}
    </div>
  )
}
```

## Scene Elements

All Caracal node types are available as lowercase JSX elements:

### Containers

```tsx
// Group
<group position={[0, 0, 0]} rotation={[0, 0, Math.PI / 4]}>
  {children}
</group>
```

### Meshes

```tsx
// Mesh with geometry and material as children
<mesh position={[0, 0, 1]} castShadow receiveShadow>
  <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
  <lambertMaterial color={[1, 0.5, 0]} />
</mesh>

// Mesh with pre-created geometry/material (refs)
<mesh geometry={myGeometry} material={myMaterial} />
```

### Geometries

```tsx
<boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
<sphereGeometry args={{ radius: 0.5, widthSegments: 32, heightSegments: 16 }} />
<planeGeometry args={{ width: 10, height: 10 }} />
<cylinderGeometry args={{ radiusTop: 0.5, radiusBottom: 0.5, height: 2 }} />
<coneGeometry args={{ radius: 0.5, height: 1 }} />
<capsuleGeometry args={{ radius: 0.3, height: 1.5 }} />
<circleGeometry args={{ radius: 1, segments: 32 }} />
```

### Materials

```tsx
<basicMaterial color={[1, 1, 1]} map={texture} transparent opacity={0.5} />
<lambertMaterial
  color={[0.8, 0.8, 0.8]}
  map={texture}
  aoMap={aoTexture}
  emissive={[1, 0.5, 0]}
  emissiveIntensity={0.7}
  materialIndexColors={indexMap}
/>
```

### Lights

```tsx
<ambientLight color={[0.3, 0.35, 0.4]} intensity={1} />
<directionalLight
  color={[1, 0.95, 0.8]}
  intensity={1}
  position={[10, 10, 20]}
  castShadow
  shadowMapSize={2048}
/>
```

### Camera

```tsx
<perspectiveCamera
  position={[0, -10, 5]}
  fov={60}
  near={0.1}
  far={500}
  makeDefault  // Makes this the active camera
/>
```

## Hooks

### `useFrame`

Register a callback that runs every frame before rendering:

```typescript
const useFrame = (callback: (state: FrameState, delta: number) => void) => {
  // Registers callback in the engine's frame loop
  // Automatically unregisters on component unmount
}

interface FrameState {
  engine: Engine
  scene: Scene
  camera: Camera
  clock: { elapsed: number, delta: number }
  pointer: { x: number, y: number }  // Normalized [-1, 1]
}
```

Usage:

```tsx
const SpinningBox = () => {
  const meshRef = useRef<Mesh>(null)

  useFrame((state, delta) => {
    meshRef.current!.rotation.z += delta * 0.5
  })

  return (
    <mesh ref={meshRef}>
      <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
      <lambertMaterial color={[1, 0.3, 0.1]} />
    </mesh>
  )
}
```

### `useEngine`

Access the engine instance:

```typescript
const useEngine = (): Engine => {
  return useContext(EngineContext)
}
```

### `useLoader`

Load assets with automatic caching and suspense support:

```typescript
const useLoader = <T,>(
  loader: (url: string, device: GPUDevice) => Promise<T>,
  url: string
): T => {
  // Uses React.Suspense for async loading
  // Caches results by URL
  // Throws promise for Suspense boundary
}
```

Usage:

```tsx
const Character = () => {
  const gltf = useLoader(loadGLTF, '/models/character.glb')

  return <primitive object={gltf.scene} />
}

// With Suspense boundary:
const App = () => (
  <Canvas>
    <Suspense fallback={null}>
      <Character />
    </Suspense>
  </Canvas>
)
```

### `useAnimations`

Manage skeletal animations on a loaded model:

```typescript
const useAnimations = (
  gltf: GLTFResult,
  ref?: RefObject<SkinnedMesh>
): {
  mixer: AnimationMixer
  actions: Record<string, AnimationAction>
  play: (name: string) => void
  crossFadeTo: (name: string, duration: number) => void
}
```

Usage:

```tsx
const AnimatedCharacter = () => {
  const gltf = useLoader(loadGLTF, '/models/character.glb')
  const { play, crossFadeTo } = useAnimations(gltf)

  useEffect(() => {
    play('Idle')
  }, [])

  const onStartWalking = () => crossFadeTo('Walk', 0.3)

  return <primitive object={gltf.scene} />
}
```

### `useRaycaster`

Convenient raycasting from pointer position:

```typescript
const useRaycaster = (): {
  intersect: (targets: Mesh[]) => SceneRayHit | null
  intersectAll: (targets: Mesh[]) => SceneRayHit[]
}
```

## `<Overlay>` Component

React component for HTML overlays (see [html-overlay.md](html-overlay.md)):

```tsx
interface OverlayProps {
  children: React.ReactNode
  position?: [number, number, number]    // Static world position
  node?: Node                            // Track a node
  offset?: [number, number, number]      // Offset from node/position
  center?: boolean                       // Center on point (default: true)
  occlude?: boolean                      // Hide when occluded
  distanceScale?: boolean                // Scale with distance
  maxDistance?: number                    // Hide beyond distance
  style?: React.CSSProperties            // Extra styles
  className?: string
}

// Usage:
<Overlay node={characterRef.current} offset={[0, 0, 2.5]} center maxDistance={50}>
  <div className="health-bar">
    <div style={{ width: `${health}%`, background: 'red', height: '4px' }} />
  </div>
</Overlay>
```

The `<Overlay>` component uses a React portal to render its children into the overlay container, while managing the overlay lifecycle through the engine's overlay system.

## `<primitive>` Element

For inserting pre-built scene graph subtrees (e.g., from glTF):

```tsx
// Inserts the glTF scene directly into the React-managed scene graph
<primitive object={gltf.scene} position={[0, 0, 0]} />
```

The `primitive` element wraps an existing Caracal node and manages its lifecycle within the React tree.

## Full Example

```tsx
import { Canvas, useFrame, useLoader, Overlay } from 'caracal/react'
import { loadGLTF, createMaterialIndexMap, OrbitControls } from 'caracal'

const materialColors = createMaterialIndexMap({
  0: { color: [1, 1, 1] },
  1: { color: [0, 0, 0] },
  2: { color: [0, 0.8, 0.7], emissive: [0, 1, 0.9], emissiveIntensity: 0.7 },
})

const Ground = () => (
  <mesh rotation={[0, 0, 0]} receiveShadow>
    <planeGeometry args={{ width: 200, height: 200 }} />
    <lambertMaterial color={[0.3, 0.5, 0.2]} />
  </mesh>
)

const Character = ({ position }: { position: [number, number, number] }) => {
  const gltf = useLoader(loadGLTF, '/models/character.glb')
  const meshRef = useRef<Node>(null)

  return (
    <group position={position}>
      <primitive ref={meshRef} object={gltf.scene} />
      <Overlay node={meshRef.current} offset={[0, 0, 2.5]} center>
        <div className="name-tag">Player</div>
      </Overlay>
    </group>
  )
}

const App = () => (
  <Canvas
    shadows
    bloom={{ intensity: 0.8, strength: 0.5 }}
    camera={{ position: [0, -15, 10], fov: 60 }}
  >
    <ambientLight color={[0.3, 0.35, 0.4]} intensity={1} />
    <directionalLight
      position={[20, 20, 30]}
      intensity={1}
      castShadow
      shadowMapSize={2048}
    />

    <Ground />

    <Suspense fallback={null}>
      <Character position={[0, 0, 0]} />
    </Suspense>

    <OrbitControls />
  </Canvas>
)
```

## Reconciler Performance

The React reconciler adds minimal overhead:

- **Prop diffing**: Only changed props trigger scene graph updates
- **No re-renders on frame tick**: `useFrame` callbacks run outside React's render cycle
- **Batched updates**: Multiple prop changes in a single React render commit are batched into a single scene graph update
- **Ref-based access**: Heavy per-frame operations use refs to bypass React's update mechanism
