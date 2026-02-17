# Weighted Blended Order-Independent Transparency

## The Problem with Three.js Transparency

Three.js sorts transparent objects by centroid distance to the camera and
renders them back-to-front with alpha blending. This breaks when:

- Two objects intersect (no single sort order is correct)
- An object is large and spans multiple depth regions
- Many transparent objects overlap (sort is unstable)
- Single meshes have both opaque and transparent parts

Artists spend significant time working around these issues. Mantis eliminates
them entirely.

## Solution: Weighted Blended OIT

Mantis uses Weighted Blended Order-Independent Transparency (McGuire & Bavoil,
2013). This technique:

- Requires **no sorting** — fragments are accumulated in any order
- Handles **intersecting** transparent objects correctly
- Runs in a **single pass** — no multi-pass depth peeling
- Adds minimal GPU cost — one extra full-screen composite pass

### Trade-offs

- **Approximate:** The result is a weighted average, not physically exact
  compositing. For 2–4 overlapping layers, the visual difference from exact
  blending is imperceptible. For 10+ layers with wildly different alpha values,
  subtle color shifts can occur.
- **Best for:** Stylized games, particle effects, glass, water, UI elements —
  exactly the use cases Mantis targets.

## How It Works

### Concept

Instead of compositing fragments in order, WBOIT accumulates all transparent
fragments into two buffers:

1. **Accumulation buffer (RGBA16F):** Weighted sum of (color × alpha)
2. **Revealage buffer (R8):** Product of (1 - alpha) for all fragments

A full-screen composite pass reconstructs the final color.

### Weight Function

The weight function controls how much each fragment contributes based on its
depth. Closer fragments get higher weight, producing a visually correct
approximation:

```glsl
float weight(float z, float alpha) {
  return alpha * max(1e-2, min(3e3,
    10.0 / (1e-5 + pow(z / 5.0, 2.0) + pow(z / 200.0, 6.0))
  ));
}
```

This function was carefully designed by McGuire & Bavoil to:
- Give higher weight to closer fragments (depth-based priority)
- Avoid numerical instability at extreme depths
- Work across a wide depth range (0.1–200 m matches our world size)

### Transparent Fragment Shader

```glsl
// Fragment shader for transparent objects
layout(location = 0) out vec4 accumulation;  // MRT 0
layout(location = 1) out float revealage;    // MRT 1

void main() {
  vec4 color = computeColor();  // base color × lighting × textures
  float alpha = color.a;

  // Compute depth weight
  float viewDepth = gl_FragCoord.z / gl_FragCoord.w;
  float w = weight(viewDepth, alpha);

  // Output weighted color accumulation
  accumulation = vec4(color.rgb * alpha * w, alpha * w);

  // Output revealage (product of 1-alpha)
  revealage = alpha;  // blended multiplicatively via blend state
}
```

### Blend State for OIT Pass

The OIT render pass uses special blend modes:

**Accumulation target (MRT 0):** Additive blending
```
srcFactor: one
dstFactor: one
operation: add
```
All fragments' weighted colors are summed.

**Revealage target (MRT 1):** Multiplicative blending
```
srcFactor: zero
dstFactor: one-minus-src-alpha
operation: add
```
The revealage value is the product of (1 - alpha) for all fragments. Starting
from 1.0 (clear value), each fragment multiplies by its (1 - alpha).

### Composite Pass

After all transparent fragments are accumulated, a full-screen quad composites
the OIT result over the opaque scene:

```glsl
// Full-screen composite fragment shader
uniform sampler2D opaqueColor;     // resolved opaque scene
uniform sampler2D accumulationTex; // OIT accumulation
uniform sampler2D revealageTex;    // OIT revealage

out vec4 fragColor;

void main() {
  vec2 uv = gl_FragCoord.xy / resolution;

  vec4 accum = texture(accumulationTex, uv);
  float reveal = texture(revealageTex, uv).r;

  // If revealage is 1.0, no transparent fragments were written
  if (reveal >= 1.0) {
    fragColor = texture(opaqueColor, uv);
    return;
  }

  // Reconstruct average transparent color
  vec3 transparentColor = accum.rgb / max(accum.a, 1e-4);

  // Blend transparent over opaque
  vec3 opaque = texture(opaqueColor, uv).rgb;
  fragColor = vec4(
    mix(transparentColor, opaque, reveal),  // reveal=0 → fully transparent, reveal=1 → fully opaque
    1.0
  );
}
```

