# Bloom (Unreal-Style)

## Overview

Kestrel implements an **Unreal Engine-style bloom** pipeline. Emissive fragments glow and bleed light into surrounding pixels. The bloom intensity is controlled per-vertex through the material index system — specific parts of a mesh (eyes, neon signs, magic effects) emit light while the rest doesn't.

## Pipeline

```
Emissive Buffer (from geometry pass MRT)
         │
         ▼
┌─────────────────┐
│  Threshold Pass  │  Extract pixels above brightness threshold
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Downsample ×5   │  Progressive half-resolution blur
│  (Mip 1→5)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Upsample ×5     │  Tent filter, accumulate back up
│  (Mip 5→0)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Composite       │  Additive blend onto scene color
└─────────────────┘
```

## Step 1: Emissive Buffer

During the geometry pass, the fragment shader writes emissive values to a second color attachment (MRT — Multiple Render Targets):

```wgsl
// Fragment output
struct FragmentOutput {
  @location(0) color: vec4<f32>,     // lit scene color
  @location(1) emissive: vec4<f32>,  // emissive only (for bloom)
}

// In the fragment shader:
var emissiveOut = vec3(0.0);
if hasMaterialIndex {
  let entry = materialMap[materialIndex];
  emissiveOut = entry.emissive.rgb * entry.emissive.a;
}
output.emissive = vec4(emissiveOut, 1.0);
```

Only vertices with non-zero emissive intensity (via material index map) contribute to the emissive buffer. Everything else is black.

## Step 2: Threshold Pass

Extract bright pixels from the emissive buffer. This is typically a simple brightness test, but since we already have a dedicated emissive buffer (not extracting from the scene color), the threshold is straightforward:

```wgsl
@fragment
fn thresholdFragment(@location(0) uv: vec2<f32>) -> @location(0) vec4<f32> {
  let emissive = textureSample(emissiveTexture, sampler, uv).rgb;
  let brightness = dot(emissive, vec3(0.2126, 0.7152, 0.0722));  // luminance
  let threshold = 0.1;  // configurable

  if brightness < threshold {
    return vec4(0.0, 0.0, 0.0, 1.0);
  }
  return vec4(emissive * smoothstep(threshold, threshold + 0.1, brightness), 1.0);
}
```

The `smoothstep` at the threshold boundary prevents harsh cutoff artifacts.

## Step 3: Downsample Chain

5 levels of progressive downsampling, each at half the resolution of the previous:

```
Level 0: full resolution (or half — from emissive buffer)
Level 1: 1/2 resolution
Level 2: 1/4 resolution
Level 3: 1/8 resolution
Level 4: 1/16 resolution
Level 5: 1/32 resolution
```

Each downsample step uses a **13-tap filter** (Karis average on the first downsample to prevent fireflies, simple bilinear on subsequent levels):

```wgsl
// First downsample: Karis average to prevent firefly artifacts
@fragment
fn downsampleKaris(@location(0) uv: vec2<f32>) -> @location(0) vec4<f32> {
  let texelSize = 1.0 / vec2<f32>(textureDimensions(srcTexture));

  // 13-tap pattern (4 bilinear samples covering a 4×4 texel area)
  let a = textureSample(src, samp, uv + texelSize * vec2(-1.0, -1.0));
  let b = textureSample(src, samp, uv + texelSize * vec2( 1.0, -1.0));
  let c = textureSample(src, samp, uv + texelSize * vec2(-1.0,  1.0));
  let d = textureSample(src, samp, uv + texelSize * vec2( 1.0,  1.0));
  let e = textureSample(src, samp, uv);

  // Karis average: weight by 1/(1+luma) to prevent bright spots from dominating
  let wa = 1.0 / (1.0 + luminance(a.rgb));
  let wb = 1.0 / (1.0 + luminance(b.rgb));
  let wc = 1.0 / (1.0 + luminance(c.rgb));
  let wd = 1.0 / (1.0 + luminance(d.rgb));
  let we = 1.0 / (1.0 + luminance(e.rgb));

  return (a*wa + b*wb + c*wc + d*wd + e*we) / (wa + wb + wc + wd + we);
}

// Subsequent downsamples: simple box filter
@fragment
fn downsampleBox(@location(0) uv: vec2<f32>) -> @location(0) vec4<f32> {
  let texelSize = 1.0 / vec2<f32>(textureDimensions(srcTexture));
  let a = textureSample(src, samp, uv + texelSize * vec2(-0.5, -0.5));
  let b = textureSample(src, samp, uv + texelSize * vec2( 0.5, -0.5));
  let c = textureSample(src, samp, uv + texelSize * vec2(-0.5,  0.5));
  let d = textureSample(src, samp, uv + texelSize * vec2( 0.5,  0.5));
  return (a + b + c + d) * 0.25;
}
```

