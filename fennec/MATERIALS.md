# Materials — Shaders, Material Index System, Vertex Colors

## Overview

Fennec ships two materials: **Basic** (unlit) and **Lambert** (diffuse lighting). Both support the **material index system** — a per-vertex attribute `_materialindex` that maps vertices to different color/emissive properties within a single mesh. This eliminates the need to split meshes by material, reducing draw calls.

## Material Types

### BasicMaterial (Unlit)

No lighting calculations. Color comes from a solid color, vertex colors, a texture, or the material index system.

```typescript
interface BasicMaterialOptions {
  color?: number                   // Hex color (0xRRGGBB), default 0xffffff
  opacity?: number                 // 0..1, default 1
  transparent?: boolean            // Enable OIT transparency
  map?: Texture                    // Color texture
  vertexColors?: boolean           // Use vertex color attribute
  side?: 'front' | 'back' | 'double'  // Face culling, default 'front'
}
```

### LambertMaterial (Diffuse)

Ambient + Lambertian diffuse lighting from directional and ambient lights. No specular.

```typescript
interface LambertMaterialOptions extends BasicMaterialOptions {
  aoMap?: Texture                  // Ambient occlusion texture
  aoMapIntensity?: number          // AO strength, default 1
  emissive?: number                // Emissive color (hex), default 0x000000
  emissiveIntensity?: number       // Emissive strength, default 1
  emissiveMap?: Texture            // Emissive texture
  receiveShadow?: boolean          // Sample shadow map, default true
}
```

### Lambert Lighting Model

```glsl
// Vertex or fragment shader (vertex lighting for performance, fragment for quality)
vec3 computeLambert(vec3 normal, vec3 lightDir, vec3 lightColor, float lightIntensity) {
  float NdotL = max(dot(normal, lightDir), 0.0);
  return lightColor * lightIntensity * NdotL;
}

vec3 finalColor = ambientColor * ambientIntensity * baseColor
                + computeLambert(worldNormal, lightDir, lightColor, lightIntensity) * baseColor
                + emissiveColor * emissiveIntensity;

// Apply AO
finalColor *= mix(1.0, aoSample, aoMapIntensity);

// Apply shadow
finalColor *= mix(shadowColor, vec3(1.0), shadowFactor);
```

## Material Index System

### Concept

Many game assets are authored as single meshes where different "materials" are encoded as a vertex attribute rather than separate submeshes. Each vertex has a `_materialindex` integer that maps to a slot in a material palette.

This approach means:
- **1 draw call per mesh** regardless of how many "materials" it has
- No mesh splitting, no extra buffers
- Colors and emissive properties are driven per-vertex in the shader

### Vertex Attribute

GLTF models carry a custom attribute `_MATERIALINDEX` (accessor type `SCALAR`, component type `UNSIGNED_BYTE` or `UNSIGNED_SHORT`). During import, this maps to vertex attribute location:

```
layout(location = 5) in float a_materialIndex;
```

### Material Palette

A material palette is a small uniform array (or texture for >16 materials) that maps each index to properties:

```typescript
interface MaterialPaletteEntry {
  color: [number, number, number]        // RGB 0..1
  emissive: [number, number, number]     // RGB 0..1
  emissiveIntensity: number              // 0..N
  opacity: number                        // 0..1
}

// Stored as a Float32Array for UBO upload
// Each entry = 8 floats (color.rgb + padding, emissive.rgb + emissiveIntensity)
// Max 32 entries = 256 floats = 1KB UBO
```

### Shader Implementation

```glsl
struct MaterialEntry {
  vec4 colorAndPad;       // rgb = color, a = opacity
  vec4 emissiveAndIntensity; // rgb = emissive color, a = emissive intensity
};

layout(std140) uniform MaterialPalette {
  MaterialEntry entries[32];
} palette;

void main() {
  int idx = int(a_materialIndex);
  vec3 matColor = palette.entries[idx].colorAndPad.rgb;
  float matOpacity = palette.entries[idx].colorAndPad.a;
  vec3 matEmissive = palette.entries[idx].emissiveAndIntensity.rgb;
  float matEmissiveIntensity = palette.entries[idx].emissiveAndIntensity.a;

  vec3 baseColor = matColor;

  #ifdef HAS_VERTEX_COLORS
  baseColor *= v_color.rgb;  // Modulate with vertex colors
  #endif

  #ifdef HAS_COLOR_MAP
  baseColor *= texture(colorMap, v_uv).rgb;
  #endif

  // Lighting (Lambert)
  vec3 lit = computeLighting(baseColor, v_normal, ...);

  // Add emissive
  vec3 finalColor = lit + matEmissive * matEmissiveIntensity;

  // Output for bloom: emissive intensity > threshold goes to bloom buffer
  float bloomStrength = max(0.0, matEmissiveIntensity - bloomThreshold);

  fragColor = vec4(finalColor, matOpacity);
  #ifdef HAS_BLOOM
  fragBloom = vec4(matEmissive * bloomStrength, 1.0);
  #endif
}
```

