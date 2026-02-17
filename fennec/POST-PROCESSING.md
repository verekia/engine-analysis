# Post-Processing — Bloom, MSAA, OIT Transparency

## Overview

Fennec's post-processing pipeline is minimal and focused: MSAA for edge quality, Weighted Blended OIT for headache-free transparency, and Unreal-style bloom driven by emissive vertices. No tone mapping or color grading — games can add those as custom passes if needed.

## Render Target Layout

```
┌──────────────────────────────────────────┐
│            MSAA Render Targets           │
│  ┌──────────────┐  ┌──────────────────┐  │
│  │ Color (MSAA) │  │  Depth (MSAA)    │  │
│  │  RGBA16F     │  │  Depth24         │  │
│  │  4x samples  │  │  4x samples      │  │
│  └──────────────┘  └──────────────────┘  │
│                                          │
│  ┌──────────────┐  ┌──────────────────┐  │
│  │ OIT Accum    │  │  OIT Revealage   │  │
│  │  RGBA16F     │  │  R8              │  │
│  │  4x samples  │  │  4x samples      │  │
│  └──────────────┘  └──────────────────┘  │
└──────────────────────────────────────────┘
          │ resolve
          ▼
┌──────────────────────────────────────────┐
│          Resolved (1x) Targets           │
│  ┌──────────────┐  ┌──────────────────┐  │
│  │ Color        │  │  Bloom Extract   │  │
│  │  RGBA16F     │  │  RGBA16F         │  │
│  └──────────────┘  └──────────────────┘  │
│  ┌──────────────┐                        │
│  │ Bloom Blur   │  (ping-pong chain)     │
│  │ Half/Quarter │                        │
│  └──────────────┘                        │
└──────────────────────────────────────────┘
          │ composite
          ▼
┌──────────────────────────────────────────┐
│           Canvas (final output)          │
└──────────────────────────────────────────┘
```

## MSAA

### Configuration

MSAA is enabled by default with 4x samples. It can be configured at engine creation:

```typescript
const engine = createEngine({
  canvas,
  antialias: true,      // Enable MSAA (default: true)
  sampleCount: 4,       // 1, 2, or 4 (default: 4)
})
```

### Implementation

MSAA is implemented at the render target level, not the canvas level. This gives us control over resolve timing (needed for OIT and bloom):

1. All scene rendering happens into MSAA render targets
2. After the opaque pass and OIT composite, the MSAA target is resolved to a single-sample texture
3. Post-processing (bloom) operates on the resolved single-sample texture
4. Final result is blitted to the canvas

The MSAA resolve step uses the GPU's native resolve operation (WebGPU `resolveTarget`, WebGL2 `blitFramebuffer`), which is hardware-accelerated and essentially free.

### Why Not Canvas MSAA

Canvas-level MSAA (`antialias: true` in context creation) doesn't work with our pipeline because:
- We can't read the MSAA canvas as a texture for post-processing
- We can't control resolve timing for OIT
- We can't have separate MSAA color + OIT targets

## Weighted Blended Order-Independent Transparency (OIT)

### Why OIT

Three.js transparency is notoriously painful because it requires:
- Sorting transparent objects back-to-front
- Sorting fails with intersecting/overlapping transparent objects
- Manual `renderOrder` tweaking
- Separate render passes for complex cases

Weighted Blended OIT eliminates all of this. Transparent objects can be rendered in any order and the result is always correct (within the limits of the approximation).

### How It Works

The technique (McGuire and Bavoil, 2013) uses two render targets during the transparent pass:

1. **Accumulation buffer** (RGBA16F): Stores weighted sum of `color * alpha * weight`
2. **Revealage buffer** (R8): Stores the product of `(1 - alpha * weight)` — how much of the background shows through

The weight function biases by depth so that closer transparent surfaces contribute more:

```glsl
// Weight function — closer fragments get higher weight
float weight(float z, float alpha) {
  // McGuire 2013 weight function (equation 10)
  return alpha * max(1e-2, min(3e3, 10.0 / (1e-5 + pow(z / 200.0, 4.0))));
}
```

### OIT Accumulation Pass

Render all transparent objects (any order) with special blend modes:

```glsl
// Fragment shader output
layout(location = 0) out vec4 accumulation;  // Weighted color
layout(location = 1) out float revealage;    // Coverage

void main() {
  vec4 color = computeFragColor();  // Normal lighting + material
  float w = weight(gl_FragCoord.z, color.a);

  accumulation = vec4(color.rgb * color.a * w, color.a * w);
  revealage = color.a * w;
}
```

**Blend state for accumulation:**
```
src: ONE, dst: ONE  (additive)
```

**Blend state for revealage:**
```
src: ZERO, dst: ONE_MINUS_SRC_ALPHA  (multiplicative)
```

Initialize revealage buffer to 1.0 (fully revealed = no transparent objects).

### OIT Composite Pass

A full-screen quad reads both buffers and composites over the opaque scene:

```glsl
void main() {
  vec4 accum = texelFetch(accumTexture, ivec2(gl_FragCoord.xy), 0);
  float reveal = texelFetch(revealTexture, ivec2(gl_FragCoord.xy), 0).r;

  // Avoid division by zero
  if (accum.a < 1e-4) discard;

  // Average weighted color
  vec3 avgColor = accum.rgb / max(accum.a, 1e-4);

  // Composite: transparent color blended over opaque
  fragColor = vec4(avgColor, 1.0 - reveal);
}
```

This final result is alpha-blended over the opaque color buffer.

### OIT Limitations

- Approximate: very thin overlapping layers with similar depth can show artifacts
- No refraction or distortion
- Works best with alpha values 0.1–0.9 (very transparent or very opaque objects look fine, but many overlapping semi-transparent layers at similar depths may wash out)

For game use cases (windows, water, particle effects, UI elements), these limitations are rarely noticeable.

### MSAA Compatibility

OIT works with MSAA. Both the accumulation and revealage buffers are MSAA render targets. They're resolved after the accumulation pass, before the composite pass.

## Bloom (Unreal-Style)

### Overview

Bloom makes emissive surfaces glow. Fennec uses a **Dual Kawase blur** (used in Unreal Engine 4+), which is faster than Gaussian blur and produces better-looking results.

The bloom pipeline:
1. **Extract** bright pixels from the resolved color buffer
2. **Downsample + blur** through a mip chain (Dual Kawase downsample kernel)
3. **Upsample + blur** back up (Dual Kawase upsample kernel)
4. **Composite** the blurred bloom over the scene

### Per-Vertex Bloom via Emissive

Bloom is driven by the emissive output of materials. Vertices with `emissiveIntensity > bloomThreshold` contribute to bloom. This is controlled through the material index system:

```typescript
// Material index 2 has emissive — it will bloom
mesh.setMaterialIndex(2, {
  color: 0x00ccaa,
  emissive: 0x00ccaa,
  emissiveIntensity: 0.7,   // Above default threshold of 0.0
})
```

### MRT Bloom Output

During the main render pass, an optional second color attachment receives bloom-contributing color. This avoids a separate threshold pass:

```glsl
// Main fragment shader, when HAS_BLOOM is defined
layout(location = 0) out vec4 fragColor;
layout(location = 1) out vec4 fragBloom;

void main() {
  // ... lighting ...
  fragColor = vec4(finalColor, opacity);

  // Bloom output: only emissive above threshold
  vec3 emissiveContrib = emissiveColor * emissiveIntensity;
  float bloomLuma = dot(emissiveContrib, vec3(0.2126, 0.7152, 0.0722));
  fragBloom = vec4(emissiveContrib * smoothstep(bloomThreshold, bloomThreshold + 0.5, bloomLuma), 1.0);
}
```

When bloom is disabled globally, the second attachment is not bound and `HAS_BLOOM` is not defined — zero overhead.

### Dual Kawase Blur

The Dual Kawase blur uses two custom filter kernels that work on a mip chain:

**Downsample kernel** (13-tap, but only 4 bilinear fetches):
```glsl
vec4 downsample(sampler2D tex, vec2 uv, vec2 halfPixel) {
  vec4 sum = texture(tex, uv) * 4.0;
  sum += texture(tex, uv - halfPixel);
  sum += texture(tex, uv + halfPixel);
  sum += texture(tex, uv + vec2(halfPixel.x, -halfPixel.y));
  sum += texture(tex, uv - vec2(halfPixel.x, -halfPixel.y));
  return sum / 8.0;
}
```

