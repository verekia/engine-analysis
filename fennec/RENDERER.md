# Renderer — GPU Abstraction Layer

## Overview

The renderer sits at the heart of Fennec. It provides a thin **GPU Abstraction Layer (GAL)** that unifies WebGPU and WebGL2 behind a common interface. The abstraction is intentionally low-level — it doesn't try to hide the GPU programming model, it just normalizes the differences.

## Backend Selection

```typescript
const createEngine = async (options: EngineOptions): Promise<Engine> => {
  // 1. Try WebGPU first
  if (options.preferWebGPU !== false && navigator.gpu) {
    const adapter = await navigator.gpu.requestAdapter()
    if (adapter) {
      const device = await adapter.requestDevice()
      return new Engine(new WebGPUBackend(device, options), options)
    }
  }
  // 2. Fall back to WebGL2
  const gl = canvas.getContext('webgl2', {
    antialias: false, // We handle MSAA ourselves
    alpha: false,
    stencil: false,
    powerPreference: options.powerPreference ?? 'high-performance',
  })
  if (!gl) throw new Error('Neither WebGPU nor WebGL2 available')
  return new Engine(new WebGL2Backend(gl, options), options)
}
```

## GAL Interface

The GAL defines the minimal set of GPU operations both backends must implement:

```typescript
interface GPUBackend {
  // Resources
  createBuffer(desc: BufferDescriptor): GPUBufferHandle
  createTexture(desc: TextureDescriptor): GPUTextureHandle
  createSampler(desc: SamplerDescriptor): GPUSamplerHandle
  createShader(desc: ShaderDescriptor): GPUShaderHandle
  createPipeline(desc: PipelineDescriptor): GPUPipelineHandle
  createRenderTarget(desc: RenderTargetDescriptor): GPURenderTargetHandle

  // Uniform management
  createUniformBuffer(sizeBytes: number): GPUUniformBufferHandle
  updateUniformBuffer(handle: GPUUniformBufferHandle, data: Float32Array, offsetBytes: number): void

  // Commands
  beginRenderPass(target: GPURenderTargetHandle, desc: RenderPassDescriptor): void
  setPipeline(pipeline: GPUPipelineHandle): void
  setVertexBuffer(slot: number, buffer: GPUBufferHandle): void
  setIndexBuffer(buffer: GPUBufferHandle, format: IndexFormat): void
  setBindGroup(slot: number, group: BindGroupDescriptor): void
  draw(vertexCount: number, instanceCount: number, firstVertex: number): void
  drawIndexed(indexCount: number, instanceCount: number, firstIndex: number): void
  endRenderPass(): void

  // Lifecycle
  submit(): void
  resize(width: number, height: number): void
  destroy(): void

  // Capabilities
  readonly capabilities: BackendCapabilities
}
```

## WebGPU Backend Details

### Render Bundles for Static Scenes

WebGPU render bundles pre-record draw commands into a reusable command buffer. For objects whose state doesn't change frame-to-frame (same material, same geometry, same pipeline), we record them into a render bundle once and replay it every frame with zero JS overhead.

```typescript
// Record bundle once
const bundleEncoder = device.createRenderBundleEncoder({
  colorFormats: ['bgra8unorm'],
  depthStencilFormat: 'depth24plus',
  sampleCount: 4,
})
// ... encode draws ...
const bundle = bundleEncoder.finish()

// Replay every frame (near-zero cost)
renderPassEncoder.executeBundles([bundle])
```

**Invalidation**: Bundles are invalidated when objects are added/removed, materials change, or visibility changes. A dirty flag on the scene triggers re-recording.

### Bind Groups

WebGPU bind groups batch uniform/texture/sampler bindings into a single unit. Fennec uses a 4-slot bind group layout:

| Slot | Contents | Update Frequency |
|------|----------|-----------------|
| 0 | Frame UBO (camera VP matrix, time, resolution) | Once per frame |
| 1 | Light UBO (directional light, ambient, shadow matrices) | Once per frame |
| 2 | Material bind group (material UBO, textures, samplers) | Per material |
| 3 | Object bind group (model matrix, bone matrices) | Per object |

This layout minimizes bind group switches. Slot 0 and 1 are bound once per frame. Slot 2 changes only when material changes (draw calls are sorted to minimize this). Slot 3 changes per object but is a small UBO update.

### Shader Compilation

WebGPU uses WGSL. Fennec generates WGSL from a template system with `#define`-like preprocessor directives:

```wgsl
// Generated vertex shader snippet
@group(0) @binding(0) var<uniform> frame: FrameUniforms;
@group(3) @binding(0) var<uniform> object: ObjectUniforms;

#if HAS_SKINNING
@group(3) @binding(1) var<storage, read> boneMatrices: array<mat4x4f>;
#endif

@vertex fn vs(@location(0) position: vec3f, ...) -> VSOutput {
  #if HAS_SKINNING
  let skinnedPos = applySkinning(position, joints, weights, boneMatrices);
  #else
  let skinnedPos = position;
  #endif
  // ...
}
```

## WebGL2 Backend Details

### Uniform Buffer Objects (UBOs)

