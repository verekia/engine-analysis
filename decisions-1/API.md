# API - Design Decisions

## Code Style

**Decision: TypeScript, no semicolons, const arrow functions (from requirements)**

```typescript
const createMesh = (geometry: Geometry, material: Material): Mesh => {
  // ...
}

const updateWorldMatrix = (node: Node): void => {
  // ...
}
```

Strict mode TypeScript throughout. Options objects for configuration (universal pattern). Async/await for loading.

## Initialization Pattern

**Decision: Single async factory function returning an engine object**

- Sources: Bonobo, Fennec, Rabbit, Mantis (4/9 — single engine class)
- Includes: device creation, renderer setup, default camera

```typescript
const engine = await createEngine(canvas, {
  backend: 'auto',          // 'auto' | 'webgpu' | 'webgl2'
  antialias: true,          // 4x MSAA
  shadows: true,            // or { mapSize: 2048, cascades: 3, ... }
  bloom: true,              // or { intensity: 0.5, levels: 5, ... }
  toneMapping: 'aces',      // 'aces' | 'reinhard' | 'none'
})
```

Auto-detects WebGPU with graceful WebGL2 fallback. The `await` handles async adapter/device creation.

## Scene Creation

**Decision: Scene as separate object (more flexible)**

- Sources: Caracal, Fennec, Hyena, Lynx, Rabbit, Shark, Wren (7/9)

```typescript
const scene = createScene()
scene.add(mesh)
scene.ambientLight = { color: [0.4, 0.4, 0.5], intensity: 0.3 }
```

Separate from engine because some applications may need multiple scenes (e.g., 3D preview panels alongside main viewport).

## Object Creation

**Decision: Factory functions**

- Sources: Caracal, Fennec, Mantis, Shark, Wren (5/9)
- Rejected: Class constructors (Hyena, Lynx, Rabbit) — factory functions align better with the "no semicolons, const arrow" style and enable better tree-shaking

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
const dirLight = createDirectionalLight({ color: [1, 1, 1], intensity: 1.0, castShadow: true })
```

## Transform API

```typescript
mesh.position.set(1, 0, 3)
mesh.rotation = quatFromEuler(0, 0, Math.PI / 4)
mesh.scale.set(2, 2, 2)
mesh.visible = false
mesh.frustumCulled = true
mesh.lookAt([10, 0, 0])
```

Position and scale are Vec3 (Float32Array views). Rotation is quaternion. Setting any property marks the node dirty for world matrix recomputation.

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

**Decision: First-class feature for per-vertex material properties**

- Sources: 7/9 implementations feature material index

```typescript
const material = createLambertMaterial({
  palette: [
    { color: [1, 1, 1], opacity: 1.0 },                                     // index 0: white opaque
    { color: [0.2, 0.2, 0.2], emissive: [0, 1, 1], emissiveIntensity: 0.7 }, // index 1: dark with cyan glow
    { color: [1, 0, 0], opacity: 0.5 },                                      // index 2: transparent red
  ],
})
```

Per-vertex `_materialindex` attribute selects palette entry. Enables complex visual effects (glowing eyes, colored armor parts) with a single draw call.

## Animation API

```typescript
const mixer = createAnimationMixer(skeleton)
const idleAction = mixer.clipAction(idleClip)
const walkAction = mixer.clipAction(walkClip)

idleAction.play()
idleAction.crossFadeTo(walkAction, 0.3)

walkAction.loop = 'repeat'
walkAction.timeScale = 1.5

// Each frame:
mixer.update(deltaTime)
```

## Raycasting API

```typescript
const raycaster = createRaycaster()
raycaster.setFromCamera([mouseX, mouseY], camera)
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
  basis: { wasmUrl: '/wasm/basis.wasm' },
})

scene.add(gltf.scene)

// Access animations
const mixer = createAnimationMixer(gltf.skeletons[0])
const action = mixer.clipAction(gltf.animations[0])
action.play()

// Cleanup
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

// Called each frame (by engine internally)
overlay.update(camera, canvasWidth, canvasHeight)
```

## Render Loop

```typescript
// Option 1: Built-in loop
engine.onFrame((deltaTime) => {
  mixer.update(deltaTime)
  controls.update(deltaTime)
})
engine.start()

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

**Decision: Boolean shorthand or detailed config object (universal pattern)**

```typescript
// Simple
const engine = await createEngine(canvas, { shadows: true, bloom: true })

// Detailed
const engine = await createEngine(canvas, {
  shadows: { mapSize: 2048, cascades: 3, maxDistance: 200 },
  bloom: { intensity: 0.5, levels: 5 },
})
```

## Backend Selection

```typescript
const engine = await createEngine(canvas, {
  backend: 'auto',    // default: try WebGPU, fall back to WebGL2
  // backend: 'webgpu',  // force WebGPU (throws if unavailable)
  // backend: 'webgl2',  // force WebGL2
})

engine.backend  // 'webgpu' | 'webgl2' — which backend was selected
```

## Frame Statistics

```typescript
const stats = engine.getStats()
// { fps, frameTime, drawCalls, triangles, stateChanges, culled, visible, ... }
```

## Resource Disposal

```typescript
// Individual resources
geometry.dispose()
material.dispose()
texture.dispose()

// GLTF batch
gltf.dispose()

// Engine (everything)
engine.dispose()
```

React bindings handle disposal automatically on unmount.

## Debug Mode

```typescript
const engine = await createEngine(canvas, { debug: true })
// Enables:
// - Verbose console warnings for common mistakes
// - GPU validation layers (WebGPU)
// - Frame stats overlay
// - Shader compilation warnings
```

## Error Handling

```typescript
try {
  const engine = await createEngine(canvas)
} catch (e) {
  if (e instanceof UnsupportedBackendError) {
    // Neither WebGPU nor WebGL2 available
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
