# Rendering Pipeline

## Hardware Abstraction Layer (HAL)

### Design Philosophy

The HAL is modeled on WebGPU's descriptor-based design. WebGPU is the "native" target; the WebGL2 backend maps this model onto WebGL2's stateful API with aggressive state caching.

The HAL is intentionally thin — no convenience methods, no smart defaults, no magic. Higher-level code (Renderer, materials) builds on it.

### HAL Interface

```typescript
// All HAL types are interfaces. Two implementations exist:
// WebGL2Device and WebGPUDevice.

interface HalDevice {
  readonly backend: 'webgl2' | 'webgpu'
  readonly limits: HalLimits
  readonly features: HalFeatures

  createBuffer(desc: HalBufferDescriptor): HalBuffer
  createTexture(desc: HalTextureDescriptor): HalTexture
  createSampler(desc: HalSamplerDescriptor): HalSampler
  createShader(desc: HalShaderDescriptor): HalShader
  createPipeline(desc: HalPipelineDescriptor): HalPipeline
  createBindGroup(desc: HalBindGroupDescriptor): HalBindGroup
  createRenderPass(desc: HalRenderPassDescriptor): HalRenderPass
  createCommandEncoder(): HalCommandEncoder

  // WebGPU-only: render bundles for cached command sequences
  createRenderBundle?(desc: HalRenderBundleDescriptor): HalRenderBundle

  submit(encoder: HalCommandEncoder): void
  destroy(): void
}
```

### Buffer Descriptors

```typescript
interface HalBufferDescriptor {
  size: number
  usage: BufferUsage           // VERTEX | INDEX | UNIFORM | COPY_DST
  data?: ArrayBufferView       // initial data (optional)
  label?: string
}

interface HalBuffer {
  write(data: ArrayBufferView, offset?: number): void
  destroy(): void
}
```

Buffers are created with a fixed size. For dynamic uniforms, we use a per-frame uniform buffer with sub-allocation (ring buffer strategy):

```
Frame N:   [uniform block 0] [uniform block 1] ... [uniform block K]
Frame N+1: [uniform block 0] [uniform block 1] ... [uniform block K]
           ^--- offset into the same large buffer
```

This avoids creating/destroying uniform buffers per frame. In WebGL2, this maps to `gl.bufferSubData` into a UBO. In WebGPU, this maps to `device.queue.writeBuffer` with dynamic offsets.

### Texture Descriptors

```typescript
interface HalTextureDescriptor {
  width: number
  height: number
  format: TextureFormat          // RGBA8, RGBA16F, DEPTH24, etc.
  usage: TextureUsage            // SAMPLED | RENDER_TARGET | COPY_DST
  sampleCount?: number           // 1 (default) or 4 (MSAA)
  mipLevelCount?: number
  data?: ArrayBufferView | ImageBitmap
  label?: string
}
```

### Pipeline Descriptors

```typescript
interface HalPipelineDescriptor {
  shader: HalShader
  vertexLayout: HalVertexLayout
  primitive: {
    topology: 'triangles' | 'lines' | 'points'
    cullMode: 'none' | 'front' | 'back'
    frontFace: 'ccw' | 'cw'
  }
  depthStencil?: {
    format: TextureFormat
    depthWrite: boolean
    depthCompare: CompareFunction
  }
  blend?: HalBlendState            // per-color-attachment blend
  multisample?: { count: number }
  label?: string
}
```

Pipelines encapsulate the full render state. This is the key performance lever — changing pipelines is the most expensive operation, so we sort draw calls by pipeline ID.

### Pipeline Caching

Pipelines are cached by a hash of their descriptor. The `PipelineCache` ensures we never create duplicate pipelines:

