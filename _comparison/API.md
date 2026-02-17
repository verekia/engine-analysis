# API Design Comparison

This document compares API design philosophies and patterns across all 9 engine implementations: bonobo, caracal, fennec, hyena, lynx, mantis, rabbit, shark, and wren.

## Universal Agreement

All implementations agree on the following core API principles:

### TypeScript-First
All implementations are built with TypeScript:
- Full type safety throughout
- Strict mode enabled
- No semicolons (consistent across 6+ implementations that specify)
- Const arrow functions preferred (where specified)

### Canvas-Based Initialization
All implementations initialize from an HTML canvas element:
```typescript
const engine = await createEngine(canvas, options)
// or
const device = await createDevice(canvas, options)
const renderer = createRenderer(device, options)
```

### Scene Graph API
All implementations provide hierarchical scene graph:
- Nodes have position, rotation, scale
- Parent/child relationships
- Transform composition (world = parent × local)
- Add/remove children

### Render Loop
All implementations support requestAnimationFrame-based rendering:
```typescript
// Option 1: Manual loop
function animate() {
  requestAnimationFrame(animate)
  renderer.render(scene, camera)
}

// Option 2: Built-in loop with callback
engine.onFrame((deltaTime) => { ... })
// or
renderer.setAnimationLoop((deltaTime) => { ... })
```

### Resource Management
All implementations require explicit resource cleanup:
- Geometry disposal
- Texture disposal
- Material disposal
- Engine/renderer disposal
- Scene disposal

## Key Variations

### 1. Initialization Pattern

**Single Engine Class (4 implementations)**
- **Bonobo, Fennec, Rabbit, Mantis** use unified engine class
```typescript
const engine = await Engine.create(canvas, options)
// or
const engine = await createEngine(canvas, options)
```
- Engine encapsulates device + renderer + scene
- Simpler for basic use cases

**Separate Device + Renderer (5 implementations)**
- **Caracal, Hyena, Lynx, Shark, Wren** separate concerns
```typescript
const device = await createDevice(canvas, options)
const renderer = createRenderer(device, options)
const scene = createScene()
```
- More flexible, better separation of concerns
- Device can be shared across systems

### 2. Scene Creation

**Scene as Property (2 implementations)**
- **Bonobo, Mantis** provide scene as engine property
```typescript
const engine = await createEngine(canvas)
engine.scene.add(mesh)
```

