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
# Renderer — Render Pipeline, Draw Call Optimization, Command Sorting

## Render Pipeline Overview

The Lynx renderer consumes a `Scene` and a `Camera` and produces a final image. It uses a multi-pass architecture where each pass writes to one or more render targets. The passes execute in a fixed order every frame:

```
Pass 0  Shadow Map Pass
        Render scene depth from the light's perspective into 3 CSM cascade maps.

Pass 1  Opaque Pass
        Render all opaque geometry front-to-back into a 4x MSAA color+depth target.
        Optional depth pre-pass can be enabled for scenes with high overdraw.

Pass 2  Transparent Pass (OIT Accumulation)
        Render all transparent geometry into two float targets using
        Weighted Blended Order-Independent Transparency (McGuire & Bavoil 2013).
        Target A: RGBA16F accumulation (premultiplied-alpha weighted color).
        Target B: R8 revealage.

Pass 3  OIT Composite
        Fullscreen quad that blends the OIT accumulation/revealage result
        onto the opaque color target.

Pass 4  Post-Processing
        Bloom: extract bright pixels -> progressive downscale chain ->
               progressive upscale chain with additive blend.
        Tone mapping: applied as the final post-process step.

Pass 5  MSAA Resolve + Final Output
        Resolve the 4x MSAA target to the canvas backbuffer (or to an
        offscreen target for further compositing).
```

### Pass Execution Diagram

```
                          ┌──────────────┐
                          │  Scene Graph  │
                          │  + Camera     │
                          └──────┬───────┘
                                 │
                          ┌──────▼───────┐
                          │Frustum Cull  │
                          │AABB vs 6-plane│
                          └──────┬───────┘
                                 │
                          ┌──────▼───────┐
                          │Command Gen   │
                          │+ Sort (radix)│
                          └──────┬───────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
   ┌──────▼───────┐      ┌──────▼───────┐       ┌──────▼──────┐
   │Pass 0        │      │Pass 1        │       │Pass 2       │
   │Shadow Maps   │      │Opaque (MSAA) │       │OIT Accum    │
   │2048² × 3 CSM │      │4× MSAA color │       │RGBA16F +    │
   │              │      │+ depth       │       │R8 revealage │
   └──────┬───────┘      └──────┬───────┘       └──────┬──────┘
          │                      │                      │
          │               ┌──────▼──────────────────────▼──────┐
          │               │Pass 3: OIT Composite               │
          │               │Blend transparency onto opaque      │
          └───────────────►                                    │
                          └──────────────┬─────────────────────┘
                                         │
                          ┌──────────────▼─────────────────────┐
                          │Pass 4: Post-Processing              │
                          │Bloom extract → down chain →         │
                          │up chain → tone map                  │
                          └──────────────┬─────────────────────┘
                                         │
                          ┌──────────────▼─────────────────────┐
                          │Pass 5: MSAA Resolve + Final Output  │
                          │Resolve to canvas / output target    │
                          └────────────────────────────────────┘
```

---

## Command Sorting Strategy

### RenderCommand Structure

Every visible mesh in the frustum generates a single `RenderCommand`. These commands form a flat array that is sorted and then iterated to produce GPU draw calls.

```typescript
type RenderCommand = {
  sortKey: bigint        // u64 — encodes pipeline, material, depth
  mesh: MeshNode         // reference to the scene graph mesh node
  material: MaterialID   // numeric material identifier
  geometry: GeometryID   // numeric geometry identifier (maps to VAO / buffer set)
  worldMatrix: Float32Array  // 4×4 column-major, Z-up right-handed
  boneMatrices: Float32Array | null  // null for non-skinned meshes
  materialIndex: number  // index into material UBO palette
}
```

### Sort Key Encoding (64-bit)

The sort key packs three fields into a single 64-bit integer so that a single radix sort produces the correct draw order while minimizing GPU state changes.

```
Bit layout (MSB → LSB):

  63       60 59            48 47                                  0
  ┌─────────┬────────────────┬─────────────────────────────────────┐
  │Pipeline │  Material ID   │              Depth                  │
  │ 4 bits  │   12 bits      │             48 bits                 │
  └─────────┴────────────────┴─────────────────────────────────────┘
```

| Field | Bits | Range | Purpose |
|---|---|---|---|
| Pipeline ID | 63-60 | 0-15 (16 pipelines) | Groups draws that share the same GPU pipeline state (shader + blend + depth + rasterizer). Changing pipeline is the most expensive state switch. |
| Material ID | 59-48 | 0-4095 (4096 materials) | Groups draws that share the same material bind group (textures, uniforms). Second most expensive switch. |
| Depth | 47-0 | 48-bit quantized | For opaque: front-to-back (smaller depth = drawn first) to maximize early-Z rejection. For transparent: back-to-front (larger depth = drawn first). |

### Key Construction

```typescript
const buildSortKeyOpaque = (
  pipelineId: number,
  materialId: number,
  viewSpaceDepth: number,
  nearPlane: number,
  farPlane: number
): bigint => {
  const pBits = BigInt(pipelineId & 0xF) << 60n
  const mBits = BigInt(materialId & 0xFFF) << 48n

  // Normalize depth to [0, 1] and quantize to 48 bits
  // Front-to-back: smaller depth gets smaller integer
  const t = (viewSpaceDepth - nearPlane) / (farPlane - nearPlane)
  const clamped = Math.max(0, Math.min(1, t))
  const dBits = BigInt(Math.floor(clamped * 0xFFFFFFFFFFFF)) & 0xFFFFFFFFFFFFn

  return pBits | mBits | dBits
}

const buildSortKeyTransparent = (
  pipelineId: number,
  materialId: number,
  viewSpaceDepth: number,
  nearPlane: number,
  farPlane: number
): bigint => {
  const pBits = BigInt(pipelineId & 0xF) << 60n
  const mBits = BigInt(materialId & 0xFFF) << 48n

  // Back-to-front: larger depth gets smaller integer (invert)
  const t = (viewSpaceDepth - nearPlane) / (farPlane - nearPlane)
  const clamped = Math.max(0, Math.min(1, t))
  const inverted = 1.0 - clamped
  const dBits = BigInt(Math.floor(inverted * 0xFFFFFFFFFFFF)) & 0xFFFFFFFFFFFFn

  return pBits | mBits | dBits
}
```

### Radix Sort

Sorting is performed with a least-significant-digit radix sort operating on the 64-bit keys. Radix sort is O(n) in command count and avoids the comparison overhead of `Array.prototype.sort`.

```typescript
// 8 passes of 8-bit radix sort over the 64-bit key
// Uses a pre-allocated auxiliary buffer to avoid per-frame allocation
const radixSort64 = (commands: RenderCommand[], aux: RenderCommand[], count: number): void => {
  const RADIX_BITS = 8
  const RADIX_SIZE = 1 << RADIX_BITS   // 256
  const PASSES = 8                      // 64 / 8

  const counts = new Uint32Array(RADIX_SIZE)

  let src = commands
  let dst = aux

  for (let pass = 0; pass < PASSES; pass++) {
    const shift = BigInt(pass * RADIX_BITS)

    // Count occurrences
    counts.fill(0)
    for (let i = 0; i < count; i++) {
      const bucket = Number((src[i].sortKey >> shift) & 0xFFn)
      counts[bucket]++
    }

    // Prefix sum
    let total = 0
    for (let i = 0; i < RADIX_SIZE; i++) {
      const c = counts[i]
      counts[i] = total
      total += c
    }

    // Scatter
    for (let i = 0; i < count; i++) {
      const bucket = Number((src[i].sortKey >> shift) & 0xFFn)
      dst[counts[bucket]++] = src[i]
    }

    // Swap buffers
    const tmp = src
    src = dst
    dst = tmp
  }

  // If final result ended up in aux, copy back
  if (src !== commands) {
    for (let i = 0; i < count; i++) {
      commands[i] = src[i]
    }
  }
}
```

### Command Caching

