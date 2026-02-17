# Transparency & Bloom

## Weighted Blended Order-Independent Transparency (OIT)

### The Problem with Sorted Transparency

Three.js (and most engines) sort transparent objects back-to-front each frame and draw them in order. This breaks in many common cases:

1. **Intersecting geometry** — two transparent planes crossing each other can never be sorted correctly
2. **Cyclic overlap** — three transparent objects where A is in front of B, B in front of C, C in front of A
3. **Large objects** — a transparent object surrounding the camera can't be sorted relative to objects inside it
4. **Per-pixel correctness** — sorting is per-object, not per-fragment. A single transparent mesh with concavities will self-occlude incorrectly.
5. **CPU cost** — sorting thousands of transparent objects every frame adds measurable overhead

### Weighted Blended OIT (McGuire & Bavoil 2013)

This technique achieves order-independent transparency in a single rendering pass with two extra render targets. No sorting needed.

**Principle**: Instead of blending fragments in order (which requires sorting), we accumulate all transparent fragments and compute an approximate blend using depth-based weights. Fragments closer to the camera get higher weights.

### Render Targets

```
Opaque pass target:   RGBA8 (standard color, MSAA)
OIT accumulation:     RGBA16F (premultiplied color × weight, sum of weights)
OIT revealage:        R8 (product of 1-alpha, i.e. total visibility through all transparent layers)
```

### OIT Pass Setup

**WebGL2:**
```typescript
// Framebuffer with two color attachments
gl.bindFramebuffer(gl.FRAMEBUFFER, oitFBO)
gl.drawBuffers([gl.COLOR_ATTACHMENT0, gl.COLOR_ATTACHMENT1])

// Attachment 0: accumulation (RGBA16F)
// Clear to (0, 0, 0, 0)
// Blend: src*ONE + dst*ONE (additive)

// Attachment 1: revealage (R8)
// Clear to (1, 1, 1, 1) — fully transparent = nothing occluded
// Blend: ZERO, ONE_MINUS_SRC_COLOR (multiplicative)
```

**WebGPU:**
```typescript
const oitPassDescriptor: GPURenderPassDescriptor = {
  colorAttachments: [
    {
      view: accumTexture.createView(),
      clearValue: [0, 0, 0, 0],
      loadOp: 'clear',
      storeOp: 'store',
    },
    {
      view: revealageTexture.createView(),
      clearValue: [1, 0, 0, 0],
      loadOp: 'clear',
      storeOp: 'store',
    },
  ],
  depthStencilAttachment: {
    view: depthTexture.createView(),
    depthLoadOp: 'load',        // reuse depth from opaque pass (read-only)
    depthStoreOp: 'store',
    depthReadOnly: true,         // DON'T write depth for transparent objects
  },
}
```

### OIT Fragment Shader

```glsl
// OIT output function — called instead of normal fragment output for transparent objects

layout(location = 0) out vec4 outAccum;     // accumulation
layout(location = 1) out float outRevealage; // revealage

void outputOIT(vec4 color) {
  float alpha = color.a;

  // Depth-based weight function
  // Objects closer to camera get higher weight → appear more opaque
  float viewDepth = gl_FragCoord.z; // 0 (near) to 1 (far)

  // Weight function from McGuire & Bavoil, tuned for typical game scenes
  float weight = alpha * max(1e-2, min(
    3e3,
    10.0 / (1e-5 + pow(viewDepth / 200.0, 4.0))
  ));

  // Accumulation: premultiplied color × weight
  outAccum = vec4(color.rgb * alpha * weight, alpha * weight);

  // Revealage: product of (1 - alpha)
  outRevealage = alpha;
}
```

### OIT Composite Pass

After drawing all transparent objects, composite the OIT result over the opaque scene:

