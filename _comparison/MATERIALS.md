# Materials Comparison Across 9 Implementations

## Universal Agreement

All 9 implementations converge on these material system fundamentals:

### 1. Two Core Material Types
Every implementation provides exactly two materials:
- **BasicMaterial**: Unlit (no lighting calculations)
- **LambertMaterial**: Diffuse lighting only (N·L model)

None implement PBR by default—all target stylized/low-poly aesthetics where Lambert is sufficient.

### 2. Material Index System
**8 out of 9** implementations support a per-vertex `_materialindex` attribute that maps vertices to material property slots:
- Single mesh, single draw call, multiple "materials" per vertex
- Eliminates mesh splitting for multi-material models
- Index stored as `uint8` or `uint16` vertex attribute
- Color and emissive properties looked up from a palette/table in shader

**Exception:** bonobo (no material index system in v1)

### 3. Vertex Colors
All implementations support vertex colors as a multiplicative factor on base color:
```
finalColor = materialColor × textureColor × vertexColor
```
When combined with material index:
```
finalColor = paletteColor × vertexColor × textureColor
```

### 4. Emissive Output for Bloom
All implementations write emissive contribution to support bloom post-processing:
- Emissive pixels bypass lighting (added after lighting calculation)
- Emissive intensity drives bloom extraction
- Material index system can assign per-vertex emissive values

### 5. Transparency via OIT
All implementations route transparent materials to a Weighted Blended Order-Independent Transparency (OIT) pass instead of traditional alpha blending.

### 6. Shader Variant System
All implementations compile shader variants based on feature flags:
- Flags like `HAS_COLOR_TEXTURE`, `HAS_VERTEX_COLORS`, `HAS_MATERIAL_INDEX`
- Variants cached by hash/key
- No runtime branching in shaders

## Key Variations

### Material Index Palette Storage

**Uniform Buffer Array (Most Common):**
- **caracal, fennec, hyena, lynx, mantis, rabbit, shark, wren**: Material palette stored in UBO as array of structs
- Max entries typically 16-64 per material

**Texture Lookup (Overflow Strategy):**
- **fennec, hyena**: When palette exceeds UBO limit, fall back to 1D texture lookup
- **lynx**: Explicit support for >64 entries via texture

**Not Supported:**
- **bonobo**: No material index system in v1

### Material Index Palette Structure

**Separate Color and Emissive Arrays (4 implementations):**
```glsl
uniform vec3 u_materialColors[32];
uniform vec4 u_materialEmissive[32];  // rgb + intensity in w
```
- **caracal, fennec, rabbit, shark**

**Combined Struct Array (4 implementations):**
```glsl
struct MaterialEntry {
  vec4 colorAndPad;           // rgb + padding
  vec4 emissiveAndIntensity;  // rgb + intensity
}
uniform MaterialEntry u_palette[32];
```
- **hyena, lynx, mantis, wren**

### Maximum Material Indices Per Palette

| Implementation | Max Indices | Overflow Strategy |
|----------------|-------------|------------------|
| **caracal** | 16 | None documented |
| **fennec** | 32 | Texture lookup |
| **hyena** | 32 | Texture lookup |
| **lynx** | 64 | Texture lookup |
| **mantis** | 32 | Texture lookup (rare) |
| **rabbit** | 32 | None documented |
| **shark** | 32 | None documented |
| **wren** | 16 | None documented |

### Shader Variant Compilation Strategy

**Lazy + Cached (Most Common):**
- **caracal, fennec, hyena, lynx, mantis, rabbit, shark, wren**: Compile on first use, cache by feature hash
- Typical scene uses 10-30 unique variants

**Eager Compilation:**
- **caracal**: "Unlike Three.js (which compiles shaders on first render, causing frame spikes), Caracal compiles eagerly"
- Compile immediately when material is created

### Emissive + Bloom Integration

