# MATERIALS — Final Decision

## Core Material Types

**Decision: Two materials — BasicMaterial (unlit) and LambertMaterial (diffuse)** (universal agreement)

### BasicMaterial

Unlit, color only. For UI elements, debug visualization, skybox-type objects, unlit glTF materials (`KHR_materials_unlit`).

```typescript
const basic = createBasicMaterial({
  color: [1, 0, 0],
  colorTexture: texture,      // optional
  vertexColors: false,        // optional
  transparent: false,         // optional
  opacity: 1.0,               // optional
})
```

### LambertMaterial

Diffuse N·L shading. For all lit objects in the stylized/low-poly target aesthetic.

```typescript
const lambert = createLambertMaterial({
  color: [0.8, 0.8, 0.8],
  colorTexture: texture,          // optional
  aoTexture: aoTexture,           // optional
  vertexColors: false,            // optional
  receiveShadow: true,            // optional
  transparent: false,             // optional
  opacity: 1.0,                   // optional
  palette: [...],                 // optional, material index system
})
```

## Material Index Palette

**Decision: 32-entry material palette via UBO** (7/9 implementations feature this)

The material index system is the engine's distinguishing feature for stylized games. Per-vertex `_materialindex` attribute selects a palette entry, enabling complex visual effects (glowing eyes, colored armor parts, different surface properties) with a single draw call.

```typescript
const characterMaterial = createLambertMaterial({
  palette: [
    { color: [0.9, 0.7, 0.6] },                                              // 0: skin
    { color: [0.2, 0.3, 0.8] },                                              // 1: shirt
    { color: [0.1, 0.1, 0.1] },                                              // 2: hair
    { color: [1, 0.8, 0.2], emissive: [1, 0.8, 0.2], emissiveIntensity: 2.0 }, // 3: glow
  ],
  receiveShadow: true,
})
```

Each palette entry contains:
- `color: [r, g, b]`
- `opacity: number` (default 1.0)
- `emissive: [r, g, b]` (default [0,0,0])
- `emissiveIntensity: number` (default 0.0)

Stored as a combined struct array in a UBO (bind group 1). Maximum 32 entries per material.

## Vertex Color Combining

**Decision: Multiplicative blending** (universal agreement)

```glsl
vec3 surfaceColor = materialColor * vertexColor.rgb;
float alpha = materialOpacity * vertexColor.a;
```

Vertex colors multiply with the material's base color. This allows vertex colors to tint without overriding the material.

## Emissive Output

**Decision: MRT with separate emissive render target for bloom** (universal agreement)

The opaque pass uses Multiple Render Targets to output both the lit color and the emissive contribution:

```glsl
// Fragment shader outputs:
layout(location = 0) out vec4 fragColor;    // Lit scene color
layout(location = 1) out vec4 fragEmissive; // Emissive-only (bloom source)
```

Emissive values are written to a separate render target that feeds directly into the bloom downsample chain. No global threshold pass is needed — bloom is driven purely by emissive output.

## Shader Variant System

**Decision: Feature flag bitmask with lazy compilation and caching** (universal agreement)

Feature flags:

```
HAS_COLOR_TEXTURE, HAS_AO_TEXTURE, HAS_VERTEX_COLORS,
HAS_MATERIAL_INDEX, HAS_SKINNING, HAS_EMISSIVE,
SHADOW_RECEIVE, IS_TRANSPARENT
```

```typescript
const shaderCache = new Map<number, CompiledShader>()

const getShader = (featureMask: number): CompiledShader => {
  let shader = shaderCache.get(featureMask)
  if (!shader) {
    shader = compileShaderVariant(featureMask)
    shaderCache.set(featureMask, shader)
  }
  return shader
}
```

Typical scene: 10-30 unique shader variants.

## Default Textures

**Decision: 1×1 white pixel default textures** (universal agreement)

When no texture is provided, a 1×1 white pixel texture is bound. This avoids shader branching for textured vs untextured paths — the texture sample simply returns white, which multiplies with the base color to produce the same result.

## Transparency Routing

Transparent materials (opacity < 1.0 or `transparent: true`) are automatically routed to the WBOIT pass. The transparent flag is encoded in the 64-bit sort key so transparent objects are drawn after all opaque geometry.

## Pipeline State

Each material maps to an immutable pipeline state (shader + blend + depth + cull). The material ID is encoded in the sort key so draws with the same material are adjacent, minimizing state switches.

## Texture Handling

- Separate texture + sampler on WebGPU
- Combined texture+sampler on WebGL2
- Compressed format support: ASTC, BC7, ETC2 (detected at runtime)
- Fallback to RGBA8 when no compressed format is available