**Upsample kernel** (9-tap, 8 bilinear fetches):
```glsl
vec4 upsample(sampler2D tex, vec2 uv, vec2 halfPixel) {
  vec4 sum = texture(tex, uv + vec2(-halfPixel.x * 2.0, 0.0));
  sum += texture(tex, uv + vec2(-halfPixel.x, halfPixel.y)) * 2.0;
  sum += texture(tex, uv + vec2(0.0, halfPixel.y * 2.0));
  sum += texture(tex, uv + vec2(halfPixel.x, halfPixel.y)) * 2.0;
  sum += texture(tex, uv + vec2(halfPixel.x * 2.0, 0.0));
  sum += texture(tex, uv + vec2(halfPixel.x, -halfPixel.y)) * 2.0;
  sum += texture(tex, uv + vec2(0.0, -halfPixel.y * 2.0));
  sum += texture(tex, uv + vec2(-halfPixel.x, -halfPixel.y)) * 2.0;
  return sum / 12.0;
}
```

### Bloom Mip Chain

```
Full res → 1/2 → 1/4 → 1/8 → 1/16 → 1/32  (downsample)
                                        ↓
Full res ← 1/2 ← 1/4 ← 1/8 ← 1/16 ← 1/32  (upsample)
```

Each level is a separate render target. The downsample pass reads level N and writes level N+1. The upsample pass reads level N+1 and additively blends into level N.

### Bloom Configuration

```typescript
interface BloomConfig {
  enabled: boolean          // Default: true
  threshold: number         // Min emissive brightness, default: 0.0
  intensity: number         // Bloom strength, default: 1.0
  radius: number            // Blur spread, default: 0.85
  levels: number            // Mip levels, default: 5
}
```

### Bloom Composite

The final upsample result (at full resolution) is blended additively onto the scene:

```glsl
void main() {
  vec3 scene = texture(sceneTexture, v_uv).rgb;
  vec3 bloom = texture(bloomTexture, v_uv).rgb;
  fragColor = vec4(scene + bloom * bloomIntensity, 1.0);
}
```

## Full Post-Processing Pipeline

```typescript
const postProcess = (backend: GPUBackend, state: PostProcessState) => {
  // 1. MSAA resolve (hardware)
  resolveMSAA(backend, state.msaaColor, state.resolvedColor)
  resolveMSAA(backend, state.msaaBloom, state.resolvedBloom)

  // 2. Bloom downsample chain
  let src = state.resolvedBloom
  for (let i = 0; i < state.bloom.levels; i++) {
    renderFullscreenQuad(backend, state.bloomDownTargets[i], bloomDownsampleShader, src)
    src = state.bloomDownTargets[i]
  }

  // 3. Bloom upsample chain
  for (let i = state.bloom.levels - 1; i >= 0; i--) {
    const dst = i > 0 ? state.bloomDownTargets[i - 1] : state.bloomUpTarget
    renderFullscreenQuad(backend, dst, bloomUpsampleShader, state.bloomDownTargets[i], {
      blendMode: 'additive',
      radius: state.bloom.radius,
    })
  }

  // 4. Final composite: scene + bloom → canvas
  renderFullscreenQuad(backend, null /* canvas */, compositeShader, {
    scene: state.resolvedColor,
    bloom: state.bloomUpTarget,
    bloomIntensity: state.bloom.intensity,
  })
}
```

## Performance Budget

| Pass | Typical Cost (mobile) | Notes |
|------|-----------------------|-------|
| MSAA resolve | ~0.1ms | Hardware operation |
| Bloom downsample (5 levels) | ~0.3ms | 5 fullscreen passes at decreasing res |
| Bloom upsample (5 levels) | ~0.3ms | 5 fullscreen passes at increasing res |
| Bloom composite | ~0.1ms | Single fullscreen pass |
| OIT composite | ~0.1ms | Single fullscreen pass |
| **Total post-processing** | **~0.9ms** | Well within 16ms budget |