The command list is **not** rebuilt every frame. It is cached and only invalidated when the scene graph changes:

- A node is added or removed
- A material assignment changes on a mesh
- A mesh's geometry reference changes
- The camera frustum changes substantially (new objects enter or leave)

When invalidated, the command list is regenerated from the visible set and re-sorted. During steady-state rendering of a static scene with a static camera, the command list is reused as-is, and the only per-frame work is updating dynamic UBO data (transforms for animated objects).

---

## Draw Call Execution

### State Tracking

Iterating sorted commands in order naturally groups draws by pipeline and material. The renderer tracks the currently bound pipeline, material, and geometry to skip redundant GPU calls.

```typescript
const executeCommands = (
  commands: RenderCommand[],
  count: number,
  device: GALDevice,
  passEncoder: GALRenderPassEncoder,
  frameUBO: GALBuffer,
  dynamicUBO: DynamicUBO
): void => {
  let currentPipeline: PipelineID = -1
  let currentMaterial: MaterialID = -1
  let currentGeometry: GeometryID = -1

  for (let i = 0; i < count; i++) {
    const cmd = commands[i]

    // Bind pipeline only when it changes
    const cmdPipeline = Number((cmd.sortKey >> 60n) & 0xFn)
    if (cmdPipeline !== currentPipeline) {
      passEncoder.setPipeline(device.getPipeline(cmdPipeline))
      currentPipeline = cmdPipeline
    }

    // Bind material only when it changes
    if (cmd.material !== currentMaterial) {
      passEncoder.setBindGroup(1, device.getMaterialBindGroup(cmd.material))
      currentMaterial = cmd.material
    }

    // Bind geometry only when it changes
    if (cmd.geometry !== currentGeometry) {
      passEncoder.setVertexBuffer(0, device.getVertexBuffer(cmd.geometry))
      passEncoder.setIndexBuffer(device.getIndexBuffer(cmd.geometry))
      currentGeometry = cmd.geometry
    }

    // Write per-object data to dynamic UBO
    const offset = dynamicUBO.write(cmd.worldMatrix, cmd.materialIndex, cmd.boneMatrices)
    passEncoder.setBindGroup(2, dynamicUBO.bindGroup, [offset])

    // Issue draw call
    const geo = device.getGeometryInfo(cmd.geometry)
    passEncoder.drawIndexed(geo.indexCount, 1, geo.indexOffset, 0, 0)
  }
}
```

### Dynamic Uniform Buffer

Per-object data is written to a large pre-allocated UBO each frame. Each write advances an internal offset. Bind groups reference the same buffer but with different dynamic offsets, so no new bind groups are allocated per frame.

```typescript
type DynamicUBO = {
  buffer: GALBuffer           // GPU buffer, large enough for max objects per frame
  cpuBuffer: ArrayBuffer      // CPU-side staging area
  floatView: Float32Array     // Float view into cpuBuffer
  offset: number              // Current write offset (bytes), 256-byte aligned
  bindGroup: GALBindGroup     // Reused bind group with dynamic offset
}

const DYNAMIC_UBO_ALIGNMENT = 256  // WebGPU requires 256-byte alignment

const dynamicUBOWrite = (
  ubo: DynamicUBO,
  worldMatrix: Float32Array,
  materialIndex: number,
  boneMatrices: Float32Array | null
): number => {
  const baseFloat = ubo.offset / 4

  // World matrix: 16 floats (64 bytes)
  ubo.floatView.set(worldMatrix, baseFloat)

  // Material palette index: 1 float (4 bytes), padded to 16 bytes
  ubo.floatView[baseFloat + 16] = materialIndex

  // Bone matrices: up to 128 × 16 floats = 8192 bytes
  if (boneMatrices) {
    ubo.floatView.set(boneMatrices, baseFloat + 20)  // offset past world + padding
  }

  const currentOffset = ubo.offset

  // Advance offset, aligned to 256 bytes
  const dataSize = boneMatrices ? 64 + 16 + boneMatrices.byteLength : 64 + 16
  ubo.offset += Math.ceil(dataSize / DYNAMIC_UBO_ALIGNMENT) * DYNAMIC_UBO_ALIGNMENT

  return currentOffset
}
```

### WebGPU: Render Bundles

For static objects (non-animated meshes whose transforms have not changed), WebGPU render bundles are pre-recorded and replayed each frame. This avoids re-encoding draw commands for the majority of a typical scene.

```typescript
const buildStaticRenderBundle = (
  device: GALDevice,
  staticCommands: RenderCommand[],
  count: number,
  renderBundleDescriptor: GPURenderBundleEncoderDescriptor,
  dynamicUBO: DynamicUBO
): GPURenderBundle => {
  const encoder = device.nativeDevice.createRenderBundleEncoder(renderBundleDescriptor)

  // Record commands into the bundle — same logic as executeCommands
  let currentPipeline: PipelineID = -1
  let currentMaterial: MaterialID = -1
  let currentGeometry: GeometryID = -1

  for (let i = 0; i < count; i++) {
    const cmd = staticCommands[i]

    const cmdPipeline = Number((cmd.sortKey >> 60n) & 0xFn)
    if (cmdPipeline !== currentPipeline) {
      encoder.setPipeline(device.getPipeline(cmdPipeline))
      currentPipeline = cmdPipeline
    }

    if (cmd.material !== currentMaterial) {
      encoder.setBindGroup(1, device.getMaterialBindGroup(cmd.material))
      currentMaterial = cmd.material
    }

    if (cmd.geometry !== currentGeometry) {
      encoder.setVertexBuffer(0, device.getVertexBuffer(cmd.geometry))
      encoder.setIndexBuffer(device.getIndexBuffer(cmd.geometry), 'uint16')
      currentGeometry = cmd.geometry
    }

    const offset = dynamicUBO.write(cmd.worldMatrix, cmd.materialIndex, cmd.boneMatrices)
    encoder.setBindGroup(2, dynamicUBO.bindGroup, [offset])

    const geo = device.getGeometryInfo(cmd.geometry)
    encoder.drawIndexed(geo.indexCount, 1, geo.indexOffset, 0, 0)
  }

  return encoder.finish()
}
```

When the scene graph marks a static bundle as dirty (node added/removed, material changed), the bundle is discarded and rebuilt on the next frame. Animated objects always use the dynamic encoder path.

### WebGL2: VAO and State Minimization

On WebGL2, the same sorted-command-list approach applies. Each geometry is backed by a Vertex Array Object (VAO). The WebGL2 backend tracks bound state and skips redundant `gl.bindVertexArray`, `gl.useProgram`, and `gl.bindTexture` calls.

```typescript
const executeCommandsWebGL2 = (
  commands: RenderCommand[],
  count: number,
  gl: WebGL2RenderingContext,
  state: WebGL2StateTracker,
  dynamicUBO: WebGL2DynamicUBO
): void => {
  for (let i = 0; i < count; i++) {
    const cmd = commands[i]

    const cmdPipeline = Number((cmd.sortKey >> 60n) & 0xFn)
    if (cmdPipeline !== state.currentPipeline) {
      const pipeline = state.pipelines[cmdPipeline]
      gl.useProgram(pipeline.program)
      state.applyDepthStencil(pipeline.depthStencil)
      state.applyBlend(pipeline.blend)
      state.applyRasterizer(pipeline.rasterizer)
      state.currentPipeline = cmdPipeline
    }

    if (cmd.material !== state.currentMaterial) {
      state.bindMaterialTextures(gl, cmd.material)
      state.currentMaterial = cmd.material
    }

    if (cmd.geometry !== state.currentGeometry) {
      gl.bindVertexArray(state.vaos[cmd.geometry])
      state.currentGeometry = cmd.geometry
    }

    // Write per-object data via gl.bufferSubData to dynamic UBO
    const offset = dynamicUBO.write(gl, cmd.worldMatrix, cmd.materialIndex, cmd.boneMatrices)
    gl.bindBufferRange(
      gl.UNIFORM_BUFFER,
      2,  // binding point 2 = per-object
      dynamicUBO.buffer,
      offset,
      dynamicUBO.stride
    )

    const geo = state.geometryInfo[cmd.geometry]
    gl.drawElements(gl.TRIANGLES, geo.indexCount, gl.UNSIGNED_SHORT, geo.indexOffset * 2)
  }
}
```

