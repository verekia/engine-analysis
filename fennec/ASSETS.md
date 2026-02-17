# Assets — GLTF, Draco, KTX2, Textures

## Overview

Fennec's asset pipeline loads GLTF/GLB files with optional Draco mesh compression and KTX2/Basis Universal texture compression. All heavy decoding runs in Web Workers to keep the main thread free. The loader returns engine-ready objects (meshes, materials, textures, skeletons) that can be directly added to the scene.

## GLTF Loader

### Supported Features

| GLTF Feature | Support | Notes |
|---|---|---|
| Meshes | Full | Positions, normals, UVs, tangents |
| Indices | Full | Uint16 and Uint32 |
| Vertex colors | Full | `COLOR_0` attribute |
| Custom attributes | Full | `_MATERIALINDEX` for material index system |
| Multiple primitives | Full | Each primitive becomes a draw call |
| Skinning | Full | Joints + weights, max 4 influences |
| Skeletal animation | Full | Translation, rotation, scale keyframes |
| Morph targets | Not supported | Out of scope for v1 |
| Cameras | Ignored | Fennec uses its own camera |
| PBR materials | Mapped to Lambert | baseColorFactor → color, emissiveFactor → emissive |
| Draco compression | Full | Via Web Worker |
| KTX2 textures | Full | Via Web Worker (Basis Universal) |
| Embedded textures | Full | data URIs and GLB binary chunk |
| External textures | Full | Relative URL resolution |

### API

```typescript
interface GLTFLoadOptions {
  draco?: boolean | string          // Enable Draco, optionally custom decoder path
  ktx2?: boolean | string           // Enable KTX2, optionally custom transcoder path
  normalize?: boolean               // Normalize to unit scale (default: false)
}

interface GLTFResult {
  scene: Group                      // Root group containing all meshes
  meshes: Mesh[]                    // Flat list of all meshes
  skinnedMeshes: SkinnedMesh[]      // Skinned meshes
  skeletons: Skeleton[]             // All skeletons
  animations: AnimationClip[]       // All animation clips
}

const loadGLTF = async (url: string, options?: GLTFLoadOptions): Promise<GLTFResult>
```

### Usage

```typescript
// Simple load
const { scene } = await loadGLTF('/models/world.glb')
scene.add(world)

// With Draco
const { scene, animations } = await loadGLTF('/models/character.glb', { draco: true })
const mixer = new AnimationMixer(scene)
mixer.play(animations[0])

// With material index
const { scene: worldScene, meshes } = await loadGLTF('/models/world.glb', { draco: true })
for (const mesh of meshes) {
  if (mesh.geometry.hasAttribute('materialIndex')) {
    mesh.setMaterialIndex(0, { color: 0xffffff })
    mesh.setMaterialIndex(1, { color: 0x222222 })
    mesh.setMaterialIndex(2, { color: 0x00ccaa, emissive: 0x00ccaa, emissiveIntensity: 0.7 })
  }
}
```

### Coordinate System Conversion

GLTF uses Y-up, right-handed. Fennec uses Z-up, right-handed. The loader applies a conversion at the root level:

```typescript
// Applied to the root group of every loaded GLTF
const gltfToFennec = mat4.fromRotation(mat4.create(), -Math.PI / 2, [1, 0, 0])
// This rotates the entire model so Y-up becomes Z-up
```

Bone transforms, animations, and bounding boxes are all converted during import so no runtime cost is incurred.

### GLTF Parse Pipeline

```
1. Fetch .glb / .gltf
   └── Parse JSON header, locate binary chunk(s)

2. Resolve buffer views and accessors
   └── Map to typed arrays (positions, normals, UVs, indices, etc.)

3. Decode compressed data (parallel Web Workers)
   ├── Draco meshes → decoded buffer views
   └── KTX2 textures → transcoded GPU texture data

4. Build engine objects
   ├── Geometry: create vertex buffers + index buffer + VAO/vertex layout
   ├── Textures: upload to GPU
   ├── Materials: map GLTF PBR → Fennec Basic/Lambert
   ├── Skeleton: build bone hierarchy + inverse bind matrices
   └── AnimationClips: parse keyframe tracks

5. Assemble scene graph
   ├── Create Groups for GLTF nodes
   ├── Parent/child hierarchy
   └── Apply Y-up → Z-up conversion

6. Return GLTFResult
```

## Draco Decoder

### Architecture

Draco decoding uses a dedicated Web Worker running the Draco WASM decoder. The worker is created on first use and reused for subsequent loads.

```typescript
// Worker pool (1-2 workers depending on navigator.hardwareConcurrency)
const dracoWorkerPool = new WorkerPool('/decoders/draco_decoder.wasm', {
  maxWorkers: Math.min(2, navigator.hardwareConcurrency || 1),
})

const decodeDraco = async (data: ArrayBuffer, attributes: DracoAttributeMap): Promise<DecodedGeometry> => {
  return dracoWorkerPool.run({ data, attributes })
}
```

### Worker Protocol

```typescript
// Main thread → Worker
interface DracoDecodeRequest {
  type: 'decode'
  id: number
  data: ArrayBuffer       // Transferable
  attributes: {
    position: number      // Draco attribute ID
    normal: number
    uv: number
    color?: number
    materialIndex?: number
    joints?: number
    weights?: number
  }
}

// Worker → Main thread
interface DracoDecodeResponse {
  type: 'decoded'
  id: number
  geometry: {
    positions: Float32Array    // Transferable
    normals: Float32Array
    uvs: Float32Array
    indices: Uint32Array
    colors?: Float32Array
    materialIndices?: Uint8Array
    joints?: Uint16Array
    weights?: Float32Array
  }
}
```

