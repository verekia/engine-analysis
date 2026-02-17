# Renderer & Backend Abstraction

## GPUDevice Interface

The `GPUDevice` is the central abstraction over WebGL 2 and WebGPU. It owns the GPU context and provides methods for resource creation and command submission.

```typescript
interface GPUDevice {
  readonly backend: 'webgl2' | 'webgpu'
  readonly canvas: HTMLCanvasElement
  readonly limits: GPULimits

  // Resource creation
  createBuffer(desc: BufferDescriptor): GPUBufferHandle
  createTexture(desc: TextureDescriptor): GPUTextureHandle
  createSampler(desc: SamplerDescriptor): GPUSamplerHandle
  createShader(desc: ShaderDescriptor): GPUShaderHandle
  createRenderTarget(desc: RenderTargetDescriptor): GPURenderTargetHandle

  // Resource updates
  writeBuffer(handle: GPUBufferHandle, data: ArrayBufferView, offset?: number): void
  writeTexture(handle: GPUTextureHandle, data: TexImageSource | ArrayBufferView, level?: number): void

  // Command encoding
  beginRenderPass(desc: RenderPassDescriptor): RenderPassEncoder
  submit(): void

  // Cleanup
  destroyBuffer(handle: GPUBufferHandle): void
  destroyTexture(handle: GPUTextureHandle): void
  destroyShader(handle: GPUShaderHandle): void

  // Capabilities
  supportsMultiDraw(): boolean
  supportsRenderBundles(): boolean
}
```

### Handle-Based Resources

Resources are referenced by opaque integer handles, not wrapper objects. This avoids GC pressure and allows the backend to store resource data in flat arrays:

```typescript
type GPUBufferHandle = number & { __brand: 'GPUBuffer' }
type GPUTextureHandle = number & { __brand: 'GPUTexture' }
type GPUShaderHandle = number & { __brand: 'GPUShader' }
```

Internally, each backend maps handles to native objects via a simple array lookup:

```typescript
// WebGL2Backend internal storage
const glBuffers: WebGLBuffer[] = []      // glBuffers[handle] → WebGLBuffer
const glTextures: WebGLTexture[] = []    // glTextures[handle] → WebGLTexture
const glPrograms: WebGLProgram[] = []    // glPrograms[handle] → WebGLProgram
```

## WebGL 2 Backend

### State Tracking

The WebGL 2 backend maintains a shadow copy of all GPU state to avoid redundant `gl.bindX` and `gl.uniformX` calls:

```typescript
interface GL2State {
  activeProgram: WebGLProgram | null
  activeVAO: WebGLVertexArrayObject | null
  activeTextures: (WebGLTexture | null)[]   // per texture unit
  depthTest: boolean
  depthWrite: boolean
  blendMode: BlendMode
  cullFace: CullFace
  viewport: [number, number, number, number]
  scissor: [number, number, number, number] | null
}
```

Before each state change, the backend checks the shadow state. Only changed states issue GL calls. Combined with state sorting (see below), this minimizes driver overhead.

### Uniform Buffer Objects (UBOs)

Shared data (camera matrices, light parameters, shadow matrices) goes into UBOs bound at fixed binding points:

| Binding | Name | Contents |
|---------|------|----------|
| 0 | `CameraUBO` | viewMatrix, projectionMatrix, viewProjection, cameraPosition, near, far |
| 1 | `LightsUBO` | directionalLight (direction, color), ambientColor, shadowMatrices[3], cascadeSplits[3] |
| 2 | `ModelUBO` | worldMatrix, normalMatrix (per draw call) |

UBO layout uses `std140` rules for cross-platform compatibility. The `ModelUBO` is the only one updated per draw call — it's a single `bufferSubData` of 128 bytes (two Mat4s).

### WEBGL_multi_draw Extension

When available (`WEBGL_multi_draw`), the renderer batches multiple draw calls that share the same VAO and shader into a single `gl.multiDrawElementsWEBGL` call. This dramatically reduces driver overhead:

```typescript
// Without multi_draw: 200 draw calls = 200 gl.drawElements
// With multi_draw: 200 draw calls sharing same shader/VAO = 1 gl.multiDrawElementsWEBGL

const ext = gl.getExtension('WEBGL_multi_draw')
if (ext && batch.length > 1) {
  ext.multiDrawElementsWEBGL(
    gl.TRIANGLES,
    batch.counts, 0,       // count per draw
    gl.UNSIGNED_SHORT,
    batch.offsets, 0,      // byte offset per draw
    batch.length           // number of draws
  )
}
```

For this to work, geometries that share the same material are packed into a shared vertex/index buffer where possible.

### MSAA (WebGL 2)

MSAA is implemented via a multisampled renderbuffer:

```
1. Create multisampled renderbuffer (4 samples)
2. Create resolve framebuffer (regular texture)
3. Render scene → multisampled renderbuffer
4. gl.blitFramebuffer → resolve to regular texture
5. Post-processing reads from resolved texture
```

## WebGPU Backend

### Render Pipeline Caching

WebGPU requires pre-created render pipelines for each combination of:
- Shader module
- Vertex buffer layout
- Blend state
- Depth/stencil state
- Render target format

Caracal caches pipelines using a composite key derived from these parameters:

```typescript
const pipelineKey = `${shaderHandle}:${vertexLayout}:${blendMode}:${depthWrite}:${targetFormat}`
```

Since we have only 2 material types and a small set of geometry layouts, the pipeline cache stays small (typically <20 entries).

### Bind Groups