```typescript
const pipelineCache = new Map<number, HalPipeline>()

const getOrCreatePipeline = (desc: HalPipelineDescriptor): HalPipeline => {
  const hash = hashPipelineDescriptor(desc)
  let pipeline = pipelineCache.get(hash)
  if (!pipeline) {
    pipeline = device.createPipeline(desc)
    pipelineCache.set(hash, pipeline)
  }
  return pipeline
}
```

### Command Encoding

```typescript
interface HalCommandEncoder {
  beginRenderPass(desc: HalRenderPassDescriptor): HalRenderPassEncoder
}

interface HalRenderPassEncoder {
  setPipeline(pipeline: HalPipeline): void
  setBindGroup(index: number, bindGroup: HalBindGroup, dynamicOffsets?: number[]): void
  setVertexBuffer(slot: number, buffer: HalBuffer, offset?: number): void
  setIndexBuffer(buffer: HalBuffer, format: 'uint16' | 'uint32', offset?: number): void
  drawIndexed(indexCount: number, instanceCount?: number, firstIndex?: number): void
  draw(vertexCount: number, instanceCount?: number): void
  executeBundles?(bundles: HalRenderBundle[]): void  // WebGPU only
  end(): void
}
```

**WebGL2 mapping**: `HalCommandEncoder` and `HalRenderPassEncoder` execute GL calls immediately. `beginRenderPass` binds the framebuffer and sets viewport/clear. Each `setPipeline`/`setBindGroup`/`draw*` call maps directly to `gl.useProgram`, `gl.bindBufferRange`, `gl.drawElements`, etc., with a state cache to skip redundant calls.

**WebGPU mapping**: Commands are recorded into a `GPUCommandEncoder` → `GPURenderPassEncoder` and submitted at the end of the frame.

## WebGL2 State Cache

The WebGL2 backend maintains a shadow copy of all GL state to avoid redundant state changes:

```typescript
interface GL2StateCache {
  program: WebGLProgram | null
  vao: WebGLVertexArrayObject | null
  framebuffer: WebGLFramebuffer | null
  viewport: [number, number, number, number]
  depthWrite: boolean
  depthFunc: number
  cullFace: number
  blendEnabled: boolean
  // ... per-attachment blend state
  boundTextures: (WebGLTexture | null)[]
  boundUBOs: (WebGLBuffer | null)[]
}
```

Each `gl.*` call is preceded by a check: `if (cache.program !== newProgram) { gl.useProgram(newProgram); cache.program = newProgram }`. This eliminates ~40-60% of redundant GL calls in typical scenes.

## WebGPU Render Bundles

For scenes with many static objects, WebGPU render bundles pre-record draw call sequences:

```typescript
// Build bundle once (or when scene changes)
const bundleEncoder = device.createRenderBundleEncoder({
  colorFormats: ['rgba8unorm'],
  depthStencilFormat: 'depth24plus',
  sampleCount: 4,
})
for (const item of staticOpaqueList) {
  bundleEncoder.setPipeline(item.pipeline)
  bundleEncoder.setBindGroup(0, item.bindGroup)
  bundleEncoder.setVertexBuffer(0, item.vertexBuffer)
  bundleEncoder.setIndexBuffer(item.indexBuffer, 'uint16')
  bundleEncoder.drawIndexed(item.indexCount)
}
const bundle = bundleEncoder.finish()

// Each frame: execute bundle instead of re-recording
passEncoder.executeBundles([bundle])
```

Bundles are invalidated when objects are added/removed from the scene or materials change. A dirty flag on the scene triggers bundle rebuild.

## MSAA

MSAA is implemented by rendering to a multisampled render target and resolving to a single-sampled texture:

```
Render target (4x MSAA) ──resolve──▶ Resolved texture ──▶ Post-processing
```

**WebGL2**: Create multisampled renderbuffers, attach to FBO, resolve via `gl.blitFramebuffer`.

**WebGPU**: Create multisampled texture (`sampleCount: 4`), set `resolveTarget` on the render pass color attachment.

