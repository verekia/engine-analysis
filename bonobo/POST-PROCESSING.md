# Post-Processing

## Status: Minimal for v1

Post-processing is kept minimal in v1. Only bloom is supported initially (single MRT pass).

## Bloom (v1)

Unreal-style bloom implementation:

### Pipeline

1. **Downsample chain** — Generate mip chain from bright areas (5-6 levels)
2. **Upsample + blur** — Gaussian blur while upsampling, accumulate into final result
3. **Composite** — Additive blend bloom result with original scene

### Implementation

```ts
// Render targets
sceneColor:  GPUTexture      // Main render target
bloomMips:   GPUTexture[]    // Downsample chain (5-6 mips)
```

### Threshold

Only bright pixels contribute to bloom:

```wgsl
let luminance = dot(color.rgb, vec3(0.299, 0.587, 0.114));
let bright = max(luminance - threshold, 0.0);
let bloom = color.rgb * bright;
```

Threshold value: ~0.8-1.0 (user-configurable)

### Blur Quality

Use separable Gaussian blur (horizontal + vertical passes) for performance:
- 5-tap or 9-tap kernel
- Bilinear sampling for effective kernel size increase

## MSAA (Future)

Multi-sample anti-aliasing:

- Render to MSAA texture (4x or 8x samples)
- Resolve to single-sample texture for post-processing
- WebGPU native support via `sampleCount` in texture descriptor

```ts
const msaaTexture = device.createTexture({
  size: [width, height],
  sampleCount: 4,
  format: 'rgba8unorm',
  usage: GPUTextureUsage.RENDER_ATTACHMENT
});
```

## Additional Effects (Future - v2+)

### Tone Mapping

- ACES filmic tone mapping
- Reinhard tone mapping
- Exposure control

### Color Grading

- LUT-based color grading (3D texture lookup)
- Saturation, contrast, brightness adjustments

### FXAA / SMAA

- Fast approximate anti-aliasing (if MSAA is too expensive)
- Subpixel morphological anti-aliasing

### Depth of Field

- Bokeh depth of field
- Circle of confusion calculation
- Near/far blur

### Motion Blur

- Per-object motion vectors
- Velocity buffer
- Screen-space motion blur

### Screen-Space Ambient Occlusion (SSAO)

- Depth-based ambient occlusion
- HBAO (Horizon-Based Ambient Occlusion)

### Screen-Space Reflections (SSR)

- Ray marching in screen space
- Depth buffer ray intersection
- Fallback to environment map

## Post-Processing Pipeline Architecture

For v2, a flexible post-processing stack:

```ts
interface PostProcessEffect {
  name: string
  render(input: GPUTexture, output: GPUTexture): void
}

class PostProcessPipeline {
  effects: PostProcessEffect[]

  addEffect(effect: PostProcessEffect): void
  removeEffect(name: string): void
  render(sceneColor: GPUTexture): GPUTexture
}
```

Effects render in sequence, ping-ponging between two temp textures:

```
sceneColor → [effect1] → temp1 → [effect2] → temp2 → [effect3] → final
```

## Performance Target

Post-processing budget: ~2-3ms @ 1080p, ~5-6ms @ 4K

- Bloom: ~1-2ms
- Tone mapping: <0.5ms
- FXAA: ~0.5ms
- Composite: <0.5ms

For v1 with bloom only, budget is easily met.
