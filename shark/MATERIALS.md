# Materials Architecture

## Material Types

Shark ships with two material types, designed for low-poly game aesthetics:

### BasicMaterial (Unlit)

No lighting calculations. The final color is simply the material color multiplied by any textures and vertex colors.

```typescript
interface BasicMaterial {
  type: 'basic'
  color: Color                    // Base color (default: white)
  opacity: number                 // 0..1 (default: 1)
  map: GpuTexture | null          // Color texture
  aoMap: GpuTexture | null        // Ambient occlusion texture
  aoMapIntensity: number          // AO strength (default: 1)
  vertexColors: boolean           // Use vertex color attribute
  transparent: boolean            // Route to OIT pass
  side: 'front' | 'back' | 'double'
  depthWrite: boolean
  depthTest: boolean
  blendMode: 'normal' | 'additive'
}
```

Fragment output:
```
finalColor = materialColor * textureColor * vertexColor * aoColor
```

### LambertMaterial (Diffuse)

Simple N·L diffuse shading plus ambient. No specular, no PBR.

```typescript
interface LambertMaterial {
  type: 'lambert'
  color: Color                    // Diffuse color
  emissive: Color                 // Emissive color (added after lighting)
  emissiveIntensity: number       // Emissive multiplier
  opacity: number
  map: GpuTexture | null          // Color/diffuse texture
  aoMap: GpuTexture | null        // AO texture
  aoMapIntensity: number
  vertexColors: boolean
  transparent: boolean
  receiveShadows: boolean
  side: 'front' | 'back' | 'double'
  depthWrite: boolean
  depthTest: boolean

  // Material index table (for multi-material single-mesh models)
  materialColors: Color[]         // Color per material index (max 32)
  materialEmissive: Color[]       // Emissive per material index (max 32)
  materialEmissiveIntensity: number[] // Emissive intensity per index
}
```

Fragment output:
```
diffuse = max(dot(normal, lightDir), 0.0) * lightColor * lightIntensity
ambient = ambientColor * ambientIntensity
ao = texture(aoMap, uv).r * aoMapIntensity
shadow = sampleCSM(worldPos, depth)

lighting = (diffuse * shadow + ambient) * ao
finalColor = baseColor * lighting + emissiveColor * emissiveIntensity
```

## Material Index System

### Problem

Game models are often exported as single meshes with multiple "sub-materials" baked in. Instead of splitting into separate draw calls per material zone, Shark uses a **per-vertex material index** to look up colors and emissive properties.

### Vertex Attribute

Models include a custom vertex attribute `a_materialIndex` (uint8):

```
Vertex 0: position, normal, uv, materialIndex=0
Vertex 1: position, normal, uv, materialIndex=0
Vertex 2: position, normal, uv, materialIndex=2
...
```

This attribute is stored in the GLTF as `_materialindex` (custom attribute).

### Uniform Table

The material stores arrays of colors and emissive values:

```typescript
const material = new LambertMaterial({
  // Material index 0 = white body
  // Material index 1 = black trim
  // Material index 2 = teal accents with glow
  materialColors: [
    new Color(1, 1, 1),      // 0: white
    new Color(0, 0, 0),      // 1: black
    new Color(0, 0.5, 0.5),  // 2: teal
  ],
  materialEmissive: [
    new Color(0, 0, 0),      // 0: no emission
    new Color(0, 0, 0),      // 1: no emission
    new Color(0, 0.5, 0.5),  // 2: teal glow
  ],
  materialEmissiveIntensity: [
    0,                        // 0: off
    0,                        // 1: off
    0.7,                      // 2: 70% bloom intensity
  ],
})
```

### Shader Implementation

