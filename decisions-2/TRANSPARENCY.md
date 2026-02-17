# TRANSPARENCY.md - Decisions

## Decision: WBOIT with McGuire Equation 10, RGBA16F + R8, No MSAA on OIT Buffers

### Technique: Weighted Blended Order-Independent Transparency (WBOIT)

Universal agreement across all 9 implementations. The prompt explicitly states "transparency that actually works without headaches (unlike three.js)."

WBOIT (McGuire & Bavoil, 2013) eliminates the sorting headaches of traditional alpha blending:
- Transparent objects can be rendered in any order
- No sorting required
- Single-pass accumulation
- Fixed cost regardless of overlap count

### Weight Function: McGuire Equation 10

**Chosen**: Standard McGuire Equation 10 (4/9: Fennec, Hyena, Lynx, Mantis)

```glsl
float weight = alpha * max(1e-2, min(3e3,
  10.0 / (1e-5 + pow(z / 200.0, 4.0))
));
```

This function gives closer transparent fragments more influence. The `z / 200.0` denominator is tuned for the engine's 200m world scale (from the prompt).

**Rejected**: Wren's linear depth variant - slightly more uniform across depth but requires extra computation
**Rejected**: Rabbit's Equation 7 - simpler but less effective at separating overlapping layers

### Render Target Formats

**Accumulation buffer**: RGBA16F (6/9 use this)
- Stores weighted premultiplied color sum + weighted alpha sum
- Half-float precision prevents banding at low alpha values

**Revealage buffer**: R8 (6/9: Caracal, Fennec, Hyena, Lynx, Mantis, Wren)
- Single channel is sufficient (stores product of `1 - alpha` across all fragments)
- Saves memory vs R16F

### MSAA: No MSAA on OIT Buffers

**Chosen**: OIT buffers at 1x, MSAA on opaque only (Mantis approach)
**Rejected**: MSAA on OIT buffers (Fennec/Caracal) - expensive on mobile, 4x memory cost for the RGBA16F accumulation buffer

Transparent objects are typically used for effects (glass, fog, particles) where MSAA provides minimal visual benefit. The opaque scene behind them is MSAA'd, which handles edge aliasing where it matters.

### Blend States

Universal agreement:

**Accumulation target** (location 0):
```
src: ONE, dst: ONE  (additive blending)
```

**Revealage target** (location 1):
```
src: ZERO, dst: ONE_MINUS_SRC_ALPHA  (multiplicative)
```

**Clear values**:
- Accumulation: `[0, 0, 0, 0]`
- Revealage: `1.0`

### Depth Buffer Handling

Universal agreement:
- **Depth test**: ON (read from opaque depth buffer)
- **Depth write**: OFF (transparent objects don't occlude each other)
- Opaque geometry correctly occludes transparent objects via the shared depth buffer

### Accumulation Pass Shader

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

### Composite Pass Shader

```glsl
void main() {
  vec4 accum = texelFetch(u_accumulation, ivec2(gl_FragCoord.xy), 0);
  float reveal = texelFetch(u_revealage, ivec2(gl_FragCoord.xy), 0).r;

  if (accum.a < 1e-4 || reveal >= 1.0 - 1e-4) {
    discard;  // No transparent fragments at this pixel
  }

  vec3 avgColor = accum.rgb / max(accum.a, 1e-4);
  fragColor = vec4(avgColor, 1.0 - reveal);
}
```

Using `texelFetch` (Fennec/Hyena approach) for exact pixel access, no filtering.

### Render Pipeline Order

```
1. Opaque Pass (depth write ON, MSAA 4x)
2. OIT Accumulation Pass (depth write OFF, depth test ON, 1x)
3. OIT Composite Pass (full-screen triangle, alpha blend over opaque)
4. Post-Processing (bloom, tone mapping)
```

OIT composite happens before bloom so transparent emissive objects can contribute to bloom if desired.

### Material API

```typescript
const glassMaterial = createLambertMaterial({
  color: [0.5, 0.8, 1.0],
  opacity: 0.3,
  transparent: true,
})
```

Per-vertex opacity via material index palette:

```typescript
const material = createLambertMaterial({
  palette: [
    { color: [1, 1, 1], opacity: 1.0 },        // Opaque region
    { color: [0.5, 0.5, 1.0], opacity: 0.4 },   // Translucent region
  ],
  transparent: true,
})
```

### WebGL2 Extension Requirements

- `EXT_color_buffer_float` (RGBA16F render targets) - 97%+ browser support
- `EXT_float_blend` (float blending for accumulation) - widely supported
- MRT via `drawBuffers` (built into WebGL2)

**Fallback**: If extensions unavailable, fall back to sorted alpha blending (check at device creation, Mantis/Shark approach).

### Known Limitations

Accepted trade-offs (universal across all implementations):

1. **Approximation**: Weighted average, not physically exact compositing. For 2-4 overlapping layers, visually indistinguishable from exact blending.
2. **Depth similarity**: Fragments at very similar depths may not separate cleanly.
3. **High opacity**: Near-opaque transparent objects (alpha > 0.9) can exhibit color bleeding. Recommendation: use alpha-test (discard) for near-opaque objects.
4. **No refraction**: WBOIT doesn't support screen-space refraction or distortion.

**Suitability**: Excellent for the target use case (stylized games with glass, water, fog, particles, UI elements in 3D space).
