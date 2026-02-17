# 10 — Weighted Blended Order-Independent Transparency (WBOIT)

## The Problem

Traditional alpha blending requires rendering transparent objects back-to-front. This causes:
- Sorting artifacts when objects overlap in complex ways
- Per-pixel sort failures (intersecting transparent geometry)
- Developer headaches with manual `renderOrder` management (the three.js experience)

## The Solution: WBOIT

Weighted Blended OIT (McGuire & Bavoil 2013) renders all transparent geometry in a **single pass** without sorting. Each transparent fragment contributes to two buffers using a depth-weighted accumulation formula. A final composite pass reconstructs the approximate blended result.

The approximation is based on the observation that for most game-style transparency (windows, particles, foliage, UI panels), the exact compositing order matters less than the overall visual contribution. WBOIT handles this well.

## How It Works

### Pass 1: Accumulate Transparent Fragments

All transparent geometry renders into two render targets simultaneously via MRT. Depth testing is **on** (against the opaque depth buffer) but depth **writing** is **off**.

**Render Target 0 (RGBA16F) — Accumulation:**
```glsl
// accumulation = sum of (premultiplied_color * weight)
fragColor0 = vec4(color.rgb * color.a * weight, color.a * weight);
```

**Render Target 1 (R16F) — Revealage:**
```glsl
// revealage = product of (1 - alpha * weight_factor)
fragColor1 = color.a * weightFactor;
// (Written with GL_ZERO, GL_ONE_MINUS_SRC_COLOR blending for multiplicative accumulation)
```

The **weight function** controls how depth affects the contribution. Fragments closer to the camera get higher weight:

```glsl
float computeWeight(float depth, float alpha) {
  // McGuire's recommended weight function (Equation 10)
  // depth is in [0,1] clip space (0 = near, 1 = far)
  float d = 1.0 - depth;  // Invert so near = 1, far = 0
  return alpha * max(0.01, min(3000.0, 10.0 / (0.00001 + pow(d, 3.0))));
}
```

### Pass 2: Composite

A fullscreen quad reads both buffers and composites over the opaque scene:

```glsl
uniform sampler2D u_accumulation;  // RGBA16F
uniform sampler2D u_revealage;     // R16F

void main() {
  vec4 accum = texture(u_accumulation, v_uv);
  float reveal = texture(u_revealage, v_uv).r;

  // Avoid division by zero
  if (accum.a < 0.00001) {
    discard;  // No transparent fragments here
  }

  // Reconstruct average transparent color
  vec3 averageColor = accum.rgb / max(accum.a, 0.00001);

  // Alpha is derived from revealage:
  // reveal represents how much of the background is still visible
  float alpha = 1.0 - reveal;

  // Composite over the opaque scene (using standard over operator)
  fragColor = vec4(averageColor * alpha, alpha);
}
```

The composite pass uses premultiplied alpha blending:
```
srcFactor: ONE
dstFactor: ONE_MINUS_SRC_ALPHA
```

## Blend State Configuration

### Accumulation Buffer (RT0)

```
srcColorFactor: ONE
dstColorFactor: ONE
srcAlphaFactor: ONE
dstAlphaFactor: ONE
colorBlendOp: ADD
alphaBlendOp: ADD
```

This sums all fragment contributions.

### Revealage Buffer (RT1)

```
srcColorFactor: ZERO
dstColorFactor: ONE_MINUS_SRC_COLOR
```

This computes `product(1 - alpha_i)` across all fragments: each fragment multiplies the existing value by `(1 - alpha)`, starting from 1.0 (clear value).

**Clear values:**
- Accumulation: `(0, 0, 0, 0)`
- Revealage: `1.0` (fully revealed = no transparent fragments)

## Render Pipeline Integration

