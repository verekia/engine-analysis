# POST-PROCESSING — Final Decision

## Bloom

**Decision: Unreal-style progressive downsample/upsample** (universal agreement)

### Input

Bloom is driven by MRT emissive output from the opaque pass — no global threshold pass needed. Only geometry with non-zero emissive values contributes to bloom. This is physically motivated: only light-emitting surfaces glow.

### Downsample Filter

**Decision: 13-tap Jimenez filter with Karis average on first pass** (universal agreement)

The 13-tap downsample kernel combines 4 bilinear samples to approximate a 4×4 box filter with Gaussian weighting, reducing aliasing during downsampling.

On the first downsample step, apply **Karis weighted average** to prevent firefly artifacts:

```glsl
// Karis average: weight by 1 / (1 + luma) to suppress bright outliers
float weight = 1.0 / (1.0 + luminance(color));
```

This prevents single bright pixels from creating disproportionately large bloom halos.

### Upsample Filter

**Decision: 3×3 tent filter (9-tap) with additive blending** (universal agreement)

The tent filter provides smooth upsampling that progressively accumulates glow from each level:

```glsl
vec3 upsample(sampler2D tex, vec2 uv, vec2 texelSize) {
  vec3 result = texture(tex, uv + vec2(-1, -1) * texelSize).rgb * 1.0
              + texture(tex, uv + vec2( 0, -1) * texelSize).rgb * 2.0
              + texture(tex, uv + vec2( 1, -1) * texelSize).rgb * 1.0
              + texture(tex, uv + vec2(-1,  0) * texelSize).rgb * 2.0
              + texture(tex, uv                            ).rgb * 4.0
              + texture(tex, uv + vec2( 1,  0) * texelSize).rgb * 2.0
              + texture(tex, uv + vec2(-1,  1) * texelSize).rgb * 1.0
              + texture(tex, uv + vec2( 0,  1) * texelSize).rgb * 2.0
              + texture(tex, uv + vec2( 1,  1) * texelSize).rgb * 1.0;
  return result / 16.0;
}
```

Each upsample step adds to the previous level, accumulating wider and wider glow.

### Bloom Chain

**Decision: 5 downsample levels, starting at half resolution** (universal agreement)

For 1920×1080 canvas:

```
Level 0: 960×540   (half res)
Level 1: 480×270
Level 2: 240×135
Level 3: 120×67
Level 4: 60×33
```

5 levels provide a wide enough glow radius for the stylized aesthetic. Starting at half resolution reduces the cost of the first (most expensive) downsample pass.

### Render Target Format

**Decision: RGBA16F for bloom chain** (universal agreement)

HDR precision preserves emissive values >1.0 through the bloom pipeline. RGBA8 would clamp bright values and produce flat, unnatural glow.

### Full-Screen Geometry

**Decision: Oversized full-screen triangle** (universal agreement)

A single triangle that covers the full viewport is more efficient than a quad (2 triangles):

```glsl
// Vertex shader (no vertex buffer needed)
vec2 position = vec2(gl_VertexIndex & 1, (gl_VertexIndex >> 1) & 1) * 4.0 - 1.0;
vec2 uv = position * 0.5 + 0.5;
```

## Tone Mapping

**Decision: ACES filmic tone mapping** (universal agreement)

```glsl
vec3 acesFilmic(vec3 x) {
  const float a = 2.51;
  const float b = 0.03;
  const float c = 2.43;
  const float d = 0.59;
  const float e = 0.14;
  return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}
```

Applied after bloom compositing. Converts HDR values to LDR for display. ACES provides a natural film-like response curve that handles both dark and bright values gracefully.

## MSAA Integration

**Decision: MSAA resolved before post-processing** (universal agreement)

The opaque pass renders into 4x MSAA targets. MSAA is resolved to 1x before bloom begins. This means bloom operates on resolved (1x) textures, which is simpler and uses less memory than processing multisampled textures.

OIT buffers are always 1x (no MSAA on transparent pass).

## Configuration

```typescript
const engine = await createEngine(canvas, {
  bloom: {
    enabled: true,
    intensity: 0.5,    // Bloom mix strength (0-1)
    levels: 5,         // Downsample levels
  },
  toneMapping: 'aces',   // 'aces' | 'reinhard' | 'none'
})
```

Boolean shorthand: `bloom: true` uses default settings.

## Performance Budget

```
Bloom downsample (5 levels):  ~0.3-0.5ms GPU
Bloom upsample (5 levels):   ~0.2-0.4ms GPU
Tone mapping + final blit:    ~0.1-0.2ms GPU
────────────────────────────────────────────
Total post-processing:        ~0.6-1.1ms GPU
```
