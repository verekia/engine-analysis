# 15 — Public API

This document defines the complete public API surface of Wren. The API is designed to be minimal, discoverable, and chainable where appropriate.

## Packages

| Package | Description | Peer Dependencies |
|---------|-------------|------------------|
| `wren` | Core engine | None |
| `wren-react` | React bindings | `react`, `react-reconciler` |

## Core API (`wren`)

### Initialization

```ts
import {
  createDevice,
  createRenderer,
  createScene,
  createPerspectiveCamera,
  createOrthographicCamera,
} from 'wren'

// Create GPU device (auto-detects WebGPU/WebGL2)
const device = await createDevice(canvas: HTMLCanvasElement, options?: {
  preferBackend?: 'webgpu' | 'webgl2'
  antialias?: boolean          // Default: true (4x MSAA)
  powerPreference?: 'high-performance' | 'low-power'
}): Promise<WrenDevice>

// Create renderer
const renderer = createRenderer(device: WrenDevice, options?: {
  shadows?: boolean | ShadowConfig
  bloom?: boolean | BloomConfig
  clearColor?: [number, number, number]
}): WrenRenderer

// Create scene
const scene = createScene(): Scene

// Create camera
const camera = createPerspectiveCamera(options?: {
  fov?: number       // Default: 60 (degrees)
  near?: number      // Default: 0.1
  far?: number       // Default: 1000
  aspect?: number    // Default: auto from canvas
}): CameraNode

const orthoCamera = createOrthographicCamera(options?: {
  left?: number, right?: number, top?: number, bottom?: number
  near?: number, far?: number
}): CameraNode
```

### Render Loop

```ts
// Manual render loop
const animate = () => {
  requestAnimationFrame(animate)
  controls.update(clock.getDelta())
  renderer.render(scene, camera)
}
animate()

// Or use built-in loop
renderer.setAnimationLoop((dt: number) => {
  controls.update(dt)
})
```

### Scene Graph

```ts
import {
  createGroup,
  createMesh,
  createSkinnedMesh,
  addChild,
  removeChild,
  setPosition,
  setRotation,
  setScale,
  setRotationFromEuler,
  lookAt,
  markDirty,
  getWorldPosition,
  traverse,
} from 'wren'

// Create nodes
const group = createGroup(name?: string): GroupNode
const mesh = createMesh(geometry: Geometry, material: Material): MeshNode
const skinnedMesh = createSkinnedMesh(geometry: Geometry, material: Material, skeleton: Skeleton): SkinnedMeshNode

// Hierarchy
addChild(parent: SceneNode, child: SceneNode): void
removeChild(parent: SceneNode, child: SceneNode): void
scene.add(node: SceneNode): void       // Shorthand for addChild(scene.root, node)
scene.remove(node: SceneNode): void

// Transforms (z-up, right-handed)
setPosition(node: SceneNode, x: number, y: number, z: number): void
setRotation(node: SceneNode, x: number, y: number, z: number, w: number): void  // Quaternion
setRotationFromEuler(node: SceneNode, x: number, y: number, z: number, order?: string): void
setScale(node: SceneNode, x: number, y: number, z: number): void
lookAt(node: SceneNode, target: Float32Array | [number, number, number], up?: Float32Array): void
markDirty(node: SceneNode): void
getWorldPosition(node: SceneNode): Float32Array  // Returns [x, y, z]

// Traversal
traverse(node: SceneNode, callback: (node: SceneNode) => void): void
scene.getNodeByName(name: string): SceneNode | null
```

### Geometry