---

## MSAA Strategy

### Render Target Layout for MSAA

Opaque geometry renders into a 4x MSAA render target. The transparency pass renders into separate non-MSAA float targets because Weighted Blended OIT requires additive blending into float textures, which does not benefit from multisampling.

```
Opaque Pass (4x MSAA):
  Color attachment: RGBA8 multisample (count=4)
  Depth attachment: Depth24Plus multisample (count=4)

Transparency Pass (no MSAA):
  Color attachment 0: RGBA16F (OIT accumulation)
  Color attachment 1: R8 (OIT revealage)
  Depth attachment: read-only copy of opaque depth (for depth testing without writing)

OIT Composite:
  Writes onto the MSAA color target (blends transparency result)

Post-Processing:
  Operates on the MSAA target

MSAA Resolve:
  Resolves 4x MSAA → 1x final output
```

### Why Resolve Happens Last

Resolving MSAA before compositing transparency would lose subpixel edge information on opaque geometry behind transparent surfaces. By keeping the MSAA target intact through OIT compositing and post-processing, the final resolve produces clean antialiased edges everywhere.

### WebGPU MSAA

```typescript
// Pipeline creation with MSAA
const createOpaquePipeline = (device: GPUDevice, format: GPUTextureFormat): GPURenderPipeline => {
  return device.createRenderPipeline({
    // ...shader, layout, vertex state...
    multisample: {
      count: 4,
    },
    fragment: {
      // ...
      targets: [{ format }],
    },
    depthStencil: {
      format: 'depth24plus',
      depthWriteEnabled: true,
      depthCompare: 'less',
    },
  })
}

// Render pass with MSAA color and resolve target
const beginOpaquePass = (
  encoder: GPUCommandEncoder,
  msaaView: GPUTextureView,
  resolveView: GPUTextureView,
  depthView: GPUTextureView
): GPURenderPassEncoder => {
  return encoder.beginRenderPass({
    colorAttachments: [{
      view: msaaView,           // 4x MSAA texture view
      resolveTarget: undefined, // resolve happens in Pass 5, not here
      loadOp: 'clear',
      storeOp: 'store',
      clearValue: { r: 0.0, g: 0.0, b: 0.0, a: 1.0 },
    }],
    depthStencilAttachment: {
      view: depthView,          // 4x MSAA depth view
      depthLoadOp: 'clear',
      depthStoreOp: 'store',
      depthClearValue: 1.0,
    },
  })
}
```

### WebGL2 MSAA

```typescript
const setupMSAAFramebuffer = (gl: WebGL2RenderingContext, width: number, height: number) => {
  const msaaFBO = gl.createFramebuffer()
  gl.bindFramebuffer(gl.FRAMEBUFFER, msaaFBO)

  // Color renderbuffer — 4x MSAA
  const colorRB = gl.createRenderbuffer()
  gl.bindRenderbuffer(gl.RENDERBUFFER, colorRB)
  gl.renderbufferStorageMultisample(gl.RENDERBUFFER, 4, gl.RGBA8, width, height)
  gl.framebufferRenderbuffer(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.RENDERBUFFER, colorRB)

  // Depth renderbuffer — 4x MSAA
  const depthRB = gl.createRenderbuffer()
  gl.bindRenderbuffer(gl.RENDERBUFFER, depthRB)
  gl.renderbufferStorageMultisample(gl.RENDERBUFFER, 4, gl.DEPTH_COMPONENT24, width, height)
  gl.framebufferRenderbuffer(gl.FRAMEBUFFER, gl.DEPTH_ATTACHMENT, gl.RENDERBUFFER, depthRB)

  return { msaaFBO, colorRB, depthRB }
}

// Resolve by blitting to a non-MSAA framebuffer
const resolveMSAA = (
  gl: WebGL2RenderingContext,
  msaaFBO: WebGLFramebuffer,
  resolveFBO: WebGLFramebuffer,
  width: number,
  height: number
): void => {
  gl.bindFramebuffer(gl.READ_FRAMEBUFFER, msaaFBO)
  gl.bindFramebuffer(gl.DRAW_FRAMEBUFFER, resolveFBO)
  gl.blitFramebuffer(
    0, 0, width, height,
    0, 0, width, height,
    gl.COLOR_BUFFER_BIT,
    gl.NEAREST
  )
}
```

---

## Frustum Culling Integration

### Pipeline Position

Frustum culling runs before command generation. Only meshes whose world-space AABBs intersect the camera frustum produce `RenderCommand` entries. This means culled objects incur zero cost in sorting and draw call execution.

```
Scene Graph Traversal
        │
        ▼
Update World Transforms (dirty nodes only)
        │
        ▼
Frustum Cull (AABB vs 6 planes)  ◄── rejects ~60-80% of objects in typical scenes
        │
        ▼
Generate RenderCommands (visible set only)
        │
        ▼
Radix Sort
        │
        ▼
Execute Draw Calls
```

### 6-Plane Frustum Test

The frustum is represented as six planes in world space (left, right, top, bottom, near, far), each stored as a `Vec4` (nx, ny, nz, d) with the normal pointing inward. A world-space AABB is tested against all six planes. If the AABB is entirely outside any single plane, the object is culled.

```typescript
type Frustum = {
  planes: Float32Array  // 6 planes × 4 components = 24 floats
}

// Returns true if the AABB is at least partially inside the frustum
const testAABBFrustum = (
  frustum: Frustum,
  minX: number, minY: number, minZ: number,
  maxX: number, maxY: number, maxZ: number
): boolean => {
  const p = frustum.planes

  for (let i = 0; i < 6; i++) {
    const offset = i * 4
    const nx = p[offset]
    const ny = p[offset + 1]
    const nz = p[offset + 2]
    const d  = p[offset + 3]

    // Find the AABB vertex most in the direction of the plane normal (p-vertex)
    const px = nx >= 0 ? maxX : minX
    const py = ny >= 0 ? maxY : minY
    const pz = nz >= 0 ? maxZ : minZ

    // If the p-vertex is outside, the entire AABB is outside this plane
    if (nx * px + ny * py + nz * pz + d < 0) {
      return false  // early-out: fully outside
    }
  }

  return true  // inside or intersecting all planes
}
```

The early-out on the first failing plane means that on average only 2-3 planes need to be tested per object (objects behind the camera fail the near or far plane immediately).

### Frustum Extraction from View-Projection Matrix

```typescript
// Extracts 6 frustum planes from a column-major 4x4 view-projection matrix
// Z-up right-handed coordinate system
const extractFrustum = (vp: Float32Array, out: Frustum): void => {
  const p = out.planes

  // Left:   row3 + row0
  p[0]  = vp[3]  + vp[0]
  p[1]  = vp[7]  + vp[4]
  p[2]  = vp[11] + vp[8]
  p[3]  = vp[15] + vp[12]
  normalizePlane(p, 0)

  // Right:  row3 - row0
  p[4]  = vp[3]  - vp[0]
  p[5]  = vp[7]  - vp[4]
  p[6]  = vp[11] - vp[8]
  p[7]  = vp[15] - vp[12]
  normalizePlane(p, 4)

  // Bottom: row3 + row1
  p[8]  = vp[3]  + vp[1]
  p[9]  = vp[7]  + vp[5]
  p[10] = vp[11] + vp[9]
  p[11] = vp[15] + vp[13]
  normalizePlane(p, 8)

  // Top:    row3 - row1
  p[12] = vp[3]  - vp[1]
  p[13] = vp[7]  - vp[5]
  p[14] = vp[11] - vp[9]
  p[15] = vp[15] - vp[13]
  normalizePlane(p, 12)

  // Near:   row3 + row2
  p[16] = vp[3]  + vp[2]
  p[17] = vp[7]  + vp[6]
  p[18] = vp[11] + vp[10]
  p[19] = vp[15] + vp[14]
  normalizePlane(p, 16)

  // Far:    row3 - row2
  p[20] = vp[3]  - vp[2]
  p[21] = vp[7]  - vp[6]
  p[22] = vp[11] - vp[10]
  p[23] = vp[15] - vp[14]
  normalizePlane(p, 20)
}

const normalizePlane = (planes: Float32Array, offset: number): void => {
  const nx = planes[offset]
  const ny = planes[offset + 1]
  const nz = planes[offset + 2]
  const len = Math.sqrt(nx * nx + ny * ny + nz * nz)
  if (len > 0) {
    const invLen = 1.0 / len
    planes[offset]     *= invLen
    planes[offset + 1] *= invLen
    planes[offset + 2] *= invLen
    planes[offset + 3] *= invLen
  }
}
```

