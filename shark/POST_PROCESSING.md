# Post-Processing Architecture

## Overview

Shark's post-processing pipeline handles three concerns:

1. **MSAA** — Hardware-accelerated multi-sample anti-aliasing
2. **Unreal-style Bloom** — Per-vertex bloom driven by emissive output
3. **Weighted Blended OIT** — Order-independent transparency

These are integrated into the main render pipeline rather than being standalone passes bolted on after the fact.

## Render Target Layout

```
+----------------------------------------------------------+
|                  MSAA Render Targets (4x)                 |
|                                                           |
|  RT0: Color (RGBA8 or RGBA16F)     -- main scene color   |
|  RT1: Emissive (RGBA16F)           -- emissive for bloom  |
|  Depth: DEPTH24STENCIL8 (4x MSAA)                        |
|                                                           |
|  Resolved to:                                             |
|  resolvedColor (RGBA8/16F, 1x)                           |
|  resolvedEmissive (RGBA16F, 1x)                           |
+----------------------------------------------------------+
|                  OIT Render Targets (1x)                  |
|                                                           |
|  RT0: Accumulation (RGBA16F)       -- weighted color sum  |
|  RT1: Revealage (R8)              -- alpha product        |
|  Uses same depth buffer (read-only, no write)             |
+----------------------------------------------------------+
|                  Bloom Chain                               |
|                                                           |
|  bloom_0: half-res (RGBA16F)                              |
|  bloom_1: quarter-res (RGBA16F)                           |
|  bloom_2: eighth-res (RGBA16F)                            |
|  bloom_3: sixteenth-res (RGBA16F)                         |
|  Each level has a horizontal + vertical blur target       |
+----------------------------------------------------------+
```

## MSAA (Multi-Sample Anti-Aliasing)

### Strategy

MSAA is applied to the opaque pass render target. The GPU rasterizer evaluates coverage at 4 sample points per pixel but runs the fragment shader once per pixel (at the center sample), then copies the result to all covered samples.

### Setup

**WebGPU**:
```typescript
// Create multisample texture
const msaaTexture = device.createTexture({
  width, height,
  format: 'rgba8unorm',
  usage: GPUTextureUsage.RENDER_ATTACHMENT,
  sampleCount: 4,
})

// Create resolve target (non-MSAA)
const resolvedTexture = device.createTexture({
  width, height,
  format: 'rgba8unorm',
  usage: GPUTextureUsage.RENDER_ATTACHMENT | GPUTextureUsage.TEXTURE_BINDING,
  sampleCount: 1,
})

// Render pass automatically resolves
const renderPassDescriptor = {
  colorAttachments: [{
    view: msaaTexture.createView(),
    resolveTarget: resolvedTexture.createView(),  // Auto-resolve on pass end
    loadOp: 'clear',
    storeOp: 'store',
  }],
}
```

**WebGL2**:
```typescript
// Create multisampled renderbuffer
gl.bindRenderbuffer(gl.RENDERBUFFER, msaaRenderbuffer)
gl.renderbufferStorageMultisample(gl.RENDERBUFFER, 4, gl.RGBA8, width, height)

// Attach to FBO
gl.framebufferRenderbuffer(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.RENDERBUFFER, msaaRenderbuffer)

// Resolve: blit from MSAA FBO to non-MSAA FBO
gl.bindFramebuffer(gl.READ_FRAMEBUFFER, msaaFBO)
gl.bindFramebuffer(gl.DRAW_FRAMEBUFFER, resolveFBO)
gl.blitFramebuffer(0, 0, w, h, 0, 0, w, h, gl.COLOR_BUFFER_BIT, gl.NEAREST)
```

### MRT with MSAA

Both the color and emissive render targets are MSAA. They resolve to separate non-MSAA textures at the end of the opaque pass. This gives us:
- `resolvedColor` — anti-aliased opaque scene
- `resolvedEmissive` — anti-aliased emissive data for bloom

## Unreal-Style Bloom

### Overview

The bloom pipeline extracts bright fragments (from the emissive render target), progressively downsamples them, applies Gaussian blur at each level, then upsamples and accumulates back to full resolution. The result is composited with the scene color.

### Pipeline