```ts
import {
  createPlaneGeometry,
  createBoxGeometry,
  createSphereGeometry,
  createConeGeometry,
  createCylinderGeometry,
  createCapsuleGeometry,
  createCircleGeometry,
  createGeometry,
  disposeGeometry,
} from 'wren'

// Parametric primitives
createPlaneGeometry(options?: { width?, height?, widthSegments?, heightSegments? }): Geometry
createBoxGeometry(options?: { width?, height?, depth?, widthSegments?, heightSegments?, depthSegments? }): Geometry
createSphereGeometry(options?: { radius?, widthSegments?, heightSegments?, phiStart?, phiLength?, thetaStart?, thetaLength? }): Geometry
createConeGeometry(options?: { radius?, height?, radialSegments?, heightSegments?, openEnded? }): Geometry
createCylinderGeometry(options?: { radiusTop?, radiusBottom?, height?, radialSegments?, heightSegments?, openEnded? }): Geometry
createCapsuleGeometry(options?: { radius?, height?, capSegments?, radialSegments? }): Geometry
createCircleGeometry(options?: { radius?, segments?, thetaStart?, thetaLength? }): Geometry

// Custom geometry from typed arrays
createGeometry(desc: {
  position: Float32Array
  normal?: Float32Array
  uv?: Float32Array
  uv2?: Float32Array
  color?: Float32Array
  _materialindex?: Uint8Array
  joints?: Uint8Array
  weights?: Float32Array
  index?: Uint16Array | Uint32Array
}): Geometry

disposeGeometry(geometry: Geometry, device: WrenDevice): void
```

### Materials

```ts
import {
  createBasicMaterial,
  createLambertMaterial,
} from 'wren'

createBasicMaterial(options?: {
  color?: [number, number, number]
  opacity?: number
  map?: WrenTexture
  aoMap?: WrenTexture
  aoMapIntensity?: number
  vertexColors?: boolean
  transparent?: boolean
  side?: 'front' | 'back' | 'double'
  depthWrite?: boolean
  depthTest?: boolean
  useMaterialIndex?: boolean
  materialIndexColors?: Float32Array     // [r,g,b, r,g,b, ...] per index
  materialIndexEmissive?: Float32Array   // [r,g,b,intensity, ...] per index
}): BasicMaterial

createLambertMaterial(options?: {
  color?: [number, number, number]
  opacity?: number
  map?: WrenTexture
  aoMap?: WrenTexture
  aoMapIntensity?: number
  emissive?: [number, number, number]
  emissiveIntensity?: number
  vertexColors?: boolean
  transparent?: boolean
  side?: 'front' | 'back' | 'double'
  receiveShadow?: boolean
  useMaterialIndex?: boolean
  materialIndexColors?: Float32Array
  materialIndexEmissive?: Float32Array
}): LambertMaterial
```

### Textures

```ts
import {
  loadTexture,
  loadKTX2Texture,
  createKTX2Transcoder,
  disposeTexture,
} from 'wren'

loadTexture(url: string, device: WrenDevice, options?: {
  srgb?: boolean
  generateMipmaps?: boolean
  flipY?: boolean
  sampler?: Partial<SamplerDescriptor>
}): Promise<WrenTexture>

// KTX2/Basis (requires transcoder)
const transcoder = await createKTX2Transcoder(options?: { wasmUrl?: string }): Promise<KTX2Transcoder>
loadKTX2Texture(url: string, device: WrenDevice, transcoder: KTX2Transcoder): Promise<WrenTexture>

disposeTexture(texture: WrenTexture, device: WrenDevice): void
```

### Lights

```ts
import {
  createDirectionalLight,
  createAmbientLight,
} from 'wren'

createDirectionalLight(options?: {
  color?: [number, number, number]
  intensity?: number
  castShadow?: boolean
  shadow?: DirectionalShadowConfig
}): DirectionalLightNode

// Ambient light is set directly on the scene
scene.ambientLight = { color: new Float32Array([1, 1, 1]), intensity: 0.3 }
// Or via a node:
createAmbientLight(options?: {
  color?: [number, number, number]
  intensity?: number
}): AmbientLightNode
```

### Animation

