# Assets

## Status: Minimal Support for v1

Asset loading is kept minimal in v1. Import meshes as raw vertex/index arrays. A glTF adapter can be a separate package.

## Custom Geometry Import

Generic geometry creation from raw vertex/index data:

```ts
function createGeometry(
  vertices: Float32Array,   // interleaved vertex data
  indices: Uint16Array | Uint32Array
): Geometry
```

This enables importing meshes from external sources by pre-processing them into the expected format.

## Vertex Data Format

Expected interleaved vertex layout:

```
Position:  vec3<f32>  (12 bytes)
Normal:    vec3<f32>  (12 bytes)
Color:     vec4<f32>  (16 bytes, optional)
UV:        vec2<f32>  (8 bytes, for textured materials)
```

## Future: glTF Loader (v2)

A separate package or module for glTF 2.0 support:

### Features

- Load meshes, materials, textures from .gltf/.glb files
- Support for Draco mesh compression
- Support for KTX2 / Basis texture compression
- Animation clips (once skeletal animation is implemented)
- Scene hierarchy (once scene graph is implemented)

### glTF Extensions

Planned support for:
- `KHR_draco_mesh_compression` — Draco decoding
- `KHR_texture_basisu` — Basis Universal textures
- `KHR_mesh_quantization` — Quantized vertex attributes
- `KHR_materials_unlit` — Unlit materials

### Implementation Strategy

The glTF loader would:
1. Parse the glTF JSON structure
2. Decompress Draco meshes (using wasm-draco or similar)
3. Decode KTX2/Basis textures (using basis_universal.js)
4. Convert to Bonobo's internal formats (Geometry, Material, Texture)
5. Upload to GPU

```ts
async function loadGLTF(url: string): Promise<GLTFScene> {
  // Parse glTF
  // Decode compressed assets
  // Create geometries + materials
  // Return scene descriptor
}
```

## Texture Loading

For v1, textures can be loaded manually:

```ts
async function loadTexture(url: string): Promise<GPUTexture> {
  const img = await loadImage(url)
  const texture = device.createTexture({
    size: [img.width, img.height],
    format: 'rgba8unorm',
    usage: GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST
  })
  device.queue.copyExternalImageToTexture(
    { source: img },
    { texture },
    [img.width, img.height]
  )
  return texture
}
```

## Asset Management (Future)

In v2, add:
- Asset registry (track loaded assets, prevent duplicates)
- Reference counting (unload unused assets)
- Streaming / LOD (load high-res assets on demand)
- Asset bundles (pack multiple assets into single file)

## Workarounds for v1

For v1, assets can be:
- Pre-converted to the expected format offline
- Loaded as raw TypedArrays from JSON/binary files
- Generated procedurally using the built-in primitives