**Dual Render Target (MRT):**
- **hyena, rabbit, shark, wren**: Write emissive to separate RT1 during opaque pass
- Bloom extraction reads RT1 directly

**Alpha Channel Encoding:**
- **caracal**: Emissive intensity in alpha channel of main color output
- Bloom extraction reads alpha to determine bloom strength

**Additive After Lighting:**
- **fennec, lynx, mantis**: Add emissive to final color in fragment shader
- Bloom extracts based on luminance threshold

### Material Property Storage

**Per-Material Uniform Block:**
- **All except bonobo**: Material properties in a dedicated UBO (bind group 1)

**Per-Entity Color:**
- **bonobo**: Color is per-entity (in instance buffer), not per-material
- Enables better instancing (500 red + 500 blue boxes = 1 draw call)

### Material Sorting Key

**64-bit Sort Key (Most Common):**
```
Bits 60-63: Pipeline ID
Bits 48-59: Material ID
Bits 0-47:  Depth (front-to-back for opaque)
```
- **lynx, rabbit, shark, wren**: Explicit 64-bit key

**32-bit or Implicit:**
- **caracal, fennec, hyena, mantis**: Sort by pipeline hash, then material, then depth
- Similar concept, different implementation

**No Sorting:**
- **bonobo v1**: Flat entity iteration (future v2 will add sorting)

## Implementation Breakdown by Palette Size

### Large Palette (64 entries)
**Count: 1**
- **lynx**: 64 entries × 32 bytes = 2048 bytes UBO

**Best For:** Complex models with many material zones (architectural visualization, detailed characters)

### Medium Palette (32 entries)
**Count: 5**
- **fennec, hyena, mantis, rabbit, shark**: 32 entries × 16-32 bytes = 512-1024 bytes

**Best For:** Typical game assets (characters, props, vehicles)

### Small Palette (16 entries)
**Count: 2**
- **caracal, wren**: 16 entries

**Best For:** Simpler stylized models, lower memory usage

### No Palette
**Count: 1**
- **bonobo**: Color per entity instead

## Shader Feature Flags Comparison

| Feature Flag | Triggered By | Support |
|--------------|-------------|---------|
| `USE_MAP` / `HAS_COLOR_TEXTURE` | `material.map !== null` | All 9 |
| `USE_AO_MAP` / `HAS_AO_TEXTURE` | `material.aoMap !== null` | 8 (not bonobo) |
| `USE_VERTEX_COLORS` / `HAS_VERTEX_COLORS` | `material.vertexColors === true` | All 9 |
| `USE_MATERIAL_INDEX` / `HAS_MATERIAL_INDEX` | Geometry has `_materialindex` attribute | 8 (not bonobo) |
| `USE_SKINNING` / `HAS_SKINNING` | Mesh is SkinnedMesh | All 9 |
| `USE_EMISSIVE` / `HAS_EMISSIVE` | `emissiveIntensity > 0` | 8 (not bonobo) |
| `SHADOW_RECEIVE` / `HAS_SHADOWS` | `receiveShadow === true` | 8 (not bonobo) |
| `IS_TRANSPARENT` | `transparent === true` | All 9 |

## Transparency Handling

**Weighted Blended OIT:**
- **All 9 implementations** use WBOIT for transparent materials
- Transparent materials set `depthWrite: false`
- Rendered in separate pass
- Order-independent (no sorting required)

**Opacity Source:**
- Material `opacity` property (uniform)
- Texture alpha channel (if present)
- Vertex color alpha (if vertex colors enabled)
- Material index palette entry opacity (some implementations)

## Material State and Pipeline Compilation

### Pipeline State Contributions

**Depth State:**
- **Opaque:** `depthWrite: true`, `depthTest: true`
- **Transparent:** `depthWrite: false`, `depthTest: true` (read-only)

**Blend State:**
- **Opaque:** None (replace)
- **Transparent:** OIT accumulation/revealage blend modes

**Cull State:**
- **Front (default):** Cull back faces
- **Back:** Cull front faces
- **Both/Double:** No culling

