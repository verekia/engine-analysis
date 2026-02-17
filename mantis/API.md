# Public API

## Quick Start

```typescript
import { createEngine, LambertMaterial, createSphere, OrbitControls } from 'mantis'

const engine = await createEngine({
  canvas: document.getElementById('canvas') as HTMLCanvasElement,
  antialias: true,
  shadows: true,
  bloom: true,
})

const { scene } = engine

// Lights
scene.createAmbientLight({ color: [0.3, 0.3, 0.4], intensity: 1.0 })
scene.createDirectionalLight({
  direction: [-0.5, -0.3, -0.8],
  color: [1, 0.95, 0.9],
  intensity: 1.0,
  castShadow: true,
})

// Camera
const camera = scene.createCamera({ fov: 60, near: 0.1, far: 500 })
camera.position = [10, -20, 15]

// Controls
const controls = new OrbitControls(engine, camera)

// A glowing sphere
const material = new LambertMaterial({
  palette: [
    { color: [0, 0.8, 0.7], emissive: [0, 0.8, 0.7], emissiveIntensity: 0.7 },
  ],
  shadows: true,
})

const geometry = scene.createGeometry(createSphere({ radius: 1 }))
const mesh = scene.createMesh(geometry, material)
mesh.position = [0, 0, 1]

// Frame loop
engine.onFrame((dt) => {
  controls.update(dt)
})

engine.start()
```

## React Quick Start

```tsx
import { Canvas, OrbitControls, useFrame } from 'mantis/react'
import { Suspense, useRef } from 'react'

const GlowingSphere = () => {
  const ref = useRef()

  useFrame((dt, elapsed) => {
    ref.current.position = [0, 0, 1 + Math.sin(elapsed) * 0.5]
  })

  return (
    <mesh ref={ref}>
      <sphereGeometry args={{ radius: 1 }} />
      <lambertMaterial
        shadows
        palette={[
          { color: [0, 0.8, 0.7], emissive: [0, 0.8, 0.7], emissiveIntensity: 0.7 },
        ]}
      />
    </mesh>
  )
}

const App = () => (
  <Canvas antialias shadows bloom camera={{ fov: 60, position: [10, -20, 15] }}>
    <ambientLight color={[0.3, 0.3, 0.4]} intensity={1.0} />
    <directionalLight direction={[-0.5, -0.3, -0.8]} castShadow />
    <OrbitControls />
    <GlowingSphere />
  </Canvas>
)
```

---

## Engine

### `createEngine(options): Promise<Engine>`

Creates the Mantis engine, auto-detecting WebGPU or falling back to WebGL2.

```typescript
interface EngineOptions {
  canvas: HTMLCanvasElement
  antialias?: boolean        // default true (4× MSAA)
  shadows?: boolean          // default false
  bloom?: boolean            // default false
  bloomIntensity?: number    // default 0.5
  bloomLevels?: number       // default 5
  pixelRatio?: number        // default devicePixelRatio
}
```

### `Engine`

```typescript
interface Engine {
  readonly scene: Scene
  readonly device: GALDevice
  readonly backend: 'webgpu' | 'webgl2'
  bloomIntensity: number
  bloomEnabled: boolean

  start(): void
  stop(): void
  onFrame(callback: (dt: number, elapsed: number) => void): () => void
  onProfile(callback: (stats: ProfileStats) => void): () => void
  raycast(ray: Ray, options?: RaycastOptions): RaycastHit | null
  raycastAll(ray: Ray, options?: RaycastOptions): RaycastHit[]
  createHtmlOverlay(options: OverlayOptions): HtmlOverlay
  dispose(): void
}
```

---

## Scene

### `Scene`

```typescript
interface Scene {
  readonly root: Group

  createMesh(geometry: Geometry, material: Material): Mesh
  createGroup(): Group
  createCamera(options: CameraOptions): Camera
  createDirectionalLight(options: DirectionalLightOptions): DirectionalLight
  createAmbientLight(options: AmbientLightOptions): AmbientLight
  createGeometry(data: GeometryData): Geometry

  findByName(name: string): Node | undefined
  dispose(): void
}
```

---

## Node / Mesh / Group

### `Node` (base class)

```typescript
interface Node {
  name: string
  position: [number, number, number]   // Z-up: [x, y, z]
  rotation: [number, number, number, number]  // quaternion [x, y, z, w]
  scale: [number, number, number] | number    // vec3 or uniform
  visible: boolean

  readonly worldMatrix: Float32Array   // read-only, 16 floats

  add(child: Node): void
  remove(child: Node): void
  lookAt(target: Vec3): void
  dispose(): void
}
```

### `Mesh`

