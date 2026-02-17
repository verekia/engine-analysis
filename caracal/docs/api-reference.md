# API Reference

## Engine Creation

```typescript
import {
  createEngine,
  createScene,
  createPerspectiveCamera,
  createOrthographicCamera,
  createMesh,
  createGroup,
  createDirectionalLight,
  createAmbientLight,
  createBasicMaterial,
  createLambertMaterial,
  createMaterialIndexMap,
  createBoxGeometry,
  createSphereGeometry,
  createPlaneGeometry,
  createCylinderGeometry,
  createConeGeometry,
  createCapsuleGeometry,
  createCircleGeometry,
  createRaycaster,
  createOrbitControls,
  createHTMLOverlaySystem,
  createAnimationMixer,
  createBoneAttachment,
  loadGLTF,
  loadTexture,
  createDracoDecoder,
  createBasisTranscoder,
} from 'caracal'
```

## Core

### createEngine

```typescript
const engine = createEngine({
  canvas: HTMLCanvasElement,           // Required: target canvas
  backend?: 'auto' | 'webgl2' | 'webgpu', // Default: 'auto'
  antialias?: boolean,                 // Default: true (4x MSAA)
  pixelRatio?: number,                 // Default: window.devicePixelRatio
  powerPreference?: 'default' | 'high-performance' | 'low-power',
})

// Engine API
engine.device: GPUDevice              // Backend device
engine.canvas: HTMLCanvasElement      // The canvas element
engine.width: number                  // Canvas width in pixels
engine.height: number                 // Canvas height in pixels

engine.setSize(width: number, height: number): void
engine.render(scene: Scene, camera: Camera): void
engine.dispose(): void

// Frame loop
engine.onTick(callback: (dt: number) => void): () => void  // Returns unsubscribe function
engine.start(): void                  // Start the frame loop
engine.stop(): void                   // Stop the frame loop
```

### Scene

```typescript
const scene = createScene()

scene.backgroundColor: Vec3           // Clear color [r, g, b]
scene.add(node: Node): void
scene.remove(node: Node): void
scene.traverse(callback: (node: Node) => void): void
scene.find(name: string): Node | null
```

### Camera

```typescript
const camera = createPerspectiveCamera({
  fov?: number,                        // Vertical FOV in degrees (default: 60)
  aspect?: number,                     // Aspect ratio (default: canvas aspect)
  near?: number,                       // Near plane (default: 0.1)
  far?: number,                        // Far plane (default: 1000)
})

const ortho = createOrthographicCamera({
  left: number,
  right: number,
  bottom: number,
  top: number,
  near?: number,
  far?: number,
})

// Camera is a Node — has position, rotation, etc.
camera.position.set(0, -10, 5)
camera.lookAt(new Vec3(0, 0, 0))
```

## Scene Graph

### Node (Base)

All scene objects share these properties:

```typescript
node.name: string
node.visible: boolean                  // Default: true
node.frustumCulled: boolean            // Default: true
node.castShadow: boolean              // Default: false
node.receiveShadow: boolean           // Default: false

// Transform
node.position: Vec3                    // Local position
node.rotation: Quat                    // Local rotation (quaternion)
node.scale: Vec3                       // Local scale (default: [1,1,1])
node.worldMatrix: Mat4                 // Read-only world transform

// Hierarchy
node.parent: Node | null
node.children: readonly Node[]
node.add(child: Node): void
node.remove(child: Node): void
node.traverse(fn: (node: Node) => void): void
node.find(name: string): Node | null
node.lookAt(target: Vec3): void
```

### Mesh

```typescript
const mesh = createMesh(geometry: Geometry, material: Material)

mesh.geometry: Geometry
mesh.material: Material
// Inherits all Node properties
```

### Group

```typescript
const group = createGroup()

// A group is just a Node with no geometry/material
// Used for hierarchical transforms
group.add(child1)
group.add(child2)
group.position.set(10, 0, 0) // Moves all children
```

## Materials

### BasicMaterial

