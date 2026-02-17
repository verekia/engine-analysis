# MATERIALS.md - Decisions

## Decision: Two Materials, 32-Entry Palette, MRT Emissive

### Material Types: Basic + Lambert Only

Universal agreement across all 9 - exactly two materials:

- **BasicMaterial**: Unlit, no lighting calculations. For UI elements, debug visualizations, skyboxes.
- **LambertMaterial**: Diffuse N*L shading. For all game content.

No PBR. Lambert is 3-5x cheaper than Cook-Torrance on mobile GPUs and provides excellent visual quality for the target aesthetic (stylized/low-poly). PBR can be added as a third material type later without architectural changes.

### Material Index System: 32-Entry UBO Palette

**Chosen**: 32-entry palette stored in a UBO as combined struct array (5/9 implementations use 32 entries)
**Rejected**: 16 entries (Caracal/Wren - too limiting for complex characters), 64 entries (Lynx - wastes UBO space for typical use cases)

```glsl
struct PaletteEntry {
  vec4 colorAndOpacity;         // rgb = color, a = opacity
  vec4 emissiveAndIntensity;    // rgb = emissive color, a = intensity
};
uniform PaletteEntry u_palette[32];  // 32 * 32 bytes = 1024 bytes
```

**Combined struct** (Hyena/Lynx/Mantis/Wren style, 4/9) rather than separate arrays (Caracal/Fennec/Rabbit/Shark) because it keeps each entry's data contiguous in the UBO, which is cleaner for indexed access.

The `_materialindex` vertex attribute (uint8) selects the palette entry per vertex. This allows a single mesh with a single draw call to have multiple "materials" - critical for the use case described in the prompt (models with `_materialindex` attributes for per-region color and emissive control).

### Vertex Colors

Universal agreement: multiplicative combine.

```
finalColor = paletteColor * vertexColor * textureColor
```

When no material index is present: `finalColor = materialColor * vertexColor * textureColor`

### Emissive Output: MRT (Dual Render Target)

**Chosen**: Write emissive to a second render target during the opaque pass via MRT (4/9: Hyena, Rabbit, Shark, Wren)
**Rejected**: Additive after lighting + luminance threshold (Fennec/Lynx/Mantis) - requires a threshold pass and loses precision
**Rejected**: Alpha channel encoding (Caracal) - clever but limited

MRT approach during the opaque pass:

```glsl
layout(location = 0) out vec4 fragColor;      // Lit scene color
layout(location = 1) out vec4 fragEmissive;    // Emissive for bloom

void main() {
  vec3 litColor = ambient + diffuse * NdotL * shadow;
  vec3 emissive = paletteEntry.emissive * paletteEntry.intensity;

  fragColor = vec4(litColor + emissive, 1.0);
  fragEmissive = vec4(emissive, 1.0);
}
```

The bloom pipeline reads the emissive RT directly - no threshold pass needed. This gives artists precise per-vertex bloom control via the material index palette.

### Shader Variant System

Universal agreement: compile-time feature flags, cached by bitmask.

Feature flags that trigger shader variants:

| Flag | Triggered By |
|------|-------------|
| `HAS_COLOR_TEXTURE` | `material.map !== null` |
| `HAS_AO_TEXTURE` | `material.aoMap !== null` |
| `HAS_VERTEX_COLORS` | Geometry has color attribute |
| `HAS_MATERIAL_INDEX` | Geometry has `_materialindex` attribute |
| `HAS_SKINNING` | Mesh is SkinnedMesh |
| `HAS_EMISSIVE` | Emissive intensity > 0 or material index has emissive entries |
| `RECEIVE_SHADOW` | `material.receiveShadow === true` |
| `IS_TRANSPARENT` | `material.transparent === true` |

### Shader Compilation: Lazy + Cached

**Chosen**: Compile on first use, cache by feature bitmask (7/9 use lazy caching)
**Considered**: Eager pre-compilation (Caracal) - prevents hitching but requires knowing all variants upfront

```typescript
const shaderCache = new Map<number, GALShaderModule>()

const getShader = (featureMask: number): GALShaderModule => {
  let shader = shaderCache.get(featureMask)
  if (!shader) {
    shader = compileShaderVariant(featureMask)
    shaderCache.set(featureMask, shader)
  }
  return shader
}
```

Optionally provide a `precompileShaders(featureMasks[])` API for users who want to avoid first-use hitching.

### Transparency

Transparent materials route to WBOIT pass (see TRANSPARENCY.md):
- `depthWrite: false`, `depthTest: true` (read-only depth)
- Opacity sources: material `opacity`, texture alpha, vertex color alpha, palette entry opacity

### Texture Handling

**Uncompressed**: RGBA8 for color and AO maps
**Compressed**: BC7 (desktop), ASTC (mobile), ETC2 (fallback) via KTX2/Basis transcoding

**Default textures**: Bind a 1x1 white pixel texture when no texture is provided (Mantis/Rabbit/Shark approach, 3/9). This avoids shader branching - the `HAS_COLOR_TEXTURE` flag still controls whether to sample, but having a default simplifies bind group creation.

**Separate texture + sampler** on WebGPU, combined `sampler2D` on WebGL2.

### Pipeline State from Material

**Depth state**:
- Opaque: `depthWrite: true, depthTest: true`
- Transparent: `depthWrite: false, depthTest: true`

**Blend state**:
- Opaque: None (replace)
- Transparent: OIT accumulation/revealage blend modes

**Cull state**:
- Default: Back-face culling
- `material.doubleSided`: No culling

### Material Sorting

Materials participate in the 64-bit sort key (see RENDERER.md). Material ID occupies 16 bits, allowing up to 65536 unique material instances before overflow. In practice, scenes use 10-200 materials.
