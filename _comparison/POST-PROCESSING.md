# Post-Processing Comparison

This document compares post-processing approaches across all 9 engine implementations.

## Universal Agreement

All implementations agree on these fundamental principles:

1. **Unreal-Style Bloom** - All 9 implementations that support bloom use the progressive downsample/upsample approach inspired by Unreal Engine
2. **Per-Vertex Emissive Control** - Bloom is driven by emissive values in the material system, allowing per-region control via material index palettes
3. **Full-Screen Triangle** - Post-processing passes use a single oversized triangle instead of a quad to avoid diagonal seam artifacts
4. **Separate MRT for Emissive** - Implementations using bloom write emissive contributions to a second render target during the main pass

## Key Variations

### 1. Bloom Implementation Status

**Implementations with full bloom (8)**:
- caracal, fennec, hyena, lynx, mantis, rabbit, shark, wren

**Minimal/deferred bloom (1)**:
- bonobo (marked as "minimal for v1", single MRT pass planned)

### 2. Blur Algorithm Choice

**Dual Kawase Blur (1)**:
- **fennec** - Uses Dual Kawase downsample (13-tap, 4 bilinear fetches) and upsample (9-tap, 8 bilinear fetches) kernels

**Gaussian Blur (5)**:
- **bonobo** - Separable Gaussian blur (5-tap or 9-tap kernel)
- **caracal** - Separable Gaussian blur with 5-tap kernel (σ=2.0)
- **lynx** - Separable 9-tap Gaussian blur
- **rabbit** - 9-tap Gaussian blur (horizontal + vertical passes)
- **shark** - Separable Gaussian blur (9-tap, weights: [0.227027, 0.194596, 0.121621, 0.054054, 0.016216])

**Jimenez 13-Tap Filter (2)**:
- **hyena** - 13-tap downsample filter with Karis average on first pass
- **wren** - 13-tap downsample filter (Jimenez/Call of Duty: Advanced Warfare technique)

**Simple Filtering (1)**:
- **mantis** - Karis average for first downsample, box filter for subsequent levels

### 3. Downsample Levels

**5 levels (6)**:
- bonobo, caracal, hyena, lynx, mantis, wren

**4 levels (2)**:
- caracal (also mentions 5-6), fennec (configurable), shark

**Variable (1)**:
- rabbit (default 5, configurable via mipCount)

### 4. Threshold Handling

**No Threshold (Physically Based) (4)**:
- **fennec** - Default threshold 0.0, any emissive blooms
- **hyena** - Threshold ~0.1 on dedicated emissive buffer
- **mantis** - No threshold needed (emissive-only buffer)
- **wren** - Default threshold 0.0, physically based approach

**Soft Threshold (3)**:
- **lynx** - Soft knee threshold with quadratic ease
- **rabbit** - Soft threshold with knee smoothing
- **caracal** - Uses emissive intensity threshold (typically 0.0)

**Hard Threshold (1)**:
- **bonobo** - Threshold ~0.8-1.0 (user-configurable)

**Per-Emissive Control (1)**:
- **shark** - Threshold from emissive alpha channel

### 5. Tone Mapping

**ACES Filmic (5)**:
- caracal, lynx, rabbit, hyena (planned), mantis (planned)

**Simple Reinhard (2)**:
- shark (current), wren (fallback option)

**Planned/Deferred (2)**:
- bonobo (v2+ feature), fennec (not explicitly mentioned)

**ACES Implementation**:
All implementations using ACES use Stephen Hill's fit with the same constants:
```glsl
a = 2.51, b = 0.03, c = 2.43, d = 0.59, e = 0.14
result = clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0)
```

### 6. Render Target Formats

**RGBA16F for Bloom Chain (6)**:
- fennec, hyena, lynx, mantis, rabbit, wren

**RGBA8 for Bloom Chain (2)**:
- caracal (explicitly chose RGBA8 to save memory), shark

**Not Specified (1)**:
- bonobo (planned)

**Emissive Buffer Formats**:
- Most use RGBA16F or RGBA8 depending on HDR requirements
- fennec uses half-res emissive buffer for performance

### 7. MSAA Integration

**MSAA with Bloom (5)**:
- caracal, fennec, hyena, shark, mantis

**MSAA Mentioned as Future (1)**:
- bonobo (v2 feature)

**No MSAA Discussion (3)**:
- lynx, rabbit, wren

**MSAA Strategy**:
- All implementations with MSAA resolve the opaque pass before bloom processing
- Bloom operates on resolved (1x) textures to save memory

## Implementation Breakdown

### Bloom Pipeline Architecture

#### Progressive Downsample/Upsample (All 9)

All implementations follow this general structure:
```
Emissive RT → Downsample Chain → Upsample Chain → Composite
```

**Starting Resolution**:
- **Half-res start (3)**: fennec, hyena, mantis (start bloom chain at canvas/2)
- **Full-res start (4)**: caracal, lynx, rabbit, shark (extract at full resolution, then downsample)
- **Not specified (2)**: bonobo, wren