```typescript
interface Mesh extends Node {
  geometry: Geometry
  material: Material
  castShadow: boolean     // default true
  receiveShadow: boolean  // controlled by material

  buildBVH(): void        // enable triangle-level raycasting
  raycast(ray: Ray): RaycastHit | null
}
```

### `Group`

```typescript
interface Group extends Node {
  // Pure transform container — no geometry or material
}
```

---

## Camera

```typescript
interface Camera extends Node {
  fov: number              // vertical field of view in degrees
  near: number
  far: number
  aspect: number           // auto-set from canvas

  readonly viewMatrix: Mat4
  readonly projectionMatrix: Mat4
  readonly vpMatrix: Mat4

  updateProjection(): void
  screenPointToRay(screenX: number, screenY: number): Ray
}
```

---

## Lights

```typescript
interface DirectionalLight extends Node {
  direction: [number, number, number]
  color: [number, number, number]
  intensity: number
  castShadow: boolean
  shadowConfig: ShadowConfig
}

interface AmbientLight extends Node {
  color: [number, number, number]
  intensity: number
}

interface ShadowConfig {
  cascades?: number         // default 3
  resolution?: number       // default 1024 per cascade
  bias?: number             // default 0.002
  normalBias?: number       // default 0.02
  far?: number              // default 200
  lambda?: number           // default 0.7 (cascade split blend)
}
```

---

## Materials

### `BasicMaterial`

```typescript
interface BasicMaterialOptions {
  color?: [number, number, number]       // default [1, 1, 1]
  colorMap?: GALTexture
  opacity?: number                        // default 1.0
  transparent?: boolean                   // default false
  side?: 'front' | 'back' | 'both'      // default 'front'
  palette?: PaletteEntry[]
}
```

### `LambertMaterial`

```typescript
interface LambertMaterialOptions extends BasicMaterialOptions {
  aoMap?: GALTexture
  shadows?: boolean                       // default true
}
```

### `PaletteEntry`

```typescript
interface PaletteEntry {
  color: [number, number, number]
  opacity?: number                        // default 1.0
  emissive?: [number, number, number]     // default [0, 0, 0]
  emissiveIntensity?: number              // default 0
}
```

### Material Methods

```typescript
interface Material {
  setPaletteEntry(index: number, entry: PaletteEntry): void
  dispose(): void
}
```

---

## Geometry Generators

```typescript
const createPlane = (opts?: { width?: number, height?: number, widthSegments?: number, heightSegments?: number }): GeometryData
const createBox = (opts?: { width?: number, height?: number, depth?: number, widthSegments?: number, heightSegments?: number, depthSegments?: number }): GeometryData
const createSphere = (opts?: { radius?: number, widthSegments?: number, heightSegments?: number }): GeometryData
const createCone = (opts?: { radius?: number, height?: number, radialSegments?: number, heightSegments?: number, openEnded?: boolean }): GeometryData
const createCylinder = (opts?: { radiusTop?: number, radiusBottom?: number, height?: number, radialSegments?: number, heightSegments?: number, openEnded?: boolean }): GeometryData
const createCapsule = (opts?: { radius?: number, height?: number, radialSegments?: number, heightSegments?: number, capSegments?: number }): GeometryData
const createCircle = (opts?: { radius?: number, segments?: number, thetaStart?: number, thetaLength?: number }): GeometryData
```

---

## Animation

### `AnimationMixer`

```typescript
interface AnimationMixer {
  play(clip: AnimationClip, options?: PlayOptions): ActiveAction
  crossFadeTo(clip: AnimationClip, duration: number): ActiveAction
  stop(action: ActiveAction): void
  stopAll(): void
  update(dt: number): void
}

interface PlayOptions {
  speed?: number      // default 1.0
  loop?: boolean      // default true
  weight?: number     // default 1.0
  fadeIn?: number     // seconds
}

interface ActiveAction {
  clip: AnimationClip
  time: number
  speed: number
  weight: number
  loop: boolean
  paused: boolean
  onFinished?: () => void
}
```

### Bone Attachment

```typescript
interface Skeleton {
  attach(mesh: Mesh, boneName: string, offset?: { position?: Vec3, rotation?: Quat }): void
  detach(mesh: Mesh): void
}
```

---

## Asset Loading

### `loadGLTF`

```typescript
const loadGLTF = async (engine: Engine, url: string, options?: GLTFOptions): Promise<GLTFResult>

interface GLTFOptions {
  material?: Material        // override material for all meshes
  buildBVH?: boolean         // build mesh BVH for raycasting
  precompileShaders?: boolean
  scale?: number             // uniform scale applied at load
}

interface GLTFResult {
  scene: Group               // root of the loaded scene hierarchy
  meshes: Mesh[]
  skeleton?: Skeleton
  animations?: AnimationClip[]
  dispose(): void
}
```

