# Voidcore — Assets

## Primary Format

glTF 2.0 as the only 3D asset format. Both `.gltf` (JSON + separate binary buffers) and `.glb` (single binary container) are supported.

glTF is the industry standard for web 3D. Supporting only one format keeps the loader focused and well-tested.

## Loading API

```typescript
const gltf = await loadGLTF('/model.glb', engine, {
  draco: { wasmUrl: '/wasm/draco_decoder.wasm' },
  ktx2: { wasmUrl: '/wasm/basis_transcoder.wasm' },
})

// Access loaded data
scene.add(gltf.scene)                           // Add the scene hierarchy
const mesh = gltf.meshes[0]                      // Direct access to meshes
const skeleton = gltf.skeletons[0]               // Direct access to skeletons
const clips = gltf.animations                    // Animation clips

// Cleanup
gltf.dispose()                                   // Release all GPU resources
```

### Return Type

```typescript
interface GLTFResult {
  scene: Group                    // Root node of the glTF scene hierarchy
  scenes: Group[]                 // All scenes in the file (glTF supports multiple)
  meshes: Mesh[]                  // Flat list of all meshes
  skeletons: Skeleton[]           // Flat list of all skeletons
  animations: AnimationClip[]     // Flat list of all animation clips
  textures: Texture[]             // Flat list of all textures
  dispose(): void                 // Release all GPU resources
}
```

Both the scene hierarchy (for adding to the scene graph) and flat lists (for direct access) are provided.

## Mesh Compression: Draco

WASM-based Draco decoder running in a Web Worker:

```typescript
const gltf = await loadGLTF('/model.glb', engine, {
  draco: { wasmUrl: '/wasm/draco_decoder.wasm' },
})
```

### How It Works

1. glTF loader detects `KHR_draco_mesh_compression` extension on a mesh primitive
2. Compressed buffer view is sent to a Web Worker via `postMessage`
3. Worker loads the Draco WASM decoder (lazily, on first use) and decodes the mesh
4. Decoded vertex data (`Float32Array` for positions/normals/UVs, `Uint8Array` for material indices) is transferred back via `Transferable` (zero-copy)
5. Main thread creates geometry from the decoded arrays

### WASM Loading

- The user provides the WASM URL — **no bundled WASM** in the package
- WASM is loaded lazily on first use (not at engine init)
- The decoder instance is cached in the worker after first load
- If no `draco` option is provided and a model uses Draco compression, the loader throws an error with a helpful message

## Texture Compression: KTX2/Basis Universal

WASM-based Basis Universal transcoder running in a Web Worker:

```typescript
const gltf = await loadGLTF('/model.glb', engine, {
  ktx2: { wasmUrl: '/wasm/basis_transcoder.wasm' },
})
```

### Format Priority

At engine initialization, detect GPU-supported compressed texture formats and transcode to the best available:

```
Priority (highest → lowest):
  1. ASTC   — mobile (Apple A-series, Qualcomm Adreno, ARM Mali)
  2. BC7    — desktop (AMD, NVIDIA, Intel)
  3. ETC2   — fallback mobile
  4. RGBA8  — universal fallback (uncompressed)
```

Detection via `device.features` (WebGPU) or `getExtension` (WebGL2):

```typescript
// WebGL2
const hasASTC = !!gl.getExtension('WEBGL_compressed_texture_astc')
const hasBC7 = !!gl.getExtension('EXT_texture_compression_bptc')
const hasETC2 = true  // Required by WebGL2 spec

// WebGPU
const hasASTC = device.features.has('texture-compression-astc')
const hasBC7 = device.features.has('texture-compression-bc')
const hasETC2 = device.features.has('texture-compression-etc2')
```

### Standard Images

PNG, JPEG, and WebP loaded via `createImageBitmap` (off-main-thread decoding where supported). Runtime `generateMipmap` for standard images. KTX2 files include pre-computed mipmaps.

## Worker Pool

2-4 workers scaled to hardware concurrency:

```typescript
const workerCount = Math.min(4, Math.max(2, navigator.hardwareConcurrency - 1))
```

Workers handle both Draco decoding and Basis transcoding. Large models with multiple mesh primitives benefit from concurrent decoding across workers.

### Worker Communication

