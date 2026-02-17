# Lighting & Shadows Comparison

This document compares lighting and shadow systems across all 9 engine implementations.

## Universal Agreement

All implementations that support lighting and shadows (8 of 9, excluding Bonobo v1) agree on:

### Lighting Model
- **Lambert diffuse shading** as the primary shading model (simple N·L dot product)
- **Directional light** as the primary light source (infinite distance, parallel rays like the sun)
- **Ambient light** for uniform base illumination (no direction, no shadows)
- **No specular highlights** - purely diffuse for stylized/low-poly aesthetic
- **Light data in UBO/uniform buffer** uploaded once per frame (zero per-draw-call overhead)

### Shadow Technique
- **Cascaded Shadow Maps (CSM)** for directional light shadows
- **3 cascades** as the standard configuration for 200×200m worlds
- **Practical split scheme** blending logarithmic and linear distribution (lambda parameter typically 0.5-0.7)
- **Orthographic projection** for each cascade (tight fit around cascade frustum slice)
- **Texel snapping** to prevent shadow shimmer on camera movement
- **Depth-only shadow pass** (minimal vertex shader, no fragment output)

### Shadow Sampling
- **PCF (Percentage Closer Filtering)** for smooth shadow edges
- **Cascade selection** based on fragment view-space depth
- **Bias techniques** (depth bias + slope-scaled or normal bias) to prevent shadow acne
- **Front-face culling** during shadow rendering (render back faces to reduce acne)

### Typical Cascade Splits
All implementations target similar split distances for 200×200m worlds:
- **Cascade 0**: 0.1m - 8-15m (near, high detail)
- **Cascade 1**: 8-15m - 40-60m (mid range)
- **Cascade 2**: 40-60m - 150-300m (far range)

---

## Key Variations

### 1. Implementation Status

**Minimal (1 implementation):**
- **Bonobo**: Shadows explicitly out of scope for v1, deferred to v2. Basic Lambert lighting only.

**Full implementation (8 implementations):**
- Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren all have production-ready CSM systems

### 2. Shadow Map Storage

**Texture atlas (6 implementations):**
- **Caracal, Hyena, Mantis, Rabbit, Shark, Wren**: Single 2D depth texture with cascades in different viewport regions
- Atlas layouts vary: 3072×1024 (3 horizontal), 4096×4096 (2×2 grid), 2048×1536 (custom)

**Texture array (2 implementations):**
- **Fennec, Lynx**: Use `TEXTURE_2D_ARRAY` (3 layers) on both WebGL2 and WebGPU
- Cleaner shader code (array index instead of UV offset calculations)
- Better cache behavior

**Hybrid (1 implementation):**
- **Wren**: Mentions both atlas and texture array, backend-specific choice

### 3. Shadow Map Resolution

**Per-cascade resolution:**
- **Caracal, Hyena, Rabbit, Shark**: 1024×1024 per cascade (default)
- **Fennec, Lynx, Mantis, Wren**: 2048×2048 per cascade (higher quality default)

**Total atlas size:**
- 1024 per cascade: 3072×1024 or equivalent
- 2048 per cascade: 6144×2048 or 4096×4096 grid

### 4. Cascade Split Lambda

**Medium blend (4 implementations):**
- **Caracal, Hyena, Rabbit, Shark**: lambda = 0.5 (balanced linear/logarithmic)

**Higher logarithmic weighting (4 implementations):**
- **Fennec, Lynx, Mantis, Wren**: lambda = 0.7 (more resolution near camera)

### 5. PCF Kernel Size

**3×3 PCF (4 implementations):**
- **Caracal, Hyena, Rabbit, Shark**: 9 samples in 3×3 grid or Poisson disk
- Good balance of quality and performance

**5×5 PCF (3 implementations):**
- **Lynx, Mantis, Wren**: 25 samples in 5×5 grid
- Smoother shadows, higher fragment shader cost

**4-sample PCF (1 implementation):**
- **Hyena**: 2×2 Poisson-like pattern (cheapest, still acceptable quality)

**Configurable (2 implementations):**
- **Fennec, Mantis**: Support both 3×3 and 5×5 via config option

### 6. Hardware Shadow Comparison

**All implementations use hardware comparison samplers:**
- **WebGL2**: `sampler2DShadow` with `TEXTURE_COMPARE_MODE = COMPARE_REF_TO_TEXTURE`
- **WebGPU**: `sampler_comparison` with `textureSampleCompare()` or `textureSampleCompareLevel()`

