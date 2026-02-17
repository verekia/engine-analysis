# RENDERER.md - Cross-Engine Comparison

This document compares the rendering pipeline and GPU abstraction strategies across all 9 engine implementations.

---

## Universal Agreement

### WebGPU-First Abstraction
All 9 implementations model their GPU abstraction after WebGPU, not WebGL2:

**Rationale** (Universal):
- WebGPU is the better API (explicit, stateless, modern)
- WebGPU is the future; WebGL2 is the compatibility layer
- Attempting WebGL2-first would cripple WebGPU performance
- Translation cost (WebGPU→WebGL2) is acceptable

**Key Concepts** (All 9):
- **Immutable pipelines**: Full render state baked at creation
- **Explicit bind groups**: Resources grouped by update frequency
- **Command recording**: Explicit begin/end passes, even on WebGL2

### Backend Selection Pattern
All 9 use identical fallback logic:

```typescript
if (navigator.gpu && !forceWebGL2) {
  adapter = await navigator.gpu.requestAdapter()
  if (adapter) {
    device = await adapter.requestDevice()
    return new WebGPUBackend(device, canvas)
  }
}
// Fallback
gl = canvas.getContext('webgl2')
return new WebGL2Backend(gl)
```

### Core Render Passes (All 9)
Every implementation uses this multi-pass pipeline:

1. **Shadow Pass** (CSM depth maps)
2. **Opaque Pass** (sorted, MSAA)
3. **Transparent Pass** (OIT accumulation)
4. **MSAA Resolve**
5. **OIT Composite**
6. **Bloom** (threshold → downsample → upsample)
7. **Final Blit** (tone mapping → screen)

### Bind Group Layout (All 9)
Three bind groups by update frequency:

| Slot | Name | Update | Contents |
|------|------|--------|----------|
| 0 | Per-frame | Once | Camera VP, lights, shadow matrices |
| 1 | Per-material | Per material | Textures, material params |
| 2 | Per-object | Per object | Model matrix, bone matrices |

### MSAA Strategy (All 9)
- **Sample count**: 4x by default (configurable 1x, 2x, 4x)
- **Resolve timing**: Before post-processing
- **WebGPU**: Native multisample texture with auto-resolve
- **WebGL2**: Renderbuffer with `gl.blitFramebuffer`

---

## Key Variations

### 1. Abstraction Layer Naming

| Term | Implementations | Notes |
|------|----------------|-------|
| **GAL** (GPU Abstraction Layer) | Caracal, Fennec, Lynx, Mantis (4) | Most common |
| **HAL** (Hardware Abstraction Layer) | Rabbit, Wren (2) | Hardware focus |
| **Device** | Bonobo, Hyena, Shark (3) | Direct naming |

**Consensus**: Name varies, but interface is ~95% identical across all.

### 2. WebGL2 State Management

#### Full State Cache (Majority)
**Who**: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren (8 engines)

**Pattern**:
```typescript
interface StateCache {
  currentProgram: WebGLProgram | null
  currentVAO: WebGLVertexArrayObject | null
  currentFBO: WebGLFramebuffer | null
  boundTextures: (WebGLTexture | null)[]
  boundUBOs: (WebGLBuffer | null)[]
  depthWrite: boolean
  depthFunc: number
  blendEnabled: boolean
  cullFace: number
  viewport: [number, number, number, number]
}

// Before every state change:
if (state.currentProgram !== newProgram) {
  gl.useProgram(newProgram)
  state.currentProgram = newProgram
}
```

**Benefits**:
- Eliminates 40-60% of redundant GL calls
- Essential for draw call sorting performance
- Preserves state across render passes

**Implementation Notes**:
- Some track per-texture-unit state separately
- Some track per-attachment blend state
- All invalidate state after framebuffer changes

#### Minimal State Tracking
**Who**: Bonobo (1 engine)

**Pattern**:
- Tracks only critical state (program, VAO)
- Relies more on sorting than state caching
- Simpler but potentially more GL calls

### 3. Draw Call Sorting Strategy

