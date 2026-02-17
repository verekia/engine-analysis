# Bloom (Unreal Engine Style)

## Overview

Mantis implements the same bloom algorithm used by Unreal Engine 4/5: a
progressive downsample chain followed by a progressive upsample chain with
additive blending. This produces a smooth, physically-motivated glow that
expands with distance from the emissive source.

The key innovation in Mantis is **per-vertex emissive control** via the
material index palette. Artists tag specific vertices as emissive (glowing
eyes, neon signs, magical effects), and bloom applies exclusively to those
regions — no full-screen threshold needed.

## Pipeline

```
Scene rendering (MRT)
  ├── Color target:    base scene color
  └── Emissive target: only emissive contributions
           │
           ▼
    ┌─────────────┐
    │  Downsample  │  5 levels, each half the previous resolution
    │  (Karis avg) │  Level 0: canvas/2 → Level 4: canvas/32
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │   Upsample   │  5 levels, each double the previous resolution
    │  (Tent filter)│  Level 4 → Level 0, additive blend at each level
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Composite   │  Resolved scene color + bloom result
    └─────────────┘
```

## Per-Vertex Emissive Control

### How It Works

During the opaque rendering pass, the fragment shader writes to two render
targets simultaneously (MRT — Multiple Render Targets):

- **MRT 0 (color):** The final lit color of the fragment
- **MRT 1 (emissive):** Only the emissive contribution

```glsl
// Fragment shader (Lambert with material index)
layout(location = 0) out vec4 outColor;
layout(location = 1) out vec4 outEmissive;

void main() {
  vec3 color = computeLitColor();  // base × lighting × shadows × AO

  // Material index lookup for emissive
  vec3 emissive = vec3(0.0);
  #ifdef HAS_MATERIAL_INDEX
    vec4 emissiveData = materialEmissive[v_materialIndex];
    emissive = emissiveData.rgb * emissiveData.a;  // color × intensity
  #endif

  outColor = vec4(color + emissive, 1.0);
  outEmissive = vec4(emissive, 1.0);
}
```

This approach has a major advantage over traditional bloom: **no threshold
needed.** Traditional bloom applies a brightness threshold to the entire scene,
which can cause unintended glow on bright surfaces (e.g., a white wall in
sunlight). Mantis's emissive target contains only intentionally emissive
pixels — everything else is black.

### Example

```typescript
const material = new LambertMaterial({
  palette: [
    { color: [0.8, 0.7, 0.6] },                                     // index 0: skin (no glow)
    { color: [0.1, 0.1, 0.1] },                                     // index 1: dark clothing (no glow)
    { color: [0, 1, 1], emissive: [0, 1, 1], emissiveIntensity: 0.7 }, // index 2: teal glow
    { color: [1, 0.3, 0], emissive: [1, 0.3, 0], emissiveIntensity: 1.0 }, // index 3: fire glow
  ],
})
```

Vertices with material index 2 will glow teal. Vertices with index 0 or 1 will
never bloom, regardless of lighting conditions.

## Downsample Chain (5 levels)

Starting from the emissive render target (resolved from MSAA), progressively
downsample to increasingly smaller textures:

```
Level 0:  canvas/2   (e.g., 960 × 540 for a 1920×1080 canvas)
Level 1:  canvas/4   (480 × 270)
Level 2:  canvas/8   (240 × 135)
Level 3:  canvas/16  (120 × 68)
Level 4:  canvas/32  (60 × 34)
```

### First Downsample: Karis Average