---

## Render Targets and Framebuffer Management

### Full Set of Render Targets

| Target | Format | Size | MSAA | Purpose |
|---|---|---|---|---|
| Opaque Color | RGBA8 | viewport | 4x | Main opaque rendering target |
| Opaque Depth | Depth24Plus | viewport | 4x | Depth buffer for opaque pass |
| Shadow Cascade 0 | Depth24 | 2048 x 2048 | 1x | CSM near cascade |
| Shadow Cascade 1 | Depth24 | 2048 x 2048 | 1x | CSM mid cascade |
| Shadow Cascade 2 | Depth24 | 2048 x 2048 | 1x | CSM far cascade |
| OIT Accumulation | RGBA16F | viewport | 1x | Weighted blended color accumulation |
| OIT Revealage | R8 | viewport | 1x | Weighted blended revealage |
| Bloom 0 | RGBA16F | viewport / 2 | 1x | Bloom bright-pass extract + downscale level 0 |
| Bloom 1 | RGBA16F | viewport / 4 | 1x | Bloom downscale level 1 |
| Bloom 2 | RGBA16F | viewport / 8 | 1x | Bloom downscale level 2 |
| Bloom 3 | RGBA16F | viewport / 16 | 1x | Bloom downscale level 3 |
| Bloom 4 | RGBA16F | viewport / 32 | 1x | Bloom downscale level 4 |
| Resolve Output | RGBA8 | viewport | 1x | MSAA resolve destination / final output |

### Lazy Allocation

Render targets are allocated on demand, not at initialization time. The renderer tracks which features the scene actually uses:

- **Shadow maps**: only allocated if the scene contains a `DirectionalLight` with `castShadow: true`
- **OIT targets**: only allocated if the scene contains at least one mesh with a transparent material
- **Bloom chain**: only allocated if post-processing bloom is enabled (`renderer.bloom = true`)
- **MSAA targets**: always allocated when MSAA is enabled (default)

When the viewport resizes, all viewport-relative targets are released and re-allocated at the new resolution on the next frame.

```typescript
type RenderTargetCache = {
  opaqueColor: GALTexture | null
  opaqueDepth: GALTexture | null
  shadowCascades: GALTexture[] | null    // array of 3 depth textures
  oitAccumulation: GALTexture | null
  oitRevealage: GALTexture | null
  bloomChain: GALTexture[] | null        // array of 5 half-res textures
  resolveOutput: GALTexture | null
  viewportWidth: number
  viewportHeight: number
}

const ensureRenderTargets = (
  cache: RenderTargetCache,
  device: GALDevice,
  width: number,
  height: number,
  scene: Scene,
  config: RendererConfig
): void => {
  // Release all if viewport changed
  if (width !== cache.viewportWidth || height !== cache.viewportHeight) {
    releaseViewportTargets(cache, device)
    cache.viewportWidth = width
    cache.viewportHeight = height
  }

  // Always need opaque targets when MSAA enabled
  if (!cache.opaqueColor && config.msaa) {
    cache.opaqueColor = device.createTexture({
      width, height,
      format: 'rgba8unorm',
      sampleCount: 4,
      usage: 'render-target',
    })
    cache.opaqueDepth = device.createTexture({
      width, height,
      format: 'depth24plus',
      sampleCount: 4,
      usage: 'render-target',
    })
  }

  // Shadow maps — only if scene has shadow-casting light
  if (!cache.shadowCascades && scene.hasShadowCastingLight()) {
    cache.shadowCascades = [0, 1, 2].map(() =>
      device.createTexture({
        width: 2048, height: 2048,
        format: 'depth24',
        sampleCount: 1,
        usage: 'render-target|sampled',
      })
    )
  }

  // OIT targets — only if scene has transparent meshes
  if (!cache.oitAccumulation && scene.hasTransparentMeshes()) {
    cache.oitAccumulation = device.createTexture({
      width, height,
      format: 'rgba16float',
      sampleCount: 1,
      usage: 'render-target|sampled',
    })
    cache.oitRevealage = device.createTexture({
      width, height,
      format: 'r8unorm',
      sampleCount: 1,
      usage: 'render-target|sampled',
    })
  }

  // Bloom chain — only if bloom enabled
  if (!cache.bloomChain && config.bloom) {
    cache.bloomChain = []
    let w = Math.floor(width / 2)
    let h = Math.floor(height / 2)
    for (let i = 0; i < 5; i++) {
      cache.bloomChain.push(device.createTexture({
        width: w, height: h,
        format: 'rgba16float',
        sampleCount: 1,
        usage: 'render-target|sampled',
      }))
      w = Math.max(1, Math.floor(w / 2))
      h = Math.max(1, Math.floor(h / 2))
    }
  }

  // Resolve output — always needed
  if (!cache.resolveOutput) {
    cache.resolveOutput = device.createTexture({
      width, height,
      format: 'rgba8unorm',
      sampleCount: 1,
      usage: 'render-target|sampled',
    })
  }
}
```

---

## Shader Programs

### Shader Inventory

| Shader | Vertex | Fragment | Purpose |
|---|---|---|---|
| Opaque Lambert | `opaque.vert` | `lambert.frag` | Diffuse-only lit shading (N dot L + ambient) |
| Opaque Basic | `opaque.vert` | `basic.frag` | Unlit solid color / texture |
| Shadow Depth | `shadow.vert` | `shadow.frag` | Depth-only output for shadow maps |
| OIT Accumulation | `opaque.vert` | `oit_accum.frag` | Weighted blended OIT accumulation pass |
| OIT Composite | `fullscreen.vert` | `oit_composite.frag` | Composites OIT result onto opaque |
| Bloom Extract | `fullscreen.vert` | `bloom_extract.frag` | Extracts bright pixels above threshold |
| Bloom Blur | `fullscreen.vert` | `bloom_blur.frag` | Dual Kawase blur (down and up) |
| Tone Map | `fullscreen.vert` | `tonemap.frag` | ACES filmic tone mapping + gamma correction |
| Fullscreen Blit | `fullscreen.vert` | `blit.frag` | Simple texture copy to screen |

### Preprocessor Define System

All shaders share a common preprocessor system. Defines are injected at the top of the shader source before compilation. This allows a single shader source file to produce multiple pipeline variants.

```typescript
type ShaderDefines = {
  HAS_UV?: boolean
  HAS_VERTEX_COLOR?: boolean
  HAS_NORMAL_MAP?: boolean
  HAS_SKINNING?: boolean
  NUM_BONES?: number
  NUM_CSM_CASCADES?: number
  RECEIVE_SHADOWS?: boolean
  USE_FOG?: boolean
  TONE_MAPPING?: 'ACES' | 'REINHARD' | 'LINEAR'
}

const buildShaderSource = (source: string, defines: ShaderDefines): string => {
  let header = ''
  for (const [key, value] of Object.entries(defines)) {
    if (value === true) {
      header += `#define ${key}\n`
    } else if (value !== false && value !== undefined) {
      header += `#define ${key} ${value}\n`
    }
  }
  return header + source
}
```

### Vertex Shader Architecture

**opaque.vert** — Used by Lambert, Basic, and OIT accumulation.

```glsl
// --- Injected defines ---
// #define HAS_UV
// #define HAS_VERTEX_COLOR
// #define HAS_SKINNING
// #define NUM_BONES 128
// #define RECEIVE_SHADOWS
// #define NUM_CSM_CASCADES 3

