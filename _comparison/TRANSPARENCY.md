# Transparency Comparison

This document compares transparency approaches across all 9 engine implementations.

## Universal Agreement

All implementations that support transparency (8 out of 9) agree on these fundamental principles:

1. **Weighted Blended OIT** - All 8 implementations use Weighted Blended Order-Independent Transparency (McGuire & Bavoil, 2013)
2. **No Sorting Required** - Transparent objects can be rendered in any order without artifacts
3. **Depth-Based Weighting** - Closer transparent fragments contribute more to the final result via weight function
4. **Two-Buffer Accumulation** - Uses accumulation buffer (weighted color sum) and revealage buffer (alpha product)
5. **Single Composite Pass** - Final full-screen pass blends OIT result over opaque scene

## Key Variations

### 1. OIT Implementation Status

**Full WBOIT Implementation (7)**:
- caracal, fennec, hyena, lynx, mantis, rabbit, wren

**Not Specified/Minimal (2)**:
- **bonobo** - Mentions potential approaches, marked as "not yet specified for v1"
- **shark** - Brief mention, no detailed implementation

### 2. Weight Function Variations

All implementations use depth-based weight functions, but with different formulas:

**McGuire Equation 10 (Standard) (4)**:
```glsl
weight = alpha * max(1e-2, min(3e3, 10.0 / (1e-5 + pow(z / 200.0, 4.0))))
```
- **fennec**, **hyena**, **lynx**, **mantis**

**Simplified Variant (1)**:
```glsl
weight = alpha * max(1e-2, min(3e3, 10.0 / (1e-5 + pow(z / 5.0, 2.0) + pow(z / 200.0, 6.0))))
```
- **caracal**

**Tuned for Scene Depth (1)**:
```glsl
weight = alpha * clamp(0.03 / (0.00001 + pow(linearDepth, 4.0)), 0.01, 3000.0)
```
- **wren** (uses linear depth instead of non-linear z-buffer)

**Equation 7 Alternative (1)**:
```glsl
weight = max(0.01, alpha * 10.0 + 0.01) * (1.0 - gl_FragCoord.z * 0.9)
```
- **rabbit** (mentions as option)

**Not Specified (1)**:
- **shark**

### 3. Render Target Formats

**Accumulation Buffer**:

**RGBA16F (6)**:
- caracal, fennec, hyena, lynx, mantis, rabbit

**RGBA16Float (1)**:
- wren (explicitly noted for high precision)

**Not specified (1)**:
- shark

**Revealage Buffer**:

**R8 or R8Unorm (6)**:
- caracal, fennec, hyena, lynx, mantis, wren (single channel is sufficient)

**Float format mentioned (1)**:
- rabbit (uses R16F in some docs)

**Not specified (1)**:
- shark

### 4. Blend State Configuration

All implementations use the same blend equations:

**Accumulation Target**:
- `src: ONE, dst: ONE` (additive)
- All color and alpha channels use additive blending

**Revealage Target**:
- `src: ZERO, dst: ONE_MINUS_SRC_ALPHA` (multiplicative)
- Computes product of (1 - alpha) across all fragments

**Clear Values**:
- Accumulation: `[0, 0, 0, 0]` (no color accumulated)
- Revealage: `1.0` (fully revealed = no transparent fragments)

### 5. Depth Buffer Handling

All implementations agree on this approach:

- **Depth Test**: ON (read from opaque depth buffer)
- **Depth Write**: OFF (don't occlude other transparent objects)
- **Depth Read-Only**: Opaque geometry correctly occludes transparent objects
- **No Sorting**: Weight function handles relative ordering

### 6. MSAA Integration

**MSAA for OIT Buffers (3)**:
- **fennec** - Both accumulation and revealage are MSAA, resolved before composite
- **caracal** - Similar approach, MSAA targets resolved after accumulation
- **shark** - Mentions MSAA support

**No MSAA for OIT (2)**:
- **mantis** - Explicitly notes accumulation uses RGBA16F without MSAA (expensive on mobile)
- **hyena** - OIT buffers are 1x (resolved opaque is MSAA)

**Not Specified (3)**:
- lynx, rabbit, wren

## Implementation Breakdown

### OIT Accumulation Pass Shader

All implementations follow this pattern:

```glsl
layout(location = 0) out vec4 accumulation;
layout(location = 1) out float revealage;

void main() {
  vec4 color = computeLitColor();  // Normal material + lighting
  float alpha = color.a;

  // Depth-based weight
  float w = weight(gl_FragCoord.z, alpha);

  // Premultiplied color accumulation
  accumulation = vec4(color.rgb * alpha * w, alpha * w);

  // Revealage (blended multiplicatively)
  revealage = alpha;
}
```

Minor variations:
- **fennec** and **hyena**: Use `revealage = alpha * w` instead of just `alpha`
- **caracal**: Similar to standard approach
- **mantis**: Emphasizes zero-allocation approach

### OIT Composite Pass Shader

All implementations follow this pattern:

```glsl
void main() {
  vec4 accum = texture(accumulation, uv);
  float reveal = texture(revealage, uv).r;

  // Skip pixels with no transparent fragments
  if (accum.a < 1e-4 || reveal >= 1.0) {
    fragColor = opaqueColor;  // or discard
    return;
  }

  // Reconstruct average transparent color
  vec3 avgColor = accum.rgb / max(accum.a, 1e-4);

  // Composite over opaque
  fragColor = vec4(avgColor, 1.0 - reveal);
  // Then alpha blend over opaque scene
}
```

Variations:
- **fennec**: Uses `texelFetch` instead of `texture` for exact pixel access
- **hyena**: Similar texelFetch approach
- **lynx**: Detailed composite with revealage check
- Most others: Standard texture sampling

### Material API

All implementations support transparency via material flags:

```typescript
const material = createMaterial({
  color: [0.5, 0.8, 1.0],
  opacity: 0.3,
  transparent: true,  // Enables OIT
})
```

**With Material Index System** (3):
- **caracal**, **mantis**, **rabbit** - Support per-vertex opacity via material index palette

Example (caracal/mantis):
```typescript
const material = createMaterial({
  palette: [
    { color: [1, 1, 1], opacity: 1.0 },     // Opaque
    { color: [0.5, 0.5, 1.0], opacity: 0.4 }, // Translucent
  ],
  transparent: true,
})
```

### Render Pipeline Integration

All implementations follow this order:

```
1. Opaque Pass (depth write ON)
   → Writes to color + depth buffers

2. OIT Accumulation Pass (depth write OFF, depth test ON)
   → Renders all transparent objects (any order)
   → Writes to accumulation + revealage buffers
   → Depth test against opaque depth buffer

3. OIT Composite Pass
   → Full-screen quad
   → Blends transparent result over opaque scene
   → Alpha blending: (SRC_ALPHA, ONE_MINUS_SRC_ALPHA)

4. Post-Processing (optional)
   → Bloom, tone mapping, etc.
```

### Performance Characteristics

Reported costs (approximate):

| Implementation | OIT Pass Cost | Composite Cost | Total OIT Overhead |
|---|---|---|---|
| fennec | ~0.3ms | ~0.1ms | ~0.4ms |
| hyena | Not specified | Not specified | "Fixed, low cost" |
| lynx | Not specified | Not specified | Minimal |
| mantis | ~0.5ms (100 obj) | ~0.15ms | ~0.65ms |
| caracal | Not specified | Not specified | Minimal |
| rabbit | Not specified | Not specified | Minimal |
| shark | ~0.5ms | ~0.1ms | ~0.6ms |
| wren | Not specified | Not specified | Minimal overhead |

All implementations note that OIT cost is independent of transparent object count/complexity.

### WebGL2 Extension Requirements

All implementations require these WebGL2 extensions:

1. **Multiple Render Targets (MRT)**: Built into WebGL2 (`drawBuffers`)
2. **Float Textures**: `EXT_color_buffer_float` (97%+ browser support)
3. **Float Blending**: `EXT_float_blend` (widely supported)

**Fallback Strategy**:
- Most implementations mention falling back to sorted transparency if extensions unavailable
- **mantis** and **shark** explicitly check extensions at device creation

### WebGPU Implementation

Implementations note WebGPU advantages:

- **fennec**: Native MRT support, cleaner API
- **hyena**: Native float render targets, no extensions needed
- **caracal**: Simpler render pass descriptor with multiple color attachments
- **lynx**: Built-in per-attachment blend state

## Limitations & Trade-offs

All implementations acknowledge these WBOIT limitations:

### Acknowledged Limitations

1. **Approximation, Not Exact** (All 7):
   - Weighted average, not physically accurate compositing
   - 2-4 overlapping layers: visually indistinguishable from exact blending
   - 10+ layers with varying alpha: may show subtle color shifts

2. **Depth Similarity Issues** (5):
   - **fennec**, **hyena**, **lynx**, **mantis**, **caracal**
   - Fragments at very similar depths may not separate cleanly
   - Weight function can't distinguish same-depth fragments

3. **High Opacity Bleeding** (3):
   - **hyena**, **mantis**, **caracal**
   - Very opaque transparent objects (alpha > 0.9) can have color bleeding
   - Recommendation: use alpha-test (discard) for near-opaque objects

4. **No Refraction** (All 7):
   - WBOIT doesn't support screen-space refraction or distortion
   - All note this as a non-goal for their use cases

### Suitability Assessment

**Works Well For** (consensus across all implementations):
- Semi-transparent surfaces (glass, water, fog)
- Particle effects
- UI elements in 3D space
- Stylized/low-poly games
- Game objects with similar opacity values

**Works Less Well For** (noted by multiple implementations):
- Extreme opacity differences (α=0.01 behind α=0.99)
- Thick volumetrics (dense smoke/fog) - requires raymarching
- Cases requiring exact per-pixel ordering
- High-contrast colored glass stacking

## Special Features & Optimizations

### Caracal
- Selective emissive-driven transparency control
- Material index system with per-vertex opacity
- Emphasis on "transparency headache" elimination

### Fennec
- Integrated with MSAA and bloom in complete pipeline
- Debug visualization of OIT buffers (`debugShowOITBuffers`)
- Detailed documentation of weight function tuning

### Hyena
- Clear explanation of weight function biasing
- Comparison table vs. other transparency approaches
- Emphasis on mobile GPU compatibility

### Lynx
- Most detailed weight function analysis
- Linear depth variant for uniform weighting across scene depth
- WebGL2 vs WebGPU comparison table
- Detailed blend state configuration per API

### Mantis
- Explicit handling of mixed opaque/transparent via material index
- Auto-splitting of mixed meshes (planned)
- Emissive transparent objects supported
- Performance comparison vs sorted alpha blending

### Rabbit
- Multiple weight function options documented
- Integration with complete post-processing pipeline
- Full-screen triangle technique documented

### Wren
- Linear depth weight function (vs non-linear Z-buffer)
- Transferable flat buffer support (Web Workers)
- WebGPU compute culling option mentioned

## Recommendations for Cherry-Picking

**For Best Quality**:
- Use **lynx's** linear-depth weight function for uniform depth handling:
  ```glsl
  float linearDepth = (2.0 * near) / (far + near - z * (far - near));
  float w = alpha * clamp(0.03 / (0.00001 + pow(linearDepth, 4.0)), 0.01, 3000.0);
  ```
- Or use standard **McGuire Equation 10** (fennec, hyena, lynx, mantis)

**For Memory Efficiency**:
- Use **R8** for revealage buffer (single channel sufficient)
- Consider **mantis's** approach: no MSAA on OIT buffers (saves 4× memory on mobile)

**For MSAA Support**:
- Follow **fennec's** approach: MSAA accumulation/revealage, resolve before composite
- Or **mantis's** approach: MSAA opaque only, OIT at 1x

**For Material Control**:
- Use **caracal's** or **mantis's** material index system for per-vertex opacity
- Enables mixed opaque/transparent meshes without splitting

**For Debugging**:
- Implement **fennec's** debug visualization: show accumulation/revealage side-by-side
- Helps tune weight function for specific scenes

**For WebGL2 Compatibility**:
- Check for required extensions at startup (**mantis**, **shark** approach)
- Provide graceful fallback to sorted transparency
- Test on actual mobile devices (extension support varies)

**Weight Function Selection**:

1. **General Purpose** - McGuire Equation 10 (most implementations):
   ```glsl
   weight = alpha * max(1e-2, min(3e3, 10.0 / (1e-5 + pow(z / 200.0, 4.0))))
   ```

2. **Wide Depth Range** - Caracal variant with dual falloff:
   ```glsl
   weight = alpha * max(1e-2, min(3e3, 10.0 / (1e-5 + pow(z/5.0, 2.0) + pow(z/200.0, 6.0))))
   ```

3. **Uniform Across Depth** - Wren linear depth:
   ```glsl
   float linearZ = (2.0 * near) / (far + near - z * (far - near));
   weight = alpha * clamp(0.03 / (0.00001 + pow(linearZ, 4.0)), 0.01, 3000.0);
   ```

4. **Simple/Fast** - Rabbit Equation 7:
   ```glsl
   weight = max(0.01, alpha * 10.0 + 0.01) * (1.0 - z * 0.9);
   ```

**Integration with Post-Processing**:
- OIT composite should happen **before** bloom extraction (fennec, caracal, mantis)
- Transparent emissive objects can contribute to bloom if desired
- Post-processing operates on final composited result
