# ASSETS.md - Decisions

## Decision: Worker Pool, Vertex Conversion for Y-up, URL-Based Cache

### Primary Format: glTF 2.0

Universal agreement across all 9 implementations. Support both `.gltf` + `.bin` and `.glb` binary formats.

### Compression Support

- **Draco mesh compression** (`KHR_draco_mesh_compression`) via WASM decoder in Web Worker
- **KTX2/Basis Universal texture compression** (`KHR_texture_basisu`) via WASM transcoder in Web Worker
- Both decoders loaded lazily on first use

### Decoding Architecture: Worker Pool

**Chosen**: Worker pool scaled to `navigator.hardwareConcurrency` (4/9: Fennec, Hyena, Lynx, Mantis)
**Rejected**: Single worker (4/9: Caracal, Rabbit, Shark, Wren) - adequate for simple models but 2-3x slower for complex models with 10+ Draco-compressed meshes

```typescript
const workerCount = Math.min(navigator.hardwareConcurrency || 2, 4)
const dracoPool = createWorkerPool(workerCount, dracoWorkerUrl)
const basisPool = createWorkerPool(workerCount, basisWorkerUrl)
```

Communication via `postMessage` with `Transferable` typed arrays for zero-copy data transfer.

For a 5MB glTF with 50 meshes and 20 textures:
- Single worker: ~850ms total
- Worker pool (4 workers): ~420ms total

### KTX2 Target Format Priority

Universal agreement on priority order:

1. **ASTC 4x4** - iOS, modern Android, Apple Silicon
2. **BC7** - Windows, Linux, Intel Macs
3. **ETC2** - older Android, WebGL2 fallback
4. **RGBA8** - uncompressed fallback

Format probing via `GPUAdapter.features` (WebGPU) or WebGL extension queries.

### Material Mapping: PBR to Lambert

**Chosen**: Simple mapping (7/9 implementations)

```
glTF baseColorFactor    -> LambertMaterial.color
glTF baseColorTexture   -> LambertMaterial.map
glTF occlusionTexture   -> LambertMaterial.aoMap
glTF emissiveFactor     -> Material.emissive
glTF alphaMode: BLEND   -> material.transparent = true
glTF alphaMode: MASK    -> alpha cutoff discard
glTF doubleSided        -> material.doubleSided = true
KHR_materials_unlit     -> BasicMaterial
```

Metallic/roughness textures and normal maps are intentionally ignored - not needed for the target stylized aesthetic.

### Y-up to Z-up Conversion: Vertex Data Conversion

**Chosen**: Convert vertex positions, normals, and animation keyframes directly at load time (4/9: Lynx, Mantis, Shark, Wren)
**Rejected**: Root node rotation (4/9: Caracal, Fennec, Hyena, Rabbit) - simpler to implement but leaves animation data in glTF space, which can cause confusion

Vertex conversion bakes the coordinate change into the geometry data itself:

```typescript
// For each vertex position/normal:
// glTF: [x, y, z] where Y is up
// Engine: [x, z, -y] where Z is up (swap Y/Z, negate new Y)
const convertYUpToZUp = (positions: Float32Array) => {
  for (let i = 0; i < positions.length; i += 3) {
    const y = positions[i + 1]
    const z = positions[i + 2]
    positions[i + 1] = -z
    positions[i + 2] = y
  }
}
```

Same conversion applied to animation keyframe positions and bone transforms. After loading, everything is in engine space with no runtime coordinate mixing.

### Custom Attributes

Universal agreement: recognize `_materialindex` as a standard custom attribute. The Draco decoder preserves custom attributes (all implementations handle this).

### Return Type

```typescript
interface GLTFResult {
  scene: Group                    // Root node (fully converted to Z-up)
  meshes: Mesh[]                  // Flat list of all meshes
  skinnedMeshes: SkinnedMesh[]    // Flat list of skinned meshes
  animations: AnimationClip[]     // All animation clips
  skeletons: Skeleton[]           // All skeletons
  dispose: () => void             // Cleanup all GPU resources
}
```

### Asset Caching: URL-Based

**Chosen**: URL-based promise cache (6/9: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit)

```typescript
const loadCache = new Map<string, Promise<GLTFResult>>()

const loadGLTF = (url: string, options?: LoadOptions): Promise<GLTFResult> => {
  if (!loadCache.has(url)) {
    loadCache.set(url, loadGLTFInternal(url, options))
  }
  return loadCache.get(url)!
}
```

Same URL returns same Promise, preventing duplicate loads. Geometry and textures are deduplicated within a single glTF file (shared GPU buffers).

### Disposal

Explicit disposal required (universal agreement):

```typescript
gltfResult.dispose() // Destroys all GPU resources (buffers, textures, pipelines)
```

React bindings auto-dispose on unmount (see REACT.md).

### Decoder Configuration

Allow custom WASM URLs for CDN hosting (Caracal/Fennec/Lynx approach):

```typescript
const gltf = await loadGLTF(url, {
  dracoDecoderUrl: '/wasm/draco/',
  basisTranscoderUrl: '/wasm/basis/',
})
```

This prevents bundling ~500KB of WASM into the application bundle.

### Mipmap Generation

- **KTX2 with pre-computed mipmaps**: Upload all levels directly
- **Standard images (PNG/JPEG/WebP)**: `createImageBitmap()` for off-main-thread decode, then `gl.generateMipmap()` (WebGL2) or compute shader (WebGPU)

### Known Limitations (v1)

- **Morph targets**: Not supported (rare in stylized games)
- **Cameras from glTF**: Ignored (engine cameras used instead)
- **Lights from glTF**: Ignored (`KHR_lights_punctual` not supported)
- **PBR materials**: Converted to Lambert (lossy but intentional)
