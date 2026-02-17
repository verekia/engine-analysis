# Material System

## Overview

Mantis provides two material types optimized for stylized and low-poly games:

1. **BasicMaterial** — Unlit. Color comes from a solid color, a texture, vertex
   colors, or the material index palette. No lighting calculations. Ideal for
   UI elements, skyboxes, flat-shaded art, and emissive surfaces.

2. **LambertMaterial** — Diffuse lighting via N·L (normal dot light direction)
   plus ambient light. Supports shadows, AO textures, vertex colors, and the
   material index palette. The cheapest lighting model that still reads as "3D".

Both materials support the full feature matrix below. Features are toggled via
boolean flags that compile to shader variants — no runtime branching.

## Feature Matrix

| Feature | Basic | Lambert | Shader Flag |
|---|---|---|---|
| Solid base color | Yes | Yes | — (always present) |
| Color texture | Yes | Yes | `HAS_COLOR_TEXTURE` |
| AO texture | No | Yes | `HAS_AO_TEXTURE` |
| Vertex colors | Yes | Yes | `HAS_VERTEX_COLORS` |
| Material index palette | Yes | Yes | `HAS_MATERIAL_INDEX` |
| Emissive (per index) | Yes | Yes | `HAS_MATERIAL_INDEX` (implicit) |
| Skinning | Yes | Yes | `HAS_SKINNING` |
| Shadows (receive) | No | Yes | `HAS_SHADOWS` |
| Transparency (alpha) | Yes | Yes | `IS_TRANSPARENT` |

## Material Index System

This is Mantis's most distinctive material feature. Models are authored as
**single meshes** with a per-vertex `uint8` attribute called `_materialIndex`.
Each index maps to an entry in a **material palette** — an array of colors and
emissive values.

This enables:
- A character mesh where skin is index 0 (beige), clothing is index 1 (blue),
  eyes are index 2 (white with emissive 0.3), sword glow is index 3 (teal with
  emissive 0.9)
- All rendered in **a single draw call**
- Bloom applies automatically to vertices with emissive > 0

### Palette Definition

```typescript
const material = new LambertMaterial({
  // Base color when no material index or vertex color is present
  color: [1, 1, 1],

  // Material index palette — up to 32 entries per material
  palette: [
    { color: [1.0, 0.9, 0.8] },                                  // index 0: skin
    { color: [0.1, 0.2, 0.8] },                                  // index 1: blue clothing
    { color: [1.0, 1.0, 1.0], emissive: [1, 1, 1], emissiveIntensity: 0.3 }, // index 2: glowing eyes
    { color: [0.0, 0.8, 0.7], emissive: [0, 0.8, 0.7], emissiveIntensity: 0.9 }, // index 3: sword glow
  ],

  shadows: true,
})
```

### GPU Representation

The palette is stored in the per-material uniform block as two arrays:

```
materialColors:   vec4[32]   // rgb color + padding, indexed by _materialIndex
emissiveColors:   vec4[32]   // rgb emissive × intensity + intensity in w
```

For materials with more than 32 palette entries (rare), the palette overflows
to a 1D texture lookup. The shader switches to texture fetch via a
`HAS_PALETTE_TEXTURE` flag.

### Vertex Shader Flow

```
1. Read position, normal from vertex attributes
2. If HAS_SKINNING: apply bone transforms
3. Transform position by modelMatrix → worldPosition
4. Transform normal by normalMatrix → worldNormal
5. Pass through UV, vertexColor, materialIndex to fragment
```

### Fragment Shader Flow

```
1. Start with baseColor = material.color (vec3 uniform)
2. If HAS_COLOR_TEXTURE: baseColor *= texture(colorMap, uv).rgb
3. If HAS_VERTEX_COLORS: baseColor *= vertexColor.rgb
4. If HAS_MATERIAL_INDEX: baseColor *= palette[materialIndex].rgb

5. If IS_LAMBERT:
   a. diffuse = max(dot(worldNormal, lightDir), 0.0)
   b. lighting = ambientColor + dirLightColor × diffuse
   c. If HAS_SHADOWS: lighting *= shadowFactor (CSM lookup)
   d. If HAS_AO_TEXTURE: lighting *= texture(aoMap, uv).r
   e. baseColor *= lighting

6. Determine emissive:
   a. emissive = vec3(0)
   b. If HAS_MATERIAL_INDEX: emissive = palette[materialIndex].emissive

7. Output:
   a. Color target: vec4(baseColor + emissive, alpha)
   b. Emissive target: vec4(emissive, 1.0)  // for bloom extraction
```

The emissive output goes to a separate render target (MRT attachment 1) which
the bloom pipeline reads. This means bloom only affects vertices/pixels that
have non-zero emissive — no threshold needed in the bloom shader.

## Alpha and Transparency

Materials can be transparent by setting `transparent: true` and providing an
alpha value or texture with alpha channel:

