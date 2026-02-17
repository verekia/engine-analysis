# 09 — Per-Vertex Bloom (Unreal-Style Post-Process)

## Overview

Wren implements a physically-based bloom pipeline inspired by Unreal Engine and the Jimenez (Call of Duty: Advanced Warfare) technique. Bloom is driven by per-vertex emissive values, set either per-material or per material-index region, making it easy to specify "this part of the mesh glows" without separate draw calls or textures.

## Emissive Source

### Per-Material Emissive

```ts
const glowingMaterial = createLambertMaterial({
  color: [0.0, 0.8, 1.0],
  emissive: [0.0, 0.8, 1.0],
  emissiveIntensity: 2.0,
})
```

### Per Material-Index Emissive

```ts
const multiMaterial = createLambertMaterial({
  useMaterialIndex: true,
  materialIndexColors: new Float32Array([
    1.0, 1.0, 1.0,            // Index 0: white body
    0.0, 0.0, 0.0,            // Index 1: black trim
    0.0, 0.7, 0.7,            // Index 2: teal lights
  ]),
  materialIndexEmissive: new Float32Array([
    0.0, 0.0, 0.0, 0.0,       // Index 0: no glow
    0.0, 0.0, 0.0, 0.0,       // Index 1: no glow
    0.0, 0.7, 0.7, 0.7,       // Index 2: teal glow, intensity 0.7
  ]),
})
```

Vertices with material index 2 will glow teal and contribute to the bloom post-process.

## Render Target Setup

The opaque and transparent passes write to two render targets simultaneously via MRT (Multiple Render Targets):

```
Render Target 0 (RGBA16F): Scene color (lit + emissive added to color)
Render Target 1 (RGBA16F): Emissive only (for bloom input)
```

The fragment shader writes:

```glsl
layout(location = 0) out vec4 fragColor;     // Full lit color
layout(location = 1) out vec4 fragEmissive;   // Emissive contribution only

void main() {
  vec3 litColor = computeLighting(baseColor, normal, shadow);
  vec3 emissive = computeEmissive();   // From material + material index

  fragColor = vec4(litColor + emissive, opacity);
  fragEmissive = vec4(emissive, 1.0);  // Only the glow part
}
```

When bloom is disabled, the engine skips RT1 and the post-process entirely.

## Bloom Pipeline

The bloom pipeline processes the emissive render target through a progressive downsample/upsample chain:

```
Emissive RT (half-res)
    │
    ▼
┌──────────────┐
│  Downsample  │  Mip 0: 1/2 res
│  (13-tap)    │  Mip 1: 1/4 res
│              │  Mip 2: 1/8 res
│              │  Mip 3: 1/16 res
│              │  Mip 4: 1/32 res
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Upsample   │  Mip 4 → Mip 3 (blend)
│  (tent 3x3)  │  Mip 3 → Mip 2 (blend)
│              │  Mip 2 → Mip 1 (blend)
│              │  Mip 1 → Mip 0 (blend)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Composite   │  Scene + Bloom * intensity
└──────────────┘
```

### Step 1: Downsample (Jimenez 13-Tap Filter)

Each downsample step reduces resolution by 2x using a high-quality 13-tap filter that samples a 4x4 neighborhood using bilinear filtering:

```glsl
// Jimenez 13-tap downsample
// Samples arranged as:
//   a . b . c
//   . d . e .
//   f . g . h
//   . i . j .
//   k . l . m
// Where each letter is a bilinear tap covering 2x2 texels

vec3 downsample13Tap(sampler2D tex, vec2 uv, vec2 texelSize) {
  vec3 a = texture(tex, uv + texelSize * vec2(-2, -2)).rgb;
  vec3 b = texture(tex, uv + texelSize * vec2( 0, -2)).rgb;
  vec3 c = texture(tex, uv + texelSize * vec2( 2, -2)).rgb;
  vec3 d = texture(tex, uv + texelSize * vec2(-1, -1)).rgb;
  vec3 e = texture(tex, uv + texelSize * vec2( 1, -1)).rgb;
  vec3 f = texture(tex, uv + texelSize * vec2(-2,  0)).rgb;
  vec3 g = texture(tex, uv                           ).rgb;
  vec3 h = texture(tex, uv + texelSize * vec2( 2,  0)).rgb;
  vec3 i = texture(tex, uv + texelSize * vec2(-1,  1)).rgb;
  vec3 j = texture(tex, uv + texelSize * vec2( 1,  1)).rgb;
  vec3 k = texture(tex, uv + texelSize * vec2(-2,  2)).rgb;
  vec3 l = texture(tex, uv + texelSize * vec2( 0,  2)).rgb;
  vec3 m = texture(tex, uv + texelSize * vec2( 2,  2)).rgb;

  // Weighted combination (anti-firefly weights)
  vec3 result = g * 0.125;                              // Center: 1/8
  result += (d + e + i + j) * 0.125;                    // Inner ring: 4 * 1/8 = 1/2
  result += (a + b + f + g) * 0.03125;                  // Quadrant UL: 1/32
  result += (b + c + g + h) * 0.03125;                  // Quadrant UR: 1/32
  result += (f + g + k + l) * 0.03125;                  // Quadrant LL: 1/32
  result += (g + l + h + m) * 0.03125;                  // Quadrant LR: 1/32
  // Note: g appears in multiple quadrants, total weight sums to ~1.0

  return result;
}
```

