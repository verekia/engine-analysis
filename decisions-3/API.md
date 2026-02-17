# API.md - Public API Design

## Design Philosophy

- **TypeScript-first**: Full type safety, strict mode, no semicolons, const arrow functions
- **Factory functions over classes**: `createMesh()`, not `new Mesh()` (5/9 implementations prefer factory, better tree-shaking)
- **Options objects for configuration**: Named parameters, sensible defaults
- **Explicit resource management**: `dispose()` for GPU resources
- **Async initialization**: Device creation is async (GPU adapter request)

## Initialization

**Decision: Separate Device + Renderer** (Caracal, Hyena, Lynx, Shark, Wren - 5/9 prefer this)

```typescript
const device = await createDevice(canvas, {
  backend: 'auto',      // 'auto' | 'webgpu' | 'webgl2'
  antialias: true,       // 4x MSAA (default: true)
  debug: false,          // Verbose validation (default: false)
})

const renderer = createRenderer(device, {
  shadows: { enabled: true, cascades: 3, mapSize: 1024 },
  bloom: { enabled: true, intensity: 0.5, levels: 5 },
})

const scene = createScene()
const camera = createPerspectiveCamera({ fov: 60, near: 0.1, far: 1000 })
camera.position.set(5, -10, 5)
camera.lookAt(0, 0, 0)
scene.add(camera)
```

### Why Separate Over Unified Engine (Bonobo, Fennec, Mantis, Rabbit)

Separating device, renderer, and scene gives:
- Device can be shared across multiple renderers (e.g., minimap + main view)
- Renderer configuration is independent of GPU initialization
- Clearer separation of concerns (GPU resource management vs rendering strategy)
- Easier testing (mock device for unit tests)

The unified engine pattern is simpler for getting started, but the separate pattern scales better and the additional verbosity is minimal (3 lines instead of 1).

## Scene Graph

```typescript
// Create objects
const mesh = createMesh(geometry, material)
const group = createGroup()
const skinnedMesh = createSkinnedMesh(geometry, material, skeleton)

// Hierarchy
scene.add(mesh)
group.add(mesh)
group.remove(mesh)

// Transform
mesh.position.set(1, 0, 3)
mesh.rotation.setFromEuler(0, 0, Math.PI / 4)
mesh.scale.set(2, 2, 2)
mesh.visible = false
```

## Materials

```typescript
// Basic (unlit)
const basic = createBasicMaterial({
  color: [1, 0, 0],
})

// Lambert (diffuse lit)
const lambert = createLambertMaterial({
  color: [0.8, 0.8, 0.8],
  receiveShadow: true,
  vertexColors: false,
})

// Lambert with material index palette
const paletted = createLambertMaterial({
  palette: [
    { color: [1, 1, 1], emissive: [0, 0, 0], emissiveIntensity: 0 },
    { color: [1, 0, 0], emissive: [1, 0, 0], emissiveIntensity: 0.5 },
    { color: [0, 0, 1] },
  ],
  receiveShadow: true,
})

// Transparent
const glass = createLambertMaterial({
  color: [0.5, 0.7, 1.0],
  transparent: true,
  opacity: 0.4,
})
```

## Geometry

```typescript
// Parametric primitives
const box = createBoxGeometry({ width: 1, height: 1, depth: 1 })
const sphere = createSphereGeometry({ radius: 1, widthSegments: 32, heightSegments: 16 })
const plane = createPlaneGeometry({ width: 10, height: 10 })
const cylinder = createCylinderGeometry({ radiusTop: 1, radiusBottom: 1, height: 2 })
const cone = createConeGeometry({ radius: 1, height: 2, radialSegments: 32 })
const capsule = createCapsuleGeometry({ radius: 0.5, height: 2 })
const circle = createCircleGeometry({ radius: 1, segments: 32 })

// Custom geometry
const custom = createGeometry({
  positions: new Float32Array([...]),
  normals: new Float32Array([...]),
  uvs: new Float32Array([...]),
  indices: new Uint16Array([...]),
  materialIndices: new Uint8Array([...]),
})
```

## Lights

```typescript
// Ambient (scene-level, no spatial properties)
scene.ambientLight = createAmbientLight({ color: [1, 1, 1], intensity: 0.3 })

// Directional (scene node with direction from rotation)
const sun = createDirectionalLight({
  color: [1, 1, 0.9],
  intensity: 0.8,
  castShadow: true,
})
sun.rotation.setFromEuler(-0.8, 0, 0.3)
scene.add(sun)
```

## Animation