```typescript
const glassMaterial = new BasicMaterial({
  color: [0.8, 0.9, 1.0],
  opacity: 0.3,
  transparent: true,
})
```

Transparent objects are routed to the OIT pass (see [TRANSPARENCY.md](./TRANSPARENCY.md))
instead of the opaque pass. The renderer checks `material.transparent` during
draw command generation.

Vertex alpha from the material index palette:

```typescript
palette: [
  { color: [1, 0, 0], opacity: 0.5 },  // semi-transparent red
]
```

## Shader Variant Compilation

Each unique combination of feature flags produces a different shader variant.
Mantis uses a **feature key** to cache compiled variants:

```typescript
// Feature key is a bitmask
const VERTEX_COLORS   = 1 << 0
const MATERIAL_INDEX  = 1 << 1
const COLOR_TEXTURE   = 1 << 2
const AO_TEXTURE      = 1 << 3
const SKINNING        = 1 << 4
const SHADOWS         = 1 << 5
const TRANSPARENT     = 1 << 6
const LAMBERT         = 1 << 7

const featureKey = (mat: Material, mesh: Mesh): number => {
  let key = 0
  if (mesh.geometry.hasAttribute('color')) key |= VERTEX_COLORS
  if (mesh.geometry.hasAttribute('_materialIndex')) key |= MATERIAL_INDEX
  if (mat.colorMap) key |= COLOR_TEXTURE
  if (mat instanceof LambertMaterial && mat.aoMap) key |= AO_TEXTURE
  if (mesh.skeleton) key |= SKINNING
  if (mat instanceof LambertMaterial && mat.shadows) key |= SHADOWS
  if (mat.transparent) key |= TRANSPARENT
  if (mat instanceof LambertMaterial) key |= LAMBERT
  return key
}
```

The compiled pipeline (shader + render state) is cached in a `Map<number, GALPipeline>`.
A typical application uses 10–20 unique variants.

### Pipeline State per Material

Each material compiles to an immutable pipeline state:

```typescript
interface PipelineState {
  shader: GALShaderModule       // compiled vertex + fragment
  depthWrite: boolean           // true for opaque, false for transparent
  depthCompare: 'less' | 'less-equal'
  cullMode: 'back' | 'front' | 'none'
  blendState: BlendState | null // null for opaque, additive for OIT
  colorTargetFormats: TextureFormat[]
}
```

This state is determined at material creation time and never changes. The
renderer does not inspect material properties per frame — it uses the
pre-compiled pipeline directly.

## Material Uniforms

### Per-Material Uniform Block (Bind Group 1, Binding 0)

```
layout (std140):
  vec4 baseColor;              // 16 bytes (rgb + alpha)
  vec4 materialColors[32];     // 512 bytes
  vec4 emissiveColors[32];     // 512 bytes
  uint flags;                  // 4 bytes (feature bitmask, used for debug)
  float padding[3];            // 12 bytes
  Total: 1056 bytes
```

This is uploaded once when the material is created or modified, then remains
static. The bind group for the material is also created once and reused.

### Texture Binding (Bind Group 1, Bindings 1–2)

```
binding 1: colorTexture + sampler    (or 1×1 white pixel if absent)
binding 2: aoTexture + sampler       (or 1×1 white pixel if absent)
```

Default 1×1 textures avoid branching in the shader — when no color/AO texture
is bound, the default white pixel multiplies to identity.

## Material API

```typescript
// Basic material (unlit)
const basic = new BasicMaterial({
  color: [1, 0.5, 0.2],          // orange
  colorMap: myTexture,            // optional
  opacity: 1.0,                   // default
  transparent: false,             // default
  side: 'front',                  // 'front' | 'back' | 'both'
  palette: [],                    // optional material index palette
})

// Lambert material (diffuse lit)
const lambert = new LambertMaterial({
  color: [1, 1, 1],
  colorMap: myTexture,
  aoMap: myAOTexture,
  shadows: true,
  palette: [
    { color: [1, 1, 1] },
    { color: [0, 0, 0] },
    { color: [0, 0.8, 0.7], emissive: [0, 0.8, 0.7], emissiveIntensity: 0.7 },
  ],
})

// Update palette at runtime (triggers uniform re-upload)
lambert.setPaletteEntry(2, {
  color: [1, 0, 0],
  emissive: [1, 0, 0],
  emissiveIntensity: 1.0,
})
```

## Why Only Basic + Lambert?

PBR (Physically Based Rendering) is overkill for the target use case — stylized,
low-poly games. Lambert provides convincing diffuse shading at a fraction of the
shader cost:

| | Lambert | PBR (Cook-Torrance) |
|---|---|---|
| Fragment shader ALU | ~15 ops | ~80+ ops |
| Textures needed | color, AO | albedo, metallic, roughness, normal, AO |
| Visual quality (stylized) | Excellent | Marginal improvement |
| Performance (mobile) | Fast | 3–5× slower per fragment |

If PBR is needed in the future, it can be added as a third material type
(`PBRMaterial`) without changing the existing architecture.
