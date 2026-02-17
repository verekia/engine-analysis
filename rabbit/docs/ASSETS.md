# Asset Loading

## Overview

Rabbit's asset pipeline handles three primary formats:
1. **glTF 2.0** — scene/model format (with optional Draco mesh compression)
2. **KTX2** — GPU-compressed texture format (with Basis Universal transcoder)
3. **Standard images** — PNG, JPEG, WebP for color and AO textures

All loaders are async and return Promises. Heavy decoding (Draco, Basis) runs in Web Workers via WASM modules that are loaded lazily on first use.

## glTF Loader

### Architecture

```
GLTFLoader.load(url)
  │
  ├─ Fetch .gltf JSON (or .glb binary)
  ├─ Parse JSON, resolve buffer references
  ├─ Fetch external .bin buffers (if any)
  │
  ├─ For each mesh primitive:
  │   ├─ If Draco-compressed → DracoDecoder.decode(bufferView)
  │   └─ Else → read vertex/index data directly from buffer
  │
  ├─ For each image:
  │   ├─ If KTX2 → KTX2Loader.transcode(bufferView)
  │   └─ Else → createImageBitmap(blob)
  │
  ├─ Build scene graph:
  │   ├─ Nodes → Node / Group
  │   ├─ Meshes → Mesh (with Geometry + Material)
  │   ├─ Skins → SkinnedMesh + Skeleton + Bones
  │   └─ Animations → AnimationClip[]
  │
  └─ Apply Y-up → Z-up conversion
```

### Y-up to Z-up Conversion

glTF specifies Y-up, right-handed coordinates. Rabbit uses Z-up, right-handed. The conversion is a 90° rotation around the X axis, applied at the root of the loaded scene:

```
Z-up from Y-up:
  x' =  x
  y' = -z
  z' =  y

As a matrix (applied to the root node of the glTF scene):
  [1  0  0  0]
  [0  0 -1  0]
  [0  1  0  0]
  [0  0  0  1]
```

This rotation is applied once to the root transform. All child transforms within the glTF remain relative to their parents and need no modification. Animation keyframes targeting position/rotation are also correct because they're defined in local space.

### Custom Attributes

The loader recognizes custom vertex attributes prefixed with `_`:
- `_MATERIALINDEX` → mapped to vertex attribute `a_materialIndex`

### Accessor Parsing

glTF accessors map to typed arrays:

| componentType | TypedArray |
|---|---|
| 5120 (BYTE) | Int8Array |
| 5121 (UNSIGNED_BYTE) | Uint8Array |
| 5122 (SHORT) | Int16Array |
| 5123 (UNSIGNED_SHORT) | Uint16Array |
| 5125 (UNSIGNED_INT) | Uint32Array |
| 5126 (FLOAT) | Float32Array |

Index buffers use Uint16Array when possible (vertex count ≤ 65535) for memory savings on mobile.

### Skeleton Building

For skinned meshes:
1. Read the `skin.joints` array → create `Bone` nodes
2. Read `skin.inverseBindMatrices` accessor → store as `Mat4[]`
3. Build bone hierarchy from the glTF node tree
4. Assign `boneIndex` to each bone (matches the joint index in the skin)

### Animation Parsing

glTF animation channels map to `AnimationClip` tracks:

```typescript
interface AnimationTrack {
  targetBone: number            // bone index
  property: 'position' | 'rotation' | 'scale'
  interpolation: 'linear' | 'step' | 'cubicspline'
  times: Float32Array           // keyframe timestamps
  values: Float32Array          // keyframe values (vec3 or quat)
}

interface AnimationClip {
  name: string
  duration: number
  tracks: AnimationTrack[]
}
```

## Draco Decoder

### Lazy WASM Loading

The Draco WASM module (~300KB) is loaded on first encounter of a Draco-compressed mesh:

```typescript
let dracoModule: DracoDecoderModule | null = null

const ensureDraco = async (): Promise<DracoDecoderModule> => {
  if (!dracoModule) {
    dracoModule = await loadDracoWasm()
  }
  return dracoModule
}
```

### Decoding Pipeline

```
Draco buffer (compressed)
  │
  ├─ Create DecoderBuffer from ArrayBuffer
  ├─ GetEncodedGeometryType → TRIANGULAR_MESH
  ├─ DecodeBufferToMesh
  │
  ├─ Extract attributes:
  │   ├─ POSITION → Float32Array
  │   ├─ NORMAL → Float32Array
  │   ├─ TEX_COORD → Float32Array
  │   ├─ COLOR → Float32Array (if present)
  │   ├─ JOINTS → Uint16Array (if present)
  │   ├─ WEIGHTS → Float32Array (if present)
  │   └─ GENERIC (_MATERIALINDEX) → Float32Array (if present)
  │
  ├─ Extract indices → Uint16Array or Uint32Array
  │
  └─ Return Geometry
```

### Worker Pool

For scenes with many Draco meshes, decoding runs in a pool of Web Workers (2-4 workers). Each worker loads its own WASM instance. The GLTFLoader distributes mesh primitives across workers for parallel decoding.