```typescript
const mat = createBasicMaterial({
  color?: [number, number, number],    // Default: [1, 1, 1]
  map?: GPUTextureHandle,              // Color texture
  aoMap?: GPUTextureHandle,            // AO texture
  vertexColors?: boolean,              // Default: false
  materialIndexColors?: MaterialIndexMap,
  opacity?: number,                    // Default: 1.0
  transparent?: boolean,               // Default: false
  side?: 'front' | 'back' | 'both',   // Default: 'front'
  depthTest?: boolean,                 // Default: true
  depthWrite?: boolean,                // Default: true (false if transparent)
})

// All properties are mutable after creation
mat.color = [1, 0, 0]
mat.opacity = 0.5
```

### LambertMaterial

```typescript
const mat = createLambertMaterial({
  // All BasicMaterial properties, plus:
  emissive?: [number, number, number], // Default: [0, 0, 0]
  emissiveIntensity?: number,          // Default: 0.0
  aoMapIntensity?: number,             // Default: 1.0
})
```

### MaterialIndexMap

```typescript
const indexMap = createMaterialIndexMap({
  [index: number]: {
    color: [number, number, number],
    emissive?: [number, number, number],
    emissiveIntensity?: number,
    opacity?: number,
  }
})

// Apply to a material:
const mat = createLambertMaterial({ materialIndexColors: indexMap })

// Update at runtime:
indexMap.setColor(2, [1, 0, 0])
indexMap.setEmissive(2, [1, 0, 0], 0.8)
```

## Geometry

All parametric geometries return a `Geometry` with `position`, `normal`, and `uv` attributes.

```typescript
const plane = createPlaneGeometry({
  width?: number,              // Default: 1
  height?: number,             // Default: 1
  widthSegments?: number,      // Default: 1
  heightSegments?: number,     // Default: 1
})

const box = createBoxGeometry({
  width?: number,              // Default: 1 (X)
  height?: number,             // Default: 1 (Y)
  depth?: number,              // Default: 1 (Z)
  widthSegments?: number,
  heightSegments?: number,
  depthSegments?: number,
})

const sphere = createSphereGeometry({
  radius?: number,             // Default: 1
  widthSegments?: number,      // Default: 32
  heightSegments?: number,     // Default: 16
  phiStart?: number,           // Default: 0
  phiLength?: number,          // Default: Math.PI * 2
  thetaStart?: number,         // Default: 0
  thetaLength?: number,        // Default: Math.PI
})

const cylinder = createCylinderGeometry({
  radiusTop?: number,          // Default: 1
  radiusBottom?: number,       // Default: 1
  height?: number,             // Default: 1
  radialSegments?: number,     // Default: 16
  heightSegments?: number,     // Default: 1
  openEnded?: boolean,         // Default: false
})

const cone = createConeGeometry({
  radius?: number,             // Default: 1
  height?: number,             // Default: 1
  radialSegments?: number,     // Default: 16
  heightSegments?: number,     // Default: 1
  openEnded?: boolean,         // Default: false
})

const capsule = createCapsuleGeometry({
  radius?: number,             // Default: 0.5
  height?: number,             // Default: 1 (total including caps)
  capSegments?: number,        // Default: 8
  radialSegments?: number,     // Default: 16
})

const circle = createCircleGeometry({
  radius?: number,             // Default: 1
  segments?: number,           // Default: 32
  thetaStart?: number,         // Default: 0
  thetaLength?: number,        // Default: Math.PI * 2
})

// Geometry API
geometry.attributes: Map<string, BufferAttribute>
geometry.index: BufferAttribute | null
geometry.computeBoundingBox(): AABB
geometry.computeBoundingSphere(): Sphere
geometry.buildBVH(): void             // Pre-build BVH for raycasting
geometry.upload(device: GPUDevice): void
geometry.dispose(device: GPUDevice): void
```

## Lights

