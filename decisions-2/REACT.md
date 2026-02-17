# REACT.md - Decisions

## Decision: Custom Reconciler (R3F-Style), Full Event System, Suspense Loading

### Reconciler: Custom react-reconciler

**Chosen**: Full `react-reconciler` based implementation (8/9: all except Bonobo)
**Rejected**: Thin `useEffect` wrapper (Bonobo) - simpler but loses automatic lifecycle management, prop spreading, and the declarative scene graph model that the prompt requests ("React bindings similar to React Three Fiber")

The custom reconciler maps React's fiber tree directly to the engine's scene graph:
- `createElement` -> create engine object (mesh, group, light, etc.)
- `appendChild` -> `parent.add(child)`
- `removeChild` -> `parent.remove(child)`
- `commitUpdate` -> apply changed props to engine object

### Canvas Root Component

```tsx
<Canvas
  camera={{ position: [5, -10, 5], fov: 60 }}
  shadows
  bloom={{ intensity: 0.5 }}
  antialias
>
  {/* Scene content */}
</Canvas>
```

`<Canvas>` creates the engine, scene, camera, render loop, and the reconciler container. It handles resize observation and cleanup on unmount.

### Element Types: Lowercase JSX

All element names are lowercase, matching engine types:

```tsx
<mesh position={[0, 0, 1]} castShadow>
  <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
  <lambertMaterial color={[1, 0, 0]} receiveShadow />
</mesh>

<group position={[0, 0, 0]}>
  <mesh>...</mesh>
  <mesh>...</mesh>
</group>

<directionalLight castShadow intensity={1.0} />
<ambientLight intensity={0.3} />
```

### Geometry/Material Attachment

Special attachment logic (8/9): geometry and material children attach as properties, not scene children:

```typescript
// In reconciler appendChild:
if (child instanceof Geometry) {
  parent.geometry = child
} else if (child instanceof Material) {
  parent.material = child
} else {
  parent.add(child)
}
```

### Core Hooks

**useFrame**: Per-frame callback, runs outside React reconciliation.

```typescript
useFrame((state, delta) => {
  meshRef.current.rotation.z += delta
})
```

No priority parameter (6/9 skip priorities) - callbacks execute in registration order. Keeping it simple avoids the complexity of priority scheduling.

**useEngine**: Access engine state from any component.

```typescript
const { scene, camera, device, renderer } = useEngine()
```

### Asset Loading: Suspense + useLoader

**Chosen**: Suspense-compatible loading (8/9)

```typescript
const useGLTF = (url: string): GLTFResult => {
  return useLoader(loadGLTF, url)
}

// Usage:
const Model = ({ url }: { url: string }) => {
  const gltf = useGLTF(url)
  return <primitive object={gltf.scene} />
}

// With Suspense:
<Suspense fallback={null}>
  <Model url="/character.glb" />
</Suspense>
```

Scene continues rendering while models load (asset loading suspends the React subtree, not the render loop).

### Animation Hook

**Chosen**: `useAnimations` hook (5/9: Caracal, Fennec, Hyena, Lynx, Rabbit)

```typescript
const { actions, mixer } = useAnimations(gltf.animations, skeletonRef)

useEffect(() => {
  actions.idle?.play()
  return () => mixer.stopAll()
}, [])
```

### Event System: Pointer Events via Raycasting

**Chosen**: Full pointer events (8/9)

```tsx
<mesh
  onClick={(e) => console.log('clicked at', e.point)}
  onPointerOver={(e) => setHovered(true)}
  onPointerOut={(e) => setHovered(false)}
>
```

Raycasting performed only on meshes with event handlers attached (performance optimization - no raycasting for non-interactive meshes).

### Automatic Disposal

All reconciler-based implementations handle cleanup on unmount:
- Geometries disposed when component unmounts
- Materials disposed when component unmounts
- Textures disposed when component unmounts
- GLTF resources disposed when component unmounts
- Reference counting for shared resources (prevent premature disposal)

### Primitive Component

For inserting pre-built scene graphs (e.g., from glTF loader):

```tsx
<primitive object={gltf.scene} position={[0, 0, 0]} />
```

The `primitive` element wraps an existing engine object without creating a new one.

### Prop Application

Props map to object properties with intelligent diffing:
- `position={[x, y, z]}` -> `node.position.set(x, y, z)`
- `rotation={[x, y, z]}` -> `node.rotation.setFromEuler(x, y, z)`
- `scale={[x, y, z]}` or `scale={2}` (uniform) -> `node.scale.set(...)`
- `color={[r, g, b]}` -> `material.color.set(r, g, b)`
- `visible={false}` -> `node.visible = false`

### Render Loop Independence

The render loop runs via `requestAnimationFrame`, completely independent of React's reconciliation. React re-renders only affect scene graph structure (add/remove/update nodes). `useFrame` callbacks bypass React entirely.

### TypeScript JSX Types

```typescript
declare global {
  namespace JSX {
    interface IntrinsicElements {
      mesh: MeshProps
      group: GroupProps
      boxGeometry: BoxGeometryProps
      sphereGeometry: SphereGeometryProps
      lambertMaterial: LambertMaterialProps
      basicMaterial: BasicMaterialProps
      directionalLight: DirectionalLightProps
      ambientLight: AmbientLightProps
      orbitControls: OrbitControlsProps
      // ...
    }
  }
}
```

### Html Overlay Integration

```tsx
<Html position={[0, 0, 2]} center occlude>
  <div className="health-bar">HP: 100</div>
</Html>
```

See HTML-OVERLAY.md for implementation details.

### Bundle Size

Estimated reconciler overhead: ~25-35KB (reconciler + host config + hooks). Ships as a separate subpath export (`engine/react`) to avoid loading reconciler code for non-React users.