### 7. Cascade Blending

**With cascade blending (6 implementations):**
- **Fennec, Hyena, Lynx, Mantis, Rabbit, Wren**: Smooth transition between cascades
- Blend zone typically 10% of cascade range or 2-5 meters
- Mix between adjacent cascade samples near boundaries

**No explicit cascade blending (2 implementations):**
- **Caracal, Shark**: Hard cascade boundaries (simpler shader, visible seam possible)

### 8. Texel Snapping Approach

**Projection matrix snapping (5 implementations):**
- **Caracal, Fennec, Hyena, Mantis, Wren**: Snap light matrix translation components to texel grid
- Prevents sub-texel shifts during camera movement

**Bounding box snapping (3 implementations):**
- **Lynx, Rabbit, Shark**: Quantize orthographic bounds (min/max X/Y) to texel boundaries
- Same effect, different implementation point

### 9. Shadow Bias Strategy

**Depth bias + slope-scaled bias (6 implementations):**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit**: Combine constant depth bias with slope factor
- `bias = constantBias + slopeBias * (1 - NdotL)` or `tan(acos(NdotL))`

**Depth bias + normal bias (2 implementations):**
- **Shark, Wren**: Depth bias + offset world position along surface normal before shadow lookup
- Normal bias measured in world units (e.g., 0.02-0.5m)

### 10. Shadow Caster Culling

**Per-cascade frustum culling (8 implementations):**
- All active implementations cull shadow casters against each cascade's light-space frustum
- Avoids rendering distant objects into near cascade and vice versa
- Significant performance gain (e.g., 800 objects → 120+350+600 split across cascades)

**Front-to-back sorting (3 implementations):**
- **Hyena, Mantis, Shark**: Explicitly sort shadow casters front-to-back for early-Z rejection
- Faster depth-only rendering (occluded fragments skip vertex shader)

### 11. Lighting Shader Integration

**Fragment shader lighting (8 implementations):**
- All compute Lambert lighting in the fragment shader during opaque pass
- Standard equation: `ambient + diffuse * NdotL * shadow`

**Emissive support (5 implementations):**
- **Fennec, Lynx, Mantis, Rabbit, Wren**: Materials can emit light (additive, unaffected by lighting/shadows)
- Feeds into bloom post-processing

### 12. Light Uniform Buffer Layout

**All implementations pack light data into per-frame UBO:**
- Directional light direction, color, intensity
- Ambient light color, intensity
- Shadow matrices (3× mat4)
- Cascade split distances
- Shadow bias parameters
- Cascade atlas offsets/scales

**Total UBO size:**
- Typical: 256-320 bytes per frame

### 13. Shadow Pass Shader Complexity

**Position-only shadow shader (6 implementations):**
- **Caracal, Fennec, Hyena, Mantis, Rabbit, Shark**: Only transform positions, no other attributes
- Ultra-minimal vertex shader, empty/trivial fragment shader

**Skinned shadow shader (8 implementations):**
- All implementations support GPU skinning in shadow vertex shader
- Same bone matrix calculation as main pass

**Alpha-tested shadow shader (2 implementations):**
- **Fennec, Lynx**: Optional fragment shader for alpha-tested materials
- Discards fragments based on texture alpha

### 14. Shadow Atlas Layout Strategy

**Horizontal strip (3 implementations):**
- **Caracal, Fennec, Wren**: 3072×1024 (three 1024×1024 regions side by side)

**Grid layout (2 implementations):**
- **Mantis, Rabbit**: 4096×4096 with 2×2 grid (2048 per cascade, one quadrant unused)

**Custom/flexible (3 implementations):**
- **Hyena, Lynx, Shark**: Configurable atlas layout based on cascade count and resolution

### 15. WebGPU vs WebGL2 Shadow Code

**Unified shader logic (8 implementations):**
- All implementations abstract cascade selection, PCF sampling behind same algorithm
- Backend differences (WGSL vs GLSL, `textureSampleCompare` vs `texture()` with `sampler2DShadow`) hidden by shader generation

**Texture array preference on WebGPU (2 implementations):**
- **Fennec, Lynx**: Prefer `texture_depth_2d_array` on WebGPU for cleaner API
- Fall back to atlas on WebGL2 if needed