```glsl
// Full-screen quad fragment shader
uniform sampler2D u_accum;
uniform sampler2D u_revealage;
uniform sampler2D u_opaqueColor;

out vec4 fragColor;

void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution;

  vec4 accum = texture(u_accum, uv);
  float revealage = texture(u_revealage, uv).r;
  vec4 opaque = texture(u_opaqueColor, uv);

  // If no transparent fragments were written, revealage = 1.0
  if (revealage >= 1.0) {
    fragColor = opaque;
    return;
  }

  // Reconstruct transparent color
  vec3 transparentColor = accum.rgb / max(accum.a, 1e-5);

  // Blend: transparent over opaque
  // revealage = product of (1-alpha) of all transparent layers
  // So (1 - revealage) is the total opacity of all transparent layers combined
  fragColor = vec4(
    transparentColor * (1.0 - revealage) + opaque.rgb * revealage,
    1.0
  );
}
```

### Blend States

The accumulation and revealage targets require specific blend states:

**Accumulation (RGBA16F):**
- Color: `src × ONE + dst × ONE` (additive)
- Alpha: `src × ONE + dst × ONE` (additive)

**Revealage (R8):**
- Color: `ZERO × dst + SRC_COLOR × dst` → effectively `dst × (1 - srcAlpha)` when outputting alpha as color
- Actually: `src = ZERO, dst = ONE_MINUS_SRC_COLOR`

In WebGL2:
```typescript
// For accumulation attachment
gl.blendFunci(0, gl.ONE, gl.ONE)
// For revealage attachment
gl.blendFunci(1, gl.ZERO, gl.ONE_MINUS_SRC_COLOR)
```

In WebGPU, blend state is configured per color target in the pipeline descriptor.

### Depth Handling

Transparent objects read the depth buffer (from the opaque pass) but do not write to it. This ensures:
1. Transparent objects behind opaque objects are correctly occluded
2. Transparent objects do not occlude each other (OIT handles this through weights)

### Limitations

