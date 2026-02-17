# POST-PROCESSING - Design Decisions

## Bloom Algorithm

**Decision: Unreal-style progressive downsample/upsample with 13-tap Jimenez filter**

- Sources: 13-tap from Hyena, Wren (best quality); Unreal-style from all 9
- Rejected: Separable Gaussian (Caracal, Lynx, Rabbit, Shark) — more passes, less efficient
- Rejected: Dual Kawase (Fennec) — good perf/quality but less standard

Pipeline:
```
Emissive RT → Karis Average → 13-tap Downsample × 5 → Tent Upsample × 5 → Composite
```

## Downsample Filter

**Decision: 13-tap Jimenez filter (Call of Duty: Advanced Warfare technique)**

- Sources: Hyena, Wren (2/9)

The 13-tap filter samples in a cross pattern using 4 bilinear fetches, producing high-quality downsampling with good anti-aliasing characteristics. More visually stable than box or Gaussian alternatives.

## First Downsample: Karis Average

**Decision: Karis weighted average on first downsample pass to prevent fireflies**

- Sources: Caracal, Hyena, Lynx, Mantis, Rabbit, Wren (6/9)

```glsl
// Weight by 1 / (1 + luminance) to suppress bright pixel dominance
float w = 1.0 / (1.0 + luminance(sample));
```

Critical for preventing single bright emissive pixels from blooming into large harsh artifacts.

## Upsample Filter

**Decision: 3×3 tent filter (9-tap)**

- Sources: Hyena, Lynx, Mantis, Rabbit, Shark, Wren (6/9)

```glsl
// 3×3 tent filter for smooth upsample
// 9 samples with tent-shaped weights
vec3 upsample = (
  1.0 * textureSample(bloom, uv + vec2(-1, -1) * texelSize) +
  2.0 * textureSample(bloom, uv + vec2( 0, -1) * texelSize) +
  1.0 * textureSample(bloom, uv + vec2( 1, -1) * texelSize) +
  2.0 * textureSample(bloom, uv + vec2(-1,  0) * texelSize) +
  4.0 * textureSample(bloom, uv) +
  2.0 * textureSample(bloom, uv + vec2( 1,  0) * texelSize) +
  1.0 * textureSample(bloom, uv + vec2(-1,  1) * texelSize) +
  2.0 * textureSample(bloom, uv + vec2( 0,  1) * texelSize) +
  1.0 * textureSample(bloom, uv + vec2( 1,  1) * texelSize)
) / 16.0;
```

Additive blending during upsample to accumulate bloom across mip levels.

## Bloom Levels

**Decision: 5 downsample levels**

- Sources: Bonobo, Caracal, Hyena, Lynx, Mantis, Wren (6/9) use 5 levels

5 levels provides wide glow spread (1/32 resolution at lowest level) while keeping pass count reasonable. Total bloom chain memory ≈ 1.33× base resolution.

## Starting Resolution

**Decision: Half-resolution start**

- Sources: Fennec, Hyena, Mantis (3/9)

Begin bloom chain at canvas/2 resolution. First downsample from emissive MRT is already a 2× reduction. Saves one full-resolution pass and reduces total bloom cost significantly.

## Emissive Input

**Decision: MRT emissive output from opaque pass (no threshold pass needed)**

- Sources: Hyena, Rabbit, Shark, Wren (4/9) use MRT; Mantis uses emissive-only buffer

During the opaque pass, the fragment shader writes emissive contribution to a second render target (RT1). Bloom extraction reads this directly — no luminance threshold pass needed. Only actually emissive geometry contributes to bloom, giving precise artist control via the material index palette.

## Threshold Handling

**Decision: No global threshold (physically based, emissive-driven)**

- Sources: Fennec, Hyena, Mantis, Wren (4/9)

Default threshold 0.0 — any emissive value contributes to bloom. Since bloom is driven by the dedicated emissive MRT (not by the scene color buffer), a threshold is unnecessary. Emissive intensity in the material palette directly controls bloom strength per material index.

## Render Target Format

**Decision: RGBA16F for bloom chain**

- Sources: Fennec, Hyena, Lynx, Mantis, Rabbit, Wren (6/9)
- Rejected: RGBA8 (Caracal, Shark) — loses HDR precision, causes banding

RGBA16F preserves dynamic range through the downsample/upsample chain. Total bloom chain memory at 1080p: ~2.5-4MB (geometric series of half-res mips).

## Tone Mapping

**Decision: ACES filmic tone mapping**

- Sources: Caracal, Lynx, Rabbit (3/9 implemented); Hyena, Mantis (2/9 planned)

```glsl
// Stephen Hill's ACES approximation
vec3 acesFilm(vec3 x) {
  float a = 2.51;
  float b = 0.03;
  float c = 2.43;
  float d = 0.59;
  float e = 0.14;
  return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}
```

ACES provides a film-like look with good highlight rolloff and shadow detail. Applied in the final blit pass after bloom composite.

## MSAA Integration

**Decision: MSAA resolved before post-processing (universal agreement)**

Opaque pass renders to MSAA render targets. MSAA resolve happens before bloom processing. Bloom operates on resolved 1x textures to save memory and simplify shaders.

## Full-Screen Triangle

**Decision: Single oversized triangle for all post-processing passes**

- Sources: All 9 implementations agree

```glsl
// Vertex shader: no vertex buffer needed
vec2 pos = vec2((gl_VertexIndex << 1) & 2, gl_VertexIndex & 2);
gl_Position = vec4(pos * 2.0 - 1.0, 0.0, 1.0);
uv = pos;
```

Single triangle eliminates the diagonal seam artifact that occurs with a screen-aligned quad. Slightly more overdraw but negligible cost.

## Bloom Configuration API

```typescript
const engine = await createEngine(canvas, {
  bloom: {
    enabled: true,
    intensity: 0.5,        // bloom strength in final composite
    radius: 0.85,          // blur spread factor
    levels: 5,             // downsample levels
  },
  toneMapping: 'aces',     // 'aces' | 'reinhard' | 'none'
})
```

## Performance Budget

At 1080p:
- Bloom (5 down + 5 up + composite): ~0.5-1.5ms GPU
- Tone mapping: ~0.1ms GPU
- Total post-processing: ~0.6-1.6ms GPU

Target: <1.5ms total post-processing on mobile (with half-res bloom start).
