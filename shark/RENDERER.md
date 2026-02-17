# Renderer Architecture

## Design Philosophy

The renderer uses a **command-oriented abstraction** that maps naturally to WebGPU's explicit API while remaining implementable on WebGL2's state-machine model. The abstraction is intentionally thin — it doesn't try to hide the GPU, it tries to present a common command vocabulary.

## GPU Abstraction Layer (GAL)

### Core Interfaces

```typescript
// All GPU resources are created through GpuDevice
interface GpuDevice {
  readonly backend: 'webgpu' | 'webgl2'
  createBuffer(desc: BufferDescriptor): GpuBuffer
  createTexture(desc: TextureDescriptor): GpuTexture
  createSampler(desc: SamplerDescriptor): GpuSampler
  createShaderModule(desc: ShaderModuleDescriptor): GpuShaderModule
  createPipeline(desc: PipelineDescriptor): GpuPipeline
  createBindGroup(desc: BindGroupDescriptor): GpuBindGroup
  createRenderPass(desc: RenderPassDescriptor): GpuRenderPass
  beginFrame(): GpuCommandEncoder
  submitFrame(encoder: GpuCommandEncoder): void
}

interface GpuCommandEncoder {
  beginRenderPass(pass: GpuRenderPass): GpuRenderPassEncoder
  copyBufferToBuffer(src, dst, size): void
}

interface GpuRenderPassEncoder {
  setPipeline(pipeline: GpuPipeline): void
  setBindGroup(index: number, group: GpuBindGroup): void
  setVertexBuffer(slot: number, buffer: GpuBuffer, offset?: number): void
  setIndexBuffer(buffer: GpuBuffer, format: IndexFormat): void
  draw(vertexCount, instanceCount?, firstVertex?, firstInstance?): void
  drawIndexed(indexCount, instanceCount?, firstIndex?, baseVertex?, firstInstance?): void
  end(): void
}
```

### Resource Types

**GpuBuffer**
```typescript
interface BufferDescriptor {
  size: number
  usage: BufferUsage          // VERTEX | INDEX | UNIFORM | STORAGE | COPY_SRC | COPY_DST
  data?: ArrayBufferView      // Initial data (optional)
  label?: string
}
```
- WebGPU: Direct mapping to `GPUBuffer`
- WebGL2: `WebGLBuffer` with usage hints (`gl.STATIC_DRAW`, `gl.DYNAMIC_DRAW`)

**GpuTexture**
```typescript
interface TextureDescriptor {
  width: number
  height: number
  depth?: number              // For texture arrays (CSM cascades)
  format: TextureFormat       // RGBA8, RGBA16F, DEPTH24, etc.
  usage: TextureUsage         // SAMPLED | RENDER_TARGET | COPY_DST
  sampleCount?: number        // MSAA (1 or 4)
  mipLevels?: number
  label?: string
}
```
- WebGPU: `GPUTexture` with views
- WebGL2: `WebGLTexture` (2D or 2D_ARRAY), `WebGLRenderbuffer` for MSAA

**GpuPipeline**
```typescript
interface PipelineDescriptor {
  vertex: { module: GpuShaderModule, entryPoint: string, buffers: VertexBufferLayout[] }
  fragment: { module: GpuShaderModule, entryPoint: string, targets: ColorTargetState[] }
  depthStencil?: DepthStencilState
  primitive: { topology: PrimitiveTopology, cullMode: CullMode, frontFace: FrontFace }
  multisample?: { count: number }
  label?: string
}
```
- WebGPU: Direct mapping to `GPURenderPipeline`
- WebGL2: Stores the complete state (blend, depth, cull, shader program, vertex layout) and applies it as a batch before drawing

### Bind Group Design

Bind groups are the key to efficient uniform management. They group related bindings that change at the same frequency:

