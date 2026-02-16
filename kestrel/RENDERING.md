# Rendering Pipeline

## Backend Abstraction

### Device Interface

The `Device` interface is the sole point of contact between Kestrel's renderer and the GPU. It is deliberately minimal — not a general-purpose GPU abstraction, but exactly the surface area Kestrel needs.

```typescript
interface Device {
  // Resource creation
  createBuffer(desc: BufferDescriptor): GpuBuffer
  createTexture(desc: TextureDescriptor): GpuTexture
  createSampler(desc: SamplerDescriptor): GpuSampler
  createPipeline(desc: PipelineDescriptor): GpuPipeline
  createBindGroup(layout: BindGroupLayout, entries: BindGroupEntry[]): GpuBindGroup
  createRenderTarget(desc: RenderTargetDescriptor): GpuRenderTarget

  // Buffer operations
  writeBuffer(buffer: GpuBuffer, data: ArrayBufferView, offset?: number): void

  // Frame lifecycle
  beginFrame(): void
  endFrame(): void

  // Render pass
  beginRenderPass(target: GpuRenderTarget, desc: RenderPassDescriptor): RenderPassHandle
  endRenderPass(pass: RenderPassHandle): void

  // Drawing
  setPipeline(pass: RenderPassHandle, pipeline: GpuPipeline): void
  setBindGroup(pass: RenderPassHandle, index: number, group: GpuBindGroup): void
  setVertexBuffer(pass: RenderPassHandle, slot: number, buffer: GpuBuffer): void
  setIndexBuffer(pass: RenderPassHandle, buffer: GpuBuffer, format: IndexFormat): void
  drawIndexed(pass: RenderPassHandle, indexCount: number, instanceCount?: number): void

  // Render bundles (WebGPU only — no-op on WebGL2)
  createRenderBundle(callback: (encoder: RenderBundleEncoder) => void): RenderBundle
  executeRenderBundle(pass: RenderPassHandle, bundle: RenderBundle): void

  // Capabilities
  readonly capabilities: DeviceCapabilities

  // Cleanup
  destroy(): void
}
```

### WebGPU Implementation

```
WebGPUDevice
├─ adapter: GPUAdapter
├─ device: GPUDevice
├─ context: GPUCanvasContext
├─ commandEncoder: GPUCommandEncoder (per frame)
└─ Maps Device interface directly to WebGPU API calls
```

Key advantages used:
- **Render bundles**: Pre-record draw commands for static geometry. Replay costs ~0 JS overhead.
- **Bind groups**: Group uniforms/textures. Swap entire groups instead of individual uniforms.
- **Pipeline state objects**: Fully baked pipeline state. No validation at draw time.

### WebGL2 Implementation

```
WebGL2Device
├─ gl: WebGL2RenderingContext
├─ stateCache: StateTracker (avoids redundant gl calls)
├─ vaoCache: Map<string, WebGLVertexArrayObject>
├─ uboCache: Map<string, WebGLBuffer>
└─ Translates Device interface to WebGL2 calls with state tracking
```

Key optimizations:
- **State cache**: Tracks current program, VAO, textures, blend state. Skips redundant `gl.*` calls.
- **UBOs**: Uniform Buffer Objects for per-frame and per-material data. Much cheaper than individual `gl.uniform*` calls.
- **VAOs**: Vertex Array Objects avoid re-binding vertex attributes per draw call.
- **FBO pooling**: Reuse framebuffer objects for render targets.

## Frame Structure

Each frame executes the following render passes in order:

```
Frame
├─ 1. Shadow Pass (CSM)
│   ├─ For each cascade (3):
│   │   ├─ Set viewport to cascade region in shadow atlas
│   │   ├─ Bind depth-only pipeline
│   │   └─ Draw shadow casters (sorted front-to-back for early-z)
│   └─ Output: Shadow atlas texture (depth)
│
├─ 2. Geometry Pass — Opaque
│   ├─ MSAA render target (color + depth + emissive)
│   ├─ Draw opaque meshes sorted by: pipeline → material → front-to-back
│   ├─ Vertex shader: transform, skinning, pass material index
│   ├─ Fragment shader: lighting (Lambert), shadows (CSM sample), AO
│   ├─ Write emissive to separate color attachment (MRT)
│   └─ Output: Color buffer, Depth buffer, Emissive buffer
│
├─ 3. Geometry Pass — Transparent (OIT)
│   ├─ Same MSAA render target, depth buffer read-only
│   ├─ Draw transparent meshes (no sorting needed)
│   ├─ Fragment shader: write to accumulation + revealage buffers
│   └─ Output: Accumulation buffer, Revealage buffer
│
├─ 4. MSAA Resolve
│   ├─ Resolve MSAA color → single-sample color
│   ├─ Resolve MSAA emissive → single-sample emissive
│   └─ Resolve OIT buffers if MSAA
│
├─ 5. OIT Composite
│   ├─ Full-screen pass
│   ├─ Blend accumulation/revealage onto resolved color
│   └─ Output: Final color with transparency composited
│
├─ 6. Bloom Pass
│   ├─ Threshold: extract bright pixels from emissive buffer
│   ├─ Downsample chain: 5 levels, each 1/2 resolution
│   ├─ Upsample chain: tent filter, accumulate
│   ├─ Composite: additive blend onto final color
│   └─ Output: Color with bloom applied
│
└─ 7. Final Blit
    ├─ Tonemap if needed (exposure, gamma)
    └─ Blit to canvas backbuffer
```

