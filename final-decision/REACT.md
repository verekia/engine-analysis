# REACT — Final Decision

## Reconciler

**Decision: Custom React reconciler via `react-reconciler`** (universal agreement)

A custom reconciler maps React's component tree to the engine's scene graph. This enables declarative scene construction with automatic lifecycle management (creation, updates, disposal).

```tsx
import { Canvas, useFrame, useGLTF } from 'engine-react'

const App = () => (
  <Canvas shadows bloom camera={{ position: [5, -10, 5], fov: 60 }}>
    <ambientLight intensity={0.3} />
    <directionalLight position={[10, -10, 10]} castShadow />
    <mesh position={[0, 0, 0.5]} castShadow>
      <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
      <lambertMaterial color={[0.8, 0.2, 0.2]} />
    </mesh>
  </Canvas>
)
```

## Canvas Component

The `<Canvas>` component is the root. It:

1. Creates a `<canvas>` DOM element
2. Initializes the engine (device, renderer, scene, camera)
3. Starts the render loop via `requestAnimationFrame`
4. Provides engine context to child components

```tsx
<Canvas
  shadows={true}                    // or { mapSize: 1024, ... }
  bloom={{ intensity: 0.5 }}
  camera={{ position: [5, -10, 5], fov: 60 }}
  backend="auto"                    // 'auto' | 'webgpu' | 'webgl2'
  antialias={true}
  onCreated={(engine) => { }}       // Called after initialization
>
  {children}
</Canvas>
```

## JSX Element Mapping

Lowercase JSX elements map to engine types:

| JSX Element | Engine Type |
|-------------|-------------|
| `<group>` | Group |
| `<mesh>` | Mesh |
| `<skinnedMesh>` | SkinnedMesh |
| `<perspectiveCamera>` | PerspectiveCamera |
| `<directionalLight>` | DirectionalLight |
| `<ambientLight>` | AmbientLight |
| `<boxGeometry>` | BoxGeometry |
| `<sphereGeometry>` | SphereGeometry |
| `<lambertMaterial>` | LambertMaterial |
| `<basicMaterial>` | BasicMaterial |

## Geometry and Material Attachment

**Decision: Special reconciler logic for child geometry/material** (universal agreement)

When a geometry or material is a child of a mesh in JSX, the reconciler attaches it to the parent mesh rather than adding it to the scene graph:

```tsx
<mesh>
  <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />  {/* mesh.geometry = ... */}
  <lambertMaterial color={[1, 0, 0]} />                       {/* mesh.material = ... */}
</mesh>
```

The `args` prop passes construction options to the factory function.

## Hooks

### useFrame

Per-frame callback. Runs every `requestAnimationFrame`, outside of React's render cycle.

```typescript
useFrame((deltaTime, engine) => {
  meshRef.current.rotation = quatFromEuler(0, 0, elapsed * 0.5)
})
```

Callbacks execute in registration order. No priority system — keep it simple.

### useEngine

Access the engine, renderer, scene, and camera:

```typescript
const { engine, scene, camera } = useEngine()
```

### useLoader

Generic Suspense-compatible asset loader:

```typescript
const texture = useLoader(loadTexture, '/textures/diffuse.ktx2')
```

Throws a Promise during loading (React Suspense catches it and shows fallback).

### useGLTF

Convenience hook for glTF loading:

```typescript
const gltf = useGLTF('/character.glb', {
  draco: { wasmUrl: '/wasm/draco.wasm' },
})
scene.add(gltf.scene)
```

Suspense-compatible. Cached by URL.

### useAnimations

Convenience hook for animation setup:

```typescript
const { actions } = useAnimations(gltf.animations, gltf.skeletons[0])
useEffect(() => { actions.idle?.play() }, [])
```

Returns a map of animation names to `AnimationAction` objects. Automatically creates a mixer and registers `mixer.update(delta)` in the frame loop.

## Pointer Events

**Decision: Raycast-based pointer events on meshes** (universal agreement)

```tsx
<mesh
  onClick={(e) => console.log('clicked', e.point)}
  onPointerOver={(e) => setHovered(true)}
  onPointerOut={(e) => setHovered(false)}
>
  <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
  <lambertMaterial color={hovered ? [1, 0, 0] : [0.5, 0.5, 0.5]} />
</mesh>
```

The reconciler registers meshes with pointer event handlers for raycasting. On each pointer event, a ray is cast from the camera through the pointer position, and events are dispatched to hit meshes.

Event interface:

```typescript
interface PointerEvent3D {
  object: Mesh
  point: Vec3
  normal: Vec3
  distance: number
  nativeEvent: MouseEvent | TouchEvent
  stopPropagation(): void
}
```

## Primitive Component

For pre-built scene graphs (e.g., loaded glTF models):

```tsx
const gltf = useGLTF('/model.glb')
return <primitive object={gltf.scene} />
```

The `primitive` component adds the object directly to the scene graph without wrapping it.

## Html Component

DOM overlays positioned in 3D space:

```tsx
<Html position={[0, 0, 2.5]} center occlude>
  <div className="label">Player Name</div>
</Html>
```

Implemented via `ReactDOM.createPortal` into the overlay container div. Uses the engine's HTML overlay system internally.

## Automatic Disposal

All GPU resources created by React components are disposed automatically on unmount:
- Geometries, materials, textures created via JSX
- glTF resources loaded via `useGLTF`
- Animation mixers from `useAnimations`

## Render Loop Independence

The render loop runs via `requestAnimationFrame`, outside of React. React state changes trigger prop updates that are applied to the scene graph objects directly (not through React re-renders of the canvas). This ensures 60fps rendering is never blocked by React's reconciliation.

## Prop Updates

When React re-renders a component, the reconciler diffs the old and new props and applies changes directly to the engine objects:

```tsx
// When position changes, reconciler calls: mesh.position.set(newX, newY, newZ)
<mesh position={[x, y, z]} />
```

## Package

The React bindings ship as a separate package:

```
engine-react/
  package.json  # peerDependencies: { engine: "^1.0.0", react: "^18.0.0" }
```

Bundle size estimate: ~25-35KB minified (reconciler + hooks + components).

## TypeScript

Full JSX type definitions for all engine elements:

```typescript
declare module 'engine-react' {
  namespace JSX {
    interface IntrinsicElements {
      mesh: MeshProps
      group: GroupProps
      boxGeometry: BoxGeometryProps
      lambertMaterial: LambertMaterialProps
      // ...
    }
  }
}
```
