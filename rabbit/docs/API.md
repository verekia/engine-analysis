# API Reference & Code Style

## Code Style

All Rabbit source follows these conventions:

```typescript
// No semicolons
// const arrow functions preferred over function declarations
// TypeScript strict mode

// Good
const createMesh = (geometry: Geometry, material: Material): Mesh => {
  const mesh = new Mesh()
  mesh.geometry = geometry
  mesh.material = material
  return mesh
}

// Avoid
function createMesh(geometry: Geometry, material: Material): Mesh {
  const mesh = new Mesh();
  mesh.geometry = geometry;
  mesh.material = material;
  return mesh;
}
```

## Engine Initialization

### Imperative API

```typescript
import {
  Engine,
  Scene,
  Camera,
  Mesh,
  Group,
  BoxGeometry,
  SphereGeometry,
  PlaneGeometry,
  CylinderGeometry,
  ConeGeometry,
  CapsuleGeometry,
  CircleGeometry,
  BasicMaterial,
  LambertMaterial,
  AmbientLight,
  DirectionalLight,
  OrbitControls,
  HtmlOverlay,
  Raycaster,
  GLTFLoader,
  AnimationMixer,
  Vec3,
  Quat,
} from 'rabbit'

// Create engine (auto-detects WebGPU, falls back to WebGL2)
const canvas = document.getElementById('canvas') as HTMLCanvasElement

const engine = await Engine.create(canvas, {
  backend: 'auto',         // 'webgpu' | 'webgl2' | 'auto'
  msaa: 4,                 // 1 | 2 | 4
  pixelRatio: window.devicePixelRatio,
  bloom: {
    enabled: true,
    threshold: 1.0,
    intensity: 0.5,
  },
})

// Create scene and camera
const scene = new Scene()
scene.backgroundColor = [0.05, 0.05, 0.08]

const camera = new Camera()
camera.fov = 60 * (Math.PI / 180)
camera.near = 0.1
camera.far = 500
camera.position.set(10, -15, 8)
camera.lookAt(new Vec3(0, 0, 0))
scene.add(camera)
```

### Backend Detection

```typescript
// Check which backend was initialized
console.log(engine.device.backend) // 'webgpu' or 'webgl2'

// Check specific features
if (engine.device.features.textureCompressionASTC) {
  console.log('ASTC texture compression available')
}
```

## Scene Setup

### Lights

```typescript
// Ambient light
const ambient = new AmbientLight()
ambient.color = [1, 1, 1]
ambient.intensity = 0.15
scene.add(ambient)

// Directional light (sun)
const sun = new DirectionalLight()
sun.color = [1, 0.95, 0.9]
sun.intensity = 1.2
sun.setRotationFromEuler(-Math.PI / 4, 0, Math.PI / 6)
sun.castShadow = true
sun.shadowMapSize = 2048
sun.shadowCascades = 3
scene.add(sun)
```

### Geometries

```typescript
// All geometries are Z-up
const box = new BoxGeometry({ width: 2, height: 2, depth: 2 })
const sphere = new SphereGeometry({ radius: 1, widthSegments: 32, heightSegments: 16 })
const plane = new PlaneGeometry({ width: 100, height: 100 })
const cylinder = new CylinderGeometry({ radiusTop: 0.5, radiusBottom: 0.5, height: 2, segments: 16 })
const cone = new ConeGeometry({ radius: 0.5, height: 2, segments: 16 })
const capsule = new CapsuleGeometry({ radius: 0.5, height: 1, capSegments: 8, radialSegments: 16 })
const circle = new CircleGeometry({ radius: 1, segments: 32 })
```

### Materials

