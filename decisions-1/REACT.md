# REACT - Design Decisions

## Reconciler Approach

**Decision: Custom React reconciler using `react-reconciler` package**

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren (8/9)
- Rejected: Plain useEffect + refs (Bonobo — too much boilerplate, loses declarative benefits)

```typescript
import Reconciler from 'react-reconciler'

const reconciler = Reconciler({
  createInstance(type, props) { /* create engine object from JSX element type */ },
  appendChild(parent, child) { /* parent.add(child) or attach geometry/material */ },
  removeChild(parent, child) { /* parent.remove(child); child.dispose() */ },
  commitUpdate(instance, payload) { /* apply prop changes to engine object */ },
  // ... full host config
})
```

Rationale: Full R3F-style declarative scene construction. React's reconciliation handles diffing and lifecycle. Supports full React features (refs, context, portals, Suspense). The ~25-35KB reconciler overhead is well worth the DX improvement.

## Canvas Root Component

**Decision: `<Canvas>` as the root component creating engine, scene, camera, and render loop**

- Sources: All 9 implementations agree

```tsx
<Canvas
  shadows
  bloom={{ intensity: 0.5 }}
  camera={{ position: [5, -10, 5], fov: 60 }}
  antialias
>
  {/* Scene contents */}
</Canvas>
```

Canvas initializes the engine (async, with Suspense internally), creates the scene and default camera, starts the render loop via `requestAnimationFrame`. Render loop runs independently of React re-renders.

## Element Type Mapping

**Decision: Lowercase JSX elements mapping directly to engine types**

- Sources: All 9 implementations agree

```tsx
<mesh position={[0, 0, 1]} rotation={[0, 0, 0]} scale={[1, 1, 1]}>
  <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
  <lambertMaterial color={[1, 0, 0]} />
</mesh>
```

TypeScript JSX intrinsic elements for type safety:
```typescript
declare global {
  namespace JSX {
    interface IntrinsicElements {
      mesh: MeshProps
      group: GroupProps
      skinnedMesh: SkinnedMeshProps
      boxGeometry: BoxGeometryProps
      sphereGeometry: SphereGeometryProps
      // ... all primitives
      basicMaterial: BasicMaterialProps
      lambertMaterial: LambertMaterialProps
      ambientLight: AmbientLightProps
      directionalLight: DirectionalLightProps
      perspectiveCamera: CameraProps
    }
  }
}
```

## Geometry/Material Attachment

**Decision: Special attachment logic — geometries and materials attach as properties, not children**

- Sources: All 8 reconciler-based implementations agree

```typescript
appendChild(parent, child) {
  if (child instanceof Geometry) {
    parent.geometry = child
  } else if (child instanceof Material) {
    parent.material = child
  } else {
    parent.add(child)  // scene graph child
  }
}
```

## Core Hooks

**Decision: useFrame, useEngine, useLoader (universal agreement)**

### useFrame
```typescript
useFrame((state, delta) => {
  meshRef.current.rotation[2] += delta  // runs every frame, outside React
})
```

No priority parameter — callbacks execute in registration order (6/9 implementations agree). Simpler API, sufficient for most use cases.

### useEngine
```typescript
const { scene, camera, renderer, gl } = useEngine()
```

Access engine internals from any component within `<Canvas>`.

### useLoader (Suspense-compatible)
```typescript
const useLoader = <T>(loaderFn: (url: string) => Promise<T>, url: string): T => {
  // Throws promise on first call → Suspense catches and shows fallback
  // Returns cached result on subsequent renders
}
```

## Convenience Hooks

### useGLTF
```typescript
const gltf = useGLTF('/model.glb')
// gltf.scene, gltf.animations, gltf.skeletons
```

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit (6/9)

### useAnimations
```typescript
const { actions, mixer } = useAnimations(gltf.animations, skeleton)
useEffect(() => { actions.idle?.play() }, [])
```

- Sources: Caracal, Fennec, Hyena, Lynx, Rabbit (5/9)

## Event System

**Decision: Pointer events via raycasting on meshes with event handlers**

- Sources: All 8 reconciler-based implementations agree

```tsx
<mesh
  onClick={(e) => console.log('clicked at', e.point)}
  onPointerOver={(e) => setHovered(true)}
  onPointerOut={(e) => setHovered(false)}
>
```

Performance optimization: raycasting is performed only against meshes that have event handlers registered — not all meshes in the scene.

## HTML Overlay Integration

**Decision: `<Html>` component for DOM overlays**

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark (7/9)

```tsx
<Html position={[0, 0, 2]} center occlude>
  <div className="health-bar">HP: 100</div>
</Html>
```

Renders children into overlay container via `ReactDOM.createPortal()`. React context flows through to overlay content.

## Primitive Component

**Decision: `<primitive>` for inserting pre-built scene graphs**

- Sources: All reconciler-based implementations agree

```tsx
const gltf = useGLTF('/character.glb')
return <primitive object={gltf.scene} />
```

Inserts an existing engine object tree into the React-managed scene graph without re-creating it as JSX.

## Asset Loading with Suspense

```tsx
<Canvas>
  <Suspense fallback={<LoadingSpinner />}>
    <Model url="/scene.glb" />
  </Suspense>
</Canvas>
```

- Asset loading suspends React subtree, not the render loop
- Scene continues rendering while models load
- No frame drops during asset loading

## Automatic Disposal

**Decision: Dispose GPU resources on unmount**

- Sources: All 8 reconciler-based implementations agree

```typescript
removeChild(parent, child) {
  parent.remove(child)
  // Recursively dispose geometry, material, textures
  if (child.geometry) child.geometry.dispose()
  if (child.material) child.material.dispose()
}
```

Reference counting for shared resources (Hyena, Mantis, Shark approach): only dispose when the last reference is removed.

## Render Loop Independence

**Decision: Render loop runs outside React via requestAnimationFrame**

- Sources: All implementations agree
- React re-renders only affect scene graph structure, not render timing
- `useFrame` callbacks bypass React reconciliation entirely
- Heavy per-frame logic uses refs to avoid React overhead:

```tsx
const ref = useRef<Mesh>(null)
useFrame(() => {
  ref.current!.rotation[2] += 0.01  // direct mutation, no React re-render
})
return <mesh ref={ref}><boxGeometry /><lambertMaterial /></mesh>
```

## Prop Batching

Multiple prop changes in a single React render batch into one scene graph update. No intermediate states are rendered.

## OrbitControls Component

```tsx
<OrbitControls
  target={[0, 0, 0]}
  minDistance={1}
  maxDistance={100}
  dampingFactor={0.1}
/>
```

## Backend Selection

```tsx
<Canvas backend="auto">    {/* default: auto-detect */}
<Canvas backend="webgpu">  {/* force WebGPU */}
<Canvas backend="webgl2">  {/* force WebGL2 */}
```

## Bundle Impact

Estimated reconciler overhead: ~25-35KB (reconciler + host config). Tree-shaken away for non-React users who import from the base engine entry point.
