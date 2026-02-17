# Asset Loading

## Overview

Caracal's asset pipeline handles:
- **glTF 2.0** (.gltf + .bin, .glb) — the primary 3D format
- **Draco** compressed geometry — via Web Worker WASM decoder
- **KTX2** textures with **Basis Universal** transcoding — via Web Worker WASM transcoder
- Standard image textures (PNG, JPEG, WebP) — via browser `createImageBitmap`

All heavy decoding happens in Web Workers to keep the main thread free for rendering.

## glTF Loader

### Architecture

```
GLTFLoader
├── Parses glTF JSON (or GLB binary header)
├── Resolves buffer references (external .bin files or embedded)
├── Builds scene graph from glTF node hierarchy
├── Creates Geometry from mesh primitives
├── Creates Materials from glTF materials
├── Builds Skeletons from skin definitions
├── Creates AnimationClips from animation definitions
└── Delegates to DracoDecoder / KTX2Loader as needed
```

### API

```typescript
interface GLTFResult {
  scene: Group                          // Root node of the loaded scene
  meshes: Mesh[]                        // All meshes (flat list)
  skinnedMeshes: SkinnedMesh[]          // Skinned meshes
  skeletons: Skeleton[]                 // All skeletons
  animations: AnimationClip[]           // All animation clips
  cameras: Camera[]                     // Cameras defined in the file
  textures: Map<number, GPUTextureHandle> // Loaded textures by glTF index
}

const loadGLTF = async (
  url: string,
  device: GPUDevice,
  options?: GLTFLoadOptions
): Promise<GLTFResult> => {
  // ...
}

interface GLTFLoadOptions {
  dracoDecoder?: DracoDecoder           // Pre-initialized Draco decoder
  basisTranscoder?: BasisTranscoder     // Pre-initialized Basis transcoder
  autoUpload?: boolean                  // Upload geometry/textures to GPU (default: true)
  autoBuildBVH?: boolean                // Build BVH for raycasting (default: false)
}
```

### Y-Up to Z-Up Conversion

glTF uses Y-up, right-handed coordinates. Caracal applies a root transform to convert:

```typescript
// Applied to the root node of every loaded glTF scene
const Y_UP_TO_Z_UP = mat4RotateX(new Mat4(), -Math.PI / 2)

// This rotates:
//   glTF Y-up (0,1,0) → Caracal Z-up (0,0,1)
//   glTF Z-forward (0,0,-1) → Caracal Y-forward (0,1,0)
```

The rotation is baked into the root `Group` node so child transforms remain in glTF-native space. This means animation data works without modification.

### Material Mapping

glTF materials map to Caracal materials:

| glTF Material Property | Caracal Material |
|------------------------|-----------------|
| `pbrMetallicRoughness.baseColorFactor` | `LambertMaterial.color` |
| `pbrMetallicRoughness.baseColorTexture` | `LambertMaterial.map` |
| `occlusionTexture` | `LambertMaterial.aoMap` |
| `emissiveFactor` | `LambertMaterial.emissive` |
| `emissiveTexture` | (multiplied with emissiveFactor) |
| `alphaMode: "BLEND"` | `transparent: true` |
| `alphaMode: "MASK"` | `alphaCutoff` (discard in shader) |
| `doubleSided: true` | `side: 'both'` |

Since Caracal only supports Basic and Lambert materials, PBR metallic/roughness properties are approximated: metallic ≈ 0 maps well to Lambert diffuse, while metallic ≈ 1 increases the base color brightness.

### Custom Attributes (_materialindex)

The glTF loader scans for custom vertex attributes matching `_MATERIALINDEX` (following the glTF `extras` convention for custom attributes). If found, it creates the `_materialindex` buffer attribute:

```typescript
// During primitive parsing
for (const [name, accessorIndex] of Object.entries(primitive.attributes)) {
  if (name === '_MATERIALINDEX') {
    const accessor = gltf.accessors[accessorIndex]
    const data = readAccessor(accessor, bufferViews)
    geometry.attributes.set('_materialindex', createBufferAttribute({
      array: new Float32Array(data), // Convert to float for shader compatibility
      itemSize: 1,
    }))
  }
}
```

## Draco Decoder

### Architecture