### Pipeline Caching Strategy

**Hash-Based Cache:**
- **All implementations** use a hash of (shader defines + blend state + depth state + cull mode)
- Map<hash, pipeline> lookup

**Cache Size:**
- Typical scene: 10-30 unique pipelines
- Worst case: ~2048 possible combinations (rarely all used)

## Bind Group Layout Comparison

All implementations follow a similar bind group frequency hierarchy:

### Bind Group 0: Per-Frame
- Camera matrices (view, projection)
- Light data (directional, ambient)
- Shadow maps
- Time, camera position

### Bind Group 1: Per-Material
- Material UBO (color, opacity, flags)
- Material palette (if using material index)
- Textures (color map, AO map)
- Samplers

### Bind Group 2: Per-Object
- World matrix
- Bone matrices (for skinned meshes)
- Object-specific data

**Exception:** bonobo uses instance buffers instead of per-object bind groups

## Texture Handling

### Texture Formats

**Uncompressed:**
- **All implementations**: RGBA8 for color/AO maps

**Compressed:**
- **caracal, fennec, lynx, rabbit, wren**: BC7 (desktop), ASTC (mobile), ETC2 (fallback)
- Via KTX2/Basis Universal transcoding

### Texture Binding

**Combined Texture + Sampler:**
- **WebGL2 implementations**: `uniform sampler2D u_colorMap`

**Separate Texture and Sampler:**
- **WebGPU implementations**: `@group(1) @binding(2) var t_colorMap: texture_2d<f32>` + separate sampler

### Default Textures

**1×1 White Pixel:**
- **mantis, rabbit, shark**: Bind 1×1 white default texture when no texture provided
- Avoids shader branching

**Conditional Binding:**
- **caracal, fennec, hyena, lynx, wren**: Only bind texture when present (shader variants handle)

## Material Index Shader Implementation Comparison

### GLSL (WebGL2) Approach

**Fennec Example:**
```glsl
layout(std140) uniform MaterialPalette {
  MaterialEntry entries[32];
} palette;

in float a_materialIndex;
flat out int v_materialIndex;  // Integer in fragment shader

void main() {
  v_materialIndex = int(a_materialIndex);
  // ...
}

// Fragment shader:
vec3 baseColor = palette.entries[v_materialIndex].colorAndPad.rgb;
```

**Caracal/Shark Example:**
```glsl
uniform vec3 u_matIdxColors[16];
uniform vec3 u_matIdxEmissive[16];

flat in uint v_materialIndex;  // Unsigned int

// Fragment:
vec3 baseColor = u_matIdxColors[v_materialIndex];
```

### WGSL (WebGPU) Approach

**Lynx Example:**
```wgsl
struct PaletteEntry {
  color_and_pad: vec4<f32>,
  emissive_and_intensity: vec4<f32>,
}

@group(1) @binding(0) var<uniform> palette: array<PaletteEntry, 64>;

@location(6) @interpolate(flat) material_index: u32

// Fragment:
let entry = palette[material_index];
```

## Why Only Basic + Lambert? (No PBR)

All 9 implementations explicitly chose NOT to include PBR materials by default:

### Stated Rationale:

**Performance (Mobile):**
- Lambert: ~15 ALU ops per fragment
- PBR (Cook-Torrance): ~80+ ALU ops
- 3-5× slower on mobile GPUs

**Texture Requirements:**
- Lambert: color + AO (2 textures)
- PBR: albedo + metallic + roughness + normal + AO (5 textures)

**Visual Quality for Target Use Case:**
- Stylized/low-poly games: Lambert provides "excellent" visual quality
- PBR: "Marginal improvement" for non-photorealistic art

**Complexity:**
- Lambert is simple, well-understood
- PBR requires more parameters, harder to art-direct

### Future PBR Support:
- **mantis**: "If PBR is needed in the future, it can be added as a third material type without changing the existing architecture"
- **caracal**: No PBR mentioned
- Others: Similar stance—defer PBR until needed

