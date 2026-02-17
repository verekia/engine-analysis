# Transparency — Weighted Blended Order-Independent Transparency (OIT)

## Overview

Fennec uses **Weighted Blended Order-Independent Transparency (OIT)** to eliminate the sorting headaches that plague traditional alpha blending. Transparent objects can be rendered in any order and the result is always correct (within the limits of the approximation).

## Why OIT

Three.js transparency is notoriously painful because it requires:
- Sorting transparent objects back-to-front
- Sorting fails with intersecting/overlapping transparent objects
- Manual `renderOrder` tweaking
- Separate render passes for complex cases

Weighted Blended OIT eliminates all of this. Transparent objects can be rendered in any order and the result is always correct (within the limits of the approximation).

## How It Works

The technique (McGuire and Bavoil, 2013) uses two render targets during the transparent pass:

1. **Accumulation buffer** (RGBA16F): Stores weighted sum of `color * alpha * weight`
2. **Revealage buffer** (R8): Stores the product of `(1 - alpha * weight)` — how much of the background shows through

The weight function biases by depth so that closer transparent surfaces contribute more:

```glsl
// Weight function — closer fragments get higher weight
float weight(float z, float alpha) {
  // McGuire 2013 weight function (equation 10)
  return alpha * max(1e-2, min(3e3, 10.0 / (1e-5 + pow(z / 200.0, 4.0))));
}
```

## OIT Accumulation Pass

Render all transparent objects (any order) with special blend modes:

```glsl
// Fragment shader output
layout(location = 0) out vec4 accumulation;  // Weighted color
layout(location = 1) out float revealage;    // Coverage

void main() {
  vec4 color = computeFragColor();  // Normal lighting + material
  float w = weight(gl_FragCoord.z, color.a);

  accumulation = vec4(color.rgb * color.a * w, color.a * w);
  revealage = color.a * w;
}
```

**Blend state for accumulation:**
```
src: ONE, dst: ONE  (additive)
```

**Blend state for revealage:**
```
src: ZERO, dst: ONE_MINUS_SRC_ALPHA  (multiplicative)
```

Initialize revealage buffer to 1.0 (fully revealed = no transparent objects).

## OIT Composite Pass

A full-screen quad reads both buffers and composites over the opaque scene:

```glsl
void main() {
  vec4 accum = texelFetch(accumTexture, ivec2(gl_FragCoord.xy), 0);
  float reveal = texelFetch(revealTexture, ivec2(gl_FragCoord.xy), 0).r;

  // Avoid division by zero
  if (accum.a < 1e-4) discard;

  // Average weighted color
  vec3 avgColor = accum.rgb / max(accum.a, 1e-4);

  // Composite: transparent color blended over opaque
  fragColor = vec4(avgColor, 1.0 - reveal);
}
```

This final result is alpha-blended over the opaque color buffer.

## Material Setup

To enable transparency on a material:

```typescript
const material = new BasicMaterial({
  color: 0x00aaff,
  opacity: 0.5,
  transparent: true,  // Enable OIT
})

// Or with Lambert material
const material = new LambertMaterial({
  color: 0xff00aa,
  opacity: 0.3,
  transparent: true,
  receiveShadow: true,  // Transparent objects can receive shadows
})
```

## Render Pipeline Integration

The render loop separates opaque and transparent objects:

```typescript
// 1. Render opaque objects (front-to-back sorted)
for (const item of opaqueObjects) {
  drawMesh(item)
}

// 2. OIT accumulation pass (transparent objects, any order)
if (transparentObjects.length > 0) {
  beginOITAccumulation()  // Bind accumulation + revealage buffers
  for (const item of transparentObjects) {
    drawMesh(item)
  }
  endOITAccumulation()
  compositeOIT()  // Full-screen quad composite
}
```

No sorting needed for transparent objects!

## OIT Limitations

- **Approximate**: Very thin overlapping layers with similar depth can show artifacts
- **No refraction or distortion**: OIT doesn't support refractive materials
- **Works best with alpha 0.1–0.9**: Very transparent or very opaque objects look fine, but many overlapping semi-transparent layers at similar depths may wash out

For game use cases (windows, water, particle effects, UI elements), these limitations are rarely noticeable.

## MSAA Compatibility

OIT works seamlessly with MSAA. Both the accumulation and revealage buffers are MSAA render targets. They're resolved after the accumulation pass, before the composite pass.

## Performance

| Operation | Cost | Notes |
|-----------|------|-------|
| OIT accumulation pass | ~0.3ms | Render transparent objects with MRT output |
| OIT composite pass | ~0.1ms | Full-screen quad, simple blend |
| **Total OIT cost** | **~0.4ms** | Minimal overhead vs traditional alpha blending |

The overhead is minimal compared to traditional alpha blending with sorting, and the "works without headaches" factor is worth it.

## WebGPU Implementation

WebGPU supports dual-source blending which can simplify OIT, but for broad compatibility Fennec uses the two-buffer approach that works on both WebGL2 and WebGPU.

```wgsl
// WebGPU fragment shader
struct OITOutput {
  @location(0) accumulation: vec4f,
  @location(1) revealage: f32,
}

@fragment
fn fs_main(in: VertexOutput) -> OITOutput {
  let color = computeFragColor(in);
  let w = weight(in.position.z, color.a);

  return OITOutput(
    vec4f(color.rgb * color.a * w, color.a * w),
    color.a * w,
  );
}
```

## Debugging

Debug mode can visualize the OIT buffers:

```typescript
// FENNEC_DEBUG=1
renderer.debugShowOITBuffers = true  // Show accumulation/revealage side-by-side
```

This helps identify issues with weight tuning or over-accumulation.
