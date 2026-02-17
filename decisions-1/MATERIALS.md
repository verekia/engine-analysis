# MATERIALS - Design Decisions

## Material Types

**Decision: Two core materials — BasicMaterial (unlit) and LambertMaterial (diffuse)**

- Sources: All 9 implementations agree
- No PBR — Lambert is sufficient for the stylized/low-poly aesthetic target
- Lambert: ~15 ALU ops/fragment vs PBR ~80+ ops — critical for mobile performance
- PBR can be added as a third material type later without architecture changes

## Material Index System

**Decision: 32-entry palette stored as combined struct array in UBO**

- Sources: Fennec, Hyena, Mantis, Rabbit, Shark (5/9) use 32 entries — best balance
- Rejected: 16 entries (Caracal, Wren) — too limiting for detailed characters
- Rejected: 64 entries (Lynx) — excessive UBO size for typical use

```glsl
struct PaletteEntry {
  vec4 colorAndOpacity;          // rgb + opacity
  vec4 emissiveAndIntensity;     // rgb + intensity
}

layout(std140) uniform MaterialPalette {
  PaletteEntry entries[32];
} palette;
```

Combined struct (Hyena, Lynx, Mantis, Wren) rather than separate arrays (Caracal, Fennec, Rabbit, Shark) because it keeps color and emissive data for the same index in the same cache line.

Palette size: 32 entries × 32 bytes = 1024 bytes UBO — well within mobile UBO limits.

For models exceeding 32 material indices (rare), fall back to 1D texture lookup (Fennec, Hyena, Mantis approach).

## Vertex Colors

**Decision: Multiplicative combine (universal agreement)**

```
finalColor = paletteColor × vertexColor × textureColor
```

When material index is not present:
```
finalColor = materialColor × vertexColor × textureColor
```

## Emissive Output for Bloom

**Decision: MRT (Multiple Render Target) — write emissive to separate RT1 during opaque pass**

- Sources: Hyena, Rabbit, Shark, Wren (4/9 use MRT)
- Rejected: Alpha channel encoding (Caracal) — loses alpha for other purposes
- Rejected: Luminance threshold only (Fennec, Lynx, Mantis) — less precise control

```glsl
// Fragment shader outputs:
layout(location = 0) out vec4 fragColor;     // lit color
layout(location = 1) out vec4 fragEmissive;  // emissive for bloom

void main() {
  // ... lighting ...
  fragColor = vec4(litColor, 1.0);
  fragEmissive = vec4(emissiveColor * emissiveIntensity, 1.0);
}
```

Bloom extraction reads RT1 directly — no threshold pass needed. Per-vertex emissive control via material index palette.

## Shader Variant System

**Decision: Feature flag bitmask + lazy compilation with cache + pre-warm common variants**

- Sources: Feature flags from all 9; bitmask from Mantis; lazy+cache from 8/9; pre-warm from Caracal

```typescript
const enum ShaderFeature {
  COLOR_TEXTURE    = 1 << 0,
  AO_TEXTURE       = 1 << 1,
  VERTEX_COLORS    = 1 << 2,
  MATERIAL_INDEX   = 1 << 3,
  SKINNING         = 1 << 4,
  EMISSIVE         = 1 << 5,
  SHADOW_RECEIVE   = 1 << 6,
  TRANSPARENT      = 1 << 7,
}

// Cache: featureMask → compiled shader
const shaderCache = new Map<number, GALShaderModule>()
```

Pre-compile the most common variants during asset loading (not on first draw) to avoid frame spikes. Lazy-compile uncommon variants on demand and cache.

## Default Textures

**Decision: 1×1 white pixel default textures**

- Sources: Mantis, Rabbit, Shark (3/9)
- Bind 1×1 white texture when no texture is provided
- Avoids `#ifdef HAS_TEXTURE` branching in shaders — always sample, multiply by 1.0 if no texture
- Simpler shader code, negligible performance cost

## Transparency Routing

**Decision: Transparent materials routed to WBOIT pass**

- Sources: All 9 implementations agree
- `material.transparent = true` → renders in OIT accumulation pass
- `material.transparent = false` → renders in opaque pass
- Alpha cutoff (`alphaTest`) for near-opaque objects uses discard in opaque pass

## Pipeline State per Material

**Decision: Immutable pipeline derived from material + geometry layout**

Depth state:
- Opaque: `depthWrite: true, depthTest: true`
- Transparent: `depthWrite: false, depthTest: true`

Blend state:
- Opaque: None (replace)
- Transparent: OIT accumulation/revealage blend modes

Cull state:
- Default: back-face culling
- `material.doubleSided = true` → no culling

## Material Sorting Key

**Decision: Material ID encoded in 64-bit sort key (bits 16-31)**

Material contributes 16 bits to the sort key, positioned after pipeline ID. Materials with the same textures and parameters share the same ID, minimizing bind group switches.

## Texture Handling

**Decision: Separate texture + sampler on WebGPU, combined on WebGL2**

```wgsl
// WebGPU
@group(1) @binding(2) var t_colorMap: texture_2d<f32>;
@group(1) @binding(3) var s_colorMap: sampler;
```

```glsl
// WebGL2
uniform sampler2D u_colorMap;
```

KTX2/Basis textures transcoded to best available format: ASTC > BC7 > ETC2 > RGBA8.

## Material API

```typescript
const material = createLambertMaterial({
  color: [1, 0.8, 0.6],
  map: colorTexture,            // optional color texture
  aoMap: aoTexture,             // optional AO texture
  opacity: 1.0,
  transparent: false,
  doubleSided: false,
  receiveShadow: true,
  vertexColors: false,
  palette: [                    // material index palette
    { color: [1, 1, 1], emissive: [0, 0, 0], emissiveIntensity: 0 },
    { color: [0, 0, 0], emissive: [0, 1, 1], emissiveIntensity: 0.7 },
  ],
})
```
