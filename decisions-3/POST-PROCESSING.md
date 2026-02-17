# POST-PROCESSING.md - Post-Processing Pipeline

## Bloom Implementation

**Decision: Unreal-style progressive downsample/upsample with MRT emissive** (universal agreement)

### Pipeline Overview

```
Opaque Pass (MRT) -> Emissive RT -> Downsample Chain -> Upsample Chain -> Composite
```

1. During the opaque pass, emissive values are written to a second render target (MRT output 1)
2. The emissive RT feeds the bloom pipeline
3. Progressive downsample (5 levels)
4. Progressive upsample with additive blending
5. Final composite: add bloom result onto scene color

### Why MRT Emissive Over Luminance Threshold

**Decision: No threshold, MRT-driven bloom** (Hyena, Mantis, Wren approach)

With a separate emissive render target, only explicitly emissive surfaces bloom. No threshold tuning is needed. Non-emissive bright surfaces (white walls in sunlight) do not bloom spuriously.

This is cleaner than threshold-based extraction (Lynx, Rabbit), which requires tweaking a magic number per scene and can cause unintended bloom on bright non-emissive surfaces.

### Downsample Filter

**Decision: 13-tap Jimenez filter with Karis average on first pass** (Hyena, Wren approach)

The 13-tap filter (from Call of Duty: Advanced Warfare / Jimenez 2014) provides high quality with 4 bilinear samples per downsample:

```glsl
// 13-tap downsample (4 bilinear fetches + 5 point fetches)
vec3 a = texture(src, uv + vec2(-2, -2) * texelSize).rgb;
vec3 b = texture(src, uv + vec2( 0, -2) * texelSize).rgb;
// ... (13 samples total)
color = (a + b + c + d) * 0.125 + (e + f + g + h) * 0.0625 + i * 0.125;
```

**Karis average on first downsample** prevents firefly artifacts from extremely bright pixels:

```glsl
float weight = 1.0 / (1.0 + luminance(color));
```

6/8 implementations agree Karis average is necessary on the first downsample.

### Upsample Filter

**Decision: 3x3 tent filter with additive blending** (Hyena, Lynx, Mantis, Rabbit, Shark, Wren - 6/8 use tent)

```glsl
// 9-tap tent (3x3 bilinear)
vec3 result =
  texture(src, uv + vec2(-1, -1) * texelSize).rgb * 0.0625 +
  texture(src, uv + vec2( 0, -1) * texelSize).rgb * 0.125  +
  texture(src, uv + vec2( 1, -1) * texelSize).rgb * 0.0625 +
  texture(src, uv + vec2(-1,  0) * texelSize).rgb * 0.125  +
  texture(src, uv + vec2( 0,  0) * texelSize).rgb * 0.25   +
  texture(src, uv + vec2( 1,  0) * texelSize).rgb * 0.125  +
  texture(src, uv + vec2(-1,  1) * texelSize).rgb * 0.0625 +
  texture(src, uv + vec2( 0,  1) * texelSize).rgb * 0.125  +
  texture(src, uv + vec2( 1,  1) * texelSize).rgb * 0.0625;
```

Each upsample level is additively blended with the next higher-resolution level, creating a wide, smooth glow.

### Downsample Levels

**Decision: 5 levels** (Caracal, Hyena, Lynx, Mantis, Wren - most common choice)

```
Level 0: canvas/2    (e.g., 960x540 at 1080p)
Level 1: canvas/4    (480x270)
Level 2: canvas/8    (240x135)
Level 3: canvas/16   (120x68)
Level 4: canvas/32   (60x34)
```

5 levels provide a wide glow spread. Memory cost: ~1.33x the base resolution (geometric series). Total for RGBA16F: ~5MB at 1080p.

### Starting Resolution

**Decision: Half-resolution start** (Fennec, Hyena, Mantis approach)

Start the bloom chain at canvas/2 instead of full resolution. This halves the cost of the first (most expensive) downsample and is imperceptible for bloom quality since bloom is inherently a low-frequency effect.

### Render Target Format

**Decision: RGBA16F for bloom chain** (Fennec, Hyena, Lynx, Mantis, Rabbit, Wren - 6/8 use this)

RGBA16F preserves HDR values throughout the bloom chain, preventing banding and clamping artifacts. RGBA8 (Caracal, Shark) saves memory but loses precision for bright emissive values.

At 1080p, the full bloom chain costs ~5MB in RGBA16F vs ~2.5MB in RGBA8. The 2.5MB difference is negligible compared to other GPU memory consumers (textures, geometry, shadow maps).

## Full-Screen Triangle

**Decision: Oversized triangle instead of quad** (universal agreement)

Post-processing passes use a single triangle that covers the entire screen:

```glsl
// Vertex shader - no vertex buffer needed
vec2 uv = vec2((gl_VertexIndex << 1) & 2, gl_VertexIndex & 2);
gl_Position = vec4(uv * 2.0 - 1.0, 0.0, 1.0);
v_uv = uv;
```

This avoids the diagonal seam artifact that occurs with a two-triangle quad.

## Tone Mapping

**Decision: ACES Filmic** (Caracal, Lynx, Rabbit, and planned in Hyena, Mantis - most popular choice)

Stephen Hill's fit:

```glsl
vec3 toneMapACES(vec3 x) {
  const float a = 2.51;
  const float b = 0.03;
  const float c = 2.43;
  const float d = 0.59;
  const float e = 0.14;
  return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 0.0);
}
```

ACES provides a film-like look with good highlight rolloff and shadow lift. It is the industry standard for game tone mapping.

### Why Not Reinhard

Reinhard (Shark, Wren) is simpler but washes out highlights and does not produce the film-like contrast curve that ACES provides. For a stylized game engine where visual quality matters, ACES is the better default.

## MSAA Integration

**Decision: MSAA resolve before bloom** (universal agreement)

MSAA is resolved from the multisample render target to a single-sample texture before any post-processing. Bloom operates on the resolved (1x) texture. This saves memory (bloom chain at 1x, not 4x) and is visually equivalent since bloom is a low-frequency effect.

## Bloom Configuration API

```typescript
{
  bloom: {
    enabled: true,
    intensity: 0.5,    // Bloom strength in final composite
    radius: 0.85,      // Upsample blend radius
    levels: 5,         // Downsample levels
  }
}
```

Threshold is intentionally omitted since MRT-driven bloom does not need one. The material palette's emissive intensity controls what blooms and how much.

## Performance Budget

| Pass | Cost (1080p) |
|------|-------------|
| MSAA Resolve | ~0.1ms |
| Downsample (5 levels) | ~0.3ms |
| Upsample (5 levels) | ~0.3ms |
| Composite + Tone Map | ~0.1ms |
| **Total post-processing** | **~0.8ms GPU** |

Starting at half resolution and using efficient filters keeps bloom under 1ms at 1080p.
