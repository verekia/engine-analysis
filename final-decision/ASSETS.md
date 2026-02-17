# ASSETS — Final Decision

## Primary Format

**Decision: glTF 2.0 as the only 3D format** (universal agreement)

Supports both `.gltf` (JSON + separate buffers) and `.glb` (binary container).

## Mesh Compression

**Decision: Draco mesh compression via Web Worker** (universal agreement)

```typescript
const gltf = await loadGLTF('/model.glb', engine, {
  draco: { wasmUrl: '/wasm/draco_decoder.wasm' },
})
```

- WASM decoder loaded lazily on first use
- User provides WASM URL (no bundled WASM)
- Draco decoding runs in a Web Worker to avoid blocking the main thread
- Decoded vertex data transferred back via `Transferable` (zero-copy)

## Texture Compression

**Decision: KTX2/Basis Universal via Web Worker** (universal agreement)

```typescript
const gltf = await loadGLTF('/model.glb', engine, {
  ktx2: { wasmUrl: '/wasm/basis_transcoder.wasm' },
})
```

### Format Priority

Detect GPU-supported formats at initialization and transcode to the best available:

```
ASTC (mobile: A-series, Adreno, Mali)
  > BC7 (desktop: AMD, NVIDIA, Intel)
    > ETC2 (fallback mobile)
      > RGBA8 (universal fallback)
```

- Standard images (PNG, JPEG, WebP) loaded via `createImageBitmap`
- Pre-computed mipmaps from KTX2; runtime `generateMipmap` for standard images

## Worker Pool

**Decision: 2-4 workers scaled to `navigator.hardwareConcurrency`** (universal agreement)

```typescript
const workerCount = Math.min(4, Math.max(2, navigator.hardwareConcurrency - 1))
```

Workers handle Draco decoding and Basis transcoding in parallel. Large models with multiple mesh primitives benefit from concurrent decoding.

Performance: ~2× speedup with 4-worker pool on complex models.

## Material Mapping

**Decision: PBR to Lambert mapping** (universal agreement)

glTF materials use PBR (metallic-roughness). Since the engine only supports Basic and Lambert:

- `baseColorFactor` → Lambert `color`
- `baseColorTexture` → Lambert `colorTexture`
- `emissiveFactor` → Lambert `emissive`
- Metallic, roughness, normal maps → ignored
- `KHR_materials_unlit` → BasicMaterial

This is intentional: the engine targets stylized/low-poly aesthetics where PBR properties beyond diffuse color aren't needed.

## Coordinate Conversion

**Decision: Y-up to Z-up conversion baked at load time** (universal agreement)

glTF uses Y-up, right-handed. The engine uses Z-up, right-handed. Conversion is applied to all vertex data at load time:

```typescript
// For every vertex position and normal:
const [x, y, z] = glTFVertex
const engineVertex = [x, z, -y]  // Swap Y↔Z, negate new Y
```

Also applied to:
- Animation keyframe positions and rotations
- Bone rest poses and inverse bind matrices
- Scene node transforms (translation, rotation)

No per-frame conversion overhead — everything is baked once during loading.

## Custom Attribute: _materialindex

**Decision: Recognize `_materialindex` as a standard custom attribute** (universal agreement)

The loader detects the `_MATERIALINDEX` vertex attribute (glTF custom attribute convention) and maps it to the engine's material index system automatically.

## Return Type

```typescript
interface GLTFResult {
  scene: Group              // Root node of the glTF scene hierarchy
  scenes: Group[]           // All scenes in the file
  meshes: Mesh[]            // Flat list of all meshes
  skeletons: Skeleton[]     // Flat list of all skeletons
  animations: AnimationClip[] // Flat list of all animation clips
  textures: Texture[]       // Flat list of all textures
  dispose(): void           // Release all GPU resources
}
```

Both the scene hierarchy and flat lists are provided: hierarchy for adding to the scene graph, flat lists for direct access.

## Caching

**Decision: URL-based Promise caching** (universal agreement)

```typescript
const cache = new Map<string, Promise<GLTFResult>>()

const loadGLTF = async (url: string, engine: Engine, options?: LoadOptions): Promise<GLTFResult> => {
  let promise = cache.get(url)
  if (!promise) {
    promise = loadGLTFImpl(url, engine, options)
    cache.set(url, promise)
  }
  return promise
}
```

Subsequent calls to `loadGLTF` with the same URL return the cached Promise. The returned `GLTFResult` can be cloned (new scene graph instances sharing the same GPU resources) for multiple instances of the same model.

## Disposal

```typescript
gltf.dispose()  // Release all GPU resources (textures, buffers)
```

React bindings handle disposal automatically on component unmount.