### 16. Shadow Caching

**Static shadow caching (2 implementations):**
- **Lynx, Wren**: Cache shadow maps when scene/camera/light are static
- Skip shadow pass entirely, reuse previous frame's shadow map
- "Dirty" flag system tracks when re-rendering is needed

**No explicit caching (6 implementations):**
- Most implementations re-render shadows every frame
- Still performant due to depth-only rendering speed

### 17. Light Node Structure

**Lights as scene nodes (6 implementations):**
- **Caracal, Fennec, Lynx, Mantis, Rabbit, Shark**: Lights extend `Node` or `Node3D`
- Direction derived from node rotation/world matrix
- Can be parented, transformed, animated

**Ambient as scene property (4 implementations):**
- **Fennec, Lynx, Mantis, Wren**: Ambient light stored directly on `Scene` object (no spatial properties)
- Simpler for global uniform illumination

### 18. Shadow Distance

**200m default (4 implementations):**
- **Caracal, Fennec, Hyena, Mantis**: Shadows cover up to 200m from camera

**150m default (2 implementations):**
- **Rabbit, Shark**: Slightly shorter max shadow distance

**300m default (1 implementation):**
- **Wren**: Longer shadow distance for larger worlds

**Configurable (all):**
- All implementations allow runtime configuration of shadow distance

---

## Implementation Breakdown

### By Feature Completeness

| Feature | Bonobo | Caracal | Fennec | Hyena | Lynx | Mantis | Rabbit | Shark | Wren |
|---------|--------|---------|--------|-------|------|--------|--------|-------|------|
| Lambert lighting | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Directional light | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Ambient light | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| CSM shadows | ❌ v2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Cascade blending | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Shadow caching | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Emissive materials | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ |

### By Shadow Quality

**Highest quality (5×5 PCF + cascade blending):**
- **Lynx, Mantis, Wren**: 25-sample PCF with smooth cascade transitions
- Best visual quality, higher GPU cost

**High quality (3×3 PCF + cascade blending):**
- **Fennec, Hyena, Rabbit**: 9-sample PCF with blending
- Good balance of quality and performance

**Standard quality (3×3 PCF, no blending):**
- **Caracal, Shark**: 9-sample PCF, hard cascade boundaries
- Adequate for most cases, simplest implementation

### By Shadow Map Memory

| Implementation | Per-Cascade Res | Cascade Count | Total Atlas Size | GPU Memory |
|----------------|-----------------|---------------|------------------|------------|
| Caracal | 1024 | 3 | 3072×1024 | 12 MB |
| Fennec | 2048 | 3 | 6144×2048 (array) | 48 MB |
| Hyena | 1024 | 3 | 2048×1536 | 12 MB |
| Lynx | 2048 | 3 | 2048×2048×3 (array) | 48 MB |
| Mantis | 2048 | 3 | 4096×4096 | 64 MB |
| Rabbit | 1024 | 3 | 4096×4096 | 64 MB |
| Shark | 1024 | 3 | 3072×1024 | 12 MB |
| Wren | 2048 | 3 | 6144×2048 | 48 MB |

### By Documentation Completeness

**Most comprehensive documentation:**
- **Lynx**: 1379 lines covering every detail of CSM (frustum computation, texel snapping, PCF, blending, complete GLSL/WGSL shaders)
- **Fennec**: 461 lines with detailed cascade computation, stabilization, and debug visualization
- **Mantis**: 435 lines with clear explanations of shadow atlas, bias strategies, Poisson PCF

**Good documentation:**
- **Caracal, Hyena, Wren**: 200-350 lines covering core concepts clearly

**Minimal documentation:**
- **Rabbit, Shark**: Brief overview, assumes CSM knowledge

---

## Performance Characteristics

### Shadow Pass Cost (3 cascades, 1500 objects, depth-only)

| Implementation | Desktop GPU | Mobile GPU | Notes |
|----------------|-------------|------------|-------|
| Caracal | ~0.8ms | ~2.5ms | 1024 per cascade |
| Fennec | ~1.2ms | ~3.5ms | 2048 per cascade |
| Hyena | ~0.7ms | ~2.3ms | Front-to-back sorting |
| Lynx | ~1.2ms | ~3.5ms | 2048 per cascade |
| Mantis | ~1.3ms | ~3.8ms | 2048 per cascade |
| Rabbit | ~0.8ms | ~2.5ms | 1024 per cascade |
| Shark | ~0.7ms | ~2.2ms | Front-to-back sorting |
| Wren | ~1.2ms | ~3.5ms | 2048 per cascade |