```
BindGroup 0 — Per-frame (changes once per frame)
  ├── Camera matrices (view, projection, viewProjection, cameraPosition)
  ├── Ambient light (color, intensity)
  ├── Directional light (direction, color, intensity)
  └── Shadow matrices (3 cascade VP matrices + split depths)

BindGroup 1 — Per-material (changes per material switch)
  ├── Material uniforms (color, emissive, opacity, materialColorTable)
  ├── Color texture + sampler
  ├── AO texture + sampler
  └── Shadow map texture + sampler

BindGroup 2 — Per-object (changes per draw call)
  ├── Model matrix
  ├── Normal matrix
  └── Bone matrices texture (skinned meshes only)
```

**WebGPU**: Native bind groups — create once, bind per draw.
**WebGL2**: Emulated via UBOs (groups 0, 2) and individual uniform calls (textures). UBO binding points mirror bind group indices.

## Shader System

### Dual-Source Strategy

Since Shark has a small, fixed set of shader programs (~6 programs), we write both WGSL and GLSL 300 es versions. A shared header defines common math operations and struct layouts.

**Shader variants** (each has WGSL + GLSL versions):
1. `basic` — Unlit color/texture
2. `lambert` — Diffuse shading with ambient
3. `shadow_depth` — Depth-only for shadow map generation
4. `oit_accum` — OIT accumulation pass (transparent objects)
5. `oit_composite` — OIT compositing (fullscreen quad)
6. `bloom` — Bloom threshold / blur / composite (3 sub-passes)

### Preprocessor Defines

Shader variants use `#define` flags to toggle features within a single source:

```
HAS_VERTEX_COLORS      — Enable vertex color attribute
HAS_MATERIAL_INDEX     — Enable material index lookup
HAS_TEXTURE            — Enable color texture sampling
HAS_AO_TEXTURE         — Enable AO texture
HAS_SKINNING           — Enable bone matrix skinning (4 weights)
RECEIVE_SHADOWS        — Enable shadow map sampling
NUM_CSM_CASCADES       — Number of shadow cascades (3)
```

WebGPU uses WGSL `override` constants or `#if`-style preprocessor. WebGL2 uses `#define` in GLSL source before compilation.

### Pipeline Cache

Pipelines are expensive to create. The renderer maintains a cache keyed by:
```
hash(shaderVariant, defineFlags, blendMode, depthWrite, cullMode, vertexLayout)
```

On first encounter, the pipeline is created and cached. Subsequent draws with the same state reuse the cached pipeline.

## Render Bundles (WebGPU)

Render bundles are the primary performance advantage of the WebGPU backend.

### How They Work

1. **Recording phase** (once, or when scene changes):
   ```
   const bundleEncoder = device.createRenderBundleEncoder(descriptor)
   // Record all draw calls for static geometry
   for (const drawCall of staticDrawCalls) {
     bundleEncoder.setPipeline(drawCall.pipeline)
     bundleEncoder.setBindGroup(0, perFrameBindGroup)
     bundleEncoder.setBindGroup(1, drawCall.materialBindGroup)
     bundleEncoder.setBindGroup(2, drawCall.objectBindGroup)
     bundleEncoder.setVertexBuffer(0, drawCall.vertexBuffer)
     bundleEncoder.setIndexBuffer(drawCall.indexBuffer, 'uint16')
     bundleEncoder.drawIndexed(drawCall.indexCount)
   }
   const bundle = bundleEncoder.finish()
   ```

2. **Replay phase** (every frame):
   ```
   passEncoder.executeBundles([opaqueBundle])
   ```

### Bundle Invalidation

Bundles are invalidated when:
- Objects are added/removed from the scene
- Material properties change (triggers bind group rebuild)
- Geometry changes

Bundles are **not** invalidated by:
- Object transform changes (per-object UBO is updated, bind group reference unchanged)
- Camera movement (per-frame UBO updated separately)
- Animation (bone matrix texture updated separately)

### Static vs Dynamic Split