The 13-tap filter is critical for suppressing "firefly" artifacts (single bright pixels dominating the bloom). The weighted average across a wide neighborhood naturally attenuates outliers.

### Step 2: Upsample (3x3 Tent Filter)

Each upsample step doubles resolution and blends with the downsample result at that level:

```glsl
vec3 upsample3x3Tent(sampler2D tex, vec2 uv, vec2 texelSize) {
  vec3 a = texture(tex, uv + texelSize * vec2(-1, -1)).rgb;
  vec3 b = texture(tex, uv + texelSize * vec2( 0, -1)).rgb;
  vec3 c = texture(tex, uv + texelSize * vec2( 1, -1)).rgb;
  vec3 d = texture(tex, uv + texelSize * vec2(-1,  0)).rgb;
  vec3 e = texture(tex, uv                           ).rgb;
  vec3 f = texture(tex, uv + texelSize * vec2( 1,  0)).rgb;
  vec3 g = texture(tex, uv + texelSize * vec2(-1,  1)).rgb;
  vec3 h = texture(tex, uv + texelSize * vec2( 0,  1)).rgb;
  vec3 i = texture(tex, uv + texelSize * vec2( 1,  1)).rgb;

  // Tent filter weights
  vec3 result = e * 4.0;
  result += (b + d + f + h) * 2.0;
  result += (a + c + g + i) * 1.0;
  result *= (1.0 / 16.0);

  return result;
}
```

At each upsample level, the result is blended with the existing downsample mip:

```glsl
vec3 bloomAtLevel = mix(downsampleMip[level], upsampledFromBelow, u_bloomRadius);
```

The `bloomRadius` parameter (0-1) controls how much the lower-resolution (wider) blur contributes. Higher values = wider bloom.

### Step 3: Composite

The final bloom result is added to the scene color:

```glsl
vec3 sceneColor = texture(u_sceneRT, v_uv).rgb;
vec3 bloomColor = texture(u_bloomRT, v_uv).rgb;
vec3 finalColor = sceneColor + bloomColor * u_bloomIntensity;

// Tonemap (simple Reinhard or ACES)
finalColor = finalColor / (finalColor + 1.0);

// Gamma correction (if not using sRGB render target)
fragColor = vec4(pow(finalColor, vec3(1.0 / 2.2)), 1.0);
```

## Configuration

```ts
interface BloomConfig {
  enabled: boolean          // Default: true
  intensity: number         // Bloom strength multiplier. Default: 0.5
  radius: number            // Bloom spread (0-1). Default: 0.85
  threshold: number         // Min emissive to bloom. Default: 0 (physically based)
  levels: number            // Downsample levels. Default: 5
}
```

### No Threshold (Physically Based)

By default, `threshold` is 0 — all emissive light contributes to bloom. This is more physically correct (real lens diffraction affects all light). Very bright emissive regions naturally dominate the bloom because the downsample chain preserves relative brightness.

If desired, a threshold can be set to only bloom emissive values above a certain intensity, but this creates a non-physical "hard glow" look.

## Performance

| Pass | Resolution | Samples | Cost (1080p) |
|------|-----------|---------|-------------|
| Downsample 0 | 960x540 | 13-tap | ~0.05ms |
| Downsample 1 | 480x270 | 13-tap | ~0.02ms |
| Downsample 2 | 240x135 | 13-tap | ~0.01ms |
| Downsample 3 | 120x68 | 13-tap | ~0.005ms |
| Downsample 4 | 60x34 | 13-tap | ~0.002ms |
| Upsample 4→0 | 4 passes | 9-tap | ~0.08ms |
| Composite | 1920x1080 | 2 taps | ~0.05ms |
| **Total** | | | **~0.22ms** |

Bloom is nearly free because most work happens at very low resolutions.

## Render Target Allocation

```ts
interface BloomResources {
  downsampleChain: GPUTextureHandle[]  // 5 textures at decreasing resolution
  upsampleChain: GPUTextureHandle[]    // 5 textures (can alias with downsample after use)
  emissiveRT: GPUTextureHandle         // Half-res input from scene pass

  downsamplePipeline: PipelineHandle
  upsamplePipeline: PipelineHandle
  compositePipeline: PipelineHandle
}
```

All bloom textures use `RGBA16F` format for HDR precision. At 1080p, the total memory for the mip chain is approximately:
- 960x540 + 480x270 + 240x135 + 120x68 + 60x34 = ~2.5 MB

This is modest and can be shared with other post-effects.