## Special Features by Implementation

### Caracal
- Emissive intensity in alpha channel (unique approach)
- Eager shader compilation (avoid hitching)
- Material index colors with bloom integration

### Fennec
- Material palette max 32 entries
- Overflow to texture lookup for large palettes
- Explicit production vs. debug fast paths

### Hyena
- Material index map as primary feature
- Transparency via OIT with no sorting
- Simple two-material philosophy

### Lynx
- Largest palette (64 entries)
- Most comprehensive material documentation
- Explicit arc-length UV parameterization for capsules

### Mantis
- SoA material palette storage
- Per-index opacity support
- Feature key as bitmask for variant selection

### Rabbit
- Material index entries with emissive per-index
- Bind group layout optimized for update frequency
- MRT for emissive output

### Shark
- Material index colors array
- Emissive MRT approach
- Default 1×1 textures to avoid branching

### Wren
- Material index with emissive intensity per-entry
- Dual-output fragment shader for bloom
- Separate buffer storage for non-float attributes

### Bonobo
- **Unique:** Per-entity color instead of per-material
- Enables better automatic instancing
- No material index system (not needed for flat architecture)

## Decision Matrix: Material System Features

### Choose Material Index System (8/9 implementations) if:
- Multi-material models common (characters, vehicles, buildings)
- Single draw call per model is critical
- Artists export single meshes with material zones
- Bloom/emissive needs per-vertex control

### Choose Per-Entity Color (Bonobo-style) if:
- Maximum instancing opportunities
- Color variation across instances more important than per-vertex
- Simpler material model acceptable
- Targeting very high entity counts

### Choose Large Palette (64 entries, Lynx-style) if:
- Complex architectural models
- Detailed character customization
- Memory budget allows

### Choose Small Palette (16 entries, Caracal/Wren-style) if:
- Mobile/memory-constrained
- Simpler stylized models
- Lower UBO size matters

## Material Index + Vertex Color Interaction Matrix

| Implementation | Order | Result |
|----------------|-------|--------|
| **All 8 with material index** | `paletteColor × vertexColor × textureColor` | Vertex colors modulate palette color |
| **All 9 without material index** | `materialColor × vertexColor × textureColor` | Standard multiplicative combine |

## Performance Characteristics Summary

| Implementation | Shader Variants (Typical) | Material Index Max | Palette UBO Size | Compilation Strategy |
|----------------|--------------------------|-------------------|------------------|---------------------|
| **bonobo** | 3 (static/skinned/textured) | N/A | N/A | Immediate |
| **caracal** | 10-20 | 16 | 256 bytes | Eager |
| **fennec** | 10-20 | 32 | 512 bytes | Lazy |
| **hyena** | 10-30 | 32 | 1024 bytes | Lazy |
| **lynx** | 10-30 | 64 | 2048 bytes | Lazy |
| **mantis** | 10-20 | 32 | 1024 bytes | Lazy |
| **rabbit** | 10-30 | 32 | 512 bytes | Lazy |
| **shark** | 10-30 | 32 | 1024 bytes | Lazy |
| **wren** | 10-30 | 16 | 512 bytes | Lazy |

## Unique Innovations Worth Cherry-Picking

### From Caracal:
- Emissive in alpha channel (single RT instead of MRT)
- Eager compilation to avoid hitching

### From Hyena:
- Material index as primary feature (not an add-on)
- Clear documentation of why PBR is excluded

### From Lynx:
- 64-entry palette (good for complex models)
- Comprehensive material documentation

### From Mantis:
- SoA approach even for material data
- Feature key bitmask (clean variant selection)

### From Bonobo:
- Per-entity color for maximum instancing
- Deliberate simplicity (3 pipelines only)

### From Fennec/Rabbit/Shark/Wren:
- MRT emissive output (cleaner bloom integration)
- Standard 32-entry palette (good balance)