## Step 4: Upsample Chain

Upsample from the smallest mip back up, accumulating blur at each level using a **tent filter** (3×3 bilinear, which approximates a Gaussian):

```wgsl
@fragment
fn upsampleTent(@location(0) uv: vec2<f32>) -> @location(0) vec4<f32> {
  let texelSize = 1.0 / vec2<f32>(textureDimensions(srcTexture));

  // 9-tap tent filter
  var result = vec4(0.0);
  result += textureSample(src, samp, uv + texelSize * vec2(-1.0, -1.0)) * 1.0;
  result += textureSample(src, samp, uv + texelSize * vec2( 0.0, -1.0)) * 2.0;
  result += textureSample(src, samp, uv + texelSize * vec2( 1.0, -1.0)) * 1.0;
  result += textureSample(src, samp, uv + texelSize * vec2(-1.0,  0.0)) * 2.0;
  result += textureSample(src, samp, uv + texelSize * vec2( 0.0,  0.0)) * 4.0;
  result += textureSample(src, samp, uv + texelSize * vec2( 1.0,  0.0)) * 2.0;
  result += textureSample(src, samp, uv + texelSize * vec2(-1.0,  1.0)) * 1.0;
  result += textureSample(src, samp, uv + texelSize * vec2( 0.0,  1.0)) * 2.0;
  result += textureSample(src, samp, uv + texelSize * vec2( 1.0,  1.0)) * 1.0;
  result /= 16.0;

  // Accumulate with the current mip level
  let current = textureSample(currentMip, samp, uv);
  return current + result * bloomIntensity;
}
```

The upsample chain accumulates progressively larger blur radii, creating the characteristic multi-scale glow of Unreal's bloom.

## Step 5: Composite

The final bloom texture is additively blended onto the scene color:

```wgsl
@fragment
fn bloomComposite(@location(0) uv: vec2<f32>) -> @location(0) vec4<f32> {
  let sceneColor = textureSample(colorTexture, samp, uv);
  let bloom = textureSample(bloomTexture, samp, uv);
  return vec4(sceneColor.rgb + bloom.rgb * bloomStrength, sceneColor.a);
}
```

## Per-Vertex Control via Material Index

The key feature is per-vertex bloom control. Example setup:

```typescript
const material = new LambertMaterial({
  materialIndexMap: [
    { color: [0.8, 0.8, 0.8], emissive: [0, 0, 0], emissiveIntensity: 0 },
    { color: [0, 0, 0],       emissive: [0, 0, 0], emissiveIntensity: 0 },
    { color: [0, 0.6, 0.6],   emissive: [0, 1, 1], emissiveIntensity: 0.7 },
  ]
})
// Material index 0: white, no glow
// Material index 1: black, no glow
// Material index 2: teal, glows cyan at 70% intensity
```

The `_materialindex` vertex attribute determines which entry each vertex uses. The emissive values are written to the emissive MRT attachment, which feeds directly into the bloom pipeline.

## Configuration API

```typescript
const engine = await Engine.create({
  canvas,
  bloom: {
    enabled: true,
    threshold: 0.1,        // luminance threshold for extraction
    intensity: 1.0,        // per-level accumulation intensity
    strength: 0.5,         // final composite strength
    mipLevels: 5,          // downsample levels (3–6)
    radius: 1.0,           // blur radius multiplier
  }
})
```

## Performance

- **5 downsample + 5 upsample = 10 full-screen passes**, but each at progressively smaller resolutions.
- Total fragment cost is approximately **2.5× one full-resolution pass** (geometric series: 1 + 1/4 + 1/16 + ...).
- On mobile, the bloom chain starts at **half resolution** (the emissive buffer is already rendered at full res, but the first downsample is free).
- All bloom textures use `RGBA8Unorm` — no expensive floating-point textures needed since the emissive values are already in [0, 1] range (controlled by `emissiveIntensity`).
- The threshold pass and composite pass can be merged with other full-screen passes to reduce draw calls.
