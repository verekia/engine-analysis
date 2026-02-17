# React Bindings Comparison Across 9 Implementations

This document compares React integration approaches across bonobo, caracal, fennec, hyena, lynx, mantis, rabbit, shark, and wren.

## Universal Agreement

All 9 implementations agree on:

### Core Philosophy
- **Declarative 3D scene construction** using JSX/React components
- **`<Canvas>` as root component** that creates engine, scene, camera, and render loop
- **React components map to engine objects** (meshes, lights, cameras, etc.)
- **Automatic lifecycle management**: mount creates, unmount destroys/disposes

### Fundamental Hooks
All implementations provide:
- **`useFrame(callback)`**: Register per-frame callbacks that run before rendering
- **`useEngine/useWorld/useWren/etc()`**: Access engine state from any component

### Standard Components
All support these scene elements as JSX:
- `<mesh>`, `<group>`, `<skinnedMesh>`
- `<boxGeometry>`, `<sphereGeometry>`, `<planeGeometry>`, `<cylinderGeometry>`, `<coneGeometry>`, `<capsuleGeometry>`, `<circleGeometry>`
- `<basicMaterial>`, `<lambertMaterial>`
- `<ambientLight>`, `<directionalLight>`
- `<primitive>` for inserting pre-built scene graphs (e.g., from GLTF)

### Prop Application
All implementations map JSX props to object properties:
```tsx
<mesh position={[0, 0, 1]} rotation={[0, 0, 0]} scale={[1, 1, 1]}>
```

### Asset Loading
All support async loading with React Suspense:
```tsx
<Suspense fallback={null}>
  <Model url="/model.glb" />
</Suspense>
```

## Key Variations

### 1. Reconciler Approach