```typescript
// Basic (unlit) material
const basicMat = new BasicMaterial({
  color: [1, 0.5, 0.2],
})

// Lambert (diffuse) material
const lambertMat = new LambertMaterial({
  color: [0.8, 0.8, 0.8],
  receiveShadows: true,
})

// With textures
const texturedMat = new LambertMaterial({
  colorMap: await textureLoader.load('/textures/albedo.png'),
  aoMap: await ktx2Loader.load('/textures/ao.ktx2'),
  receiveShadows: true,
})

// With vertex colors
const vertexColorMat = new LambertMaterial({
  vertexColors: true,
  color: [1, 1, 1], // base color is multiplied by vertex colors
})

// Transparent material
const glassMat = new BasicMaterial({
  color: [0.3, 0.5, 0.9],
  transparent: true,
  opacity: 0.4,
})

// Material with per-index coloring + emissive bloom
const characterMat = new LambertMaterial({
  materialIndexEntries: [
    { color: [1, 1, 1] },                                                    // 0: white (skin)
    { color: [0.05, 0.05, 0.05] },                                           // 1: black (armor)
    { color: [0, 0.8, 0.7], emissive: [0, 0.8, 0.7], emissiveIntensity: 0.7 }, // 2: teal glow
  ],
  receiveShadows: true,
})
```

### Meshes and Groups

```typescript
// Create mesh
const cube = new Mesh()
cube.geometry = box
cube.material = lambertMat
cube.position.set(0, 0, 1) // Z-up: 1 unit above ground
scene.add(cube)

// Groups
const village = new Group()
village.name = 'village'
village.position.set(50, 50, 0)
scene.add(village)

const house = new Mesh()
house.geometry = box
house.material = lambertMat
village.add(house) // child of village group

// Parent/child transforms compose
village.position.set(50, 50, 0)
house.position.set(5, 0, 0)
// house world position = (55, 50, 0)
```

## Loading Assets

### glTF with Draco

```typescript
const gltfLoader = new GLTFLoader(engine.device, {
  dracoDecoderPath: '/draco/',   // path to draco_decoder.wasm
})

const gltf = await gltfLoader.load('/models/character.glb')

// gltf.scene: Node (root of the loaded scene graph)
// gltf.animations: AnimationClip[]
// gltf.meshes: Mesh[]

scene.add(gltf.scene)
```

### KTX2 Textures

```typescript
const ktx2Loader = new KTX2Loader(engine.device, {
  transcoderPath: '/basis/',    // path to basis_transcoder.wasm
})

const texture = await ktx2Loader.load('/textures/ground_color.ktx2')
// Automatically transcoded to ASTC/BC7/ETC2 based on device capability
```

## Animation

```typescript
const gltf = await gltfLoader.load('/models/character.glb')
const character = gltf.scene

// Find the skinned mesh
const skinnedMesh = character.findFirst(
  (node) => node instanceof SkinnedMesh
) as SkinnedMesh

// Create mixer
const mixer = new AnimationMixer(skinnedMesh.skeleton)

// Play idle animation
const idleClip = gltf.animations.find(a => a.name === 'Idle')!
const idleAction = mixer.clipAction(idleClip)
idleAction.setLoop('repeat').play()

// Crossfade to run
const runClip = gltf.animations.find(a => a.name === 'Run')!
const runAction = mixer.clipAction(runClip)
mixer.crossFade(idleAction, runAction, 0.3)

// Attach weapon to hand bone
const handBone = skinnedMesh.skeleton.getBoneByName('hand_r')!
const swordGltf = await gltfLoader.load('/models/sword.glb')
handBone.add(swordGltf.scene)

// Update in render loop
engine.onTick((dt) => {
  mixer.update(dt)
})
```

## Raycasting

```typescript
const raycaster = new Raycaster()

canvas.addEventListener('click', (e) => {
  // Convert mouse position to NDC [-1, 1]
  const rect = canvas.getBoundingClientRect()
  const x = ((e.clientX - rect.left) / rect.width) * 2 - 1
  const y = -((e.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera({ x, y }, camera)

  const hits = raycaster.intersectObjects(scene.children, true)
  if (hits.length > 0) {
    const hit = hits[0]
    console.log('Hit:', hit.object.name, 'at', hit.point, 'distance:', hit.distance)
  }
})
```

## Orbit Controls

```typescript
const controls = new OrbitControls(camera, canvas, {
  target: new Vec3(0, 0, 0),
  minDistance: 2,
  maxDistance: 100,
  enableDamping: true,
  dampingFactor: 0.1,
})

engine.onTick((dt) => {
  controls.update(dt)
})

// Programmatic control
controls.target.set(50, 50, 0) // orbit around a new point
```

