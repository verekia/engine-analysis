# Asset Loading Architecture

## Overview

Shark supports loading 3D assets from GLTF 2.0 files with optional Draco mesh compression and KTX2/Basis Universal texture compression. The loading pipeline is fully asynchronous and designed for streaming.

## GLTF Loader

### Supported Features

| GLTF Feature | Support | Notes |
|---|---|---|
| Meshes (triangles) | Yes | Converted to Shark Geometry |
| Materials (PBR metallic-roughness) | Partial | Mapped to BasicMaterial or LambertMaterial |
| Textures (PNG, JPEG) | Yes | Loaded as GpuTexture |
| Skeleton / Joints | Yes | Mapped to Shark Skeleton |
| Animations | Yes | Mapped to AnimationClip |
| Scene hierarchy | Yes | Mapped to Node3D tree |
| Cameras | No | Use Shark cameras instead |
| Morph targets | No | Out of scope |
| Extensions: KHR_draco_mesh_compression | Yes | Draco WASM decoder |
| Extensions: KHR_texture_basisu | Yes | KTX2/Basis transcoder |
| Custom attributes: `_materialindex` | Yes | Mapped to materialIndex vertex attribute |

### Loading Pipeline

```
GLTFLoader.load(url)
  │
  ├── 1. Fetch .gltf / .glb file
  │     ├── .gltf: JSON + separate .bin files
  │     └── .glb: Single binary (JSON chunk + BIN chunk)
  │
  ├── 2. Parse JSON structure
  │     ├── Extract scenes, nodes, meshes, materials, textures, skins, animations
  │     └── Resolve buffer references
  │
  ├── 3. Load binary buffers (parallel)
  │     ├── Fetch external .bin files (if .gltf)
  │     └── Or use embedded binary chunk (if .glb)
  │
  ├── 4. Decode meshes (parallel per mesh)
  │     ├── If Draco compressed → DracoDecoder.decode(buffer)
  │     ├── Else → read accessor data directly from buffer views
  │     ├── Extract: positions, normals, uvs, indices, joints, weights
  │     ├── Extract: _materialindex custom attribute
  │     └── Build Geometry object
  │
  ├── 5. Load textures (parallel per texture)
  │     ├── If KTX2 → KTX2Loader.transcode(buffer)
  │     ├── Else → createImageBitmap(blob)
  │     └── Upload to GpuTexture
  │
  ├── 6. Build materials
  │     ├── Map GLTF PBR → Shark LambertMaterial
  │     │   ├── baseColorFactor → color
  │     │   ├── baseColorTexture → map
  │     │   ├── emissiveFactor → emissive
  │     │   └── alphaMode → transparent/opacity
  │     └── Unlit extension → BasicMaterial
  │
  ├── 7. Build scene graph
  │     ├── Create Node3D / Mesh / SkinnedMesh for each GLTF node
  │     ├── Apply transforms (position, rotation, scale or matrix)
  │     ├── Convert Y-up → Z-up (rotate -90° around X at root)
  │     └── Establish parent-child relationships
  │
  ├── 8. Build skeletons
  │     ├── Create Skeleton from GLTF skin
  │     ├── Map joint indices to Node3D bones
  │     └── Store inverse bind matrices
  │
  └── 9. Build animations
        ├── Create AnimationClip for each GLTF animation
        ├── Sample channels: translation, rotation, scale
        └── Map channel targets to joint indices
```

### Y-up to Z-up Conversion

GLTF uses Y-up, right-handed coordinates. Shark uses Z-up, right-handed. The conversion is applied at the root level:

```typescript
// Rotation matrix: -90° around X axis
// Y → Z, Z → -Y
const gltfToShark = mat4FromRotationX(-Math.PI / 2)

// Applied to root node of imported scene
importedRoot.applyMatrix(gltfToShark)
```

This rotation is baked into the vertex positions and normals during import, so it doesn't cost anything at runtime.

### Return Type

```typescript
interface GLTFResult {
  scene: Group                    // Root node with full hierarchy
  scenes: Group[]                 // All scenes in the file
  meshes: Mesh[]                  // All meshes (flat list)
  skinnedMeshes: SkinnedMesh[]    // All skinned meshes
  animations: AnimationClip[]     // All animation clips
  skeletons: Skeleton[]           // All skeletons
}
```

## Draco Decoder

### Integration

Draco is a mesh compression library by Google. Shark uses the WASM build of `draco3d` for decoding.

