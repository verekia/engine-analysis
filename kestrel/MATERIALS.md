# Materials

## Overview

Kestrel ships two materials: **BasicMaterial** (unlit) and **LambertMaterial** (diffuse lighting). Both support the material index system for per-vertex color and emissive control.

## Material Index System

The core innovation for art direction is the **material index** system. Models are exported as single meshes with a custom vertex attribute `_materialindex` (an integer per vertex, typically `uint8`). At runtime, a **MaterialIndexMap** defines what color, emissive color, and emissive intensity each index maps to.

### Concept

A low-poly character mesh has vertices painted with material indices:
- Index 0: skin (peach color, no emissive)
- Index 1: shirt (teal, no emissive)
- Index 2: eyes (white, emissive intensity 0.7 → they glow)
- Index 3: belt (black, no emissive)

```typescript
const material = new LambertMaterial({
  materialIndexMap: [
    { color: [1.0, 0.85, 0.72], emissive: [0, 0, 0], emissiveIntensity: 0 },       // 0: skin
    { color: [0.0, 0.5, 0.5],   emissive: [0, 0, 0], emissiveIntensity: 0 },       // 1: shirt
    { color: [1.0, 1.0, 1.0],   emissive: [1.0, 1.0, 1.0], emissiveIntensity: 0.7 }, // 2: eyes
    { color: [0.05, 0.05, 0.05], emissive: [0, 0, 0], emissiveIntensity: 0 },       // 3: belt
  ]
})
```

### GPU Implementation

The material index map is uploaded as a uniform buffer (or texture for large maps):

```wgsl
// Per-material uniform
struct MaterialEntry {
  color: vec4<f32>,
  emissive: vec4<f32>,  // rgb = emissive color, a = intensity
}

@group(1) @binding(0) var<uniform> materialMap: array<MaterialEntry, 32>;

// Vertex shader reads _materialindex attribute
@location(5) materialIndex: u32

// Fragment shader
let entry = materialMap[materialIndex];
let baseColor = entry.color.rgb;
let emissiveContribution = entry.emissive.rgb * entry.emissive.a;
```

The maximum material index count is 32 per material (fits comfortably in a UBO). If more are needed, a 1D texture lookup is used instead.

### Vertex Colors

Vertex colors (`vec4<f32>` per vertex) are multiplied with the material index color:

```wgsl
let finalColor = entry.color.rgb * vertexColor.rgb;
```

This allows baked lighting or color variation on top of the material index palette.

## BasicMaterial

Unlit material. Renders `color × texture × vertexColor`. No lighting calculations.

```typescript
interface BasicMaterialOptions {
  color?: [number, number, number]   // default [1, 1, 1]
  colorTexture?: Texture             // optional — multiplied with color
  vertexColors?: boolean             // default false
  materialIndexMap?: MaterialEntry[] // optional per-index color mapping
  opacity?: number                   // default 1.0
  transparent?: boolean              // default false — enables OIT path
  side?: 'front' | 'back' | 'both'  // default 'front'
}
```

Fragment shader (simplified):
```wgsl
var color = material.color;
if hasColorTexture:    color *= textureSample(colorTex, uv);
if hasMaterialIndex:   color *= materialMap[materialIndex].color;
if hasVertexColors:    color *= vertexColor;
output.color = vec4(color, material.opacity);
```

## LambertMaterial

Diffuse N·L (Lambertian) lighting with ambient, shadows, AO, and emissive.

```typescript
interface LambertMaterialOptions {
  color?: [number, number, number]       // default [1, 1, 1]
  colorTexture?: Texture                 // optional diffuse texture
  aoTexture?: Texture                    // optional ambient occlusion texture
  vertexColors?: boolean                 // default false
  materialIndexMap?: MaterialEntry[]     // optional per-index mapping
  opacity?: number                       // default 1.0
  transparent?: boolean                  // default false
  side?: 'front' | 'back' | 'both'      // default 'front'
  receiveShadows?: boolean               // default true
  castShadows?: boolean                  // default true (used by renderer)
}
```

Fragment shader (simplified):
```wgsl
// Base color
var baseColor = material.color;
if hasColorTexture:    baseColor *= textureSample(colorTex, uv);
if hasMaterialIndex:   baseColor *= materialMap[materialIndex].color;
if hasVertexColors:    baseColor *= vertexColor;

// Lighting
let ambient = ambientLight.color * ambientLight.intensity;
let NdotL = max(dot(normal, -directionalLight.direction), 0.0);
let diffuse = directionalLight.color * directionalLight.intensity * NdotL;

// Shadows
var shadow = 1.0;
if receiveShadows:
  shadow = sampleCascadedShadowMap(worldPosition, NdotL);

// AO
var ao = 1.0;
if hasAOTexture:
  ao = textureSample(aoTex, uv).r;

// Final color
let litColor = baseColor * (ambient * ao + diffuse * shadow);

// Emissive (written to emissive attachment for bloom)
var emissiveOut = vec3(0.0);
if hasMaterialIndex:
  let entry = materialMap[materialIndex];
  emissiveOut = entry.emissive.rgb * entry.emissive.a;

output.color = vec4(litColor, material.opacity);
output.emissive = vec4(emissiveOut, 1.0);  // MRT attachment 1
```

## Pipeline Compilation and Caching

Materials don't hold GPU state directly. When a `Mesh(geometry, material)` is first rendered, the renderer compiles a `GpuPipeline` for the combination of:

1. **Material type** (Basic / Lambert)
2. **Shader features** derived from the material and geometry:
   - `hasColorTexture`, `hasAOTexture`, `hasVertexColors`, `hasMaterialIndex`, `hasSkinning`, `hasShadows`, `hasEmissive`
3. **Render state** from material:
   - `transparent` (selects OIT pipeline variant)
   - `side` (cull mode)

The pipeline is cached by a string key. Subsequent meshes with the same feature set reuse the pipeline.

```
Pipeline cache key example:
"lambert|colorTex:1|aoTex:1|vColors:0|matIdx:1|skin:0|shadow:1|emissive:1|oitF|cullBack"
```

## Bind Group Layout

Materials contribute to bind group 1 (group 0 is per-frame, group 2 is per-object):

```
Group 0: Frame uniforms (view, projection, lights, shadows)
Group 1: Material uniforms + textures
  binding 0: MaterialUniforms (color, opacity, flags)
  binding 1: MaterialIndexMap (array of entries)
  binding 2: Color texture + sampler
  binding 3: AO texture + sampler
Group 2: Object uniforms (world matrix)
```

On WebGL2, this maps to UBO binding points and texture units.

## Transparency

When `transparent: true`, the mesh is routed to the OIT render pass instead of the opaque pass. The fragment shader writes to accumulation and revealage buffers instead of the main color buffer. See [TRANSPARENCY.md](./TRANSPARENCY.md) for details.

The material API is simple — just set `transparent: true` and `opacity` to your desired value. No manual sorting, no render order headaches.

## Material Disposal

```typescript
material.dispose():
  // Does NOT destroy cached pipelines (other meshes may use them)
  // Releases material-specific GPU resources (uniform buffer, bind group)
  device.destroyBuffer(material._uniformBuffer)
  device.destroyBindGroup(material._bindGroup)
```

Textures are reference-counted and disposed when no material references them.