The MSAA sample count is configurable (1, 2, 4) but defaults to 4. On low-end mobile GPUs, it can be set to 1 (disabled) via engine config.

## Render Loop

```typescript
const render = (scene: Scene, camera: Camera) => {
  // 1. Update camera matrices
  camera.updateViewProjectionMatrix()

  // 2. Extract frustum planes from VP matrix
  frustum.extractFromMatrix(camera.viewProjectionMatrix)

  // 3. Traverse scene graph, collect visible renderables
  const opaqueList: RenderItem[] = []
  const transparentList: RenderItem[] = []

  scene.traverse((node) => {
    if (node instanceof Mesh) {
      if (!frustum.intersectsAABB(node.worldAABB)) return // culled

      const item: RenderItem = {
        node,
        pipelineId: node.material.pipelineId,
        sortKey: node.material.transparent
          ? 0 // OIT doesn't need sorting
          : computeSortKey(node.material.pipelineId, camera, node),
      }

      if (node.material.transparent) {
        transparentList.push(item)
      } else {
        opaqueList.push(item)
      }
    }
  })

  // 4. Sort opaque list: by pipeline first, then front-to-back within pipeline
  opaqueList.sort(compareRenderItems)

  // 5. Shadow pass
  renderShadowMaps(scene, opaqueList)

  // 6. Opaque pass (MSAA render target)
  renderOpaquePass(opaqueList, camera)

  // 7. Transparent pass (OIT targets)
  renderTransparentPass(transparentList, camera)

  // 8. MSAA resolve
  resolveMultisample()

  // 9. OIT composite
  compositeOIT()

  // 10. Bloom
  bloomPass.execute()

  // 11. Tone mapping → screen
  toneMappingPass.execute()
}
```

## Sort Key Encoding

To minimize sorting cost, we encode the sort key as a single 64-bit number (two 32-bit ints in a Float64Array):

```
Opaque sort key (front-to-back within pipeline):
  [32 bits: pipeline ID] [32 bits: depth as uint]

This ensures:
  1. Objects with the same pipeline are grouped together (fewer state changes)
  2. Within a pipeline, objects are front-to-back (better early-Z rejection)
```

We use `Float64Array` as a sort-key holder since JS can compare 64-bit floats natively with `<` operator, avoiding multi-field comparisons.

## Shader Architecture

Shaders are written in WGSL (WebGPU) and GLSL 300 es (WebGL2). Both versions are maintained as template strings with `#define`-based feature toggling:

```glsl
// GLSL 300 es vertex shader template
#ifdef HAS_VERTEX_COLORS
  in vec4 a_color;
  out vec4 v_color;
#endif

#ifdef HAS_MATERIAL_INDEX
  in float a_materialIndex;
  flat out int v_materialIndex;
#endif

#ifdef HAS_SKINNING
  in vec4 a_joints;
  in vec4 a_weights;
  uniform BoneMatrices {
    mat4 u_bones[MAX_BONES];
  };
#endif
```

The `ShaderGenerator` composes the final shader source from these chunks based on material features:

```typescript
const generateShader = (material: Material, features: ShaderFeatures): string => {
  const defines: string[] = []
  if (features.hasVertexColors) defines.push('#define HAS_VERTEX_COLORS')
  if (features.hasMaterialIndex) defines.push('#define HAS_MATERIAL_INDEX')
  if (features.hasSkinning) defines.push('#define HAS_SKINNING')
  if (features.hasColorMap) defines.push('#define HAS_COLOR_MAP')
  if (features.hasAOMap) defines.push('#define HAS_AO_MAP')
  if (features.hasEmissive) defines.push('#define HAS_EMISSIVE')
  if (features.hasShadows) defines.push('#define HAS_SHADOWS')
  // ...
  return defines.join('\n') + '\n' + shaderSource
}
```

For WebGPU, an equivalent WGSL generation approach is used with override constants and conditional compilation via preprocessing.