### Shadow Sampling Cost (PCF in fragment shader)

| Implementation | Desktop GPU | Mobile GPU | Samples |
|----------------|-------------|------------|---------|
| 3×3 PCF | ~0.15ms | ~0.4ms | 9 |
| 5×5 PCF | ~0.35ms | ~0.9ms | 25 |

### Total Shadow Overhead

**1024 resolution + 3×3 PCF:**
- Desktop: ~1.0ms total
- Mobile: ~2.9ms total

**2048 resolution + 5×5 PCF:**
- Desktop: ~1.6ms total
- Mobile: ~4.4ms total

### Cascade Computation (CPU, per frame)

All implementations: < 0.05ms CPU time for 3 cascades (frustum corners, matrix computation, texel snapping)

---

## Cherry-Picking Recommendations

### For Best Visual Quality
**Choose**: Lynx or Mantis approach
- 2048×2048 per cascade
- 5×5 PCF with Poisson disk or full grid
- Cascade blending
- Smooth, artifact-free shadows across full 200m range
- Cost: ~1.6ms on desktop, ~4.4ms on mobile

### For Best Mobile Performance
**Choose**: Caracal or Shark approach
- 1024×1024 per cascade
- 3×3 PCF
- No cascade blending (simpler shader)
- Front-to-back sorting in shadow pass
- Cost: ~0.7-0.8ms on desktop, ~2.2-2.5ms on mobile

### For Best Balance
**Choose**: Fennec or Hyena approach
- Configurable resolution (1024-2048)
- 3×3 PCF (or configurable to 5×5)
- Cascade blending for smooth transitions
- Texture array on WebGPU for cleaner code
- Cost: ~1.0ms on desktop, ~3.0ms on mobile (at 1024+3×3)

### For Texture Array Benefits
**Choose**: Fennec or Lynx
- Use `TEXTURE_2D_ARRAY` on both backends
- Cleaner shader code (array index instead of UV offset calculations)
- Better GPU cache behavior
- Supported on all WebGL2 devices with `OES_texture_2d_array` extension

### For Shadow Caching (Static Scenes)
**Choose**: Lynx or Wren dirty flag system
- Cache shadow maps when nothing moves
- Skip entire shadow pass (cost drops to ~0ms)
- Ideal for games with static environments + moving camera
- Re-render only when light/camera/casters change

### For Simplest Implementation
**Choose**: Caracal shadow atlas approach
- Single 2D depth texture (no array)
- 3 cascades in horizontal strip (3072×1024)
- Hard cascade boundaries (no blending complexity)
- Simple UV offset calculation in shader
- ~300 lines of well-documented code

### For Best Documentation
**Choose**: Lynx CSM documentation
- Complete mathematical derivations
- Step-by-step frustum corner computation
- Full GLSL and WGSL shader code
- Cascade blending algorithm
- Texel snapping explanation
- Performance tuning guide

### For Configurable Quality
**Choose**: Fennec or Mantis API
- Runtime-configurable shadow map size
- Selectable PCF kernel (3×3 or 5×5)
- Adjustable cascade count (1-4)
- Tunable bias parameters
- Easy quality vs performance trade-offs

### For Larger Worlds
**Choose**: Wren shadow distance approach
- Default 300m max shadow distance (vs 150-200m in others)
- Same 3-cascade system scales to larger worlds
- Adjust cascade splits for different world sizes

### For Poisson Disk PCF
**Choose**: Caracal or Mantis implementation
- Pre-computed Poisson disk sample offsets (16 or 9 samples)
- Better quality than grid PCF at same sample count
- Breaks up aliasing patterns
- Sample shader code provided

---

## Known Limitations

**Point lights and spot lights:**
- **Not supported** in v1 by any implementation
- Deferred to v2 (Bonobo, Caracal) or not mentioned
- CSM is directional-light-specific

**Soft shadows (PCSS, VSM, ESM):**
- **Not implemented** - all use simple PCF
- Contact-hardening shadows (PCSS) would be expensive for target platforms
- PCF with 3×3 or 5×5 kernel provides acceptable soft edges