```typescript
const ambient = createAmbientLight({
  color?: [number, number, number],    // Default: [1, 1, 1]
  intensity?: number,                  // Default: 1.0
})

const directional = createDirectionalLight({
  color?: [number, number, number],    // Default: [1, 1, 1]
  intensity?: number,                  // Default: 1.0
  castShadow?: boolean,               // Default: false
  shadowMapSize?: number,             // Default: 2048
  shadowBias?: number,                // Default: 0.001
  shadowNormalBias?: number,          // Default: 0.02
  shadowCascades?: number,            // Default: 3
  shadowDistance?: number,            // Default: 200
})

// Light is a Node — direction comes from rotation
directional.position.set(10, 10, 20) // Position affects shadow map view
directional.lookAt(new Vec3(0, 0, 0))
```

## Animation

```typescript
// Create mixer from skeleton
const mixer = createAnimationMixer(skeleton: Skeleton)

// Play an animation
const action = mixer.play(clip: AnimationClip, {
  loop?: boolean,              // Default: true
  speed?: number,              // Default: 1.0
  startTime?: number,          // Default: 0
})

// Crossfade to another animation
mixer.crossFadeTo(clip: AnimationClip, duration: number): AnimationAction

// Stop all animations
mixer.stop(): void

// Update (call each frame)
mixer.update(deltaTime: number): void

// Action control
action.time: number
action.speed: number
action.paused: boolean
action.loop: boolean
action.weight: number          // 0-1, used during crossfade

// Bone attachment
const attachment = createBoneAttachment(
  mesh: Mesh,                  // Static mesh to attach
  bone: Bone,                  // Bone to follow
  offset?: {
    position?: [number, number, number],
    rotation?: [number, number, number, number],  // Quaternion
  }
)

attachment.detach(): void
```

## Asset Loading

```typescript
// Initialize decoders (once at startup)
const draco = createDracoDecoder()
await draco.init('/wasm/draco_decoder.wasm')

const basis = createBasisTranscoder()
await basis.init('/wasm/basis_transcoder.wasm')

// Load glTF
const gltf = await loadGLTF(url: string, device: GPUDevice, {
  dracoDecoder?: DracoDecoder,
  basisTranscoder?: BasisTranscoder,
  autoUpload?: boolean,        // Default: true
  autoBuildBVH?: boolean,      // Default: false
})

gltf.scene: Group              // Root scene node
gltf.meshes: Mesh[]
gltf.skinnedMeshes: SkinnedMesh[]
gltf.skeletons: Skeleton[]
gltf.animations: AnimationClip[]

// Load texture
const texture = await loadTexture(url: string, device: GPUDevice, {
  generateMipmaps?: boolean,
  wrapS?: 'repeat' | 'clamp' | 'mirror',
  wrapT?: 'repeat' | 'clamp' | 'mirror',
  minFilter?: 'nearest' | 'linear' | 'linear-mipmap-linear',
  magFilter?: 'nearest' | 'linear',
  flipY?: boolean,
  anisotropy?: number,
})
```

## Raycasting

```typescript
const raycaster = createRaycaster()

// Set from camera + screen coordinates
raycaster.setFromCamera(camera: Camera, screenX: number, screenY: number): void

// Set manually
raycaster.set(origin: Vec3, direction: Vec3): void
raycaster.near: number         // Default: 0
raycaster.far: number          // Default: Infinity

// Cast
const hit = raycaster.intersectMesh(mesh: Mesh): RayHit | null
const hits = raycaster.intersectMeshAll(mesh: Mesh): RayHit[]
const hit = raycaster.intersectScene(scene: Scene): SceneRayHit | null
const hits = raycaster.intersectSceneAll(scene: Scene): SceneRayHit[]

// Hit result
hit.distance: number
hit.point: Vec3                // World-space intersection point
hit.triangleIndex: number
hit.barycentricU: number
hit.barycentricV: number
hit.faceNormal: Vec3
hit.mesh: Mesh                 // (SceneRayHit only)
```

## Controls