```ts
import {
  createAnimationMixer,
} from 'wren'

const mixer = createAnimationMixer(skeleton: Skeleton): AnimationMixer

// Play an animation
const action = mixer.play(clip: AnimationClip, options?: {
  loop?: boolean
  speed?: number
  weight?: number
  startTime?: number
}): AnimationAction

// Crossfade to a new animation
const newAction = mixer.crossFadeTo(clip: AnimationClip, duration: number, options?: PlayOptions): AnimationAction

// Stop
mixer.stop(action: AnimationAction): void
mixer.stopAll(): void

// Update each frame
mixer.update(deltaTime: number): void
```

### GLTF Loading

```ts
import {
  loadGLTF,
  createDracoDecoder,
  createKTX2Transcoder,
} from 'wren'

// Optional: pre-initialize decoders
const draco = await createDracoDecoder(options?: { wasmUrl?: string }): Promise<DracoDecoder>
const ktx2 = await createKTX2Transcoder(options?: { wasmUrl?: string }): Promise<KTX2Transcoder>

const gltf = await loadGLTF(url: string, device: WrenDevice, options?: {
  draco?: DracoDecoder
  ktx2?: KTX2Transcoder
}): Promise<GLTFResult>

// GLTFResult contains:
gltf.scenes      // GroupNode[] — scene hierarchies
gltf.meshes       // MeshNode[]
gltf.skinnedMeshes // SkinnedMeshNode[]
gltf.materials    // Material[]
gltf.textures     // WrenTexture[]
gltf.animations   // AnimationClip[]
gltf.skeletons    // Skeleton[]

// Add to scene
scene.add(gltf.scenes[0])
```

### Raycasting

```ts
import { createRaycaster } from 'wren'

const raycaster = createRaycaster(): Raycaster

// From mouse coordinates
raycaster.setFromCamera(
  coords: { x: number, y: number },  // Normalized device coords (-1 to 1)
  camera: CameraNode
): void

// Or set ray directly
raycaster.ray.origin.set([0, 0, 0])
raycaster.ray.direction.set([0, 1, 0])

// Intersect
const hits = raycaster.intersectObject(mesh: MeshNode, recursive?: boolean): RayHit[]
const hits = raycaster.intersectObjects(meshes: MeshNode[]): RayHit[]

// RayHit:
hit.t                  // Distance
hit.point              // World-space intersection point
hit.normal             // Interpolated normal
hit.triangleIndex      // Triangle index
hit.object             // The mesh that was hit
hit.u, hit.v           // Barycentric coordinates
```

### Controls

```ts
import { createOrbitControls } from 'wren'

const controls = createOrbitControls(camera: CameraNode, canvas: HTMLCanvasElement): OrbitControls

// Configuration
controls.target = new Float32Array([0, 0, 0])
controls.minDistance = 1
controls.maxDistance = 100
controls.dampingFactor = 0.1
controls.enablePan = true

// Update each frame
controls.update(deltaTime: number): void

// Cleanup
controls.dispose(): void
```

### HTML Overlay

```ts
import { createHtmlOverlay } from 'wren'

const overlay = createHtmlOverlay(canvas: HTMLCanvasElement): HtmlOverlay

const handle = overlay.add(element: HTMLElement, options: {
  position?: Float32Array | [number, number, number]
  node?: SceneNode
  bone?: { skeleton: Skeleton, boneName: string }
  center?: boolean
  occlude?: boolean
  zIndexRange?: [number, number]
  distanceFactor?: number
}): OverlayHandle

overlay.remove(handle: OverlayHandle): void

// Update each frame (reprojects all overlays)
overlay.update(camera: CameraNode): void

overlay.dispose(): void
```

### Renderer Stats

```ts
renderer.onFrameStats = (stats: FrameStats) => {
  stats.drawCalls       // Number of draw calls submitted
  stats.triangles       // Total triangles rendered
  stats.stateChanges    // Pipeline + bind group changes
  stats.cullRejected    // Objects culled by frustum
  stats.cullPassed      // Objects that passed culling
  stats.sortTimeMs      // Time spent sorting
  stats.submitTimeMs    // Time spent submitting draws
  stats.gpuTimeMs       // GPU time (if timer query available)
}
```