```typescript
// Main thread → Worker
worker.postMessage({
  type: 'draco-decode',
  buffer: compressedArrayBuffer,    // Transferable
  attributes: ['position', 'normal', 'uv', '_materialindex'],
}, [compressedArrayBuffer])

// Worker → Main thread
self.postMessage({
  type: 'draco-result',
  positions: positionsBuffer,       // Transferable
  normals: normalsBuffer,
  uvs: uvsBuffer,
  indices: indicesBuffer,
  materialIndices: materialIndicesBuffer,
}, [positionsBuffer, normalsBuffer, uvsBuffer, indicesBuffer, materialIndicesBuffer])
```

All data transferred via `Transferable` for zero-copy transfer. The sending side loses access to the buffer (ownership transfer).

**Performance:** ~2× speedup with 4-worker pool on complex models with multiple mesh primitives.

## Material Mapping: PBR → Lambert

glTF materials use PBR (metallic-roughness). Since Voidcore only supports Basic and Lambert materials, a simplified mapping is applied:

| glTF Property | Voidcore Mapping |
|---------------|-----------------|
| `baseColorFactor` | Lambert `color` |
| `baseColorTexture` | Lambert `colorTexture` |
| `emissiveFactor` | Lambert `emissive` |
| `emissiveTexture` | Not supported (logged in debug mode) |
| `metallicFactor` | Ignored |
| `roughnessFactor` | Ignored |
| `normalTexture` | Ignored |
| `occlusionTexture` | Lambert `aoTexture` |
| `KHR_materials_unlit` | → BasicMaterial |
| `alphaMode: 'BLEND'` | Lambert with `transparent: true` |
| `alphaMode: 'MASK'` | Lambert with alpha test (future consideration) |
| `doubleSided: true` | Cull mode: none |

This mapping is intentional. The engine targets stylized/low-poly aesthetics where metallic, roughness, and normal maps aren't needed.

In debug mode, the loader logs a warning when PBR-specific properties (metallic, roughness, normal maps) are present, so artists know they won't be used.

## Coordinate Conversion: Y-up → Z-up

glTF uses Y-up, right-handed. Voidcore uses Z-up, right-handed. Conversion applied to all data at load time:

```typescript
// For every vertex position and normal:
const [x, y, z] = glTFVertex
const engineVertex = [x, z, -y]    // Y→Z, Z→-Y
```

Also applied to:
- **Animation keyframe positions**: Y↔Z swapped with negation
- **Animation keyframe rotations**: quaternion components reordered for Z-up
- **Bone rest poses**: inverse bind matrices transformed
- **Scene node transforms**: translation and rotation converted
- **AABB bounds**: recomputed from converted positions

**No per-frame conversion overhead.** Everything is baked once during loading. The animation system, scene graph, and renderer all operate purely in Z-up.

## Custom Attribute: `_materialindex`

The loader recognizes `_MATERIALINDEX` as a standard custom vertex attribute (glTF custom attribute convention with underscore prefix):

```typescript
// In glTF JSON:
{
  "primitives": [{
    "attributes": {
      "POSITION": 0,
      "NORMAL": 1,
      "_MATERIALINDEX": 4    // Custom attribute
    }
  }]
}
```

When detected, the loader reads the attribute data (typically `Uint8` accessor) and stores it as the geometry's `materialIndices` attribute, ready for the material index palette system.

## Caching

URL-based Promise caching prevents duplicate loads:

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

Subsequent calls to `loadGLTF` with the same URL return the cached Promise. Multiple React components mounting with the same model URL trigger only one network request.

### Cloning for Multiple Instances

The returned `GLTFResult` can be "cloned" for multiple instances of the same model:

```typescript
const gltf = await loadGLTF('/character.glb', engine)

// Clone creates new scene graph nodes sharing the same GPU resources (geometry, textures)
const instance1 = gltf.scene           // Original
const instance2 = cloneSceneGraph(gltf.scene)  // New nodes, shared geometry/materials
```

GPU resources (vertex buffers, textures) are shared. Only scene graph nodes and animation state are cloned.

## Disposal

```typescript
gltf.dispose()
```

Releases all GPU resources (textures, vertex/index buffers) loaded by this glTF. Does **not** remove nodes from the scene graph — the user should call `scene.remove(gltf.scene)` separately if desired.

In React mode (`useGLTF` hook), disposal is automatic on component unmount.

## Error Handling

- **Missing WASM URL**: throws with a message explaining that the Draco/Basis WASM URL must be provided
- **Network errors**: the Promise rejects with the fetch error
- **Invalid glTF**: throws a descriptive parse error
- **Unsupported extensions**: warns in debug mode, skips gracefully in production
- **Missing attributes**: warns if expected attributes (e.g., normals) are absent, generates them if possible (flat normals from positions)