```typescript
const controls = createOrbitControls(camera: Camera, canvas: HTMLCanvasElement, {
  target?: Vec3,               // Look-at target (default: origin)
  minDistance?: number,         // Min zoom distance (default: 0.1)
  maxDistance?: number,         // Max zoom distance (default: Infinity)
  minPolarAngle?: number,      // Min vertical angle (default: 0)
  maxPolarAngle?: number,      // Max vertical angle (default: π)
  enableDamping?: boolean,     // Smooth movement (default: true)
  dampingFactor?: number,      // Damping strength (default: 0.1)
  rotateSpeed?: number,        // Rotation speed (default: 1.0)
  zoomSpeed?: number,          // Zoom speed (default: 1.0)
  panSpeed?: number,           // Pan speed (default: 1.0)
  enablePan?: boolean,         // Allow panning (default: true)
  enableZoom?: boolean,        // Allow zooming (default: true)
  enableRotate?: boolean,      // Allow rotation (default: true)
})

// Update (call each frame)
controls.update(deltaTime: number): void

// Change target
controls.target.set(x, y, z)

// Clean up
controls.dispose(): void
```

## HTML Overlay

```typescript
const overlays = createHTMLOverlaySystem(engine: Engine)

const overlay = overlays.create({
  position?: Vec3,
  node?: Node,
  offset?: Vec3,
  element: HTMLElement,
  center?: boolean,            // Default: true
  occlude?: boolean,           // Default: false
  distanceScale?: boolean,     // Default: false
  maxDistance?: number,         // Default: Infinity
  pointerEvents?: boolean,     // Default: false
})

overlay.show(): void
overlay.hide(): void
overlay.setPosition(pos: Vec3): void
overlay.setNode(node: Node): void
overlay.dispose(): void

// Update all overlays (call each frame)
overlays.update(camera: Camera): void

overlays.dispose(): void
```

## Post-Processing

```typescript
// Configured via engine or directly:
engine.postProcessing.bloom.enabled = true
engine.postProcessing.bloom.threshold = 0.0
engine.postProcessing.bloom.intensity = 1.0
engine.postProcessing.bloom.strength = 0.5
engine.postProcessing.bloom.radius = 0.85
engine.postProcessing.bloom.levels = 4

engine.postProcessing.toneMapping.enabled = true
engine.postProcessing.toneMapping.exposure = 1.0
engine.postProcessing.toneMapping.method = 'aces' // 'aces' | 'reinhard' | 'none'
```

## Math

```typescript
import { Vec3, Vec4, Mat4, Quat, Euler, AABB, Ray, Sphere } from 'caracal'

// Vec3
const v = new Vec3(x, y, z)
v.set(x, y, z): Vec3
v.add(other: Vec3): Vec3
v.sub(other: Vec3): Vec3
v.scale(s: number): Vec3
v.dot(other: Vec3): number
v.cross(other: Vec3): Vec3
v.length(): number
v.normalize(): Vec3
v.lerp(other: Vec3, t: number): Vec3
v.distanceTo(other: Vec3): number
v.clone(): Vec3
v.copy(other: Vec3): Vec3
v.toArray(): [number, number, number]

// Mat4 (column-major)
const m = new Mat4()
Mat4.identity(): Mat4
Mat4.perspective(fov, aspect, near, far): Mat4
Mat4.ortho(left, right, bottom, top, near, far): Mat4
Mat4.lookAt(eye, target, up): Mat4
m.multiply(other: Mat4): Mat4
m.invert(): Mat4
m.transpose(): Mat4
m.compose(position: Vec3, rotation: Quat, scale: Vec3): Mat4
m.decompose(): { position: Vec3, rotation: Quat, scale: Vec3 }
m.transformPoint(v: Vec3): Vec3
m.transformDirection(v: Vec3): Vec3

// Quat
const q = new Quat(x, y, z, w)
Quat.identity(): Quat
q.setFromEuler(x, y, z, order?: string): Quat    // Default order: 'ZYX'
q.setFromAxisAngle(axis: Vec3, angle: number): Quat
q.slerp(other: Quat, t: number): Quat
q.multiply(other: Quat): Quat
q.invert(): Quat
q.normalize(): Quat

// Constants (Z-up)
Vec3.UP    = new Vec3(0, 0, 1)  // Z-up
Vec3.RIGHT = new Vec3(1, 0, 0)
Vec3.FORWARD = new Vec3(0, 1, 0)
Vec3.ZERO  = new Vec3(0, 0, 0)
```

## Complete Usage Example