#### Sort Key Bit Layout

All implementations use 64-bit keys, but pack them differently:

**Bonobo**:
```
[pipeline 8b] [material 8b] [geometry 8b] [unused 40b]
```

**Caracal**:
```
[shader 8b] [material 8b] [geometry 8b] [depth 8b]
```

**Fennec, Hyena, Lynx, Mantis**:
```
[pipeline 16b] [material 16b] [depth 32b]
```

**Rabbit, Shark**:
```
[pipeline 8b] [material 16b] [texture 16b] [depth 24b]
```

**Wren**:
```
[layer 2b] [pass 2b] [transparent 1b] [pipeline 11b] [material 16b] [depth 32b]
```

**Consensus**: Pipeline first (most expensive state change), then material, then depth for opaque.

#### Sort Algorithm

**Radix Sort** (5 implementations):
- **Who**: Mantis, Shark, Wren, Caracal, Fennec
- **Complexity**: O(n)
- **Pattern**: Two passes (high word, low word) or full 64-bit
- **Benefits**: Stable, predictable, no worst-case

**Standard Sort** (4 implementations):
- **Who**: Bonobo, Hyena, Lynx, Rabbit
- **Complexity**: O(n log n) average
- **Pattern**: `Array.sort()` or Timsort
- **Benefits**: Simpler, built-in

**Performance**: For 2000 draws, both complete in < 0.3ms

### 4. WebGPU Render Bundles

#### Aggressive Use (Primary Optimization)
**Who**: Hyena, Fennec, Caracal (3 engines emphasize)

**Pattern**:
```typescript
// Build bundle for all static geometry
const staticBundle = recordRenderBundle(staticObjects)

// Each frame: just replay
passEncoder.executeBundles([staticBundle])
// Then draw dynamic objects normally
```

**Invalidation**:
- When objects added/removed
- When materials change
- When visibility changes (some implementations)

**Benefits**:
- Near-zero JS overhead for static scenes
- Pre-validation of draw commands
- GPU-side caching possible

#### Conservative Use (Available but Not Emphasized)
**Who**: Lynx, Mantis, Rabbit, Shark, Wren, Bonobo (6 engines)

**Pattern**:
- Render bundles mentioned as WebGPU advantage
- Not primary optimization strategy
- May use for specific cases (UI, static backgrounds)

### 5. Uniform Upload Strategy

#### Ring Buffer (Triple Buffered)
**Who**: Mantis, Shark, Wren (3 engines explicitly)

**Details**:
```typescript
const RING_SIZE = 4 * 1024 * 1024  // 4MB
const FRAMES = 3                   // Triple buffering

// Each frame writes to new region
const offset = (frameIndex % FRAMES) * (RING_SIZE / FRAMES)
writeBuffer(uniformBuffer, offset, frameUniforms)
```

**Benefits**:
- Zero per-object allocation
- GPU never stalls on previous frames
- Amortized cost of large buffer

**Trade-offs**:
- Memory overhead (3x minimum usage)
- Must track in-flight frames

#### Per-Frame Upload
**Who**: Bonobo, Caracal, Fennec, Hyena, Lynx, Rabbit (6 engines)

**Details**:
```typescript
// Upload per-frame UBO once
writeBuffer(cameraUBO, cameraData)
writeBuffer(lightsUBO, lightsData)

// Upload per-object UBO per draw
writeBuffer(objectUBO, objectData)
setBindGroup(2, objectBindGroup, [0])
```

**Benefits**:
- Simpler to reason about
- No over-allocation
- Fine-grained updates

**Trade-offs**:
- More upload calls
- Potential stalls if not careful

#### Dynamic Offsets (Universal for Per-Object)
**Who**: All 9 implementations

**Pattern**:
```typescript
// WebGPU
setBindGroup(2, sharedObjectBindGroup, [objectByteOffset])

// WebGL2
gl.bindBufferRange(GL.UNIFORM_BUFFER, 2, buffer, objectByteOffset, objectSize)
```

**Benefits**:
- Single bind group, different offset
- Works identically in both backends
- Industry standard

