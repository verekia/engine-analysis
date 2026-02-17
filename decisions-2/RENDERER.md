# RENDERER.md - Decisions

## Decision: WebGPU-Modeled GAL with Full State Cache and Radix Sort

### GPU Abstraction Layer

**Name**: GAL (GPU Abstraction Layer) - most common across implementations (4/9)

**Interface** (95% identical across all 9 implementations):

```typescript
interface GALDevice {
  readonly backend: 'webgpu' | 'webgl2'

  createBuffer(desc: BufferDescriptor): GALBuffer
  createTexture(desc: TextureDescriptor): GALTexture
  createSampler(desc: SamplerDescriptor): GALSampler
  createShaderModule(desc: ShaderDescriptor): GALShaderModule
  createPipeline(desc: PipelineDescriptor): GALPipeline
  createBindGroup(desc: BindGroupDescriptor): GALBindGroup

  beginFrame(): GALCommandEncoder
  beginRenderPass(desc: RenderPassDescriptor): GALRenderPass
  submit(): void
}
```

### Backend Selection

Universal pattern across all 9:

```typescript
if (navigator.gpu && !forceWebGL2) {
  adapter = await navigator.gpu.requestAdapter()
  if (adapter) {
    device = await adapter.requestDevice()
    return new WebGPUBackend(device, canvas)
  }
}
gl = canvas.getContext('webgl2')
return new WebGL2Backend(gl)
```

### Core Render Passes

All 9 agree on this multi-pass pipeline:

1. **Shadow Pass** - CSM depth maps (3 cascades)
2. **Opaque Pass** - sorted front-to-back, MSAA enabled
3. **Transparent Pass** - OIT accumulation (WBOIT)
4. **MSAA Resolve**
5. **OIT Composite**
6. **Bloom** - threshold, progressive downsample/upsample
7. **Final Blit** - tone mapping to screen

### Draw Call Sorting: Radix Sort with 64-bit Keys

**Chosen**: Radix sort (5/9: Caracal, Fennec, Mantis, Shark, Wren)
**Rejected**: `Array.sort` (4/9: Bonobo, Hyena, Lynx, Rabbit) - simpler but O(n log n) vs O(n)

For 2000 draws both complete in <0.3ms, but radix sort is predictable, stable, and has no worst case. The implementation is straightforward: two passes on 32-bit words of the 64-bit key.

**Sort key layout** (Wren-inspired, most comprehensive):

```
[layer 2b] [transparent 1b] [pipeline 11b] [material 16b] [depth 32b]
```

Pipeline first (most expensive state change), then material, then depth. Opaque objects sort front-to-back (early-Z rejection), transparent objects sort back-to-front (for WBOIT weight function, though WBOIT doesn't strictly require it).

### WebGL2 State Management: Full State Cache

**Chosen**: Full state cache (8/9 implementations)
**Rejected**: Minimal tracking (Bonobo only)

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
}
```

Before every state change, check cache and skip if unchanged. Eliminates 40-60% of redundant GL calls. Essential for making sorted rendering effective on WebGL2.

### Render Bundles: Available But Not Primary Strategy

**Chosen**: Support render bundles when on WebGPU as an optimization for static geometry, but don't make it the primary architecture (6/9: conservative use)
**Rejected**: Render bundles as primary optimization (3/9: Caracal, Fennec, Hyena)

Render bundles help with static scenes but add invalidation complexity. The primary performance strategy is sort-based rendering with low per-draw overhead, which works on both backends. Render bundles are an optional WebGPU enhancement.

### Shader Management: Dual-Source WGSL + GLSL

**Chosen**: Ship both WGSL and GLSL shaders, no runtime transpilation (universal consensus)

```
shaders/
  lambert.vert.wgsl
  lambert.frag.wgsl
  lambert.vert.glsl
  lambert.frag.glsl
```

Feature flags via preprocessor defines:

```glsl
#ifdef HAS_VERTEX_COLORS
  in vec4 a_color;
#endif
#ifdef HAS_SKINNING
  in vec4 a_joints;
  in vec4 a_weights;
#endif
```

Shader variants cached by feature bitmask. Typically 10-30 unique variants in a scene.

### Pipeline Caching

Universal across all 9: hash-based pipeline cache.

```typescript
const pipelineCache = new Map<number, GALPipeline>()

const getOrCreatePipeline = (desc: PipelineDescriptor): GALPipeline => {
  const key = hashPipelineDescriptor(desc)
  let pipeline = pipelineCache.get(key)
  if (!pipeline) {
    pipeline = device.createPipeline(desc)
    pipelineCache.set(key, pipeline)
  }
  return pipeline
}
```

Typical scene: 10-50 unique pipelines.

### MSAA Strategy

Universal across all 9:
- **Sample count**: 4x by default (configurable 1x, 2x, 4x)
- **Resolve timing**: Before post-processing
- **WebGPU**: Native multisample texture with auto-resolve
- **WebGL2**: Renderbuffer with `gl.blitFramebuffer`

### Render Target Layout

Pre-allocated at initialization, recreated on resize:

| Target | Format | MSAA | Purpose |
|--------|--------|------|---------|
| Color | RGBA8 | 4x | Main scene |
| Depth | Depth24Plus | 4x | Z-buffer |
| Emissive | RGBA8 | 4x | Bloom input (MRT) |
| OIT Accum | RGBA16F | 1x | Transparency |
| OIT Reveal | R8 | 1x | Transparency |
| Shadow | Depth24Plus | 1x | CSM atlas/array |
| Bloom Mips | RGBA16F | 1x | Bloom chain |
| Resolved | RGBA8 | 1x | Post-MSAA |

### WebGL2 Translation Strategies

**Pipeline -> GL State Bundle**: Map immutable pipeline to `{program, blendState, depthState, cullState}`. Apply state on bind with cache checks.

**BindGroup -> UBO + Texture Bindings**: Map each bind group to a set of UBO ranges and texture unit bindings.

**VAO Caching**: Cache `WebGLVertexArrayObject` per geometry+pipeline pair to eliminate attribute setup overhead.

### Per-Draw Overhead Target

Target: **<2-3 microseconds per draw call** in JavaScript (Fennec/Hyena/Lynx/Mantis/Wren range)

For 2000 draws: ~4-6ms JS overhead, leaving ~10ms for GPU work within the 16.6ms frame budget.