**Downsample Strategy**:
- **Karis Average First Pass (6)**: caracal, hyena, lynx, mantis, rabbit, wren (prevent firefly artifacts)
- **Box/Bilinear Subsequent (5)**: caracal, hyena, mantis, rabbit, wren
- **Dual Kawase Throughout (1)**: fennec
- **13-Tap Throughout (2)**: hyena, wren
- **Not specified (1)**: bonobo

**Upsample Strategy**:
- **Tent Filter (6)**: hyena, lynx, mantis, rabbit, shark, wren (3x3 or 9-tap)
- **Dual Kawase (1)**: fennec
- **Gaussian Blur (2)**: caracal, shark
- **Not specified (1)**: bonobo

**Additive Blending**:
All 8 implementations with full bloom use additive blending to accumulate bloom levels during upsample

### Configuration APIs

All implementations expose similar configuration:

```typescript
interface BloomConfig {
  enabled: boolean           // Default: true
  threshold: number          // Brightness threshold (varies: 0.0-1.0)
  intensity/strength: number // Bloom strength (default: 0.5-1.0)
  radius: number             // Blur spread (default: 0.85-1.0)
  levels/mipCount: number    // Downsample levels (default: 4-5)
}
```

Specific variations:
- **caracal**: Adds `emissiveIntensity` per material index entry
- **fennec**: Emphasizes bloom driven by emissive values, minimal global controls
- **shark**: Includes separate `bloomStrength` for final composite
- **rabbit**: Adds `softThreshold` parameter for knee smoothing

### Performance Budgets

Reported bloom costs (approximate, at 1080p):

| Implementation | Reported Cost | Notes |
|---|---|---|
| bonobo | 1-2ms | Target for full bloom (v1: minimal) |
| caracal | Not specified | Emphasizes efficiency via reduced formats |
| fennec | ~0.9ms total | Including all post-processing |
| hyena | ~1.1ms | 5 downsample + 5 upsample + composite |
| lynx | ~0.55ms | Includes tone mapping |
| mantis | ~1.1ms | 5 downsample + 5 upsample |
| rabbit | Not specified | Mentions efficient multi-scale approach |
| shark | ~0.8ms | Bloom blur only (8 passes) |
| wren | ~0.22ms | Fastest reported (most work at low res) |

### Memory Usage

**Bloom Chain Memory (1080p)**:
- **RGBA16F format**: ~16 MB total for 5-level chain (lynx, wren reports ~2.5 MB)
- **RGBA8 format**: ~8 MB total for 5-level chain (caracal)
- All implementations note geometric series: total memory ≈ 1.33× base resolution

## Special Features

### Caracal
- Selective bloom driven entirely by emissive channel
- Material index system with per-entry emissive controls
- RGBA8 bloom chain for memory efficiency

### Fennec
- Dual Kawase blur (faster than Gaussian, better quality)
- MRT bloom output during main pass (skip threshold)
- Integrated with OIT and MSAA in complete pipeline

### Hyena
- Dedicated emissive buffer via MRT
- Karis average on first downsample to prevent fireflies
- 13-tap filter (Jimenez/CoD technique)

### Lynx
- Soft threshold with quadratic knee transition
- Detailed 13-tap downsample, 9-tap upsample
- ACES tone mapping with exposure control

### Mantis
- Per-vertex emissive via material index palette
- MRT emissive isolation (no threshold pass needed)
- Emphasis on zero-allocation approach

### Shark
- Dual-target MSAA integration
- Gaussian blur with configurable radius
- Dynamic pass skipping when bloom disabled

### Wren
- Jimenez 13-tap filter (anti-firefly)
- 3x3 tent upsample filter
- Physically based (no threshold by default)
- Fastest reported performance

## Recommendations for Cherry-Picking

**For Best Performance**:
- Use **wren's** approach: half-res start, 13-tap downsample, physically based (no threshold)
- Consider **fennec's** Dual Kawase blur for better quality/performance ratio

**For Best Quality**:
- Use **lynx's** soft threshold with knee transition
- Combine with **hyena's** Karis average on first downsample
- Use 5+ bloom levels for wide glow spread

**For Memory Efficiency**:
- Follow **caracal's** RGBA8 bloom chain approach
- Start bloom chain at half-res like **fennec**, **hyena**, **mantis**

**For Simplicity**:
- Follow **mantis's** approach: MRT emissive output, Karis average first pass, box filter subsequent
- No threshold needed, straightforward implementation

**For Material Artist Control**:
- Use **caracal's** or **mantis's** material index system with per-entry emissive controls
- Allows precise per-vertex bloom without separate textures

**MSAA Integration**:
- Follow **fennec's** or **caracal's** approach: MSAA render targets, resolve before bloom
- Bloom operates on resolved 1x textures to save memory

**Tone Mapping**:
- Use **ACES filmic** (Stephen Hill's fit) for film-like look - most popular choice
- **Reinhard** for simpler/faster tone mapping (shark, wren)
- Always gamma correct if not using sRGB framebuffer