Objects are classified as:
- **Static**: Transform doesn't change frame-to-frame → bundled
- **Dynamic**: Animated meshes, moving objects → recorded each frame

The renderer maintains two lists and renders bundled objects first, then dynamic objects.

## WebGL2 Backend — State Sorting

Without render bundles, WebGL2 minimizes state changes through draw call sorting:

### Sort Key (64-bit integer)

```
Bits 63-60:  Render pass (shadow=0, opaque=1, transparent=2, post=3)
Bits 59-52:  Pipeline hash (shader + blend + depth state) — 256 variants
Bits 51-36:  Material ID — 65536 materials
Bits 35-20:  Geometry ID (VAO) — 65536 geometries
Bits 19-0:   Object ID — 1M objects
```

Draw calls are sorted by this key each frame using a radix sort (O(n), no allocations with pre-allocated buffers). Consecutive draw calls with the same upper bits require no state changes.

### WebGL2 State Tracker

To avoid redundant GL calls, a state tracker records the current GPU state:

```typescript
// Only call gl.bindBuffer if it's actually different
const setVertexBuffer = (slot: number, buffer: WebGLBuffer) => {
  if (currentState.vertexBuffers[slot] !== buffer) {
    gl.bindBuffer(gl.ARRAY_BUFFER, buffer)
    currentState.vertexBuffers[slot] = buffer
  }
}
```

This applies to all state: programs, textures, blend mode, depth test, cull mode, etc.

### Multi-Draw Extension

If `WEBGL_multi_draw` is available, consecutive draw calls with the same pipeline + material can be batched into a single `gl.multiDrawElementsWEBGL` call, further reducing driver overhead.

## Frame Lifecycle

```
beginFrame()
  ├── Update per-frame UBO (camera, lights, time)
  ├── Shadow pass
  │   ├── For each cascade:
  │   │   ├── Set viewport to cascade region in atlas
  │   │   ├── Bind shadow depth pipeline
  │   │   └── Draw shadow casters (front-face culling for bias)
  ├── MSAA render target setup
  ├── Opaque pass
  │   ├── WebGPU: executeBundles([opaqueStaticBundle]) + draw dynamic
  │   ├── WebGL2: iterate sorted draw calls, apply state changes, draw
  ├── OIT pass (if transparent objects exist)
  │   ├── Render to accumulation + revealage MRT
  │   └── Composite over opaque result
  ├── Bloom pass (if emissive objects exist)
  │   ├── Threshold (extract bright fragments)
  │   ├── Downsample chain (half, quarter, eighth res)
  │   ├── Gaussian blur at each level
  │   ├── Upsample + accumulate
  │   └── Composite with tone mapping
  ├── MSAA resolve
  └── Present to screen
submitFrame()
```

## Device Initialization

```typescript
const createDevice = async (canvas: HTMLCanvasElement): Promise<GpuDevice> => {
  // Try WebGPU first
  if (navigator.gpu) {
    const adapter = await navigator.gpu.requestAdapter()
    if (adapter) {
      const device = await adapter.requestDevice()
      const context = canvas.getContext('webgpu')
      return new WebGpuDevice(device, context)
    }
  }
  // Fall back to WebGL2
  const gl = canvas.getContext('webgl2', {
    antialias: false,           // We handle MSAA ourselves
    alpha: false,
    depth: true,
    stencil: false,
    powerPreference: 'high-performance',
    preserveDrawingBuffer: false,
  })
  if (!gl) throw new Error('Neither WebGPU nor WebGL2 is available')
  return new WebGl2Device(gl)
}
```

## Memory Management

- **Buffer pools**: Pre-allocated typed array pools for uniform data. Each frame writes into the pool, uploads the used portion.
- **Texture atlases**: Small textures packed into atlases to reduce bind calls.
- **Pipeline cache**: LRU cache with a generous size (few hundred entries max for a game).
- **No per-frame allocations**: All temporary math objects (`Vec3`, `Mat4`) are pre-allocated module-level scratch variables.
