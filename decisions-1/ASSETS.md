# ASSETS - Design Decisions

## Primary Format

**Decision: glTF 2.0 as the primary 3D model format (universal agreement)**

- Full support for .gltf (JSON + .bin) and .glb (binary)
- Meshes, materials, textures, hierarchies, animations, skeletons
- Y-up to Z-up conversion at load time

## Compression Support

**Decision: Draco mesh compression + KTX2/Basis Universal texture compression**

- Sources: All 8 active implementations agree
- `KHR_draco_mesh_compression` for mesh data
- `KHR_texture_basisu` for texture data
- Both decoders run in Web Workers via WASM modules

## Worker Strategy

**Decision: Worker pool (2-4 workers based on hardwareConcurrency)**

- Sources: Fennec, Hyena, Lynx, Mantis (4/8 use worker pools)
- Rejected: Single worker (Caracal, Rabbit, Shark, Wren) — slower for complex models

```typescript
const workerCount = Math.min(navigator.hardwareConcurrency || 2, 4)
const dracoPool = createWorkerPool('draco-decoder.wasm', workerCount)
const basisPool = createWorkerPool('basis-transcoder.wasm', workerCount)
```

For models with 10+ Draco-compressed meshes, worker pool is 2-3× faster than single worker. Minimal memory overhead (~2MB per worker for WASM instance).

Communication via `postMessage` with `Transferable` typed arrays for zero-copy transfer.

## WASM Module Loading

**Decision: Lazy load on first use, configurable URLs for CDN hosting**

- Sources: Caracal, Fennec, Lynx (configurable WASM URLs)

```typescript
const engine = await createEngine(canvas, {
  draco: { wasmUrl: '/wasm/draco_decoder.wasm' },
  basis: { wasmUrl: '/wasm/basis_transcoder.wasm' },
})
```

Reduces initial bundle by ~500KB. WASM modules loaded and cached on first glTF load that requires them.

## Texture Format Priority

**Decision: ASTC > BC7 > ETC2 > RGBA8 (universal agreement)**

1. **ASTC 4×4** — mobile (iOS, modern Android, Apple Silicon)
2. **BC7** — desktop (Windows, Linux, Intel Macs)
3. **ETC2** — older Android, WebGL2 fallback
4. **RGBA8** — uncompressed fallback

Format detected at startup by querying device capabilities (`GPUAdapter.features` or WebGL extensions). 4-8× texture memory savings with compressed formats.

## Material Mapping

**Decision: PBR → Lambert simple mapping + KHR_materials_unlit → BasicMaterial**

- Sources: 7/8 active implementations use simple PBR→Lambert mapping; 6/8 support unlit extension

```typescript
// PBR to Lambert mapping:
material.color = gltfMaterial.pbrMetallicRoughness.baseColorFactor
material.map = gltfMaterial.pbrMetallicRoughness.baseColorTexture
material.aoMap = gltfMaterial.occlusionTexture
material.emissive = gltfMaterial.emissiveFactor

// Alpha mode:
if (gltfMaterial.alphaMode === 'BLEND') material.transparent = true
if (gltfMaterial.alphaMode === 'MASK') material.alphaTest = gltfMaterial.alphaCutoff
if (gltfMaterial.doubleSided) material.doubleSided = true

// Unlit extension:
if (gltfMaterial.extensions?.KHR_materials_unlit) → BasicMaterial
```

Metallic/roughness textures and normal maps discarded — intentional for the stylized low-poly target.

## Y-up to Z-up Conversion

**Decision: Vertex data conversion (fully baked, no runtime overhead)**

- Sources: Lynx, Mantis, Shark, Wren (4/8)
- Rejected: Root node rotation (Caracal, Fennec, Hyena, Rabbit) — leaves animation data in glTF space, can cause confusion

Convert vertex positions, normals, and animation keyframes directly at load time:
- Swap Y↔Z components, negate as needed for handedness
- Baked into geometry data — no runtime coordinate space mixing
- No extra root rotation node in the scene graph

## Asset Caching

**Decision: URL-based cache using Map<string, Promise<GLTFResult>>**

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit (6/8)

```typescript
const loadCache = new Map<string, Promise<GLTFResult>>()

const loadGLTF = async (url: string, options?: LoadOptions): Promise<GLTFResult> => {
  if (!loadCache.has(url)) {
    loadCache.set(url, loadGLTFInternal(url, options))
  }
  return loadCache.get(url)!
}
```

Same URL returns same Promise, preventing duplicate loads. Geometry and texture GPU resources are shared across multiple meshes referencing the same data.

## Return Type

**Decision: Flat lists + scene hierarchy**

```typescript
interface GLTFResult {
  scene: Group                    // Root node with full hierarchy
  scenes: Group[]                 // All scenes (glTF supports multiple)
  meshes: Mesh[]                  // Flat list of all meshes
  skinnedMeshes: SkinnedMesh[]    // Flat list of skinned meshes
  animations: AnimationClip[]     // Flat list of animation clips
  skeletons: Skeleton[]           // Flat list of skeletons
  dispose: () => void             // Dispose all GPU resources
}
```

## Custom Attributes

**Decision: Recognize `_materialindex` as standard, preserve all custom attributes**

- Sources: All implementations recognize `_materialindex`
- Draco decoder preserves custom attributes
- Custom attributes accessible on geometry objects after load

## Disposal

**Decision: Explicit disposal required; React bindings auto-dispose on unmount**

```typescript
// Imperative usage: explicit disposal
const gltf = await loadGLTF('/model.glb')
// ... later ...
gltf.dispose()  // frees all GPU resources

// React usage: automatic disposal on unmount
const Model = ({ url }) => {
  const gltf = useGLTF(url)
  return <primitive object={gltf.scene} />
  // GPU resources disposed when component unmounts
}
```

## Mipmap Generation

**Decision: Pre-computed mipmaps from KTX2, runtime generation for standard images**

- KTX2 textures: upload all mip levels provided in the file
- Standard images (PNG, JPEG): WebGPU compute shader (Fennec, Lynx, Mantis, Wren) or `gl.generateMipmap()` on WebGL2
- Decode standard images off-main-thread via `createImageBitmap()`

## Error Handling

**Decision: Graceful fallback with clear error messages**

- Network failures: reject Promise with descriptive error
- Missing Draco/Basis WASM: fall back to uncompressed data if available, warn
- Invalid glTF: reject with parsing error details
- Missing textures: use 1×1 white fallback, warn