### WASM Loading

The Draco WASM binary is fetched and cached:

```typescript
// In the worker
let dracoModule: DracoDecoderModule | null = null

self.onmessage = async (e: MessageEvent<DracoDecodeRequest>) => {
  if (!dracoModule) {
    dracoModule = await DracoDecoderModule({ wasmBinary: e.data.wasmBinary })
  }
  const decoder = new dracoModule.Decoder()
  const buffer = new dracoModule.DecoderBuffer()
  buffer.Init(new Int8Array(e.data.data), e.data.data.byteLength)
  // ... decode and extract attributes ...
  // Transfer buffers back (zero-copy)
  self.postMessage(response, [positions.buffer, normals.buffer, ...])
}
```

## KTX2 / Basis Universal Transcoder

### Architecture

KTX2 files contain Basis Universal compressed texture data that must be transcoded to a GPU-native format at load time. The transcoder runs in a Web Worker.

### Target Format Selection

The transcoder picks the best GPU-native format based on device capabilities:

```typescript
const selectTranscodeTarget = (capabilities: BackendCapabilities): BasisTranscodeFormat => {
  // Desktop (WebGPU or WebGL2 with extensions)
  if (capabilities.bc7) return 'BC7'            // Best quality
  if (capabilities.s3tc) return 'BC1' // or BC3  // Widely supported
  // Mobile
  if (capabilities.astc) return 'ASTC_4x4'      // iOS, modern Android
  if (capabilities.etc2) return 'ETC2'           // Android
  // Fallback
  return 'RGBA8'                                  // Uncompressed (last resort)
}
```

### Transcoder Worker

```typescript
// Main thread → Worker
interface KTX2TranscodeRequest {
  type: 'transcode'
  id: number
  data: ArrayBuffer            // KTX2 file data (transferable)
  targetFormat: BasisTranscodeFormat
}

// Worker → Main thread
interface KTX2TranscodeResponse {
  type: 'transcoded'
  id: number
  mipmaps: Array<{
    data: ArrayBuffer          // Transferable
    width: number
    height: number
  }>
  format: TextureFormat        // The actual GPU format used
}
```

### Mipmap Handling

KTX2 files can contain pre-computed mipmaps. If they do, all mip levels are transcoded and uploaded. If not, mipmaps are generated on the GPU after upload (for non-compressed formats) or the texture is used without mipmaps (for compressed formats).

## Texture Loading Pipeline

```
Texture source → Identify format → Process → Upload to GPU

PNG/JPG:
  fetch → createImageBitmap → texImage2D / copyExternalImageToTexture

KTX2:
  fetch → Worker transcode → compressedTexImage2D / writeTexture

Embedded (data URI):
  base64 decode → same as PNG/JPG

Embedded (GLB binary chunk):
  slice ArrayBuffer → createImageBitmap → upload
```

### Texture Upload (WebGL2)

```typescript
const uploadTexture = (gl: WebGL2RenderingContext, texture: WebGLTexture, data: TextureData) => {
  gl.bindTexture(gl.TEXTURE_2D, texture)

  if (data.compressed) {
    for (let level = 0; level < data.mipmaps.length; level++) {
      const mip = data.mipmaps[level]
      gl.compressedTexImage2D(gl.TEXTURE_2D, level, data.internalFormat,
        mip.width, mip.height, 0, new Uint8Array(mip.data))
    }
  } else {
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA8, data.width, data.height, 0,
      gl.RGBA, gl.UNSIGNED_BYTE, data.bitmap)
    if (data.generateMipmaps) gl.generateMipmap(gl.TEXTURE_2D)
  }
}
```

### Texture Upload (WebGPU)

```typescript
const uploadTexture = (device: GPUDevice, texture: GPUTexture, data: TextureData) => {
  if (data.compressed) {
    for (let level = 0; level < data.mipmaps.length; level++) {
      const mip = data.mipmaps[level]
      device.queue.writeTexture(
        { texture, mipLevel: level },
        mip.data,
        { bytesPerRow: mip.bytesPerRow, rowsPerImage: mip.height },
        { width: mip.width, height: mip.height },
      )
    }
  } else {
    device.queue.copyExternalImageToTexture(
      { source: data.bitmap },
      { texture },
      { width: data.width, height: data.height },
    )
    // Mipmap generation via compute shader or render passes
  }
}
```

## Caching Strategy

Loaded assets are cached by URL to prevent redundant loads:

```typescript
const assetCache = new Map<string, Promise<GLTFResult>>()

const loadGLTFCached = (url: string, options?: GLTFLoadOptions): Promise<GLTFResult> => {
  const key = `${url}:${JSON.stringify(options)}`
  let promise = assetCache.get(key)
  if (!promise) {
    promise = loadGLTF(url, options)
    assetCache.set(key, promise)
  }
  return promise
}
```

GPU resources (buffers, textures) are cloned for each instance — the parsed data is shared but each mesh gets its own world matrix and material state. This means loading the same model 100 times only decodes once but creates 100 independent scene objects.

## Disposal

```typescript
const disposeGLTF = (result: GLTFResult) => {
  for (const mesh of result.meshes) {
    mesh.geometry.dispose()  // Delete GPU buffers
    mesh.material.dispose()  // Delete GPU textures, pipelines
  }
  assetCache.delete(result._cacheKey)
}
```
