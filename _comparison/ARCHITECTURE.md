# ARCHITECTURE.md - Cross-Engine Comparison

This document compares the architectural decisions and module structures across all 9 engine implementations.

---

## Universal Agreement

### Layered Architecture
All implementations use a strict layered architecture with one-way dependencies:

```
React Bindings (top)
    ↓
High-Level API / Controls / Overlay
    ↓
Rendering Pipeline / Features
    ↓
GPU Abstraction Layer (GAL/HAL/Device)
    ↓
Math (bottom, zero dependencies)
```

### Zero-Dependency Math Layer
- **All 9 implementations**: Custom math library with no external dependencies
- Types: `Vec2`, `Vec3`, `Vec4`, `Mat4`, `Quat`, `AABB`, `Frustum`, `Ray`
- Backed by `Float32Array` for zero-allocation operations
- Z-up, right-handed coordinate system baked in

### GPU Abstraction Modeling
- **Universal decision**: Abstraction modeled after WebGPU, not WebGL2
- **Rationale**: WebGPU is the better API; WebGL2 is the translation layer
- Pipelines are immutable objects
- Bind groups are explicit
- Command recording is explicit

### Module Dependency Flow
All implementations enforce acyclic dependencies:
- Lower layers never depend on upper layers
- Math depends on nothing
- GPU layer depends only on math (and native APIs)
- Rendering depends on GPU + math
- High-level features depend on rendering + core
- React depends on everything but is a leaf module

---

## Key Variations

### 1. Module Organization Strategy

#### Monolithic Single Package
**Implementations**: Hyena, Rabbit, Shark, Wren, Bonobo (5 engines)

**Structure**:
```
engine/
├── src/
│   ├── core/
│   ├── math/
│   ├── renderer/
│   ├── scene/
│   ├── materials/
│   ├── animation/
│   └── ...
└── package.json
```

**Benefits**:
- Simpler distribution (single npm package)
- Easier to understand the whole system
- Fewer cross-package version conflicts
- One version number for everything

**Trade-offs**:
- Less granular tree-shaking
- All dependencies bundled
- Cannot mix versions of sub-modules

#### Multi-Package Monorepo
**Implementations**: Fennec, Caracal, Lynx, Mantis (4 engines)

**Structure**:
```
packages/
├── engine-core/
├── engine-materials/
├── engine-loaders/
├── engine-animation/
├── engine-react/
└── ...
```

**Benefits**:
- Tree-shakeable (import only what you need)
- Can version packages independently
- Clear separation of concerns
- Lazy-load optional features (loaders, react)

**Trade-offs**:
- More complex build setup
- Potential version mismatch issues
- More package.json files to maintain

### 2. Data Storage Philosophy

#### Structure of Arrays (SoA) - Flat Typed Arrays
**Primary**: Bonobo, Mantis, Shark (3 engines emphasize this)

**Pattern**:
```typescript
// All entity data in contiguous typed arrays
positions:      Float32Array(MAX_ENTITIES * 3)
rotations:      Float32Array(MAX_ENTITIES * 3)
scales:         Float32Array(MAX_ENTITIES * 3)
worldMatrices:  Float32Array(MAX_ENTITIES * 16)
flags:          Uint32Array(MAX_ENTITIES)
```

**Benefits**:
- Zero GC pressure
- Cache-friendly iteration
- GPU-uploadable (direct copy to buffers)
- Predictable memory layout

**Trade-offs**:
- Less flexible for sparse data
- Fixed maximum entity count
- More complex to add new properties

#### Object-Oriented with SoA Internals
**Primary**: Caracal, Fennec, Hyena, Lynx, Rabbit, Wren (6 engines)

**Pattern**:
```typescript
// Public API: object-oriented
class Node {
  position: Vec3
  rotation: Quat
  scale: Vec3

  // Internal: flat arrays for rendering
  private _worldMatrixIndex: number
}

// Renderer accesses flat arrays directly
worldMatrices: Float32Array(nodeCount * 16)
```

**Benefits**:
- Familiar API for users
- Flexible property addition
- Dynamic scene changes easy
- Best of both worlds internally

**Trade-offs**:
- Small overhead for object wrappers
- Two representations to keep in sync

### 3. Scene Graph Update Strategy

#### Dirty Flag Propagation (Universal)
All 9 implementations use dirty flags, but with variations:

**Breadth-First Update** (Mantis, Hyena):
```typescript
// Update all nodes at same depth level before going deeper
queue = [root]
while (queue.length) {
  node = queue.shift()
  if (node.dirty) updateWorldMatrix(node)
  queue.push(...node.children)
}
```

**Depth-First Update** (Most implementations):
```typescript
// Update parent, then immediately update children
function updateNode(node) {
  if (node.dirty) updateWorldMatrix(node)
  node.children.forEach(updateNode)
}
```

**Deferred Propagation** (Bonobo, Fennec):
```typescript
// Mark dirty in O(1), propagate once per frame
node.setPosition(x, y, z)  // Just sets dirty flag
// ...later...
propagateDirtyFlags()      // One pass at frame start
updateWorldMatrices()      // Only for dirty nodes
```

