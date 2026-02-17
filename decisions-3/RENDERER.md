# RENDERER.md - Rendering Pipeline

## GPU Abstraction

**Decision: WebGPU-modeled Device interface** (universal agreement)

The GPU abstraction layer is called **Device** (not GAL or HAL - simpler naming, 3/9 implementations use this). Both WebGPU and WebGL2 backends implement the same interface.

### Backend Selection

```typescript
const device = await createDevice(canvas, {
  backend: 'auto' // 'auto' | 'webgpu' | 'webgl2'
})
```

Auto-detection: try `navigator.gpu.requestAdapter()` first, fall back to `canvas.getContext('webgl2')`.

### Core Interface

```typescript
interface Device {
  readonly backend: 'webgpu' | 'webgl2'

  createBuffer(desc): Buffer
  createTexture(desc): Texture
  createSampler(desc): Sampler
  createShaderModule(desc): ShaderModule
  createPipeline(desc): Pipeline
  createBindGroup(desc): BindGroup

  beginFrame(): FrameEncoder
  submit(): void
}
```

## Render Passes

Every frame executes these passes in order (universal agreement):

1. **Shadow Pass** - 3 CSM cascades, depth-only rendering
2. **Opaque Pass** - Sorted draw calls, MSAA, MRT (color + emissive)
3. **Transparent Pass** - OIT accumulation, depth read-only
4. **MSAA Resolve** - Resolve multisample to single-sample
5. **OIT Composite** - Blend transparent result over opaque
6. **Bloom** - Downsample chain, upsample chain, composite
7. **Final Blit** - Tone mapping, gamma correction, output to screen

## Bind Group Layout

**Decision: 3 bind groups by update frequency** (universal agreement)

| Slot | Name | Update Frequency | Contents |
|------|------|-----------------|----------|
| 0 | Per-frame | Once per frame | Camera VP matrix, light data, shadow matrices, cascade splits |
| 1 | Per-material | Per material switch | Material UBO, palette, textures, samplers |
| 2 | Per-object | Per draw call (dynamic offset) | World matrix, bone matrices |

Dynamic offsets on bind group 2 allow a single bind group with different offsets per object, minimizing rebind overhead.

## Draw Call Sorting

**Decision: 64-bit radix sort** (Mantis, Shark, Wren approach)

### Sort Key Layout

```
Bits 63-62: Layer (2b)      - opaque=0, transparent=1, overlay=2
Bits 61-60: Pass (2b)       - shadow=0, main=1, post=2
Bit  59:    Transparent (1b) - 0=opaque, 1=transparent
Bits 58-48: Pipeline ID (11b) - shader variant + blend/depth state
Bits 47-32: Material ID (16b) - material + texture binding
Bits 31-0:  Depth (32b)      - front-to-back (opaque) or back-to-front (transparent)
```

Pipeline switches are the most expensive state change, so they occupy the highest bits. Material/texture switches are moderate cost. Depth ordering enables early-Z rejection for opaque and correct visual ordering for transparent.

### Why Radix Sort Over Standard Sort

Radix sort is O(n) vs O(n log n). For 2000 draw calls, radix sort takes ~0.05ms (two passes over 32-bit words). Standard `Array.sort` takes ~0.1-0.2ms. The difference is small at 2000 draws, but radix sort is predictable (no worst case), stable, and the implementation is straightforward.

Implementation: two-pass radix sort on the high and low 32-bit words of the 64-bit key.

## WebGL2 Backend

### Full State Cache

**Decision: Track all GL state to eliminate redundant calls** (8/9 implementations agree)

```typescript
interface StateCache {
  program: WebGLProgram | null
  vao: WebGLVertexArrayObject | null
  fbo: WebGLFramebuffer | null
  boundTextures: (WebGLTexture | null)[]
  boundUBOs: (WebGLBuffer | null)[]
  depthWrite: boolean
  depthFunc: number
  blendEnabled: boolean
  blendSrc: number
  blendDst: number
  cullFace: number
  viewport: [number, number, number, number]
}
```

Before every GL call, check if the state actually changed. Skip if identical. This eliminates 40-60% of redundant GL calls when draws are sorted.

