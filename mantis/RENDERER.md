# Renderer & GPU Abstraction Layer

## GPU Abstraction Layer (GAL)

The GAL provides a thin interface over WebGPU and WebGL2. It is modeled after
WebGPU's concepts because WebGPU's API was designed to abstract over
Vulkan/Metal/D3D12 — it naturally abstracts over WebGL2 as well. The abstraction
is **not** a general-purpose WebGPU polyfill. It covers only the subset Mantis
needs, keeping the interface small and the backends simple.

### Core Interfaces

```typescript
interface GALDevice {
  createBuffer(desc: BufferDesc): GALBuffer
  createTexture(desc: TextureDesc): GALTexture
  createSampler(desc: SamplerDesc): GALSampler
  createShaderModule(desc: ShaderModuleDesc): GALShaderModule
  createPipeline(desc: PipelineDesc): GALPipeline
  createBindGroupLayout(desc: BindGroupLayoutDesc): GALBindGroupLayout
  createBindGroup(desc: BindGroupDesc): GALBindGroup
  beginRenderPass(desc: RenderPassDesc): GALRenderPass
  submitCommands(): void
  writeBuffer(buffer: GALBuffer, offset: number, data: ArrayBufferView): void
  writeTexture(texture: GALTexture, data: ArrayBufferView, layout: TextureLayout): void
  canvas: HTMLCanvasElement
  limits: DeviceLimits
}

interface GALRenderPass {
  setPipeline(pipeline: GALPipeline): void
  setVertexBuffer(slot: number, buffer: GALBuffer, offset?: number): void
  setIndexBuffer(buffer: GALBuffer, format: IndexFormat): void
  setBindGroup(index: number, group: GALBindGroup, dynamicOffsets?: number[]): void
  draw(vertexCount: number, instanceCount?: number, firstVertex?: number): void
  drawIndexed(indexCount: number, instanceCount?: number, firstIndex?: number): void
  end(): void
}

interface GALBuffer {
  byteLength: number
  destroy(): void
}

interface GALTexture {
  width: number
  height: number
  format: TextureFormat
  createView(desc?: TextureViewDesc): GALTextureView
  destroy(): void
}

interface GALPipeline {
  // Opaque handle — internal structure differs per backend
}

interface GALBindGroup {
  // Opaque handle
}
```

### WebGPU Backend

The WebGPU backend is a thin wrapper — each GAL method maps nearly 1:1 to a
WebGPU API call:

- `GALDevice` wraps `GPUDevice`
- `GALBuffer` wraps `GPUBuffer`
- `GALTexture` wraps `GPUTexture`
- `GALPipeline` wraps `GPURenderPipeline`
- `GALBindGroup` wraps `GPUBindGroup`
- `GALRenderPass` wraps `GPURenderPassEncoder`
- `submitCommands()` calls `device.queue.submit([commandEncoder.finish()])`

**Render Bundles (WebGPU-only)**

For static geometry that does not change between frames, WebGPU render bundles
pre-record draw commands on the GPU. The renderer detects objects marked as
`static: true` and records their draw commands into a `GPURenderBundle` once.
Subsequent frames replay the bundle at near-zero JS cost.

```typescript
// Pseudo-code for bundle creation
const bundleEncoder = device.createRenderBundleEncoder({ colorFormats, depthStencilFormat })
for (const cmd of staticCommands) {
  bundleEncoder.setPipeline(cmd.pipeline)
  bundleEncoder.setVertexBuffer(0, cmd.vertexBuffer)
  bundleEncoder.setIndexBuffer(cmd.indexBuffer, 'uint16')
  bundleEncoder.setBindGroup(0, cmd.frameBindGroup)
  bundleEncoder.setBindGroup(1, cmd.materialBindGroup)
  bundleEncoder.setBindGroup(2, cmd.objectBindGroup, [cmd.dynamicOffset])
  bundleEncoder.drawIndexed(cmd.indexCount)
}
const bundle = bundleEncoder.finish()

// Each frame — replay instead of re-issuing
renderPass.executeBundles([bundle])
```

On WebGL2, the `static` flag is ignored — objects are drawn normally. There is
no WebGL2 equivalent of render bundles.

### WebGL2 Backend

The WebGL2 backend maps WebGPU-style concepts onto WebGL2's state machine with
caching to minimize redundant state changes:

| GAL Concept | WebGL2 Mapping |
|---|---|
| `Pipeline` | `WebGLProgram` + blend/depth/cull state snapshot |
| `Buffer` | `WebGLBuffer` (ARRAY_BUFFER or ELEMENT_ARRAY_BUFFER or UNIFORM_BUFFER) |
| `Texture` | `WebGLTexture` + sampler parameters |
| `BindGroup` | A recorded set of uniform buffer bindings + texture unit assignments |
| `RenderPass` | Framebuffer bind + viewport + clear |
| `ShaderModule` | Compiled `WebGLShader` (vertex or fragment) |

