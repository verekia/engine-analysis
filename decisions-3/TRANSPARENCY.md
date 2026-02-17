# TRANSPARENCY.md - Order-Independent Transparency

## Approach

**Decision: Weighted Blended Order-Independent Transparency (WBOIT)** (universal agreement)

McGuire & Bavoil 2013. Single-pass transparent rendering without sorting.

### Why WBOIT

The engine uses automatic instancing and draw call sorting for performance. Per-object depth sorting for transparency would break instancing and require re-sorting every frame from the camera's perspective. WBOIT eliminates sorting entirely:

- No depth sorting of transparent objects required
- No draw order dependency
- Compatible with instancing (transparent instances rendered in any order)
- Single additional render pass
- Handles overlapping transparent surfaces correctly (within limits)

### Known Limitations

- Approximation, not exact: weight function determines quality of ordering approximation
- Struggles with similar-depth overlapping surfaces at very different opacities
- Not suitable for stained glass or complex refractive materials (out of scope for stylized games)

For the target aesthetic (stylized/low-poly with occasional transparent surfaces like water, windows, particle effects), WBOIT provides more than adequate quality.

## Implementation

### Render Targets

| Target | Format | Purpose |
|--------|--------|---------|
| Accumulation | RGBA16F | Weighted color sum |
| Revealage | R8 | Alpha product for coverage |

Both targets are at canvas resolution, 1x MSAA (transparency is rendered after MSAA resolve of the opaque pass).

### Transparent Pass

Transparent meshes are rendered after the opaque pass with:
- **Depth test**: Enabled (read-only, no depth write)
- **Depth write**: Disabled
- **Blend state**:
  - Accumulation: `src: ONE, dst: ONE` (additive)
  - Revealage: `src: ZERO, dst: ONE_MINUS_SRC_ALPHA` (multiplicative)

### Weight Function

**Decision: Depth-based weight** (universal agreement)

```glsl
float weight(float depth, float alpha) {
  // Clamp depth to avoid precision issues
  float d = clamp(depth, 1e-5, 3e4);
  // Weight function: closer fragments contribute more
  float w = alpha * max(1e-2, 3e3 * pow(1.0 - d * 0.001, 3.0));
  return clamp(w, 1e-2, 3e4);
}
```

The specific weight function varies slightly across implementations, but all use a depth-based approach where closer fragments receive higher weight. The clamping prevents numerical issues (infinities, denormals).

### Fragment Shader (Transparent Pass)

```glsl
layout(location = 0) out vec4 accumulation;
layout(location = 1) out float revealage;

void main() {
  vec4 color = computeLitColor();  // Lambert shading, same as opaque
  float w = weight(gl_FragCoord.z, color.a);

  accumulation = vec4(color.rgb * color.a * w, color.a * w);
  revealage = color.a;
}
```

### Composite Pass

Full-screen pass that blends the transparent result over the opaque scene:

```glsl
uniform sampler2D u_accumulation;
uniform sampler2D u_revealage;
uniform sampler2D u_opaqueColor;

void main() {
  vec4 accum = texture(u_accumulation, v_uv);
  float reveal = texture(u_revealage, v_uv).r;

  // Reconstruct average transparent color
  vec3 transparentColor = accum.rgb / max(accum.a, 1e-5);

  // Blend over opaque
  vec3 opaqueColor = texture(u_opaqueColor, v_uv).rgb;
  fragColor = vec4(mix(transparentColor, opaqueColor, reveal), 1.0);
}
```

### Blend State Configuration

```typescript
// Accumulation target
{
  srcFactor: 'one',
  dstFactor: 'one',
  operation: 'add',
  srcAlphaFactor: 'one',
  dstAlphaFactor: 'one',
  alphaOperation: 'add',
}

// Revealage target
{
  srcFactor: 'zero',
  dstFactor: 'one-minus-src-alpha',
  operation: 'add',
}
```

## Integration with Material System

Materials set transparency via the `transparent` flag:

```typescript
const glassMaterial = createLambertMaterial({
  color: [0.5, 0.7, 1.0],
  transparent: true,
  opacity: 0.4,
})
```

Transparent meshes are:
1. Excluded from the opaque pass
2. Excluded from shadow casting (they don't write depth)
3. Rendered in the transparent pass
4. Composited over the opaque result

### Material Index Transparency

Palette entries can specify per-index opacity:

```typescript
{
  palette: [
    { color: [1, 0, 0], opacity: 1.0 },      // Fully opaque
    { color: [0, 0.5, 1], opacity: 0.3 },     // Transparent blue
  ]
}
```

When any palette entry has `opacity < 1.0`, the mesh is routed to the transparent pass.

## WebGL2 Compatibility

WBOIT requires:
- MRT (multiple render targets): Supported in WebGL2 via `drawBuffers`
- RGBA16F textures: Supported via `EXT_color_buffer_float` extension
- R8 texture as render target: Natively supported in WebGL2

All requirements are met by WebGL2. The engine checks for `EXT_color_buffer_float` and falls back to RGBA8 accumulation if unavailable (slight quality loss).

## Performance

| Metric | Cost |
|--------|------|
| Transparent pass (200 objects) | ~0.5ms GPU |
| Composite pass (full-screen) | ~0.1ms GPU |
| Memory (accumulation + revealage, 1080p) | ~16MB RGBA16F + ~2MB R8 |
| **Total OIT overhead** | **~0.6ms GPU** |

OIT adds ~0.6ms to the frame, well within budget. The accumulation target is the main memory cost; at 1080p with RGBA16F, it is ~16MB. This is acceptable given the target device class.

## Render Pass Order

```
1. Opaque pass (with MSAA)
2. MSAA resolve
3. Transparent pass (WBOIT, 1x, depth read-only from opaque)
4. OIT composite (full-screen blend over resolved opaque)
5. Bloom (operates on composited result)
6. Tone mapping + final blit
```

Bloom happens after OIT composite so that emissive transparent surfaces also contribute to bloom.
