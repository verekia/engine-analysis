# Lynx Engine Architecture

Lynx is a minimal high-performance 3D game engine for the web. It targets both WebGPU and WebGL2 backends through a unified GPU Abstraction Layer, uses a Z-up right-handed coordinate system, and is written in TypeScript with `const` arrow functions and no semicolons.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Frame Lifecycle](#2-frame-lifecycle)
3. [Data Flow](#3-data-flow)
4. [Module Dependencies](#4-module-dependencies)
5. [Initialization Sequence](#5-initialization-sequence)
6. [Threading Model](#6-threading-model)
7. [Memory Management](#7-memory-management)

---

## 1. System Overview

The engine is organized into six horizontal layers. Each layer depends only on the layers beneath it, never on layers above. This strict dependency direction ensures that lower layers remain reusable and testable in isolation.

```
┌─────────────────────────────────────────────────────────┐
│                    HTML Overlay                          │
│         DOM positioning from 3D coordinates             │
├─────────────────────────────────────────────────────────┤
│                  React Bindings                         │
│       Optional react-reconciler based package           │
├─────────────────────────────────────────────────────────┤
│                  High-Level API                         │
│         Scene, Mesh, Camera, Light, etc.                │
├─────────────────────────────────────────────────────────┤
│                 Render Pipeline                         │
│  Command sorting, multi-pass rendering (shadow,         │
│  opaque, transparent OIT, post-process)                 │
├─────────────────────────────────────────────────────────┤
│                   Core Systems                          │
│  Scene graph, materials, geometry, lighting,            │
│  animation, raycasting                                  │
├─────────────────────────────────────────────────────────┤
│            GPU Abstraction Layer (GAL)                  │
│         Thin wrapper over WebGPU / WebGL2               │
└─────────────────────────────────────────────────────────┘
```

### 1.1 GPU Abstraction Layer (GAL)

The GAL is the foundation of the rendering stack. It provides a unified interface over both WebGPU and WebGL2, hiding backend-specific details behind a common set of abstractions. The GAL has zero dependencies on any other Lynx module.

```typescript
type BackendType = "webgpu" | "webgl2"

interface GalDevice {
  readonly backend: BackendType
  readonly limits: DeviceLimits
  createBuffer: (desc: BufferDescriptor) => GalBuffer
  createTexture: (desc: TextureDescriptor) => GalTexture
  createSampler: (desc: SamplerDescriptor) => GalSampler
  createShaderModule: (desc: ShaderModuleDescriptor) => GalShaderModule
  createRenderPipeline: (desc: RenderPipelineDescriptor) => GalRenderPipeline
  createBindGroup: (desc: BindGroupDescriptor) => GalBindGroup
  createRenderPass: (desc: RenderPassDescriptor) => GalRenderPass
  submit: (commands: GalCommandBuffer[]) => void
  destroy: () => void
}

interface GalBuffer {
  readonly size: number
  readonly usage: BufferUsageFlags
  write: (data: ArrayBufferView, offset?: number) => void
  destroy: () => void
}

interface GalTexture {
  readonly width: number
  readonly height: number
  readonly format: TextureFormat
  readonly mipLevelCount: number
  createView: (desc?: TextureViewDescriptor) => GalTextureView
  destroy: () => void
}

interface GalRenderPass {
  setPipeline: (pipeline: GalRenderPipeline) => void
  setBindGroup: (index: number, group: GalBindGroup) => void
  setVertexBuffer: (slot: number, buffer: GalBuffer, offset?: number) => void
  setIndexBuffer: (buffer: GalBuffer, format: IndexFormat, offset?: number) => void
  drawIndexed: (indexCount: number, instanceCount?: number, firstIndex?: number) => void
  draw: (vertexCount: number, instanceCount?: number) => void
  end: () => void
}
```

The two backend implementations live under `gal/webgpu/` and `gal/webgl2/`. A factory function selects the appropriate backend at initialization:

```typescript
const createGalDevice = async (
  canvas: HTMLCanvasElement,
  preferredBackend?: BackendType
): Promise<GalDevice> => {
  if (preferredBackend !== "webgl2" && navigator.gpu) {
    const adapter = await navigator.gpu.requestAdapter()
    if (adapter) {
      const device = await adapter.requestDevice()
      return createWebGPUDevice(canvas, device)
    }
  }
  const gl = canvas.getContext("webgl2")
  if (!gl) throw new Error("Lynx: Neither WebGPU nor WebGL2 is available")
  return createWebGL2Device(canvas, gl)
}
```

### 1.2 Core Systems

Core systems provide the fundamental building blocks that the render pipeline consumes.

**Scene Graph** -- A tree of `SceneNode` objects. Each node stores a local transform (position, rotation, scale) and a lazily-computed world matrix. Dirty flags propagate downward to children when a parent transform changes.

```typescript
interface SceneNode {
  readonly id: number
  name: string
  parent: SceneNode | null
  readonly children: ReadonlyArray<SceneNode>
  localPosition: Vec3
  localRotation: Quat
  localScale: Vec3
  readonly worldMatrix: Mat4
  readonly worldMatrixDirty: boolean
  visible: boolean
  readonly aabb: AABB
  addChild: (child: SceneNode) => void
  removeChild: (child: SceneNode) => void
  traverse: (callback: (node: SceneNode) => void) => void
}
```

**Materials** -- Describe surface appearance through a set of parameters and texture bindings. Each material is associated with a shader program and a pipeline state hash. Materials are interned: two materials with identical parameters share the same GPU pipeline.

```typescript
interface Material {
  readonly id: number
  readonly pipelineHash: number
  shader: ShaderModule
  params: MaterialParams
  textures: Map<string, GalTexture>
  blendMode: BlendMode
  cullMode: CullMode
  depthWrite: boolean
  depthTest: boolean
}

type BlendMode = "opaque" | "transparent" | "additive" | "multiply"

interface MaterialParams {
  baseColor: Vec4
  metallic: number
  roughness: number
  emissive: Vec3
  emissiveIntensity: number
  alphaCutoff: number
  [key: string]: number | Vec2 | Vec3 | Vec4
}
```

**Geometry** -- Manages vertex attribute layouts and index buffers. Geometry objects own their GPU buffers and expose AABB data for culling.

```typescript
interface Geometry {
  readonly id: number
  readonly vertexCount: number
  readonly indexCount: number
  readonly vertexBuffers: Map<AttributeSemantic, GalBuffer>
  readonly indexBuffer: GalBuffer | null
  readonly indexFormat: IndexFormat
  readonly aabb: AABB
  readonly attributes: ReadonlyArray<VertexAttribute>
  destroy: () => void
}

type AttributeSemantic =
  | "position"
  | "normal"
  | "tangent"
  | "texcoord0"
  | "texcoord1"
  | "color0"
  | "joints0"
  | "weights0"

interface VertexAttribute {
  semantic: AttributeSemantic
  format: VertexFormat
  offset: number
  stride: number
  bufferSlot: number
}
```

**Lighting** -- Supports directional, point, and spot lights. Light data is packed into a uniform buffer each frame. Shadow-casting lights produce cascade entries for the shadow pass.

```typescript
type LightType = "directional" | "point" | "spot"

interface Light {
  readonly id: number
  type: LightType
  color: Vec3
  intensity: number
  range: number
  innerConeAngle: number
  outerConeAngle: number
  castShadow: boolean
  shadowBias: number
  shadowCascadeCount: number
  readonly node: SceneNode
}

interface PackedLightData {
  readonly buffer: Float32Array
  readonly count: number
  readonly directionalCount: number
  readonly pointCount: number
  readonly spotCount: number
}
```

**Animation** -- Keyframe-based animation system supporting position, rotation, scale, and morph target weight channels. Interpolation modes include step, linear, and cubic spline.

```typescript
interface AnimationClip {
  readonly name: string
  readonly duration: number
  readonly channels: ReadonlyArray<AnimationChannel>
}

interface AnimationChannel {
  targetNode: SceneNode
  property: "position" | "rotation" | "scale" | "weights"
  interpolation: "step" | "linear" | "cubicspline"
  times: Float32Array
  values: Float32Array
}

interface AnimationMixer {
  play: (clip: AnimationClip, options?: PlaybackOptions) => AnimationAction
  update: (deltaTime: number) => void
  stopAll: () => void
}
```

**Raycasting** -- Performs intersection tests against the scene graph using AABBs for broad-phase and triangle-level tests for narrow-phase.

```typescript
interface Ray {
  origin: Vec3
  direction: Vec3
}

interface RaycastHit {
  node: SceneNode
  distance: number
  point: Vec3
  normal: Vec3
  triangleIndex: number
}

const raycast = (ray: Ray, scene: SceneNode, maxDistance?: number): RaycastHit | null => {
  // Broad phase: test ray against AABB tree
  // Narrow phase: test ray against triangles of candidate meshes
  // Return closest hit or null
}
```

### 1.3 Render Pipeline

The render pipeline consumes the scene graph and core systems to produce the final image. It manages command list construction, sorting, and multi-pass execution.

The pipeline runs the following passes each frame:

| Pass | Description | Render Target |
|------|-------------|---------------|
| Shadow | Render shadow casters into CSM cascades | Shadow map atlas (depth-only) |
| Opaque | Render sorted opaque commands with MSAA | MSAA color + depth |
| Transparency | Weighted Blended OIT for transparent objects | OIT accumulation + revealage |
| Post-process | Bloom, tone mapping, MSAA resolve | Intermediate textures, then final backbuffer |

```typescript
interface RenderPipeline {
  readonly shadowPass: ShadowPass
  readonly opaquePass: OpaquePass
  readonly transparencyPass: TransparencyPass
  readonly postProcessPass: PostProcessPass
  execute: (frameContext: FrameContext) => void
}

interface FrameContext {
  readonly device: GalDevice
  readonly camera: Camera
  readonly scene: SceneNode
  readonly lights: PackedLightData
  readonly commandList: RenderCommandList
  readonly deltaTime: number
  readonly frameIndex: number
  readonly pools: FramePools
}
```

### 1.4 High-Level API

The user-facing API provides concrete classes that wrap the core systems and render pipeline into a simple, ergonomic interface.

```typescript
interface LynxEngine {
  readonly canvas: HTMLCanvasElement
  readonly device: GalDevice
  readonly renderer: RenderPipeline
  readonly scene: Scene
  readonly clock: Clock
  start: () => void
  stop: () => void
  resize: (width: number, height: number) => void
  destroy: () => void
}

const createEngine = async (options: EngineOptions): Promise<LynxEngine> => {
  const device = await createGalDevice(options.canvas, options.preferredBackend)
  const renderer = createRenderPipeline(device, options.renderOptions)
  const scene = createScene()
  const clock = createClock()
  // ... assemble and return engine
}
```

Key high-level objects:

```typescript
interface Scene {
  readonly root: SceneNode
  readonly background: Background
  add: (object: SceneObject) => void
  remove: (object: SceneObject) => void
  getByName: (name: string) => SceneObject | null
  traverse: (callback: (object: SceneObject) => void) => void
}

interface Mesh extends SceneObject {
  geometry: Geometry
  material: Material
  castShadow: boolean
  receiveShadow: boolean
  frustumCulled: boolean
}

interface Camera extends SceneObject {
  projectionMatrix: Mat4
  viewMatrix: Mat4
  near: number
  far: number
  readonly frustum: Frustum
  worldToScreen: (point: Vec3, out: Vec2) => Vec2
  screenToRay: (screenPos: Vec2) => Ray
}

interface PerspectiveCamera extends Camera {
  fov: number
  aspect: number
}

interface OrthographicCamera extends Camera {
  left: number
  right: number
  top: number
  bottom: number
}
```

### 1.5 React Bindings

An optional package provides a react-reconciler based integration, allowing Lynx scenes to be defined declaratively with React components. This package depends on the high-level API and React but is entirely optional.

```typescript
// @lynx/react package

interface LynxCanvasProps {
  width?: number
  height?: number
  antialias?: boolean
  preferredBackend?: BackendType
  onCreated?: (engine: LynxEngine) => void
  children?: React.ReactNode
}

// Usage example:
// <LynxCanvas>
//   <perspectiveCamera position={[0, -5, 2]} fov={60} />
//   <mesh position={[0, 0, 0.5]}>
//     <boxGeometry args={[1, 1, 1]} />
//     <standardMaterial baseColor={[1, 0.2, 0.1, 1]} roughness={0.4} />
//   </mesh>
//   <directionalLight
//     position={[5, -5, 10]}
//     intensity={2.0}
//     castShadow
//   />
//   <ambientLight intensity={0.3} />
// </LynxCanvas>
```

The reconciler maps JSX elements to engine objects:

```typescript
const reconciler = ReactReconciler({
  createInstance: (type: string, props: Record<string, unknown>) => {
    // Map element type to Lynx engine object
    // e.g. "mesh" -> createMeshNode()
    //      "standardMaterial" -> createStandardMaterial(props)
  },
  appendChild: (parent: SceneNode, child: SceneNode) => {
    parent.addChild(child)
  },
  removeChild: (parent: SceneNode, child: SceneNode) => {
    parent.removeChild(child)
  },
  commitUpdate: (instance: SceneObject, updatePayload: unknown, type: string, oldProps: Record<string, unknown>, newProps: Record<string, unknown>) => {
    // Diff props and apply changes to the engine object
  },
  // ... other reconciler methods
})
```

### 1.6 HTML Overlay

The HTML overlay system enables DOM elements to be anchored to 3D positions in the scene. Each tracked position is projected through the camera every frame, and the resulting screen coordinates are applied as CSS transforms.

```typescript
interface HtmlOverlay {
  track: (id: string, worldPosition: Vec3, element: HTMLElement) => void
  untrack: (id: string) => void
  update: (camera: Camera, canvasRect: DOMRect) => void
}

const createHtmlOverlay = (container: HTMLElement): HtmlOverlay => {
  const tracked = new Map<string, TrackedEntry>()

  const update = (camera: Camera, canvasRect: DOMRect) => {
    tracked.forEach((entry) => {
      const screen = camera.worldToScreen(entry.worldPosition, entry.screenPos)

      // Check if behind camera
      if (screen[0] < -1 || screen[0] > 1 || screen[1] < -1 || screen[1] > 1) {
        entry.element.style.display = "none"
        return
      }

      const x = (screen[0] * 0.5 + 0.5) * canvasRect.width
      const y = (1.0 - (screen[1] * 0.5 + 0.5)) * canvasRect.height

      entry.element.style.display = ""
      entry.element.style.transform = `translate(${x}px, ${y}px)`
    })
  }

  return { track, untrack, update }
}
```

---

## 2. Frame Lifecycle

Each frame follows a strict sequence of phases. The ordering guarantees that data is always consistent when consumed by downstream passes.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        FRAME N                                      │
│                                                                     │
│  1. Scene Graph Dirty Flag Propagation                              │
│  2. World Matrix Updates (dirty nodes only)                         │
│  3. Frustum Culling (AABB test against camera frustum)              │
│  4. Render Command List Rebuild (only when scene changes)           │
│  5. Command Sorting                                                 │
│     ├── Pipeline state                                              │
│     ├── Material                                                    │
│     ├── Front-to-back (opaque)                                      │
│     └── Back-to-front (transparent input to OIT)                    │
│  6. Shadow Pass (CSM cascades)                                      │
│  7. Opaque Pass (sorted opaque commands, MSAA)                      │
│  8. Transparency Pass (Weighted Blended OIT)                        │
│  9. Post-Process                                                    │
│     ├── Bloom extraction                                            │
│     ├── Downscale / upscale chain                                   │
│     ├── Tone mapping                                                │
│     ├── MSAA resolve                                                │
│     └── Final blit                                                  │
│ 10. HTML Overlay Update                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.1 Scene Graph Dirty Flag Propagation

When a node's local transform is modified, a dirty flag is set on that node. This flag propagates to all descendants. Propagation is deferred: it does not happen at the time of the property set. Instead, all dirty flags are resolved in a single top-down pass at the start of the frame.

```typescript
const propagateDirtyFlags = (root: SceneNode) => {
  const stack: SceneNode[] = [root]
  while (stack.length > 0) {
    const node = stack.pop()!
    for (let i = 0; i < node.children.length; i++) {
      const child = node.children[i]
      if (node.worldMatrixDirty) {
        markDirty(child)
      }
      stack.push(child)
    }
  }
}

const markDirty = (node: SceneNode) => {
  (node as MutableSceneNode)._worldMatrixDirty = true
  (node as MutableSceneNode)._aabbDirty = true
}
```

### 2.2 World Matrix Updates

After dirty flags have propagated, only nodes marked dirty recompute their world matrices. This avoids recalculating the entire tree when only a small subtree has changed.

```typescript
const updateWorldMatrices = (root: SceneNode, pools: FramePools) => {
  const stack: SceneNode[] = [root]
  while (stack.length > 0) {
    const node = stack.pop()!
    if (node.worldMatrixDirty) {
      const local = mat4ComposeFromTRS(
        node.localPosition,
        node.localRotation,
        node.localScale,
        pools.tempMat4
      )
      if (node.parent) {
        mat4Multiply(node.parent.worldMatrix, local, node.worldMatrix as Float32Array)
      } else {
        mat4Copy(local, node.worldMatrix as Float32Array)
      }
      recomputeAABB(node)
      clearDirtyFlag(node)
    }
    for (let i = 0; i < node.children.length; i++) {
      stack.push(node.children[i])
    }
  }
}
```

### 2.3 Frustum Culling

Every visible mesh node is tested against the camera frustum using its axis-aligned bounding box (AABB). The AABB is expressed in world space and is recomputed whenever the world matrix changes. The frustum test checks the AABB against all six frustum planes.

```typescript
const cullAgainstFrustum = (
  nodes: ReadonlyArray<SceneNode>,
  frustum: Frustum,
  visibleOut: SceneNode[]
) => {
  visibleOut.length = 0
  for (let i = 0; i < nodes.length; i++) {
    const node = nodes[i]
    if (!node.visible) continue
    if (!(node as Mesh).frustumCulled || aabbIntersectsFrustum(node.aabb, frustum)) {
      visibleOut.push(node)
    }
  }
}

const aabbIntersectsFrustum = (aabb: AABB, frustum: Frustum): boolean => {
  for (let i = 0; i < 6; i++) {
    const plane = frustum.planes[i]
    // Find the AABB corner most in the direction of the plane normal
    const px = plane.normal[0] >= 0 ? aabb.max[0] : aabb.min[0]
    const py = plane.normal[1] >= 0 ? aabb.max[1] : aabb.min[1]
    const pz = plane.normal[2] >= 0 ? aabb.max[2] : aabb.min[2]
    const dist = plane.normal[0] * px + plane.normal[1] * py + plane.normal[2] * pz + plane.d
    if (dist < 0) return false  // Entirely outside this plane
  }
  return true
}
```

### 2.4 Render Command List Rebuild

The render command list is a flat array of `RenderCommand` structs. It is only rebuilt when the set of visible objects changes (tracked by a scene version counter). When the scene is static and the camera merely rotates, the existing command list is reused and only the sort keys are updated.

```typescript
interface RenderCommand {
  sortKey: number           // Composite sort key for ordering
  pipelineHash: number      // Pipeline state hash
  materialId: number        // Material identifier
  geometryId: number        // Geometry identifier
  worldMatrixOffset: number // Offset into the world matrix buffer
  depth: number             // Distance from camera (for sorting)
  isTransparent: boolean    // Determines which pass this command enters
}

const rebuildCommandList = (
  visibleNodes: ReadonlyArray<SceneNode>,
  camera: Camera,
  commandList: RenderCommandList
) => {
  commandList.clear()
  for (let i = 0; i < visibleNodes.length; i++) {
    const node = visibleNodes[i] as Mesh
    if (!node.geometry || !node.material) continue

    const depth = computeDepthFromCamera(node.aabb, camera)
    const cmd = commandList.acquire()
    cmd.pipelineHash = node.material.pipelineHash
    cmd.materialId = node.material.id
    cmd.geometryId = node.geometry.id
    cmd.worldMatrixOffset = node.id * 16
    cmd.depth = depth
    cmd.isTransparent = node.material.blendMode !== "opaque"
    cmd.sortKey = computeSortKey(cmd)
  }
}
```

### 2.5 Command Sorting

Commands are sorted using a composite sort key packed into a single 64-bit number (represented as two 32-bit numbers due to JavaScript limitations). The sort order differs for opaque and transparent commands.

**Opaque commands** are sorted to minimize GPU state changes and maximize depth buffer efficiency:
1. Pipeline state (highest priority -- reduces pipeline switches)
2. Material (reduces bind group switches)
3. Front-to-back by depth (maximizes early-z rejection)

**Transparent commands** feeding the OIT pass are sorted back-to-front. While Weighted Blended OIT is technically order-independent, back-to-front input ordering improves the quality of the weight function, especially for overlapping surfaces at similar depths.

```typescript
const computeSortKey = (cmd: RenderCommand): number => {
  if (!cmd.isTransparent) {
    // Opaque: pipeline (16 bits) | material (16 bits) | depth front-to-back (32 bits)
    return (
      ((cmd.pipelineHash & 0xFFFF) << 48) |
      ((cmd.materialId & 0xFFFF) << 32) |
      (floatToSortableInt(cmd.depth) & 0xFFFFFFFF)
    )
  } else {
    // Transparent: depth back-to-front only
    return floatToSortableInt(-cmd.depth)
  }
}

const sortCommandList = (commandList: RenderCommandList) => {
  commandList.opaqueCommands.sort((a, b) => a.sortKey - b.sortKey)
  commandList.transparentCommands.sort((a, b) => a.sortKey - b.sortKey)
}
```

### 2.6 Shadow Pass

Shadow-casting directional lights use Cascaded Shadow Maps (CSM). The visible scene is split into cascades based on the camera's depth range, and each cascade renders shadow casters from the light's perspective into a region of a shared shadow map atlas.

```typescript
interface ShadowPass {
  readonly atlas: GalTexture         // Depth-only shadow atlas
  readonly cascadeCount: number
  readonly cascadeViews: Mat4[]      // Light-space view-projection per cascade
  readonly cascadeSplits: Float32Array
  execute: (context: FrameContext) => void
}

const executeShadowPass = (context: FrameContext, shadowPass: ShadowPass) => {
  const { device, commandList, lights } = context

  computeCascadeSplits(context.camera, shadowPass.cascadeCount, shadowPass.cascadeSplits)

  for (let cascade = 0; cascade < shadowPass.cascadeCount; cascade++) {
    const lightViewProj = computeCascadeLightMatrix(
      context.camera,
      lights,
      shadowPass.cascadeSplits[cascade],
      shadowPass.cascadeSplits[cascade + 1]
    )
    shadowPass.cascadeViews[cascade] = lightViewProj

    const viewport = getCascadeViewport(cascade, shadowPass.atlas.width)

    const pass = device.createRenderPass({
      depthAttachment: {
        view: shadowPass.atlas.createView(),
        loadOp: cascade === 0 ? "clear" : "load",
        storeOp: "store",
      },
      viewport,
    })

    for (let i = 0; i < commandList.shadowCasters.length; i++) {
      const cmd = commandList.shadowCasters[i]
      pass.setPipeline(shadowPass.depthOnlyPipeline)
      pass.setVertexBuffer(0, cmd.positionBuffer)
      pass.setIndexBuffer(cmd.indexBuffer, cmd.indexFormat)
      pass.drawIndexed(cmd.indexCount)
    }

    pass.end()
  }
}
```

### 2.7 Opaque Pass

The opaque pass renders all non-transparent objects into the MSAA color and depth targets. Because commands are sorted by pipeline then material then front-to-back, GPU state transitions are minimized and the depth buffer rejects occluded fragments early.

```typescript
const executeOpaquePass = (context: FrameContext, targets: RenderTargets) => {
  const { device, commandList, camera, lights } = context

  const pass = device.createRenderPass({
    colorAttachments: [{
      view: targets.msaaColor.createView(),
      resolveTarget: undefined, // Resolved in post-process
      loadOp: "clear",
      storeOp: "store",
      clearColor: { r: 0, g: 0, b: 0, a: 1 },
    }],
    depthAttachment: {
      view: targets.msaaDepth.createView(),
      loadOp: "clear",
      storeOp: "store",
      clearDepth: 1.0,
    },
  })

  let currentPipeline: GalRenderPipeline | null = null
  let currentMaterial: number = -1

  for (let i = 0; i < commandList.opaqueCommands.length; i++) {
    const cmd = commandList.opaqueCommands[i]

    if (cmd.pipelineHash !== currentPipeline?.hash) {
      currentPipeline = getPipeline(cmd.pipelineHash)
      pass.setPipeline(currentPipeline)
      pass.setBindGroup(0, camera.bindGroup)  // Per-frame: camera, lights
    }

    if (cmd.materialId !== currentMaterial) {
      currentMaterial = cmd.materialId
      pass.setBindGroup(1, getMaterialBindGroup(cmd.materialId))
    }

    pass.setBindGroup(2, getObjectBindGroup(cmd.worldMatrixOffset))
    bindGeometryBuffers(pass, cmd.geometryId)
    pass.drawIndexed(getIndexCount(cmd.geometryId))
  }

  pass.end()
}
```

### 2.8 Transparency Pass (Weighted Blended OIT)

Transparent objects are rendered using Weighted Blended Order-Independent Transparency. This technique avoids the need for exact per-fragment sorting by accumulating weighted color and an alpha revealage factor into two separate render targets in a single pass.

```typescript
interface OITTargets {
  accumulation: GalTexture  // RGBA16F: weighted color accumulation
  revealage: GalTexture     // R8: revealage (product of 1 - alpha)
}

const executeTransparencyPass = (context: FrameContext, targets: RenderTargets, oitTargets: OITTargets) => {
  const { device, commandList } = context

  const pass = device.createRenderPass({
    colorAttachments: [
      {
        view: oitTargets.accumulation.createView(),
        loadOp: "clear",
        storeOp: "store",
        clearColor: { r: 0, g: 0, b: 0, a: 0 },
        // Blend: ONE, ONE (additive for accumulation)
      },
      {
        view: oitTargets.revealage.createView(),
        loadOp: "clear",
        storeOp: "store",
        clearColor: { r: 1, g: 0, b: 0, a: 0 },
        // Blend: ZERO, ONE_MINUS_SRC_COLOR (multiplicative for revealage)
      },
    ],
    depthAttachment: {
      view: targets.msaaDepth.createView(),
      loadOp: "load",   // Reuse depth from opaque pass
      storeOp: "store",
      depthWriteEnabled: false,  // Do not write depth for transparent objects
    },
  })

  for (let i = 0; i < commandList.transparentCommands.length; i++) {
    const cmd = commandList.transparentCommands[i]
    pass.setPipeline(getOITPipeline(cmd.pipelineHash))
    pass.setBindGroup(0, context.camera.bindGroup)
    pass.setBindGroup(1, getMaterialBindGroup(cmd.materialId))
    pass.setBindGroup(2, getObjectBindGroup(cmd.worldMatrixOffset))
    bindGeometryBuffers(pass, cmd.geometryId)
    pass.drawIndexed(getIndexCount(cmd.geometryId))
  }

  pass.end()

  // OIT composite: combine accumulation and revealage over the opaque result
  compositeOIT(device, oitTargets, targets.msaaColor)
}
```

The OIT fragment shader computes the weight function and outputs to both targets:

```glsl
// OIT fragment shader (WGSL shown as pseudocode)
//
// weight = alpha * clamp(0.03 / (1e-5 + pow(linearDepth / 200.0, 4.0)), 0.01, 3000.0)
// accumTarget = vec4(color.rgb * alpha * weight, alpha * weight)
// revealageTarget = alpha
```

### 2.9 Post-Process

Post-processing runs as a series of full-screen passes, each reading from the previous pass's output.

```
Bloom Extraction ─── bright pixels above threshold
       │
       ▼
Downscale Chain ──── progressively halve resolution (5-6 mip levels)
       │
       ▼
Upscale Chain ────── progressively double resolution, blending with each mip
       │
       ▼
Tone Mapping ─────── apply ACES filmic or Khronos PBR Neutral tone mapping
       │
       ▼
MSAA Resolve ─────── resolve multi-sampled target to single-sample
       │
       ▼
Final Blit ───────── copy to the swap chain backbuffer
```

```typescript
interface PostProcessPass {
  bloomExtract: FullScreenPass
  bloomDownscale: FullScreenPass[]
  bloomUpscale: FullScreenPass[]
  toneMapping: FullScreenPass
  msaaResolve: FullScreenPass
  finalBlit: FullScreenPass
}

const executePostProcess = (
  device: GalDevice,
  postProcess: PostProcessPass,
  source: GalTexture,
  msaaSource: GalTexture,
  destination: GalTextureView,
  bloomThreshold: number,
  exposure: number
) => {
  // 1. Extract bright pixels above the bloom threshold
  postProcess.bloomExtract.execute(device, {
    input: source,
    uniforms: { threshold: bloomThreshold },
  })

  // 2. Progressive downscale chain
  let currentSource = postProcess.bloomExtract.output
  for (let i = 0; i < postProcess.bloomDownscale.length; i++) {
    postProcess.bloomDownscale[i].execute(device, { input: currentSource })
    currentSource = postProcess.bloomDownscale[i].output
  }

  // 3. Progressive upscale chain with additive blending
  for (let i = postProcess.bloomUpscale.length - 1; i >= 0; i--) {
    const downscaleTex = postProcess.bloomDownscale[i].output
    postProcess.bloomUpscale[i].execute(device, {
      input: currentSource,
      blend: downscaleTex,
    })
    currentSource = postProcess.bloomUpscale[i].output
  }

  // 4. Tone mapping: combine scene color + bloom, apply curve
  postProcess.toneMapping.execute(device, {
    input: source,
    bloom: currentSource,
    uniforms: { exposure, bloomStrength: 0.04 },
  })

  // 5. MSAA resolve
  postProcess.msaaResolve.execute(device, {
    input: msaaSource,
  })

  // 6. Final blit to swap chain
  postProcess.finalBlit.execute(device, {
    input: postProcess.toneMapping.output,
    output: destination,
  })
}
```

### 2.10 HTML Overlay Update

At the end of the frame, after all GPU work has been submitted, the HTML overlay system projects every tracked 3D position to screen coordinates and updates the corresponding DOM elements. This runs on the main thread and uses `style.transform` to avoid triggering layout.

```typescript
const updateHtmlOverlay = (
  overlay: HtmlOverlay,
  camera: Camera,
  canvasRect: DOMRect
) => {
  overlay.update(camera, canvasRect)
}
```

---

## 3. Data Flow

The following diagram traces how data moves from the user-facing scene graph all the way down to GPU submission.

```
                     USER CODE
                        │
                        ▼
              ┌──────────────────┐
              │   High-Level API │   Mesh, Camera, Light objects
              │   (Scene graph)  │   with local transforms
              └────────┬─────────┘
                       │
          ┌────────────┼────────────────┐
          ▼            ▼                ▼
   ┌────────────┐ ┌──────────┐  ┌────────────┐
   │   Dirty    │ │ Frustum  │  │  Material   │
   │   Flag     │ │ Culling  │  │  Interning  │
   │ Propagation│ │ (AABB)   │  │ & Pipeline  │
   │ & Matrix   │ │          │  │   Hashing   │
   │  Update    │ │          │  │             │
   └──────┬─────┘ └────┬─────┘  └──────┬─────┘
          │             │               │
          ▼             ▼               ▼
   ┌──────────────────────────────────────────┐
   │           Render Command List             │
   │                                           │
   │  Each command references:                 │
   │   - Pipeline hash (→ GPU pipeline)        │
   │   - Material ID (→ bind group)            │
   │   - Geometry ID (→ vertex/index buffers)  │
   │   - World matrix offset (→ UBO)           │
   │   - Depth (→ sort key)                    │
   └─────────────────────┬─────────────────────┘
                         │
                    Sort by key
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │  Shadow    │ │  Opaque    │ │Transparency│
   │  Casters   │ │  Commands  │ │  Commands  │
   └──────┬─────┘ └──────┬─────┘ └──────┬─────┘
          │              │              │
          ▼              ▼              ▼
   ┌──────────────────────────────────────────┐
   │         GPU Abstraction Layer (GAL)       │
   │                                           │
   │  - Bind pipeline state                    │
   │  - Bind resources (UBOs, textures)        │
   │  - Set vertex/index buffers               │
   │  - Issue draw calls                       │
   │  - Execute render passes                  │
   └─────────────────────┬─────────────────────┘
                         │
                         ▼
   ┌──────────────────────────────────────────┐
   │         WebGPU  or  WebGL2               │
   │          (native GPU driver)             │
   └──────────────────────────────────────────┘
```

### Uniform Buffer Data Flow

Per-frame uniform data follows a dedicated path to minimize CPU-GPU synchronization:

```
  Camera matrices (view, proj)  ──┐
  Light data (packed array)     ──┤
  Shadow cascade matrices       ──┼──▶  Frame Uniform Buffer (ring-buffered)
  Post-process params           ──┘         │
                                            ▼
                                    Bind Group 0 (per-frame)

  Material parameters           ──────▶  Material Uniform Buffer
                                            │
                                            ▼
                                    Bind Group 1 (per-material)

  World matrices (all objects)  ──────▶  Object Uniform Buffer (dynamic offset)
                                            │
                                            ▼
                                    Bind Group 2 (per-object, dynamic offset)
```

---

## 4. Module Dependencies

The dependency graph is strictly acyclic. Arrows point from dependent to dependency.

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │   @lynx/react ──────────────────┐                                │
  │                                 ▼                                │
  │   @lynx/overlay ─────────▶ @lynx/core                           │
  │                                 │                                │
  │                    ┌────────────┼──────────────┐                 │
  │                    ▼            ▼              ▼                 │
  │              @lynx/renderer  @lynx/scene  @lynx/animation        │
  │                 │      │        │                                │
  │                 │      │        ▼                                │
  │                 │      ├──▶ @lynx/materials                      │
  │                 │      │        │                                │
  │                 │      │        ▼                                │
  │                 ▼      ├──▶ @lynx/geometry                       │
  │              @lynx/gal │                                         │
  │                 ▲      │    @lynx/lighting                       │
  │                 │      │        │                                │
  │                 │      └────────┘                                │
  │                 │                                                │
  │              @lynx/math  (Vec2, Vec3, Vec4, Mat4, Quat, AABB)   │
  │                 ▲                                                │
  │                 │                                                │
  │          (depended upon by nearly all modules)                   │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

### Dependency Rules

| Module | Depends On | Depended On By |
|--------|-----------|----------------|
| `@lynx/math` | nothing | everything |
| `@lynx/gal` | nothing (only Web APIs) | renderer, materials, geometry |
| `@lynx/scene` | math | core, renderer, animation, raycasting |
| `@lynx/materials` | math, gal | renderer, core |
| `@lynx/geometry` | math, gal | renderer, core, raycasting |
| `@lynx/lighting` | math, gal | renderer, core |
| `@lynx/animation` | math, scene | core |
| `@lynx/raycasting` | math, scene, geometry | core |
| `@lynx/renderer` | gal, scene, materials, geometry, lighting, math | core |
| `@lynx/core` | renderer, scene, materials, geometry, lighting, animation, math | react, overlay |
| `@lynx/overlay` | core, math | (leaf) |
| `@lynx/react` | core, react | (leaf) |

### Key Principles

- **GAL depends on nothing.** It is a pure abstraction over browser GPU APIs. It can be tested independently with mock contexts.
- **Scene graph depends only on math.** It has no knowledge of rendering, materials, or GPU resources.
- **Renderer depends on GAL + scene graph + materials + geometry + lighting.** It orchestrates the multi-pass pipeline by reading from the scene graph and issuing GAL commands.
- **High-level API (core) depends on everything below it.** It is the integration layer that wires all subsystems together.
- **React bindings and HTML overlay are leaf modules.** They depend on core but nothing depends on them. They are entirely optional.

---

## 5. Initialization Sequence

Engine initialization follows a deterministic sequence. Each step depends on the previous step completing successfully.

```
   createEngine(options)
         │
         ▼
   ┌─────────────────────────────┐
   │ 1. Detect WebGPU support    │  navigator.gpu?.requestAdapter()
   │    Fall back to WebGL2      │  canvas.getContext("webgl2")
   └──────────────┬──────────────┘
                  │
                  ▼
   ┌─────────────────────────────┐
   │ 2. Create GalDevice         │  Wrap native device in GAL interface
   │    Query device limits       │  Max texture size, max UBO size, etc.
   └──────────────┬──────────────┘
                  │
                  ▼
   ┌─────────────────────────────┐
   │ 3. Create default render    │  MSAA color target (4x by default)
   │    targets                  │  MSAA depth target
   │                             │  OIT accumulation (RGBA16F)
   │                             │  OIT revealage (R8)
   └──────────────┬──────────────┘
                  │
                  ▼
   ┌─────────────────────────────┐
   │ 4. MSAA setup               │  Create multi-sampled textures
   │                             │  Configure resolve targets
   └──────────────┬──────────────┘
                  │
                  ▼
   ┌─────────────────────────────┐
   │ 5. Shadow map allocation    │  Allocate shadow atlas texture
   │                             │  Default: 2048x2048 depth texture
   │                             │  Partition into cascade regions
   └──────────────┬──────────────┘
                  │
                  ▼
   ┌─────────────────────────────┐
   │ 6. Create default resources │  Default white texture (1x1)
   │                             │  Default normal map (1x1, [0.5,0.5,1])
   │                             │  Default sampler (linear, repeat)
   │                             │  Screen-quad geometry
   └──────────────┬──────────────┘
                  │
                  ▼
   ┌─────────────────────────────┐
   │ 7. Create uniform buffers   │  Frame UBO (camera, lights, shadow)
   │                             │  Object UBO (world matrices, dynamic)
   │                             │  Post-process UBO
   └──────────────┬──────────────┘
                  │
                  ▼
   ┌─────────────────────────────┐
   │ 8. Compile default shaders  │  PBR standard shader
   │                             │  Depth-only (shadow) shader
   │                             │  OIT composite shader
   │                             │  Post-process shaders
   └──────────────┬──────────────┘
                  │
                  ▼
   ┌─────────────────────────────┐
   │ 9. Initialize subsystems    │  Scene graph (empty root node)
   │                             │  Animation mixer
   │                             │  HTML overlay container
   │                             │  Frame pool allocator
   └──────────────┬──────────────┘
                  │
                  ▼
   ┌─────────────────────────────┐
   │ 10. Start render loop       │  requestAnimationFrame callback
   │                             │  Return LynxEngine handle
   └─────────────────────────────┘
```

Implementation:

```typescript
const createEngine = async (options: EngineOptions): Promise<LynxEngine> => {
  // 1. Create GAL device (WebGPU with WebGL2 fallback)
  const device = await createGalDevice(options.canvas, options.preferredBackend)

  // 2. Query limits
  const limits = device.limits
  const maxTextureSize = limits.maxTextureDimension2D
  const msaaSamples = options.antialias !== false ? 4 : 1

  // 3-4. Create default render targets with MSAA
  const width = options.canvas.width
  const height = options.canvas.height
  const renderTargets = createRenderTargets(device, width, height, msaaSamples)

  // 5. Allocate shadow map atlas
  const shadowAtlasSize = options.shadowMapSize ?? 2048
  const shadowAtlas = device.createTexture({
    width: shadowAtlasSize,
    height: shadowAtlasSize,
    format: "depth32float",
    usage: TextureUsage.RENDER_ATTACHMENT | TextureUsage.SAMPLED,
  })

  // 6. Create default resources
  const defaults = createDefaultResources(device)

  // 7. Create uniform buffers
  const frameUBO = device.createBuffer({
    size: FRAME_UBO_SIZE,
    usage: BufferUsage.UNIFORM | BufferUsage.COPY_DST,
  })

  const objectUBO = device.createBuffer({
    size: INITIAL_OBJECT_UBO_SIZE,
    usage: BufferUsage.UNIFORM | BufferUsage.COPY_DST,
  })

  // 8. Compile default shaders
  const shaders = await compileDefaultShaders(device)

  // 9. Initialize subsystems
  const scene = createScene()
  const animationMixer = createAnimationMixer()
  const overlay = createHtmlOverlay(options.overlayContainer ?? options.canvas.parentElement!)
  const pools = createFramePools()

  // 10. Assemble and return engine
  const renderer = createRenderPipeline(device, renderTargets, shadowAtlas, shaders, defaults)
  const clock = createClock()

  const engine: LynxEngine = {
    canvas: options.canvas,
    device,
    renderer,
    scene,
    clock,
    start: () => startRenderLoop(engine, animationMixer, overlay, pools),
    stop: () => stopRenderLoop(engine),
    resize: (w, h) => resizeRenderTargets(device, renderTargets, w, h, msaaSamples),
    destroy: () => destroyEngine(engine, renderTargets, shadowAtlas, defaults),
  }

  return engine
}
```

### Render Target Creation

```typescript
interface RenderTargets {
  msaaColor: GalTexture
  msaaDepth: GalTexture
  resolveColor: GalTexture
  oitAccumulation: GalTexture
  oitRevealage: GalTexture
}

const createRenderTargets = (
  device: GalDevice,
  width: number,
  height: number,
  msaaSamples: number
): RenderTargets => {
  const msaaColor = device.createTexture({
    width, height,
    format: "rgba16float",
    sampleCount: msaaSamples,
    usage: TextureUsage.RENDER_ATTACHMENT,
  })

  const msaaDepth = device.createTexture({
    width, height,
    format: "depth24plus",
    sampleCount: msaaSamples,
    usage: TextureUsage.RENDER_ATTACHMENT,
  })

  const resolveColor = device.createTexture({
    width, height,
    format: "rgba16float",
    usage: TextureUsage.RENDER_ATTACHMENT | TextureUsage.SAMPLED,
  })

  const oitAccumulation = device.createTexture({
    width, height,
    format: "rgba16float",
    usage: TextureUsage.RENDER_ATTACHMENT | TextureUsage.SAMPLED,
  })

  const oitRevealage = device.createTexture({
    width, height,
    format: "r8unorm",
    usage: TextureUsage.RENDER_ATTACHMENT | TextureUsage.SAMPLED,
  })

  return { msaaColor, msaaDepth, resolveColor, oitAccumulation, oitRevealage }
}
```

---

## 6. Threading Model

Lynx is single-threaded for its core loop. The main thread owns the scene graph, the renderer, and all GPU resources. Computationally expensive offline tasks are offloaded to Web Workers to avoid blocking frame production.

```
┌──────────────────────────────────────────────────────────────────┐
│                        MAIN THREAD                               │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │
│  │  Scene   │  │ Renderer │  │Animation │  │  HTML Overlay │    │
│  │  Graph   │  │ Pipeline │  │  Mixer   │  │   Updates     │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘    │
│                                                                  │
│  Owns: all GPU resources, scene state, render loop               │
│  Runs: requestAnimationFrame loop                                │
└───────────────────────┬──────────────────────────────────────────┘
                        │
              postMessage / transferable
                        │
        ┌───────────────┼───────────────┐
        ▼                               ▼
┌───────────────────┐          ┌───────────────────┐
│   WORKER: Draco   │          │  WORKER: Basis    │
│   Mesh Decoding   │          │  Texture Transcode│
│                   │          │                   │
│  Input:           │          │  Input:           │
│   .drc bytes      │          │   .basis/.ktx2    │
│   (ArrayBuffer    │          │   (ArrayBuffer    │
│    transferred)   │          │    transferred)   │
│                   │          │                   │
│  Output:          │          │  Output:          │
│   Decoded vertex  │          │   Transcoded      │
│   + index data    │          │   texture data    │
│   (ArrayBuffer    │          │   (ArrayBuffer    │
│    transferred)   │          │    transferred)   │
└───────────────────┘          └───────────────────┘
```

### Worker Communication Protocol

Data transfers between the main thread and workers use transferable `ArrayBuffer` objects. This avoids copying large mesh and texture data. Once an `ArrayBuffer` is transferred, it becomes detached (zero-length) on the sending side, ensuring zero-copy semantics.

```typescript
interface WorkerRequest {
  id: number
  type: "decode-draco" | "transcode-basis"
  payload: ArrayBuffer  // Transferred, not copied
}

interface WorkerResponse {
  id: number
  type: "decode-draco-result" | "transcode-basis-result"
  payload: ArrayBuffer  // Transferred back
  meta: DecodeMeta | TranscodeMeta
}

// Main thread: sending work to the Draco worker
const decodeDraco = (dracoBytes: ArrayBuffer): Promise<DecodedMesh> => {
  const id = nextRequestId++

  return new Promise((resolve) => {
    pendingRequests.set(id, resolve)

    dracoWorker.postMessage(
      { id, type: "decode-draco", payload: dracoBytes } satisfies WorkerRequest,
      [dracoBytes]  // Transfer list: zero-copy handoff
    )
  })
}

// Main thread: receiving results
const handleWorkerMessage = (event: MessageEvent<WorkerResponse>) => {
  const { id, payload, meta } = event.data
  const resolve = pendingRequests.get(id)
  if (!resolve) return
  pendingRequests.delete(id)

  // payload is now owned by the main thread (transferred back)
  resolve({ buffer: payload, meta })
}
```

### Why Single-Threaded Rendering

The scene graph and renderer remain on the main thread for three reasons:

1. **WebGPU/WebGL2 contexts are bound to a single thread.** While `OffscreenCanvas` allows GPU work on a worker, transferring control of the canvas prevents the main thread from compositing HTML overlays.
2. **Scene graph mutations must be synchronized with rendering.** Splitting these across threads would require complex locking or double-buffering of the entire scene.
3. **Frame budgets are tight.** At 60 fps, a frame has 16.6 ms. The overhead of cross-thread synchronization would consume a significant fraction of that budget for small to mid-size scenes.

Offline tasks like Draco decoding (CPU-intensive decompression) and Basis transcoding (format conversion) are embarrassingly parallel and have no dependency on the current frame's scene state. They are ideal candidates for worker offloading.

---

## 7. Memory Management

Lynx's memory strategy is designed to eliminate per-frame allocations during steady-state rendering. Garbage collection pauses are the enemy of smooth frame delivery, so the engine pre-allocates typed array pools and reuses them every frame.

### 7.1 Per-Frame Typed Array Pools

Temporary data needed during a single frame (matrices, sort keys, command buffers) is drawn from pre-allocated pools. These pools are reset (not freed) at the start of each frame.

```typescript
interface FramePools {
  readonly mat4Pool: TypedArrayPool<Float32Array>
  readonly vec4Pool: TypedArrayPool<Float32Array>
  readonly vec3Pool: TypedArrayPool<Float32Array>
  readonly commandPool: TypedArrayPool<Uint32Array>
  readonly sortKeyPool: TypedArrayPool<Float64Array>
  resetAll: () => void
}

interface TypedArrayPool<T extends TypedArray> {
  readonly capacity: number
  readonly used: number
  acquire: (count: number) => T
  reset: () => void
}

const createTypedArrayPool = <T extends TypedArray>(
  ctor: TypedArrayConstructor<T>,
  initialCapacity: number
): TypedArrayPool<T> => {
  let buffer = new ctor(initialCapacity)
  let offset = 0

  const acquire = (count: number): T => {
    if (offset + count > buffer.length) {
      // Geometric growth: double capacity
      const newBuffer = new ctor(buffer.length * 2)
      newBuffer.set(buffer)
      buffer = newBuffer
    }
    const slice = buffer.subarray(offset, offset + count) as T
    offset += count
    return slice
  }

  const reset = () => {
    offset = 0
    // Note: buffer is NOT freed. It persists across frames.
  }

  return {
    get capacity() { return buffer.length },
    get used() { return offset },
    acquire,
    reset,
  }
}

const createFramePools = (): FramePools => {
  const mat4Pool = createTypedArrayPool(Float32Array, 4096 * 16)   // ~256 KB, 4096 matrices
  const vec4Pool = createTypedArrayPool(Float32Array, 2048 * 4)    // ~32 KB
  const vec3Pool = createTypedArrayPool(Float32Array, 2048 * 3)    // ~24 KB
  const commandPool = createTypedArrayPool(Uint32Array, 4096 * 8)  // ~128 KB, 4096 commands
  const sortKeyPool = createTypedArrayPool(Float64Array, 4096)     // ~32 KB

  const resetAll = () => {
    mat4Pool.reset()
    vec4Pool.reset()
    vec3Pool.reset()
    commandPool.reset()
    sortKeyPool.reset()
  }

  return { mat4Pool, vec4Pool, vec3Pool, commandPool, sortKeyPool, resetAll }
}
```

### 7.2 GPU Buffer Management

GPU buffers (uniform buffers, vertex buffers, index buffers) are long-lived. They are allocated once and resized only when the existing capacity is insufficient. Resizing uses geometric growth (typically 2x) to amortize the cost of reallocation.

```typescript
interface GrowableGPUBuffer {
  readonly buffer: GalBuffer
  readonly capacity: number
  readonly usage: BufferUsageFlags
  ensureCapacity: (requiredBytes: number) => void
  write: (data: ArrayBufferView, offset: number) => void
}

const createGrowableGPUBuffer = (
  device: GalDevice,
  usage: BufferUsageFlags,
  initialCapacity: number
): GrowableGPUBuffer => {
  let buffer = device.createBuffer({ size: initialCapacity, usage })
  let capacity = initialCapacity

  const ensureCapacity = (requiredBytes: number) => {
    if (requiredBytes <= capacity) return

    // Geometric growth: at least double, or the required size
    const newCapacity = Math.max(capacity * 2, requiredBytes)
    const newBuffer = device.createBuffer({ size: newCapacity, usage })

    // Old buffer will be garbage collected; GPU resources freed
    buffer.destroy()
    buffer = newBuffer
    capacity = newCapacity
  }

  const write = (data: ArrayBufferView, offset: number) => {
    ensureCapacity(offset + data.byteLength)
    buffer.write(data, offset)
  }

  return {
    get buffer() { return buffer },
    get capacity() { return capacity },
    usage,
    ensureCapacity,
    write,
  }
}
```

### 7.3 World Matrix Buffer

The world matrix buffer is a single large GPU uniform buffer that stores the 4x4 world matrix for every renderable object. Objects are assigned a stable index, and their matrix is written to `index * 64` bytes. During draw calls, a dynamic offset is used to bind the correct matrix without switching bind groups.

```typescript
const MATRIX_STRIDE = 64  // 16 floats * 4 bytes = 64 bytes per mat4

const updateWorldMatrixBuffer = (
  gpuBuffer: GrowableGPUBuffer,
  nodes: ReadonlyArray<SceneNode>,
  stagingBuffer: Float32Array
) => {
  // Ensure GPU buffer can hold all matrices
  gpuBuffer.ensureCapacity(nodes.length * MATRIX_STRIDE)

  // Write only dirty matrices into the staging buffer
  let anyDirty = false
  for (let i = 0; i < nodes.length; i++) {
    if (nodes[i].worldMatrixDirty) {
      stagingBuffer.set(nodes[i].worldMatrix as Float32Array, i * 16)
      anyDirty = true
    }
  }

  // Upload entire staging buffer if anything changed
  // (could be optimized to upload only dirty ranges)
  if (anyDirty) {
    gpuBuffer.write(stagingBuffer, 0)
  }
}
```

### 7.4 Steady-State Allocation Profile

During steady-state rendering (no new assets loading, no scene mutations beyond transforms), Lynx makes zero JavaScript heap allocations per frame. The allocation profile looks like this:

| Phase | Allocations | Strategy |
|-------|-------------|----------|
| Dirty flag propagation | 0 | In-place flag writes |
| World matrix update | 0 | Pool-allocated temp matrices, staging buffer reused |
| Frustum culling | 0 | Pre-allocated visible list (array `.length` reset, not reallocated) |
| Command list rebuild | 0 | Command pool with stable capacity |
| Command sorting | 0 | In-place sort of pre-allocated array |
| Shadow pass | 0 | Pre-allocated cascade data, GPU commands only |
| Opaque pass | 0 | GPU commands only |
| Transparency pass | 0 | GPU commands only |
| Post-process | 0 | GPU commands only |
| HTML overlay | 0 | Reuses Vec2 for screen projection, mutates `style.transform` |

The only allocations that occur during runtime are:

- **Scene mutations**: Adding or removing nodes allocates new `SceneNode` objects and may grow internal arrays.
- **Asset loading**: Loading meshes and textures allocates GPU resources and temporary decode buffers (offloaded to workers).
- **Resize**: Resizing the canvas destroys and recreates render targets.
- **Pool growth**: If the scene exceeds pool capacity, a one-time geometric growth allocation occurs. Once the pool stabilizes at the high-water mark, no further growth happens.

---

## Appendix: Coordinate System

Lynx uses a **Z-up right-handed coordinate system**:

```
        Z (up)
        │
        │
        │
        └──────── X (right)
       ╱
      ╱
     Y (forward)
```

- **X** points right
- **Y** points forward (into the screen in default camera orientation)
- **Z** points up
- Cross product: X x Y = Z

This is consistent with many CAD and simulation environments. The default camera looks along the +Y axis with +Z as up.

---

## Appendix: Key Type Definitions

```typescript
type Vec2 = Float32Array  // length 2: [x, y]
type Vec3 = Float32Array  // length 3: [x, y, z]
type Vec4 = Float32Array  // length 4: [x, y, z, w]
type Mat4 = Float32Array  // length 16: column-major 4x4 matrix
type Quat = Float32Array  // length 4: [x, y, z, w] unit quaternion

interface AABB {
  min: Vec3  // [minX, minY, minZ]
  max: Vec3  // [maxX, maxY, maxZ]
}

interface Frustum {
  planes: FrustumPlane[]  // 6 planes: near, far, left, right, top, bottom
}

interface FrustumPlane {
  normal: Vec3
  d: number
}

type TypedArray = Float32Array | Float64Array | Int32Array | Uint32Array | Uint16Array | Uint8Array

type TypedArrayConstructor<T extends TypedArray> = {
  new (length: number): T
  new (buffer: ArrayBuffer, byteOffset?: number, length?: number): T
}

interface EngineOptions {
  canvas: HTMLCanvasElement
  preferredBackend?: BackendType
  antialias?: boolean
  shadowMapSize?: number
  overlayContainer?: HTMLElement
  renderOptions?: RenderOptions
}

interface RenderOptions {
  msaaSamples?: 1 | 4
  shadowCascades?: 2 | 3 | 4
  bloomEnabled?: boolean
  bloomThreshold?: number
  toneMapping?: "aces" | "pbrNeutral" | "none"
  exposure?: number
}
```