### `loadTexture`

```typescript
const loadTexture = async (engine: Engine, url: string): Promise<GALTexture>
// Supports: PNG, JPEG, WebP, KTX2 (auto-detected by extension or content type)
```

---

## Raycasting

```typescript
interface Ray {
  origin: Vec3
  direction: Vec3     // normalized
}

interface RaycastOptions {
  maxDistance?: number
  filter?: (object: Mesh) => boolean
}

interface RaycastHit {
  object: Mesh
  point: Vec3           // world-space hit position
  normal: Vec3          // world-space surface normal
  distance: number
  faceIndex: number     // triangle index
  uv: Vec2              // barycentric UV
}
```

---

## Orbit Controls

```typescript
interface OrbitControls {
  target: Vec3
  distance: number
  azimuth: number
  elevation: number
  enabled: boolean

  setTarget(target: Vec3, opts?: AnimateOptions): void
  setDistance(distance: number, opts?: AnimateOptions): void
  setAzimuth(azimuth: number, opts?: AnimateOptions): void
  setElevation(elevation: number, opts?: AnimateOptions): void
  update(dt: number): void
  dispose(): void
}

interface AnimateOptions {
  animate?: boolean
  duration?: number     // seconds
}
```

---

## HTML Overlay

```typescript
interface HtmlOverlay {
  worldPosition: Vec3
  content: HTMLElement
  visible: boolean
  center: boolean
  occlude: boolean | 'raycast'

  setContent(element: HTMLElement): void
  dispose(): void
}
```

---

## React Bindings

### Components

```tsx
<Canvas />
<mesh />
<group />
<directionalLight />
<ambientLight />
<primitive object={node} />
<Html />
<OrbitControls />

// Geometry (children of <mesh>)
<planeGeometry args={{}} />
<boxGeometry args={{}} />
<sphereGeometry args={{}} />
<coneGeometry args={{}} />
<cylinderGeometry args={{}} />
<capsuleGeometry args={{}} />
<circleGeometry args={{}} />

// Materials (children of <mesh>)
<basicMaterial />
<lambertMaterial />
```

### Hooks

```typescript
const useFrame: (callback: (dt: number, elapsed: number) => void) => void
const useEngine: () => Engine
const useScene: () => Scene
const useLoader: <T>(loader: Loader<T>, url: string) => T
const useRaycast: () => (screenX: number, screenY: number) => RaycastHit | null
```

---

## Complete Example: Game Scene

```tsx
import { Canvas, OrbitControls, Html, useLoader, useFrame } from 'mantis/react'
import { GLTFLoader } from 'mantis'
import { Suspense, useRef, useState } from 'react'

const Character = ({ position }: { position: [number, number, number] }) => {
  const gltf = useLoader(GLTFLoader, '/models/character.glb')
  const ref = useRef()

  useFrame((dt) => {
    // Animation update handled by mixer registered in gltf
  })

  return (
    <primitive ref={ref} object={gltf.scene} position={position}>
      <Html position={[0, 0, 2.5]} center>
        <div className="nametag">Player</div>
      </Html>
    </primitive>
  )
}

const Ground = () => (
  <mesh position={[0, 0, 0]} receiveShadow>
    <planeGeometry args={{ width: 200, height: 200 }} />
    <lambertMaterial color={[0.3, 0.6, 0.2]} shadows />
  </mesh>
)

const GlowingCrystal = () => {
  const [hovered, setHovered] = useState(false)

  return (
    <mesh
      position={[5, 3, 1]}
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
      onClick={() => console.log('Crystal clicked!')}
    >
      <coneGeometry args={{ radius: 0.3, height: 1.5, radialSegments: 6 }} />
      <lambertMaterial
        palette={[{
          color: [0, 0.8, 0.7],
          emissive: [0, 0.8, 0.7],
          emissiveIntensity: hovered ? 1.0 : 0.5,
        }]}
        shadows
      />
    </mesh>
  )
}

const App = () => (
  <Canvas antialias shadows bloom bloomIntensity={0.6}
    camera={{ fov: 60, near: 0.1, far: 500, position: [15, -25, 20] }}
    style={{ width: '100vw', height: '100vh' }}
  >
    <ambientLight color={[0.3, 0.3, 0.4]} intensity={1.0} />
    <directionalLight
      direction={[-0.5, -0.3, -0.8]}
      color={[1, 0.95, 0.9]}
      castShadow
    />
    <OrbitControls target={[0, 0, 0]} distance={20} enableDamping />

    <Ground />
    <GlowingCrystal />

    <Suspense fallback={null}>
      <Character position={[0, 0, 0]} />
    </Suspense>
  </Canvas>
)
```
