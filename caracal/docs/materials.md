# Materials & Shader System

## Material Types

Caracal ships with two materials, covering the most common use cases for stylized/low-poly games:

### BasicMaterial (Unlit)

No lighting calculations. The final color is the material color, optionally modulated by a texture and/or vertex colors. Useful for:
- UI elements in 3D space
- Emissive objects (self-lit)
- Stylized effects (toon outlines, skyboxes)
- Debug visualization

```typescript
const material = createBasicMaterial({
  color: [1, 1, 1],               // Base color (RGB, 0-1)
  map: colorTexture,               // Optional: color texture
  aoMap: aoTexture,                // Optional: ambient occlusion map
  vertexColors: false,             // Use vertex color attribute
  materialIndexColors: undefined,  // Per-material-index color map
  opacity: 1.0,
  transparent: false,
  side: 'front',                   // 'front' | 'back' | 'both'
})
```

### LambertMaterial (Diffuse Lit)

Lambertian diffuse shading with optional textures, ambient occlusion, and shadow receiving. Covers most low-poly game art:

```typescript
const material = createLambertMaterial({
  color: [1, 1, 1],
  map: colorTexture,
  aoMap: aoTexture,
  aoMapIntensity: 1.0,
  vertexColors: false,
  materialIndexColors: undefined,
  emissive: [0, 0, 0],            // Emissive color
  emissiveIntensity: 0.0,         // Emissive strength
  opacity: 1.0,
  transparent: false,
  side: 'front',
  receiveShadow: true,
})
```

## Material Index System

This is Caracal's key differentiator for stylized game workflows. Instead of splitting a model into multiple meshes per material, the entire model is a **single mesh** with a `_materialindex` vertex attribute. Each vertex is tagged with an integer index, and the engine maps each index to a color + emissive configuration.

### How It Works

1. **In the 3D modeling tool** (Blender/etc.): Assign different material slots to faces. On export, these become the `_materialindex` attribute (an integer per vertex).

2. **In Caracal**: Define a `MaterialIndexMap` that maps each index to visual properties:

```typescript
const materialIndexColors = createMaterialIndexMap({
  0: { color: [1, 1, 1] },                              // White
  1: { color: [0, 0, 0] },                              // Black
  2: { color: [0, 0.8, 0.7], emissive: [0, 0.8, 0.7], emissiveIntensity: 0.7 },  // Teal, glowing
  3: { color: [1, 0.2, 0.1], emissive: [1, 0.2, 0.1], emissiveIntensity: 1.0 },  // Red, strong glow
})

const material = createLambertMaterial({
  materialIndexColors,
  // base color/emissive are overridden per-vertex by the index map
})
```

3. **In the shader**: The `_materialindex` attribute is used to look up color and emissive from uniform arrays.

### Shader Implementation

The material index lookup happens in both GLSL and WGSL shaders:

**GLSL 300 ES (WebGL 2):**
```glsl
// Vertex shader
in float a_materialIndex;
out float v_materialIndex;

void main() {
  v_materialIndex = a_materialIndex;
  // ... standard transform
}

// Fragment shader
uniform vec3 u_matIdxColors[MAX_MATERIAL_INDICES];   // default: 16
uniform vec3 u_matIdxEmissive[MAX_MATERIAL_INDICES];
uniform float u_matIdxEmissiveIntensity[MAX_MATERIAL_INDICES];

in float v_materialIndex;

void main() {
  int idx = int(v_materialIndex + 0.5); // round to nearest int
  vec3 baseColor = u_matIdxColors[idx];
  vec3 emissive = u_matIdxEmissive[idx] * u_matIdxEmissiveIntensity[idx];

  // ... lighting calculations using baseColor
  // ... add emissive to final output
  // emissive output drives the bloom pass
}
```

**WGSL (WebGPU):**
```wgsl
struct MaterialIndexData {
  colors: array<vec3f, 16>,
  emissive: array<vec3f, 16>,
  emissiveIntensity: array<f32, 16>,
}
@group(1) @binding(1) var<uniform> matIdx: MaterialIndexData;

@fragment
fn main(@location(3) materialIndex: f32) -> @location(0) vec4f {
  let idx = u32(materialIndex + 0.5);
  let baseColor = matIdx.colors[idx];
  let emissive = matIdx.emissive[idx] * matIdx.emissiveIntensity[idx];
  // ...
}
```

### Vertex Colors

When `vertexColors` is enabled, the `a_color` attribute (vec3 or vec4) is multiplied with the material's base color. This works **in addition to** material index colors:

```
finalBaseColor = materialColor * vertexColor * materialIndexColor
```

If only material index colors are used (no vertex colors), `vertexColor` defaults to white. This gives maximum flexibility — artists can combine global tinting, per-vertex painting, and per-material-slot colors.

## Shader Generation

