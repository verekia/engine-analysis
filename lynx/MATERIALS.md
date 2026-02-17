# Lynx Engine — Material System

The material system describes how surfaces look. Each material maps to a set of shader defines, uniform data, and texture bindings that together produce a GPU render pipeline. Lynx ships two material types — `BasicMaterial` (unlit) and `LambertMaterial` (diffuse-only lighting) — plus a material index system that allows a single mesh to reference multiple material property sets via per-vertex indices.

Lynx uses a **Z-up, right-handed coordinate system**. TypeScript, no semicolons, `const` arrow functions.

---

## Table of Contents

1. [Material Types](#1-material-types)
2. [Material Index System](#2-material-index-system)
3. [Vertex Colors](#3-vertex-colors)
4. [Shader Generation](#4-shader-generation)
5. [Material Palette UBO Layout](#5-material-palette-ubo-layout)
6. [Texture Support](#6-texture-support)
7. [Material Sorting for Rendering](#7-material-sorting-for-rendering)

---

## 1. Material Types

### 1.1 Common Base

Every material shares a common base that the renderer uses for pipeline construction and draw call sorting.

```typescript
type BlendMode = "opaque" | "transparent" | "additive"

type CullMode = "back" | "front" | "none"

interface MaterialBase {
  readonly id: number
  readonly type: string
  readonly pipelineHash: number
  opacity: number
  transparent: boolean
  vertexColors: boolean
  depthWrite: boolean
  depthTest: boolean
  cullMode: CullMode
  blendMode: BlendMode
  readonly defines: Set<string>
  readonly textures: Map<string, TextureHandle>
}
```

The `pipelineHash` is computed from the shader defines, blend state, and depth state. Two materials with the same hash share the same GPU pipeline, even if their uniform values (color, emissive, etc.) differ.

```typescript
const computePipelineHash = (material: MaterialBase): number => {
  let hash = 0
  hash = hashCombine(hash, hashString(material.type))
  for (const define of material.defines) {
    hash = hashCombine(hash, hashString(define))
  }
  hash = hashCombine(hash, material.transparent ? 1 : 0)
  hash = hashCombine(hash, material.depthWrite ? 1 : 0)
  hash = hashCombine(hash, hashString(material.cullMode))
  hash = hashCombine(hash, hashString(material.blendMode))
  return hash >>> 0
}
```

---

### 1.2 BasicMaterial

Unlit. No lighting calculations at all. The fragment shader outputs the color directly. Good for UI elements, particles, stylized flat-shaded art, skyboxes, and debug visualization.

```typescript
interface BasicMaterial extends MaterialBase {
  readonly type: "basic"
  color: Vec3                          // linear RGB, default [1, 1, 1]
  map: TextureHandle | null            // optional color map (RGBA)
  opacity: number                      // 0..1, default 1
  transparent: boolean                 // if true, renders in OIT pass
  vertexColors: boolean                // if true, multiplies vertex color attribute
}
```

**Factory:**

```typescript
const createBasicMaterial = (params?: Partial<BasicMaterialParams>): BasicMaterial => {
  const color = params?.color ?? vec3(1, 1, 1)
  const map = params?.map ?? null
  const opacity = params?.opacity ?? 1
  const transparent = params?.transparent ?? false
  const vertexColors = params?.vertexColors ?? false

  const defines = new Set<string>()
  if (map) defines.add("USE_MAP")
  if (vertexColors) defines.add("USE_VERTEX_COLORS")

  const textures = new Map<string, TextureHandle>()
  if (map) textures.set("u_colorMap", map)

  const material: BasicMaterial = {
    id: nextMaterialId++,
    type: "basic",
    pipelineHash: 0,
    color,
    map,
    opacity,
    transparent,
    vertexColors,
    depthWrite: !transparent,
    depthTest: true,
    cullMode: "back",
    blendMode: transparent ? "transparent" : "opaque",
    defines,
    textures,
  }

  material.pipelineHash = computePipelineHash(material)
  return material
}
```

**Usage:**

```typescript
// Solid red, no texture
const redMaterial = createBasicMaterial({ color: vec3(1, 0, 0) })

// Textured sprite, transparent
const spriteMaterial = createBasicMaterial({
  map: loadTexture("particle.ktx2"),
  transparent: true,
  opacity: 0.8,
})

// Vertex-colored debug wireframe
const debugMaterial = createBasicMaterial({ vertexColors: true })
```

---

### 1.3 LambertMaterial

Diffuse-only lighting. The fragment shader computes a single N dot L term against each light, plus ambient. No specular highlight. This is the workhorse material for stylized games, low-poly art, and any scene where specular reflection is not needed.

```typescript
interface LambertMaterial extends MaterialBase {
  readonly type: "lambert"
  color: Vec3                          // linear RGB diffuse color, default [1, 1, 1]
  map: TextureHandle | null            // optional color map (RGBA)
  aoMap: TextureHandle | null          // optional ambient occlusion map (R channel)
  opacity: number                      // 0..1, default 1
  transparent: boolean                 // if true, renders in OIT pass
  vertexColors: boolean                // if true, multiplies vertex color attribute
  emissive: Vec3                       // linear RGB emissive color, default [0, 0, 0]
  emissiveIntensity: number            // scalar multiplier, default 0
}
```

**Factory:**

```typescript
const createLambertMaterial = (params?: Partial<LambertMaterialParams>): LambertMaterial => {
  const color = params?.color ?? vec3(1, 1, 1)
  const map = params?.map ?? null
  const aoMap = params?.aoMap ?? null
  const opacity = params?.opacity ?? 1
  const transparent = params?.transparent ?? false
  const vertexColors = params?.vertexColors ?? false
  const emissive = params?.emissive ?? vec3(0, 0, 0)
  const emissiveIntensity = params?.emissiveIntensity ?? 0

  const defines = new Set<string>()
  if (map) defines.add("USE_MAP")
  if (aoMap) defines.add("USE_AO_MAP")
  if (vertexColors) defines.add("USE_VERTEX_COLORS")
  if (emissiveIntensity > 0) defines.add("USE_EMISSIVE")

  const textures = new Map<string, TextureHandle>()
  if (map) textures.set("u_colorMap", map)
  if (aoMap) textures.set("u_aoMap", aoMap)

  const material: LambertMaterial = {
    id: nextMaterialId++,
    type: "lambert",
    pipelineHash: 0,
    color,
    map,
    aoMap,
    opacity,
    transparent,
    vertexColors,
    emissive,
    emissiveIntensity,
    depthWrite: !transparent,
    depthTest: true,
    cullMode: "back",
    blendMode: transparent ? "transparent" : "opaque",
    defines,
    textures,
  }

  material.pipelineHash = computePipelineHash(material)
  return material
}
```

**Usage:**

```typescript
// Simple diffuse surface
const groundMaterial = createLambertMaterial({
  color: vec3(0.3, 0.6, 0.2),
  map: loadTexture("grass_diffuse.ktx2"),
  aoMap: loadTexture("grass_ao.ktx2"),
})

// Glowing teal panel that feeds the bloom pass
const glowMaterial = createLambertMaterial({
  color: vec3(0.0, 0.8, 0.7),
  emissive: vec3(0.0, 0.8, 0.7),
  emissiveIntensity: 2.5,
})

// Semi-transparent stained glass
const glassMaterial = createLambertMaterial({
  color: vec3(0.8, 0.2, 0.1),
  transparent: true,
  opacity: 0.4,
})
```

---

## 2. Material Index System

### 2.1 The Problem

Game art is often exported as a single mesh with many logical materials — a character model with skin, cloth, metal, and leather regions, or a building with walls, windows, and trim. The traditional approach is to split the mesh into submeshes, one per material. This has downsides:

- More draw calls (one per submesh)
- Poor GPU utilization for small submeshes
- Complex asset pipeline (mesh splitting, index remapping)

### 2.2 The Solution: `_materialindex`

Lynx supports a per-vertex attribute called `_materialindex`. Each vertex carries an integer index into a **MaterialPalette** — an array of material property sets. The entire mesh is drawn in a single draw call. The vertex shader passes the index to the fragment shader as a flat varying, and the fragment shader looks up the per-material properties (color, emissive, emissive intensity) from a UBO array.

```
  ┌──────────────────────────────────────────────────────────┐
  │  Single Mesh Geometry                                     │
  │                                                           │
  │  vertex 0: position, normal, _materialindex = 0           │
  │  vertex 1: position, normal, _materialindex = 0           │
  │  vertex 2: position, normal, _materialindex = 1           │
  │  vertex 3: position, normal, _materialindex = 2           │
  │  ...                                                      │
  └──────────────────────────────────────────────────────────┘
                           │
                           ▼
  ┌──────────────────────────────────────────────────────────┐
  │  MaterialPalette (UBO)                                    │
  │                                                           │
  │  [0]: { color: [1, 1, 1],    emissive: [0, 0, 0], ei: 0 }│  white
  │  [1]: { color: [0, 0, 0],    emissive: [0, 0, 0], ei: 0 }│  black
  │  [2]: { color: [0, 0.8, 0.7],emissive: [0, 0.8, 0.7], ei: 0.7 }│  teal glow
  │  ...                                                      │
  └──────────────────────────────────────────────────────────┘
```

### 2.3 TypeScript Interface

```typescript
interface MaterialPaletteEntry {
  color: Vec3                          // linear RGB
  emissive: Vec3                       // linear RGB
  emissiveIntensity: number            // scalar multiplier
}

interface MaterialPalette {
  readonly entries: MaterialPaletteEntry[]
  readonly buffer: Float32Array        // GPU-ready packed data
  readonly ubo: GalBuffer             // GPU uniform buffer
  dirty: boolean
}

const MAX_PALETTE_ENTRIES = 64

const createMaterialPalette = (
  device: GalDevice,
  entries: MaterialPaletteEntry[]
): MaterialPalette => {
  if (entries.length > MAX_PALETTE_ENTRIES) {
    throw new Error(`MaterialPalette: max ${MAX_PALETTE_ENTRIES} entries, got ${entries.length}`)
  }

  // 32 bytes per entry: vec4 (color + pad) + vec4 (emissive + emissiveIntensity)
  const buffer = new Float32Array(MAX_PALETTE_ENTRIES * 8)
  const ubo = device.createBuffer({
    label: "material-palette-ubo",
    size: buffer.byteLength,
    usage: ["uniform"],
    dynamic: true,
  })

  const palette: MaterialPalette = {
    entries,
    buffer,
    ubo,
    dirty: true,
  }

  packPaletteBuffer(palette)
  return palette
}

const packPaletteBuffer = (palette: MaterialPalette): void => {
  const buf = palette.buffer
  for (let i = 0; i < palette.entries.length; i++) {
    const entry = palette.entries[i]
    const offset = i * 8   // 8 floats per entry (two vec4s)

    // vec4: color.rgb + padding
    buf[offset + 0] = entry.color[0]
    buf[offset + 1] = entry.color[1]
    buf[offset + 2] = entry.color[2]
    buf[offset + 3] = 0   // padding (std140 alignment)

    // vec4: emissive.rgb + emissiveIntensity
    buf[offset + 4] = entry.emissive[0]
    buf[offset + 5] = entry.emissive[1]
    buf[offset + 6] = entry.emissive[2]
    buf[offset + 7] = entry.emissiveIntensity
  }
  palette.dirty = true
}

const updatePaletteEntry = (
  palette: MaterialPalette,
  index: number,
  entry: Partial<MaterialPaletteEntry>
): void => {
  const existing = palette.entries[index]
  if (entry.color) existing.color = entry.color
  if (entry.emissive) existing.emissive = entry.emissive
  if (entry.emissiveIntensity !== undefined) existing.emissiveIntensity = entry.emissiveIntensity
  packPaletteBuffer(palette)
}

const uploadPalette = (device: GalDevice, palette: MaterialPalette): void => {
  if (!palette.dirty) return
  device.writeBuffer(palette.ubo, palette.buffer)
  palette.dirty = false
}
```

### 2.4 Geometry Attribute

The `_materialindex` attribute is stored as a vertex attribute on the geometry, alongside position, normal, UV, etc. It can be `uint8` (0-255) or `uint16` (0-65535), though in practice only 64 values are used since that is the palette UBO limit.

```typescript
type AttributeSemantic =
  | "position"
  | "normal"
  | "tangent"
  | "texcoord0"
  | "texcoord1"
  | "color0"
  | "joints0"
  | "weights0"
  | "_materialindex"     // <-- custom attribute

interface VertexAttribute {
  semantic: AttributeSemantic
  format: VertexFormat        // "uint8", "uint16", "float32x3", etc.
  offset: number
  stride: number
  bufferSlot: number
}
```

When a geometry has the `_materialindex` attribute, the material system automatically adds the `USE_MATERIAL_INDEX` define to the shader.

### 2.5 Shader Code

The vertex shader reads `_materialindex` as an integer attribute and passes it to the fragment shader as a `flat` varying. The fragment shader uses it to index into the palette UBO.

**Vertex Shader (GLSL 300 es):**

```glsl
#version 300 es
precision highp float;

// --- Uniforms ---
layout(std140) uniform FrameUniforms {
    mat4 u_viewMatrix;
    mat4 u_projectionMatrix;
    vec3 u_cameraPosition;
    float u_time;
    vec3 u_sunDirection;
    float u_padding0;
    vec3 u_sunColor;
    float u_sunIntensity;
    vec3 u_ambientColor;
    float u_ambientIntensity;
#ifdef SHADOW_RECEIVE
    mat4 u_shadowMatrices[3];
    float u_cascadeSplits[3];
#endif
};

layout(std140) uniform ObjectUniforms {
    mat4 u_worldMatrix;
};

// --- Attributes ---
layout(location = 0) in vec3 a_position;
layout(location = 1) in vec3 a_normal;
#ifdef USE_MAP
layout(location = 2) in vec2 a_uv;
#endif
#ifdef USE_VERTEX_COLORS
layout(location = 3) in vec4 a_color;
#endif
#ifdef USE_SKINNING
layout(location = 4) in vec4 a_joints;
layout(location = 5) in vec4 a_weights;
#endif
#ifdef USE_MATERIAL_INDEX
layout(location = 6) in uint a_materialIndex;
#endif

// --- Varyings ---
out vec3 v_worldPosition;
out vec3 v_worldNormal;
#ifdef USE_MAP
out vec2 v_uv;
#endif
#ifdef USE_VERTEX_COLORS
out vec4 v_color;
#endif
#ifdef USE_MATERIAL_INDEX
flat out uint v_materialIndex;
#endif
out float v_viewDepth;

void main() {
    vec4 localPos = vec4(a_position, 1.0);
    vec3 localNormal = a_normal;

#ifdef USE_SKINNING
    // ... skinning omitted for brevity ...
#endif

    vec4 worldPos = u_worldMatrix * localPos;
    v_worldPosition = worldPos.xyz;
    v_worldNormal = normalize(mat3(u_worldMatrix) * localNormal);

#ifdef USE_MAP
    v_uv = a_uv;
#endif
#ifdef USE_VERTEX_COLORS
    v_color = a_color;
#endif
#ifdef USE_MATERIAL_INDEX
    v_materialIndex = a_materialIndex;
#endif

    vec4 viewPos = u_viewMatrix * worldPos;
    v_viewDepth = -viewPos.z;

    gl_Position = u_projectionMatrix * viewPos;
}
```

**Fragment Shader — Lambert with Material Index (GLSL 300 es):**

```glsl
#version 300 es
precision highp float;

// --- Frame Uniforms ---
layout(std140) uniform FrameUniforms {
    mat4 u_viewMatrix;
    mat4 u_projectionMatrix;
    vec3 u_cameraPosition;
    float u_time;
    vec3 u_sunDirection;
    float u_padding0;
    vec3 u_sunColor;
    float u_sunIntensity;
    vec3 u_ambientColor;
    float u_ambientIntensity;
};

// --- Material Palette UBO ---
#ifdef USE_MATERIAL_INDEX
struct PaletteEntry {
    vec4 colorAndPad;          // rgb = color, a = padding
    vec4 emissiveAndIntensity; // rgb = emissive color, a = emissive intensity
};

layout(std140) uniform MaterialPalette {
    PaletteEntry u_palette[64];
};
#else
layout(std140) uniform MaterialUniforms {
    vec4 u_baseColor;
    vec4 u_emissiveAndIntensity;
};
#endif

// --- Textures ---
#ifdef USE_MAP
uniform sampler2D u_colorMap;
#endif
#ifdef USE_AO_MAP
uniform sampler2D u_aoMap;
#endif

// --- Varyings ---
in vec3 v_worldPosition;
in vec3 v_worldNormal;
#ifdef USE_MAP
in vec2 v_uv;
#endif
#ifdef USE_VERTEX_COLORS
in vec4 v_color;
#endif
#ifdef USE_MATERIAL_INDEX
flat in uint v_materialIndex;
#endif
in float v_viewDepth;

// --- Output ---
layout(location = 0) out vec4 fragColor;

void main() {
    // --- Determine base color and emissive from material source ---
    vec3 baseColor;
    vec3 emissiveColor;
    float emissiveIntensity;

#ifdef USE_MATERIAL_INDEX
    PaletteEntry entry = u_palette[v_materialIndex];
    baseColor = entry.colorAndPad.rgb;
    emissiveColor = entry.emissiveAndIntensity.rgb;
    emissiveIntensity = entry.emissiveAndIntensity.a;
#else
    baseColor = u_baseColor.rgb;
    emissiveColor = u_emissiveAndIntensity.rgb;
    emissiveIntensity = u_emissiveAndIntensity.a;
#endif

    vec4 albedo = vec4(baseColor, 1.0);

    // --- Texture map ---
#ifdef USE_MAP
    albedo *= texture(u_colorMap, v_uv);
#endif

    // --- Vertex colors ---
#ifdef USE_VERTEX_COLORS
    albedo *= v_color;
#endif

    // --- Ambient occlusion ---
    float ao = 1.0;
#ifdef USE_AO_MAP
    ao = texture(u_aoMap, v_uv).r;
#endif

    // --- Lambert diffuse lighting ---
    vec3 N = normalize(v_worldNormal);
    vec3 L = normalize(u_sunDirection);
    float NdotL = max(dot(N, L), 0.0);

    vec3 diffuse = u_sunColor * u_sunIntensity * NdotL;
    vec3 ambient = u_ambientColor * u_ambientIntensity * ao;

    vec3 lighting = ambient + diffuse;
    vec3 color = albedo.rgb * lighting;

    // --- Emissive contribution (feeds bloom extraction) ---
#ifdef USE_EMISSIVE
    color += emissiveColor * emissiveIntensity;
#endif
#ifdef USE_MATERIAL_INDEX
    // Material index always supports emissive (per-entry)
    color += emissiveColor * emissiveIntensity;
#endif

    fragColor = vec4(color, albedo.a);
}
```

**WGSL equivalent (WebGPU):**

```wgsl
// Material Palette struct
struct PaletteEntry {
    color_and_pad: vec4<f32>,             // rgb = color, a = padding
    emissive_and_intensity: vec4<f32>,     // rgb = emissive, a = intensity
}

struct MaterialPaletteUBO {
    entries: array<PaletteEntry, 64>,
}

@group(1) @binding(0) var<uniform> palette: MaterialPaletteUBO;

struct FragmentInput {
    @location(0) world_position: vec3<f32>,
    @location(1) world_normal: vec3<f32>,
#ifdef USE_MAP
    @location(2) uv: vec2<f32>,
#endif
#ifdef USE_VERTEX_COLORS
    @location(3) color: vec4<f32>,
#endif
#ifdef USE_MATERIAL_INDEX
    @location(4) @interpolate(flat) material_index: u32,
#endif
    @location(5) view_depth: f32,
}

@fragment
fn main(input: FragmentInput) -> @location(0) vec4<f32> {
    let entry = palette.entries[input.material_index];
    let base_color = entry.color_and_pad.rgb;
    let emissive_color = entry.emissive_and_intensity.rgb;
    let emissive_intensity = entry.emissive_and_intensity.a;

    var albedo = vec4<f32>(base_color, 1.0);

    // ... texture, vertex color, lighting as above ...

    var color = albedo.rgb * lighting;
    color += emissive_color * emissive_intensity;

    return vec4<f32>(color, albedo.a);
}
```

### 2.6 Example: Setting Up a Material-Indexed Mesh

```typescript
// Define the palette: 3 material entries
const palette = createMaterialPalette(device, [
  // Index 0: white (walls)
  {
    color: vec3(1, 1, 1),
    emissive: vec3(0, 0, 0),
    emissiveIntensity: 0,
  },
  // Index 1: black (window frames)
  {
    color: vec3(0, 0, 0),
    emissive: vec3(0, 0, 0),
    emissiveIntensity: 0,
  },
  // Index 2: teal with emissive glow (neon trim)
  {
    color: vec3(0, 0.8, 0.7),
    emissive: vec3(0, 0.8, 0.7),
    emissiveIntensity: 0.7,
  },
])

// Create a Lambert material with material index support
const buildingMaterial = createLambertMaterial({
  color: vec3(1, 1, 1),  // base color (overridden by palette in shader)
})
buildingMaterial.defines.add("USE_MATERIAL_INDEX")
buildingMaterial.pipelineHash = computePipelineHash(buildingMaterial)

// Load the geometry — exported from Blender as a single mesh
// with the _materialindex vertex attribute baked in
const buildingGeometry = loadGeometry("building.glb")
// buildingGeometry.attributes includes:
//   position: float32x3
//   normal:   float32x3
//   texcoord0: float32x2
//   _materialindex: uint8   <-- per-vertex material index

// Create the mesh
const building = createMesh(buildingGeometry, buildingMaterial)
building.materialPalette = palette
addChild(scene, building)

// During rendering, the palette UBO is bound alongside the material.
// Fragments with _materialindex = 2 will glow teal and feed the bloom pass.
// The bloom extraction shader picks up any fragment where
// emissiveColor * emissiveIntensity > 0 and routes it into the
// downscale/upscale bloom chain.
```

### 2.7 How Emissive Feeds Bloom

The emissive output from the material index system connects directly to the post-processing bloom pipeline:

```
Fragment shader outputs:
  color = albedo * lighting + emissiveColor * emissiveIntensity
              │
              ▼
Opaque pass writes to MSAA color target (RGBA16F)
              │
              ▼
Bloom extraction pass: bright pixels where luminance > threshold
  Fragments with emissiveIntensity > 0 are naturally above
  the threshold because their color is boosted beyond lighting.
              │
              ▼
Bloom downscale/upscale chain → soft glow halo
              │
              ▼
Tone mapping combines bloom with scene color
              │
              ▼
Final output: teal neon trim glows on the building
```

---

## 3. Vertex Colors

### 3.1 How Vertex Colors Work

When `vertexColors: true` is set on a material, the engine enables the `USE_VERTEX_COLORS` shader define. The geometry must provide a `color0` vertex attribute (`vec4`, typically `unorm8x4` — four bytes per vertex, normalized to 0..1).

In the fragment shader, the vertex color modulates the base color:

```glsl
#ifdef USE_VERTEX_COLORS
    albedo *= v_color;
#endif
```

This multiplication is component-wise RGBA. A vertex color of `[1, 1, 1, 1]` has no effect. A vertex color of `[1, 0, 0, 1]` tints the surface red. A vertex color with `a < 1` reduces opacity (relevant for transparent materials).

### 3.2 Vertex Colors with Material Index

Vertex colors and the material index system can be used together. The combination order is:

```
finalColor = vertexColor * paletteColor
```

This means:

1. The palette entry provides the base color for the material region.
2. The vertex color modulates that base color per-vertex.

This is useful for baked lighting (vertex colors from light baking) applied on top of material-index-driven coloring.

```glsl
// In the fragment shader, when both are active:
#ifdef USE_MATERIAL_INDEX
    PaletteEntry entry = u_palette[v_materialIndex];
    baseColor = entry.colorAndPad.rgb;
#endif

vec4 albedo = vec4(baseColor, 1.0);

#ifdef USE_VERTEX_COLORS
    albedo *= v_color;    // modulates palette color
#endif
```

### 3.3 Vertex Colors Without Material Index

Vertex colors can be used standalone, without the material index system. In this case, the material's `color` property serves as the base, and vertex colors modulate it:

```
finalColor = vertexColor * material.color
```

```typescript
// A mesh with per-vertex coloring and no palette
const coloredMesh = createMesh(
  coloredGeometry,   // geometry with color0 attribute
  createBasicMaterial({ vertexColors: true })
)
addChild(scene, coloredMesh)
```

### 3.4 Interaction Matrix

| vertexColors | materialIndex | Result |
|---|---|---|
| false | false | `albedo = material.color` |
| true | false | `albedo = material.color * vertexColor` |
| false | true | `albedo = paletteColor` |
| true | true | `albedo = paletteColor * vertexColor` |

---

## 4. Shader Generation

### 4.1 Preprocessor Defines

Materials map to shader variants through preprocessor defines. Each combination of defines produces a distinct shader program. The engine compiles variants lazily on first use and caches them.

| Define | Type | Triggered By |
|---|---|---|
| `USE_MAP` | boolean | `material.map !== null` |
| `USE_AO_MAP` | boolean | `material.aoMap !== null` |
| `USE_VERTEX_COLORS` | boolean | `material.vertexColors === true` |
| `USE_MATERIAL_INDEX` | boolean | geometry has `_materialindex` attribute |
| `USE_SKINNING` | boolean | mesh is a `SkinnedMesh` |
| `USE_EMISSIVE` | boolean | `material.emissiveIntensity > 0` |
| `SHADOW_RECEIVE` | boolean | mesh `receiveShadow === true` and scene has shadow-casting light |

### 4.2 Define Collection

When the renderer encounters a mesh, it collects defines from both the material and the mesh context:

```typescript
const collectDefines = (mesh: Mesh, material: MaterialBase): Set<string> => {
  const defines = new Set(material.defines)

  // Geometry-driven defines
  if (mesh.geometry.hasAttribute("_materialindex")) {
    defines.add("USE_MATERIAL_INDEX")
  }

  // Mesh-driven defines
  if (mesh.type === "SkinnedMesh") {
    defines.add("USE_SKINNING")
  }
  if (mesh.receiveShadow) {
    defines.add("SHADOW_RECEIVE")
  }

  return defines
}
```

### 4.3 Pipeline Key and Caching

The pipeline key is a hash of the shader defines combined with the blend state and depth state. Meshes that produce the same key share the same GPU pipeline.

```typescript
interface PipelineKey {
  readonly shaderType: string           // "basic" | "lambert"
  readonly defines: ReadonlySet<string>
  readonly blendMode: BlendMode
  readonly depthWrite: boolean
  readonly depthTest: boolean
  readonly cullMode: CullMode
}

const pipelineKeyToHash = (key: PipelineKey): number => {
  let hash = hashString(key.shaderType)
  const sortedDefines = [...key.defines].sort()
  for (const d of sortedDefines) {
    hash = hashCombine(hash, hashString(d))
  }
  hash = hashCombine(hash, hashString(key.blendMode))
  hash = hashCombine(hash, key.depthWrite ? 1 : 0)
  hash = hashCombine(hash, key.depthTest ? 1 : 0)
  hash = hashCombine(hash, hashString(key.cullMode))
  return hash >>> 0
}

// Pipeline cache: hash -> GalRenderPipeline
const pipelineCache = new Map<number, GalRenderPipeline>()

const getOrCreatePipeline = (
  device: GalDevice,
  key: PipelineKey
): GalRenderPipeline => {
  const hash = pipelineKeyToHash(key)
  const cached = pipelineCache.get(hash)
  if (cached) return cached

  // Compile new shader variant
  const vertSource = buildShaderSource(vertexShaderTemplate, key.defines)
  const fragSource = buildShaderSource(
    key.shaderType === "basic" ? basicFragTemplate : lambertFragTemplate,
    key.defines
  )

  const vertModule = device.createShaderModule({
    label: `${key.shaderType}-vert-${hash}`,
    code: vertSource,
    stage: "vertex",
  })

  const fragModule = device.createShaderModule({
    label: `${key.shaderType}-frag-${hash}`,
    code: fragSource,
    stage: "fragment",
  })

  const pipeline = device.createRenderPipeline({
    label: `pipeline-${hash}`,
    vertex: {
      module: vertModule,
      entryPoint: "main",
      buffers: buildVertexBufferLayouts(key.defines),
    },
    fragment: {
      module: fragModule,
      entryPoint: "main",
      targets: [{ format: "rgba16float", blend: getBlendState(key.blendMode) }],
    },
    depthStencil: {
      format: "depth24plus",
      depthWriteEnabled: key.depthWrite,
      depthCompare: key.depthTest ? "less" : "always",
    },
    primitive: {
      topology: "triangle-list",
      cullMode: key.cullMode,
      frontFace: "ccw",
    },
    multisample: { count: 4 },
  })

  pipelineCache.set(hash, pipeline)
  return pipeline
}
```

### 4.4 Shader Source Injection

Defines are injected as `#define` lines (GLSL) or as constant declarations (WGSL) before compilation.

```typescript
const buildShaderSource = (
  template: string,
  defines: ReadonlySet<string>
): string => {
  let header = ""
  for (const define of defines) {
    header += `#define ${define}\n`
  }
  // Insert after #version line for GLSL
  const versionEnd = template.indexOf("\n") + 1
  return template.slice(0, versionEnd) + header + template.slice(versionEnd)
}
```

### 4.5 Valid Variant Combinations

Not all define combinations are valid. The build system prunes invalid permutations:

| Combination | Valid? | Reason |
|---|---|---|
| `USE_MAP` + `USE_VERTEX_COLORS` | Yes | Texture modulated by vertex color |
| `USE_MATERIAL_INDEX` + `USE_MAP` | Yes | Palette color + texture |
| `USE_MATERIAL_INDEX` + `USE_EMISSIVE` | Redundant | Palette entries always carry emissive; `USE_EMISSIVE` is for non-palette materials |
| `USE_SKINNING` + `USE_MATERIAL_INDEX` | Yes | Skinned mesh with per-vertex material selection |
| `USE_AO_MAP` without `USE_MAP` | Valid but unusual | AO requires UV coordinates; having AO without a color map is uncommon but supported |

---

## 5. Material Palette UBO Layout

### 5.1 Constraints

- **Max 64 entries per palette.** This fits comfortably within the minimum guaranteed UBO size on both WebGPU (65536 bytes) and WebGL2 (16384 bytes).
- **32 bytes per entry.** Two `vec4`s in std140 layout.
- **Total: 32 x 64 = 2048 bytes per palette UBO.** Well within limits.

### 5.2 GLSL Struct Layout (std140)

```glsl
// GLSL 300 es — std140 layout
struct PaletteEntry {
    vec4 colorAndPad;              // offset 0:  [r, g, b, pad]      16 bytes
    vec4 emissiveAndIntensity;     // offset 16: [r, g, b, intensity] 16 bytes
};                                 // total: 32 bytes per entry

layout(std140) uniform MaterialPalette {
    PaletteEntry u_palette[64];    // 32 * 64 = 2048 bytes
};
```

**Memory layout (per entry):**

```
Byte offset   Field                  Type      Size
─────────────────────────────────────────────────
 0            color.r                float32   4
 4            color.g                float32   4
 8            color.b                float32   4
12            (padding)              float32   4
16            emissive.r             float32   4
20            emissive.g             float32   4
24            emissive.b             float32   4
28            emissiveIntensity      float32   4
─────────────────────────────────────────────────
32 bytes total per entry
```

### 5.3 WGSL Struct Layout

```wgsl
struct PaletteEntry {
    color_and_pad: vec4<f32>,               // 16 bytes: rgb + pad
    emissive_and_intensity: vec4<f32>,       // 16 bytes: rgb + intensity
}
// 32 bytes per entry, naturally aligned for WGSL uniform rules

struct MaterialPaletteUBO {
    entries: array<PaletteEntry, 64>,        // 2048 bytes total
}

@group(1) @binding(0) var<uniform> palette: MaterialPaletteUBO;
```

### 5.4 CPU-Side Packing

The `Float32Array` buffer is packed to match the std140 layout exactly:

```typescript
// Palette buffer layout: 8 floats per entry
//
// entry[i] starts at float offset i * 8
//
// [i*8 + 0] = color.r
// [i*8 + 1] = color.g
// [i*8 + 2] = color.b
// [i*8 + 3] = 0.0  (padding)
// [i*8 + 4] = emissive.r
// [i*8 + 5] = emissive.g
// [i*8 + 6] = emissive.b
// [i*8 + 7] = emissiveIntensity

const PALETTE_FLOATS_PER_ENTRY = 8
const PALETTE_BYTES_PER_ENTRY = 32
const PALETTE_MAX_ENTRIES = 64
const PALETTE_TOTAL_BYTES = PALETTE_BYTES_PER_ENTRY * PALETTE_MAX_ENTRIES  // 2048
```

### 5.5 Bind Group Layout

The palette UBO occupies bind group 1 when the `USE_MATERIAL_INDEX` define is active. When using a regular (non-indexed) material, bind group 1 holds the per-material uniform buffer instead.

```
Bind Group 0: Per-frame data (camera, lights, shadow matrices)
Bind Group 1: Per-material data
              ├── Without material index: MaterialUniforms UBO + textures
              └── With material index:    MaterialPalette UBO + textures
Bind Group 2: Per-object data (world matrix, dynamic offset)
```

---

## 6. Texture Support

### 6.1 Texture Formats

| Texture Type | Primary Format | Compressed Alternatives | Usage |
|---|---|---|---|
| Color map | RGBA8 unorm | BC7 (desktop), ASTC 4x4 (mobile), ETC2 (fallback mobile) | Surface color / albedo |
| AO map | R8 unorm | Part of a compressed channel pack | Ambient occlusion |
| Emissive map | RGBA8 unorm | BC7 / ASTC / ETC2 | Emissive surface detail (future) |

Compressed textures are delivered via **KTX2** containers with **Basis Universal** supercompressed payloads. At load time, a Web Worker transcodes the Basis data to the best format supported by the device:

```typescript
const selectCompressedFormat = (device: GalDevice): GalTextureFormat => {
  const formats = device.limits.compressedTextureFormats

  // Prefer BC7 (desktop), then ASTC (high-end mobile), then ETC2 (mobile fallback)
  if (formats.includes("bc7-rgba-unorm")) return "bc7-rgba-unorm"
  if (formats.includes("astc-4x4-unorm")) return "astc-4x4-unorm"
  if (formats.includes("etc2-rgba8unorm")) return "etc2-rgba8unorm"

  // No compressed format — fall back to uncompressed RGBA8
  return "rgba8unorm"
}
```

### 6.2 Texture Handles

Textures are referenced by handle, not by value. Multiple materials can reference the same texture without duplicating GPU memory.

```typescript
interface TextureHandle {
  readonly id: number
  readonly texture: GalTexture
  readonly view: GalTextureView
  readonly width: number
  readonly height: number
  readonly format: GalTextureFormat
  readonly mipLevelCount: number
}

// Texture registry — shared across all materials
const textureRegistry = new Map<string, TextureHandle>()

const loadTexture = async (
  device: GalDevice,
  url: string
): Promise<TextureHandle> => {
  const existing = textureRegistry.get(url)
  if (existing) return existing

  const data = await fetchAndDecode(url)

  const mipCount = Math.floor(Math.log2(Math.max(data.width, data.height))) + 1

  const texture = device.createTexture({
    label: url,
    dimension: "2d",
    format: data.format,
    width: data.width,
    height: data.height,
    mipLevelCount: mipCount,
    usage: ["texture-binding", "copy-dst"],
  })

  // Upload base level and generate mipmaps
  device.writeTexture(texture, data.pixels, {
    width: data.width,
    height: data.height,
    mipLevel: 0,
  })
  generateMipmaps(device, texture, mipCount)

  const handle: TextureHandle = {
    id: nextTextureId++,
    texture,
    view: texture.createView(),
    width: data.width,
    height: data.height,
    format: data.format,
    mipLevelCount: mipCount,
  }

  textureRegistry.set(url, handle)
  return handle
}
```

### 6.3 Sampler Configuration

Samplers are created once and reused. The default sampler uses trilinear filtering with mipmaps and anisotropic filtering when the hardware supports it.

```typescript
const createDefaultSampler = (device: GalDevice): GalSampler => {
  const maxAniso = device.limits.maxSamplersPerShaderStage > 0 ? 4 : 1

  return device.createSampler({
    label: "default-sampler",
    magFilter: "linear",
    minFilter: "linear",
    mipmapFilter: "linear",
    addressModeU: "repeat",
    addressModeV: "repeat",
    maxAnisotropy: maxAniso,
  })
}

const createClampSampler = (device: GalDevice): GalSampler =>
  device.createSampler({
    label: "clamp-sampler",
    magFilter: "linear",
    minFilter: "linear",
    mipmapFilter: "linear",
    addressModeU: "clamp-to-edge",
    addressModeV: "clamp-to-edge",
    maxAnisotropy: 1,
  })

const createShadowSampler = (device: GalDevice): GalSampler =>
  device.createSampler({
    label: "shadow-sampler",
    magFilter: "linear",
    minFilter: "linear",
    compare: "less-equal",
  })
```

### 6.4 Texture Binding in Materials

When a material has textures, they are bound in the material's bind group alongside the uniform buffer:

```typescript
const createMaterialBindGroup = (
  device: GalDevice,
  material: MaterialBase,
  sampler: GalSampler,
  paletteUBO?: GalBuffer
): GalBindGroup => {
  const entries: GalBindGroupEntry[] = []

  // Binding 0: Material UBO or Palette UBO
  if (paletteUBO) {
    entries.push({ binding: 0, resource: { type: "buffer", buffer: paletteUBO } })
  } else {
    entries.push({ binding: 0, resource: { type: "buffer", buffer: createMaterialUBO(device, material) } })
  }

  // Binding 1: Sampler
  entries.push({ binding: 1, resource: { type: "sampler", sampler } })

  // Binding 2+: Textures
  let textureBinding = 2
  if (material.textures.has("u_colorMap")) {
    entries.push({
      binding: textureBinding++,
      resource: { type: "texture", textureView: material.textures.get("u_colorMap")!.view },
    })
  }
  if (material.textures.has("u_aoMap")) {
    entries.push({
      binding: textureBinding++,
      resource: { type: "texture", textureView: material.textures.get("u_aoMap")!.view },
    })
  }

  return device.createBindGroup({
    label: `material-bg-${material.id}`,
    layout: getMaterialBindGroupLayout(material.defines),
    entries,
  })
}
```

---

## 7. Material Sorting for Rendering

### 7.1 Material ID in the Sort Key

Every material is assigned a unique numeric `id` at creation time. This ID is packed into the 64-bit sort key used by the render command list, occupying bits 48-59 (12 bits, supporting up to 4096 unique materials).

```
Sort Key (64 bits):

  63       60 59            48 47                                  0
  ┌─────────┬────────────────┬─────────────────────────────────────┐
  │Pipeline │  Material ID   │              Depth                  │
  │ 4 bits  │   12 bits      │             48 bits                 │
  └─────────┴────────────────┴─────────────────────────────────────┘
```

Sorting by pipeline first, then by material, ensures that:
1. Pipeline switches (most expensive) are minimized.
2. Within the same pipeline, material bind group switches are minimized.
3. Within the same material, draws are ordered front-to-back (opaque) for maximum early-Z rejection.

### 7.2 Opaque Sorting

Opaque materials are sorted to minimize GPU state changes. The sort order is:

1. **Pipeline hash** (highest priority) — groups draws by shader variant + blend/depth state
2. **Material ID** — groups draws sharing the same textures and uniforms
3. **Depth front-to-back** — maximizes early-Z rejection

```typescript
const buildSortKeyOpaque = (
  pipelineId: number,
  materialId: number,
  viewSpaceDepth: number,
  nearPlane: number,
  farPlane: number
): bigint => {
  const pBits = BigInt(pipelineId & 0xF) << 60n
  const mBits = BigInt(materialId & 0xFFF) << 48n
  const t = (viewSpaceDepth - nearPlane) / (farPlane - nearPlane)
  const clamped = Math.max(0, Math.min(1, t))
  const dBits = BigInt(Math.floor(clamped * 0xFFFFFFFFFFFF)) & 0xFFFFFFFFFFFFn
  return pBits | mBits | dBits
}
```

### 7.3 Transparent Sorting

Transparent materials are rendered in the Weighted Blended OIT pass. OIT is order-independent by design, so exact sort order is less critical. However, back-to-front input ordering improves weight function quality for overlapping surfaces at similar depths.

```typescript
const buildSortKeyTransparent = (
  pipelineId: number,
  materialId: number,
  viewSpaceDepth: number,
  nearPlane: number,
  farPlane: number
): bigint => {
  const pBits = BigInt(pipelineId & 0xF) << 60n
  const mBits = BigInt(materialId & 0xFFF) << 48n
  // Invert depth: farther objects sort first (smaller key)
  const t = (viewSpaceDepth - nearPlane) / (farPlane - nearPlane)
  const clamped = Math.max(0, Math.min(1, t))
  const inverted = 1.0 - clamped
  const dBits = BigInt(Math.floor(inverted * 0xFFFFFFFFFFFF)) & 0xFFFFFFFFFFFFn
  return pBits | mBits | dBits
}
```

### 7.4 State Change Cost Hierarchy

The renderer exploits the fact that GPU state changes have vastly different costs:

| State Change | Relative Cost | Frequency After Sorting |
|---|---|---|
| Pipeline switch (shader + blend + depth) | Very high | Rare (typically 2-6 per frame) |
| Material bind group switch (UBO + textures) | Medium | Moderate (once per unique material) |
| Per-object dynamic UBO offset | Low | Every draw call |
| Vertex/index buffer switch | Low | Every draw call (or less, with shared geometry) |

By sorting commands so that expensive state changes occur as infrequently as possible, the renderer achieves high draw call throughput. On a typical scene with 5 shader variants and 50 materials, sorting reduces pipeline switches from potentially 2000 (one per draw) to 5, and material switches to 50.

### 7.5 Interaction with Material Palette

Meshes using the material index system share a single material and a single palette UBO. From the renderer's perspective, all vertices in such a mesh have the same material ID. This means:

- One draw call for the entire mesh (no submesh splitting)
- One material bind group switch (the palette UBO)
- Per-vertex material variation happens entirely in the shader

This is a significant draw call optimization. A building with 30 material regions that would traditionally require 30 draw calls becomes a single draw call with a single pipeline and a single material bind group.

---

## Complete Usage Example

```typescript
// --- Initialize engine ---
const device = await createGalDevice(canvas, "webgpu")
const defaultSampler = createDefaultSampler(device)

// --- Load textures ---
const grassDiffuse = await loadTexture(device, "grass_diffuse.ktx2")
const grassAO = await loadTexture(device, "grass_ao.ktx2")
const crateTexture = await loadTexture(device, "crate.ktx2")

// --- Create materials ---

// Ground: Lambert with diffuse + AO textures
const groundMaterial = createLambertMaterial({
  color: vec3(0.4, 0.7, 0.3),
  map: grassDiffuse,
  aoMap: grassAO,
})

// Crate: Lambert with texture
const crateMaterial = createLambertMaterial({
  color: vec3(1, 1, 1),
  map: crateTexture,
})

// UI overlay: Basic unlit, transparent
const uiMaterial = createBasicMaterial({
  color: vec3(1, 1, 1),
  transparent: true,
  opacity: 0.9,
})

// Debug wireframe: Basic with vertex colors
const debugMaterial = createBasicMaterial({
  vertexColors: true,
})

// Building with material palette (single mesh, multiple materials)
const buildingPalette = createMaterialPalette(device, [
  { color: vec3(0.9, 0.85, 0.8), emissive: vec3(0, 0, 0), emissiveIntensity: 0 },
  { color: vec3(0.2, 0.2, 0.25), emissive: vec3(0, 0, 0), emissiveIntensity: 0 },
  { color: vec3(0, 0.8, 0.7),    emissive: vec3(0, 0.8, 0.7), emissiveIntensity: 0.7 },
  { color: vec3(1, 0.3, 0.1),    emissive: vec3(1, 0.3, 0.1), emissiveIntensity: 1.5 },
])

const buildingMaterial = createLambertMaterial({ color: vec3(1, 1, 1) })
buildingMaterial.defines.add("USE_MATERIAL_INDEX")
buildingMaterial.pipelineHash = computePipelineHash(buildingMaterial)

// --- Create scene ---
const scene = createScene()
scene.ambientLight = { color: vec3(1, 1, 1), intensity: 0.3 }

const ground = createMesh(planeGeometry, groundMaterial)
addChild(scene, ground)

const crate = createMesh(boxGeometry, crateMaterial)
setPosition(crate, 3, -2, 0.5)
addChild(scene, crate)

const building = createMesh(buildingGeometry, buildingMaterial)
building.materialPalette = buildingPalette
setPosition(building, 0, 0, 0)
addChild(scene, building)

// --- Render loop ---
const frame = (): void => {
  // Upload palette if changed
  uploadPalette(device, buildingPalette)

  // Animate: pulse the neon trim emissive intensity
  const t = performance.now() / 1000
  updatePaletteEntry(buildingPalette, 2, {
    emissiveIntensity: 0.5 + 0.3 * Math.sin(t * 2),
  })

  // Render (handles frustum cull, sort, multi-pass)
  renderer.render(scene, camera)

  requestAnimationFrame(frame)
}

requestAnimationFrame(frame)
```

---

## Summary

| Concept | Key Detail |
|---|---|
| BasicMaterial | Unlit, color + optional texture, no lighting math |
| LambertMaterial | Diffuse N dot L + ambient, supports AO and emissive |
| `_materialindex` | Per-vertex uint attribute indexing into a MaterialPalette UBO |
| MaterialPalette | Array of up to 64 entries, 32 bytes each, in a single UBO (2048 bytes) |
| Vertex colors | Modulate base color (from material or palette) per-vertex |
| Shader variants | Preprocessor defines produce cached pipeline permutations |
| Emissive | Feeds bloom extraction; fragments with emissive > 0 glow |
| Sorting | Pipeline (4 bits) > Material ID (12 bits) > Depth (48 bits) |
| Textures | KTX2/Basis compressed, shared by handle, trilinear + aniso sampling |
| Transparency | Weighted Blended OIT, order-independent, separate pass |
