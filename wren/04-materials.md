# 04 — Materials, Vertex Colors & Material Index System

Wren supports two material types (Basic and Lambert) with a powerful vertex-color and material-index system that enables per-mesh-region coloring and emissive bloom without requiring separate draw calls.

## Material Types

### Basic Material (Unlit)

Outputs color directly, no lighting calculations. Used for UI elements, debug visualization, skyboxes, and emissive-only objects.

```ts
interface BasicMaterial {
  type: 'basic'
  id: number
  color: Float32Array             // [r, g, b] linear
  opacity: number                 // 0-1
  map: TextureHandle | null       // Color/albedo texture
  aoMap: TextureHandle | null     // Ambient occlusion
  aoMapIntensity: number          // 0-1
  vertexColors: boolean           // Use vertex color attribute
  transparent: boolean            // Enable OIT pass
  side: 'front' | 'back' | 'double'
  depthWrite: boolean
  depthTest: boolean

  // Material index system
  useMaterialIndex: boolean       // Enable _materialindex attribute lookup
  materialIndexColors: Float32Array | null   // Flat array: [r,g,b, r,g,b, ...] per index
  materialIndexEmissive: Float32Array | null // Flat array: [r,g,b,intensity, ...] per index
}
```

### Lambert Material (Diffuse Lit)

Standard diffuse lighting with Lambertian reflectance. Receives shadows and ambient light.

```ts
interface LambertMaterial {
  type: 'lambert'
  id: number
  color: Float32Array             // [r, g, b] base diffuse color
  opacity: number
  map: TextureHandle | null       // Albedo texture
  aoMap: TextureHandle | null     // Ambient occlusion
  aoMapIntensity: number
  emissive: Float32Array          // [r, g, b] emissive color
  emissiveIntensity: number       // Emissive multiplier
  vertexColors: boolean
  transparent: boolean
  side: 'front' | 'back' | 'double'
  depthWrite: boolean
  depthTest: boolean
  receiveShadow: boolean

  // Material index system
  useMaterialIndex: boolean
  materialIndexColors: Float32Array | null
  materialIndexEmissive: Float32Array | null
}
```

## Pipeline ID Generation

Each unique combination of material type + feature flags produces a pipeline variant. The pipeline ID is derived from:

```ts
const getPipelineKey = (material: Material, mesh: MeshNode): number => {
  let key = 0
  key |= (material.type === 'lambert' ? 1 : 0) << 0
  key |= (material.vertexColors ? 1 : 0) << 1
  key |= (material.map ? 1 : 0) << 2
  key |= (material.aoMap ? 1 : 0) << 3
  key |= (material.useMaterialIndex ? 1 : 0) << 4
  key |= (mesh.type === 'skinned_mesh' ? 1 : 0) << 5
  key |= (material.transparent ? 1 : 0) << 6
  key |= (material.type === 'lambert' && material.receiveShadow ? 1 : 0) << 7
  key |= (hasEmissive(material) ? 1 : 0) << 8
  key |= (sideToInt(material.side)) << 9  // 2 bits
  return key
}
```

This produces at most ~2048 unique pipeline variants. In practice, a typical game uses 10-30. Pipelines are compiled lazily and cached.

## Material Index System

This is the key feature for painting single-mesh models with per-region colors and emissive properties. Models are exported as a single mesh with a custom `_materialindex` vertex attribute (uint8 or uint16) that assigns each vertex to a material slot.

### How It Works

1. **Vertex attribute**: Each vertex has a `_materialindex` value (0, 1, 2, ...) baked into the geometry.
2. **Color table**: The material defines a `materialIndexColors` array that maps each index to an RGB color.
3. **Emissive table**: The material defines a `materialIndexEmissive` array that maps each index to an RGB color + intensity.
4. **Shader lookup**: The vertex/fragment shader reads the material index, looks up the color and emissive values, and applies them.

### Usage Example