Caracal uses a template-based shader generation system. Since there are only two material types with a limited set of feature flags, we avoid a full transpiler in favor of conditional string templates:

### Feature Flags

Each material compiles a shader variant based on its configuration. Feature flags determine which code paths are included:

```typescript
interface ShaderFlags {
  hasMap: boolean            // Color texture
  hasAoMap: boolean          // AO texture
  hasVertexColors: boolean   // Vertex color attribute
  hasMaterialIndex: boolean  // Material index system
  hasSkinning: boolean       // GPU skinning (4 bones per vertex)
  hasEmissive: boolean       // Emissive output (for bloom)
  receiveShadow: boolean     // Shadow map sampling
  isTransparent: boolean     // OIT output mode
}
```

### Shader Variants

With 2 material types × 8 flags, the theoretical maximum is 2 × 256 = 512 variants. In practice, only ~10-20 variants are used in a typical scene. Shaders are compiled lazily on first use and cached by their flag combination.

### Dual Output (GLSL + WGSL)

The shader generator produces both languages from the same template:

```typescript
const generateFragmentShader = (
  type: 'basic' | 'lambert',
  flags: ShaderFlags,
  target: 'glsl300' | 'wgsl'
): string => {
  const chunks: string[] = []

  if (target === 'glsl300') {
    chunks.push('#version 300 es')
    chunks.push('precision highp float;')
  }

  // Shared logic expressed as template chunks
  if (flags.hasMap) {
    chunks.push(target === 'glsl300'
      ? 'uniform sampler2D u_map; // ...'
      : '@group(1) @binding(0) var t_map: texture_2d<f32>; // ...'
    )
  }

  // ... assemble shader from chunks
  return chunks.join('\n')
}
```

Each chunk is a small, self-contained piece of shader code. The generator assembles them based on flags. This is simpler than a full IR but sufficient for two material types.

## Material Uniforms

Materials expose uniform values through a flat descriptor. The renderer uploads these efficiently:

### Per-Material Uniforms (bind group 1)

| Uniform | Type | Material |
|---------|------|----------|
| `u_color` | vec3 | Both |
| `u_opacity` | float | Both |
| `u_emissive` | vec3 | Lambert |
| `u_emissiveIntensity` | float | Lambert |
| `u_aoMapIntensity` | float | Lambert |
| `u_matIdxColors[16]` | vec3[] | Both (if materialIndex) |
| `u_matIdxEmissive[16]` | vec3[] | Both (if materialIndex) |
| `u_matIdxEmissiveIntensity[16]` | float[] | Both (if materialIndex) |

### Textures (bind group 1)

| Slot | Uniform | Purpose |
|------|---------|---------|
| 0 | `u_map` / `t_map` | Color/albedo texture |
| 1 | `u_aoMap` / `t_aoMap` | Ambient occlusion |
| 2 | `u_shadowMap` / `t_shadowMap` | CSM shadow atlas |

## Material State

Each material defines GPU state that affects the render pipeline:

```typescript
interface MaterialState {
  depthTest: boolean       // default: true
  depthWrite: boolean      // default: true for opaque, false for transparent
  blendMode: BlendMode     // 'none' | 'normal' | 'additive' | 'oit'
  cullFace: CullFace       // 'front' | 'back' | 'none'
  side: Side               // shorthand: 'front' → cullBack, 'back' → cullFront, 'both' → noCull
}
```

When `transparent: true`:
- `depthWrite` is set to `false`
- `blendMode` is set to `'oit'` (OIT accumulation/revealage blend modes)
- The mesh is routed to the transparent pass instead of the opaque pass

## Shader Compilation Strategy

Unlike Three.js (which compiles shaders on first render, causing frame spikes), Caracal compiles eagerly:

```typescript
// At material creation time:
const material = createLambertMaterial({ map: texture, receiveShadow: true })
// → Immediately compiles shader variant for (lambert, hasMap, receiveShadow)
// → If shader is already cached, returns instantly

// At render time: shader is guaranteed ready, zero compilation cost
```

For WebGPU, pipeline creation is also eager — done when the material is first paired with a geometry layout (which defines the vertex buffer format). Since geometry layouts are few, this happens quickly at scene setup.

## Emissive and Bloom Integration

The bloom pass needs to know which pixels should glow. Caracal writes emissive intensity to the alpha channel of the scene color output:

```glsl
// Fragment output for opaque pass
out vec4 fragColor;

void main() {
  vec3 litColor = /* diffuse + ambient + shadow */;
  vec3 emissive = u_emissive * u_emissiveIntensity;
  // (or from material index lookup)

  fragColor = vec4(litColor + emissive, emissiveIntensity);
  // Alpha channel = bloom strength (0 = no bloom, 1 = full bloom)
}
```

The bloom extraction pass reads this alpha channel to selectively extract pixels that should bloom. This avoids a separate render target for emissive and keeps the pass count low.
