# 01 — Hardware Abstraction Layer (HAL)

The HAL provides a unified GPU interface that both the WebGL2 and WebGPU backends implement. The abstraction is modeled after WebGPU's explicit programming model because it maps cleanly onto WebGL2's state machine (the reverse is not true).

## Core Abstractions

### Device

The entry point. Created via `createDevice(canvas, options?)`. Probes for WebGPU first, falls back to WebGL2.

```ts
type DeviceBackend = 'webgpu' | 'webgl2'

interface WrenDevice {
  readonly backend: DeviceBackend
  readonly canvas: HTMLCanvasElement
  readonly limits: DeviceLimits

  createBuffer(desc: BufferDescriptor): GPUBufferHandle
  createTexture(desc: TextureDescriptor): GPUTextureHandle
  createSampler(desc: SamplerDescriptor): GPUSamplerHandle
  createShaderModule(desc: ShaderModuleDescriptor): ShaderModuleHandle
  createPipeline(desc: PipelineDescriptor): PipelineHandle
  createBindGroup(desc: BindGroupDescriptor): BindGroupHandle
  createRenderTarget(desc: RenderTargetDescriptor): RenderTargetHandle

  beginFrame(): FrameEncoder
  submit(encoder: FrameEncoder): void

  destroy(): void
}
```

### FrameEncoder

Replaces WebGPU's `GPUCommandEncoder` + `GPURenderPassEncoder`. One active per frame.

```ts
interface FrameEncoder {
  beginRenderPass(desc: RenderPassDescriptor): RenderPassEncoder
  copyBufferToBuffer(src, dst, size): void
  finish(): void
}

interface RenderPassEncoder {
  setPipeline(pipeline: PipelineHandle): void
  setBindGroup(index: number, group: BindGroupHandle): void
  setVertexBuffer(slot: number, buffer: GPUBufferHandle, offset?: number): void
  setIndexBuffer(buffer: GPUBufferHandle, format: IndexFormat): void
  draw(vertexCount: number, instanceCount?: number, firstVertex?: number): void
  drawIndexed(indexCount: number, instanceCount?: number, firstIndex?: number): void
  end(): void
}
```

### Pipeline

Bundles shader + vertex layout + blend/depth/stencil state + primitive topology. This is the most important abstraction for performance — pipeline switches are the most expensive state change.

```ts
interface PipelineDescriptor {
  vertex: {
    module: ShaderModuleHandle
    entryPoint: string
    buffers: VertexBufferLayout[]
  }
  fragment: {
    module: ShaderModuleHandle
    entryPoint: string
    targets: ColorTargetState[]
  }
  depthStencil?: DepthStencilState
  primitive?: PrimitiveState
  multisample?: MultisampleState
}
```

### BindGroup

Groups uniform buffers, textures, and samplers into a single binding unit. Maps to WebGPU bind groups directly. On WebGL2, this is emulated by tracking which uniforms/textures need to be bound and applying them as a batch.

```ts
interface BindGroupDescriptor {
  layout: BindGroupLayoutHandle
  entries: BindGroupEntry[]
}

type BindGroupEntry =
  | { binding: number, buffer: GPUBufferHandle, offset?: number, size?: number }
  | { binding: number, texture: GPUTextureHandle }
  | { binding: number, sampler: GPUSamplerHandle }
```

## WebGL2 Backend Implementation

The WebGL2 backend emulates the WebGPU model by managing GL state internally:

### Pipeline → GL Program + State Block

```
PipelineHandle = {
  program: WebGLProgram
  blendState: { enabled, srcRGB, dstRGB, srcA, dstA, equation }
  depthState: { test, write, func }
  cullState: { enabled, face }
  stencilState: { ... }
}
```

When `setPipeline()` is called, the backend diffs against the current GL state and only issues `gl.*` calls for state that changed. This **state cache** is critical for performance.

### BindGroup → Uniform/Texture Binding Set

Each `BindGroupEntry` maps to:
- **Buffer binding** → `gl.bindBufferRange(GL.UNIFORM_BUFFER, bindingIndex, ...)`
- **Texture binding** → `gl.activeTexture(GL.TEXTURE0 + slot)` + `gl.bindTexture(...)`
- **Sampler binding** → `gl.samplerParameteri(...)` (WebGL2 has native sampler objects)

The backend tracks which UBO slots and texture units are currently bound and skips redundant binds.

### FrameEncoder → Immediate Execution

Unlike WebGPU, WebGL2 does not support deferred command recording. The WebGL2 `FrameEncoder` executes GL calls immediately. However, this is transparent to the caller — the sort-based renderer ensures calls arrive in optimal order regardless.