```ts
const material = createLambertMaterial({
  useMaterialIndex: true,
  materialIndexColors: new Float32Array([
    1.0, 1.0, 1.0,    // Index 0: white
    0.0, 0.0, 0.0,    // Index 1: black
    0.0, 0.7, 0.7,    // Index 2: teal
  ]),
  materialIndexEmissive: new Float32Array([
    0.0, 0.0, 0.0, 0.0,   // Index 0: no emission
    0.0, 0.0, 0.0, 0.0,   // Index 1: no emission
    0.0, 0.7, 0.7, 0.7,   // Index 2: teal emission, intensity 0.7
  ]),
})
```

### Shader Implementation

The material index lookup happens in the vertex shader (for vertex colors) and fragment shader (for emissive contribution to bloom):

```glsl
// Vertex shader (GLSL 300 es)
#ifdef HAS_MATERIAL_INDEX
  in uint a_materialIndex;
  uniform vec3 u_materialColors[MAX_MATERIAL_INDICES]; // 16 max
  uniform vec4 u_materialEmissive[MAX_MATERIAL_INDICES]; // rgb + intensity

  out vec3 v_materialColor;
  out vec4 v_materialEmissive;
#endif

void main() {
  // ...
  #ifdef HAS_MATERIAL_INDEX
    v_materialColor = u_materialColors[a_materialIndex];
    v_materialEmissive = u_materialEmissive[a_materialIndex];
  #endif
}
```

```glsl
// Fragment shader
#ifdef HAS_MATERIAL_INDEX
  in vec3 v_materialColor;
  in vec4 v_materialEmissive;
#endif

// Dual output for bloom
layout(location = 0) out vec4 fragColor;
layout(location = 1) out vec4 fragEmissive;  // Written only during bloom-enabled pass

void main() {
  vec3 baseColor = u_color;

  #ifdef HAS_MATERIAL_INDEX
    baseColor *= v_materialColor;
  #endif

  #ifdef HAS_VERTEX_COLORS
    baseColor *= v_color.rgb;
  #endif

  // Lighting (Lambert)
  vec3 litColor = baseColor * (ambient + diffuse * shadow);

  fragColor = vec4(litColor, opacity);

  // Emissive output for bloom
  #ifdef HAS_EMISSIVE
    vec3 emissive = u_emissive * u_emissiveIntensity;
    #ifdef HAS_MATERIAL_INDEX
      emissive += v_materialEmissive.rgb * v_materialEmissive.a;
    #endif
    fragColor.rgb += emissive;
    fragEmissive = vec4(emissive, 1.0);
  #endif
}
```

### Vertex Colors and Material Index Interaction

When both `vertexColors` and `useMaterialIndex` are enabled:
- Material index color is applied first (as a base per-region tint)
- Vertex colors are multiplied on top (for per-vertex variation within a region)

This allows a model to have per-region base colors (via material index) with per-vertex variation (via vertex colors).

## Material Uniform Buffer Layout

Materials are uploaded as a uniform buffer (UBO) to minimize per-draw uniform calls:

```
// Per-Material UBO (bind group 1)
struct MaterialUniforms {
  color: vec4           // rgb + opacity        (16 bytes)
  emissive: vec4        // rgb + intensity       (16 bytes)
  aoIntensity: f32      // AO map intensity      (4 bytes)
  padding: vec3                                  (12 bytes)
  materialColors: array<vec4, 16>  // rgb + pad  (256 bytes)
  materialEmissive: array<vec4, 16> // rgb + int (256 bytes)
}
// Total: 560 bytes per material
```

The material UBO is updated only when material properties change, not every frame.

## Material Caching

Materials are deduplicated by their property values. When a material is created with identical properties to an existing one, the existing material's GPU resources are reused. This reduces bind group switches during rendering.

```ts
const materialCache = new Map<string, MaterialHandle>()

const getMaterialHash = (props: MaterialProps): string => {
  // Hash all property values into a string key
  // In practice, use a numeric hash for performance
}
```

## Interaction with Transparency

Materials with `transparent: true` are routed to the WBOIT pass instead of the opaque pass. See [10-transparency.md](10-transparency.md). The material's `opacity` value and any texture alpha contribute to the WBOIT weight function.

## Interaction with Bloom

Materials that have any emissive output (either via `emissive`/`emissiveIntensity` or via `materialIndexEmissive`) write to a secondary render target during the opaque/transparent pass. This emissive buffer is the input to the bloom post-process. See [09-bloom.md](09-bloom.md).