```typescript
const gltf = await loadGLTF('/character.glb', device)
const mixer = createAnimationMixer(gltf.skeletons[0])

const idle = mixer.clipAction(gltf.animations[0])
idle.play()

// Crossfade to walk
const walk = mixer.clipAction(gltf.animations[1])
mixer.crossFadeTo(walk, 0.3)

// Per frame
mixer.update(deltaTime)
```

## Asset Loading

```typescript
// GLTF with compression
const gltf = await loadGLTF('/model.glb', device, {
  draco: { wasmUrl: '/draco_decoder.wasm' },
  ktx2: { wasmUrl: '/basis_transcoder.wasm' },
})

// Add to scene
scene.add(gltf.scene)

// Access individual parts
const mesh = gltf.meshes[0]
const animations = gltf.animations
const skeleton = gltf.skeletons[0]

// Cleanup
gltf.dispose()
```

## Raycasting

```typescript
const raycaster = createRaycaster()

// From camera + screen coordinates
raycaster.setFromCamera(camera, ndcX, ndcY)

// Direct ray
raycaster.set(origin, direction)

// Query
const hits = raycaster.intersectObjects(scene.children, { recursive: true })
// hits[0].object, hits[0].point, hits[0].distance, hits[0].normal
```

## Controls

```typescript
const controls = new OrbitControls(camera, canvas, {
  target: [0, 0, 0],
  minDistance: 1,
  maxDistance: 100,
  dampingFactor: 0.1,
})

// Per frame
controls.update(deltaTime)

// Programmatic
controls.setTarget([5, 0, 0], { animate: true, duration: 1.0 })

// Cleanup
controls.dispose()
```

## HTML Overlay

```typescript
const overlay = createHTMLOverlaySystem(canvas)

const label = overlay.add({
  element: document.createElement('div'),
  node: mesh,
  offset: [0, 0, 2],
  center: true,
  occlude: true,
})

// Automatically updated each frame
// Manual removal:
overlay.remove(label)
overlay.dispose()
```

## Render Loop

```typescript
// Manual loop
const animate = () => {
  requestAnimationFrame(animate)
  controls.update(deltaTime)
  mixer.update(deltaTime)
  renderer.render(scene, camera)
}
animate()

// Or built-in loop
renderer.setAnimationLoop((delta) => {
  controls.update(delta)
  mixer.update(delta)
})
```

## React Quick Start

```tsx
import { Canvas, useFrame, useGLTF } from 'engine-react'

const Character = () => {
  const gltf = useGLTF('/character.glb')
  const { actions } = useAnimations(gltf.animations, gltf.skeletons[0])
  useEffect(() => { actions.idle?.play() }, [])
  return <primitive object={gltf.scene} />
}

const App = () => (
  <Canvas shadows bloom camera={{ position: [5, -10, 5], fov: 60 }}>
    <ambientLight intensity={0.3} />
    <directionalLight position={[10, -10, 10]} castShadow />
    <OrbitControls />

    <mesh position={[0, 0, 0.5]} castShadow receiveShadow>
      <boxGeometry args={{ width: 1, height: 1, depth: 1 }} />
      <lambertMaterial color={[0.8, 0.2, 0.2]} />
    </mesh>

    <Suspense fallback={null}>
      <Character />
    </Suspense>
  </Canvas>
)
```

## Resource Disposal

All GPU resources must be explicitly disposed when no longer needed:

```typescript
geometry.dispose()
material.dispose()
texture.dispose()
renderer.dispose()
device.dispose()
```

In React, disposal is automatic on component unmount.

## Frame Statistics

```typescript
const stats = renderer.getStats()
// { fps, frameTime, drawCalls, triangles, visibleObjects, culledObjects }
```

## Backend Selection

```typescript
// Auto-detect (default): try WebGPU, fall back to WebGL2
const device = await createDevice(canvas)

// Force specific backend
const device = await createDevice(canvas, { backend: 'webgpu' })
const device = await createDevice(canvas, { backend: 'webgl2' })

// Query backend
device.backend  // 'webgpu' | 'webgl2'
```

## Error Handling

```typescript
try {
  const device = await createDevice(canvas)
} catch (e) {
  if (e instanceof UnsupportedBackendError) {
    // Neither WebGPU nor WebGL2 available
    showFallbackUI()
  }
}
```

Debug mode provides verbose warnings for common mistakes:
```typescript
const device = await createDevice(canvas, { debug: true })
// Logs: "Warning: Geometry 'box' has no normals, lighting will not work correctly"
// Logs: "Warning: Material palette entry 5 referenced but only 3 entries defined"
```