WebGPU bind groups map to UBO binding points:

| Group | Description | Update frequency |
|-------|-------------|-----------------|
| 0 | Camera + Lights | Once per frame |
| 1 | Material uniforms + textures | Once per material |
| 2 | Model transform (worldMatrix) | Once per draw call |

Group 0 is set once per render pass. Group 1 changes per material. Group 2 changes per object. This layout minimizes bind group switches.

### Render Bundles

For static geometry that doesn't change between frames, WebGPU render bundles pre-encode the draw commands:

```typescript
const bundleEncoder = device.createRenderBundleEncoder({
  colorFormats: ['bgra8unorm'],
  depthStencilFormat: 'depth24plus',
  sampleCount: 4,
})

// Encode all static draw calls once
for (const drawCall of staticDrawCalls) {
  bundleEncoder.setPipeline(drawCall.pipeline)
  bundleEncoder.setBindGroup(0, cameraBindGroup)
  bundleEncoder.setBindGroup(1, drawCall.materialBindGroup)
  bundleEncoder.setBindGroup(2, drawCall.modelBindGroup)
  bundleEncoder.setVertexBuffer(0, drawCall.vertexBuffer)
  bundleEncoder.setIndexBuffer(drawCall.indexBuffer, 'uint16')
  bundleEncoder.drawIndexed(drawCall.indexCount)
}

const bundle = bundleEncoder.finish()

// Each frame: just execute the bundle (near-zero CPU cost)
renderPass.executeBundles([bundle])
```

Bundles are invalidated when objects are added/removed or materials change, but for static scenes they eliminate virtually all per-frame CPU work.

### MSAA (WebGPU)

WebGPU has native multisample texture support:

```typescript
const msaaTexture = device.createTexture({
  size: [width, height],
  sampleCount: 4,
  format: 'bgra8unorm',
  usage: GPUTextureUsage.RENDER_ATTACHMENT,
})

const renderPassDesc = {
  colorAttachments: [{
    view: msaaTexture.createView(),
    resolveTarget: swapChainTexture.createView(), // auto-resolve
    loadOp: 'clear',
    storeOp: 'store',
  }],
}
```

The resolve happens automatically at the end of the render pass — no manual blit needed.

## Render Loop Architecture

The renderer executes a multi-pass pipeline each frame:

```
Frame Start
│
├─ Pass 1: Shadow Maps (CSM)
│   ├─ Cascade 0: render shadow casters → shadow atlas region 0
│   ├─ Cascade 1: render shadow casters → shadow atlas region 1
│   └─ Cascade 2: render shadow casters → shadow atlas region 2
│
├─ Pass 2: Opaque Geometry
│   ├─ Target: MSAA color + depth
│   ├─ Draw order: state-sorted (shader → material → front-to-back)
│   └─ Writes: color, depth
│
├─ Pass 3: Transparent Geometry (OIT Accumulation)
│   ├─ Target: OIT accumulation (RGBA16F) + revealage (R8)
│   ├─ Draw order: any (OIT is order-independent)
│   ├─ Depth: read-only (uses opaque depth buffer)
│   └─ Blend: additive for accum, multiplicative for revealage
│
├─ Pass 4: OIT Composite
│   ├─ Target: MSAA color (composites over opaque)
│   └─ Full-screen quad blending OIT result
│
├─ Pass 5: MSAA Resolve
│   └─ Resolve MSAA → single-sample texture
│
├─ Pass 6: Bloom
│   ├─ Extract bright pixels (emissive threshold)
│   ├─ Progressive downscale (½, ¼, ⅛, 1/16)
│   ├─ Separable Gaussian blur at each level
│   ├─ Progressive upscale + composite
│   └─ Add bloom to resolved image
│
└─ Pass 7: Tone Map + Output
    └─ ACES tone mapping → canvas/swapchain
```

## State Sorting

Draw calls are sorted using a 64-bit sort key packed into two 32-bit integers (for radix sort efficiency):

```
┌──────────┬──────────┬──────────┬──────────────────────┐
│ Shader   │ Material │ Geometry │ Depth (front→back)   │
│ (8 bits) │ (8 bits) │ (8 bits) │ (8 bits)             │
└──────────┴──────────┴──────────┴──────────────────────┘
     MSB                                           LSB
```

- **Shader** (most significant): GPU pipeline/program switches are the most expensive
- **Material**: Texture and uniform changes are next
- **Geometry**: VAO/vertex buffer switches
- **Depth**: Front-to-back for early-Z rejection (less significant since state changes matter more)

The sort uses a single-pass **radix sort** on the 32-bit key, which is O(n) and allocation-free (uses pre-allocated scratch arrays).

## Render Target Management

The renderer pre-allocates a set of render targets at initialization and resizes them when the canvas dimensions change:

| Target | Format | Purpose |
|--------|--------|---------|
| `msaaColor` | RGBA8 (4× MSAA) | Main scene render |
| `msaaDepth` | Depth24 (4× MSAA) | Depth buffer for scene |
| `resolvedColor` | RGBA8 | MSAA resolve target |
| `oitAccum` | RGBA16F | OIT accumulation (weighted color) |
| `oitRevealage` | R8 | OIT revealage (alpha product) |
| `shadowAtlas` | Depth16 | Cascaded shadow maps (2048×2048) |
| `bloomMips[0..3]` | RGBA8 (½, ¼, ⅛, 1/16) | Bloom downscale/blur chain |

All targets are created eagerly at startup. On resize, they're destroyed and recreated. This avoids any allocation during rendering.
