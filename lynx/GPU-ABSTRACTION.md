# GPU Abstraction Layer (GAL)

The GPU Abstraction Layer is the foundation of Lynx's rendering stack. It provides a single API that targets both WebGPU and WebGL2, allowing the rest of the engine to be completely backend-agnostic. Every module above the GAL in the architecture diagram -- the renderer, materials, shadow passes, OIT, bloom -- talks exclusively to GAL interfaces. No `GPUDevice` or `WebGL2RenderingContext` appears anywhere outside `src/gal/`.

## Design Philosophy

### Modeled After WebGPU

The GAL's API surface mirrors WebGPU because WebGPU is the better API. It is explicit, stateless (no hidden global state machine), and maps cleanly to modern GPU hardware. The WebGL2 backend is the translation layer, not the other way around.

This means:

- **Pipelines are immutable objects.** You create a `GalRenderPipeline` with all state baked in (shaders, blend, depth, vertex layout, topology, MSAA). In WebGPU this maps directly to `GPURenderPipeline`. In WebGL2 this is a cached bundle of GL state that gets applied as a unit.
- **Bind groups are explicit.** You declare what resources (UBOs, textures, samplers) a draw call needs in a `GalBindGroup`. In WebGPU this is a `GPUBindGroup`. In WebGL2 this is a list of UBO binding points and texture unit assignments that get applied together.
- **Command recording is explicit.** You open a `GalCommandEncoder`, begin render passes, record draw calls, end passes, and submit. In WebGPU this records to a command buffer. In WebGL2 the commands execute immediately (there is no command buffer), but the same API shape is maintained so the engine code above does not care.

### Zero-Overhead Abstraction

The abstraction adds no measurable cost over calling the native API directly. This is achieved through:

1. **No virtual dispatch in hot paths.** Backend selection happens once at initialization. The returned objects are concrete implementations, not interface-based polymorphic types. TypeScript's structural typing gives us the interface contract at compile time. At runtime, calling `device.createBuffer()` on a `WebGPUDevice` calls the WebGPU implementation directly -- no vtable lookup, no indirection.

2. **No wrapper allocations per frame.** GAL objects are created at load time and reused. The per-frame hot path (recording draw calls) allocates nothing.

3. **Backend-specific fast paths.** The WebGPU backend uses render bundles for static geometry. The WebGL2 backend uses redundant-state-skip tracking. These are not abstracted into a common "render bundle" concept because WebGL2 cannot do true render bundles. Instead, each backend optimizes its own hot path independently.

### Trade-offs

| Trade-off | Decision | Rationale |
|---|---|---|
| API shape follows WebGPU, not WebGL2 | WebGL2 backend does extra translation work | WebGPU is the future; WebGL2 is the fallback. Translation cost is acceptable because the alternative (lowest-common-denominator API) would cripple WebGPU performance. |
| No virtual dispatch | Two separate code paths for backends | Eliminates per-draw-call overhead. The engine is small enough that maintaining two backends is manageable. |
| Immutable pipelines | Pipeline creation is expensive (especially WebGL2 which must compile shaders lazily) | Immutability enables caching, hashing, and render bundle recording. Creation cost is amortized over thousands of frames. |
| Dual shader shipping | Larger asset payload | Attempting runtime WGSL-to-GLSL transpilation for all shaders is fragile. Simple shaders can use a minimal transpiler; complex shaders ship both variants. |

---

## Core Interfaces

### Type Foundations

```typescript
type GalBackend = 'webgpu' | 'webgl2'

type GalBufferUsage = 'vertex' | 'index' | 'uniform' | 'storage'

type GalTextureDimension = '2d' | '2d-array' | 'cube'

type GalTextureFormat =
  | 'rgba8unorm'
  | 'rgba8unorm-srgb'
  | 'bgra8unorm'
  | 'bgra8unorm-srgb'
  | 'rgba16float'
  | 'rgba32float'
  | 'depth24plus'
  | 'depth24plus-stencil8'
  | 'depth32float'
  | 'r8unorm'
  | 'rg8unorm'
  | 'r16float'
  | 'rg16float'
  | 'bc7-rgba-unorm'      // Desktop compressed
  | 'etc2-rgba8unorm'     // Mobile compressed
  | 'astc-4x4-unorm'      // High-end mobile compressed

type GalPrimitiveTopology = 'triangle-list' | 'triangle-strip' | 'line-list' | 'line-strip' | 'point-list'

type GalVertexFormat =
  | 'float32' | 'float32x2' | 'float32x3' | 'float32x4'
  | 'uint8x4' | 'unorm8x4'
  | 'uint16x2' | 'uint16x4'
  | 'sint16x2' | 'sint16x4'
  | 'unorm16x2' | 'unorm16x4'

type GalCompareFunction = 'never' | 'less' | 'equal' | 'less-equal' | 'greater' | 'not-equal' | 'greater-equal' | 'always'

type GalBlendFactor =
  | 'zero' | 'one'
  | 'src' | 'one-minus-src'
  | 'dst' | 'one-minus-dst'
  | 'src-alpha' | 'one-minus-src-alpha'
  | 'dst-alpha' | 'one-minus-dst-alpha'

type GalBlendOperation = 'add' | 'subtract' | 'reverse-subtract' | 'min' | 'max'

type GalFilterMode = 'nearest' | 'linear'
type GalMipmapFilterMode = 'nearest' | 'linear'
type GalAddressMode = 'clamp-to-edge' | 'repeat' | 'mirror-repeat'

type GalShaderStage = 'vertex' | 'fragment'

type GalLoadOp = 'clear' | 'load'
type GalStoreOp = 'store' | 'discard'
```

---

### GalDevice

The device is the root of the GAL. It wraps the platform-specific GPU context and serves as the factory for all other GAL objects. Nothing GPU-related can be created without a device.

```typescript
interface GalDeviceDescriptor {
  readonly canvas: HTMLCanvasElement
  readonly preferredBackend?: GalBackend          // Default: auto-detect
  readonly powerPreference?: 'low-power' | 'high-performance'
  readonly antialias?: boolean                     // Request MSAA (default true)
  readonly sampleCount?: number                    // 1 or 4 (default 4)
}

interface GalDeviceLimits {
  readonly maxTextureSize: number                  // e.g. 8192 or 16384
  readonly maxTextureLayers: number                // For 2d-array textures
  readonly maxUniformBufferSize: number            // Bytes
  readonly maxStorageBufferSize: number            // 0 for WebGL2
  readonly maxBindGroups: number                   // WebGPU: 4, WebGL2: emulated
  readonly maxSamplersPerShaderStage: number
  readonly maxTexturesPerShaderStage: number
  readonly supportsStorageBuffers: boolean         // WebGPU: true, WebGL2: false
  readonly supportsIndirectDraw: boolean           // WebGPU: true, WebGL2: false
  readonly supportsRenderBundles: boolean          // WebGPU: true, WebGL2: false
  readonly compressedTextureFormats: readonly GalTextureFormat[]
}

interface GalDevice {
  readonly backend: GalBackend
  readonly limits: GalDeviceLimits

  createBuffer: (desc: GalBufferDescriptor) => GalBuffer
  createTexture: (desc: GalTextureDescriptor) => GalTexture
  createSampler: (desc: GalSamplerDescriptor) => GalSampler
  createShaderModule: (desc: GalShaderModuleDescriptor) => GalShaderModule
  createRenderPipeline: (desc: GalRenderPipelineDescriptor) => GalRenderPipeline
  createBindGroup: (desc: GalBindGroupDescriptor) => GalBindGroup
  createCommandEncoder: () => GalCommandEncoder

  /** Write data into an existing buffer (maps to queue.writeBuffer or gl.bufferSubData) */
  writeBuffer: (buffer: GalBuffer, data: ArrayBufferView, offset?: number) => void

  /** Write pixel data into a texture (maps to queue.writeTexture or gl.texSubImage2D) */
  writeTexture: (texture: GalTexture, data: ArrayBufferView, layout: GalTextureWriteLayout) => void

  /** Submit recorded commands (maps to queue.submit or no-op on WebGL2) */
  submit: (commands: GalCommandBuffer[]) => void

  /** Get the preferred texture format for the canvas surface */
  getPreferredFormat: () => GalTextureFormat

  /** Get the current canvas backbuffer texture for render pass output */
  getCurrentTexture: () => GalTexture

  destroy: () => void
}
```

