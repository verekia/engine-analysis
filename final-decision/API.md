# API — Final Decision

## Code Style

**Decision: TypeScript, no semicolons, const arrow functions** (universal agreement)

```typescript
const createMesh = (geometry: Geometry, material: Material): Mesh => {
  // ...
}
```

- TypeScript strict mode throughout
- Options objects for all configuration
- Async/await for loading
- `camelCase` for functions and variables
- `PascalCase` for types and interfaces
- `SCREAMING_SNAKE` for constants

## Initialization

**Decision: Single async factory `createEngine()`** (4/9: Bonobo, Fennec, Mantis, Rabbit — simpler DX)

```typescript
const engine = await createEngine(canvas, {
  backend: 'auto',          // 'auto' | 'webgpu' | 'webgl2'
  antialias: true,          // 4x MSAA
  shadows: true,            // or { mapSize: 1024, cascades: 3, ... }
  bloom: true,              // or { intensity: 0.5, levels: 5, ... }
  toneMapping: 'aces',      // 'aces' | 'reinhard' | 'none'
  debug: false,             // Verbose validation
})
```

Auto-detects WebGPU with graceful WebGL2 fallback. The `await` handles async adapter/device creation.

For advanced users, lower-level access is available via `engine.device` and `engine.renderer`.

## Scene Creation

**Decision: Scene as a separate object** (7/9: Caracal, Fennec, Hyena, Lynx, Rabbit, Shark, Wren)

```typescript
const scene = createScene()
scene.ambientLight = { color: [0.4, 0.45, 0.5], intensity: 0.3 }
scene.add(mesh)
```

Separate from engine because some applications may need multiple scenes.

## Object Creation

**Decision: Factory functions** (5/9: Caracal, Fennec, Mantis, Shark, Wren)

```typescript
// Geometry
const box = createBoxGeometry({ width: 1, height: 1, depth: 1 })
const sphere = createSphereGeometry({ radius: 0.5, widthSegments: 32 })

// Materials
const basic = createBasicMaterial({ color: [1, 0, 0] })
const lambert = createLambertMaterial({
  color: [0.8, 0.6, 0.4],
  receiveShadow: true,
  palette: [
    { color: [1, 1, 1] },
    { color: [0, 0, 0], emissive: [0, 1, 1], emissiveIntensity: 0.7 },
  ],
})

// Scene objects
const mesh = createMesh(box, lambert)
const group = createGroup()
const camera = createPerspectiveCamera({ fov: 60, near: 0.1, far: 1000 })
const sun = createDirectionalLight({ color: [1, 1, 1], intensity: 1.0, castShadow: true })
```

All factory functions take a single options object with defaults for every field.

## Transform API

```typescript
mesh.position.set(1, 0, 3)
mesh.rotation = quatFromEuler(0, 0, Math.PI / 4)
mesh.scale.set(2, 2, 2)
mesh.visible = false
mesh.frustumCulled = true
mesh.castShadow = true
mesh.receiveShadow = true
mesh.lookAt([10, 0, 0])
```

Position and scale are Vec3 (Float32Array views). Rotation is quaternion. Setting any property marks the node dirty.

## Scene Graph API

```typescript
// Parent/child
scene.add(mesh)
group.add(child1, child2)
group.remove(child1)

// Traversal
scene.traverse((node) => { /* visits every node depth-first */ })

// Lookup
scene.getByName('player')

// Bone attachment
const handBone = skeleton.getBone('hand_R')
handBone.add(swordMesh)
```

## Material Index System

**Decision: First-class feature** (7/9 implementations)

```typescript
const material = createLambertMaterial({
  palette: [
    { color: [1, 1, 1], opacity: 1.0 },
    { color: [0.2, 0.2, 0.2], emissive: [0, 1, 1], emissiveIntensity: 0.7 },
    { color: [1, 0, 0], opacity: 0.5 },
  ],
})
```

Per-vertex `_materialindex` attribute selects palette entry. Enables complex visual effects with a single draw call.

## Animation API

```typescript
const mixer = createAnimationMixer(skeleton)
const idle = mixer.clipAction(idleClip)
const walk = mixer.clipAction(walkClip)

idle.play()
idle.crossFadeTo(walk, 0.3)  // 300ms crossfade

walk.loop = 'repeat'
walk.timeScale = 1.5

// Each frame:
mixer.update(deltaTime)
```

## Raycasting API

