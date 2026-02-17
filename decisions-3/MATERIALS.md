# MATERIALS.md - Material System

## Two Core Material Types

**Decision: BasicMaterial (unlit) + LambertMaterial (diffuse)** (universal agreement)

No PBR. Lambert diffuse is sufficient for the target aesthetic (stylized, low-poly games):

- **BasicMaterial**: No lighting calculations. Use for UI elements, skyboxes, unlit effects.
- **LambertMaterial**: Simple N dot L diffuse shading with directional + ambient light.

### Why No PBR

- Lambert: ~15 ALU ops per fragment. PBR (Cook-Torrance): ~80+ ALU ops.
- Lambert needs 2 textures (color + AO). PBR needs 5 (albedo + metallic + roughness + normal + AO).
- For stylized/low-poly art, Lambert provides excellent visual quality with marginal benefit from PBR.
- PBR can be added as a third material type in the future without changing the architecture.

## Material Index System

**Decision: 32-entry palette via UBO** (Fennec, Hyena, Mantis, Rabbit, Shark approach - the most common choice)

The material index system is a core feature that enables single-mesh, single-draw-call multi-material rendering. Models are exported as single meshes with a `_materialindex` vertex attribute. Each index maps to a palette entry with color and emissive properties.

### Palette Structure

**Decision: Combined struct array** (Hyena, Lynx, Mantis, Wren approach - cleaner than separate arrays)

```glsl
struct PaletteEntry {
  vec4 colorAndOpacity;           // rgb + opacity
  vec4 emissiveAndIntensity;      // rgb + intensity
}

layout(std140) uniform MaterialPalette {
  PaletteEntry entries[32];
} palette;
```

32 entries at 32 bytes each = 1024 bytes UBO. This fits well within minimum UBO size limits on all WebGL2 devices.

### Why 32 Entries (Not 16 or 64)

- 16 (Caracal, Wren): Too few for detailed characters with many material zones.
- 64 (Lynx): Wastes UBO space for typical game assets. Most models use 5-15 material indices.
- 32: Good balance. Covers virtually all game asset needs. 1024 bytes is well within UBO limits.

### Overflow Strategy

For the rare case where a model needs >32 material indices, fall back to a 1D texture lookup (Fennec, Hyena, Mantis approach). This is a rare path that does not need to be optimized.

### Shader Usage

```glsl
flat in uint v_materialIndex;

void main() {
  PaletteEntry entry = palette.entries[v_materialIndex];
  vec3 baseColor = entry.colorAndOpacity.rgb;
  float opacity = entry.colorAndOpacity.a;
  vec3 emissive = entry.emissiveAndIntensity.rgb * entry.emissiveAndIntensity.a;
  // ...
}
```

The `flat` interpolation qualifier passes the integer index without interpolation.

## Vertex Colors

**Decision: Multiplicative combine** (universal agreement)

```
finalColor = paletteColor * vertexColor * textureColor
```

Vertex colors modulate the base color from the palette (or material color if no palette). This allows tinting, ambient occlusion baking, and color variation without additional textures.

## Emissive Output for Bloom

**Decision: MRT with separate emissive render target** (Hyena, Rabbit, Shark, Wren approach)

During the opaque pass, the fragment shader writes to two render targets:

- **RT0**: Final lit color (for display)
- **RT1**: Emissive color * emissive intensity (for bloom extraction)

This is cleaner than encoding emissive in the alpha channel (Caracal) or using a luminance threshold (Fennec, Lynx, Mantis):

- No threshold tuning needed
- Per-vertex emissive control via material index palette entries
- Bloom extracts directly from RT1 without any filtering heuristic
- Emissive objects bloom regardless of their brightness

### Per-Vertex Emissive via Material Index

```typescript
const material = createLambertMaterial({
  palette: [
    { color: [1, 1, 1], emissive: [0, 0, 0], emissiveIntensity: 0 },     // White, no glow
    { color: [0, 0, 0] },                                                   // Black, no glow
    { color: [0, 0.8, 0.8], emissive: [0, 0.8, 0.8], emissiveIntensity: 0.7 } // Teal with bloom
  ]
})
```

## Shader Variant System

**Decision: Feature flags with lazy compilation and caching** (majority approach)

### Feature Flags

| Flag | Triggered By | Effect |
|------|-------------|--------|
| `HAS_COLOR_TEXTURE` | `material.map !== null` | Sample color texture |
| `HAS_AO_TEXTURE` | `material.aoMap !== null` | Sample AO texture |
| `HAS_VERTEX_COLORS` | `material.vertexColors === true` | Read vertex color attribute |
| `HAS_MATERIAL_INDEX` | Geometry has `_materialindex` | Palette lookup |
| `HAS_SKINNING` | Mesh is SkinnedMesh | Bone matrix transform |
| `HAS_EMISSIVE` | Any emissive intensity > 0 | Write to emissive RT |
| `SHADOW_RECEIVE` | `material.receiveShadow === true` | Sample shadow map |
| `IS_TRANSPARENT` | `material.transparent === true` | OIT output |

### Compilation Strategy

**Decision: Lazy compilation with caching + warm-up of common variants** (hybrid of majority lazy + Caracal eager)

Shader variants are compiled on first use and cached by feature bitmask. During the asset loading phase, the most common variants are pre-compiled to avoid hitching:

```typescript
// Pre-warm during loading
device.precompileVariants([
  LAMBERT | SHADOW_RECEIVE,
  LAMBERT | SHADOW_RECEIVE | HAS_COLOR_TEXTURE,
  LAMBERT | SHADOW_RECEIVE | HAS_MATERIAL_INDEX,
  LAMBERT | SHADOW_RECEIVE | HAS_SKINNING,
  BASIC,
])
```

Cache lookup: `Map<featureBitmask, CompiledShader>`. Typical scene uses 10-30 unique variants.

## Transparency via OIT

Transparent materials are routed to the Weighted Blended OIT pass. They set `depthWrite: false` and `depthTest: true` (read-only depth). See TRANSPARENCY.md for details.

## Pipeline State

Each material contributes to the pipeline descriptor:

- **Opaque**: `depthWrite: true`, `depthTest: true`, no blending
- **Transparent**: `depthWrite: false`, `depthTest: true`, OIT blend modes
- **Double-sided**: `cullMode: 'none'` (default: `cullMode: 'back'`)

## Material Sorting Key

Materials contribute to the 64-bit draw call sort key:
- Pipeline ID (bits 58-48): shader variant + blend/depth state
- Material ID (bits 47-32): unique material identifier for state grouping

This groups draws by pipeline first (most expensive switch), then by material (moderate cost), minimizing state changes across 2000 draws.

## Default Textures

**Decision: Bind 1x1 white pixel when no texture provided** (Mantis, Rabbit, Shark approach)

Instead of branching in the shader (`if (hasTexture) sample else use white`), always bind a texture. When no user texture is set, bind a 1x1 white pixel texture. The multiply by white is a no-op, and the shader code stays branchless:

```glsl
vec4 texColor = texture(u_colorMap, v_uv);
finalColor = baseColor * texColor.rgb;  // Always works
```