**State Cache**

WebGL2 is a state machine — setting the same state twice is a no-op in the
driver but still has JS overhead. The WebGL2 backend maintains a shadow state
cache:

```typescript
// Only call gl bindVertexArray if it actually changed
const setVAO = (vao: WebGLVertexArrayObject | null) => {
  if (vao !== currentVAO) {
    gl.bindVertexArray(vao)
    currentVAO = vao
  }
}
```

Cached state includes: current program, VAO, framebuffer, blend state,
depth state, cull state, active texture units, bound UBO slots.

**UBO Emulation**

WebGL2 supports Uniform Buffer Objects natively. Bind groups map to UBO bind
points:

- Bind group 0 → UBO bind point 0 (per-frame data: camera, lights)
- Bind group 1 → UBO bind point 1 (per-material data: colors, flags)
- Bind group 2 → UBO bind point 2 (per-object data: model matrix) with dynamic
  offset via `bindBufferRange`

```typescript
// WebGL2 dynamic offset binding
gl.bindBufferRange(
  gl.UNIFORM_BUFFER,
  2,                    // bind point
  ringBuffer.glBuffer,
  dynamicOffset,        // byte offset into ring buffer
  objectUniformSize     // 256 bytes per object
)
```

**VAO Management**

Each unique (geometry, pipeline) pair gets a pre-built VAO. VAOs are created
lazily on first use and cached by a composite key. This eliminates per-draw
vertex attribute setup.

---

## Frame Pipeline

The renderer orchestrates each frame through a fixed sequence of render passes:

```
┌─────────────────────────────────────────────────────────────────┐
│ Pass 1: Shadow Maps (depth-only, 3 cascades)                    │
│   Target: 3072×1024 depth atlas                                  │
│   Objects: opaque only, sorted front-to-back per cascade         │
├─────────────────────────────────────────────────────────────────┤
│ Pass 2: Opaque Geometry                                          │
│   Targets: color (RGBA8) + emissive (RGBA8) + depth (D24)       │
│   MSAA: 4× multisample                                           │
│   Objects: sorted by draw key (pipeline → material → depth)      │
├─────────────────────────────────────────────────────────────────┤
│ Pass 3: Transparent Geometry (OIT)                               │
│   Targets: accumulation (RGBA16F) + revealage (R8)               │
│   Blend: additive (accum) + multiplicative (revealage)           │
│   Objects: no sorting required                                    │
├─────────────────────────────────────────────────────────────────┤
│ Pass 4: MSAA Resolve                                             │
│   Resolve multisample color + emissive to single-sample textures │
├─────────────────────────────────────────────────────────────────┤
│ Pass 5: OIT Composite                                            │
│   Full-screen pass: blend OIT result over resolved opaque color  │
├─────────────────────────────────────────────────────────────────┤
│ Pass 6: Bloom                                                    │
│   Threshold emissive → 5× downsample → 5× upsample → composite  │
├─────────────────────────────────────────────────────────────────┤
│ Pass 7: Final Blit                                               │
│   Copy result to swap chain / canvas                              │
└─────────────────────────────────────────────────────────────────┘
```

### Render Targets

| Target | Format | Size | MSAA | Purpose |
|---|---|---|---|---|
| Color | RGBA8Unorm | Canvas size | 4× | Main scene color |
| Emissive | RGBA8Unorm | Canvas size | 4× | Emissive for bloom extraction |
| Depth | Depth24Plus | Canvas size | 4× | Depth buffer |
| Shadow atlas | Depth24Plus | 3072×1024 | 1× | 3 cascade shadow maps |
| OIT Accumulation | RGBA16Float | Canvas size | 1× | Weighted color accumulation |
| OIT Revealage | R8Unorm | Canvas size | 1× | Alpha product |
| Bloom chain | RGBA8Unorm | Half-res × 5 | 1× | Downsample/upsample chain |
| Resolved color | RGBA8Unorm | Canvas size | 1× | Post-MSAA resolve |
| Resolved emissive | RGBA8Unorm | Canvas size | 1× | Post-MSAA resolve |

### Bind Group Layout

Three bind groups partition uniform data by update frequency:

**Group 0 — Per-Frame (updated once per frame)**
```
binding 0: CameraUniforms    { viewMatrix, projMatrix, vpMatrix, cameraPos, near, far }
binding 1: LightUniforms     { ambientColor, dirLightDir, dirLightColor, cascadeSplits, cascadeVP[3] }
binding 2: shadow map texture + sampler
```

**Group 1 — Per-Material (updated when material changes)**
```
binding 0: MaterialUniforms  { baseColor, materialColors[32], emissiveColors[32], flags }
binding 1: color texture + sampler
binding 2: ao texture + sampler
```

**Group 2 — Per-Object (dynamic offset into ring buffer)**
```
binding 0: ObjectUniforms    { modelMatrix, normalMatrix, jointTexture? }
```

