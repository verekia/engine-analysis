# REACT.md - React Bindings

## Reconciler Approach

**Decision: Custom React reconciler via `react-reconciler`** (8/9 implementations agree)

A custom reconciler maps React's fiber tree directly to the engine's scene graph. This provides:

- Declarative scene construction with JSX
- Automatic lifecycle management (mount creates, unmount disposes)
- Props map directly to object properties with intelligent diffing
- Full React features (refs, context, Suspense, portals)
- R3F-compatible patterns (familiar to the ecosystem)

### Why Not Thin Wrapper (Bonobo)?

Bonobo's `useEffect` + refs approach is simpler (~50 lines per component vs ~2000 for a reconciler) but:
- No automatic prop spreading (must manually sync every prop)
- More boilerplate per component
- No JSX-level composition (`<mesh><boxGeometry />` cannot auto-attach geometry to parent)
- No declarative add/remove (must manage scene.add/remove manually)

The reconciler is a one-time implementation cost that pays for itself across every component the user writes.

## Canvas Component

```tsx
<Canvas
  shadows
  bloom={{ intensity: 0.5, levels: 5 }}
  camera={{ position: [5, -10, 5], fov: 60 }}
  backend="auto"
  antialias
>
  {children}
</Canvas>
```

The `<Canvas>` component:
1. Creates a `<canvas>` DOM element
2. Initializes the Device (WebGPU or WebGL2)
3. Creates the Renderer, Scene, and Camera
4. Starts the render loop via `requestAnimationFrame`
5. Mounts the React reconciler tree as children of the scene

The render loop runs outside React - React re-renders only affect scene graph structure, not rendering timing.

## JSX Elements

**Decision: Lowercase JSX elements mapping to engine types** (universal agreement)

```tsx
<mesh position={[0, 0, 1]} rotation={[0, 0, Math.PI / 4]} castShadow>
  <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
  <lambertMaterial color={[1, 0, 0]} receiveShadow />
</mesh>

<group position={[5, 0, 0]}>
  <mesh>
    <sphereGeometry args={{ radius: 0.5 }} />
    <basicMaterial color={[0, 1, 0]} />
  </mesh>
</group>

<ambientLight intensity={0.3} />
<directionalLight position={[10, -10, 10]} castShadow />
```

### Geometry/Material Attachment

**Decision: Special attachment logic in reconciler** (8/8 reconciler implementations agree)

When a geometry or material is added as a child of a mesh in JSX, the reconciler attaches it as a property rather than a scene child:

```typescript
// In reconciler's appendChild:
if (child instanceof Geometry) {
  parent.geometry = child
} else if (child instanceof Material) {
  parent.material = child
} else {
  parent.add(child)  // Regular scene graph add
}
```

### Args Prop for Construction Parameters

```tsx
<boxGeometry args={{ width: 2, height: 1, depth: 1 }} />
<sphereGeometry args={{ radius: 1, widthSegments: 32 }} />
```

The `args` prop is passed to the factory function at creation time. Changing `args` recreates the geometry (not a hot path - geometry creation is rare after initial mount).

## Hooks

### useFrame

**Decision: Simple callback, no priority** (Lynx, Mantis, Rabbit, Shark, Wren - 5/8 use no priority)

```typescript
useFrame((state: FrameState, delta: number) => {
  meshRef.current.rotation[2] += delta
})
```

Callbacks execute in registration order. Priority ordering (Caracal, Fennec, Hyena) adds complexity without a clear use case - if execution order matters, users can structure their component tree accordingly.

### useEngine

```typescript
const { device, renderer, scene, camera } = useEngine()
```

Access to engine internals from any component. Provided via React context from `<Canvas>`.

### useLoader

**Decision: Suspense-compatible generic loader** (8/8 reconciler implementations)

```typescript
const gltf = useLoader(loadGLTF, '/model.glb')
```

Throws a Promise on first call (suspends the component). Returns the result on subsequent renders. Caches by URL.

### useGLTF (Convenience)

```typescript
const { scene, animations, skeletons } = useGLTF('/character.glb')
```

Wrapper around `useLoader(loadGLTF, url)` for the most common loading pattern.

### useAnimations

**Decision: Action-based animation hook** (Caracal, Fennec, Hyena, Lynx, Rabbit)

```typescript
const { actions, mixer } = useAnimations(gltf.animations, skeleton)

useEffect(() => {
  actions.idle?.play()
}, [])
```

Returns an object where keys are clip names and values are AnimationAction instances. Auto-disposes the mixer on unmount.

## Pointer Events

**Decision: Raycast-based pointer events on meshes** (8/8 reconciler implementations)

```tsx
<mesh
  onClick={(e) => console.log('clicked', e.point)}
  onPointerOver={(e) => setHovered(true)}
  onPointerOut={() => setHovered(false)}
>
  <boxGeometry />
  <lambertMaterial color={hovered ? [1, 1, 0] : [1, 0, 0]} />
</mesh>
```

**Performance**: Raycasting is performed only on meshes that have event handlers attached. The reconciler tracks which meshes have pointer events and only raycasts against those. For a scene with 2000 meshes where 10 have onClick handlers, only 10 meshes are tested per pointer event.

## Primitive Component

**Decision: `<primitive>` for inserting pre-built scene graphs** (universal in reconciler implementations)

```tsx
const gltf = useGLTF('/world.glb')
return <primitive object={gltf.scene} />
```

`<primitive>` takes a pre-built scene object (e.g., from GLTF loader) and mounts it into the React tree. Props are applied to the root node. This avoids converting the entire GLTF hierarchy into individual React components.

## Html Component

**Decision: `<Html>` for DOM overlays** (7/8 reconciler implementations)

```tsx
<Html position={[0, 0, 2]} center occlude>
  <div className="health-bar">HP: 100</div>
</Html>
```

Renders children into the HTML overlay container via `ReactDOM.createPortal()`. React context flows through the portal. See HTML-OVERLAY.md for implementation details.

## Automatic Disposal

**Decision: Auto-dispose on unmount** (universal in reconciler implementations)

When a component unmounts:
- Geometry is disposed (GPU buffers freed)
- Material is disposed (GPU resources freed)
- Textures are reference-counted - disposed when last material using them unmounts
- Scene nodes are removed from parent
- Animation mixers are stopped and disposed

This prevents GPU memory leaks in React applications where components mount/unmount dynamically.

## TypeScript Support

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
      ambientLight: AmbientLightProps
      directionalLight: DirectionalLightProps
      primitive: PrimitiveProps
      // ...
    }
  }
}
```

Full type safety for all JSX elements and props.

## Performance Notes

- Render loop runs via `requestAnimationFrame`, independent of React renders
- `useFrame` callbacks bypass React reconciliation entirely (direct mutation)
- Prop changes batch into a single scene graph update per React render
- Asset loading suspends the React subtree without blocking the render loop
- The reconciler adds ~25-35KB to the bundle (react-reconciler + host config)