```
resolvedEmissive (full res)
    |
    v  Threshold (extract pixels above brightness threshold)
    |
    v  Downsample to half-res
    v  Downsample to quarter-res
    v  Downsample to eighth-res
    v  Downsample to sixteenth-res
    |
    v  Gaussian blur at sixteenth-res (H + V)
    v  Upsample + add to eighth-res
    v  Gaussian blur at eighth-res (H + V)
    v  Upsample + add to quarter-res
    v  Gaussian blur at quarter-res (H + V)
    v  Upsample + add to half-res
    v  Gaussian blur at half-res (H + V)
    |
    v  Composite: finalColor = sceneColor + bloom * bloomIntensity
```

### Parameters

```typescript
interface BloomConfig {
  enabled: boolean            // Toggle bloom (default: true)
  threshold: number           // Brightness threshold (default: 0.8)
  intensity: number           // Bloom strength (default: 0.5)
  radius: number              // Blur radius multiplier (default: 1.0)
  levels: number              // Downsample levels (default: 4)
}
```

### Threshold Pass

Extracts bright fragments from the emissive RT:

```glsl
uniform float u_bloomThreshold;

void main() {
  vec4 emissive = texture(u_emissiveTexture, v_uv);
  float brightness = dot(emissive.rgb, vec3(0.2126, 0.7152, 0.0722)); // Luminance
  float soft = brightness - u_bloomThreshold + 0.01; // Soft knee
  soft = clamp(soft, 0.0, 0.01 * 2.0);
  soft = soft * soft / (4.0 * 0.01 + 0.0001);
  float contribution = max(soft, brightness - u_bloomThreshold) / max(brightness, 0.0001);
  outColor = vec4(emissive.rgb * contribution, 1.0);
}
```

### Downsample (Bilinear)

Each level halves the resolution using a 2x2 bilinear filter:

```glsl
void main() {
  vec2 texelSize = 1.0 / u_sourceSize;
  vec3 color = texture(u_source, v_uv + texelSize * vec2(-0.5, -0.5)).rgb
             + texture(u_source, v_uv + texelSize * vec2( 0.5, -0.5)).rgb
             + texture(u_source, v_uv + texelSize * vec2(-0.5,  0.5)).rgb
             + texture(u_source, v_uv + texelSize * vec2( 0.5,  0.5)).rgb;
  outColor = vec4(color * 0.25, 1.0);
}
```

### Gaussian Blur (Separable, 9-tap)

Two passes per level (horizontal then vertical):

```glsl
const float weights[5] = float[](0.227027, 0.1945946, 0.1216216, 0.054054, 0.016216);

void main() {
  vec2 texelSize = 1.0 / u_sourceSize;
  vec3 result = texture(u_source, v_uv).rgb * weights[0];

  for (int i = 1; i < 5; i++) {
    vec2 offset = u_direction * texelSize * float(i) * u_radius;
    result += texture(u_source, v_uv + offset).rgb * weights[i];
    result += texture(u_source, v_uv - offset).rgb * weights[i];
  }

  outColor = vec4(result, 1.0);
}
```

`u_direction` is `(1, 0)` for horizontal and `(0, 1)` for vertical.

### Upsample + Accumulate

Each level is upsampled and added to the next-larger level:

```glsl
void main() {
  vec3 upsampled = texture(u_smallerLevel, v_uv).rgb;  // Bilinear upsampling
  vec3 current = texture(u_currentLevel, v_uv).rgb;
  outColor = vec4(current + upsampled, 1.0);
}
```

### Final Composite

```glsl
void main() {
  vec3 scene = texture(u_sceneColor, v_uv).rgb;
  vec3 bloom = texture(u_bloomResult, v_uv).rgb;
  vec3 result = scene + bloom * u_bloomIntensity;

  // Tone mapping (Reinhard)
  result = result / (result + vec3(1.0));

  // Gamma correction
  result = pow(result, vec3(1.0 / 2.2));

  outColor = vec4(result, 1.0);
}
```

## Weighted Blended Order-Independent Transparency (OIT)

### The Problem with Sorted Transparency

Three.js sorts transparent objects back-to-front by their center points. This fails when:
- Objects intersect each other
- Large objects overlap small ones (center-point ordering is wrong)
- Many transparent objects overlap (sorting is expensive and approximate)

### The OIT Solution

Weighted Blended OIT (McGuire & Bavoil 2013) renders all transparent fragments in a single pass without sorting. It uses two render targets:

1. **Accumulation** (RGBA16F): Premultiplied color weighted by a depth-dependent function
2. **Revealage** (R8): Product of (1 - alpha) for all fragments

### Weight Function

