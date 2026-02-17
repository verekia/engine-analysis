# Materials & Shaders

## Material Types

Rabbit ships two material types, both supporting the full feature matrix of vertex colors, material indices, textures, and emissive.

### BasicMaterial (Unlit)

No lighting calculations. Color comes from:
1. Base color (uniform `vec3`)
2. Color map (texture, optional)
3. Vertex colors (optional, multiplicative)
4. Material index colors (optional, overrides base color per-vertex)

```typescript
interface BasicMaterialDescriptor {
  color?: [number, number, number]          // default: [1, 1, 1]
  colorMap?: HalTexture                     // optional albedo texture
  vertexColors?: boolean                    // use vertex color attribute
  transparent?: boolean                     // enable OIT rendering
  opacity?: number                          // 0..1, default 1
  materialIndexEntries?: MaterialIndexEntry[] // per-index overrides
  side?: 'front' | 'back' | 'both'         // face culling
}
```

### LambertMaterial (Diffuse)

Lambertian diffuse lighting (N·L) plus ambient. Supports everything BasicMaterial does, plus AO maps, emissive, and shadow receiving.

```typescript
interface LambertMaterialDescriptor {
  color?: [number, number, number]
  colorMap?: HalTexture
  aoMap?: HalTexture                        // ambient occlusion
  vertexColors?: boolean
  transparent?: boolean
  opacity?: number
  emissive?: [number, number, number]       // emissive color
  emissiveIntensity?: number                // multiplier, default 0
  materialIndexEntries?: MaterialIndexEntry[]
  side?: 'front' | 'back' | 'both'
  receiveShadows?: boolean                  // default true
}
```

## Material Index System

### The Problem

Game models are often exported as single meshes with multiple logical materials (e.g., a character with skin, armor, and glowing eyes). Three.js handles this with material arrays and group indices on BufferGeometry, which creates separate draw calls per group.

Rabbit takes a different approach: the mesh stays as a single draw call, and a per-vertex `_materialindex` attribute selects material properties from a uniform array.

### Vertex Attribute

The glTF exporter writes a custom vertex attribute `_MATERIALINDEX` (stored as a `float` for WebGL2 compatibility). Each vertex has a material index value (0, 1, 2, ...).

### Material Index Entries

```typescript
interface MaterialIndexEntry {
  color: [number, number, number]           // RGB color for this index
  emissive?: [number, number, number]       // emissive color (default [0,0,0])
  emissiveIntensity?: number                // emissive multiplier (default 0)
}
```

Example usage:

```typescript
const material = new LambertMaterial({
  materialIndexEntries: [
    { color: [1, 1, 1] },                                   // index 0: white
    { color: [0.05, 0.05, 0.05] },                          // index 1: near-black
    { color: [0, 0.8, 0.7], emissive: [0, 0.8, 0.7], emissiveIntensity: 0.7 },  // index 2: teal, glowing
  ],
})
```

### Shader Implementation

In the shader, material index entries are passed as a uniform buffer:

```glsl
// GLSL 300 es
struct MaterialEntry {
  vec3 color;
  float emissiveIntensity;
  vec3 emissive;
  float _pad;
};

layout(std140) uniform MaterialIndices {
  MaterialEntry u_materialEntries[MAX_MATERIAL_ENTRIES]; // MAX = 32
};

#ifdef HAS_MATERIAL_INDEX
  flat in int v_materialIndex;
#endif

void main() {
  vec3 baseColor = u_color;

  #ifdef HAS_MATERIAL_INDEX
    MaterialEntry entry = u_materialEntries[v_materialIndex];
    baseColor = entry.color;
  #endif

  #ifdef HAS_VERTEX_COLORS
    baseColor *= v_color.rgb;
  #endif

  #ifdef HAS_COLOR_MAP
    baseColor *= texture(u_colorMap, v_uv).rgb;
  #endif

  // Lambert lighting
  #ifdef IS_LAMBERT
    float NdotL = max(dot(v_normal, u_lightDir), 0.0);
    vec3 diffuse = baseColor * (u_ambientColor + u_lightColor * NdotL);
  #else
    vec3 diffuse = baseColor;
  #endif

  // AO
  #ifdef HAS_AO_MAP
    float ao = texture(u_aoMap, v_uv).r;
    diffuse *= ao;
  #endif

  // Emissive
  vec3 emissiveContrib = vec3(0.0);
  #ifdef HAS_EMISSIVE
    #ifdef HAS_MATERIAL_INDEX
      emissiveContrib = entry.emissive * entry.emissiveIntensity;
    #else
      emissiveContrib = u_emissive * u_emissiveIntensity;
    #endif
  #endif

  vec3 finalColor = diffuse + emissiveContrib;

  // Shadow
  #ifdef HAS_SHADOWS
    float shadow = calcShadow(v_shadowCoord, v_depth);
    // Only shadow the diffuse lighting, not emissive
    finalColor = (diffuse * shadow) + emissiveContrib;
  #endif

  // Output
  #ifdef IS_TRANSPARENT
    // OIT output — see TRANSPARENCY-BLOOM.md
    outputOIT(vec4(finalColor, u_opacity));
  #else
    fragColor = vec4(finalColor, 1.0);
    // Write emissive to a separate target for bloom
    #ifdef HAS_BLOOM_TARGET
      bloomColor = vec4(emissiveContrib, 1.0);
    #endif
  #endif
}
```