// --- Uniform blocks ---
layout(std140) uniform FrameUniforms {       // binding 0
    mat4 u_viewMatrix;
    mat4 u_projectionMatrix;
    mat4 u_viewProjectionMatrix;
    vec3 u_cameraPosition;                   // Z-up world space
    float u_time;
    vec3 u_sunDirection;                     // normalized, Z-up
    float u_padding0;
    vec3 u_sunColor;
    float u_sunIntensity;
    vec3 u_ambientColor;
    float u_ambientIntensity;
#ifdef RECEIVE_SHADOWS
    mat4 u_shadowMatrices[NUM_CSM_CASCADES]; // world → shadow clip per cascade
    float u_cascadeSplits[NUM_CSM_CASCADES];
#endif
};

layout(std140) uniform ObjectUniforms {      // binding 2, dynamic offset
    mat4 u_worldMatrix;
    float u_materialIndex;
    float u_pad1;
    float u_pad2;
    float u_pad3;
#ifdef HAS_SKINNING
    mat4 u_boneMatrices[NUM_BONES];
#endif
};

// --- Attributes ---
layout(location = 0) in vec3 a_position;     // Z-up right-handed
layout(location = 1) in vec3 a_normal;
#ifdef HAS_UV
layout(location = 2) in vec2 a_uv;
#endif
#ifdef HAS_VERTEX_COLOR
layout(location = 3) in vec4 a_color;
#endif
#ifdef HAS_SKINNING
layout(location = 4) in vec4 a_joints;
layout(location = 5) in vec4 a_weights;
#endif

// --- Varyings ---
out vec3 v_worldPosition;
out vec3 v_worldNormal;
#ifdef HAS_UV
out vec2 v_uv;
#endif
#ifdef HAS_VERTEX_COLOR
out vec4 v_color;
#endif
#ifdef RECEIVE_SHADOWS
out vec4 v_shadowCoords[NUM_CSM_CASCADES];
#endif
out float v_viewDepth;

void main() {
    vec4 localPos = vec4(a_position, 1.0);
    vec3 localNormal = a_normal;

#ifdef HAS_SKINNING
    mat4 skinMatrix =
        u_boneMatrices[int(a_joints.x)] * a_weights.x +
        u_boneMatrices[int(a_joints.y)] * a_weights.y +
        u_boneMatrices[int(a_joints.z)] * a_weights.z +
        u_boneMatrices[int(a_joints.w)] * a_weights.w;
    localPos = skinMatrix * localPos;
    localNormal = mat3(skinMatrix) * localNormal;
#endif

    vec4 worldPos = u_worldMatrix * localPos;
    v_worldPosition = worldPos.xyz;
    v_worldNormal = normalize(mat3(u_worldMatrix) * localNormal);

#ifdef HAS_UV
    v_uv = a_uv;
#endif
#ifdef HAS_VERTEX_COLOR
    v_color = a_color;
#endif

#ifdef RECEIVE_SHADOWS
    for (int i = 0; i < NUM_CSM_CASCADES; i++) {
        v_shadowCoords[i] = u_shadowMatrices[i] * worldPos;
    }
#endif

    vec4 viewPos = u_viewMatrix * worldPos;
    v_viewDepth = -viewPos.z;   // positive depth into screen (Z-up RH)

    gl_Position = u_projectionMatrix * viewPos;
}
```

### Fragment Shader: Lambert

**lambert.frag** — Diffuse-only lit shading.

```glsl
// --- Injected defines ---
// #define HAS_UV
// #define HAS_VERTEX_COLOR
// #define RECEIVE_SHADOWS
// #define NUM_CSM_CASCADES 3

precision highp float;

layout(std140) uniform FrameUniforms {       // same as vertex
    mat4 u_viewMatrix;
    mat4 u_projectionMatrix;
    mat4 u_viewProjectionMatrix;
    vec3 u_cameraPosition;
    float u_time;
    vec3 u_sunDirection;
    float u_padding0;
    vec3 u_sunColor;
    float u_sunIntensity;
    vec3 u_ambientColor;
    float u_ambientIntensity;
#ifdef RECEIVE_SHADOWS
    mat4 u_shadowMatrices[NUM_CSM_CASCADES];
    float u_cascadeSplits[NUM_CSM_CASCADES];
#endif
};

layout(std140) uniform MaterialUniforms {    // binding 1
    vec4 u_baseColor;
    float u_emissiveIntensity;
    float u_pad_m1;
    float u_pad_m2;
    float u_pad_m3;
};

#ifdef HAS_UV
uniform sampler2D u_baseColorMap;
#endif
#ifdef RECEIVE_SHADOWS
uniform sampler2DShadow u_shadowMaps[NUM_CSM_CASCADES];
#endif

in vec3 v_worldPosition;
in vec3 v_worldNormal;
#ifdef HAS_UV
in vec2 v_uv;
#endif
#ifdef HAS_VERTEX_COLOR
in vec4 v_color;
#endif
#ifdef RECEIVE_SHADOWS
in vec4 v_shadowCoords[NUM_CSM_CASCADES];
#endif
in float v_viewDepth;

layout(location = 0) out vec4 fragColor;

float sampleShadowPCF(sampler2DShadow map, vec4 coords) {
    vec3 projected = coords.xyz / coords.w;
    projected = projected * 0.5 + 0.5;

    float shadow = 0.0;
    vec2 texelSize = vec2(1.0 / 2048.0);

    // 5x5 PCF kernel
    for (int x = -2; x <= 2; x++) {
        for (int y = -2; y <= 2; y++) {
            vec3 sampleCoord = vec3(projected.xy + vec2(x, y) * texelSize, projected.z);
            shadow += texture(map, sampleCoord);
        }
    }
    return shadow / 25.0;
}

void main() {
    vec4 albedo = u_baseColor;

#ifdef HAS_UV
    albedo *= texture(u_baseColorMap, v_uv);
#endif
#ifdef HAS_VERTEX_COLOR
    albedo *= v_color;
#endif

    vec3 N = normalize(v_worldNormal);
    vec3 L = normalize(u_sunDirection);

    // Lambert diffuse
    float NdotL = max(dot(N, L), 0.0);
    vec3 diffuse = u_sunColor * u_sunIntensity * NdotL;
    vec3 ambient = u_ambientColor * u_ambientIntensity;

    // Shadow
    float shadow = 1.0;
#ifdef RECEIVE_SHADOWS
    // Select cascade based on view depth
    int cascade = NUM_CSM_CASCADES - 1;
    for (int i = 0; i < NUM_CSM_CASCADES; i++) {
        if (v_viewDepth < u_cascadeSplits[i]) {
            cascade = i;
            break;
        }
    }
    shadow = sampleShadowPCF(u_shadowMaps[cascade], v_shadowCoords[cascade]);
#endif

    vec3 lighting = ambient + diffuse * shadow;
    vec3 color = albedo.rgb * lighting;

    // Emissive contribution (for bloom extraction)
    color += albedo.rgb * u_emissiveIntensity;

    fragColor = vec4(color, albedo.a);
}
```

### Fragment Shader: Basic (Unlit)

**basic.frag** — No lighting, just base color and optional texture/vertex color.

```glsl
precision highp float;

layout(std140) uniform MaterialUniforms {
    vec4 u_baseColor;
    float u_emissiveIntensity;
    float u_pad_m1;
    float u_pad_m2;
    float u_pad_m3;
};

#ifdef HAS_UV
uniform sampler2D u_baseColorMap;
#endif