## HTML Overlay

```typescript
const overlay = new HtmlOverlay(canvas)

// Add a floating label
const label = document.createElement('div')
label.textContent = 'Player 1'
label.style.color = 'white'
label.style.fontSize = '14px'
overlay.add(1, label, new Vec3(0, 0, 3))

// Track a node
overlay.attachToNode(1, characterMesh, new Vec3(0, 0, 2.5))

// Health bar with pointer events
const healthBar = document.createElement('div')
healthBar.innerHTML = '<div class="hp-fill" style="width:75%"></div>'
overlay.add(2, healthBar, new Vec3(), { pointerEvents: true })
overlay.attachToNode(2, characterMesh, new Vec3(0, 0, 2.8))

// Update each frame (engine does this automatically if overlay is registered)
engine.onTick(() => {
  overlay.syncTrackedNodes()
  overlay.update(camera, canvas.width, canvas.height)
})
```

## Render Loop

```typescript
// Register per-frame callbacks
engine.onTick((deltaTime) => {
  // Update game logic
  mixer.update(deltaTime)
  controls.update(deltaTime)

  // Move objects
  cube.position.z = Math.sin(engine.elapsed) * 2

  // The engine automatically calls renderer.render(scene, camera) after all callbacks
})

// Start the loop
engine.start()

// Or manual control
engine.stop()
engine.tick(1 / 60) // advance one frame manually
```

## Visibility and Culling

```typescript
// Hide an object (and all children)
house.visible = false

// The renderer automatically frustum-culls invisible objects
// No manual culling needed — it's always on

// Check if an object is in the camera frustum
const inView = camera.frustum.intersectsAABB(house.worldAABB)
```

## Configuration Reference

### Engine Config

```typescript
interface EngineConfig {
  backend?: 'webgpu' | 'webgl2' | 'auto'
  msaa?: 1 | 2 | 4
  pixelRatio?: number
  bloom?: BloomConfig | false
  toneMapping?: 'aces' | 'linear' | 'reinhard'
  exposure?: number
  maxBones?: number           // default 170
  maxMaterialEntries?: number // default 32
}
```

### Bloom Config

```typescript
interface BloomConfig {
  enabled?: boolean            // default true
  threshold?: number           // default 1.0
  softThreshold?: number       // default 0.5
  intensity?: number           // default 0.5
  radius?: number              // default 1.0
  mipCount?: number            // default 5
}
```

### Frame Stats

```typescript
interface FrameStats {
  fps: number
  frameTime: number           // ms
  jsTime: number              // ms
  gpuTime: number             // ms (if available)
  drawCalls: number
  triangles: number
  stateChanges: number
  culled: number
  visibleObjects: number
}
```

## Coordinate System Cheat Sheet

```
       Z (up)
       │
       │
       │
       └──────── X (right)
      /
     /
    Y (forward/north)

Right-handed: X cross Y = Z

camera.lookAt defaults:
  - Position: (5, -10, 5)
  - Target: (0, 0, 0)
  - Up: (0, 0, 1)

Euler rotation order: ZXY
  - Yaw: rotation around Z (turn left/right)
  - Pitch: rotation around X (look up/down)
  - Roll: rotation around Y (tilt)
```

## Error Handling

```typescript
// Engine creation can fail (no WebGL2/WebGPU support)
try {
  const engine = await Engine.create(canvas)
} catch (e) {
  if (e instanceof UnsupportedBackendError) {
    console.error('WebGL2/WebGPU not supported')
    // Show fallback UI
  }
}

// Asset loading errors
try {
  const gltf = await gltfLoader.load('/models/missing.glb')
} catch (e) {
  console.error('Failed to load model:', e.message)
}
```

## Cleanup

```typescript
// Remove specific objects
scene.remove(cube)
cube.geometry.dispose()
cube.material.dispose()

// Destroy everything
engine.destroy()
// Releases all GPU resources, cancels render loop, removes event listeners
```