**Scene as Separate Object (7 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Rabbit, Shark, Wren** create independently
```typescript
const scene = createScene()
// or
const scene = new Scene()
```

### 3. Entity/Object Creation API

**Factory Functions (5 implementations)**
- **Caracal, Fennec, Mantis, Shark, Wren** use factory functions
```typescript
const mesh = createMesh(geometry, material)
const group = createGroup()
const camera = createPerspectiveCamera(options)
const light = createDirectionalLight(options)
```
- Functional style
- Easy to tree-shake
- Stateless creation

**Class Constructors (3 implementations)**
- **Hyena, Lynx, Rabbit** use class constructors
```typescript
const mesh = new Mesh(geometry, material)
const group = new Group()
const camera = new Camera(options)
const light = new DirectionalLight(options)
```
- OOP style
- More traditional
- Can extend/override

**Mixed Approach (1 implementation)**
- **Bonobo** uses both patterns in different contexts

### 4. Material Creation

**Approach A: Material Classes/Factories (8 implementations)**
Most provide separate material types:
```typescript
const basic = createBasicMaterial({ color: [1, 0, 0] })
const lambert = createLambertMaterial({ color: [1, 0, 0], receiveShadow: true })
```

**Approach B: Unified Material (1 implementation)**
- **Bonobo** uses single material type with options
```typescript
const material = createMaterial('static' | 'skinned' | 'textured', options)
```

### 5. Material Index System

**Explicitly Featured (7 implementations)**
- **Bonobo, Fennec, Caracal, Mantis, Rabbit, Shark, Wren** highlight material index
- Allows per-vertex material assignment in single mesh
- Major feature for low-poly games

Example patterns:
```typescript
// Fennec, Rabbit
material.setMaterialIndex(0, { color: [1, 1, 1] })
material.setMaterialIndex(1, { color: [0, 0, 0] })

// Mantis
const material = new LambertMaterial({
  palette: [
    { color: [1, 1, 1] },
    { color: [0, 0, 0], emissive: [0, 1, 1], emissiveIntensity: 0.7 }
  ]
})

// Caracal
const indexMap = createMaterialIndexMap({
  0: { color: [1, 1, 1] },
  1: { color: [0, 0, 0] }
})
```

**Not Emphasized (2 implementations)**
- **Hyena, Lynx** may support but don't highlight it

### 6. Geometry Primitives

**Comprehensive Set (8 implementations)**
All provide standard primitives via functions or classes:
- Box/Cube
- Sphere
- Plane
- Cylinder
- Cone
- Capsule
- Circle

**Options Object Pattern (Universal)**
All use options objects for primitive parameters:
```typescript
createBoxGeometry({ width: 1, height: 1, depth: 1 })
createSphereGeometry({ radius: 1, widthSegments: 32, heightSegments: 16 })
```

### 7. Animation API

**AnimationMixer Pattern (7 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Wren** use mixer pattern
```typescript
const mixer = createAnimationMixer(skeleton)
const action = mixer.play(clip, options)
mixer.update(deltaTime)
```

**Crossfade Support (6 implementations)**
```typescript
mixer.crossFadeTo(newClip, duration)
// or
action.crossFadeTo(newAction, duration)
```

**Not Specified (2 implementations)**
- **Bonobo, Shark** don't detail animation API

### 8. Raycasting API

**Raycaster Class/Factory (8 implementations)**
Most provide dedicated raycaster:
```typescript
const raycaster = createRaycaster()
raycaster.setFromCamera(camera, mouseX, mouseY)
const hits = raycaster.intersectScene(scene)
// or
const hits = raycaster.intersectObjects(objects, recursive)
```

**Hit Result Structure (Consistent)**
```typescript
interface RayHit {
  object: Mesh
  point: Vec3          // world position
  distance: number
  normal: Vec3         // interpolated normal
  triangleIndex: number
  uv?: Vec2           // barycentric coords
}
```

**Not Specified (1 implementation)**
- **Bonobo** doesn't detail raycasting

### 9. Controls API

**OrbitControls Standard (9 implementations)**
All that specify controls provide orbit controls:
```typescript
const controls = createOrbitControls(camera, canvas, options)
// or
const controls = new OrbitControls(camera, canvas, options)

// Common options
{
  target: Vec3,
  minDistance: number,
  maxDistance: number,
  enableDamping: boolean,
  dampingFactor: number,
}

controls.update(deltaTime)
```

### 10. HTML Overlay API

**Overlay Manager Pattern (6 implementations)**
- **Caracal, Fennec, Mantis, Rabbit, Shark, Wren** provide HTML overlay system

Common patterns:
```typescript
const overlay = createHTMLOverlay(canvas)
// or
const overlay = createHTMLOverlaySystem(engine)

overlay.add(element, {
  position: Vec3,
  node: Node,         // track a scene node
  center: boolean,
  occlude: boolean,   // hide when behind objects
  pointerEvents: boolean
})

overlay.update(camera)
```

**Not Specified (3 implementations)**
- **Bonobo, Hyena, Lynx** don't detail overlay system

### 11. GLTF Loading

**Loader Function Pattern (7 implementations)**
```typescript
const gltf = await loadGLTF(url, device, options)

// Result structure
{
  scene: Group,           // or scenes: Group[]
  meshes: Mesh[],
  materials: Material[],
  textures: Texture[],
  animations: AnimationClip[],
  skeletons: Skeleton[]
}
```

**Decoder Support (6 implementations)**
- **Caracal, Fennec, Mantis, Rabbit, Shark, Wren** support Draco and KTX2
```typescript
const draco = await createDracoDecoder({ wasmUrl: '/draco/' })
const ktx2 = await createKTX2Transcoder({ wasmUrl: '/basis/' })

const gltf = await loadGLTF(url, device, { draco, ktx2 })
```

**Not Specified (2 implementations)**
- **Bonobo, Hyena** don't detail GLTF loading

## React Bindings

### 12. React API Availability

**Dedicated React Package (7 implementations)**
- **Bonobo, Caracal, Fennec, Hyena, Lynx, Mantis, Wren** provide React bindings
- Typically named: `<package>-react` or `<package>/react`

**React Three Fiber Style (6 implementations)**
Most React APIs follow R3F conventions:
```tsx
<Canvas shadows antialias>
  <perspectiveCamera position={[0, -10, 5]} />
  <directionalLight castShadow />
  <ambientLight intensity={0.3} />

  <mesh position={[0, 0, 1]}>
    <boxGeometry args={{ width: 1 }} />
    <lambertMaterial color={[1, 0, 0]} />
  </mesh>
</Canvas>
```

**Not Specified (2 implementations)**
- **Rabbit, Shark** don't detail React API

### 13. React Hooks

**Common Hooks (6 implementations)**
Most provide these hooks:
```typescript
useFrame((state, delta) => { /* per-frame callback */ })
useEngine() // or useThree() or useWren()
useLoader(loader, url)  // Suspense-compatible
```

**Additional Hooks (Varies)**
- `useScene()`, `useCamera()`, `useDevice()`
- `useRaycast()`
- `useOverlay()`
- `useControls()`

## Configuration Philosophy

### 14. Backend Selection

**Auto-Detection with Override (8 implementations)**
```typescript
createEngine(canvas, {
  backend: 'auto' | 'webgpu' | 'webgl2'  // default: 'auto'
})
```
- Auto tries WebGPU first, falls back to WebGL2
- Manual override for testing

**Not Specified (1 implementation)**
- **Hyena** doesn't detail backend selection

### 15. MSAA Configuration

**Enabled by Default (7 implementations)**
Most enable 4x MSAA by default:
```typescript
{
  antialias: true,    // default
  sampleCount: 4,     // or msaa: 4
}
```

**Configurable Levels (5 implementations)**
- Options: 1, 2, or 4 samples
- 4x is standard for quality/performance balance

### 16. Shadow Configuration

**Boolean or Config Object (7 implementations)**
```typescript
// Simple
{ shadows: true }

// Detailed
{
  shadows: {
    enabled: true,
    cascades: 3,
    mapSize: 1024,
    bias: 0.001,
    normalBias: 0.02
  }
}
```

### 17. Bloom Configuration

**Boolean or Config Object (6 implementations)**
```typescript
// Simple
{ bloom: true }

// Detailed
{
  bloom: {
    enabled: true,
    threshold: 1.0,
    intensity: 0.5,
    radius: 0.85,
    levels: 5
  }
}
```

## Debug and Profiling APIs

### 18. Frame Statistics

**Comprehensive Stats API (7 implementations)**
Most provide detailed frame stats:
```typescript
const stats = engine.getStats()
// or
renderer.onFrameStats((stats) => { ... })

{
  fps: number,
  frameTime: number,      // ms
  jsTime: number,         // ms
  gpuTime: number,        // ms (when available)
  drawCalls: number,
  triangles: number,
  stateChanges: number,
  culled: number,
  visibleObjects: number
}
```

**Not Specified (2 implementations)**
- **Bonobo, Shark** less detail on stats

### 19. Error Handling

**Try-Catch Pattern (3 implementations)**
- **Fennec, Rabbit, Hyena** show explicit error handling
```typescript
try {
  const engine = await createEngine(canvas)
} catch (e) {
  if (e instanceof UnsupportedBackendError) {
    // Show fallback UI
  }
}
```

**Debug Mode (2 implementations)**
- **Fennec, Caracal** support debug flag
```typescript
createEngine(canvas, { debug: true })  // verbose errors
```

## API Philosophy Statements

Where implementations explicitly state their API philosophy:

**Bonobo:**
- Explicit and predictable
- Zero-cost abstractions
- React-friendly
- Debuggable (typed arrays in devtools)

**Fennec:**
- Performance by default
- Minimal surface area
- Transparency that works (OIT)
- TypeScript-first

**Hyena:**
- Minimal abstraction
- Type safety
- Performance (zero-cost abstractions)
- Clarity (explicit over implicit)

**Mantis:**
- Simple and pragmatic
- TypeScript strict mode
- Functional API where possible

**Wren:**
- Minimal and discoverable
- Chainable where appropriate
- Explicit resource management

## Implementation Breakdown Table

| Feature | Bonobo | Caracal | Fennec | Hyena | Lynx | Mantis | Rabbit | Shark | Wren |
|---------|--------|---------|--------|-------|------|--------|--------|-------|------|
| **Init Pattern** | Engine | Device+Renderer | Engine | Device+Renderer | Device+Renderer | Engine | Engine | Device+Renderer | Device+Renderer |
| **Creation Style** | Mixed | Factory | Factory | Class | Class | Factory | Class | Factory | Factory |
| **Material Index** | Yes | Yes | Yes | - | - | Yes | Yes | Yes | Yes |
| **Animation** | - | Mixer | Mixer | Mixer | Mixer | Mixer | Mixer | - | Mixer |
| **Raycasting** | - | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| **Orbit Controls** | - | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| **HTML Overlay** | - | Yes | Yes | - | - | Yes | Yes | Yes | Yes |
| **GLTF** | - | Yes | Yes | Yes | - | Yes | Yes | Yes | Yes |
| **React** | Yes | Yes | Yes | Yes | Yes | Yes | - | - | Yes |
| **Stats API** | Basic | Yes | Yes | Yes | Yes | Yes | Yes | - | Yes |

## Package Structure

**Monorepo Pattern (3 implementations)**
- **Fennec, Caracal, Mantis** use multiple packages:
```
fennec/
├── fennec-core/
├── fennec-materials/
├── fennec-loaders/
├── fennec-animation/
├── fennec-spatial/
├── fennec-controls/
├── fennec-overlay/
└── fennec-react/
```

**Single Package (5 implementations)**
- **Bonobo, Hyena, Lynx, Rabbit, Wren** use single package
- React bindings as subpath or separate package

**Not Specified (1 implementation)**
- **Shark** doesn't detail package structure

## Recommendations for Cherry-Picking

### For Simplicity:
- Use **single engine class** (Bonobo, Fennec, Mantis, Rabbit)
- **Factory functions** for object creation (5/9 use this)
- **Boolean config** for shadows/bloom
- **Auto backend detection**

### For Flexibility:
- Separate **device + renderer** (5/9 use this)
- **Class constructors** for extensibility
- **Detailed config objects**
- Multiple packages for tree-shaking

### For Low-Poly Games:
- Emphasize **material index system** (7/9 feature this)
- Provide **comprehensive primitives** (universal)
- Include **GLTF with Draco** support (6/9 have this)

### For React Development:
- Follow **R3F conventions** (6/9 do this)
- Provide **useFrame hook** (universal in React bindings)
- Support **Suspense** for loading (4/9 mention this)
- Offer **declarative JSX** for scene construction

### For Professional Use:
- Include **comprehensive stats API** (7/9 have this)
- Provide **error handling examples** (3/9 show this)
- Support **debug mode** (2/9 have this)
- Document **resource disposal** (universal)

### Essential Features:
Based on implementation consensus, these are must-haves:
- TypeScript with strict mode
- Canvas-based initialization
- Scene graph with transforms
- Animation mixer pattern
- Raycaster for picking
- Orbit controls
- GLTF loading with Draco/KTX2
- React bindings (7/9 provide)
- Frame statistics API

### Style Conventions:
Most popular conventions:
- No semicolons (6/9 specify this)
- Const arrow functions (6/9 prefer)
- Options objects for configuration (universal)
- Async/await for loading (universal)
- Factory functions over constructors (5/9 vs 3/9)