in vec3 v_worldPosition;
in vec3 v_worldNormal;
#ifdef HAS_UV
in vec2 v_uv;
#endif
#ifdef HAS_VERTEX_COLOR
in vec4 v_color;
#endif

layout(location = 0) out vec4 fragColor;

void main() {
    vec4 color = u_baseColor;

#ifdef HAS_UV
    color *= texture(u_baseColorMap, v_uv);
#endif
#ifdef HAS_VERTEX_COLOR
    color *= v_color;
#endif

    fragColor = color;
}
```

### Fragment Shader: Shadow Depth

**shadow.vert / shadow.frag** — Minimal shaders that output only depth.

```glsl
// shadow.vert
layout(std140) uniform ShadowUniforms {
    mat4 u_lightViewProjection;
};
layout(std140) uniform ObjectUniforms {
    mat4 u_worldMatrix;
    float u_materialIndex;
    float u_pad1;
    float u_pad2;
    float u_pad3;
#ifdef HAS_SKINNING
    mat4 u_boneMatrices[NUM_BONES];
#endif
};

layout(location = 0) in vec3 a_position;
#ifdef HAS_SKINNING
layout(location = 4) in vec4 a_joints;
layout(location = 5) in vec4 a_weights;
#endif

void main() {
    vec4 localPos = vec4(a_position, 1.0);
#ifdef HAS_SKINNING
    mat4 skinMatrix =
        u_boneMatrices[int(a_joints.x)] * a_weights.x +
        u_boneMatrices[int(a_joints.y)] * a_weights.y +
        u_boneMatrices[int(a_joints.z)] * a_weights.z +
        u_boneMatrices[int(a_joints.w)] * a_weights.w;
    localPos = skinMatrix * localPos;
#endif
    gl_Position = u_lightViewProjection * u_worldMatrix * localPos;
}

// shadow.frag
precision highp float;
void main() {
    // Depth is written automatically by the hardware
    // No color output needed — depth-only pass
}
```

### Fragment Shader: OIT Accumulation

**oit_accum.frag** — Weighted Blended OIT (McGuire & Bavoil 2013).

```glsl
precision highp float;

layout(std140) uniform MaterialUniforms {
    vec4 u_baseColor;
    float u_emissiveIntensity;
    float u_pad_m1;
    float u_pad_m2;
    float u_pad_m3;
};

#ifdef HAS_UV
uniform sampler2D u_baseColorMap;
#endif

in vec3 v_worldPosition;
in vec3 v_worldNormal;
#ifdef HAS_UV
in vec2 v_uv;
#endif
#ifdef HAS_VERTEX_COLOR
in vec4 v_color;
#endif
in float v_viewDepth;

// MRT output: accumulation (RGBA16F) and revealage (R8)
layout(location = 0) out vec4 accumulation;
layout(location = 1) out float revealage;

void main() {
    vec4 color = u_baseColor;

#ifdef HAS_UV
    color *= texture(u_baseColorMap, v_uv);
#endif
#ifdef HAS_VERTEX_COLOR
    color *= v_color;
#endif

    // TODO: lighting could be applied here too for lit transparency

    float alpha = color.a;

    // Depth weight function (McGuire & Bavoil 2013, Equation 10)
    float viewDepth = v_viewDepth;
    float weight = max(min(1.0, max(max(color.r, color.g), color.b) * alpha), alpha) *
                   clamp(0.03 / (1e-5 + pow(viewDepth / 200.0, 4.0)), 1e-2, 3e3);

    // Accumulation: premultiplied-alpha color × weight
    accumulation = vec4(color.rgb * alpha * weight, alpha * weight);

    // Revealage: how much background shows through
    revealage = alpha;
}
```

**Blend state for OIT accumulation pass:**
- Accumulation attachment: `src=ONE, dst=ONE` (additive)
- Revealage attachment: `src=ZERO, dst=ONE_MINUS_SRC_COLOR` (product)

### Fragment Shader: OIT Composite

**oit_composite.frag** — Blends OIT result onto the opaque framebuffer.

```glsl
precision highp float;

uniform sampler2D u_accumulation;
uniform sampler2D u_revealage;

in vec2 v_uv;

layout(location = 0) out vec4 fragColor;

void main() {
    vec4 accum = texture(u_accumulation, v_uv);
    float reveal = texture(u_revealage, v_uv).r;

    // If revealage is ~1.0, nothing transparent was drawn here
    if (reveal >= 1.0) {
        discard;
    }

    // Reconstruct average color
    vec3 averageColor = accum.rgb / max(accum.a, 1e-5);

    // Composite: transparent color * (1 - revealage) over destination
    fragColor = vec4(averageColor, 1.0 - reveal);
}
```

**Blend state for OIT composite pass:** `src=SRC_ALPHA, dst=ONE_MINUS_SRC_ALPHA` (standard alpha blending).

### Fragment Shader: Bloom Extract

**bloom_extract.frag** — Extracts pixels brighter than a threshold.

```glsl
precision highp float;

uniform sampler2D u_sceneColor;
uniform float u_threshold;

in vec2 v_uv;

layout(location = 0) out vec4 fragColor;

void main() {
    vec3 color = texture(u_sceneColor, v_uv).rgb;

    float brightness = dot(color, vec3(0.2126, 0.7152, 0.0722));
    vec3 bloom = color * max(brightness - u_threshold, 0.0) / max(brightness, 1e-5);

    fragColor = vec4(bloom, 1.0);
}
```

### Fragment Shader: Bloom Blur (Dual Kawase)

**bloom_blur.frag** — Used for both downscale and upscale passes with different UV offsets.

```glsl
precision highp float;

uniform sampler2D u_sourceTexture;
uniform vec2 u_texelSize;     // 1.0 / source resolution
uniform float u_direction;    // 0.0 = downsample, 1.0 = upsample

in vec2 v_uv;

layout(location = 0) out vec4 fragColor;