### VAO Caching

Cache one VAO per geometry+pipeline combination:

```typescript
const vaoKey = `${geometry.id}:${pipeline.id}`
```

Created lazily on first use, destroyed when geometry or pipeline is disposed.

### Pipeline as State Bundle

A WebGL2 "pipeline" is a frozen set of: program + blend state + depth state + cull state. Applying a pipeline sets all of these in one call (with state cache checks).

## WebGPU Backend

### Render Bundles

**Decision: Available for static geometry but not the primary optimization** (6/9 approach)

Render bundles pre-record draw commands for static geometry. They are useful for scenes where most objects don't change, but the engine should not depend on them for baseline performance. The sort-based renderer with dynamic offsets is the primary strategy.

Render bundles are invalidated when:
- Objects are added or removed from the scene
- Materials change
- Visibility changes

### Pipeline Caching

All pipelines are cached by a hash of their descriptor (shader variant, blend mode, depth state, cull mode, vertex layout, sample count). Typical scenes use 10-30 unique pipelines.

## Shader Management

**Decision: Dual-source WGSL + GLSL** (universal agreement)

Maintain separate WGSL and GLSL shader files. No runtime transpilation (fragile and slow).

### Shader Variants via Feature Flags

```glsl
#ifdef HAS_COLOR_TEXTURE
  uniform sampler2D u_colorMap;
#endif

#ifdef HAS_VERTEX_COLORS
  in vec4 a_color;
#endif

#ifdef HAS_MATERIAL_INDEX
  in float a_materialIndex;
#endif

#ifdef HAS_SKINNING
  in vec4 a_joints;
  in vec4 a_weights;
#endif
```

Variants are compiled lazily on first use and cached by feature bitmask. Typical scene: 10-30 unique variants.

### Shader Warm-up

**Decision: Pre-compile common variants during loading** (Caracal approach)

Unlike Three.js (which compiles on first render, causing frame spikes), pre-compile the most common shader variants during the asset loading phase. This prevents hitching during gameplay.

## MSAA

**Decision: 4x MSAA by default** (universal agreement)

- **WebGPU**: Native multisample texture with auto-resolve
- **WebGL2**: Renderbuffer with `gl.blitFramebuffer` for resolve
- Resolve happens before post-processing (bloom operates on 1x resolved texture)
- Configurable: 1x (disabled), 2x, 4x
- On tile-based mobile GPUs, MSAA resolve is nearly free (on-chip)

## Render Target Layout

| Target | Format | MSAA | Size | Purpose |
|--------|--------|------|------|---------|
| Color | RGBA8 | 4x | Canvas | Main scene color |
| Emissive | RGBA8 | 4x | Canvas | Bloom input (MRT output 1) |
| Depth | Depth24Plus | 4x | Canvas | Z-buffer |
| OIT Accumulation | RGBA16F | 1x | Canvas | Transparency weighted color |
| OIT Revealage | R8 | 1x | Canvas | Transparency alpha product |
| Shadow Atlas | Depth24Plus | 1x | Per config | CSM depth maps |
| Bloom Mips | RGBA16F | 1x | Halving | Progressive blur chain |
| Resolved Color | RGBA8 | 1x | Canvas | Post-MSAA resolve |
| Resolved Emissive | RGBA8 | 1x | Canvas | Post-MSAA emissive |

All targets are pre-allocated at initialization and recreated on canvas resize.

## Uniform Upload Strategy

**Decision: Dynamic offsets on a shared buffer** (universal agreement for per-object data)

Per-frame data (camera, lights) is uploaded once per frame to bind group 0.

Per-object data (world matrix, material overrides) uses dynamic offsets into a shared buffer. This avoids per-object bind group creation and minimizes binding overhead.

For WebGPU, `setBindGroup(2, objectBindGroup, [byteOffset])`.
For WebGL2, `gl.bindBufferRange(GL.UNIFORM_BUFFER, 2, buffer, byteOffset, size)`.

The uniform buffer is sized for the maximum expected visible objects (e.g., 4096 * 256 bytes = 1MB). If the scene exceeds this, the buffer grows geometrically.