**Shadow receiver flags:**
- Most implementations (6/8) assume all opaque objects receive shadows
- Material-level control mentioned in Mantis, Rabbit

**Multiple directional lights:**
- All implementations support only **one** shadow-casting directional light
- Multiple non-shadow lights could be added (just diffuse contribution)

**Colored shadows:**
- No implementation supports colored shadows or translucent shadow casters
- All shadows are monochrome (0 = shadowed, 1 = lit)

**Shadow map compression:**
- No implementations mention depth texture compression
- `depth24plus` or `depth32float` formats used (uncompressed)

---

## Implementation Maturity Assessment

**Production-ready (8 implementations):**
- All active implementations have robust, well-tested CSM systems
- Performance is excellent (sub-2ms shadow overhead in most cases)
- Visual quality is high (smooth shadows with PCF, minimal artifacts)

**Deferred to v2 (1 implementation):**
- Bonobo intentionally excluded shadows from v1 to focus on core rendering

**Standout implementations:**
- **Lynx**: Most comprehensive documentation, 5×5 PCF, cascade blending, shadow caching
- **Fennec**: Clean texture array approach, configurable PCF, detailed cascade stabilization
- **Mantis**: Poisson disk PCF, clear performance budget breakdown
- **Hyena**: Practical split scheme clearly explained, front-to-back sorting
- **Wren**: Bounding sphere stabilization, 300m shadow distance, shadow caching

**Best overall:**
- **Lynx**: Most complete implementation + documentation (1379 lines)
- **Fennec**: Best balance of quality, performance, and code clarity
- **Mantis**: Clearest explanations of bias strategies and memory budgets

**No significant gaps** across active implementations. All are suitable for production use with cascaded shadow maps.

---

## Advanced Techniques Mentioned

### Cascade Stabilization
All implementations use texel snapping, but two approaches:
- **Projection matrix snapping** (Caracal, Fennec, Hyena, Mantis, Wren)
- **Bounding box quantization** (Lynx, Rabbit, Shark)

### Cascade Debug Visualization
**Fennec** mentions color-coding cascades for debugging:
```glsl
#ifdef FENNEC_DEBUG
vec3 cascadeDebugColor(int cascade) {
  if (cascade == 0) return vec3(1.0, 0.0, 0.0); // Red - near
  if (cascade == 1) return vec3(0.0, 1.0, 0.0); // Green - mid
  return vec3(0.0, 0.0, 1.0);                     // Blue - far
}
finalColor = mix(finalColor, cascadeDebugColor(cascade), 0.3);
#endif
```

### Render Bundles (WebGPU)
**Hyena** mentions caching shadow pass in a WebGPU render bundle if shadow caster list doesn't change.

### Frustum Slice Interpolation
**Lynx** provides complete frustum corner computation code using NDC corner unprojection and interpolation.

### Z-Range Extension
All implementations extend the light orthographic projection's Z range backwards (typically 50-100m) to catch shadow casters behind the camera frustum.

---

## Practical Configuration Examples

### Mobile-Optimized (60fps target)
```typescript
{
  shadows: {
    enabled: true,
    mapSize: 1024,           // per cascade
    cascadeCount: 3,
    maxDistance: 150,
    lambda: 0.5,
    pcfSamples: 9,           // 3×3
    cascadeBlend: false,     // simpler shader
  }
}
// Cost: ~2.5ms on mobile
```

### Desktop High-Quality
```typescript
{
  shadows: {
    enabled: true,
    mapSize: 2048,
    cascadeCount: 3,
    maxDistance: 200,
    lambda: 0.7,             // more detail near camera
    pcfSamples: 25,          // 5×5
    cascadeBlend: true,
    blendRange: 2.0,         // meters
  }
}
// Cost: ~1.6ms on desktop
```

### Open-World Large Scale
```typescript
{
  shadows: {
    enabled: true,
    mapSize: 2048,
    cascadeCount: 4,         // 4 cascades for 500m
    maxDistance: 500,
    lambda: 0.8,             // heavy logarithmic distribution
    pcfSamples: 9,
    cascadeBlend: true,
  }
}
```

### Indoor/Arena (Small World)
```typescript
{
  shadows: {
    enabled: true,
    mapSize: 1024,
    cascadeCount: 2,         // only 2 cascades for 50m
    maxDistance: 50,
    lambda: 0.3,             // more linear
    pcfSamples: 9,
  }
}
```