```glsl
// Uniform buffer (per-material bind group)
uniform MaterialTable {
  vec4 u_materialColors[32];     // rgb = color, a = unused
  vec4 u_materialEmissive[32];   // rgb = emissive color, a = intensity
};

// Vertex attribute
in uint a_materialIndex;

// In fragment shader:
vec3 baseColor = u_materialColors[materialIndex].rgb;
vec3 emissiveColor = u_materialEmissive[materialIndex].rgb;
float emissiveIntensity = u_materialEmissive[materialIndex].a;

// Combine with vertex color if enabled
if (HAS_VERTEX_COLORS) {
  baseColor *= v_color.rgb;
}

// Final emissive contribution (written to emissive RT for bloom)
vec3 emissive = emissiveColor * emissiveIntensity;
```

### Vertex Color Interaction

When both vertex colors and material indices are present:
1. Material index determines the base color from the table
2. Vertex color is **multiplied** with the table color
3. This allows vertex color to add variation (e.g., gradients, dirt) on top of material-defined colors

When only vertex colors are used (no material index):
- `baseColor = vertexColor * materialColor`

## Emissive and Bloom Pipeline

### Per-Vertex Bloom

Bloom is driven by the emissive output. The render pass uses MRT (Multiple Render Targets):

```
RT0: Main color output (RGBA8 or RGBA16F)
RT1: Emissive output (RGBA16F) — bright fragments for bloom
```

The fragment shader writes emissive to RT1:
```glsl
layout(location = 0) out vec4 outColor;
layout(location = 1) out vec4 outEmissive;

void main() {
  // ... lighting calculations ...
  outColor = vec4(finalColor, opacity);
  outEmissive = vec4(emissive, 1.0);  // Only bright fragments
}
```

The bloom post-processing pass reads RT1 and applies the blur chain. See [POST_PROCESSING.md](./POST_PROCESSING.md) for details.

### Emissive Intensity Controls

- `emissiveIntensity = 0`: No glow at all
- `emissiveIntensity = 0.1..0.5`: Subtle glow
- `emissiveIntensity = 0.5..1.0`: Noticeable bloom
- `emissiveIntensity > 1.0`: Intense bloom (HDR blowout)

## Shader Variant Selection

Materials compile to shader variants based on their configuration. The variant is determined at material creation time and cached:

```typescript
const getShaderDefines = (material: Material): ShaderDefines => ({
  HAS_VERTEX_COLORS: material.vertexColors,
  HAS_MATERIAL_INDEX: material.materialColors.length > 0,
  HAS_TEXTURE: material.map !== null,
  HAS_AO_TEXTURE: material.aoMap !== null,
  HAS_SKINNING: false,  // Set by SkinnedMesh at draw time
  RECEIVE_SHADOWS: material.type === 'lambert' && material.receiveShadows,
  NUM_CSM_CASCADES: 3,
})
```

The defines are hashed to produce a pipeline cache key. Identical define sets reuse the same compiled pipeline.

## Material Bind Group Layout

```
Binding 0: MaterialUniforms (uniform buffer)
  - color (vec4)
  - emissive (vec4) — for non-indexed emissive
  - opacity (float)
  - aoMapIntensity (float)
  - materialColors[32] (vec4 array)
  - materialEmissive[32] (vec4 array — a channel = intensity)

Binding 1: colorTexture (sampled texture)
Binding 2: colorSampler (sampler)
Binding 3: aoTexture (sampled texture)
Binding 4: aoSampler (sampler)
Binding 5: shadowMap (sampled texture) — depth texture array
Binding 6: shadowSampler (comparison sampler)
```

For materials without textures, a 1×1 white pixel default texture is bound. This avoids branching in shaders and allows a single pipeline to handle textured and untextured objects.

## Transparency Handling

Materials with `transparent: true` are routed to the OIT pass instead of the opaque pass:

| Property | Opaque Pass | OIT Pass |
|---|---|---|
| Depth write | Yes | No |
| Depth test | Yes | Yes (read-only) |
| Blend mode | None (overwrite) | Custom (accumulation) |
| Sort order | By pipeline state | Unordered (OIT handles it) |
| Render target | Main RT | Accumulation + Revealage MRT |

See [POST_PROCESSING.md](./POST_PROCESSING.md) for the OIT compositing details.