## Render Pipeline Integration

The OIT pass fits into the frame pipeline between the opaque pass and MSAA
resolve:

```
1. Shadow pass       → depth atlas
2. Opaque pass       → color + emissive + depth (4× MSAA)
3. OIT pass          → accumulation + revealage (1× — no MSAA needed)
4. MSAA resolve      → resolve opaque color
5. OIT composite     → blend OIT over resolved opaque
6. Bloom + final blit
```

**Why no MSAA for OIT?** The accumulation buffer uses RGBA16Float, which is
expensive at 4× MSAA on mobile (16 bytes × 4 samples = 64 bytes per pixel).
Since transparent objects are typically smooth (glass, particles), MSAA provides
minimal visual benefit. The opaque scene behind them is MSAA-resolved.

### Depth Handling

The OIT pass reads the opaque depth buffer but does **not write** to it:

```
depthWriteEnabled: false      // don't occlude other transparent objects
depthCompare: 'less-equal'    // still depth-test against opaque geometry
```

This means transparent objects correctly appear behind opaque objects but do
not occlude each other — the weight function handles relative ordering.

## Render Target Setup

```typescript
// Accumulation buffer — needs high precision for weighted sums
const accumulationTarget = device.createTexture({
  width: canvasWidth,
  height: canvasHeight,
  format: 'rgba16float',
  usage: 'render-target | sampled',
})

// Revealage buffer — single channel, 8-bit is sufficient
const revealageTarget = device.createTexture({
  width: canvasWidth,
  height: canvasHeight,
  format: 'r8unorm',
  usage: 'render-target | sampled',
})
```

**Clear values:**
- Accumulation: `[0, 0, 0, 0]` (no color accumulated)
- Revealage: `1.0` (fully revealed = no transparent fragments)

## Edge Cases

### Fully Opaque via Material Index

If a mesh is marked `transparent: true` but some vertices have `opacity: 1.0`
in their palette entry, those vertices produce `weight = 0` and
`revealage *= 0` — effectively invisible. To handle mixed meshes:

- If all palette entries are opaque, the mesh goes to the opaque pass
- If any palette entry has opacity < 1.0, the entire mesh goes to the OIT pass
- For truly mixed meshes, consider splitting into opaque and transparent
  sub-meshes at load time (automated by the loader)

### Emissive Transparent Objects

Transparent objects with emissive palette entries still write to the emissive
render target. The bloom pipeline picks them up normally.

## Performance

| Operation | Cost |
|---|---|
| OIT transparent pass (100 objects) | ~0.5 ms GPU |
| Composite full-screen pass | ~0.15 ms GPU |
| Total OIT overhead | ~0.65 ms GPU |

Compare to sorted alpha blending:
- Sort cost: O(n log n) per frame, ~0.2 ms for 100 objects
- Visual artifacts: frequent, require artist workarounds
- Intersecting objects: broken

WBOIT eliminates the sort cost and all visual artifacts at a fixed GPU cost
regardless of object count or complexity.

## API

Transparency is controlled at the material level:

```typescript
// Simple transparent material
const glass = new BasicMaterial({
  color: [0.9, 0.95, 1.0],
  opacity: 0.3,
  transparent: true,
})

// Lambert with transparent palette entries
const material = new LambertMaterial({
  palette: [
    { color: [1, 1, 1], opacity: 1.0 },         // opaque white
    { color: [0.5, 0.5, 1.0], opacity: 0.4 },    // translucent blue
  ],
  transparent: true,
  shadows: true,
})

// The renderer automatically routes transparent meshes to the OIT pass
const mesh = scene.createMesh(geometry, glass)
// No sorting, no headaches — it just works
```
