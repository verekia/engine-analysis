# Materials

## Material System

Materials in Bonobo are simple and pipeline-driven. Each material type corresponds to a different rendering pipeline.

## Material Types

Start simple. Three material types matching three pipelines:

```ts
interface StaticMaterial {
  type: 'static'
  id: number
}

interface SkinnedMaterial {
  type: 'skinned'
  id: number
  // Joint matrices provided per-entity at draw time
}

interface TexturedMaterial {
  type: 'textured'
  id: number
  texture: GPUTexture
  sampler: GPUSampler
}
```

Material = pipeline selection + any extra bindings (textures, etc.).

## Per-Entity vs Per-Material Properties

### Per-Entity Properties

Color is **per-entity**, not per-material. This enables better instancing:

- 500 red boxes and 500 blue boxes with the same geometry can still instance into 1 draw call
- Color is stored in the instance buffer, not the material

```
Instance buffer layout (per instance):
worldMatrix:  mat4x4<f32>   (64 bytes)
color:        vec4<f32>     (16 bytes)
flags:        u32           (4 bytes)
padding:      12 bytes      (align to 96)
                            ─────────────
                            96 bytes/instance
```

### Per-Material Properties

Material defines:
- Pipeline to use (static, skinned, textured)
- Shared textures/samplers (for textured materials)
- Any other shader-specific bindings

## Material Registry

Materials are stored in a registry:

```ts
// Inside World class
materialRegistry: Map<materialId, MaterialDesc>
```

Entities reference materials by ID:

```ts
materialIds: Uint32Array(MAX_ENTITIES)
```

## Vertex Colors

Vertex colors are supported as part of the geometry vertex format:

```ts
interface Geometry {
  vertexBuffer: GPUBuffer  // interleaved [pos, normal, color, ...]
  // ...
}
```

Vertex attributes:
- Position: `vec3<f32>`
- Normal: `vec3<f32>`
- Color: `vec4<f32>` (optional, per-vertex)
- UV: `vec2<f32>` (for textured materials)

## Emissive / Unlit

Per-entity unlit flag for UI elements, skyboxes, emissive objects:

```ts
// Set via entity flags
entity.setFlag(UNLIT_FLAG)
```

In the shader, skip lighting calculations if the unlit flag is set.

## Future Extensions (v2)

- PBR materials (metallic, roughness, normal maps)
- Material property blocks (per-material uniform buffers)
- Emissive per material index
- Lambert vs Phong vs PBR shader variants