```typescript
import {
  createEngine, createScene, createPerspectiveCamera,
  createMesh, createGroup, createDirectionalLight, createAmbientLight,
  createLambertMaterial, createMaterialIndexMap,
  createBoxGeometry, createPlaneGeometry,
  createOrbitControls, createRaycaster,
  loadGLTF, createDracoDecoder, createBasisTranscoder,
  createAnimationMixer, createBoneAttachment,
  createHTMLOverlaySystem,
  Vec3,
} from 'caracal'

// Setup
const engine = createEngine({
  canvas: document.getElementById('canvas') as HTMLCanvasElement,
  antialias: true,
})

const scene = createScene()
scene.backgroundColor = [0.1, 0.1, 0.15]

const camera = createPerspectiveCamera({ fov: 60, near: 0.1, far: 500 })
camera.position.set(0, -15, 10)
camera.lookAt(new Vec3(0, 0, 0))

// Lights
const ambient = createAmbientLight({ color: [0.3, 0.35, 0.4] })
scene.add(ambient)

const sun = createDirectionalLight({
  color: [1, 0.95, 0.8],
  castShadow: true,
  shadowMapSize: 2048,
})
sun.position.set(20, 20, 30)
sun.lookAt(new Vec3(0, 0, 0))
scene.add(sun)

// Ground
const ground = createMesh(
  createPlaneGeometry({ width: 200, height: 200 }),
  createLambertMaterial({ color: [0.3, 0.5, 0.2] })
)
ground.receiveShadow = true
scene.add(ground)

// Material index map for character
const charColors = createMaterialIndexMap({
  0: { color: [0.9, 0.8, 0.7] },                                     // Skin
  1: { color: [0.2, 0.3, 0.8] },                                     // Clothing
  2: { color: [0, 0.8, 0.7], emissive: [0, 1, 0.9], emissiveIntensity: 0.7 }, // Glowing accents
})

// Initialize decoders
const draco = createDracoDecoder()
const basis = createBasisTranscoder()
await Promise.all([
  draco.init('/wasm/draco_decoder.wasm'),
  basis.init('/wasm/basis_transcoder.wasm'),
])

// Load character
const gltf = await loadGLTF('/models/character.glb', engine.device, {
  dracoDecoder: draco,
  basisTranscoder: basis,
})

// Apply material index colors
const charMesh = gltf.meshes[0]
charMesh.material = createLambertMaterial({ materialIndexColors: charColors })
charMesh.castShadow = true
scene.add(gltf.scene)

// Animation
const mixer = createAnimationMixer(gltf.skeletons[0])
mixer.play(gltf.animations.find(a => a.name === 'Idle')!)

// Attach sword to hand
const swordGltf = await loadGLTF('/models/sword.glb', engine.device)
const handBone = gltf.skeletons[0].bones.find(b => b.name === 'hand_R')!
createBoneAttachment(swordGltf.meshes[0], handBone)
scene.add(swordGltf.scene)

// Controls
const controls = createOrbitControls(camera, engine.canvas, { enableDamping: true })

// HTML overlay (health bar)
const overlays = createHTMLOverlaySystem(engine)
const healthDiv = document.createElement('div')
healthDiv.innerHTML = '<div style="width:60px;height:6px;background:#333;border-radius:3px"><div style="width:75%;height:100%;background:#4f4;border-radius:3px"></div></div>'
overlays.create({ node: charMesh, offset: new Vec3(0, 0, 2.2), element: healthDiv, center: true })

// Raycaster
const raycaster = createRaycaster()
engine.canvas.addEventListener('click', (e) => {
  const x = (e.clientX / engine.width) * 2 - 1
  const y = -((e.clientY / engine.height) * 2 - 1)
  raycaster.setFromCamera(camera, x, y)
  const hit = raycaster.intersectScene(scene)
  if (hit) console.log('Hit:', hit.mesh.name, 'at', hit.point)
})

// Bloom
engine.postProcessing.bloom.enabled = true
engine.postProcessing.bloom.strength = 0.5

// Render loop
engine.onTick((dt) => {
  mixer.update(dt)
  controls.update(dt)
  overlays.update(camera)
  engine.render(scene, camera)
})
engine.start()
```