### 6. Shader Management

#### Dual-Source (WGSL + GLSL)
**Who**: All 9 implementations

**Approaches**:

**Separate Files** (Most):
```
shaders/
  lambert.vert.wgsl
  lambert.frag.wgsl
  lambert.vert.glsl
  lambert.frag.glsl
```

**Template Generation** (Some):
```
shaders/
  lambert.vert.template
  lambert.frag.template
  + generator that outputs WGSL and GLSL
```

**Feature Flags** (All):
```glsl
#ifdef HAS_VERTEX_COLORS
  in vec4 a_color;
#endif

#ifdef HAS_SKINNING
  in vec4 a_joints;
  in vec4 a_weights;
#endif
```

**Shader Caching** (All):
```typescript
const shaderCache = new Map<featureBitMask, CompiledShader>()
```

**Consensus**: Transpiling WGSL→GLSL at runtime is fragile; dual-source is safer.

### 7. Pipeline Caching

#### Hash-Based Caching (All 9)
Every implementation caches pipelines by a hash of their descriptor:

```typescript
const pipelineKey = hash({
  shader: shaderVariant,
  vertexLayout,
  blendMode,
  depthWrite,
  cullMode,
  sampleCount,
})

const getPipeline = (desc) => {
  const key = hash(desc)
  if (!pipelineCache.has(key)) {
    pipelineCache.set(key, device.createPipeline(desc))
  }
  return pipelineCache.get(key)
}
```

**Cache Size**:
- Typically 10-50 pipelines for simple scenes
- Up to 200-300 for complex games
- LRU eviction in some implementations

### 8. Render Target Management

#### Pre-Allocated at Init (All 9)
All implementations create render targets at initialization and recreate on resize:

**Common Targets**:
| Target | Format | MSAA | Size | Purpose |
|--------|--------|------|------|---------|
| Color | RGBA8 | 4x | Canvas | Main scene |
| Depth | Depth24Plus | 4x | Canvas | Z-buffer |
| Emissive | RGBA8 | 4x | Canvas | Bloom input |
| OIT Accum | RGBA16F | 1x | Canvas | Transparency |
| OIT Reveal | R8 | 1x | Canvas | Transparency |
| Shadow Atlas | Depth24Plus | 1x | 2048-3072 | CSM |
| Bloom Mips | RGBA8 | 1x | Halving | Bloom chain |
| Resolved | RGBA8 | 1x | Canvas | Post-MSAA |

**Resize Strategy** (Universal):
```typescript
onResize = (width, height) => {
  destroyRenderTargets()
  recreateRenderTargets(width, height)
  camera.aspect = width / height
  camera.updateProjection()
}
```

---

## Implementation Breakdown

### By State Caching Strategy

| Strategy | Count | Implementations |
|----------|-------|----------------|
| Full state cache | 8 | Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren |
| Minimal tracking | 1 | Bonobo |

### By Sort Algorithm

| Algorithm | Count | Implementations |
|-----------|-------|----------------|
| Radix sort | 5 | Caracal, Fennec, Mantis, Shark, Wren |
| Standard sort | 4 | Bonobo, Hyena, Lynx, Rabbit |

### By Uniform Strategy

| Strategy | Count | Implementations |
|----------|-------|----------------|
| Ring buffer | 3 | Mantis, Shark, Wren |
| Per-frame upload | 6 | Bonobo, Caracal, Fennec, Hyena, Lynx, Rabbit |
| Dynamic offsets | 9 | All (for per-object) |

### By Render Bundle Usage

| Usage Level | Count | Implementations |
|-------------|-------|----------------|
| Primary optimization | 3 | Caracal, Fennec, Hyena |
| Available feature | 6 | Bonobo, Lynx, Mantis, Rabbit, Shark, Wren |

---

## GPU Abstraction Interface Comparison

### Common Interface (95% Overlap)

All implementations provide nearly identical abstractions:

```typescript
// Resource creation
createBuffer(desc: BufferDescriptor): Buffer
createTexture(desc: TextureDescriptor): Texture
createSampler(desc: SamplerDescriptor): Sampler
createShaderModule(desc: ShaderDescriptor): ShaderModule
createPipeline(desc: PipelineDescriptor): Pipeline
createBindGroup(desc: BindGroupDescriptor): BindGroup

// Command recording
beginFrame(): CommandEncoder
beginRenderPass(desc: RenderPassDescriptor): RenderPass
submit(commands): void

// Render pass operations
setPipeline(pipeline): void
setBindGroup(index, group, offsets?): void
setVertexBuffer(slot, buffer, offset?): void
setIndexBuffer(buffer, format): void
drawIndexed(count, instances?, first?): void
```

### Terminology Mapping

| WebGPU Concept | Abstraction Names | Count |
|----------------|-------------------|-------|
| GPUDevice | Device, GALDevice, HalDevice | All |
| GPUBuffer | Buffer, GALBuffer, HalBuffer, GpuBuffer | All |
| GPUTexture | Texture, GALTexture, HalTexture, GpuTexture | All |
| GPURenderPipeline | Pipeline, GALPipeline, HalPipeline, GpuPipeline | All |
| GPUBindGroup | BindGroup, GALBindGroup, HalBindGroup, GpuBindGroup | All |
| GPUCommandEncoder | CommandEncoder, FrameEncoder, Encoder | All |
| GPURenderPassEncoder | RenderPass, RenderPassEncoder, PassEncoder | All |

**Observation**: Despite naming variations, the structure is remarkably consistent.

---

## WebGL2 Translation Strategies

### Pipeline → GL State Bundle (All 9)

All implementations map WebGPU-style pipelines to a WebGL state bundle:

```typescript
interface WebGL2Pipeline {
  program: WebGLProgram
  state: {
    blend: { enabled, srcRGB, dstRGB, srcA, dstA, op }
    depth: { test, write, func }
    cull: { enabled, face, frontFace }
    viewport?: [x, y, w, h]
  }
}

// Apply state on bind
const applyPipeline = (pipeline: WebGL2Pipeline) => {
  gl.useProgram(pipeline.program)
  applyBlendState(pipeline.state.blend)
  applyDepthState(pipeline.state.depth)
  applyCullState(pipeline.state.cull)
}
```

### BindGroup → UBO + Texture Bindings (All 9)

```typescript
interface WebGL2BindGroup {
  uniformBuffers: Array<{
    binding: number
    buffer: WebGLBuffer
    offset: number
    size: number
  }>
  textures: Array<{
    binding: number
    texture: WebGLTexture
    unit: number
  }>
  samplers: Array<{
    binding: number
    sampler: WebGLSampler
  }>
}

// Apply on bind
const applyBindGroup = (index: number, group: WebGL2BindGroup, offsets?: number[]) => {
  group.uniformBuffers.forEach((ub, i) => {
    const offset = offsets?.[i] ?? ub.offset
    gl.bindBufferRange(GL.UNIFORM_BUFFER, ub.binding, ub.buffer, offset, ub.size)
  })
  group.textures.forEach(t => {
    gl.activeTexture(GL.TEXTURE0 + t.unit)
    gl.bindTexture(GL.TEXTURE_2D, t.texture)
  })
  group.samplers.forEach(s => {
    gl.bindSampler(s.binding, s.sampler)
  })
}
```

### VAO Management (All 9)

Every geometry + pipeline combination gets a cached VAO:

```typescript
const vaoCache = new Map<string, WebGLVertexArrayObject>()

const getOrCreateVAO = (geometry: Geometry, pipeline: Pipeline): WebGLVertexArrayObject => {
  const key = `${geometry.id}:${pipeline.id}`
  if (!vaoCache.has(key)) {
    const vao = gl.createVertexArray()
    gl.bindVertexArray(vao)
    setupVertexAttributes(geometry, pipeline.vertexLayout)
    vaoCache.set(key, vao)
  }
  return vaoCache.get(key)
}
```

---

## Performance Optimizations Summary