### Shader Translation

Shaders are authored as dual sources:

```
shaders/
  lambert.vert.glsl    # GLSL 300 es
  lambert.frag.glsl    # GLSL 300 es
  lambert.vert.wgsl    # WGSL
  lambert.frag.wgsl    # WGSL
```

A shared `#include`-style preprocessor handles common code (lighting calculations, skinning, shadow sampling). At build time, shader variants are generated for feature combinations:

| Variant Flag | Bit | Effect |
|---|---|---|
| `HAS_VERTEX_COLORS` | 0 | Enables `a_color` attribute |
| `HAS_SKINNING` | 1 | Enables bone matrix lookup |
| `HAS_SHADOW_MAP` | 2 | Enables CSM sampling |
| `HAS_AO_MAP` | 3 | Enables AO texture |
| `HAS_EMISSIVE` | 4 | Enables emissive output for bloom |
| `HAS_MATERIAL_INDEX` | 5 | Enables `a_materialIndex` attribute lookup |

Variant keys are packed into a bitmask. The renderer requests the appropriate variant from a shader cache, compiling on first use.

## WebGPU Backend Implementation

The WebGPU backend is a thin wrapper around the native API:

- `createPipeline` → `device.createRenderPipeline()`
- `createBindGroup` → `device.createBindGroup()`
- `beginFrame` → `device.createCommandEncoder()`
- `beginRenderPass` → `encoder.beginRenderPass()`
- `submit` → `device.queue.submit([encoder.finish()])`

### Render Bundles (WebGPU-only Enhancement)

For static geometry that doesn't change between frames, the renderer can optionally record draw calls into `GPURenderBundle` objects:

```ts
const bundleEncoder = device.createRenderBundleEncoder({
  colorFormats: ['bgra8unorm'],
  depthStencilFormat: 'depth24plus',
})
// Record draw calls...
const bundle = bundleEncoder.finish()

// Each frame:
renderPass.executeBundles([bundle])
```

This reduces per-frame JS→GPU overhead from N draw calls to 1 call. Bundles are invalidated and re-recorded when the set of visible static objects changes (after frustum culling).

### Compute-Based Frustum Culling (WebGPU-only Enhancement)

When WebGPU is available, frustum culling can run as a compute shader that reads object AABBs from a storage buffer and writes visible-object indices to an indirect draw buffer. This moves culling entirely off the CPU.

## Device Limits

Both backends expose a `DeviceLimits` object so higher-level code can adapt:

```ts
interface DeviceLimits {
  maxTextureSize: number            // 4096 (mobile) to 16384 (desktop)
  maxUniformBufferSize: number      // 16KB (WebGL2) to 64KB (WebGPU)
  maxStorageBufferSize: number      // 0 (WebGL2) or 128MB+ (WebGPU)
  maxBindGroups: number             // 4
  maxSampledTextures: number        // 16
  supportsRenderBundles: boolean    // WebGPU only
  supportsCompute: boolean          // WebGPU only
  supportsStorageBuffers: boolean   // WebGPU only
  supportsFloat32Filtering: boolean // Extension-dependent
}
```

## Buffer Management

Buffers are created with usage flags matching WebGPU's model:

```ts
type BufferUsage = 'vertex' | 'index' | 'uniform' | 'storage' | 'copy-src' | 'copy-dst'

interface BufferDescriptor {
  size: number
  usage: BufferUsage[]
  data?: ArrayBufferView
  label?: string
}
```

The WebGL2 backend maps usage to GL buffer targets:
- `vertex` → `GL.ARRAY_BUFFER`
- `index` → `GL.ELEMENT_ARRAY_BUFFER`
- `uniform` → `GL.UNIFORM_BUFFER`

Dynamic uniform data uses a **ring buffer** to avoid stalling the GPU pipeline. Each frame gets a fresh region of a persistent uniform buffer, cycling through 3 regions (triple buffering).

## Initialization

```ts
const device = await createDevice(canvas, {
  preferBackend: 'webgpu',       // Try WebGPU first
  antialias: true,               // MSAA
  powerPreference: 'high-performance',
})

console.log(device.backend) // 'webgpu' or 'webgl2'
```

The `createDevice` function:
1. Checks `navigator.gpu` availability
2. Calls `navigator.gpu.requestAdapter()` then `adapter.requestDevice()`
3. On failure (or if `preferBackend` is `'webgl2'`), falls back to `canvas.getContext('webgl2')`
4. Wraps the native context in the appropriate backend implementation
5. Returns the unified `WrenDevice`