### 4. Render List Management

#### Rebuild Every Frame
**Who**: Bonobo, Fennec, Hyena, Rabbit, Wren (5 engines)

**Pattern**:
```typescript
// Each frame: collect visible objects
const visibleList = []
frustumCull(scene, camera, visibleList)
sortByKey(visibleList)
buildDrawCommands(visibleList)
```

**Benefits**:
- Simple, no state to maintain
- Always correct
- Easy to debug

**Trade-offs**:
- O(n log n) sort every frame
- Allocation of draw command array

#### Persistent Sorted List
**Who**: Caracal, Lynx, Mantis, Shark (4 engines)

**Pattern**:
```typescript
// Maintain sorted list, update incrementally
class RenderList {
  private commands: DrawCommand[] = []

  add(mesh) { /* insert at correct sorted position */ }
  remove(mesh) { /* splice out */ }
  setVisible(mesh, visible) { /* set flag, don't modify list */ }
}
```

**Benefits**:
- O(visible) iteration instead of O(total) sort
- Zero allocation if scene unchanged
- Fast when visibility changes but scene static

**Trade-offs**:
- More complex state management
- Need to handle insertions/deletions carefully

### 5. Uniform Buffer Strategy

#### Triple-Buffered Uniform Ring Buffer
**Who**: Mantis, Shark, Wren (3 engines explicitly mention)

**Pattern**:
```typescript
const RING_SIZE = 4 * 1024 * 1024  // 4MB
const FRAME_COUNT = 3               // Triple buffering

uniformBuffer = createBuffer(RING_SIZE)
frameOffset = (frameIndex % FRAME_COUNT) * (RING_SIZE / FRAME_COUNT)

// Write per-object data sequentially
for (object of visibleObjects) {
  writeAt(frameOffset + currentOffset, object.uniforms)
  currentOffset += uniformSize
}
```

**Benefits**:
- Zero per-object allocation
- No buffer creation/destruction
- GPU doesn't stall on previous frame
- Amortized cost of large allocation

**Trade-offs**:
- Wastes memory if scene is small
- Must track which frames are in flight

#### Per-Object Uniform Buffers
**Who**: Bonobo, Fennec, Hyena (3 engines, mostly in WebGL2 mode)

**Pattern**:
```typescript
// Create UBO per object, update only when changed
objectUBO = createBuffer(uniformSize)
if (object.dirty) {
  writeBuffer(objectUBO, object.uniforms)
}
setBindGroup(2, createBindGroup({ buffer: objectUBO }))
```

**Benefits**:
- Only update when object changes
- No over-allocation
- Simple to reason about

**Trade-offs**:
- More UBOs to manage
- Bind group switching overhead

#### Dynamic Offsets (Universal for Per-Object)
**Who**: All 9 implementations (WebGPU bind groups, WebGL2 UBO binding)

**Pattern**:
```typescript
// Single bind group, different offset per object
setBindGroup(2, sharedObjectBindGroup, [object.matrixOffset])
```

**Benefits**:
- Minimal binding overhead (offset only)
- Works identically in WebGPU and WebGL2
- Industry-standard approach

---

## Implementation Breakdown

### By Primary Data Layout

| Approach | Count | Implementations |
|----------|-------|----------------|
| SoA-first | 3 | Bonobo, Mantis, Shark |
| OOP with SoA internals | 6 | Caracal, Fennec, Hyena, Lynx, Rabbit, Wren |

### By Package Structure

| Structure | Count | Implementations |
|-----------|-------|----------------|
| Monolithic | 5 | Bonobo, Hyena, Rabbit, Shark, Wren |
| Multi-package | 4 | Caracal, Fennec, Lynx, Mantis |

### By Render List Strategy

| Strategy | Count | Implementations |
|----------|-------|----------------|
| Rebuild every frame | 5 | Bonobo, Fennec, Hyena, Rabbit, Wren |
| Persistent sorted | 4 | Caracal, Lynx, Mantis, Shark |

---

## Core Systems Comparison

### Frame Lifecycle (Universal Pattern)

All 9 implementations follow this sequence:

```
1. Time/Clock update
2. User callbacks (onFrame, useFrame)
3. Animation system update
4. Scene graph dirty flag propagation
5. World matrix recomputation (dirty only)
6. Frustum culling
7. Build/update render list
8. Sort draw commands
9. Upload per-frame uniforms
10. Execute render passes:
    - Shadow maps (CSM)
    - Opaque geometry
    - Transparent (OIT)
    - Post-processing
11. Submit to GPU
12. HTML overlay update
13. Reset scratch pools
```

**Variations**:
- Steps 3-5 order varies slightly
- Some combine culling and render list building
- Some update matrices lazily during rendering

### GPU Abstraction Interface

#### Common Pattern (All 9):

```typescript
interface Device {
  readonly backend: 'webgpu' | 'webgl2'

  createBuffer(desc): Buffer
  createTexture(desc): Texture
  createSampler(desc): Sampler
  createShader(desc): Shader
  createPipeline(desc): Pipeline
  createBindGroup(desc): BindGroup

  beginRenderPass(desc): RenderPass
  submit(): void
}
```

