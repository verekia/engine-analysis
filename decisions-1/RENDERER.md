# RENDERER - Design Decisions

## Backend Architecture

**Decision: WebGPU-first abstraction with WebGL2 translation layer**

- Sources: All 9 implementations agree

```typescript
if (navigator.gpu && !forceWebGL2) {
  const adapter = await navigator.gpu.requestAdapter()
  if (adapter) {
    const device = await adapter.requestDevice()
    return new WebGPUBackend(device, canvas)
  }
}
const gl = canvas.getContext('webgl2')
return new WebGL2Backend(gl)
```

## Render Pass Pipeline

**Decision: Multi-pass pipeline (universal pattern)**

```
1. Shadow Pass        (3 CSM depth maps, depth-only)
2. Opaque Pass        (sorted by sort key, MSAA, MRT for emissive)
3. Transparent Pass   (OIT accumulation, depth read-only)
4. MSAA Resolve
5. OIT Composite
6. Bloom              (threshold → downsample → upsample)
7. Final Blit         (tone mapping → screen)
```

## Bind Group Layout

**Decision: Three bind groups by update frequency (universal pattern)**

| Slot | Name | Update Frequency | Contents |
|------|------|-----------------|----------|
| 0 | Per-frame | Once per frame | Camera VP matrix, light data, shadow matrices, cascade splits |
| 1 | Per-material | Per material switch | Material UBO, palette, textures, samplers |
| 2 | Per-object | Per draw call | World matrix, bone matrices (via dynamic offset) |

## Draw Call Sorting

**Decision: 64-bit composite sort keys with radix sort**

- Sources: Wren key layout (most complete); Mantis, Shark, Caracal, Fennec for radix sort (5/9)

Sort key layout (Wren-inspired):
```
[layer 2b] [pass 2b] [transparent 1b] [pipeline 11b] [material 16b] [depth 32b]
```

- Pipeline first (most expensive state change)
- Material second
- Depth last (front-to-back for opaque, for early-Z rejection)
- For transparent objects: depth is back-to-front (though OIT doesn't require ordering)

**Sort algorithm: Two-pass radix sort on 32-bit words**

```typescript
// O(n) sort - two passes on high and low 32-bit words
radixSort(drawCommands, keyExtractor)
```

Rationale: O(n) vs O(n log n). For 2000 draws, radix sort completes in ~0.05ms. Stable and predictable with no worst-case behavior.

## WebGL2 State Management

**Decision: Full state cache**

- Sources: 8/9 implementations use full state caching

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
```

Every GL state change is checked against the cache first. Eliminates 40-60% of redundant GL calls when combined with draw call sorting.

## WebGL2 Translation

**Decision: Pipeline → GL State Bundle, BindGroup → UBO + Textures**

Pipeline maps to a state bundle (program + blend + depth + cull). Applying a pipeline sets all associated GL state. Bind groups translate to UBO bindings + texture unit assignments. VAOs are cached per geometry+pipeline pair.

## Uniform Upload Strategy

**Decision: Triple-buffered ring buffer with dynamic offsets**

- Sources: Mantis, Shark, Wren (3/9 emphasize ring buffer); all 9 use dynamic offsets

4MB ring buffer with 3 frame regions. Each frame writes per-object uniforms sequentially into its region. Per-object switching uses dynamic offsets only — no bind group creation or buffer allocation per draw.

## Render Bundles (WebGPU)

**Decision: Available for static geometry, not primary optimization**

- Sources: 6/9 treat as available feature, not primary strategy
- Record render bundles for static objects when scene is mostly unchanging
- Fall back to normal draw submission for dynamic scenes
- Invalidate bundles on object add/remove/material change

Rationale: Sort-based rendering already achieves target performance. Render bundles add complexity for marginal gain in dynamic scenes. Reserve them for games with mostly static environments.

## Shader Management

**Decision: Dual-source shaders (WGSL + GLSL), feature flags, hash-based caching**

- Sources: All 9 implementations agree on dual-source and caching

Ship both WGSL and GLSL shaders. No runtime transpilation (fragile).

Feature flags for shader variants:
```
HAS_COLOR_TEXTURE, HAS_AO_TEXTURE, HAS_VERTEX_COLORS,
HAS_MATERIAL_INDEX, HAS_SKINNING, HAS_EMISSIVE,
SHADOW_RECEIVE, IS_TRANSPARENT
```

Cache compiled variants by feature bitmask:
```typescript
const shaderCache = new Map<number, CompiledShader>()
```

Pre-compile common variants during loading to avoid frame spikes (Caracal approach).

## Pipeline Caching

**Decision: Hash-based pipeline cache (universal pattern)**

```typescript
const pipelineKey = hash({
  shader: shaderVariantId,
  vertexLayout,
  blendMode,
  depthWrite,
  cullMode,
  sampleCount,
})

const pipelineCache = new Map<number, GALPipeline>()
```

Typical scene: 10-30 unique pipelines.

## MSAA Strategy

**Decision: 4x MSAA by default, resolved before post-processing**

- Sources: All 9 agree on 4x default

```
WebGPU: Native multisample texture with auto-resolve
WebGL2: Renderbuffer with gl.blitFramebuffer
```

MSAA is resolved before bloom and OIT composite to avoid operating on multisampled textures in post-processing (saves memory, simpler shaders).

## Render Targets

**Decision: Pre-allocated at init, recreated on resize**

| Target | Format | MSAA | Size | Purpose |
|--------|--------|------|------|---------|
| Color | RGBA8 | 4x | Canvas | Main scene |
| Depth | Depth24Plus | 4x | Canvas | Z-buffer |
| Emissive | RGBA8 | 4x | Canvas | Bloom input (MRT) |
| OIT Accum | RGBA16F | 1x | Canvas | Transparency |
| OIT Reveal | R8 | 1x | Canvas | Transparency |
| Shadow | Depth24Plus | 1x | 2048×2048×3 | CSM (texture array) |
| Bloom Mips | RGBA16F | 1x | Halving | Bloom chain |
| Resolved | RGBA8 | 1x | Canvas | Post-MSAA |

Resize destroys and recreates all canvas-sized targets.

## Performance Budget

```
Scene graph update:       0.2-0.5ms  (dirty nodes only)
Frustum culling:          0.1-0.2ms  (AABB tests)
Draw call sorting:        0.05-0.1ms (radix sort)
Uniform uploads:          0.2-0.5ms  (ring buffer writes)
Shadow pass (3 CSM):      1.5-3.0ms  (GPU, depth-only)
Main pass (2000 draws):   3.0-5.0ms  (GPU, sorted)
MSAA resolve:             0.2-0.5ms  (GPU)
OIT composite:            0.3-0.5ms  (GPU, fullscreen)
Bloom:                    0.5-1.5ms  (GPU, progressive)
JS overhead total:        1.0-2.0ms
GPU work total:           6.0-11.0ms
──────────────────────────────────────
Total frame time:         7.0-13.0ms
Headroom:                 3.6-9.6ms
```