### API for Material Index Configuration

```typescript
// On a loaded GLTF mesh that has _materialindex attribute
const world = await loadGLTF('/world.glb')

// Set per-index properties
world.setMaterialIndex(0, { color: 0xffffff })                          // White
world.setMaterialIndex(1, { color: 0x000000 })                          // Black
world.setMaterialIndex(2, {
  color: 0x00ccaa,
  emissive: 0x00ccaa,
  emissiveIntensity: 0.7    // This will bloom
})

// Internally updates the MaterialPalette UBO
```

## Vertex Colors

Vertex colors (`COLOR_0` in GLTF) are supported as a multiplicative factor on the base color. When a mesh has both vertex colors and the material index system, vertex colors modulate the palette color:

```
finalBaseColor = materialPaletteColor * vertexColor * textureColor
```

This allows a single mesh to have per-vertex tinting on top of the material index coloring.

## Shader Variant System

Fennec generates shader variants at material creation time based on feature flags. This avoids branching in shaders:

```typescript
type ShaderDefines = {
  HAS_COLOR_MAP: boolean
  HAS_AO_MAP: boolean
  HAS_VERTEX_COLORS: boolean
  HAS_MATERIAL_INDEX: boolean
  HAS_SKINNING: boolean
  HAS_BLOOM: boolean
  HAS_SHADOW: boolean
  NUM_BONES: number
  MATERIAL_PALETTE_SIZE: number
}
```

### Variant Caching

Shader variants are cached by a hash of their defines. Two meshes with identical feature sets share the same compiled shader:

```typescript
const shaderCache = new Map<number, GPUShaderHandle>()

const getShader = (backend: GPUBackend, type: 'basic' | 'lambert', defines: ShaderDefines): GPUShaderHandle => {
  const key = hashDefines(type, defines)
  let shader = shaderCache.get(key)
  if (!shader) {
    const source = generateShaderSource(type, defines)
    shader = backend.createShader({ source, defines })
    shaderCache.set(key, shader)
  }
  return shader
}
```

### Pipeline State

Each unique combination of (shader, blend mode, depth state, cull mode, vertex layout) produces a pipeline object. Pipelines are also cached and shared:

```typescript
interface PipelineDescriptor {
  shader: GPUShaderHandle
  vertexLayout: VertexLayout
  blendMode: 'opaque' | 'oit_accumulation' | 'oit_composite'
  depthWrite: boolean
  depthTest: boolean
  cullMode: 'front' | 'back' | 'none'
  sampleCount: number
}
```

## Texture Handling

### Texture Object

```typescript
interface Texture {
  readonly id: number
  readonly width: number
  readonly height: number
  readonly format: TextureFormat      // 'rgba8', 'bc7', 'etc2', 'astc4x4', etc.
  readonly gpuTexture: GPUTextureHandle
  minFilter: 'nearest' | 'linear'
  magFilter: 'nearest' | 'linear'
  wrapS: 'repeat' | 'clamp' | 'mirror'
  wrapT: 'repeat' | 'clamp' | 'mirror'
  generateMipmaps: boolean
}
```

### Supported Formats

| Format | Source | Notes |
|--------|--------|-------|
| RGBA8 | PNG, JPG | Standard color/AO textures |
| BC7 | KTX2 (Basis) | Desktop compressed, best quality |
| ETC2 | KTX2 (Basis) | Mobile compressed (Android/iOS) |
| ASTC 4x4 | KTX2 (Basis) | Mobile compressed (iOS preferred) |

Basis Universal textures are transcoded at load time to the best format supported by the current GPU. See [ASSETS.md](./ASSETS.md) for the transcoding pipeline.

## Sort Key Construction

Materials contribute to the draw call sort key. The sort key is a single number that encodes render priority:

```typescript
// 48-bit sort key (safe for JS number comparison)
// Bits 40-47: render order (0=opaque, 1=transparent)
// Bits 24-39: shader ID (16 bits, max 65536 shaders)
// Bits 12-23: material ID (12 bits, max 4096 materials)
// Bits  0-11: geometry ID (12 bits, max 4096 geometries)
const buildSortKey = (renderOrder: number, shaderID: number, materialID: number, geometryID: number): number =>
  (renderOrder << 40) | (shaderID << 24) | (materialID << 12) | geometryID
```

Opaque objects are sorted front-to-back (ascending camera distance) within the same sort key to benefit from early-z rejection. Transparent objects skip sorting entirely (OIT handles order independence).