Draco decoding runs in a Web Worker to avoid blocking the main thread:

```
Main Thread                          Worker Thread
────────────                         ─────────────
GLTFLoader detects Draco extension
  │
  ├─ Posts compressed buffer ──────► Worker receives buffer
  │                                    │
  │                                    ├─ Initializes Draco WASM module (once)
  │                                    │
  │                                    ├─ Decodes geometry
  │                                    │   ├─ Positions (Float32Array)
  │                                    │   ├─ Normals (Float32Array)
  │                                    │   ├─ UVs (Float32Array)
  │                                    │   ├─ Indices (Uint16/32Array)
  │                                    │   └─ Custom attributes
  │                                    │
  │  ◄─ Transfers decoded arrays ──── Posts result (Transferable)
  │
  ├─ Creates Geometry from decoded data
```

### API

```typescript
interface DracoDecoder {
  // Initialize the WASM module (call once at startup)
  init(wasmUrl: string): Promise<void>

  // Decode a Draco-compressed buffer
  decode(buffer: ArrayBuffer, attributeMap?: DracoAttributeMap): Promise<DracoResult>

  // Clean up worker
  dispose(): void
}

interface DracoResult {
  positions: Float32Array
  normals: Float32Array | null
  uvs: Float32Array | null
  indices: Uint16Array | Uint32Array
  customAttributes: Map<string, Float32Array>
  vertexCount: number
  indexCount: number
}

interface DracoAttributeMap {
  // Maps glTF attribute names to Draco attribute IDs
  // The loader provides this from the glTF Draco extension
  [gltfName: string]: number
}
```

### Integration with glTF

When the glTF loader encounters the `KHR_draco_mesh_compression` extension on a mesh primitive:

```typescript
const loadDracoPrimitive = async (
  primitive: GLTFPrimitive,
  dracoDecoder: DracoDecoder,
  bufferViews: ArrayBuffer[]
): Promise<Geometry> => {
  const dracoExt = primitive.extensions.KHR_draco_mesh_compression
  const buffer = bufferViews[dracoExt.bufferView]

  const result = await dracoDecoder.decode(buffer, dracoExt.attributes)

  const geometry = createGeometry()
  geometry.attributes.set('position', createBufferAttribute({
    array: result.positions,
    itemSize: 3,
  }))

  if (result.normals) {
    geometry.attributes.set('normal', createBufferAttribute({
      array: result.normals,
      itemSize: 3,
    }))
  }

  if (result.uvs) {
    geometry.attributes.set('uv', createBufferAttribute({
      array: result.uvs,
      itemSize: 2,
    }))
  }

  geometry.index = createBufferAttribute({
    array: result.indices,
    itemSize: 1,
  })

  // Handle custom attributes like _MATERIALINDEX
  for (const [name, data] of result.customAttributes) {
    geometry.attributes.set(name.toLowerCase(), createBufferAttribute({
      array: data,
      itemSize: 1,
    }))
  }

  return geometry
}
```

## KTX2 / Basis Universal Loader

### Why KTX2 + Basis

Standard PNG/JPEG textures must be decompressed to raw RGBA on the GPU, consuming full memory (a 1024×1024 RGBA texture = 4 MB VRAM). **GPU-compressed formats** (BC1-7, ETC2, ASTC) stay compressed in VRAM, using 4-8× less memory.

**Basis Universal** is a "supercompressed" format that transcodes to the optimal GPU format at runtime:

| GPU | Target Format | Ratio |
|-----|--------------|-------|
| Desktop (NVIDIA/AMD) | BC7 (highest quality) or BC1/BC3 | 4:1 to 8:1 |
| iOS / Apple Silicon | ASTC 4×4 | 4:1 |
| Android (most) | ETC2 | 4:1 to 8:1 |
| Android (older) | ETC1 (via ETC1S mode) | 6:1 |

### Architecture

Like Draco, Basis transcoding runs in a Web Worker:

```
Main Thread                          Worker Thread
────────────                         ─────────────
KTX2Loader receives .ktx2 file
  │
  ├─ Parses KTX2 container header
  │   (mip levels, format, dimensions)
  │
  ├─ Posts compressed data ────────► Worker receives data
  │                                    │
  │                                    ├─ Initializes Basis WASM (once)
  │                                    │
  │                                    ├─ Detects optimal target format
  │                                    │   (queries GPU capabilities)
  │                                    │
  │                                    ├─ Transcodes to target format
  │                                    │   (per mip level)
  │                                    │
  │  ◄─ Transfers transcoded data ─── Posts result (Transferable)
  │
  ├─ Uploads as compressed texture
  │   (gl.compressedTexImage2D / device.createTexture)
```

### API

```typescript
interface KTX2Loader {
  init(basisWasmUrl: string): Promise<void>

  load(
    url: string,
    device: GPUDevice
  ): Promise<GPUTextureHandle>

  transcode(
    ktx2Buffer: ArrayBuffer,
    device: GPUDevice
  ): Promise<GPUTextureHandle>

  dispose(): void
}
```

### Target Format Selection

The transcoder selects the optimal GPU format based on device capabilities:

```typescript
const selectTargetFormat = (device: GPUDevice): BasisTargetFormat => {
  if (device.backend === 'webgpu') {
    // WebGPU: prefer BC7 (desktop) or ASTC (mobile)
    if (device.limits.supportsBC) return 'bc7-rgba-unorm'
    if (device.limits.supportsASTC) return 'astc-4x4-unorm'
    if (device.limits.supportsETC2) return 'etc2-rgba8unorm'
  } else {
    // WebGL 2: check compressed texture extensions
    const gl = device._gl
    if (gl.getExtension('EXT_texture_compression_bptc')) return 'bc7'
    if (gl.getExtension('WEBGL_compressed_texture_s3tc')) return 'bc1' // or bc3 for alpha
    if (gl.getExtension('WEBGL_compressed_texture_astc')) return 'astc4x4'
    if (gl.getExtension('WEBGL_compressed_texture_etc')) return 'etc2'
  }

  // Fallback: transcode to uncompressed RGBA
  return 'rgba8'
}
```

## Standard Texture Loading

For non-KTX2 textures (PNG, JPEG, WebP), Caracal uses `createImageBitmap` for off-main-thread decoding:

```typescript
const loadTexture = async (
  url: string,
  device: GPUDevice,
  options?: TextureOptions
): Promise<GPUTextureHandle> => {
  const response = await fetch(url)
  const blob = await response.blob()
  const bitmap = await createImageBitmap(blob, {
    premultiplyAlpha: 'none',
    colorSpaceConversion: 'none',
  })

  const handle = device.createTexture({
    width: bitmap.width,
    height: bitmap.height,
    format: 'rgba8unorm',
    usage: 'sampled',
    generateMipmaps: options?.generateMipmaps ?? true,
  })

  device.writeTexture(handle, bitmap)
  bitmap.close() // Free memory

  return handle
}
```

### Texture Options

```typescript
interface TextureOptions {
  generateMipmaps: boolean     // default: true
  wrapS: WrapMode              // default: 'repeat'
  wrapT: WrapMode              // default: 'repeat'
  minFilter: FilterMode        // default: 'linear-mipmap-linear'
  magFilter: FilterMode        // default: 'linear'
  flipY: boolean               // default: false (glTF convention)
  anisotropy: number           // default: 1 (max for quality: 16)
}
```

## Pre-Initialization

WASM modules (Draco, Basis) should be initialized once at application startup to avoid delays during asset loading:

```typescript
// At app startup:
const dracoDecoder = createDracoDecoder()
const basisTranscoder = createBasisTranscoder()

await Promise.all([
  dracoDecoder.init('/wasm/draco_decoder.wasm'),
  basisTranscoder.init('/wasm/basis_transcoder.wasm'),
])

// Now loading is fast:
const model = await loadGLTF('/models/character.glb', device, {
  dracoDecoder,
  basisTranscoder,
})
```

## Asset Disposal

All loaded assets must be explicitly disposed:

```typescript
// Dispose a loaded glTF scene
const disposeGLTF = (result: GLTFResult, device: GPUDevice) => {
  for (const mesh of result.meshes) {
    mesh.geometry.dispose(device)
  }
  for (const [, texture] of result.textures) {
    device.destroyTexture(texture)
  }
}
```

No automatic garbage collection of GPU resources — explicit lifecycle management prevents memory leaks and gives the developer full control.
