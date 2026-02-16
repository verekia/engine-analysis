# Transparency (Weighted Blended OIT)

## The Problem

Transparency in real-time 3D is notoriously difficult because correct blending requires rendering fragments in back-to-front order. Three.js sorts transparent objects by distance, which:

1. Fails for overlapping or intersecting transparent objects
2. Fails for large transparent objects that span the depth range
3. Introduces per-frame sorting cost
4. Has visible popping when sort order changes

## Solution: Weighted Blended Order-Independent Transparency

Kestrel uses **Weighted Blended OIT** (McGuire & Bavoil, 2013). This technique:

- Renders transparent objects in **any order** (no sorting)
- Handles overlapping and intersecting geometry correctly
- Has a fixed cost regardless of complexity
- Requires only **one extra render pass** and two extra buffers
- Works on both WebGPU and WebGL2

## How It Works

### Concept

Instead of blending fragments one at a time in sorted order, WBOIT accumulates all transparent fragments simultaneously using two buffers:

1. **Accumulation buffer** (RGBA16Float): Weighted sum of `color × alpha × weight`
2. **Revealage buffer** (R8): Product of `(1 - alpha × weight)` — how much of the background shows through

A depth-dependent weight function biases closer fragments to contribute more, approximating correct order without sorting.

### Weight Function

```wgsl
fn oitWeight(depth: f32, alpha: f32) -> f32 {
  // Depth-based weight: closer fragments get higher weight
  // This is the key to order-independence
  let z = depth;  // clip-space z, [0, 1]
  return alpha * max(1e-2, min(3e3, 10.0 / (1e-5 + pow(z / 5.0, 2.0) + pow(z / 200.0, 6.0))));
}
```

This weight function is from the original paper. It prioritizes fragments near the camera and falls off with depth. The specific constants are tuned for typical game depth ranges.

## Render Pipeline

### Step 1: Opaque Pass (normal)

Render all opaque meshes to the main color and depth buffer as usual. The depth buffer is populated with opaque geometry.

### Step 2: OIT Pass (transparent)

Render all transparent meshes with **depth testing ON, depth writing OFF**, writing to the accumulation and revealage buffers:

```wgsl
struct OITOutput {
  @location(0) accumulation: vec4<f32>,  // RGBA16Float
  @location(1) revealage: vec4<f32>,     // R8Unorm
}

@fragment
fn oitFragment(...) -> OITOutput {
  // Compute the fragment color as usual (lighting, textures, etc.)
  let color = computeLitColor(...);
  let alpha = material.opacity * color.a;

  // Compute OIT weight
  let depth = fragCoord.z;
  let weight = oitWeight(depth, alpha);

  return OITOutput(
    vec4(color.rgb * alpha * weight, alpha * weight),  // accumulation
    vec4(alpha, 0.0, 0.0, 0.0),                        // revealage (will use ONE_MINUS_SRC blending)
  );
}
```

**Blend states** for the OIT pass:
- **Accumulation** (attachment 0): `srcFactor: ONE, dstFactor: ONE` (additive)
- **Revealage** (attachment 1): `srcFactor: ZERO, dstFactor: ONE_MINUS_SRC_ALPHA` (multiplicative)

These blend modes accumulate contributions from all fragments regardless of order.

### Step 3: OIT Composite

A full-screen pass composites the OIT result onto the opaque color buffer:

```wgsl
@fragment
fn oitComposite(@location(0) uv: vec2<f32>) -> @location(0) vec4<f32> {
  let accum = textureSample(accumulationTexture, samp, uv);
  let revealage = textureSample(revealageTexture, samp, uv).r;

  // Avoid division by zero
  if accum.a < 1e-4 {
    discard;
  }

  // Reconstruct average transparent color
  let avgColor = accum.rgb / max(accum.a, 1e-4);

  // Blend with background
  let opaqueColor = textureSample(colorTexture, samp, uv).rgb;
  let result = avgColor * (1.0 - revealage) + opaqueColor * revealage;

  return vec4(result, 1.0);
}
```

## Buffer Setup

| Buffer | Format | Purpose |
|---|---|---|
| Accumulation | RGBA16Float | Weighted color sum (needs float precision) |
| Revealage | R8Unorm | Transmittance product (single channel) |

On WebGL2, `RGBA16F` requires the `EXT_color_buffer_half_float` extension. If unavailable, fall back to `RGBA32F` (widely supported) or `RGBA8` with reduced precision.

## MSAA Interaction

The OIT buffers are created with the same MSAA sample count as the main render target. They are resolved alongside the main buffers before the OIT composite pass reads them.

## Usage API

From the user's perspective, transparency is trivial:

```typescript
const glass = new Mesh(
  new BoxGeometry({ width: 2, height: 2, depth: 0.1 }),
  new LambertMaterial({
    color: [0.5, 0.8, 1.0],
    opacity: 0.3,
    transparent: true,
  })
)
scene.add(glass)
// Done. No sorting, no renderOrder hacks, no headaches.
```

Multiple overlapping transparent objects work correctly:

```typescript
// These can be in any order, overlap, intersect — doesn't matter
scene.add(transparentSphere1)
scene.add(transparentSphere2)
scene.add(transparentPlane)
```

## Limitations

WBOIT is an approximation. Known limitations:

1. **Same-depth fragments**: When two transparent fragments have the same depth, the weight function can't distinguish them. This is rarely visible in practice.
2. **High-opacity fragments**: Very opaque transparent objects (opacity > 0.9) can have slight color bleeding. For near-opaque objects, consider using alpha testing (discard) instead.
3. **Color accuracy**: The weighted average is an approximation of true back-to-front compositing. For 2–3 overlapping transparent layers, the result is visually indistinguishable from correct blending. For 10+ layers at the same pixel, slight color shifts may occur.
4. **No refraction**: WBOIT doesn't support screen-space refraction. This is a non-goal for Kestrel's minimal material set.

## Why Not Other Approaches?

| Approach | Pros | Cons |
|---|---|---|
| **Sorted rendering** (three.js) | Correct for non-overlapping | Fails on overlap, sorting cost, popping |
| **Depth peeling** | Exact | Multiple passes (slow) |
| **Linked-list OIT** | Exact | Requires compute/SSBO, high memory |
| **WBOIT** (chosen) | Fast, single pass, no sorting | Approximate (good enough) |
| **Moment-based OIT** | Better than WBOIT | More buffers, more complex |

WBOIT is the best fit for a game engine targeting mobile: one extra pass, two extra buffers, no compute shaders required, and visually correct for typical game transparency (windows, particles, water surfaces, ghosts).
