# TRANSPARENCY — Final Decision

## Technique

**Decision: Weighted Blended Order-Independent Transparency (WBOIT)** (universal agreement)

WBOIT renders all transparent objects in a single pass without sorting. No per-frame sort required, no visual artifacts from incorrect ordering in complex overlapping cases.

## Weight Function

**Decision: McGuire Equation 10** (universal agreement)

```glsl
float weight(float depth, float alpha) {
  return alpha * clamp(0.03 / (1e-5 + pow(depth / 200.0, 4.0)), 1e-2, 3e3);
}
```

Depth-based weighting approximates correct ordering: closer transparent surfaces contribute more to the final color. The `/200.0` matches the 200m world range.

## Render Targets

| Target | Format | Purpose |
|--------|--------|---------|
| Accumulation | RGBA16F | Weighted premultiplied color sum |
| Revealage | R8 | Alpha product (how much background shows through) |

No MSAA on OIT buffers — OIT already provides edge quality through weighted blending, and MSAA on float targets is expensive.

## Blend States

### Accumulation Buffer

```
srcFactor: ONE
dstFactor: ONE
equation: ADD
```

Additive blending accumulates weighted color contributions.

### Revealage Buffer

```
srcFactor: ZERO
dstFactor: ONE_MINUS_SRC_ALPHA
equation: ADD
```

Multiplies accumulated alpha: `result = ∏(1 - αᵢ)`.

## Transparent Pass Configuration

- Depth test: **ON** (transparent objects respect opaque depth)
- Depth write: **OFF** (transparent objects don't occlude each other)
- Both buffers written simultaneously via MRT

## Fragment Shader (Transparent Pass)

```glsl
void main() {
  vec4 color = computeLitColor();  // Same lighting as opaque
  float w = weight(gl_FragCoord.z, color.a);

  // Output 0: accumulation
  fragAccum = vec4(color.rgb * color.a * w, color.a * w);

  // Output 1: revealage
  fragReveal = color.a;
}
```

## Composite Pass

Full-screen pass that blends the OIT result over the resolved opaque image:

```glsl
void main() {
  vec4 accum = texelFetch(u_accumTexture, ivec2(gl_FragCoord.xy), 0);
  float reveal = texelFetch(u_revealTexture, ivec2(gl_FragCoord.xy), 0).r;

  // Reconstruct average color
  vec3 averageColor = accum.rgb / max(accum.a, 1e-5);

  // Composite over opaque
  fragColor = vec4(averageColor * (1.0 - reveal) + opaqueColor.rgb * reveal, 1.0);
}
```

## Render Pipeline Order

```
1. Shadow pass
2. Opaque pass (MSAA, MRT: color + emissive)
3. Transparent pass (OIT accumulation into RGBA16F + R8)
4. MSAA resolve (opaque color + emissive)
5. OIT composite (blend transparent over resolved opaque)
6. Bloom (on composited result)
7. Tone mapping + final blit
```

## Material API

```typescript
// Fully transparent material
const glass = createLambertMaterial({
  color: [0.5, 0.7, 1.0],
  transparent: true,
  opacity: 0.4,
})

// Per-entry transparency via palette
const material = createLambertMaterial({
  palette: [
    { color: [1, 1, 1], opacity: 1.0 },    // opaque
    { color: [1, 0, 0], opacity: 0.3 },    // transparent red
  ],
})
```

Materials with `transparent: true` or palette entries with `opacity < 1.0` are automatically routed to the OIT pass via the sort key's transparent bit.

## WebGL2 Compatibility

Required extensions:
- `EXT_color_buffer_float` — for RGBA16F render target
- `EXT_float_blend` — for additive blending on float targets

Both are widely supported on WebGL2 devices. If unavailable, transparency falls back to simple alpha blending (sorted back-to-front).

## Known Limitations

WBOIT is an approximation:
- Very similar depths with different colors may show subtle ordering errors
- Extremely high alpha overlap (>8 layers) can saturate the accumulation buffer
- Not suitable for refractive/volumetric effects (not needed for the target aesthetic)

These limitations are acceptable for the stylized/low-poly target. WBOIT eliminates the far worse artifacts of incorrect sort-based transparency.

## Performance Budget

```
Transparent pass (200 objects):  ~0.3-0.5ms GPU
OIT composite (fullscreen):      ~0.1-0.15ms GPU
───────────────────────────────────────────────
Total OIT overhead:              ~0.4-0.65ms GPU
```