The weight function ensures near fragments contribute more than far fragments:

```glsl
float computeWeight(float alpha, float depth) {
  float z = depth;
  return alpha * max(1e-2, min(3e3, 10.0 / (1e-5 + pow(z / 5.0, 2.0) + pow(z / 200.0, 6.0))));
}
```

### Accumulation Pass

Transparent objects are rendered with specific blend modes:

```glsl
layout(location = 0) out vec4 outAccumulation;
layout(location = 1) out float outRevealage;

void main() {
  vec4 color = computeLitColor(); // Normal lighting calculations
  float w = computeWeight(color.a, gl_FragCoord.z);

  outAccumulation = vec4(color.rgb * color.a * w, color.a * w);
  outRevealage = color.a;
}
```

**Blend states**:
```
RT0 (Accumulation):  src=ONE, dst=ONE          (additive)
RT1 (Revealage):     src=ZERO, dst=ONE_MINUS_SRC_ALPHA  (product)
```

Depth buffer is bound read-only (test enabled, write disabled). This correctly occludes transparent fragments behind opaque surfaces.

### Composite Pass

A fullscreen quad composites the OIT result over the opaque scene:

```glsl
void main() {
  vec4 accum = texture(u_accumulation, v_uv);
  float revealage = texture(u_revealage, v_uv).r;

  // Fully opaque pixel (no transparent fragments)
  if (revealage >= 1.0) discard;

  // Reconstruct average color
  vec3 averageColor = accum.rgb / max(accum.a, 1e-5);

  // Composite over opaque scene
  vec3 opaqueColor = texture(u_sceneColor, v_uv).rgb;
  vec3 result = averageColor * (1.0 - revealage) + opaqueColor * revealage;

  outColor = vec4(result, 1.0);
}
```

### OIT Render Target Setup

**WebGPU**: Native float render target support.

**WebGL2**: Requires `EXT_color_buffer_float` extension for RGBA16F render targets. This is widely supported on desktop and most modern mobile GPUs. If unavailable, falls back to sorted transparency (less accurate but universal).

### OIT Limitations and Mitigations

**Limitations**:
- Approximate — doesn't perfectly handle high-contrast overlapping transparency
- Weight function needs tuning for extreme depth ranges
- Doesn't support refractive transparency

**Mitigations**:
- Works beautifully for typical game transparency (particles, windows, water, UI elements)
- Weight function tuned for Shark's expected depth range (0.1 to 500m)
- For exact ordering requirements, objects can use alpha-test (`discard`) instead

## Full Post-Processing Pipeline

```
1. Opaque pass -> MSAA color RT + MSAA emissive RT
2. MSAA resolve -> resolvedColor + resolvedEmissive
3. OIT accumulation pass -> oitAccum + oitRevealage (using opaque depth, read-only)
4. OIT composite -> composited (resolvedColor + OIT blended over it)
5. Bloom threshold on resolvedEmissive -> bloom chain input
6. Bloom downsample (4 levels)
7. Bloom blur (H+V at each level)
8. Bloom upsample + accumulate
9. Final composite: composited + bloom -> screen
   (includes tone mapping + gamma correction)
```

### Dynamic Pass Skipping

- **When bloom is disabled**: Skip emissive MRT, bloom chain, bloom composite. Final pass applies tone mapping + gamma only.
- **When no transparent objects**: Skip OIT render targets, accumulation, composite. resolvedColor goes directly to final composite.
- **When nothing emissive and nothing transparent**: Pipeline collapses to MSAA render + resolve + tone mapping.

These dynamic skips prevent paying for features not used in a given frame.

## Performance

| Pass | Cost (1080p, mobile) | Notes |
|------|---------------------|-------|
| MSAA resolve | ~0.3ms | Hardware blit |
| OIT accumulation | ~0.5ms | Same cost as forward pass for transparent subset |
| OIT composite | ~0.1ms | Fullscreen quad, simple math |
| Bloom threshold | ~0.1ms | Fullscreen quad at full-res |
| Bloom downsample (4 levels) | ~0.2ms | 4 fullscreen quads at decreasing res |
| Bloom blur (8 passes) | ~0.8ms | 8 fullscreen quads at various res |
| Bloom upsample (4 levels) | ~0.2ms | 4 fullscreen quads at increasing res |
| Final composite | ~0.1ms | Fullscreen quad, tone map + gamma |
| **Total** | **~2.3ms** | |
