# Post-Processing Pipeline

## Overview

Caracal's post-processing pipeline runs after the main scene render and OIT composite. It consists of:

1. **Bloom extraction + blur** — Selective Unreal-style bloom driven by emissive
2. **Bloom composite** — Add blurred bloom back to scene
3. **Tone mapping** — ACES filmic curve to convert HDR → LDR

```
Scene Color (after OIT composite)
│
├─ Bloom Extract → bright pixels only (threshold from emissive alpha)
│   ├─ Downsample ½ → Blur
│   ├─ Downsample ¼ → Blur
│   ├─ Downsample ⅛ → Blur
│   └─ Downsample 1/16 → Blur
│       └─ Upsample chain (add each level back)
│           └─ Bloom result
│
├─ Composite: scene + bloom
│
└─ Tone Map (ACES) → Canvas output
```

## Selective Bloom (Unreal-Style)

### Bloom Sources

Bloom in Caracal is **selective** — only pixels with emissive contribution bloom. This is controlled via the material index system:

```typescript
// Material index 2 has emissive — its pixels will bloom
const materialIndexColors = createMaterialIndexMap({
  0: { color: [1, 1, 1] },                                        // No bloom
  1: { color: [0, 0, 0] },                                        // No bloom
  2: { color: [0, 0.8, 0.7], emissive: [0, 1, 0.9], emissiveIntensity: 0.7 }, // Blooms
})
```

The scene render writes emissive intensity to the **alpha channel** of the color output:

```glsl
// In the main fragment shader:
fragColor = vec4(litColor + emissive, emissiveIntensity);
// emissiveIntensity > 0 means "this pixel should bloom"
```

### Bloom Extraction

The extraction pass reads the scene color and outputs only the bloom-contributing pixels:

```glsl
// bloom_extract.frag
uniform sampler2D u_sceneColor;
uniform float u_bloomThreshold;  // typically 0.0 (any emissive blooms)
uniform float u_bloomIntensity;  // global bloom strength (default: 1.0)

void main() {
  vec4 color = texture(u_sceneColor, v_uv);
  float emissiveStrength = color.a; // stored by main pass

  if (emissiveStrength <= u_bloomThreshold) {
    fragColor = vec4(0.0);
    return;
  }

  // Extract bloom color (the emissive contribution)
  // Scale by intensity and emissive strength
  fragColor = vec4(color.rgb * emissiveStrength * u_bloomIntensity, 1.0);
}
```

### Progressive Downscale

The extracted bloom image is progressively downscaled to create multiple mip levels:

```
Level 0: Full resolution (or ½ — start downscaled for performance)
Level 1: ½ × ½
Level 2: ¼ × ¼
Level 3: ⅛ × ⅛
Level 4: 1/16 × 1/16
```

Each downscale uses a simple bilinear downsample shader (sampling 4 texels per output pixel):

```glsl
// downsample.frag
void main() {
  vec2 texelSize = 1.0 / vec2(textureSize(u_source, 0));

  // 4-tap bilinear downsample (Karis average for first mip to prevent fireflies)
  vec3 a = texture(u_source, v_uv + vec2(-1, -1) * texelSize).rgb;
  vec3 b = texture(u_source, v_uv + vec2( 1, -1) * texelSize).rgb;
  vec3 c = texture(u_source, v_uv + vec2(-1,  1) * texelSize).rgb;
  vec3 d = texture(u_source, v_uv + vec2( 1,  1) * texelSize).rgb;

  fragColor = vec4((a + b + c + d) * 0.25, 1.0);
}
```

For the first downsample (from extract → level 0), we use a **Karis average** to prevent bright pixel fireflies:

```glsl
float karisWeight(vec3 color) {
  float brightness = max(max(color.r, color.g), color.b);
  return 1.0 / (1.0 + brightness);
}

// First downsample only:
vec3 result = (a * karisWeight(a) + b * karisWeight(b) +
               c * karisWeight(c) + d * karisWeight(d)) /
              (karisWeight(a) + karisWeight(b) + karisWeight(c) + karisWeight(d));
```

### Separable Gaussian Blur

Each mip level is blurred with a separable Gaussian filter (horizontal pass + vertical pass):

```glsl
// gaussian_blur.frag
uniform sampler2D u_source;
uniform vec2 u_direction; // (1,0) for horizontal, (0,1) for vertical
uniform float u_weights[5]; // Gaussian kernel weights

void main() {
  vec2 texelSize = 1.0 / vec2(textureSize(u_source, 0));
  vec3 result = texture(u_source, v_uv).rgb * u_weights[0];

  for (int i = 1; i < 5; i++) {
    vec2 offset = u_direction * texelSize * float(i);
    result += texture(u_source, v_uv + offset).rgb * u_weights[i];
    result += texture(u_source, v_uv - offset).rgb * u_weights[i];
  }

  fragColor = vec4(result, 1.0);
}
```

Kernel weights for σ = 2.0: `[0.227027, 0.194596, 0.121621, 0.054054, 0.016216]`

Each mip level gets 2 blur passes (H + V). Since each mip is ½ the previous size, the blur covers a progressively larger screen area — creating the characteristic bloom "glow spread."

### Progressive Upsample

The blurred mips are composited back together from smallest → largest:

```glsl
// upsample.frag
uniform sampler2D u_currentLevel;  // Current mip (blurred)
uniform sampler2D u_previousLevel; // Larger mip (from previous upsample)
uniform float u_bloomRadius;       // Blend factor per level (typically 0.5-1.0)

void main() {
  vec3 current = texture(u_currentLevel, v_uv).rgb;
  vec3 previous = texture(u_previousLevel, v_uv).rgb;

  fragColor = vec4(mix(previous, current, u_bloomRadius), 1.0);
}
```

The upsample chain starts from the smallest mip and adds each level, creating a multi-scale bloom that combines tight inner glow with wide outer glow.

### Bloom Composite

The final bloom result is added to the scene color:

```glsl
// bloom_composite.frag
uniform sampler2D u_sceneColor;
uniform sampler2D u_bloomTexture;
uniform float u_bloomStrength; // default: 0.5

void main() {
  vec3 scene = texture(u_sceneColor, v_uv).rgb;
  vec3 bloom = texture(u_bloomTexture, v_uv).rgb;

  fragColor = vec4(scene + bloom * u_bloomStrength, 1.0);
}
```

## Tone Mapping

After bloom composite, the HDR scene color is mapped to displayable LDR range using the ACES filmic curve:

```glsl
// ACES filmic tone mapping (Stephen Hill's fit)
vec3 acesToneMap(vec3 x) {
  const float a = 2.51;
  const float b = 0.03;
  const float c = 2.43;
  const float d = 0.59;
  const float e = 0.14;
  return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}

void main() {
  vec3 hdr = texture(u_composited, v_uv).rgb;

  // Exposure (optional, default 1.0)
  hdr *= u_exposure;

  // Tone map
  vec3 ldr = acesToneMap(hdr);

  // Gamma correction (if not using sRGB framebuffer)
  ldr = pow(ldr, vec3(1.0 / 2.2));

  fragColor = vec4(ldr, 1.0);
}
```

### sRGB Handling

- **WebGL 2**: Render to `SRGB8_ALPHA8` framebuffer format → hardware gamma correction, skip `pow` in shader
- **WebGPU**: Use `bgra8unorm-srgb` swap chain format → same automatic conversion

When sRGB framebuffers are used, the tone mapping shader omits the manual gamma correction.

## Render Target Setup

The post-processing pipeline needs these render targets:

| Target | Format | Size | Purpose |
|--------|--------|------|---------|
| `sceneColor` | RGBA16F or RGBA8 | Full | Scene render (after MSAA resolve) |
| `bloomExtract` | RGBA8 | ½ | Extracted bloom pixels |
| `bloomMip[0]` | RGBA8 | ½ | First blur mip |
| `bloomMip[1]` | RGBA8 | ¼ | Second blur mip |
| `bloomMip[2]` | RGBA8 | ⅛ | Third blur mip |
| `bloomMip[3]` | RGBA8 | 1/16 | Fourth blur mip |
| `bloomTemp` | RGBA8 | varies | Temp for separable blur passes |

Using RGBA8 for bloom mips saves memory (vs RGBA16F) and is sufficient since the bloom values are already in a reasonable range after extraction.

## Post-Processing Pipeline Manager

```typescript
interface PostProcessor {
  enabled: boolean
  bloom: BloomConfig
  toneMapping: ToneMappingConfig

  // Called once when canvas resizes
  resize(width: number, height: number, device: GPUDevice): void

  // Called each frame after scene render
  apply(device: GPUDevice, sceneColorTarget: GPURenderTargetHandle): void
}

interface BloomConfig {
  enabled: boolean          // default: true
  threshold: number         // default: 0.0 (any emissive blooms)
  intensity: number         // default: 1.0
  radius: number            // default: 0.85 (per-level blend factor)
  strength: number          // default: 0.5 (final composite strength)
  levels: number            // default: 4 (mip levels)
}

interface ToneMappingConfig {
  enabled: boolean          // default: true
  exposure: number          // default: 1.0
  method: 'aces' | 'reinhard' | 'none'  // default: 'aces'
}
```

## Full-Screen Quad

All post-processing passes render a full-screen triangle (more efficient than a quad — avoids the diagonal seam):

```glsl
// fullscreen.vert — no vertex buffer needed
void main() {
  // Generate full-screen triangle from vertex ID
  vec2 pos = vec2(
    float((gl_VertexID << 1) & 2) * 2.0 - 1.0,
    float(gl_VertexID & 2) * 2.0 - 1.0
  );
  v_uv = pos * 0.5 + 0.5;
  gl_Position = vec4(pos, 0.0, 1.0);
}
```

This shader generates a single triangle that covers the entire screen using just `gl_VertexID` — no vertex buffer allocation needed.

## Performance Considerations

- **Bloom mips at reduced resolution**: Each level is ½ the previous, so total bloom work is roughly 1.33× the cost of a single full-res pass — very efficient
- **Separable blur**: 9 texture samples instead of 81 for a 9×9 kernel
- **Skip bloom if no emissive**: If no objects in the scene have emissive, the bloom pass is skipped entirely (checked via a `_hasEmissive` flag on the scene)
- **Disable individually**: Bloom and tone mapping can be toggled independently
- **Mobile**: On mobile GPUs, bloom levels can be reduced to 2-3 for better performance