**Custom React Reconciler (8 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren**: Use `react-reconciler` package
- Implements full host config to map React fiber tree to scene graph
- 2000-3000+ lines of reconciler code
- Supports full React features (refs, context, portals, etc.)

**No Custom Reconciler (1 implementation)**
- **Bonobo**: Uses plain `useEffect` + refs instead of custom reconciler
- 50 lines per component vs 2000+ line reconciler
- Explicit props, no deep spreading
- Simpler but less "magical"

### 2. Reconciler Philosophy

**Full R3F-style reconciler (7 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren**: Closely follow React Three Fiber patterns
- JSX elements directly create/update/remove engine objects
- Props map to object properties with intelligent diffing

**Thin wrapper (1 implementation)**
- **Bonobo**: Minimal React layer, no reconciler complexity
- Components call `world.spawn()` / `world.despawn()` in effects
- Tradeoff: Lose automatic prop spreading, gain simplicity

### 3. Element Type Mapping

**Lowercase JSX elements (9 implementations)**
All use lowercase element names matching engine types:
```tsx
<mesh>, <group>, <boxGeometry>, <lambertMaterial>
```

**Args prop for geometry/material (8 implementations)**
- **All except Bonobo**: Use `args` prop for constructor arguments
```tsx
<boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
```

**Direct props (1 implementation)**
- **Bonobo**: Pass geometry/material as props directly
```tsx
<Mesh geometry={boxGeometry} material={redMaterial} />
```

### 4. Geometry/Material Attachment

**Special attachment logic (8 implementations)**
Geometries and materials attach to parent mesh as properties, not scene children:
```typescript
if (child instanceof Geometry) {
  parent.geometry = child
} else if (child instanceof Material) {
  parent.material = child
} else {
  parent.add(child)
}
```

**Ref-based (1 implementation)**
- **Bonobo**: Pass pre-created geometry/material objects via props

### 5. useFrame Implementation

**Priority parameter (3 implementations)**
- **Caracal, Fennec, Hyena**: `useFrame(callback, priority)` for execution ordering

**No priority (6 implementations)**
- **Bonobo, Lynx, Mantis, Rabbit, Shark, Wren**: Simple `useFrame(callback)`
- Callbacks execute in registration order

**Delta time (all implementations)**
All pass delta time to callback:
```typescript
useFrame((state, delta) => {
  mesh.rotation.z += delta
})
```

### 6. Loader Hook

**useLoader with Suspense (8 implementations)**
- **All except Bonobo**: Generic `useLoader` that throws promises for Suspense
```typescript
const useLoader = <T>(loader, url) => {
  // Throws promise on first call, returns result on subsequent
}
```

**Convenience hooks (6 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit**: Provide `useGLTF` wrapper

**No loader hook (1 implementation)**
- **Bonobo**: Planned for future, not in v1 spec

### 7. Animation Hooks

**useAnimations (5 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Rabbit**: Hook for managing skeletal animations
```typescript
const { actions } = useAnimations(gltf.animations, skeleton)
useEffect(() => actions.idle?.play(), [])
```

**Not specified (4 implementations)**
- **Bonobo, Mantis, Shark, Wren**: Don't provide specialized animation hooks

### 8. Event System

**Pointer events via raycasting (8 implementations)**
- **All except Bonobo**: Support `onClick`, `onPointerOver`, `onPointerOut` on meshes
```tsx
<mesh onClick={(e) => console.log(e.point)} />
```
- Raycasting performed only on meshes with event handlers (performance optimization)

**Not specified (1 implementation)**
- **Bonobo**: Event system planned for future

### 9. HTML Overlay Integration

**`<Html>` component (7 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark**: Provide `<Html>` component for DOM overlays
```tsx
<Html position={[0, 0, 2]} center>
  <div>Health: 100</div>
</Html>
```

**`<Overlay>` component (1 implementation)**
- **Caracal**: Uses `<Overlay>` instead of `<Html>` (same concept)

**Future/not specified (1 implementation)**
- **Bonobo**: Overlay system out of scope for v1

### 10. Canvas Props

**Shadows (7 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark**: `shadows` prop on `<Canvas>`

**Bloom (6 implementations)**
- **Caracal, Hyena, Lynx, Mantis, Rabbit, Shark**: `bloom` prop with config object or boolean

**Camera config (9 implementations)**
All support camera configuration via Canvas props:
```tsx
<Canvas camera={{ position: [5, 5, 5], fov: 60 }}>
```

**Backend selection (4 implementations)**
- **Caracal, Fennec, Rabbit, Wren**: `backend` or `preferWebGPU` prop

## Implementation Breakdown

### By Reconciler Complexity
- **No reconciler**: Bonobo (1)
- **Standard reconciler**: Fennec, Hyena, Lynx, Rabbit, Wren (5)
- **Enhanced reconciler**: Caracal, Mantis, Shark (3) - additional features like extend()

### By Hook Richness
- **Essential only**: Bonobo, Shark, Wren (3)
- **Standard set**: Fennec, Hyena, Rabbit (3)
- **Extended set**: Caracal, Lynx, Mantis (3) - animations, raycasting utilities

### By Event Support
- **Full pointer events**: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren (8)
- **Planned**: Bonobo (1)

### By HTML Overlay
- **Integrated**: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark (7)
- **Future**: Bonobo, Wren (2)

### By Loader Integration
- **Suspense + useLoader**: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren (8)
- **Future**: Bonobo (1)

## Cherry-Picking Guide

### For Simplest React Integration
**Use Bonobo's approach**: No reconciler, plain useEffect + refs
- Pros: Simple, explicit, easy to debug
- Cons: No automatic prop spreading, more boilerplate per component

### For Full R3F Compatibility
**Use Caracal, Fennec, or Hyena**: Closest to React Three Fiber patterns
- Comprehensive reconciler
- All standard hooks (useFrame, useLoader, useAnimations)
- Full event system

### For Minimal Reconciler Code
**Use Wren or Rabbit**: Clean, focused reconciler implementations without extra features
- Core functionality only
- Smaller bundle size

### For Extended Features
**Use Mantis or Lynx**: Additional utilities and optimizations
- Extended hook library
- Advanced prop handling
- Performance optimizations (memoization hints, etc.)

### For Type Safety
All implementations provide TypeScript support with JSX intrinsic elements:
```typescript
declare global {
  namespace JSX {
    interface IntrinsicElements {
      mesh: MeshProps
      boxGeometry: BoxGeometryProps
      // ...
    }
  }
}
```

### For Automatic Disposal
**All reconciler-based implementations (8)** handle cleanup automatically:
- Geometries, materials, textures disposed on unmount
- Reference counting for shared resources (Hyena, Mantis, Shark)

## Performance Considerations

### Render Loop Independence
**All implementations**: Render loop runs outside React via `requestAnimationFrame`
- React re-renders only affect scene graph structure, not render timing
- Frame callbacks (useFrame) bypass React reconciliation

### Prop Batching
**All reconciler implementations**: Multiple prop changes in single React render batch into one scene graph update

### Ref-based Updates
**All implementations**: Heavy per-frame logic uses refs to avoid React overhead:
```tsx
const ref = useRef<Mesh>()
useFrame(() => {
  ref.current.rotation.z += 0.01  // Direct mutation, no React
})
```

### Suspense Performance
**All loader implementations**: Asset loading suspends React subtree, not render loop
- Scene continues rendering while models load
- No frame drops during asset loading

## Bundle Size Comparison

Estimated reconciler overhead:
- **Bonobo**: ~5KB (no reconciler)
- **Standard implementations**: ~25-35KB (reconciler + host config)
- **Extended implementations**: ~40-50KB (reconciler + additional features)

*Note: Actual sizes depend on tree-shaking and shared dependencies*

## Migration Path

From **React Three Fiber** to these implementations:
1. **Easiest**: Caracal, Fennec, Hyena (most R3F-compatible)
2. **Moderate**: Lynx, Mantis, Rabbit, Shark, Wren (similar patterns, minor adjustments)
3. **Requires refactor**: Bonobo (different paradigm without reconciler)

## Recommendation Summary

- **Learning/simplicity**: Bonobo (no reconciler) or Wren (minimal reconciler)
- **Production/full features**: Caracal, Hyena, or Mantis (comprehensive implementations)
- **Performance-critical**: Rabbit or Shark (lean, focused implementations)
- **R3F migration**: Caracal or Fennec (closest patterns)