```typescript
const raycaster = createRaycaster()
raycaster.setFromCamera([mouseNdcX, mouseNdcY], camera)
const hits = raycaster.intersectObjects(scene.children, true)

hits[0].distance   // world-space distance
hits[0].point      // world-space position
hits[0].normal     // interpolated normal
hits[0].object     // the mesh that was hit
```

## Controls API

```typescript
const controls = createOrbitControls(camera, canvas, {
  target: [0, 0, 0],
  dampingFactor: 0.1,
  minDistance: 1,
  maxDistance: 100,
})

controls.update(deltaTime)
controls.setTarget([10, 0, 5], { animate: true, duration: 0.5 })
controls.dispose()
```

## Asset Loading API

```typescript
const gltf = await loadGLTF('/model.glb', engine, {
  draco: { wasmUrl: '/wasm/draco.wasm' },
  ktx2: { wasmUrl: '/wasm/basis.wasm' },
})

scene.add(gltf.scene)

const mixer = createAnimationMixer(gltf.skeletons[0])
mixer.clipAction(gltf.animations[0]).play()

gltf.dispose()
```

## HTML Overlay API

```typescript
const overlay = createOverlayManager(canvas)

overlay.add({
  element: labelDiv,
  node: characterMesh,
  offset: [0, 0, 2.5],
  center: true,
  occlude: true,
})

// Updated internally by engine each frame
```

## Render Loop

```typescript
// Option 1: Built-in loop
engine.onFrame((deltaTime) => {
  mixer.update(deltaTime)
  controls.update(deltaTime)
})
engine.start(scene, camera)

// Option 2: Manual control
const animate = () => {
  requestAnimationFrame(animate)
  mixer.update(deltaTime)
  controls.update(deltaTime)
  engine.render(scene, camera)
}
animate()
```

## Configuration Pattern

**Decision: Boolean shorthand or detailed config object** (universal pattern)

```typescript
// Simple
const engine = await createEngine(canvas, { shadows: true, bloom: true })

// Detailed
const engine = await createEngine(canvas, {
  shadows: { mapSize: 1024, cascades: 3, maxDistance: 200 },
  bloom: { intensity: 0.5, levels: 5 },
})
```

Both forms accepted. Boolean `true` uses sensible defaults.

## Backend Selection

```typescript
const engine = await createEngine(canvas, {
  backend: 'auto',     // default: try WebGPU, fall back to WebGL2
  // backend: 'webgpu',   // force WebGPU (throws if unavailable)
  // backend: 'webgl2',   // force WebGL2
})

engine.backend  // 'webgpu' | 'webgl2'
```

## Frame Statistics

```typescript
const stats = engine.getStats()
// { fps, frameTime, jsTime, gpuTime, drawCalls, triangles, stateChanges, culledObjects, visibleObjects }
```

## Resource Disposal

```typescript
geometry.dispose()
material.dispose()
gltf.dispose()
engine.dispose()  // Everything
```

React bindings handle disposal automatically on unmount.

## Debug Mode

```typescript
const engine = await createEngine(canvas, { debug: true })
// Verbose validation errors, shader compilation diagnostics, GPU validation layers
```

## Error Handling

```typescript
try {
  const engine = await createEngine(canvas)
} catch (e) {
  if (e instanceof UnsupportedBackendError) {
    showFallbackUI()
  }
}
```

## Complete Example

```typescript
const engine = await createEngine(canvas, {
  shadows: true,
  bloom: { intensity: 0.5 },
})

const scene = createScene()
scene.ambientLight = { color: [0.4, 0.4, 0.5], intensity: 0.3 }

const sun = createDirectionalLight({ color: [1, 1, 0.95], intensity: 1.0, castShadow: true })
sun.position.set(50, 50, 100)
scene.add(sun)

const ground = createMesh(
  createPlaneGeometry({ width: 200, height: 200 }),
  createLambertMaterial({ color: [0.3, 0.5, 0.2], receiveShadow: true })
)
scene.add(ground)

const gltf = await loadGLTF('/character.glb', engine)
scene.add(gltf.scene)

const mixer = createAnimationMixer(gltf.skeletons[0])
mixer.clipAction(gltf.animations[0]).play()

const camera = createPerspectiveCamera({ fov: 60 })
camera.position.set(5, -10, 5)
camera.lookAt([0, 0, 1])

const controls = createOrbitControls(camera, canvas, { target: [0, 0, 1] })

engine.onFrame((dt) => {
  mixer.update(dt)
  controls.update(dt)
})
engine.start(scene, camera)
```