WebGL2 supports UBOs, which we use identically to WebGPU's uniform buffers:

```typescript
// Bind UBO to binding point
gl.bindBufferBase(gl.UNIFORM_BUFFER, 0, frameUBO)
gl.bindBufferBase(gl.UNIFORM_BUFFER, 1, lightUBO)

// Per-object update (sub-buffer update, no full re-upload)
gl.bindBuffer(gl.UNIFORM_BUFFER, objectUBO)
gl.bufferSubData(gl.UNIFORM_BUFFER, 0, objectData)
gl.bindBufferBase(gl.UNIFORM_BUFFER, 3, objectUBO)
```

### VAO State Caching

Every geometry creates a VAO (Vertex Array Object) that caches all vertex attribute state. Switching geometries is a single `gl.bindVertexArray()` call.

### State Sorting for Minimal GL Calls

The WebGL2 backend tracks current GPU state and skips redundant calls:

```typescript
// Internal state tracking
let _currentProgram: WebGLProgram | null = null
let _currentVAO: WebGLVertexArrayObject | null = null
let _currentMaterial: number = -1

const setProgram = (program: WebGLProgram) => {
  if (program === _currentProgram) return
  gl.useProgram(program)
  _currentProgram = program
}
```

### Shader Compilation

WebGL2 uses GLSL ES 3.0. Shaders are generated from the same template system, compiled to GLSL:

```glsl
#version 300 es
precision highp float;

layout(std140) uniform FrameUniforms {
  mat4 viewProjection;
  vec4 cameraPosition;
  float time;
};

#ifdef HAS_SKINNING
uniform highp sampler2D boneTexture;
#endif
```

## MSAA Implementation

Both backends use a multi-sampled render target that is resolved before post-processing.

### WebGPU MSAA

```typescript
const msaaTexture = device.createTexture({
  size: [width, height],
  sampleCount: 4,
  format: 'bgra8unorm',
  usage: GPUTextureUsage.RENDER_ATTACHMENT,
})

// Render pass resolves MSAA → single-sample texture
const renderPassDesc = {
  colorAttachments: [{
    view: msaaTexture.createView(),
    resolveTarget: resolvedTexture.createView(),
    loadOp: 'clear',
    storeOp: 'discard', // MSAA texture discarded after resolve
  }],
}
```

### WebGL2 MSAA

```typescript
// Create multisampled renderbuffer
gl.bindRenderbuffer(gl.RENDERBUFFER, msaaColorRB)
gl.renderbufferStorageMultisample(gl.RENDERBUFFER, 4, gl.RGBA8, width, height)

// Resolve via blit
gl.bindFramebuffer(gl.READ_FRAMEBUFFER, msaaFBO)
gl.bindFramebuffer(gl.DRAW_FRAMEBUFFER, resolvedFBO)
gl.blitFramebuffer(0, 0, w, h, 0, 0, w, h, gl.COLOR_BUFFER_BIT, gl.NEAREST)
```

## Render Loop

```typescript
const renderFrame = (scene: Scene, camera: Camera) => {
  // 1. Update transforms
  scene.updateWorldMatrices()

  // 2. Frustum cull
  const visibleObjects = frustumCull(scene.objects, camera.frustum)

  // 3. Build render lists
  const { opaque, transparent } = buildRenderLists(visibleObjects)

  // 4. Sort opaque by sort key (shader|material|geometry)
  sortBySortKey(opaque)

  // 5. Shadow pass
  if (scene.shadowLight) {
    renderShadowMaps(scene.shadowLight, scene.objects, camera)
  }

  // 6. Upload shared uniforms
  uploadFrameUniforms(camera)
  uploadLightUniforms(scene.lights)

  // 7. Main render pass
  backend.beginRenderPass(msaaTarget, { clear: true, clearColor: scene.background })

  // 7a. Opaque pass
  for (const item of opaque) {
    bindMaterial(item.material)
    bindObject(item)
    backend.drawIndexed(item.geometry.indexCount, 1, 0)
  }

  // 7b. OIT accumulation pass (transparent)
  if (transparent.length > 0) {
    beginOITAccumulation()
    for (const item of transparent) {
      bindMaterial(item.material)
      bindObject(item)
      backend.drawIndexed(item.geometry.indexCount, 1, 0)
    }
    endOITAccumulation()
    compositeOIT()
  }

  backend.endRenderPass()

  // 8. Post-processing
  resolveMSAA()
  renderBloom()
  finalBlit()

  // 9. Submit
  backend.submit()
}
```

## Resize Handling

On resize (window, devicePixelRatio change), all render targets are recreated. WebGPU textures are destroyed and reallocated. WebGL2 renderbuffers are resized. A `ResizeObserver` on the canvas container triggers this.

```typescript
const resizeObserver = new ResizeObserver((entries) => {
  const { width, height } = entries[0].contentRect
  const dpr = Math.min(window.devicePixelRatio, 2) // Cap at 2x for mobile
  canvas.width = Math.floor(width * dpr)
  canvas.height = Math.floor(height * dpr)
  backend.resize(canvas.width, canvas.height)
})
```