### Renderer Configuration

```ts
renderer.setSize(width: number, height: number): void
renderer.setPixelRatio(ratio: number): void
renderer.setClearColor(r: number, g: number, b: number): void

renderer.shadowConfig = { cascades: 3, mapSize: 1024 }
renderer.bloomConfig = { enabled: true, intensity: 0.5, radius: 0.85, levels: 5 }
```

### Disposal

```ts
// Dispose individual resources
disposeGeometry(geometry, device)
disposeTexture(texture, device)

// Dispose entire scene (recursively disposes all GPU resources)
scene.dispose(device)

// Dispose renderer and device
renderer.dispose()
device.destroy()
```

## React API (`wren-react`)

```tsx
import { WrenCanvas, useFrame, useWren, useLoader, useOverlay } from 'wren-react'

// Root component
<WrenCanvas
  camera={{ position: [5, 5, 5], fov: 60 }}
  shadows
  bloom={{ intensity: 0.5 }}
  antialias
  preferBackend="webgpu"
  onCreated={(state) => console.log('Ready:', state.device.backend)}
  style={{ width: '100vw', height: '100vh' }}
>
  <Scene />
</WrenCanvas>

// Hooks
useFrame((state, delta) => { /* runs every frame */ })
const { device, renderer, scene, camera } = useWren()
const gltf = useLoader(loadGLTF, '/model.glb')  // Suspense-compatible
const overlayRef = useOverlay({ node: meshRef.current })

// JSX Elements
<group position={[0, 0, 0]} rotation={[0, 0, Math.PI/4]} scale={2}>
  <mesh position={[0, 0, 1]} castShadow receiveShadow onClick={handleClick}>
    <boxGeometry width={2} height={2} depth={2} />
    <lambertMaterial
      color={[0.2, 0.6, 1.0]}
      emissive={[0.0, 0.3, 0.5]}
      emissiveIntensity={0.5}
    />
  </mesh>
</group>

<directionalLight position={[10, 10, 20]} intensity={1} castShadow />
<ambientLight intensity={0.3} />

// Primitives (inject pre-built scene nodes)
<primitive object={gltf.scenes[0]} />

// HTML overlay in React
<mesh ref={meshRef}>...</mesh>
<htmlOverlay node={meshRef} center distanceFactor={100}>
  <div className="label">Player 1</div>
</htmlOverlay>
```

## Complete Minimal Example

```ts
import {
  createDevice,
  createRenderer,
  createScene,
  createPerspectiveCamera,
  createMesh,
  createBoxGeometry,
  createLambertMaterial,
  createDirectionalLight,
  createOrbitControls,
  setPosition,
  addChild,
} from 'wren'

const canvas = document.querySelector('canvas')!

const main = async () => {
  const device = await createDevice(canvas)
  const renderer = createRenderer(device, { shadows: true, bloom: true })
  const scene = createScene()
  const camera = createPerspectiveCamera({ fov: 60 })
  setPosition(camera, 5, 5, 5)

  // Light
  const light = createDirectionalLight({ intensity: 1, castShadow: true })
  setPosition(light, 10, 10, 20)
  scene.add(light)

  // Box
  const box = createMesh(
    createBoxGeometry({ width: 2, height: 2, depth: 2 }),
    createLambertMaterial({ color: [0.2, 0.6, 1.0] })
  )
  setPosition(box, 0, 0, 1)
  box.castShadow = true
  scene.add(box)

  // Ground
  const ground = createMesh(
    createPlaneGeometry({ width: 20, height: 20 }),
    createLambertMaterial({ color: [0.8, 0.8, 0.8] })
  )
  ground.receiveShadow = true
  scene.add(ground)

  // Controls
  const controls = createOrbitControls(camera, canvas)

  // Render loop
  renderer.setAnimationLoop((dt) => {
    controls.update(dt)
    box.rotation[2] += dt  // Spin around Z
    markDirty(box)
  })
}

main()
```