**WebGPU mapping:** `GalDevice` wraps a `GPUDevice` + `GPUCanvasContext`. `createBuffer` calls `device.createBuffer()`. `submit` calls `device.queue.submit()`. `getCurrentTexture` calls `context.getCurrentTexture()`.

**WebGL2 mapping:** `GalDevice` wraps a `WebGL2RenderingContext`. `createBuffer` calls `gl.createBuffer()` + `gl.bufferData()`. `submit` is a no-op because WebGL2 commands execute immediately during encoding. `getCurrentTexture` returns a sentinel object representing the default framebuffer.

---

### GalBuffer

```typescript
interface GalBufferDescriptor {
  readonly label?: string
  readonly size: number                            // Bytes
  readonly usage: GalBufferUsage | GalBufferUsage[]
  readonly dynamic?: boolean                       // Hint for frequent CPU writes (default false)
}

interface GalBuffer {
  readonly label: string
  readonly size: number
  readonly usage: GalBufferUsage[]
  readonly dynamic: boolean

  destroy: () => void

  /**
   * Internal handle — not part of public API.
   * WebGPU: GPUBuffer
   * WebGL2: WebGLBuffer
   */
  readonly _native: GPUBuffer | WebGLBuffer
}
```

**WebGPU mapping:** Maps directly to `GPUBuffer`. The `usage` field is translated to `GPUBufferUsageFlags` (e.g., `GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST`). Dynamic buffers use `GPUBufferUsage.COPY_DST` so they can be updated via `queue.writeBuffer()`.

**WebGL2 mapping:** Maps to a `WebGLBuffer`. The `usage` determines the GL binding target (`ARRAY_BUFFER` for vertex, `ELEMENT_ARRAY_BUFFER` for index, `UNIFORM_BUFFER` for uniform). The `dynamic` flag maps to `gl.DYNAMIC_DRAW` vs `gl.STATIC_DRAW` usage hints.

**Key detail:** WebGL2 buffers have a single binding target. If a buffer is used as both vertex and index data (uncommon but possible), the WebGL2 backend must re-bind it to different targets. In practice, vertex and index buffers are separate, so this is a non-issue.

---

### GalTexture

```typescript
interface GalTextureDescriptor {
  readonly label?: string
  readonly dimension: GalTextureDimension
  readonly format: GalTextureFormat
  readonly width: number
  readonly height: number
  readonly depthOrArrayLayers?: number             // Default 1. >1 for 2d-array (CSM) or 6 for cube
  readonly mipLevelCount?: number                  // Default 1 (no mips)
  readonly sampleCount?: number                    // 1 or 4 (MSAA render targets)
  readonly usage: GalTextureUsage[]
}

type GalTextureUsage =
  | 'texture-binding'      // Can be sampled in shaders
  | 'storage-binding'      // Can be written in compute (WebGPU only)
  | 'render-attachment'    // Can be used as a render target
  | 'copy-src'             // Can be source of a copy
  | 'copy-dst'             // Can be destination of a copy / write

interface GalTexture {
  readonly label: string
  readonly dimension: GalTextureDimension
  readonly format: GalTextureFormat
  readonly width: number
  readonly height: number
  readonly depthOrArrayLayers: number
  readonly mipLevelCount: number

  /** Create a view of a specific mip level or array layer */
  createView: (desc?: GalTextureViewDescriptor) => GalTextureView

  destroy: () => void
}

interface GalTextureViewDescriptor {
  readonly dimension?: GalTextureDimension | 'cube'
  readonly baseArrayLayer?: number
  readonly arrayLayerCount?: number
  readonly baseMipLevel?: number
  readonly mipLevelCount?: number
}

interface GalTextureView {
  readonly texture: GalTexture
  readonly _native: GPUTextureView | WebGLTexture // WebGL2: same texture, layer info stored separately
  readonly _layer?: number                         // WebGL2: layer index for framebuffer attachment
  readonly _mipLevel?: number                      // WebGL2: mip level for framebuffer attachment
}
```

**WebGPU mapping:** Direct mapping to `GPUTexture`. Dimension `'2d-array'` maps to `GPUTextureDimension` `'2d'` with `depthOrArrayLayers > 1`. Cube maps use `depthOrArrayLayers = 6` and the view dimension `'cube'`. `createView` maps to `texture.createView()`.

**WebGL2 mapping:** Dimension `'2d'` creates a `gl.TEXTURE_2D`. Dimension `'2d-array'` creates a `gl.TEXTURE_2D_ARRAY` (used for cascaded shadow maps -- each cascade is one layer). Dimension `'cube'` creates a `gl.TEXTURE_CUBE_MAP`. Views are simulated: WebGL2 does not have texture views. The `GalTextureView` object stores the base texture plus metadata (layer, mip level) that the render pass uses when attaching to a framebuffer via `gl.framebufferTextureLayer()`.

**CSM-specific note:** Shadow cascades use a `2d-array` texture with `depthOrArrayLayers = 3` (for 3 cascades). Each cascade renders to a different layer. In WebGPU, each cascade pass targets a `GPUTextureView` of a specific array layer. In WebGL2, each cascade pass calls `gl.framebufferTextureLayer(gl.FRAMEBUFFER, gl.DEPTH_ATTACHMENT, texture, 0, layerIndex)`.

---

### GalSampler

```typescript
interface GalSamplerDescriptor {
  readonly label?: string
  readonly magFilter?: GalFilterMode               // Default 'linear'
  readonly minFilter?: GalFilterMode               // Default 'linear'
  readonly mipmapFilter?: GalMipmapFilterMode      // Default 'linear'
  readonly addressModeU?: GalAddressMode           // Default 'clamp-to-edge'
  readonly addressModeV?: GalAddressMode           // Default 'clamp-to-edge'
  readonly addressModeW?: GalAddressMode           // Default 'clamp-to-edge'
  readonly compare?: GalCompareFunction            // For shadow map sampling
  readonly maxAnisotropy?: number                  // Default 1
}

interface GalSampler {
  readonly _native: GPUSampler | WebGLSampler
}
```

**WebGPU mapping:** Direct mapping to `GPUSampler` via `device.createSampler()`.

**WebGL2 mapping:** Maps to `WebGLSampler` created via `gl.createSampler()`. Parameters set via `gl.samplerParameteri()` / `gl.samplerParameterf()`. The `compare` field maps to `gl.TEXTURE_COMPARE_MODE = gl.COMPARE_REF_TO_TEXTURE` and `gl.TEXTURE_COMPARE_FUNC` for shadow map sampling with `texture()` in GLSL.

**Shadow sampler note:** Both backends support hardware shadow map comparison (depth comparison during sampling). The shadow pass uses `compare: 'less-equal'` on the sampler, which enables single-tap hardware PCF on mobile GPUs that support it, with the 5x5 PCF kernel implemented in the shader on top of this.

---

### GalShaderModule

