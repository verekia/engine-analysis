# RENDERER — Final Decision

## Backend Architecture

**Decision: WebGPU-first abstraction with WebGL2 translation layer** (universal agreement)

```typescript
const device = await createDevice(canvas, {
  backend: 'auto' // 'auto' | 'webgpu' | 'webgl2'
})
```

Auto-detection: try `navigator.gpu.requestAdapter()` first, fall back to `canvas.getContext('webgl2')`.

## Render Pass Pipeline

Every frame executes these passes in order (universal agreement):

1. **Shadow Pass** — 3 CSM cascades, depth-only rendering
2. **Opaque Pass** — Sorted draw calls, MSAA, MRT (color + emissive)
3. **Transparent Pass** — OIT accumulation, depth read-only
4. **MSAA Resolve** — Resolve multisample to single-sample
5. **OIT Composite** — Blend transparent result over opaque
6. **Bloom** — Downsample chain, upsample chain, composite
7. **Final Blit** — Tone mapping, gamma correction, output to screen

## Bind Group Layout

**Decision: 3 bind groups by update frequency** (universal agreement)

| Slot | Name | Update Frequency | Contents |
|------|------|-----------------|----------|
| 0 | Per-frame | Once per frame | Camera VP matrix, light data, shadow matrices, cascade splits |
| 1 | Per-material | Per material switch | Material UBO, palette, textures, samplers |
| 2 | Per-object | Per draw call (dynamic offset) | World matrix, bone matrices |

Dynamic offsets on bind group 2 allow a single bind group with different offsets per object, minimizing rebind overhead.

## Draw Call Sorting

**Decision: 64-bit radix sort** (5/9: Caracal, Fennec, Mantis, Shark, Wren)

### Sort Key Layout

```
Bits 63-62: Layer (2b)       — opaque=0, transparent=1, overlay=2
Bits 61-60: Pass (2b)        — shadow=0, main=1, post=2
Bit  59:    Transparent (1b) — 0=opaque, 1=transparent
Bits 58-48: Pipeline ID (11b) — shader variant + blend/depth state
Bits 47-32: Material ID (16b) — material + texture binding
Bits 31-0:  Depth (32b)       — front-to-back (opaque) or back-to-front (transparent)
```

Pipeline switches are the most expensive state change, so they occupy the highest bits. Material/texture switches are moderate cost. Depth ordering enables early-Z rejection for opaque and correct visual ordering for transparent.

### Why Radix Sort

Radix sort is O(n) vs O(n log n) for `Array.sort`. For 2000 draw calls, radix sort takes ~0.05ms (two passes over 32-bit words). Predictable, stable, no worst-case behavior.

State change reduction with sorting: 90-99% (from ~2000 state changes unsorted to ~10-200 sorted).

## WebGL2 Backend

### Full State Cache

**Decision: Track all GL state to eliminate redundant calls** (8/9 agree)

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

Before every GL call, check if the state actually changed. Skip if identical. Eliminates 40-60% of redundant GL calls when draws are sorted.

### Pipeline as State Bundle

A WebGL2 "pipeline" is a frozen set of: program + blend state + depth state + cull state. Applying a pipeline sets all associated GL state in one call (with state cache checks).

### VAO Caching

Cache one VAO per geometry+pipeline combination. Created lazily on first use, destroyed when geometry or pipeline is disposed.

### BindGroup Translation

Bind groups translate to UBO bindings + texture unit assignments via `bindBufferRange` and `activeTexture`/`bindTexture`.

## WebGPU Backend

### Render Bundles

**Decision: Available for static geometry but not the primary optimization** (6/9 approach)

Render bundles pre-record draw commands for static geometry. Useful for scenes where most objects don't change, but the engine should not depend on them for baseline performance. The sort-based renderer with dynamic offsets is the primary strategy.

Render bundles are invalidated when:
- Objects are added or removed from the scene
- Materials change
- Visibility changes

### Pipeline Caching

All pipelines cached by a hash of their descriptor (shader variant, blend mode, depth state, cull mode, vertex layout, sample count). Typical scenes use 10-30 unique pipelines.

## Shader Management

**Decision: Dual-source WGSL + GLSL** (universal agreement)

Maintain separate WGSL and GLSL shader files. No runtime transpilation.

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

Variants compiled lazily on first use and cached by feature bitmask. Typical scene: 10-30 unique variants.

### Shader Warm-up

**Decision: Pre-compile common variants during loading** (Caracal approach)

Pre-warm the most common shader combinations during asset loading to prevent hitching during gameplay:
- Lambert opaque + shadow + texture
- Lambert opaque + shadow + vertex colors
- Lambert transparent
- Basic opaque
- Shadow depth-only

## MSAA

**Decision: 4x MSAA by default** (universal agreement)

- **WebGPU**: Native multisample texture with auto-resolve
- **WebGL2**: Renderbuffer with `gl.blitFramebuffer` for resolve
- Resolve happens before post-processing (bloom operates on 1x resolved texture)
- Configurable: 1x (disabled), 2x, 4x
- On tile-based mobile GPUs, MSAA resolve is nearly free (on-chip)

## Render Target Layout

Pre-allocated at initialization, recreated on canvas resize:

| Target | Format | MSAA | Size | Purpose |
|--------|--------|------|------|---------|
| Color | RGBA8 | 4x | Canvas | Main scene color |
| Emissive | RGBA8 | 4x | Canvas | Bloom input (MRT output 1) |
| Depth | Depth24Plus | 4x | Canvas | Z-buffer |
| OIT Accumulation | RGBA16F | 1x | Canvas | Transparency weighted color |
| OIT Revealage | R8 | 1x | Canvas | Transparency alpha product |
| Shadow Atlas | Depth24Plus | 1x | Per config | CSM depth maps (texture array) |
| Bloom Mips | RGBA16F | 1x | Halving | Progressive blur chain |
| Resolved Color | RGBA8 | 1x | Canvas | Post-MSAA resolve |
| Resolved Emissive | RGBA8 | 1x | Canvas | Post-MSAA emissive |

## Uniform Upload Strategy

**Decision: Dynamic offsets on a shared buffer** (universal agreement)

Per-frame data (camera, lights) is uploaded once per frame to bind group 0.

Per-object data (world matrix, material overrides) uses dynamic offsets into a shared buffer. This avoids per-object bind group creation and minimizes binding overhead.

The uniform buffer is sized for the maximum expected visible objects (e.g., 4096 × 256 bytes = 1MB). If the scene exceeds this, the buffer grows geometrically.

## Per-Draw Overhead Target

Target: **<2-3 microseconds per draw call** in JavaScript

For 2000 draws: ~4-6ms JS overhead, leaving ~10ms for GPU work within the 16.6ms frame budget.

## Performance Budget

```
Scene graph update:       0.2-0.5ms  (dirty nodes only)
Frustum culling:          0.1-0.2ms  (AABB tests)
Draw call sorting:        0.05-0.1ms (radix sort)
Uniform uploads:          0.2-0.5ms  (buffer writes)
Shadow pass (3 CSM):      1.5-2.5ms  (GPU, depth-only)
Main pass (2000 draws):   3.0-5.0ms  (GPU, sorted)
MSAA resolve:             0.2-0.5ms  (GPU)
OIT composite:            0.3-0.5ms  (GPU, fullscreen)
Bloom:                    0.5-1.5ms  (GPU, progressive)
JS overhead total:        1.0-2.0ms
GPU work total:           6.0-10.5ms
──────────────────────────────────────
Total frame time:         7.0-12.5ms
Headroom:                 4.1-9.6ms
```
