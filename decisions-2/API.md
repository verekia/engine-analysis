# API.md - Decisions

## Decision: Factory Functions, Single Engine Entry Point, Material Index Emphasis

### Language: TypeScript-First

Universal agreement: TypeScript with strict mode, no semicolons, const arrow functions.

### Initialization: Single Engine Entry Point

**Chosen**: Unified engine entry point (4/9: Bonobo, Fennec, Mantis, Rabbit)
**Rejected**: Separate device + renderer (5/9) - more flexible but adds setup boilerplate that doesn't benefit the target use case

```typescript
const engine = await createEngine(canvas, {
  backend: 'auto',        // 'auto' | 'webgpu' | 'webgl2'
  antialias: true,
  sampleCount: 4,
  shadows: true,
  bloom: { intensity: 0.5 },
})

const { scene, camera, renderer } = engine
```

`createEngine` is `async` because WebGPU adapter/device creation is async. Auto-detection tries WebGPU first, falls back to WebGL2.

For advanced users who need lower-level control, the device and renderer remain accessible via `engine.device` and `engine.renderer`.

### Object Creation: Factory Functions

**Chosen**: Factory functions (5/9: Caracal, Fennec, Mantis, Shark, Wren)
**Rejected**: Class constructors (3/9: Hyena, Lynx, Rabbit) - less tree-shakeable, `new` keyword doesn't signal anything meaningful here

```typescript
// Scene graph
const group = createGroup()
const mesh = createMesh(geometry, material)
const camera = createPerspectiveCamera({ fov: 60, near: 0.1, far: 1000 })

// Lights
const sun = createDirectionalLight({ color: [1, 1, 0.9], intensity: 1.0, castShadow: true })
const ambient = createAmbientLight({ color: [0.4, 0.4, 0.5], intensity: 0.3 })

// Geometry
const box = createBoxGeometry({ width: 1, height: 1, depth: 1 })
const sphere = createSphereGeometry({ radius: 1, widthSegments: 32 })

// Materials
const basic = createBasicMaterial({ color: [1, 0, 0] })
const lambert = createLambertMaterial({ color: [0.8, 0.8, 0.8], receiveShadow: true })
```

All factory functions take a single options object with defaults for every field.

### Material Index System: First-Class Feature

**Chosen**: Prominent API for material index palettes (7/9 feature this)

This is the engine's distinguishing feature for the target use case (stylized/low-poly games with per-vertex material control):

```typescript
const characterMaterial = createLambertMaterial({
  palette: [
    { color: [0.9, 0.7, 0.6] },                                    // 0: skin
    { color: [0.2, 0.3, 0.8] },                                    // 1: shirt
    { color: [0.1, 0.1, 0.1] },                                    // 2: hair
    { color: [1.0, 0.8, 0.2], emissive: [1, 0.8, 0.2], emissiveIntensity: 2.0 },  // 3: glowing amulet
  ],
  receiveShadow: true,
})

// Geometry has _materialindex attribute (uint8 per vertex)
// Vertex shader uses it to index into the palette UBO
```

### Scene Graph API

```typescript
scene.add(mesh)
scene.add(group)
group.add(childMesh)

mesh.position.set(1, 2, 3)
mesh.rotation.setFromEuler(0, 0, Math.PI / 4)
mesh.scale.set(2, 2, 2)

mesh.visible = false
mesh.castShadow = true

group.remove(childMesh)
```

### Render Loop

```typescript
// Option 1: Built-in loop
engine.onFrame((delta) => {
  // User logic
})
engine.start()

// Option 2: Manual loop
const animate = () => {
  requestAnimationFrame(animate)
  engine.update()  // Updates animation, dirty flags, etc.
  engine.render()  // Submits GPU commands
}
animate()
```

### Animation API

```typescript
const gltf = await loadGLTF('/character.glb', engine, {
  dracoDecoderUrl: '/wasm/draco/',
  basisTranscoderUrl: '/wasm/basis/',
})

scene.add(gltf.scene)

const mixer = createAnimationMixer(gltf.skeletons[0])
const idle = mixer.clipAction(gltf.animations[0])
idle.play()

engine.onFrame((delta) => {
  mixer.update(delta)
})
```

### Raycasting API

```typescript
const raycaster = createRaycaster()

// From camera + mouse position
raycaster.setFromCamera([mouseNdcX, mouseNdcY], camera)
const hits = raycaster.intersectObjects(scene.children, true)

if (hits.length > 0) {
  const { object, point, normal, distance, triangleIndex } = hits[0]
}
```

### Controls API

```typescript
const controls = createOrbitControls(camera, canvas, {
  target: [0, 0, 0],
  minDistance: 1,
  maxDistance: 100,
  enableDamping: true,
})

engine.onFrame((delta) => {
  controls.update(delta)
})
```

### HTML Overlay API

```typescript
const overlays = createOverlaySystem(canvas)

const label = overlays.add(labelElement, {
  node: characterMesh,
  offset: [0, 0, 2.5],
  center: true,
  occlude: true,
})

engine.onFrame(() => {
  overlays.update(camera, canvas.width, canvas.height)
})
```

### Configuration Philosophy

**Boolean or config object** for features (universal pattern):

```typescript
// Simple
{ shadows: true }

// Detailed
{
  shadows: {
    enabled: true,
    mapSize: 2048,
    cascades: 3,
    maxDistance: 200,
  }
}
```

Both forms accepted. Boolean `true` uses sensible defaults.

### Resource Disposal

Explicit disposal required (universal agreement):

```typescript
geometry.dispose()
material.dispose()
gltf.dispose()
engine.dispose()  // Cleans up everything
```

React bindings handle disposal automatically on unmount.

### Error Handling

```typescript
try {
  const engine = await createEngine(canvas)
} catch (e) {
  if (e instanceof UnsupportedBackendError) {
    // Show fallback message
  }
}
```

Debug mode for development:

```typescript
const engine = await createEngine(canvas, { debug: true })
// Verbose validation errors, shader compilation warnings, etc.
```

### Stats API

```typescript
engine.onFrameStats((stats) => {
  console.log(stats.fps, stats.drawCalls, stats.triangles)
})
```

### Style Conventions

Based on consensus across implementations:
- No semicolons (6/9 specify this)
- Const arrow functions (6/9 prefer)
- Options objects for all configuration (universal)
- Async/await for loading (universal)
- Factory functions over constructors (5/9)
- `camelCase` for functions and variables
- `PascalCase` for types and interfaces
- `SCREAMING_SNAKE` for constants

### Complete Example

```typescript
const engine = await createEngine(canvas, {
  shadows: true,
  bloom: { intensity: 0.5 },
})

const { scene, camera } = engine
camera.position.set(5, -10, 5)
camera.lookAt(0, 0, 0)

const sun = createDirectionalLight({ intensity: 1.0, castShadow: true })
sun.rotation.setFromEuler(Math.PI / 4, 0, Math.PI / 6)
scene.add(sun)

scene.add(createAmbientLight({ intensity: 0.3 }))

const ground = createMesh(
  createPlaneGeometry({ width: 50, height: 50 }),
  createLambertMaterial({ color: [0.3, 0.6, 0.2], receiveShadow: true })
)
scene.add(ground)

const gltf = await loadGLTF('/character.glb', engine)
scene.add(gltf.scene)

const mixer = createAnimationMixer(gltf.skeletons[0])
mixer.clipAction(gltf.animations[0]).play()

const controls = createOrbitControls(camera, canvas)

engine.onFrame((delta) => {
  mixer.update(delta)
  controls.update(delta)
})

engine.start()
```