```typescript
interface GalShaderModuleDescriptor {
  readonly label?: string
  readonly code: string                            // WGSL (WebGPU) or GLSL 300 es (WebGL2)
  readonly stage: GalShaderStage
  readonly defines?: Record<string, string | number | boolean>
}

interface GalShaderModule {
  readonly label: string
  readonly stage: GalShaderStage

  /** WebGPU: GPUShaderModule. WebGL2: compiled WebGLShader */
  readonly _native: GPUShaderModule | WebGLShader
}
```

**WebGPU mapping:** `code` is WGSL. Passed to `device.createShaderModule({ code })`. The `defines` are injected as constants at the top of the WGSL source (WGSL supports `override` declarations for pipeline-overridable constants, but for broader compatibility, defines are preprocessed as string replacements before compilation).

**WebGL2 mapping:** `code` is GLSL 300 es. The `defines` are injected as `#define` lines after `#version 300 es`. The shader is compiled via `gl.createShader()` + `gl.shaderSource()` + `gl.compileShader()`. Compilation errors are caught and reported with the shader label.

**Dual shader strategy:** The engine does not transpile WGSL to GLSL at runtime for complex shaders. Instead, both WGSL and GLSL variants are generated at build time from a shared shader template (see [Shader Strategy](#shader-strategy) below). The `GalShaderModule` constructor receives the correct variant based on the active backend.

---

### GalRenderPipeline

```typescript
interface GalVertexAttribute {
  readonly shaderLocation: number                  // Attribute index in shader
  readonly format: GalVertexFormat
  readonly offset: number                          // Byte offset within buffer stride
}

interface GalVertexBufferLayout {
  readonly arrayStride: number                     // Bytes per vertex
  readonly stepMode?: 'vertex' | 'instance'        // Default 'vertex'
  readonly attributes: readonly GalVertexAttribute[]
}

interface GalBlendState {
  readonly color: {
    readonly srcFactor: GalBlendFactor
    readonly dstFactor: GalBlendFactor
    readonly operation: GalBlendOperation
  }
  readonly alpha: {
    readonly srcFactor: GalBlendFactor
    readonly dstFactor: GalBlendFactor
    readonly operation: GalBlendOperation
  }
}

interface GalDepthStencilState {
  readonly format: GalTextureFormat
  readonly depthWriteEnabled: boolean
  readonly depthCompare: GalCompareFunction
}

interface GalColorTargetState {
  readonly format: GalTextureFormat
  readonly blend?: GalBlendState
  readonly writeMask?: number                      // Bitmask: R=1, G=2, B=4, A=8 (default 0xF)
}

interface GalRenderPipelineDescriptor {
  readonly label?: string
  readonly vertex: {
    readonly module: GalShaderModule
    readonly entryPoint: string
    readonly buffers: readonly GalVertexBufferLayout[]
  }
  readonly fragment: {
    readonly module: GalShaderModule
    readonly entryPoint: string
    readonly targets: readonly GalColorTargetState[]
  }
  readonly depthStencil?: GalDepthStencilState
  readonly primitive?: {
    readonly topology?: GalPrimitiveTopology       // Default 'triangle-list'
    readonly cullMode?: 'none' | 'front' | 'back'  // Default 'back'
    readonly frontFace?: 'ccw' | 'cw'              // Default 'ccw'
  }
  readonly multisample?: {
    readonly count?: number                        // 1 or 4 (default 1)
  }
}

interface GalRenderPipeline {
  readonly label: string

  /**
   * WebGPU: GPURenderPipeline
   * WebGL2: a compiled+linked WebGLProgram plus cached GL state
   */
  readonly _native: GPURenderPipeline | WebGLProgram
  readonly _glState?: WebGL2PipelineState          // WebGL2 only: cached state to apply
}
```

**WebGPU mapping:** Maps directly to `device.createRenderPipeline()`. The descriptor translates 1:1 with minor name adjustments. Pipelines are created asynchronously where possible via `createRenderPipelineAsync()` to avoid janking the main thread during shader compilation.

**WebGL2 mapping:** This is where the GAL does the most translation work. A `GalRenderPipeline` in WebGL2 consists of:

1. A **linked `WebGLProgram`** (vertex shader + fragment shader compiled and linked together).
2. A **`WebGL2PipelineState` snapshot** containing all the GL state this pipeline requires:
   - Blend enable/disable, blend func, blend equation
   - Depth test enable/disable, depth func, depth mask
   - Cull face enable/disable, cull face mode, front face
   - Color write mask

When a pipeline is bound during a render pass, the WebGL2 backend compares this snapshot against the currently tracked GL state and issues only the `gl.*` calls for state that actually changed (see [WebGL2 State Tracking](#webgl2-state-tracking)).

```typescript
/** Internal WebGL2 pipeline state cache */
interface WebGL2PipelineState {
  readonly program: WebGLProgram
  readonly blendEnabled: boolean
  readonly blendSrcRGB: number
  readonly blendDstRGB: number
  readonly blendSrcAlpha: number
  readonly blendDstAlpha: number
  readonly blendEqRGB: number
  readonly blendEqAlpha: number
  readonly depthTestEnabled: boolean
  readonly depthFunc: number
  readonly depthMask: boolean
  readonly cullEnabled: boolean
  readonly cullFace: number
  readonly frontFace: number
  readonly colorMask: [boolean, boolean, boolean, boolean]
}
```

**Immutability guarantee:** Once created, a `GalRenderPipeline` cannot be modified. This enables the WebGL2 backend to pre-compute the full GL state snapshot at creation time. It also enables render bundle recording in WebGPU -- a bundle references specific pipelines and records their draw calls for replay.

---

### GalBindGroup

```typescript
type GalBindingResource =
  | { type: 'buffer', buffer: GalBuffer, offset?: number, size?: number }
  | { type: 'texture', textureView: GalTextureView }
  | { type: 'sampler', sampler: GalSampler }

interface GalBindGroupDescriptor {
  readonly label?: string
  readonly layout: GalBindGroupLayout
  readonly entries: readonly {
    readonly binding: number
    readonly resource: GalBindingResource
  }[]
}

interface GalBindGroupLayoutEntry {
  readonly binding: number
  readonly visibility: GalShaderStage[]
  readonly type: 'uniform-buffer' | 'storage-buffer' | 'texture' | 'sampler' | 'comparison-sampler'
}

interface GalBindGroupLayout {
  readonly entries: readonly GalBindGroupLayoutEntry[]
}

interface GalBindGroup {
  readonly label: string

  /**
   * WebGPU: GPUBindGroup
   * WebGL2: array of binding commands to execute
   */
  readonly _native: GPUBindGroup | WebGL2BindGroupState
}
```

**WebGPU mapping:** Bind groups map directly to `GPUBindGroup`. The layout maps to `GPUBindGroupLayout`. WebGPU validates at creation time that the resources match the layout. Binding is a single call: `renderPass.setBindGroup(index, bindGroup)`.

**WebGL2 mapping:** WebGL2 has no bind group concept. The `WebGL2BindGroupState` is an array of commands:

```typescript
interface WebGL2BindGroupState {
  /** UBO bindings: [bindingPoint, buffer, offset, size][] */
  readonly uniformBuffers: readonly [number, WebGLBuffer, number, number][]
  /** Texture bindings: [textureUnit, target, texture, sampler][] */
  readonly textures: readonly [number, number, WebGLTexture, WebGLSampler][]
}
```

When `setBindGroup` is called in the WebGL2 render pass, the backend iterates the binding commands:
- For each UBO: `gl.bindBufferRange(gl.UNIFORM_BUFFER, bindingPoint, buffer, offset, size)`
- For each texture: `gl.activeTexture(gl.TEXTURE0 + unit)`, `gl.bindTexture(target, texture)`, `gl.bindSampler(unit, sampler)`

The binding point numbers and texture unit numbers are derived from the bind group index and binding index. The engine uses a simple mapping: bind group 0 uses UBO binding points 0-3 and texture units 0-7, bind group 1 uses UBO binding points 4-7 and texture units 8-15, etc.

**Bind group convention used by Lynx:**

| Group | Purpose | Typical contents |
|---|---|---|
| 0 | Per-frame data | Camera matrices (view, projection, viewProjection), time, screen size |
| 1 | Per-material data | Material UBO (color, roughness, metalness, etc.), albedo texture, normal map, sampler |
| 2 | Per-object data | Model matrix, normal matrix (via dynamic UBO offset) |
| 3 | Shadow / lighting | CSM shadow map array texture, shadow sampler, cascade split distances, light direction UBO |

---

### GalRenderPass

```typescript
interface GalColorAttachment {
  readonly view: GalTextureView
  readonly resolveTarget?: GalTextureView          // MSAA resolve target
  readonly loadOp: GalLoadOp
  readonly storeOp: GalStoreOp
  readonly clearValue?: [number, number, number, number]  // RGBA, used if loadOp is 'clear'
}

interface GalDepthStencilAttachment {
  readonly view: GalTextureView
  readonly depthLoadOp: GalLoadOp
  readonly depthStoreOp: GalStoreOp
  readonly depthClearValue?: number                // Default 1.0
}

interface GalRenderPassDescriptor {
  readonly label?: string
  readonly colorAttachments: readonly GalColorAttachment[]
  readonly depthStencilAttachment?: GalDepthStencilAttachment
}

interface GalRenderPass {
  setPipeline: (pipeline: GalRenderPipeline) => void
  setBindGroup: (index: number, bindGroup: GalBindGroup, dynamicOffsets?: readonly number[]) => void
  setVertexBuffer: (slot: number, buffer: GalBuffer, offset?: number) => void
  setIndexBuffer: (buffer: GalBuffer, format: 'uint16' | 'uint32', offset?: number) => void

  draw: (vertexCount: number, instanceCount?: number, firstVertex?: number, firstInstance?: number) => void
  drawIndexed: (indexCount: number, instanceCount?: number, firstIndex?: number, baseVertex?: number, firstInstance?: number) => void

  /** WebGPU only: execute a pre-recorded render bundle */
  executeBundles: (bundles: readonly GalRenderBundle[]) => void

  setViewport: (x: number, y: number, width: number, height: number, minDepth?: number, maxDepth?: number) => void
  setScissorRect: (x: number, y: number, width: number, height: number) => void

  end: () => void
}
```

**WebGPU mapping:** Wraps `GPURenderPassEncoder`. `beginRenderPass` on the command encoder calls `encoder.beginRenderPass()` with the translated descriptor. All methods map 1:1 to `GPURenderPassEncoder` methods.

**WebGL2 mapping:** This is the most complex translation in the GAL. On `beginRenderPass`:

1. **Framebuffer setup:** If the color attachment is the canvas backbuffer, bind the default framebuffer (`gl.bindFramebuffer(gl.FRAMEBUFFER, null)`). Otherwise, bind (or create and cache) a `WebGLFramebuffer` with the specified color and depth attachments.
2. **Clear:** If `loadOp` is `'clear'`, issue `gl.clearColor()` + `gl.clearDepth()` + `gl.clear()`.
3. **MSAA:** If `sampleCount > 1`, the color attachment is an MSAA renderbuffer. On `end()`, the backend resolves via `gl.blitFramebuffer()` from the MSAA framebuffer to the resolve target framebuffer.

During the pass, draw calls execute immediately:
- `setPipeline` applies the cached `WebGL2PipelineState` via state-tracked GL calls.
- `setBindGroup` iterates the `WebGL2BindGroupState` and binds UBOs and textures.
- `setVertexBuffer` calls `gl.bindVertexArray()` (VAOs are created and cached per vertex-buffer+index-buffer+pipeline combination).
- `draw` / `drawIndexed` calls `gl.drawArrays()` or `gl.drawElements()` or their instanced variants.

**VAO caching (WebGL2):** Vertex Array Objects capture the vertex attribute setup. The WebGL2 backend maintains a cache keyed by `[pipeline, vertexBuffers[], indexBuffer]`. On the first draw with a given combination, a VAO is created and configured. Subsequent draws with the same combination reuse the VAO, turning vertex setup into a single `gl.bindVertexArray()` call.

---

### GalCommandEncoder

```typescript
interface GalCommandBuffer {
  readonly _native: GPUCommandBuffer | null        // null for WebGL2 (commands already executed)
}

interface GalCommandEncoder {
  beginRenderPass: (desc: GalRenderPassDescriptor) => GalRenderPass
  finish: () => GalCommandBuffer
}
```

**WebGPU mapping:** Wraps `GPUCommandEncoder`. `beginRenderPass` maps to `encoder.beginRenderPass()`. `finish` maps to `encoder.finish()`, returning a `GPUCommandBuffer` that is submitted via `device.submit()`.

**WebGL2 mapping:** There is no command buffer in WebGL2. The encoder is a thin wrapper. `beginRenderPass` immediately sets up the framebuffer and returns a `GalRenderPass` that issues GL calls on each method. `finish` returns a sentinel `GalCommandBuffer` with `_native: null`. `device.submit()` is a no-op for WebGL2 since everything already executed.

This is a fundamental architectural difference between the two backends. WebGPU records commands into a buffer that is submitted atomically. WebGL2 executes each command as it is recorded. The GAL preserves WebGPU's record-then-submit API shape because:

1. It makes the engine code cleaner (explicit begin/end boundaries).
2. It enables future optimization (e.g., a WebGL2 backend could batch certain calls).
3. The overhead of the thin wrapper is negligible (one function call, no allocations).

---

### GalRenderBundle (WebGPU Only)

```typescript
interface GalRenderBundleEncoder {
  setPipeline: (pipeline: GalRenderPipeline) => void
  setBindGroup: (index: number, bindGroup: GalBindGroup, dynamicOffsets?: readonly number[]) => void
  setVertexBuffer: (slot: number, buffer: GalBuffer, offset?: number) => void
  setIndexBuffer: (buffer: GalBuffer, format: 'uint16' | 'uint32', offset?: number) => void
  draw: (vertexCount: number, instanceCount?: number, firstVertex?: number, firstInstance?: number) => void
  drawIndexed: (indexCount: number, instanceCount?: number, firstIndex?: number, baseVertex?: number, firstInstance?: number) => void
  finish: () => GalRenderBundle
}

interface GalRenderBundle {
  readonly _native: GPURenderBundle
}
```

This interface only exists on the WebGPU backend. See [Render Bundles](#render-bundles-webgpu-only) below.

---

## Backend Detection and Initialization

Backend selection happens once at startup. The engine attempts WebGPU first, falls back to WebGL2 if unavailable, and rejects if neither is available.

```typescript
const initGalDevice = async (desc: GalDeviceDescriptor): Promise<GalDevice> => {
  // 1. Try WebGPU if not explicitly requesting WebGL2
  if (desc.preferredBackend !== 'webgl2' && typeof navigator !== 'undefined' && navigator.gpu) {
    const adapter = await navigator.gpu.requestAdapter({
      powerPreference: desc.powerPreference ?? 'high-performance'
    })

    if (adapter) {
      const device = await adapter.requestDevice({
        requiredFeatures: getRequiredFeatures(adapter),
        requiredLimits: getRequiredLimits(adapter)
      })

      const context = desc.canvas.getContext('webgpu')
      if (context) {
        const format = navigator.gpu.getPreferredCanvasFormat()
        context.configure({
          device,
          format,
          alphaMode: 'premultiplied'
        })

        return createWebGPUDevice(device, context, format, desc)
      }
    }
  }

  // 2. Fall back to WebGL2
  const gl = desc.canvas.getContext('webgl2', {
    antialias: false,              // We handle MSAA ourselves
    alpha: false,                  // No compositing alpha
    depth: true,
    stencil: false,
    powerPreference: desc.powerPreference ?? 'high-performance',
    preserveDrawingBuffer: false   // Perf: allow buffer swap
  })

  if (!gl) {
    throw new Error('Lynx: Neither WebGPU nor WebGL2 is available on this device')
  }

  enableRequiredExtensions(gl)
  return createWebGL2Device(gl, desc)
}

/**
 * Request WebGPU features that the engine can take advantage of.
 * These are optional — the engine adapts if they are not available.
 */
const getRequiredFeatures = (adapter: GPUAdapter): GPUFeatureName[] => {
  const features: GPUFeatureName[] = []

  if (adapter.features.has('texture-compression-bc')) {
    features.push('texture-compression-bc')
  }
  if (adapter.features.has('texture-compression-etc2')) {
    features.push('texture-compression-etc2')
  }
  if (adapter.features.has('texture-compression-astc')) {
    features.push('texture-compression-astc')
  }
  if (adapter.features.has('depth-clip-control')) {
    features.push('depth-clip-control')
  }

  return features
}

/**
 * Request device limits that the engine needs.
 * Fall back gracefully if limits are below ideal.
 */
const getRequiredLimits = (adapter: GPUAdapter): Record<string, number> => ({
  maxBindGroups: Math.min(4, adapter.limits.maxBindGroups),
  maxUniformBufferBindingSize: Math.min(65536, adapter.limits.maxUniformBufferBindingSize),
  maxStorageBufferBindingSize: adapter.limits.maxStorageBufferBindingSize,
  maxTextureDimension2D: adapter.limits.maxTextureDimension2D
})

/**
 * Enable WebGL2 extensions the engine relies on.
 */
const enableRequiredExtensions = (gl: WebGL2RenderingContext): void => {
  // Compressed texture formats
  gl.getExtension('EXT_texture_compression_bptc')     // BC7
  gl.getExtension('WEBGL_compressed_texture_etc')      // ETC2
  gl.getExtension('WEBGL_compressed_texture_astc')     // ASTC

  // Anisotropic filtering
  gl.getExtension('EXT_texture_filter_anisotropic')

  // Color buffer float (for HDR render targets)
  gl.getExtension('EXT_color_buffer_float')

  // Parallel shader compilation
  gl.getExtension('KHR_parallel_shader_compile')

  // Timer queries (for performance profiling in dev mode)
  gl.getExtension('EXT_disjoint_timer_query_webgl2')
}
```

**Capability query:** After initialization, `device.limits` exposes what the hardware supports. The renderer uses this to adapt:

| Capability | Adaptation |
|---|---|
| `maxTextureSize < 4096` | Reduce shadow map resolution |
| `compressedTextureFormats` is empty | Fall back to RGBA8 (larger VRAM, slower loads) |
| `supportsStorageBuffers === false` (WebGL2) | Use uniform buffers with dynamic offsets for per-object data |
| `supportsRenderBundles === false` (WebGL2) | Use VAO cache path instead |

---

## Shader Strategy

### Dual Shader Pipeline

Lynx ships both WGSL and GLSL 300 es for every shader. The two variants are generated at build time from a common source-of-truth template.

```
shaders/
├── src/
│   ├── standard.shader.ts      # Shared shader template (TypeScript)
│   ├── shadow.shader.ts
│   ├── oit-accum.shader.ts
│   ├── bloom-downsample.shader.ts
│   ├── bloom-upsample.shader.ts
│   ├── tonemap.shader.ts
│   └── fullscreen-quad.shader.ts
├── generated/
│   ├── webgpu/
│   │   ├── standard.vert.wgsl
│   │   ├── standard.frag.wgsl
│   │   ├── shadow.vert.wgsl
│   │   └── ...
│   └── webgl2/
│       ├── standard.vert.glsl
│       ├── standard.frag.glsl
│       ├── shadow.vert.glsl
│       └── ...
└── build-shaders.ts             # Build script
```

Each `.shader.ts` file exports a function that returns shader source for a given backend:

```typescript
interface ShaderConfig {
  backend: GalBackend
  defines: Record<string, string | number | boolean>
}

// Example: standard.shader.ts
const standardVertexShader = (config: ShaderConfig): string => {
  if (config.backend === 'webgpu') {
    return /* wgsl */`
      struct FrameUniforms {
        viewProjection: mat4x4<f32>,
        cameraPosition: vec3<f32>,
        time: f32,
      }

      struct ObjectUniforms {
        model: mat4x4<f32>,
        normalMatrix: mat3x3<f32>,
      }

      @group(0) @binding(0) var<uniform> frame: FrameUniforms;
      @group(2) @binding(0) var<uniform> object: ObjectUniforms;

      struct VertexInput {
        @location(0) position: vec3<f32>,
        @location(1) normal: vec3<f32>,
        @location(2) uv: vec2<f32>,
${config.defines.SKINNING ? `
        @location(3) joints: vec4<u32>,
        @location(4) weights: vec4<f32>,
` : ''}
      }

      struct VertexOutput {
        @builtin(position) clipPosition: vec4<f32>,
        @location(0) worldPosition: vec3<f32>,
        @location(1) worldNormal: vec3<f32>,
        @location(2) uv: vec2<f32>,
      }

      @vertex
      fn main(input: VertexInput) -> VertexOutput {
        var out: VertexOutput;
        let worldPos = object.model * vec4<f32>(input.position, 1.0);
        out.clipPosition = frame.viewProjection * worldPos;
        out.worldPosition = worldPos.xyz;
        out.worldNormal = object.normalMatrix * input.normal;
        out.uv = input.uv;
        return out;
      }
    `
  }

  // WebGL2: GLSL 300 es
  return /* glsl */`#version 300 es
    precision highp float;

    layout(std140) uniform FrameUniforms {
      mat4 u_viewProjection;
      vec3 u_cameraPosition;
      float u_time;
    };

    layout(std140) uniform ObjectUniforms {
      mat4 u_model;
      mat3 u_normalMatrix;
    };

    layout(location = 0) in vec3 a_position;
    layout(location = 1) in vec3 a_normal;
    layout(location = 2) in vec2 a_uv;
${config.defines.SKINNING ? `
    layout(location = 3) in uvec4 a_joints;
    layout(location = 4) in vec4 a_weights;
` : ''}

    out vec3 v_worldPosition;
    out vec3 v_worldNormal;
    out vec2 v_uv;

    void main() {
      vec4 worldPos = u_model * vec4(a_position, 1.0);
      gl_Position = u_viewProjection * worldPos;
      v_worldPosition = worldPos.xyz;
      v_worldNormal = u_normalMatrix * a_normal;
      v_uv = a_uv;
    }
  `
}
```

### Preprocessor Defines

Feature toggles are injected as defines. The shader template branches on them at generation time.

| Define | Values | Purpose |
|---|---|---|
| `SHADOW_CASCADES` | `0`, `3` | Number of CSM cascades. 0 disables shadows. |
| `OIT_PASS` | `0`, `1` | Whether this is the OIT accumulation pass |
| `SKINNING` | `0`, `1` | Enable skeletal animation vertex attributes and bone matrix lookup |
| `VERTEX_COLORS` | `0`, `1` | Enable per-vertex color attribute |
| `NORMAL_MAP` | `0`, `1` | Enable tangent-space normal mapping |
| `EMISSIVE` | `0`, `1` | Enable emissive color output for bloom |
| `NUM_LIGHTS` | `1`-`4` | Number of directional lights |
| `PCF_SAMPLES` | `5`, `9`, `25` | Shadow PCF kernel size (5 = 5-tap Poisson, 25 = 5x5 grid) |
| `TONE_MAPPING` | `'aces'`, `'reinhard'` | Tone mapping operator |

### Shader Variant Explosion

With many defines, the number of shader variants could explode combinatorially. Lynx manages this by:

1. **Permutation pruning.** Not all combinations are valid. OIT shaders never have shadows. Shadow depth shaders never have normal maps. The build script only generates valid combinations.
2. **Lazy compilation.** Shader modules are compiled on first use, not at initialization. The device caches compiled modules by their define key.
3. **Material-driven variants.** The material system determines which defines a mesh needs. Meshes sharing the same material (and thus the same defines) share the same compiled pipeline.

---

## Render Bundles (WebGPU Only)

Render bundles are WebGPU's mechanism for pre-recording sequences of draw commands. Once recorded, a bundle can be replayed in a render pass with near-zero JavaScript overhead. This is critical for hitting 2000+ draw calls at 60fps.

### How Bundles Work

```typescript
/**
 * Record a render bundle for a batch of static meshes.
 * This is called once when the scene is built or when static geometry changes.
 */
const recordStaticBundle = (
  device: GalDevice,
  staticMeshes: readonly StaticMeshRenderCommand[],
  colorFormat: GalTextureFormat,
  depthFormat: GalTextureFormat,
  sampleCount: number
): GalRenderBundle => {
  const encoder = device.createRenderBundleEncoder({
    colorFormats: [colorFormat],
    depthStencilFormat: depthFormat,
    sampleCount
  })

  let currentPipeline: GalRenderPipeline | null = null
  let currentBindGroup0: GalBindGroup | null = null
  let currentBindGroup1: GalBindGroup | null = null

  for (const cmd of staticMeshes) {
    // Minimize state changes — commands are pre-sorted by pipeline, then material
    if (cmd.pipeline !== currentPipeline) {
      encoder.setPipeline(cmd.pipeline)
      currentPipeline = cmd.pipeline
    }
    if (cmd.frameBindGroup !== currentBindGroup0) {
      encoder.setBindGroup(0, cmd.frameBindGroup)
      currentBindGroup0 = cmd.frameBindGroup
    }
    if (cmd.materialBindGroup !== currentBindGroup1) {
      encoder.setBindGroup(1, cmd.materialBindGroup)
      currentBindGroup1 = cmd.materialBindGroup
    }

    encoder.setBindGroup(2, cmd.objectBindGroup, [cmd.uniformOffset])
    encoder.setVertexBuffer(0, cmd.vertexBuffer)
    encoder.setIndexBuffer(cmd.indexBuffer, cmd.indexFormat)
    encoder.drawIndexed(cmd.indexCount, 1, cmd.firstIndex, 0, 0)
  }

  return encoder.finish()
}

/**
 * During frame rendering, static meshes are drawn by replaying the bundle.
 * This replaces potentially thousands of individual JS→GPU calls with one.
 */
const executeOpaquePass = (renderPass: GalRenderPass, staticBundle: GalRenderBundle, dynamicCommands: readonly DynamicMeshRenderCommand[]): void => {
  // Static geometry: single bundle replay (near-zero JS cost)
  renderPass.executeBundles([staticBundle])

  // Dynamic geometry: individual draw calls (these objects moved/changed this frame)
  for (const cmd of dynamicCommands) {
    renderPass.setPipeline(cmd.pipeline)
    renderPass.setBindGroup(0, cmd.frameBindGroup)
    renderPass.setBindGroup(1, cmd.materialBindGroup)
    renderPass.setBindGroup(2, cmd.objectBindGroup, [cmd.uniformOffset])
    renderPass.setVertexBuffer(0, cmd.vertexBuffer)
    renderPass.setIndexBuffer(cmd.indexBuffer, cmd.indexFormat)
    renderPass.drawIndexed(cmd.indexCount)
  }
}
```

### WebGL2 Fallback: Cached State Sequences

WebGL2 cannot pre-record commands. Instead, the WebGL2 backend caches the draw sequence for static meshes:

```typescript
interface CachedDrawSequence {
  readonly commands: readonly CachedDrawCommand[]
  dirty: boolean                                    // Set to true when static scene changes
}

interface CachedDrawCommand {
  readonly vao: WebGLVertexArrayObject
  readonly program: WebGLProgram
  readonly pipelineState: WebGL2PipelineState
  readonly uboBindings: readonly [number, WebGLBuffer, number, number][]
  readonly textureBindings: readonly [number, number, WebGLTexture, WebGLSampler][]
  readonly indexCount: number
  readonly indexType: number
  readonly indexOffset: number
}
```

The cached sequence helps in two ways:

1. **No per-frame sorting.** The draw order is computed once and reused.
2. **VAOs are pre-built.** No per-draw vertex attribute setup.

But unlike WebGPU render bundles, every draw call still goes through JavaScript and issues real GL calls. The performance gap is significant: on a scene with 2000 static meshes, WebGPU with render bundles might spend 0.3ms in JS, while WebGL2 with cached sequences spends 3-5ms. Still far better than the 8-12ms without caching.

---

## WebGL2 State Tracking

The WebGL2 backend maintains a shadow copy of all GL state that the engine touches. Before issuing any `gl.*` state call, it compares the desired value against the shadow and skips if identical. This eliminates redundant driver calls, which is the single most effective optimization for WebGL2 draw call throughput.

### Tracked State

```typescript
interface WebGL2StateTracker {
  // Current program
  currentProgram: WebGLProgram | null

  // Vertex array
  currentVAO: WebGLVertexArrayObject | null

  // Framebuffer
  currentFramebuffer: WebGLFramebuffer | null

  // Blend
  blendEnabled: boolean
  blendSrcRGB: number
  blendDstRGB: number
  blendSrcAlpha: number
  blendDstAlpha: number
  blendEqRGB: number
  blendEqAlpha: number

  // Depth
  depthTestEnabled: boolean
  depthFunc: number
  depthMask: boolean

  // Stencil (minimal — only used for special effects)
  stencilTestEnabled: boolean

  // Culling
  cullEnabled: boolean
  cullFace: number
  frontFace: number

  // Color mask
  colorMask: [boolean, boolean, boolean, boolean]

  // Viewport / scissor
  viewport: [number, number, number, number]
  scissorEnabled: boolean
  scissorRect: [number, number, number, number]

  // UBO binding points (max 16)
  uniformBufferBindings: (WebGLBuffer | null)[]
  uniformBufferOffsets: number[]
  uniformBufferSizes: number[]

  // Texture units (max 16)
  activeTextureUnit: number
  textureBindings: (WebGLTexture | null)[]   // indexed by unit
  samplerBindings: (WebGLSampler | null)[]   // indexed by unit
}
```

### Example: setPipeline with State Tracking

```typescript
const setPipelineWebGL2 = (state: WebGL2StateTracker, gl: WebGL2RenderingContext, pipeline: WebGL2PipelineState): void => {
  // Program
  if (state.currentProgram !== pipeline.program) {
    gl.useProgram(pipeline.program)
    state.currentProgram = pipeline.program
  }

  // Blend
  if (pipeline.blendEnabled !== state.blendEnabled) {
    if (pipeline.blendEnabled) {
      gl.enable(gl.BLEND)
    } else {
      gl.disable(gl.BLEND)
    }
    state.blendEnabled = pipeline.blendEnabled
  }

  if (pipeline.blendEnabled) {
    if (
      pipeline.blendSrcRGB !== state.blendSrcRGB ||
      pipeline.blendDstRGB !== state.blendDstRGB ||
      pipeline.blendSrcAlpha !== state.blendSrcAlpha ||
      pipeline.blendDstAlpha !== state.blendDstAlpha
    ) {
      gl.blendFuncSeparate(
        pipeline.blendSrcRGB,
        pipeline.blendDstRGB,
        pipeline.blendSrcAlpha,
        pipeline.blendDstAlpha
      )
      state.blendSrcRGB = pipeline.blendSrcRGB
      state.blendDstRGB = pipeline.blendDstRGB
      state.blendSrcAlpha = pipeline.blendSrcAlpha
      state.blendDstAlpha = pipeline.blendDstAlpha
    }

    if (pipeline.blendEqRGB !== state.blendEqRGB || pipeline.blendEqAlpha !== state.blendEqAlpha) {
      gl.blendEquationSeparate(pipeline.blendEqRGB, pipeline.blendEqAlpha)
      state.blendEqRGB = pipeline.blendEqRGB
      state.blendEqAlpha = pipeline.blendEqAlpha
    }
  }

  // Depth
  if (pipeline.depthTestEnabled !== state.depthTestEnabled) {
    if (pipeline.depthTestEnabled) {
      gl.enable(gl.DEPTH_TEST)
    } else {
      gl.disable(gl.DEPTH_TEST)
    }
    state.depthTestEnabled = pipeline.depthTestEnabled
  }

  if (pipeline.depthFunc !== state.depthFunc) {
    gl.depthFunc(pipeline.depthFunc)
    state.depthFunc = pipeline.depthFunc
  }

  if (pipeline.depthMask !== state.depthMask) {
    gl.depthMask(pipeline.depthMask)
    state.depthMask = pipeline.depthMask
  }

  // Cull face
  if (pipeline.cullEnabled !== state.cullEnabled) {
    if (pipeline.cullEnabled) {
      gl.enable(gl.CULL_FACE)
    } else {
      gl.disable(gl.CULL_FACE)
    }
    state.cullEnabled = pipeline.cullEnabled
  }

  if (pipeline.cullFace !== state.cullFace) {
    gl.cullFace(pipeline.cullFace)
    state.cullFace = pipeline.cullFace
  }

  if (pipeline.frontFace !== state.frontFace) {
    gl.frontFace(pipeline.frontFace)
    state.frontFace = pipeline.frontFace
  }

  // Color mask
  if (
    pipeline.colorMask[0] !== state.colorMask[0] ||
    pipeline.colorMask[1] !== state.colorMask[1] ||
    pipeline.colorMask[2] !== state.colorMask[2] ||
    pipeline.colorMask[3] !== state.colorMask[3]
  ) {
    gl.colorMask(...pipeline.colorMask)
    state.colorMask = [...pipeline.colorMask]
  }
}
```

### Why This Matters

In a typical 2000-draw-call frame with sorted draw commands:

| Without state tracking | With state tracking |
|---|---|
| ~12,000 GL calls (6 state calls per draw) | ~3,000 GL calls (most state unchanged between sorted draws) |
| ~10ms JS time | ~3ms JS time |

When draw commands are sorted by pipeline first, then by material, consecutive draws almost always share the same pipeline state. The state tracker reduces this to program changes (when pipeline changes) and UBO/texture rebinds (when material changes). Within the same material batch, only the per-object dynamic UBO offset changes -- a single `gl.bindBufferRange()` call per draw.

---

## Buffer Management

### Ring Buffer for Per-Frame Uniform Data

Each frame needs to upload uniform data (camera matrices, lighting parameters, time) that is constant across all draw calls. A ring buffer avoids stalls by writing to a different region of the buffer each frame.

```typescript
interface UniformRingBuffer {
  readonly buffer: GalBuffer
  readonly capacity: number                        // Total size in bytes
  readonly alignment: number                       // UBO offset alignment (typically 256)
  currentOffset: number                            // Write cursor
  readonly frameOffsets: number[]                   // Start offset for each in-flight frame
  frameIndex: number                               // Circular frame counter
}

const RING_BUFFER_FRAMES = 3                        // Triple buffering

const createUniformRingBuffer = (device: GalDevice, sizePerFrame: number): UniformRingBuffer => {
  const alignment = 256                             // WebGPU requires 256-byte alignment for dynamic offsets
  const alignedSize = Math.ceil(sizePerFrame / alignment) * alignment
  const capacity = alignedSize * RING_BUFFER_FRAMES

  return {
    buffer: device.createBuffer({
      label: 'uniform-ring-buffer',
      size: capacity,
      usage: ['uniform'],
      dynamic: true
    }),
    capacity,
    alignment,
    currentOffset: 0,
    frameOffsets: new Array(RING_BUFFER_FRAMES).fill(0),
    frameIndex: 0
  }
}

const beginFrame = (ring: UniformRingBuffer): number => {
  ring.frameIndex = (ring.frameIndex + 1) % RING_BUFFER_FRAMES
  const alignedFrameSize = Math.ceil(ring.capacity / RING_BUFFER_FRAMES / ring.alignment) * ring.alignment
  ring.currentOffset = ring.frameIndex * alignedFrameSize
  ring.frameOffsets[ring.frameIndex] = ring.currentOffset
  return ring.currentOffset
}

const allocateUniform = (ring: UniformRingBuffer, size: number): number => {
  const alignedSize = Math.ceil(size / ring.alignment) * ring.alignment
  const offset = ring.currentOffset
  ring.currentOffset += alignedSize
  return offset
}
```

**Why triple buffering:** WebGPU can have up to 2 frames in flight (one being recorded, one being executed). With triple buffering, the CPU is always writing to a region that the GPU is not reading. This avoids implicit synchronization stalls.

**WebGL2 note:** WebGL2 does not have the same in-flight frame concept (it is synchronous), but triple buffering still helps because `gl.bufferSubData()` on a region the GPU might be reading can cause a pipeline stall on some drivers. Writing to a different region avoids this.

### Dynamic UBO with Offsets for Per-Object Data

Per-object data (model matrix, normal matrix) is stored in a single large UBO. Each draw call uses a different dynamic offset into this buffer. This avoids creating a separate UBO per object.

```typescript
interface DynamicUniformAllocator {
  readonly buffer: GalBuffer
  readonly data: Float32Array                      // CPU-side staging
  readonly alignment: number
  readonly slotSize: number                        // Aligned size of one object's data
  readonly maxSlots: number
  nextSlot: number
}

/** Per-object data layout: model matrix (16 floats) + normal matrix (12 floats, std140 padded) */
const OBJECT_UNIFORM_SIZE = (16 + 12) * 4          // 112 bytes
const OBJECT_UNIFORM_ALIGNED = 256                  // Aligned to 256 bytes

const createDynamicUniformAllocator = (device: GalDevice, maxObjects: number): DynamicUniformAllocator => {
  const totalSize = OBJECT_UNIFORM_ALIGNED * maxObjects

  return {
    buffer: device.createBuffer({
      label: 'dynamic-object-ubo',
      size: totalSize,
      usage: ['uniform'],
      dynamic: true
    }),
    data: new Float32Array(totalSize / 4),
    alignment: OBJECT_UNIFORM_ALIGNED,
    slotSize: OBJECT_UNIFORM_ALIGNED,
    maxSlots: maxObjects,
    nextSlot: 0
  }
}

/**
 * Write one object's uniforms and return the dynamic offset.
 * Called during command list building, before rendering.
 */
const writeObjectUniforms = (
  allocator: DynamicUniformAllocator,
  modelMatrix: Float32Array,                       // 16 floats
  normalMatrix: Float32Array                       // 9 floats (will be padded to std140)
): number => {
  const slot = allocator.nextSlot++
  const baseFloat = (slot * allocator.slotSize) / 4

  // Model matrix: 16 floats, tightly packed
  allocator.data.set(modelMatrix, baseFloat)

  // Normal matrix: 3x3, but std140 layout pads each column to vec4
  // Column 0
  allocator.data[baseFloat + 16] = normalMatrix[0]
  allocator.data[baseFloat + 17] = normalMatrix[1]
  allocator.data[baseFloat + 18] = normalMatrix[2]
  allocator.data[baseFloat + 19] = 0  // padding

  // Column 1
  allocator.data[baseFloat + 20] = normalMatrix[3]
  allocator.data[baseFloat + 21] = normalMatrix[4]
  allocator.data[baseFloat + 22] = normalMatrix[5]
  allocator.data[baseFloat + 23] = 0  // padding

  // Column 2
  allocator.data[baseFloat + 24] = normalMatrix[6]
  allocator.data[baseFloat + 25] = normalMatrix[7]
  allocator.data[baseFloat + 26] = normalMatrix[8]
  allocator.data[baseFloat + 27] = 0  // padding

  return slot * allocator.slotSize
}

/**
 * Flush all written uniforms to the GPU in a single upload.
 * Called once per frame, after all objects have been processed.
 */
const flushObjectUniforms = (device: GalDevice, allocator: DynamicUniformAllocator): void => {
  const bytesUsed = allocator.nextSlot * allocator.slotSize
  if (bytesUsed > 0) {
    device.writeBuffer(
      allocator.buffer,
      new Float32Array(allocator.data.buffer, 0, bytesUsed / 4)
    )
  }
  allocator.nextSlot = 0  // Reset for next frame
}
```

**How dynamic offsets work:** When `setBindGroup(2, objectBindGroup, [offset])` is called, the buffer bound in bind group 2 is accessed at the specified byte offset. In WebGPU, this maps directly to dynamic offsets on `GPUBindGroup`. In WebGL2, this maps to `gl.bindBufferRange(gl.UNIFORM_BUFFER, bindingPoint, buffer, offset, size)`.

### Vertex and Index Buffers

Vertex and index buffers are `STATIC_DRAW` by default. This tells the driver to place them in GPU-local memory for fastest access.

```typescript
const createMeshBuffers = (device: GalDevice, geometry: BufferGeometry): { vertexBuffer: GalBuffer, indexBuffer: GalBuffer } => {
  const vertexBuffer = device.createBuffer({
    label: `${geometry.label}-vertices`,
    size: geometry.interleavedData.byteLength,
    usage: ['vertex'],
    dynamic: false                                 // Static: placed in GPU-optimal memory
  })
  device.writeBuffer(vertexBuffer, geometry.interleavedData)

  const indexBuffer = device.createBuffer({
    label: `${geometry.label}-indices`,
    size: geometry.indexData.byteLength,
    usage: ['index'],
    dynamic: false
  })
  device.writeBuffer(indexBuffer, geometry.indexData)

  return { vertexBuffer, indexBuffer }
}
```

Buffers flagged `dynamic: true` are used for data that changes every frame (e.g., particle positions, animated vertex morph targets). The dynamic flag maps to:
- **WebGPU:** `GPUBufferUsage.COPY_DST` (allows `queue.writeBuffer`)
- **WebGL2:** `gl.DYNAMIC_DRAW` usage hint (tells driver to expect frequent updates)

Static buffers should never be updated after initial upload. If a mesh needs to change its geometry, destroy the old buffer and create a new one. This is a deliberate design choice -- dynamic vertex buffers are a performance trap that the engine avoids by default.

---

## Full Frame Flow Through the GAL

To illustrate how all the pieces fit together, here is the complete sequence of GAL calls for a single frame:

```
1. beginFrame(ringBuffer)                              // Advance ring buffer write position
2. Write per-frame uniforms                            // Camera, lights, time → ring buffer
3. For each visible object:
     writeObjectUniforms(allocator, model, normal)     // Per-object → dynamic UBO
4. flushObjectUniforms(device, allocator)              // Single GPU upload
5. encoder = device.createCommandEncoder()

   // --- Shadow Pass (×3 cascades) ---
6. For each cascade:
     shadowPass = encoder.beginRenderPass(shadowDesc)  // Depth-only, one layer of CSM array
     shadowPass.setPipeline(shadowPipeline)
     shadowPass.setBindGroup(0, cascadeFrameBindGroup)
     For each shadow caster:
       shadowPass.setBindGroup(2, objectBG, [offset])
       shadowPass.setVertexBuffer(0, vb)
       shadowPass.setIndexBuffer(ib, 'uint16')
       shadowPass.drawIndexed(count)
     shadowPass.end()

   // --- Opaque Pass ---
7. opaquePass = encoder.beginRenderPass(opaqueDesc)    // Color + depth, MSAA
   opaquePass.executeBundles([staticBundle])            // WebGPU: replay static geometry
   For each dynamic opaque mesh:                       // Dynamic meshes drawn individually
     opaquePass.setPipeline(...)
     opaquePass.setBindGroup(...)
     opaquePass.drawIndexed(...)
   opaquePass.end()

   // --- OIT Accumulation Pass ---
8. oitPass = encoder.beginRenderPass(oitAccumDesc)     // Two color targets (accum + revealage)
   For each transparent mesh:
     oitPass.setPipeline(oitPipeline)
     oitPass.setBindGroup(...)
     oitPass.drawIndexed(...)
   oitPass.end()

   // --- OIT Composite Pass ---
9. compositePass = encoder.beginRenderPass(compositeDesc)
   compositePass.setPipeline(oitCompositePipeline)
   compositePass.setBindGroup(0, oitResultBindGroup)
   compositePass.draw(3)                               // Fullscreen triangle
   compositePass.end()

   // --- Post-processing (Bloom + Tonemap) ---
10. bloomDownsample passes...
    bloomUpsample passes...
    tonemapPass = encoder.beginRenderPass(tonemapDesc) // Output to canvas backbuffer
    tonemapPass.setPipeline(tonemapPipeline)
    tonemapPass.draw(3)
    tonemapPass.end()

11. commands = encoder.finish()
12. device.submit([commands])
```

On WebGPU, step 12 submits the command buffer to the GPU queue. On WebGL2, steps 6-10 already executed all GL calls inline, and steps 11-12 are no-ops.

---

## Summary of Backend Differences

| Aspect | WebGPU | WebGL2 |
|---|---|---|
| Command model | Record, then submit | Execute immediately |
| Pipeline state | Immutable `GPURenderPipeline` | Cached GL state snapshot, applied with state tracking |
| Bind groups | Native `GPUBindGroup` | Emulated as UBO bindings + texture unit bindings |
| Render bundles | Native `GPURenderBundle` (near-zero replay cost) | Emulated as cached draw sequence (still has per-call JS overhead) |
| Texture views | Native `GPUTextureView` | Emulated (store layer/mip metadata, apply at framebuffer bind) |
| Dynamic UBO offsets | Native (single `setBindGroup` call with offsets) | `gl.bindBufferRange` per draw |
| Shader language | WGSL | GLSL 300 es |
| Storage buffers | Supported | Not available (use UBOs instead) |
| Compute shaders | Supported (future use) | Not available |
| Indirect draw | Supported (future use) | Not available |
| MSAA resolve | Automatic via `resolveTarget` | Manual `gl.blitFramebuffer` |
| State management | Stateless (pipeline objects) | State machine (tracked and deduplicated) |

The GAL ensures that all of these differences are invisible to the rendering code above it. The renderer records the same sequence of `setPipeline`, `setBindGroup`, `draw` calls regardless of backend. The performance characteristics differ -- WebGPU is faster due to render bundles and lower per-call overhead -- but correctness is identical.