```typescript
// Initialization (once)
const dracoDecoder = await DracoDecoder.init({
  wasmUrl: '/draco/draco_decoder.wasm',
  wasmWrapperUrl: '/draco/draco_wasm_wrapper.js',
})

// Decoding (per mesh)
const decoded = dracoDecoder.decode(compressedBuffer, attributeMap)
// decoded.positions: Float32Array
// decoded.normals: Float32Array
// decoded.uvs: Float32Array
// decoded.indices: Uint32Array
// decoded.customAttributes['_materialindex']: Uint8Array
```

### Web Worker Offloading

Draco decoding is CPU-intensive. For large models, decoding runs in a Web Worker:

```
Main thread                    Worker thread
     │                              │
     ├── postMessage(buffer) ──────►│
     │                              ├── dracoDecoder.decode(buffer)
     │◄── postMessage(result) ──────┤
     │                              │
     ├── Build Geometry             │
```

The worker is created once and reused. For multiple meshes in a single GLTF, decode requests are queued and processed sequentially in the worker (Draco's WASM module is not thread-safe within a single instance).

### Custom Attribute Handling

GLTF custom attributes like `_materialindex` are passed through Draco as generic attributes. The attribute map tells the decoder which Draco attribute ID maps to which semantic:

```typescript
const attributeMap = {
  POSITION: 0,
  NORMAL: 1,
  TEX_COORD_0: 2,
  _materialindex: 3,  // Custom attribute
}
```

## KTX2 / Basis Universal Loader

### Overview

KTX2 is a container format for GPU-compressed textures. Basis Universal is a "supercompressed" texture format that can be transcoded to any GPU-native format (BC, ETC, ASTC) at load time.

### Transcoding Pipeline

```
KTX2 file → Basis Universal supercompressed data
                │
                ├── Detect GPU capabilities
                │   ├── ASTC (most mobile GPUs, best quality)
                │   ├── ETC2 (OpenGL ES 3.0+ mobile)
                │   ├── BC7 / BC3 (desktop GPUs)
                │   └── RGBA8 (fallback, no compression)
                │
                ├── Transcode to best available format
                │   └── basis_universal WASM transcoder
                │
                └── Upload to GpuTexture with native format
```

### Target Format Selection

```typescript
const selectTranscodeFormat = (device: GpuDevice): BasisFormat => {
  // Query compressed texture support
  if (device.supports('texture-compression-astc')) return BasisFormat.ASTC_4x4
  if (device.supports('texture-compression-etc2')) return BasisFormat.ETC2_RGBA
  if (device.supports('texture-compression-bc'))   return BasisFormat.BC7
  return BasisFormat.RGBA8  // Uncompressed fallback
}
```

- **WebGPU**: Query `GPUAdapter.features` for compression extensions
- **WebGL2**: Query `gl.getExtension('WEBGL_compressed_texture_astc')`, etc.

### WASM Transcoder

```typescript
// Initialization (once)
const basisTranscoder = await BasisTranscoder.init({
  wasmUrl: '/basis/basis_transcoder.wasm',
  wasmWrapperUrl: '/basis/basis_transcoder.js',
})

// Transcoding (per texture)
const result = basisTranscoder.transcode(ktx2Buffer, targetFormat)
// result.width: number
// result.height: number
// result.format: TextureFormat (e.g., ASTC_4x4_RGBA)
// result.mipLevels: ArrayBufferView[] (one per mip)
```

Like Draco, transcoding runs in a Web Worker to avoid blocking the main thread.

### Mipmap Handling

KTX2 files can include pre-computed mipmaps. If present, all mip levels are transcoded and uploaded. If absent, mipmaps are generated after transcoding:

- **WebGPU**: Use compute shader or `copyTextureToTexture` with downsampled data
- **WebGL2**: Use `gl.generateMipmap()` for uncompressed fallback; compressed formats require all levels in the KTX2

## Caching and Deduplication

### Texture Deduplication

If multiple materials reference the same texture URI, the texture is loaded and uploaded once. A cache maps URIs to `GpuTexture` objects:

```typescript
const textureCache = new Map<string, GpuTexture>()

const loadTexture = async (uri: string): Promise<GpuTexture> => {
  if (textureCache.has(uri)) return textureCache.get(uri)!
  const texture = await doLoadTexture(uri)
  textureCache.set(uri, texture)
  return texture
}
```

### Geometry Deduplication

GLTF meshes referencing the same primitives are deduplicated at the geometry level. Multiple `Mesh` nodes can share the same `Geometry` object.

## Disposal

All loaded resources must be explicitly disposed when no longer needed:

```typescript
const disposeGLTF = (result: GLTFResult, device: GpuDevice) => {
  for (const mesh of result.meshes) {
    mesh.geometry.dispose(device)
    mesh.material.dispose(device)  // Disposes owned textures
  }
}
```

The React bindings handle this automatically when components unmount.