The first downsample level uses the **Karis average** (also called "weighted
luminance average") to prevent bright pixels from dominating. This eliminates
"firefly" artifacts where a single bright pixel creates an oversized glow:

```glsl
// Karis average — weights samples by inverse luminance
float karisWeight(vec3 color) {
  float luma = dot(color, vec3(0.2126, 0.7152, 0.0722));
  return 1.0 / (1.0 + luma);
}

vec3 downsampleKaris(sampler2D src, vec2 uv, vec2 texelSize) {
  // 13-tap filter (4 bilinear taps + center)
  vec3 a = texture(src, uv + texelSize * vec2(-1, -1)).rgb;
  vec3 b = texture(src, uv + texelSize * vec2( 1, -1)).rgb;
  vec3 c = texture(src, uv + texelSize * vec2(-1,  1)).rgb;
  vec3 d = texture(src, uv + texelSize * vec2( 1,  1)).rgb;
  vec3 e = texture(src, uv).rgb;

  float wa = karisWeight(a);
  float wb = karisWeight(b);
  float wc = karisWeight(c);
  float wd = karisWeight(d);
  float we = karisWeight(e);

  float invTotalWeight = 1.0 / (wa + wb + wc + wd + we);

  return (a * wa + b * wb + c * wc + d * wd + e * we) * invTotalWeight;
}
```

### Subsequent Downsamples: Box Filter

Levels 1–4 use a simple 4-tap box filter (bilinear sampling does most of the
work):

```glsl
vec3 downsampleBox(sampler2D src, vec2 uv, vec2 texelSize) {
  vec3 a = texture(src, uv + texelSize * vec2(-0.5, -0.5)).rgb;
  vec3 b = texture(src, uv + texelSize * vec2( 0.5, -0.5)).rgb;
  vec3 c = texture(src, uv + texelSize * vec2(-0.5,  0.5)).rgb;
  vec3 d = texture(src, uv + texelSize * vec2( 0.5,  0.5)).rgb;
  return (a + b + c + d) * 0.25;
}
```

## Upsample Chain (5 levels)

Starting from the smallest level, progressively upsample back to half canvas
resolution, **additively blending** each level with the previous:

### Tent Filter (3×3)

The tent filter (bilinear kernel) produces smooth transitions between bloom
levels:

```glsl
vec3 upsampleTent(sampler2D src, vec2 uv, vec2 texelSize) {
  // 9-tap tent filter
  vec3 sum = vec3(0.0);

  sum += texture(src, uv + texelSize * vec2(-1, -1)).rgb * 1.0;
  sum += texture(src, uv + texelSize * vec2( 0, -1)).rgb * 2.0;
  sum += texture(src, uv + texelSize * vec2( 1, -1)).rgb * 1.0;
  sum += texture(src, uv + texelSize * vec2(-1,  0)).rgb * 2.0;
  sum += texture(src, uv + texelSize * vec2( 0,  0)).rgb * 4.0;
  sum += texture(src, uv + texelSize * vec2( 1,  0)).rgb * 2.0;
  sum += texture(src, uv + texelSize * vec2(-1,  1)).rgb * 1.0;
  sum += texture(src, uv + texelSize * vec2( 0,  1)).rgb * 2.0;
  sum += texture(src, uv + texelSize * vec2( 1,  1)).rgb * 1.0;

  return sum / 16.0;
}
```

### Additive Blending at Each Level

Each upsample level blends its result with the corresponding downsample level:

```glsl
// Upsample pass N: blend upsampled previous level with downsample level N
vec3 upsampled = upsampleTent(previousUpsampleLevel, uv, texelSize);
vec3 current = texture(downsampleLevel[N], uv).rgb;
outColor = vec4(current + upsampled, 1.0);
```

This progressive accumulation creates the characteristic multi-radius glow:
- Small bright spots get a tight glow (from high-resolution levels)
- Large emissive areas get a wide glow (from low-resolution levels)

## Final Composite

The bloom result (upsample level 0) is additively blended with the resolved
scene color:

```glsl
// Final composite shader
uniform sampler2D sceneColor;      // resolved opaque + OIT composite
uniform sampler2D bloomResult;     // upsample level 0
uniform float bloomIntensity;      // default 0.5

void main() {
  vec2 uv = gl_FragCoord.xy / resolution;
  vec3 scene = texture(sceneColor, uv).rgb;
  vec3 bloom = texture(bloomResult, uv).rgb;

  vec3 result = scene + bloom * bloomIntensity;

  fragColor = vec4(result, 1.0);
}
```

## Render Targets for Bloom

| Target | Format | Size | Purpose |
|---|---|---|---|
| Emissive (MSAA) | RGBA8Unorm | Canvas | MRT 1 during opaque pass |
| Emissive (resolved) | RGBA8Unorm | Canvas | MSAA-resolved emissive |
| Downsample 0 | RGBA8Unorm | Canvas/2 | First downsample |
| Downsample 1 | RGBA8Unorm | Canvas/4 | Second downsample |
| Downsample 2 | RGBA8Unorm | Canvas/8 | Third downsample |
| Downsample 3 | RGBA8Unorm | Canvas/16 | Fourth downsample |
| Downsample 4 | RGBA8Unorm | Canvas/32 | Fifth downsample |
| Upsample 4 | RGBA8Unorm | Canvas/32 | Bottom of upsample chain |
| Upsample 3 | RGBA8Unorm | Canvas/16 | — |
| Upsample 2 | RGBA8Unorm | Canvas/8 | — |
| Upsample 1 | RGBA8Unorm | Canvas/4 | — |
| Upsample 0 | RGBA8Unorm | Canvas/2 | Final bloom result |

Total additional memory: ~1.5× canvas resolution (geometric series sum).

## Performance

The bloom pipeline is highly efficient because:
1. **Half-resolution base** — The chain starts at canvas/2, not full resolution
2. **Geometric series** — Total fragment work: 1/4 + 1/16 + 1/64 + 1/256 +
   1/1024 ≈ 0.33 of full resolution (per chain direction)
3. **Simple shaders** — Each pass is 4–9 texture samples + arithmetic
4. **No threshold pass** — Emissive is isolated at source via MRT

| Step | Fragments (1920×1080 canvas) | Cost estimate |
|---|---|---|
| 5 downsample passes | ~690K total | ~0.4 ms GPU |
| 5 upsample passes | ~690K total | ~0.5 ms GPU |
| Final composite | ~2M (full canvas) | ~0.2 ms GPU |
| **Total** | | **~1.1 ms GPU** |

## Bloom API

```typescript
const engine = await createEngine({
  canvas,
  bloom: true,                  // enable bloom pipeline
  bloomIntensity: 0.5,          // default
  bloomLevels: 5,               // default
})

// Adjust at runtime
engine.bloomIntensity = 0.8
engine.bloomEnabled = false     // disable entirely (skips all bloom passes)
```

Bloom is controlled per-vertex via the material palette — no per-mesh bloom
flags or render layers needed:

```typescript
const material = new LambertMaterial({
  palette: [
    { color: [1, 1, 1] },                                                // no bloom
    { color: [0, 1, 0.5], emissive: [0, 1, 0.5], emissiveIntensity: 0.7 }, // blooms
  ],
})
```