- Not physically accurate for deeply stacked colored glass (e.g., red glass behind blue glass won't look correct)
- Very similar depths with very different colors may not separate cleanly
- For game use (particles, windows, water, UI elements), it looks correct and natural

## Bloom (Unreal-Style)

### Overview

Bloom adds a glow effect around bright areas of the scene. Rabbit uses the progressive downsample/upsample approach from Unreal Engine (based on Jimenez 2014, "Next Generation Post Processing in Call of Duty: Advanced Warfare").

This approach is high quality and efficient: it uses a chain of progressively smaller textures (mip chain) for the blur, avoiding expensive large-kernel Gaussian blurs.

### Pipeline

```
Scene color (after OIT composite)
        │
        ▼
   ┌─────────────┐
   │ Brightness   │  Extract pixels above a threshold
   │ Threshold    │  Output: same resolution as scene
   └──────┬──────┘
          │
   Progressive Downsample (each level = half resolution)
          │
          ▼
   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌──────────┐
   │ Mip 0       │──▶│ Mip 1       │──▶│ Mip 2       │──▶│ Mip 3    │
   │ full res    │   │ 1/2 res     │   │ 1/4 res     │   │ 1/8 res  │
   └─────────────┘   └─────────────┘   └─────────────┘   └──────────┘
                                                                │
   Progressive Upsample (each level = double resolution)        │
                                                                ▼
   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌──────────┐
   │ Up 0        │◀──│ Up 1        │◀──│ Up 2        │◀──│ Up 3     │
   │ full res    │   │ 1/2 res     │   │ 1/4 res     │   │ 1/8 res  │
   └──────┬──────┘   └─────────────┘   └─────────────┘   └──────────┘
          │
          ▼
   ┌─────────────┐
   │ Composite   │  Additively blend bloom over scene
   └──────┬──────┘
          │
          ▼
     Final color
```

### Brightness Threshold

```glsl
uniform sampler2D u_sceneColor;
uniform float u_threshold;     // default 1.0 (HDR values above 1)
uniform float u_softThreshold; // default 0.5 (smooth knee)

out vec4 fragColor;

void main() {
  vec3 color = texture(u_sceneColor, v_uv).rgb;

  // Luminance
  float luma = dot(color, vec3(0.2126, 0.7152, 0.0722));

  // Soft threshold (smooth transition instead of hard cutoff)
  float knee = u_threshold * u_softThreshold;
  float soft = luma - u_threshold + knee;
  soft = clamp(soft, 0.0, 2.0 * knee);
  soft = soft * soft / (4.0 * knee + 1e-4);

  float contribution = max(soft, luma - u_threshold);
  contribution /= max(luma, 1e-4);

  fragColor = vec4(color * contribution, 1.0);
}
```

### Progressive Downsample

Each downsample step uses a 13-tap filter (tent filter) to avoid aliasing:

```glsl
// 13-tap downsample (Karis average on first pass to avoid fireflies)
uniform sampler2D u_src;
uniform vec2 u_texelSize; // 1.0 / source resolution

out vec4 fragColor;

void main() {
  vec2 uv = v_uv;

  // 13-tap pattern:
  //   a - b - c
  //   - d - e -
  //   f - g - h
  //   - i - j -
  //   k - l - m
  vec3 a = texture(u_src, uv + vec2(-2, -2) * u_texelSize).rgb;
  vec3 b = texture(u_src, uv + vec2( 0, -2) * u_texelSize).rgb;
  vec3 c = texture(u_src, uv + vec2( 2, -2) * u_texelSize).rgb;
  vec3 d = texture(u_src, uv + vec2(-1, -1) * u_texelSize).rgb;
  vec3 e = texture(u_src, uv + vec2( 1, -1) * u_texelSize).rgb;
  vec3 f = texture(u_src, uv + vec2(-2,  0) * u_texelSize).rgb;
  vec3 g = texture(u_src, uv).rgb;
  vec3 h = texture(u_src, uv + vec2( 2,  0) * u_texelSize).rgb;
  vec3 i = texture(u_src, uv + vec2(-1,  1) * u_texelSize).rgb;
  vec3 j = texture(u_src, uv + vec2( 1,  1) * u_texelSize).rgb;
  vec3 k = texture(u_src, uv + vec2(-2,  2) * u_texelSize).rgb;
  vec3 l = texture(u_src, uv + vec2( 0,  2) * u_texelSize).rgb;
  vec3 m = texture(u_src, uv + vec2( 2,  2) * u_texelSize).rgb;

  // Weighted average (tent filter weights)
  vec3 result = (d + e + i + j) * 0.25 * 0.5;
  result += (a + b + d + g) * 0.25 * 0.125;
  result += (b + c + e + g) * 0.25 * 0.125;
  result += (g + f + d + i) * 0.25 * 0.125; // approximate
  result += (g + h + j + e) * 0.25 * 0.125; // approximate

  fragColor = vec4(result, 1.0);
}
```

On the first downsample pass, we use the Karis average to prevent bright pixel fireflies:

```glsl
// Karis average: weight each sample group by 1/(1+luma) to prevent fireflies
float karisWeight(vec3 c) {
  return 1.0 / (1.0 + dot(c, vec3(0.2126, 0.7152, 0.0722)));
}
```

### Progressive Upsample

Each upsample step uses a 9-tap tent filter and adds the result to the next mip level:

```glsl
uniform sampler2D u_src;       // current mip (low res)
uniform sampler2D u_highRes;   // higher res mip to blend with
uniform vec2 u_texelSize;
uniform float u_bloomRadius;   // blur radius, default 1.0
uniform float u_bloomIntensity; // per-mip intensity

out vec4 fragColor;

void main() {
  vec2 uv = v_uv;
  float r = u_bloomRadius;

  // 9-tap tent filter
  vec3 result = vec3(0.0);
  result += texture(u_src, uv + vec2(-r, -r) * u_texelSize).rgb * 1.0;
  result += texture(u_src, uv + vec2( 0, -r) * u_texelSize).rgb * 2.0;
  result += texture(u_src, uv + vec2( r, -r) * u_texelSize).rgb * 1.0;
  result += texture(u_src, uv + vec2(-r,  0) * u_texelSize).rgb * 2.0;
  result += texture(u_src, uv                              ).rgb * 4.0;
  result += texture(u_src, uv + vec2( r,  0) * u_texelSize).rgb * 2.0;
  result += texture(u_src, uv + vec2(-r,  r) * u_texelSize).rgb * 1.0;
  result += texture(u_src, uv + vec2( 0,  r) * u_texelSize).rgb * 2.0;
  result += texture(u_src, uv + vec2( r,  r) * u_texelSize).rgb * 1.0;
  result /= 16.0;

  // Add to the higher-resolution mip
  vec3 highRes = texture(u_highRes, uv).rgb;
  fragColor = vec4(highRes + result * u_bloomIntensity, 1.0);
}
```

### Bloom Configuration

```typescript
interface BloomConfig {
  enabled: boolean            // default true
  threshold: number           // brightness threshold, default 1.0
  softThreshold: number       // knee softness, default 0.5
  intensity: number           // overall bloom strength, default 0.5
  radius: number              // blur radius per pass, default 1.0
  mipCount: number            // downsample levels, default 5
}
```

### Per-Vertex Bloom

Bloom is driven by emissive values in the material. When a vertex has emissive intensity > 0 (via material or material index entries), the emissive contribution is written to the scene color as HDR values > 1.0. The bloom threshold then picks up these bright areas.

The flow:
1. Material index entry 2 has `emissive: [0, 0.8, 0.7]`, `emissiveIntensity: 0.7`
2. Fragment shader computes `emissiveContrib = [0, 0.56, 0.49]`
3. Final fragment color includes this emissive: `diffuse + emissiveContrib`
4. Bloom threshold extracts the bright portion
5. Blur and composite produce the glow

No special "bloom material" or "bloom layer" is needed — it's entirely driven by emissive values.

## Tone Mapping

After bloom compositing, the HDR scene is tone-mapped to LDR for display:

```glsl
// ACES Filmic Tone Mapping
vec3 ACESFilmic(vec3 x) {
  float a = 2.51;
  float b = 0.03;
  float c = 2.43;
  float d = 0.59;
  float e = 0.14;
  return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}

void main() {
  vec3 color = texture(u_sceneColor, v_uv).rgb;

  // Apply exposure
  color *= u_exposure; // default 1.0

  // Tone map
  color = ACESFilmic(color);

  // Gamma correction (if not using sRGB framebuffer)
  color = pow(color, vec3(1.0 / 2.2));

  fragColor = vec4(color, 1.0);
}
```

## Complete Post-Processing Pipeline

```
MSAA render target (opaque)
         │
         ├─ MSAA resolve → resolved opaque color
         │
OIT targets (accumulation + revealage)
         │
         ├─ OIT composite (transparent over opaque) → combined HDR color
         │
         ├─ Brightness threshold → bright areas only
         │
         ├─ Downsample chain (5 mips)
         │
         ├─ Upsample chain (5 mips, accumulating bloom)
         │
         ├─ Bloom composite (add bloom to scene) → bloomed HDR color
         │
         ├─ Tone mapping (ACES) → LDR color
         │
         └─ Output to screen (canvas / swapchain)
```

Each step is a full-screen quad draw (6 vertices, no index buffer). The vertex shader is shared across all post-processing passes:

```glsl
const vec2 positions[3] = vec2[3](
  vec2(-1.0, -1.0),
  vec2( 3.0, -1.0),
  vec2(-1.0,  3.0)
);

out vec2 v_uv;

void main() {
  gl_Position = vec4(positions[gl_VertexID], 0.0, 1.0);
  v_uv = gl_Position.xy * 0.5 + 0.5;
}
```

(We use a single oversized triangle instead of a quad to avoid the diagonal seam artifact and save one triangle.)