```typescript
class DracoWorkerPool {
  private workers: Worker[]
  private queue: DracoTask[]
  private available: Worker[]

  decode(buffer: ArrayBuffer): Promise<DecodedGeometry> {
    return new Promise((resolve) => {
      const worker = this.available.pop()
      if (worker) {
        this.dispatch(worker, buffer, resolve)
      } else {
        this.queue.push({ buffer, resolve })
      }
    })
  }
}
```

## KTX2 / Basis Universal Loader

### Overview

KTX2 is a container format for GPU-compressed textures. The actual texture data is encoded with Basis Universal, which can be transcoded to the optimal GPU format for the current device:

| GPU | Target format | Quality |
|---|---|---|
| Desktop (NVIDIA/AMD/Intel) | BC7 or BC3 (S3TC) | Best |
| iOS / Apple Silicon | ASTC 4x4 | Best |
| Android (Adreno/Mali) | ASTC 4x4 or ETC2 | Good |
| Fallback | RGBA8 (uncompressed) | Baseline |

### Format Detection

```typescript
const detectBestFormat = (device: HalDevice): TranscodeTarget => {
  if (device.features.textureCompressionASTC) return 'astc-4x4-unorm'
  if (device.features.textureCompressionBC) return 'bc7-rgba-unorm'
  if (device.features.textureCompressionETC2) return 'etc2-rgba8unorm'
  return 'rgba8unorm' // fallback: transcode to raw RGBA
}
```

### Transcoding Pipeline

```
KTX2 file (ArrayBuffer)
  │
  ├─ Parse KTX2 header (supercompression, format, dimensions, mip levels)
  │
  ├─ If Zstandard supercompression → decompress via bundled zstd-lite
  │
  ├─ Initialize Basis transcoder (WASM, ~200KB, loaded lazily)
  │
  ├─ For each mip level:
  │   └─ basisTranscoder.transcode(level, targetFormat) → CompressedTexelBlock[]
  │
  └─ Create HalTexture with compressed data and mip chain
```

### Basis WASM Worker

Like Draco, the Basis transcoder runs in a Web Worker:

```typescript
class KTX2Loader {
  private worker: Worker | null = null

  async load(url: string, device: HalDevice): Promise<HalTexture> {
    const buffer = await fetch(url).then(r => r.arrayBuffer())
    const targetFormat = detectBestFormat(device)

    if (!this.worker) {
      this.worker = new Worker(new URL('./basis-worker.ts', import.meta.url))
      await this.initWorker()
    }

    const result = await this.postMessage({ buffer, targetFormat })
    return device.createTexture({
      width: result.width,
      height: result.height,
      format: targetFormat,
      mipLevelCount: result.mipLevels.length,
      data: result.mipLevels,
    })
  }
}
```

## Texture Loader

For standard image formats (PNG, JPEG, WebP):

```typescript
class TextureLoader {
  private cache = new Map<string, HalTexture>()

  async load(url: string, device: HalDevice): Promise<HalTexture> {
    const cached = this.cache.get(url)
    if (cached) return cached

    const response = await fetch(url)
    const blob = await response.blob()

    // createImageBitmap is faster than Image + canvas
    // premultiplyAlpha: none — we handle alpha in shaders
    const bitmap = await createImageBitmap(blob, {
      premultiplyAlpha: 'none',
      colorSpaceConversion: 'none',
    })

    const texture = device.createTexture({
      width: bitmap.width,
      height: bitmap.height,
      format: 'rgba8unorm',
      usage: TextureUsage.SAMPLED | TextureUsage.COPY_DST,
      data: bitmap,
      mipLevelCount: Math.floor(Math.log2(Math.max(bitmap.width, bitmap.height))) + 1,
    })

    // Generate mipmaps
    generateMipmaps(device, texture)

    bitmap.close()
    this.cache.set(url, texture)
    return texture
  }
}
```

### Mipmap Generation

**WebGPU**: Use a compute shader or render pass that downsamples each mip level from the previous one.

**WebGL2**: Use `gl.generateMipmap()` after uploading the base level. For compressed textures, mipmaps are provided in the KTX2 data.

## Geometry Object

All loaders produce `Geometry` objects:

```typescript
interface Geometry {
  vertexBuffers: {
    position: HalBuffer     // vec3 float
    normal?: HalBuffer       // vec3 float
    uv?: HalBuffer           // vec2 float
    color?: HalBuffer        // vec4 float (vertex colors)
    joints?: HalBuffer       // uvec4 uint16 (bone indices)
    weights?: HalBuffer      // vec4 float (bone weights)
    materialIndex?: HalBuffer // float (material index)
  }
  indexBuffer: HalBuffer     // uint16 or uint32
  indexCount: number
  indexFormat: 'uint16' | 'uint32'
  vertexCount: number
  aabb: AABB                 // computed from position data
  vertexLayout: HalVertexLayout  // describes attribute locations and formats
}
```

## Resource Management

All GPU resources (buffers, textures) are reference-counted. When a Mesh or Material is removed from the scene and has no other references, its GPU resources are scheduled for deletion at the end of the frame (to avoid destroying resources still in use by in-flight GPU commands).

```typescript
class RefCounted {
  private _refCount = 0

  addRef(): void { this._refCount++ }
  release(): void {
    this._refCount--
    if (this._refCount <= 0) {
      this.destroy()
    }
  }
  abstract destroy(): void
}
```