#### Naming Variations:

| Concept | Names Used |
|---------|-----------|
| Abstraction Layer | GAL (5), HAL (2), Device (2) |
| Buffer | GpuBuffer, GALBuffer, HalBuffer, Buffer |
| Texture | GpuTexture, GALTexture, HalTexture, Texture |
| Pipeline | GpuPipeline, GALPipeline, HalPipeline, Pipeline |

**Consensus**: Terminology varies but interface is nearly identical

### Memory Management Philosophy

#### Universal Patterns (All 9):

1. **Pre-allocated pools for math**:
   ```typescript
   const _scratchVec3 = new Float32Array(3)
   const _scratchMat4 = new Float32Array(16)
   ```

2. **Zero allocations in render loop**:
   - No `new` calls in hot path
   - Pre-allocated command arrays
   - Typed array pools reset each frame

3. **Resource lifetime management**:
   - GPU resources live until explicitly destroyed
   - Geometry/material creation expensive, expected to be long-lived
   - Scene graph nodes can be added/removed dynamically

#### Variations:

**Object Pooling**:
- **Who**: Fennec, Lynx (explicit mention)
- **Pattern**: Pool frequently created/destroyed objects (particles, projectiles)

**Geometric Buffer Growth**:
- **Who**: Caracal, Mantis, Shark (explicit)
- **Pattern**: Double buffer size when capacity exceeded
- **Trade-off**: Amortized O(1) insertion, may over-allocate

---

## Threading Model

### Universal Pattern (All 9):

**Main Thread**:
- Scene graph
- Render loop
- User callbacks
- GPU command submission

**Web Workers**:
- Draco mesh decoding
- Basis texture transcoding
- (Optional) BVH construction for large scenes

**Communication**:
- `postMessage` with `Transferable` typed arrays
- Zero-copy data transfer
- Async/Promise-based API

**Rationale** (Universal Agreement):
- WebGPU/WebGL2 contexts bound to main thread
- OffscreenCanvas prevents HTML overlay compositing
- Tight frame budgets make synchronization expensive
- CPU-heavy asset decoding is embarrassingly parallel

---

## What Makes Each Implementation Unique

### Bonobo
- **Flat entity storage**: No scene graph by default
- **Automatic instancing**: Merge draw calls with same geometry+material
- **Thin React layer**: No custom reconciler

### Caracal
- **Backend abstraction at resource level**: Not draw-call level
- **GPUDevice interface**: Emphasizes resource factory pattern
- **Document-heavy**: Most detailed architecture docs

### Fennec
- **Layered module dependencies**: Strict dependency graph
- **Package-first design**: Each feature is its own package
- **Error handling**: Debug mode with verbose errors, production mode silent

### Hyena
- **Device interface**: Minimal abstraction, just what's needed
- **WebGPU render bundles**: Primary optimization strategy
- **Offline worker tasks**: Emphasizes async asset loading

### Lynx
- **Zero-overhead abstraction**: No virtual dispatch in hot paths
- **Immutable pipelines**: Full state baking
- **Dual shader shipping**: WGSL + GLSL, no transpilation

### Mantis
- **Command buffer pattern**: Build command array, sort, execute
- **Ring buffer uniforms**: 4MB triple-buffered uniform storage
- **Radix sort**: O(n) draw call sorting

### Rabbit
- **HAL terminology**: Hardware Abstraction Layer naming
- **System layers diagram**: Five-layer architecture
- **Mobile-first**: Explicit mobile optimization focus

### Shark
- **Sort-once, draw-many**: Persistent render lists
- **Flat typed arrays**: Structure-of-arrays throughout
- **State sorting**: 64-bit composite sort keys

### Wren
- **Sort-based renderer**: 64-bit encoded render keys
- **Radix sort**: Two-pass 32-bit word sorting
- **Frame encoder**: Replaces WebGPU's command encoder concept

---

## Recommendations for Cherry-Picking

### If You Want Simplest Architecture
- **Start with**: Bonobo's flat entity system
- **Add**: Scene graph only if you need hierarchy
- **Benefit**: Fewer concepts, easier to understand

### If You Want Most Maintainable
- **Use**: Multi-package structure (Fennec, Caracal, Lynx, Mantis)
- **Add**: Strict dependency enforcement
- **Benefit**: Tree-shakeable, clear boundaries

### If You Want Best Performance
- **Combine**:
  - Mantis's ring buffer + radix sort
  - Shark's persistent render lists
  - Bonobo's automatic instancing
- **Benefit**: Maximum draw call throughput

### If You Want Best Developer Experience
- **Use**: Custom reconciler (8 engines)
- **Add**: Automatic disposal on unmount
- **Add**: TypeScript throughout
- **Benefit**: Declarative, type-safe, familiar

### If You Want Simplest GPU Abstraction
- **Use**: WebGPU-modeled interface (universal)
- **Implement**: WebGL2 as state machine emulation
- **Benefit**: Future-proof, cleaner API

### If You Want Most Flexible
- **Use**: Object-oriented public API (6 engines)
- **Use**: SoA internally for rendering
- **Benefit**: Familiar to users, fast for renderer