```
Frame Pipeline:
┌──────────────────────────────┐
│ 1. Shadow passes             │
├──────────────────────────────┤
│ 2. Opaque pass               │  → Scene RT + Depth buffer + Emissive RT
│    (depth write ON)          │
├──────────────────────────────┤
│ 3. Transparency pass (WBOIT) │  → Accumulation RT + Revealage RT
│    (depth test ON,           │     (reads opaque depth buffer)
│     depth write OFF)         │
├──────────────────────────────┤
│ 4. WBOIT composite           │  → Composites transparency over Scene RT
├──────────────────────────────┤
│ 5. Post-processing (bloom)   │  → Final output
└──────────────────────────────┘
```

## Render Target Setup

```ts
interface WBOITResources {
  accumulationRT: GPUTextureHandle  // RGBA16F
  revealageRT: GPUTextureHandle     // R16F (single channel)
  compositePipeline: PipelineHandle

  // These share the opaque pass's depth buffer (read-only in transparency pass)
}

const createWBOITResources = (device: WrenDevice, width: number, height: number): WBOITResources => {
  const accumulationRT = device.createTexture({
    width, height,
    format: 'rgba16float',
    usage: ['render-target', 'sampled'],
    label: 'wboit_accumulation',
  })

  const revealageRT = device.createTexture({
    width, height,
    format: 'r16float',
    usage: ['render-target', 'sampled'],
    label: 'wboit_revealage',
  })

  return { accumulationRT, revealageRT, compositePipeline }
}
```

## Weight Function Tuning

The weight function is the most important tuning parameter. Different scenes may need different weights:

```glsl
// Option A: McGuire's Equation 7 (simple, good for near/far scenes)
float weight = max(0.01, alpha * 10.0 + 0.01) * (1.0 - gl_FragCoord.z * 0.9);

// Option B: McGuire's Equation 10 (more aggressive depth weighting)
float weight = alpha * max(0.01, 3000.0 * pow(1.0 - gl_FragCoord.z, 3.0));

// Option C: Custom for Wren (tuned for 200m game worlds)
float linearDepth = (2.0 * u_near) / (u_far + u_near - gl_FragCoord.z * (u_far - u_near));
float weight = alpha * clamp(0.03 / (0.00001 + pow(linearDepth, 4.0)), 0.01, 3000.0);
```

Option C uses linear depth (instead of non-linear Z-buffer depth) for more uniform weighting across the scene's depth range.

## Limitations and Mitigations

| Limitation | Impact | Mitigation |
|-----------|--------|------------|
| Color blending is approximate | High-alpha overlapping surfaces may wash out | Tune weight function, keep transparent alphas moderate (0.3-0.7) |
| No refraction | Cannot simulate glass distortion | Not a goal for this engine |
| High-alpha stacking | Multiple opaque-like transparent objects at same depth look wrong | Use actual opaque materials where possible |
| Emissive through transparency | Bloom from objects behind transparency may leak | Accept as artistic effect, or mask bloom with revealage |

## Interaction with Material Index System

Transparent materials can use the material index system just like opaque ones. The fragment shader computes color and emissive from the material index tables, then writes to the WBOIT accumulation buffer:

```glsl
// Transparent fragment with material index
vec3 color = u_materialColors[a_materialIndex];
float alpha = u_opacity;

// WBOIT output
float weight = computeWeight(gl_FragCoord.z, alpha);
fragAccumulation = vec4(color * alpha * weight, alpha * weight);
fragRevealage = alpha;  // Blended multiplicatively
```

## WebGL2 Compatibility

WBOIT requires:
- **MRT**: Native in WebGL2 (`drawBuffers`)
- **Float textures**: Requires `EXT_color_buffer_float` extension (widely supported)
- **Float blending**: Requires `EXT_float_blend` extension (widely supported)

These extensions are checked at device creation. If unavailable (extremely rare on modern hardware), transparency falls back to sorted alpha blending.

## Performance

WBOIT adds minimal overhead:

| Component | Cost |
|-----------|------|
| Transparent pass (same as an opaque pass but with different blend state) | ~0.5ms (depends on overdraw) |
| Composite fullscreen quad | ~0.1ms |
| Extra memory (2 render targets at screen res) | ~16 MB at 1080p (RGBA16F + R16F) |