The per-object bind group uses **dynamic offsets** rather than separate bind
groups per object. A single bind group is created once; each draw call provides
a different byte offset into the ring buffer. This is the primary mechanism for
achieving high draw call counts:

- WebGPU: `setBindGroup(2, objectBindGroup, [offset])` — native dynamic offset
- WebGL2: `bindBufferRange(UNIFORM_BUFFER, 2, buffer, offset, size)` — maps directly

### Command Buffer Pattern

Instead of issuing draw calls directly, the renderer builds a flat array of
draw commands during the sort phase:

```typescript
interface DrawCommand {
  sortKey: bigint          // 64-bit sort key
  pipelineId: number       // index into pipeline cache
  vertexBufferId: number   // index into buffer registry
  indexBufferId: number    // index into buffer registry
  materialBindGroupId: number
  ringBufferOffset: number // dynamic offset for per-object UBO
  indexCount: number
  indexOffset: number
  vertexOffset: number
}
```

Commands are stored in a pre-allocated array (no per-frame allocation). After
radix sorting by `sortKey`, the renderer executes them linearly, skipping
redundant state changes:

```typescript
const executeCommands = (pass: GALRenderPass, cmds: DrawCommand[]) => {
  let prevPipeline = -1
  let prevMaterial = -1
  let prevVertex = -1

  for (let i = 0; i < cmdCount; i++) {
    const cmd = cmds[i]

    if (cmd.pipelineId !== prevPipeline) {
      pass.setPipeline(pipelines[cmd.pipelineId])
      prevPipeline = cmd.pipelineId
    }
    if (cmd.vertexBufferId !== prevVertex) {
      pass.setVertexBuffer(0, buffers[cmd.vertexBufferId])
      pass.setIndexBuffer(indexBuffers[cmd.indexBufferId], 'uint16')
      prevVertex = cmd.vertexBufferId
    }
    if (cmd.materialBindGroupId !== prevMaterial) {
      pass.setBindGroup(1, materialBindGroups[cmd.materialBindGroupId])
      prevMaterial = cmd.materialBindGroupId
    }

    // Per-object bind group with dynamic offset — always set (offset changes)
    pass.setBindGroup(2, objectBindGroup, [cmd.ringBufferOffset])
    pass.drawIndexed(cmd.indexCount, 1, cmd.indexOffset, cmd.vertexOffset)
  }
}
```

This pattern ensures:
- Pipeline switches are minimized (sorted first in the key)
- Material/texture switches are minimized (sorted second)
- Only the dynamic offset changes per draw call in the best case
- Zero allocation during execution

### Shader Management

Shaders are written in WGSL (WebGPU) and GLSL 300 ES (WebGL2). They share
the same logical structure and are generated from a common template with
feature flags:

```typescript
type ShaderFeatures = {
  hasVertexColors: boolean
  hasMaterialIndex: boolean
  hasColorTexture: boolean
  hasAOTexture: boolean
  hasSkinning: boolean
  hasShadows: boolean
  isLambert: boolean  // false = basic/unlit
}
```

Each unique combination of features compiles to a separate shader variant. With
2 material types × 2^6 feature flags, the theoretical maximum is 128 variants,
but in practice only 10–20 are used. Variants are compiled lazily on first use
and cached.

**WGSL Template (WebGPU)**
```wgsl
// Vertex shader
@group(0) @binding(0) var<uniform> camera: CameraUniforms;
@group(2) @binding(0) var<uniform> object: ObjectUniforms;

struct VertexInput {
  @location(0) position: vec3f,
  @location(1) normal: vec3f,
#if HAS_VERTEX_COLORS
  @location(2) color: vec4f,
#endif
#if HAS_MATERIAL_INDEX
  @location(3) materialIndex: u32,
#endif
  // ...
};
```

**GLSL 300 ES Template (WebGL2)**
```glsl
#version 300 es
precision highp float;

layout(std140) uniform CameraUniforms { /* ... */ };
layout(std140) uniform ObjectUniforms { /* ... */ };

layout(location = 0) in vec3 a_position;
layout(location = 1) in vec3 a_normal;
#ifdef HAS_VERTEX_COLORS
layout(location = 2) in vec4 a_color;
#endif
#ifdef HAS_MATERIAL_INDEX
layout(location = 3) in uint a_materialIndex;
#endif
```

The shader compiler preprocesses `#if` / `#ifdef` directives at compile time,
producing clean shader source with no runtime branching.

### Resize Handling

On canvas resize, all resolution-dependent render targets are recreated. The
frame pipeline detects size changes at the start of each frame:

```typescript
const checkResize = () => {
  const w = canvas.clientWidth * devicePixelRatio | 0
  const h = canvas.clientHeight * devicePixelRatio | 0
  if (w !== currentWidth || h !== currentHeight) {
    recreateRenderTargets(w, h)
    camera.aspect = w / h
    camera.updateProjection()
    engine.emit('resize', { width: w, height: h })
  }
}
```