void main() {
    vec2 ts = u_texelSize;

    if (u_direction < 0.5) {
        // Downsample: 5-tap pattern
        vec3 color = texture(u_sourceTexture, v_uv).rgb * 4.0;
        color += texture(u_sourceTexture, v_uv + vec2(-ts.x, -ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2( ts.x, -ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2(-ts.x,  ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2( ts.x,  ts.y)).rgb;
        fragColor = vec4(color / 8.0, 1.0);
    } else {
        // Upsample: 8-tap tent filter
        vec3 color = vec3(0.0);
        color += texture(u_sourceTexture, v_uv + vec2(-ts.x,  0.0)).rgb * 2.0;
        color += texture(u_sourceTexture, v_uv + vec2( ts.x,  0.0)).rgb * 2.0;
        color += texture(u_sourceTexture, v_uv + vec2( 0.0, -ts.y)).rgb * 2.0;
        color += texture(u_sourceTexture, v_uv + vec2( 0.0,  ts.y)).rgb * 2.0;
        color += texture(u_sourceTexture, v_uv + vec2(-ts.x, -ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2( ts.x, -ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2(-ts.x,  ts.y)).rgb;
        color += texture(u_sourceTexture, v_uv + vec2( ts.x,  ts.y)).rgb;
        fragColor = vec4(color / 12.0, 1.0);
    }
}
```

### Fragment Shader: Tone Map

**tonemap.frag** — ACES filmic tone mapping with gamma correction.

```glsl
precision highp float;

uniform sampler2D u_hdrColor;
uniform sampler2D u_bloomTexture;
uniform float u_bloomIntensity;
uniform float u_exposure;

in vec2 v_uv;

layout(location = 0) out vec4 fragColor;

// ACES filmic tone mapping (Narkowicz 2015 fit)
vec3 acesToneMap(vec3 x) {
    float a = 2.51;
    float b = 0.03;
    float c = 2.43;
    float d = 0.59;
    float e = 0.14;
    return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}

void main() {
    vec3 hdr = texture(u_hdrColor, v_uv).rgb;
    vec3 bloom = texture(u_bloomTexture, v_uv).rgb;

    // Add bloom
    hdr += bloom * u_bloomIntensity;

    // Exposure
    hdr *= u_exposure;

    // Tone map
    vec3 ldr = acesToneMap(hdr);

    // Gamma correction (linear → sRGB)
    ldr = pow(ldr, vec3(1.0 / 2.2));

    fragColor = vec4(ldr, 1.0);
}
```

### Vertex Shader: Fullscreen Quad

**fullscreen.vert** — Used by all post-processing and composite shaders. Generates a fullscreen triangle from vertex ID alone (no vertex buffer needed).

```glsl
out vec2 v_uv;

void main() {
    // Fullscreen triangle trick — 3 vertices, no VBO
    v_uv = vec2((gl_VertexID << 1) & 2, gl_VertexID & 2);
    gl_Position = vec4(v_uv * 2.0 - 1.0, 0.0, 1.0);
}
```

### Fragment Shader: Blit

**blit.frag** — Simple texture-to-screen copy.

```glsl
precision highp float;

uniform sampler2D u_source;

in vec2 v_uv;

layout(location = 0) out vec4 fragColor;

void main() {
    fragColor = texture(u_source, v_uv);
}
```

---

## Main Render Loop

The following TypeScript pseudocode shows the complete per-frame render function. It ties together all of the systems described above: frustum culling, command generation, sorting, multi-pass rendering, and post-processing.

```typescript
type RendererConfig = {
  msaa: boolean
  msaaSamples: number
  bloom: boolean
  bloomThreshold: number
  bloomIntensity: number
  bloomLevels: number
  exposure: number
  toneMapping: 'ACES' | 'REINHARD' | 'LINEAR'
  depthPrepass: boolean
}

type RendererState = {
  device: GALDevice
  config: RendererConfig
  renderTargets: RenderTargetCache
  commandPool: RenderCommand[]
  commandPoolAux: RenderCommand[]
  commandCount: number
  commandsDirty: boolean
  opaqueCount: number
  transparentCount: number
  dynamicUBO: DynamicUBO
  frameUBO: GALBuffer
  frustum: Frustum
  shaders: ShaderCache
  staticBundle: GPURenderBundle | null
  staticBundleDirty: boolean
}

const render = (state: RendererState, scene: Scene, camera: Camera): void => {
  const { device, config, renderTargets } = state
  const width = device.canvasWidth
  const height = device.canvasHeight

  // ---------------------------------------------------------------
  // Phase 1: Update transforms and cull
  // ---------------------------------------------------------------

  // Walk the scene graph and update world matrices for dirty nodes
  scene.updateWorldTransforms()

  // Build view-projection matrix and extract frustum planes
  const viewMatrix = camera.getViewMatrix()        // Z-up right-handed
  const projMatrix = camera.getProjectionMatrix()
  const vpMatrix = mat4Multiply(projMatrix, viewMatrix)
  extractFrustum(vpMatrix, state.frustum)

  // ---------------------------------------------------------------
  // Phase 2: Generate and sort commands (only if dirty)
  // ---------------------------------------------------------------

  if (state.commandsDirty || camera.frustumChanged) {
    state.commandCount = 0
    state.opaqueCount = 0
    state.transparentCount = 0

    const near = camera.near
    const far = camera.far

    // Iterate all mesh nodes in the scene
    const meshNodes = scene.getMeshNodes()
    for (let i = 0; i < meshNodes.length; i++) {
      const node = meshNodes[i]
      const aabb = node.worldAABB

      // Frustum cull — 6-plane test with early-out
      if (!testAABBFrustum(
        state.frustum,
        aabb.minX, aabb.minY, aabb.minZ,
        aabb.maxX, aabb.maxY, aabb.maxZ
      )) {
        continue  // culled — no command generated
      }

      // Compute view-space depth for sort key
      const centerX = (aabb.minX + aabb.maxX) * 0.5
      const centerY = (aabb.minY + aabb.maxY) * 0.5
      const centerZ = (aabb.minZ + aabb.maxZ) * 0.5
      const viewDepth = -(
        viewMatrix[2] * centerX +
        viewMatrix[6] * centerY +
        viewMatrix[10] * centerZ +
        viewMatrix[14]
      )

      const cmd = state.commandPool[state.commandCount]
      cmd.mesh = node
      cmd.material = node.materialId
      cmd.geometry = node.geometryId
      cmd.worldMatrix = node.worldMatrix
      cmd.boneMatrices = node.boneMatrices
      cmd.materialIndex = node.materialIndex

      if (node.isTransparent) {
        cmd.sortKey = buildSortKeyTransparent(
          node.pipelineId, node.materialId, viewDepth, near, far
        )
        state.transparentCount++
      } else {
        cmd.sortKey = buildSortKeyOpaque(
          node.pipelineId, node.materialId, viewDepth, near, far
        )
        state.opaqueCount++
      }

      state.commandCount++
    }

    // Partition: opaque commands first, then transparent
    // (Simple approach: sort all, opaque pipelines have lower IDs)
    radixSort64(state.commandPool, state.commandPoolAux, state.commandCount)

    state.commandsDirty = false
    state.staticBundleDirty = true
  }

  // ---------------------------------------------------------------
  // Phase 3: Ensure render targets are allocated
  // ---------------------------------------------------------------

  ensureRenderTargets(renderTargets, device, width, height, scene, config)

  // ---------------------------------------------------------------
  // Phase 4: Upload per-frame uniforms
  // ---------------------------------------------------------------

  state.dynamicUBO.offset = 0  // reset dynamic UBO for this frame

  const frameData = buildFrameUniformData(camera, scene, config)
  device.writeBuffer(state.frameUBO, 0, frameData)

  // ---------------------------------------------------------------
  // Phase 5: Begin GPU command encoding
  // ---------------------------------------------------------------

  const encoder = device.beginCommandEncoder()

  // ---------------------------------------------------------------
  // Pass 0: Shadow Maps
  // ---------------------------------------------------------------

  if (renderTargets.shadowCascades && scene.hasShadowCastingLight()) {
    const light = scene.getDirectionalLight()
    const cascadeMatrices = computeCSMCascades(camera, light, 3)

    for (let cascade = 0; cascade < 3; cascade++) {
      const shadowPass = encoder.beginRenderPass({
        colorAttachments: [],
        depthStencilAttachment: {
          view: renderTargets.shadowCascades[cascade].view,
          depthLoadOp: 'clear',
          depthStoreOp: 'store',
          depthClearValue: 1.0,
        },
      })

      // Render all shadow-casting opaque objects
      const shadowUniforms = { lightViewProjection: cascadeMatrices[cascade] }
      device.writeBuffer(state.frameUBO, 0, shadowUniforms)

      for (let i = 0; i < state.opaqueCount; i++) {
        const cmd = state.commandPool[i]
        if (!cmd.mesh.castShadow) continue

        shadowPass.setPipeline(device.getPipeline(state.shaders.shadowPipeline))

        const offset = dynamicUBOWrite(
          state.dynamicUBO,
          cmd.worldMatrix,
          cmd.materialIndex,
          cmd.boneMatrices
        )
        shadowPass.setBindGroup(2, state.dynamicUBO.bindGroup, [offset])
        shadowPass.setVertexBuffer(0, device.getVertexBuffer(cmd.geometry))
        shadowPass.setIndexBuffer(device.getIndexBuffer(cmd.geometry))

        const geo = device.getGeometryInfo(cmd.geometry)
        shadowPass.drawIndexed(geo.indexCount, 1, geo.indexOffset, 0, 0)
      }

      shadowPass.end()
    }

    // Re-upload frame uniforms with camera matrices (shadow pass overwrote them)
    device.writeBuffer(state.frameUBO, 0, frameData)
  }

  // ---------------------------------------------------------------
  // Pass 1: Opaque Pass (4x MSAA)
  // ---------------------------------------------------------------

  const opaquePass = encoder.beginRenderPass({
    colorAttachments: [{
      view: renderTargets.opaqueColor!.view,
      loadOp: 'clear',
      storeOp: 'store',
      clearValue: { r: 0.0, g: 0.0, b: 0.0, a: 1.0 },
    }],
    depthStencilAttachment: {
      view: renderTargets.opaqueDepth!.view,
      depthLoadOp: 'clear',
      depthStoreOp: 'store',
      depthClearValue: 1.0,
    },
  })

  opaquePass.setBindGroup(0, device.getFrameBindGroup(state.frameUBO))

  executeCommands(
    state.commandPool,
    state.opaqueCount,
    device,
    opaquePass,
    state.frameUBO,
    state.dynamicUBO
  )

  opaquePass.end()

  // ---------------------------------------------------------------
  // Pass 2: Transparent Pass (OIT Accumulation)
  // ---------------------------------------------------------------

  if (state.transparentCount > 0 && renderTargets.oitAccumulation) {
    const oitPass = encoder.beginRenderPass({
      colorAttachments: [
        {
          view: renderTargets.oitAccumulation.view,
          loadOp: 'clear',
          storeOp: 'store',
          clearValue: { r: 0.0, g: 0.0, b: 0.0, a: 0.0 },
        },
        {
          view: renderTargets.oitRevealage!.view,
          loadOp: 'clear',
          storeOp: 'store',
          clearValue: { r: 1.0, g: 0.0, b: 0.0, a: 0.0 },
        },
      ],
      depthStencilAttachment: {
        view: renderTargets.opaqueDepth!.view,
        depthLoadOp: 'load',        // reuse opaque depth
        depthStoreOp: 'store',
        depthReadOnly: true,         // depth test but no write
      },
    })

    oitPass.setBindGroup(0, device.getFrameBindGroup(state.frameUBO))

    // Transparent commands start after opaque commands in the sorted array
    const transparentOffset = state.opaqueCount

    executeCommands(
      state.commandPool.slice(transparentOffset),
      state.transparentCount,
      device,
      oitPass,
      state.frameUBO,
      state.dynamicUBO
    )

    oitPass.end()

    // ---------------------------------------------------------------
    // Pass 3: OIT Composite
    // ---------------------------------------------------------------

    const compositePass = encoder.beginRenderPass({
      colorAttachments: [{
        view: renderTargets.opaqueColor!.view,
        loadOp: 'load',    // preserve opaque result
        storeOp: 'store',
      }],
    })

    compositePass.setPipeline(device.getPipeline(state.shaders.oitCompositePipeline))
    compositePass.setBindGroup(0, device.createBindGroup({
      texture0: renderTargets.oitAccumulation,
      texture1: renderTargets.oitRevealage!,
    }))
    compositePass.draw(3)  // fullscreen triangle
    compositePass.end()
  }

  // ---------------------------------------------------------------
  // Pass 4: Post-Processing
  // ---------------------------------------------------------------

  if (config.bloom && renderTargets.bloomChain) {
    const bloomChain = renderTargets.bloomChain

    // Bloom extract: scene color → bloom level 0
    const extractPass = encoder.beginRenderPass({
      colorAttachments: [{
        view: bloomChain[0].view,
        loadOp: 'clear',
        storeOp: 'store',
      }],
    })

    extractPass.setPipeline(device.getPipeline(state.shaders.bloomExtractPipeline))
    extractPass.setBindGroup(0, device.createBindGroup({
      sceneColor: renderTargets.opaqueColor!,
      threshold: config.bloomThreshold,
    }))
    extractPass.draw(3)
    extractPass.end()

    // Progressive downscale
    for (let i = 1; i < config.bloomLevels; i++) {
      const downPass = encoder.beginRenderPass({
        colorAttachments: [{
          view: bloomChain[i].view,
          loadOp: 'clear',
          storeOp: 'store',
        }],
      })

      downPass.setPipeline(device.getPipeline(state.shaders.bloomBlurPipeline))
      downPass.setBindGroup(0, device.createBindGroup({
        sourceTexture: bloomChain[i - 1],
        texelSize: [1.0 / bloomChain[i - 1].width, 1.0 / bloomChain[i - 1].height],
        direction: 0.0,  // downsample
      }))
      downPass.draw(3)
      downPass.end()
    }

    // Progressive upscale with additive blend
    for (let i = config.bloomLevels - 2; i >= 0; i--) {
      const upPass = encoder.beginRenderPass({
        colorAttachments: [{
          view: bloomChain[i].view,
          loadOp: 'load',     // additive onto existing content
          storeOp: 'store',
        }],
      })

      upPass.setPipeline(device.getPipeline(state.shaders.bloomBlurPipeline))
      upPass.setBindGroup(0, device.createBindGroup({
        sourceTexture: bloomChain[i + 1],
        texelSize: [1.0 / bloomChain[i + 1].width, 1.0 / bloomChain[i + 1].height],
        direction: 1.0,  // upsample
      }))
      upPass.draw(3)
      upPass.end()
    }
  }

  // Tone mapping
  const toneMapPass = encoder.beginRenderPass({
    colorAttachments: [{
      view: renderTargets.opaqueColor!.view,
      loadOp: 'load',
      storeOp: 'store',
    }],
  })

  toneMapPass.setPipeline(device.getPipeline(state.shaders.toneMapPipeline))
  toneMapPass.setBindGroup(0, device.createBindGroup({
    hdrColor: renderTargets.opaqueColor!,
    bloomTexture: config.bloom ? renderTargets.bloomChain![0] : device.whiteTexture,
    bloomIntensity: config.bloom ? config.bloomIntensity : 0.0,
    exposure: config.exposure,
  }))
  toneMapPass.draw(3)
  toneMapPass.end()

  // ---------------------------------------------------------------
  // Pass 5: MSAA Resolve + Final Output
  // ---------------------------------------------------------------

  if (config.msaa) {
    encoder.resolveTexture(
      renderTargets.opaqueColor!,   // MSAA source
      renderTargets.resolveOutput!   // 1x destination
    )

    // Blit resolved result to canvas
    const finalPass = encoder.beginRenderPass({
      colorAttachments: [{
        view: device.getCurrentCanvasView(),
        loadOp: 'clear',
        storeOp: 'store',
      }],
    })

    finalPass.setPipeline(device.getPipeline(state.shaders.blitPipeline))
    finalPass.setBindGroup(0, device.createBindGroup({
      source: renderTargets.resolveOutput!,
    }))
    finalPass.draw(3)
    finalPass.end()
  } else {
    // No MSAA — blit directly from opaque color to canvas
    const finalPass = encoder.beginRenderPass({
      colorAttachments: [{
        view: device.getCurrentCanvasView(),
        loadOp: 'clear',
        storeOp: 'store',
      }],
    })

    finalPass.setPipeline(device.getPipeline(state.shaders.blitPipeline))
    finalPass.setBindGroup(0, device.createBindGroup({
      source: renderTargets.opaqueColor!,
    }))
    finalPass.draw(3)
    finalPass.end()
  }

  // ---------------------------------------------------------------
  // Submit
  // ---------------------------------------------------------------

  device.submitCommands(encoder.finish())
}
```

### Frame Lifecycle Summary

```
1. scene.updateWorldTransforms()     — walk dirty flags, update matrices
2. extractFrustum()                  — build 6 planes from VP matrix
3. frustum cull + command gen        — O(n) scan, ~20-40% survive
4. radixSort64()                     — O(n) sort on 64-bit key
5. ensureRenderTargets()             — lazy alloc if needed
6. upload frame UBO                  — camera, lights, shadows
7. Pass 0: shadow maps              — 3 cascades, depth only
8. Pass 1: opaque                   — sorted front-to-back, 4x MSAA
9. Pass 2: OIT accumulation         — transparent, additive blend to float MRT
10. Pass 3: OIT composite           — fullscreen quad, alpha blend onto opaque
11. Pass 4: post-processing         — bloom chain + tone mapping
12. Pass 5: MSAA resolve + blit     — resolve 4x → 1x, copy to canvas
13. device.submitCommands()          — single command buffer submission
```
