# TRANSPARENCY - Design Decisions

## Technique

**Decision: Weighted Blended Order-Independent Transparency (WBOIT, McGuire & Bavoil 2013)**

- Sources: All 8 active implementations agree (7 with full documentation)
- Single-pass, no sorting required — "transparency that actually works without headaches"
- Eliminates the traditional transparent sorting artifacts that plague Three.js

## Weight Function

**Decision: McGuire Equation 10 (standard)**

- Sources: Fennec, Hyena, Lynx, Mantis (4/9 — most popular)

```glsl
float weight(float z, float alpha) {
  return alpha * max(1e-2, min(3e3, 10.0 / (1e-5 + pow(z / 200.0, 4.0))));
}
```

Rationale: Well-tested, produces good results for the 200m world depth range. The `z / 200.0` term aligns with the engine's 200m shadow/world scale. Closer fragments get higher weight, creating correct depth ordering in the weighted average.

Alternative for wider depth ranges: Caracal's dual-falloff variant. Alternative for uniform depth handling: Wren's linear depth variant. These can be offered as configuration options later.

## Render Targets

**Decision: RGBA16F accumulation + R8 revealage, no MSAA on OIT buffers**

Accumulation buffer:
- Format: RGBA16F
- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit (6/9)
- RGBA16F provides sufficient precision for weighted color accumulation

Revealage buffer:
- Format: R8 (single channel sufficient)
- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Wren (6/9)

No MSAA on OIT buffers:
- Sources: Mantis (explicit recommendation)
- Rationale: MSAA on float16 render targets is expensive on mobile. The opaque pass is MSAA; OIT operates at 1x. Since OIT is an approximation anyway, the visual difference is negligible.

## Blend State Configuration

**Decision: Standard OIT blend equations (universal agreement)**

Accumulation target:
```
src: ONE, dst: ONE  (additive blending)
```
All color and alpha channels use additive blending.

Revealage target:
```
src: ZERO, dst: ONE_MINUS_SRC_ALPHA  (multiplicative)
```
Computes product of (1 - alpha) across all fragments.

Clear values:
- Accumulation: `[0, 0, 0, 0]`
- Revealage: `1.0` (fully revealed = no transparent fragments)

## Depth Buffer Handling

**Decision: Depth test ON, depth write OFF (universal agreement)**

```
depthTest: true   // read from opaque depth buffer
depthWrite: false  // don't occlude other transparent objects
```

Opaque geometry correctly occludes transparent objects. Transparent fragments are not depth-sorted against each other — the weight function handles relative ordering.

## Accumulation Shader

```glsl
layout(location = 0) out vec4 accumulation;
layout(location = 1) out float revealage;

void main() {
  vec4 color = computeLitColor();
  float alpha = color.a;
  float w = weight(gl_FragCoord.z, alpha);

  accumulation = vec4(color.rgb * alpha * w, alpha * w);
  revealage = alpha;
}
```

## Composite Shader

```glsl
void main() {
  vec4 accum = texelFetch(accumTexture, ivec2(gl_FragCoord.xy), 0);
  float reveal = texelFetch(revealTexture, ivec2(gl_FragCoord.xy), 0).r;

  // Skip pixels with no transparent fragments
  if (accum.a < 1e-4 || reveal >= 1.0) {
    discard;
  }

  // Reconstruct average transparent color
  vec3 avgColor = accum.rgb / max(accum.a, 1e-4);

  // Output for alpha blending over opaque scene
  fragColor = vec4(avgColor, 1.0 - reveal);
}
```

Uses `texelFetch` (Fennec, Hyena approach) for exact pixel access instead of texture sampling.

## Render Pipeline Order

```
1. Opaque Pass (depth write ON, MSAA)
2. OIT Accumulation Pass (depth write OFF, depth test ON, 1x)
3. MSAA Resolve (opaque)
4. OIT Composite (alpha blend over resolved opaque)
5. Post-Processing (bloom, tone mapping)
```

OIT composite happens before bloom so that transparent emissive objects can contribute to bloom.

## Material API

```typescript
const glass = createLambertMaterial({
  color: [0.5, 0.8, 1.0],
  opacity: 0.3,
  transparent: true,
})

// With material index palette (per-vertex transparency)
const mixed = createLambertMaterial({
  palette: [
    { color: [1, 1, 1], opacity: 1.0 },           // opaque
    { color: [0.5, 0.5, 1.0], opacity: 0.4 },      // translucent blue
  ],
  transparent: true,
})
```

## WebGL2 Extension Requirements

Required extensions (all well-supported):
- **MRT**: Built into WebGL2 (`drawBuffers`)
- **Float textures**: `EXT_color_buffer_float` (97%+ support)
- **Float blending**: `EXT_float_blend` (widely supported)

Check at startup; fall back to sorted transparency if extensions unavailable (Mantis, Shark approach).

## Known Limitations (Accepted Trade-offs)

WBOIT is an approximation, not physically exact:

- **2-4 overlapping layers**: visually indistinguishable from exact blending
- **10+ layers with varying alpha**: may show subtle color shifts
- **Near-same depth fragments**: weight function can't distinguish them
- **Very high opacity (>0.9)**: use alpha-test (discard) instead for near-opaque objects
- **No refraction**: not a goal for the target aesthetic

These limitations are acceptable for the stylized low-poly game target. The massive DX improvement ("transparency that actually works") far outweighs the edge cases.

## Performance

OIT cost is independent of transparent object count:
- Accumulation pass: ~0.3-0.5ms
- Composite pass: ~0.1-0.15ms
- Total overhead: ~0.4-0.65ms

No sorting, no overdraw management, no order-dependent artifacts.
