# POST-PROCESSING.md - Decisions

## Decision: Unreal Bloom with 13-Tap Downsample, Karis Average, ACES Tone Mapping

### Bloom: Unreal-Style Progressive Downsample/Upsample

Universal agreement across all 9 implementations. The prompt explicitly requests "Unreal Bloom."

Pipeline:

```
Emissive RT (from MRT) -> Downsample Chain -> Upsample Chain -> Composite
```

### Starting Resolution: Half-Res

**Chosen**: Start bloom chain at canvas/2 (3/9: Fennec, Hyena, Mantis)
**Rejected**: Full-res start (4/9) - doubles memory and GPU cost of the first downsample for negligible quality difference

Since bloom is inherently a blurry effect, starting at half resolution is visually indistinguishable from full resolution.

### Downsample Filter: 13-Tap (Jimenez) with Karis Average on First Pass

**Chosen**: 13-tap downsample filter (2/9: Hyena, Wren - Jimenez/Call of Duty: Advanced Warfare technique) with Karis average on first pass (6/9 use Karis)
**Rejected**: Gaussian blur (5/9) - requires two separate passes (horizontal + vertical) per level, more draw calls
**Rejected**: Dual Kawase (Fennec) - good quality but less proven than the Jimenez approach

The 13-tap filter samples a pattern that naturally covers the 2x2 texel neighborhood with good anti-aliasing properties, in a single pass per mip level.

**Karis average** on the first downsample prevents firefly artifacts from extremely bright pixels dominating the bloom:

```glsl
// First downsample only
float weight = 1.0 / (1.0 + luminance(color));
accumColor += color * weight;
accumWeight += weight;
```

### Upsample Filter: 3x3 Tent Filter

**Chosen**: 3x3 tent (9-tap) upsample filter (6/9: Hyena, Lynx, Mantis, Rabbit, Shark, Wren)

Each upsample level blends additively with the next larger mip, creating the progressive glow effect.

### Downsample Levels: 5

**Chosen**: 5 downsample levels (6/9: Bonobo, Caracal, Hyena, Lynx, Mantis, Wren)

At 1080p starting from half-res:
- Level 0: 960x540
- Level 1: 480x270
- Level 2: 240x135
- Level 3: 120x68
- Level 4: 60x34

Total bloom chain memory (RGBA16F): ~2.5MB (geometric series: ~1.33x base resolution)

### Threshold: Physically Based (No Threshold)

**Chosen**: Default threshold 0.0 - any emissive value blooms (4/9: Fennec, Hyena, Mantis, Wren)
**Rejected**: Hard threshold (Bonobo) - loses soft glow from low-intensity emissive
**Rejected**: Soft knee threshold (Lynx/Rabbit) - adds complexity without clear benefit when using MRT emissive

Since the bloom pipeline reads from the dedicated emissive RT (via MRT, see MATERIALS.md), only emissive content enters the bloom chain. No threshold pass is needed. This gives artists precise control: emissive intensity in the material palette directly controls bloom strength.

### Render Target Format: RGBA16F

**Chosen**: RGBA16F for the bloom chain (6/9: Fennec, Hyena, Lynx, Mantis, Rabbit, Wren)
**Rejected**: RGBA8 (Caracal) - saves memory but loses HDR precision in the bloom chain, causing banding

### Full-Screen Triangle

Universal agreement: use a single oversized triangle for all full-screen passes instead of a quad, to avoid the diagonal seam artifact.

### Tone Mapping: ACES Filmic

**Chosen**: ACES filmic tone mapping (5/9: Caracal, Lynx, Rabbit, Hyena, Mantis)
**Rejected**: Reinhard (Shark/Wren) - too simple, washes out bright colors

Stephen Hill's ACES approximation (universal constants):

```glsl
vec3 ACESFilm(vec3 x) {
  const float a = 2.51;
  const float b = 0.03;
  const float c = 2.43;
  const float d = 0.59;
  const float e = 0.14;
  return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}
```

Applied in the final blit pass after bloom compositing.

### MSAA Integration

**Chosen**: MSAA resolve before bloom processing (universal agreement)

- Opaque pass renders to 4x MSAA targets (color + depth + emissive)
- MSAA resolved to 1x targets
- Bloom operates on resolved emissive at 1x
- OIT operates at 1x (no MSAA for OIT buffers - Mantis approach, saves 4x memory on mobile)

### Bloom Configuration API

```typescript
{
  bloom: {
    enabled: true,
    intensity: 0.5,       // Bloom strength in final composite
    radius: 0.85,         // Blur spread factor
    levels: 5,            // Downsample levels
  }
}
```

Bloom strength is primarily controlled per-vertex via the material index palette's emissive intensity, not via a global threshold.

### Performance Budget

Target: <1ms at 1080p (Wren reports 0.22ms, Lynx 0.55ms, Fennec 0.9ms including all post-processing)

The half-res start + 5 levels approach keeps most work at very low resolutions. The 13-tap filter is a single pass per level (10 passes total: 5 down + 5 up), each operating on progressively smaller textures.

### Final Blit Pass

Single full-screen triangle that combines:
1. Resolved opaque color
2. OIT composite result
3. Bloom upsample result
4. ACES tone mapping
5. Gamma correction (if not using sRGB framebuffer)

Output to the screen/swap chain.