## MSAA Strategy

MSAA is applied at the geometry pass level, not as a post-process.

**WebGPU**:
- Create multisample render target with `sampleCount: 4`.
- Resolve to single-sample texture automatically via `resolveTarget` in the render pass descriptor.
- MSAA is essentially free on tile-based mobile GPUs (which resolve on-chip).

**WebGL2**:
- Create multisample renderbuffer via `gl.renderbufferStorageMultisample(GL.RENDERBUFFER, 4, ...)`.
- Resolve via `gl.blitFramebuffer()` from multisample FBO to single-sample FBO.

MSAA is resolved **before** post-processing (bloom, OIT composite) to avoid sampling artifacts.

## Pipeline / Shader Variants

Materials define a set of **feature flags** that select shader variants:

```typescript
type ShaderFeatures = {
  hasColorTexture: boolean
  hasAOTexture: boolean
  hasVertexColors: boolean
  hasMaterialIndex: boolean
  hasSkinning: boolean
  hasShadows: boolean
  hasEmissive: boolean
}
```

Each unique combination compiles to a distinct GPU pipeline. Pipelines are cached by their feature key:

```typescript
const key = `lambert_${features.hasColorTexture ? 'T' : 'F'}_${features.hasAOTexture ? 'T' : 'F'}_...`
```

This avoids branching in shaders (which kills GPU parallelism on mobile). Each variant is a straight-line shader.

## Draw Call Sorting

The render list is sorted to minimize state changes. Sort key (64-bit integer packed into two 32-bit values):

```
Bits 63-56: Pipeline ID (8 bits — up to 256 unique pipelines)
Bits 55-48: Material ID (8 bits — up to 256 unique materials)
Bits 47-32: Texture binding hash (16 bits)
Bits 31-0:  Depth (32 bits — front-to-back for opaque)
```

For 2000 draw calls, this sort is <0.1ms with a simple radix sort on the 64-bit key.

## Render List Management

The render list is **not rebuilt every frame**. It is a persistent sorted array that is incrementally updated:

- **Add**: When a mesh is added to the scene, insert into the render list at the correct sorted position.
- **Remove**: When removed, splice out.
- **Dirty**: When a material or texture changes, re-sort the affected entry.
- **Visibility**: Frustum culling sets a `visible` flag; the draw loop skips invisible entries without modifying the list.

This means per-frame cost of the render list is O(visible_count), not O(total_count × log(total_count)).

## Shader Language

- **WebGPU**: WGSL (native)
- **WebGL2**: GLSL 300 es

Shaders are authored in both languages in the `shaders/` directory. They share the same algorithmic structure but are written natively (no transpilation) to maximize each backend's strengths.

Shared constants (light counts, cascade counts, etc.) are injected via preprocessor defines (`#define`) in GLSL and via pipeline-overridable constants in WGSL.

## Uniform Buffer Layout

Per-frame uniforms (shared across all draw calls):
```
struct FrameUniforms {          // 256 bytes, std140/std430
  viewMatrix: mat4x4<f32>       // 64 bytes
  projectionMatrix: mat4x4<f32> // 64 bytes
  viewProjection: mat4x4<f32>   // 64 bytes
  cameraPosition: vec4<f32>     // 16 bytes
  ambientColor: vec4<f32>       // 16 bytes
  directionalDir: vec4<f32>     // 16 bytes
  directionalColor: vec4<f32>   // 16 bytes
}
```

Per-object uniforms:
```
struct ObjectUniforms {          // 80 bytes
  worldMatrix: mat4x4<f32>      // 64 bytes
  materialColor: vec4<f32>      // 16 bytes
}
```

Shadow uniforms:
```
struct ShadowUniforms {                  // 208 bytes
  cascadeViewProjections: mat4x4<f32>[3] // 192 bytes
  cascadeSplits: vec4<f32>               // 16 bytes
}
```

Per-frame uniforms are uploaded once. Per-object uniforms use a dynamic offset UBO (WebGL2) or dynamic bind group offsets (WebGPU) to avoid re-binding.

## Render Target Configuration

| Target | Format | MSAA | Purpose |
|---|---|---|---|
| Color | RGBA8Unorm | 4x | Main scene color |
| Depth | Depth24Plus | 4x | Depth testing |
| Emissive | RGBA8Unorm | 4x | Emissive for bloom extraction |
| OIT Accum | RGBA16Float | 4x | Weighted blended OIT accumulation |
| OIT Revealage | R8Unorm | 4x | OIT revealage channel |
| Shadow Atlas | Depth24Plus | 1x | CSM depth map (e.g., 2048×2048) |
| Bloom Mips | RGBA8Unorm | 1x | Downsample/upsample chain |
| Resolved Color | RGBA8Unorm | 1x | Post-MSAA resolve |