### WGSL Equivalent

The WebGPU shaders use WGSL with equivalent logic. Shader variants are handled via a preprocessor step that strips unused code blocks before compilation:

```wgsl
struct MaterialEntry {
  color: vec3f,
  emissive_intensity: f32,
  emissive: vec3f,
  _pad: f32,
}

@group(1) @binding(0)
var<uniform> material_entries: array<MaterialEntry, 32>;

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
  var base_color = uniforms.color;

  // #if HAS_MATERIAL_INDEX
  let entry = material_entries[in.material_index];
  base_color = entry.color;
  // #endif

  // ... rest of shading
}
```

## Shader Variant System

Materials generate shader variants based on their configuration. Each unique combination of features produces a distinct shader/pipeline pair, which is cached.

### Feature Flags

```typescript
interface ShaderFeatures {
  hasVertexColors: boolean
  hasMaterialIndex: boolean
  hasColorMap: boolean
  hasAOMap: boolean
  hasSkinning: boolean
  hasEmissive: boolean
  hasShadows: boolean
  isLambert: boolean
  isTransparent: boolean
  hasBloomTarget: boolean
}
```

### Variant Key

A bitmask encodes the feature set:

```typescript
const computeVariantKey = (features: ShaderFeatures): number => {
  let key = 0
  if (features.hasVertexColors)  key |= 1 << 0
  if (features.hasMaterialIndex) key |= 1 << 1
  if (features.hasColorMap)      key |= 1 << 2
  if (features.hasAOMap)         key |= 1 << 3
  if (features.hasSkinning)      key |= 1 << 4
  if (features.hasEmissive)      key |= 1 << 5
  if (features.hasShadows)       key |= 1 << 6
  if (features.isLambert)        key |= 1 << 7
  if (features.isTransparent)    key |= 1 << 8
  if (features.hasBloomTarget)   key |= 1 << 9
  return key
}
```

The `ShaderCache` stores compiled shaders keyed by this variant bitmask. A typical game scene might use 10-30 unique variants.

## Pipeline ID and Draw Call Sorting

Each material instance has a `pipelineId` that encodes:
- Shader variant key (which shader program)
- Blend state (opaque vs transparent)
- Depth state
- Cull mode

Draw calls sorted by `pipelineId` ensure minimal pipeline switches. Within the same pipeline, objects are sorted front-to-back for early-Z rejection.

## Bind Groups

Resources are organized into bind groups by update frequency:

| Group | Contents | Update frequency |
|-------|----------|-----------------|
| 0 | Camera matrices, lights, shadow maps | Once per frame |
| 1 | Material uniforms, material index entries | Once per material |
| 2 | Textures (color map, AO map) | Once per material |
| 3 | Object uniforms (model matrix, bone matrices) | Once per draw call |

This layout minimizes bind group switches:
- Group 0 is set once per pass
- Groups 1 and 2 change only when material changes (sorted to minimize)
- Group 3 changes every draw call but is the cheapest to update (dynamic offset into uniform buffer)

In WebGL2, these map to UBO binding points (0-3) and texture units.