### Universal (All 9 Implementations)

1. **Draw call sorting** by composite key (pipeline → material → depth)
2. **Frustum culling** before building render list
3. **State caching** (at least current program/VAO)
4. **Dynamic offsets** for per-object uniforms
5. **Pre-allocated typed arrays** for all per-frame data
6. **Zero allocations** in render loop
7. **MSAA** resolved before post-processing
8. **Shader variant caching** by feature flags

### Common (5+ Implementations)

1. **Radix sort** for O(n) draw call ordering
2. **Full state cache** tracking all GL state
3. **Render bundles** for static geometry (WebGPU)
4. **VAO caching** per geometry+pipeline pair
5. **UBO pooling** for dynamic uniforms

### Specialized (1-3 Implementations)

1. **Ring buffer** for per-object uniforms (Mantis, Shark, Wren)
2. **Persistent render lists** (Caracal, Lynx, Mantis, Shark)
3. **Multi-draw extension** batching (Caracal mentions explicitly)
4. **Render bundle emphasis** as primary strategy (Caracal, Fennec, Hyena)
5. **Compute culling** on WebGPU (Wren mentions)

---

## Frame Performance Budget (Typical)

All implementations target 16.6ms per frame (60 fps). Breakdown:

```
Scene graph update:       0.3-0.5ms  (dirty nodes only)
Frustum culling:          0.2-0.4ms  (AABB tests)
Draw call sorting:        0.1-0.3ms  (radix or timsort)
Uniform uploads:          0.2-0.5ms  (per-frame + per-object)
Shadow pass (3 CSM):      2.0-3.0ms  (GPU, depth-only)
Main pass (2000 draws):   3.0-5.0ms  (GPU, sorted)
MSAA resolve:             0.5-1.0ms  (GPU)
OIT composite:            0.5-1.0ms  (GPU, fullscreen)
Bloom:                    1.0-2.0ms  (GPU, progressive)
JS overhead total:        1.5-2.5ms  (CPU work)
GPU work total:           7.0-12.0ms (rendering)
──────────────────────────────────
Total frame time:         8.5-14.5ms
Headroom:                 2.1-8.1ms
```

---

## Recommendations for Cherry-Picking

### For Maximum Draw Call Throughput
1. **Radix sort** (Mantis, Shark, Wren) - O(n) vs O(n log n)
2. **Ring buffer uniforms** (Mantis, Shark, Wren) - Zero per-object allocation
3. **Full state cache** (8 engines) - Eliminate redundant GL calls
4. **Render bundles** (All) - Near-zero cost for static scenes

### For Simplest Implementation
1. **Standard sort** (4 engines) - Built-in, easier to debug
2. **Per-frame upload** (6 engines) - Simpler mental model
3. **WebGPU-modeled abstraction** (All) - Clean, stateless interface
4. **Dual-source shaders** (All) - No transpilation complexity

### For Best WebGL2 Performance
1. **Full state cache** (8 engines) - Critical for WebGL2
2. **VAO caching** (All) - Eliminate attribute setup
3. **UBO for per-frame data** (All) - Fewer uniform calls
4. **Multi-draw extension** (Caracal) - Batch compatible draws

### For Best WebGPU Performance
1. **Render bundles** (All) - Primary WebGPU advantage
2. **Bind group hierarchy** (All) - Minimize rebinds
3. **Dynamic offsets** (All) - Fast per-object switching
4. **Pipeline caching** (All) - Amortize expensive creation

### For Most Maintainable
1. **WebGPU-modeled abstraction** (All) - Future-proof
2. **Separate shader files** (Most) - Clear organization
3. **Feature flag variants** (All) - Clean shader code
4. **Hash-based caching** (All) - Automatic deduplication

### For Best Mobile Performance
1. **Triple-buffered ring buffer** (3 engines) - Avoid stalls
2. **Aggressive frustum culling** (All) - Reduce draw count
3. **MSAA resolve before post** (All) - Tile-based GPU friendly
4. **Minimal post-processing** - Bandwidth matters on mobile
