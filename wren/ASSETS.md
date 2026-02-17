# Asset Loading (GLTF, Draco, KTX2, Basis)

This document covers asset loading systems for GLTF models, Draco mesh compression, and KTX2/Basis Universal texture compression.

## GLTF Loader

Loads glTF 2.0 files (.gltf + .bin, or .glb). Returns a structured result containing scenes, meshes, materials, textures, animations, and skeletons.

```ts
interface GLTFResult {
  scenes: GroupNode[]
  meshes: MeshNode[]
  skinnedMeshes: SkinnedMeshNode[]
  materials: Material[]
  textures: TextureHandle[]
  animations: AnimationClip[]
  skeletons: Skeleton[]
}

const loadGLTF = async (
  url: string,
  device: WrenDevice,
  options?: {
    draco?: DracoDecoder        // Pre-initialized Draco decoder
    ktx2?: KTX2Transcoder       // Pre-initialized KTX2 transcoder
  }
): Promise<GLTFResult>
```

### GLTF Processing Pipeline

1. **Parse**: Read JSON (or GLB header + JSON chunk)
2. **Load buffers**: Fetch external .bin files or extract GLB binary chunk
3. **Decode compressed meshes**: If a mesh uses `KHR_draco_mesh_compression`, send the compressed buffer to the Draco Web Worker for decoding
4. **Process meshes**: Convert accessor data to `Geometry` objects, applying the Y-up to Z-up rotation
5. **Process materials**: Map glTF PBR materials to Wren's Basic/Lambert materials (PBR metallic-roughness is approximated as Lambert with the base color)
6. **Load textures**: Decode images or KTX2 textures
7. **Process animations**: Convert animation samplers to `AnimationClip` objects
8. **Process skins**: Build `Skeleton` objects from joint hierarchies
9. **Build scene graph**: Construct the node hierarchy

### Coordinate System Conversion

GLTF uses Y-up, right-handed. Wren uses Z-up, right-handed. On import, the root of each GLTF scene is wrapped in a group node with a -90 degree rotation around X:

```ts
const gltfToWren = mat4FromRotationX(-Math.PI / 2)
```

This rotation is baked into the root group's transform and propagates to all children automatically.

### Custom Attributes

The loader recognizes custom vertex attributes prefixed with `_`:
- `_materialindex` → Maps to `_materialindex` attribute for material index system
- Other custom attributes are preserved and accessible via `geometry.attributes`

## Draco Decoder

The Draco decoder runs in a Web Worker to avoid blocking the main thread:

```ts
interface DracoDecoder {
  decode(buffer: ArrayBuffer): Promise<DecodedMesh>
  dispose(): void
}

interface DecodedMesh {
  positions: Float32Array
  normals: Float32Array | null
  uvs: Float32Array | null
  colors: Float32Array | null
  indices: Uint16Array | Uint32Array
  customAttributes: Map<string, { data: TypedArray, itemSize: number }>
}

const createDracoDecoder = async (): Promise<DracoDecoder>
```

### Worker Architecture

```
Main Thread                          Worker Thread
──────────                           ─────────────
createDracoDecoder()  ──────────►    Load draco_decoder.wasm
                     ◄──────────    Module ready

decode(buffer)        ──────────►    DracoDecoder.decode(buffer)
  (transfer)                         Read attributes
                     ◄──────────    Return typed arrays
                       (transfer)
```

The compressed buffer and decoded arrays are transferred (not copied) between threads using `postMessage` with transferable objects.

### Initialization

The WASM module (~300KB) is loaded lazily on first `createDracoDecoder()` call. The decoder URL can be configured:

```ts
const decoder = await createDracoDecoder({
  wasmUrl: '/draco/draco_decoder.wasm',
  workerUrl: '/draco/draco_worker.js',  // Optional: custom worker script
})
```

## Textures

### Texture Abstraction

```ts
interface WrenTexture {
  id: number
  width: number
  height: number
  format: TextureFormat
  mipLevels: number
  gpuTexture: GPUTextureHandle | null
  gpuSampler: GPUSamplerHandle | null
  source: ImageBitmap | CompressedTextureData | null  // Retained until upload
  label: string
}

type TextureFormat =
  | 'rgba8unorm'
  | 'rgba8srgb'
  | 'r8unorm'          // For single-channel (AO)
  | 'bc1-rgba-unorm'   // Desktop compressed (DXT1)
  | 'bc3-rgba-unorm'   // Desktop compressed (DXT5)
  | 'bc7-rgba-unorm'   // Desktop compressed (BC7)
  | 'etc2-rgba8unorm'  // Android compressed
  | 'astc-4x4-unorm'   // Mobile compressed (ASTC)

interface SamplerDescriptor {
  minFilter: 'nearest' | 'linear'
  magFilter: 'nearest' | 'linear'
  mipmapFilter: 'nearest' | 'linear'
  wrapU: 'repeat' | 'clamp' | 'mirror'
  wrapV: 'repeat' | 'clamp' | 'mirror'
  maxAnisotropy: number            // 1-16, default 4
}
```

### Color Textures

Standard RGBA textures used for albedo/diffuse maps. Loaded from PNG, JPEG, or WebP.

```ts
const loadTexture = async (
  url: string,
  device: WrenDevice,
  options?: {
    srgb?: boolean         // Default: true (color textures are sRGB)
    generateMipmaps?: boolean  // Default: true
    flipY?: boolean        // Default: false (GLTF convention)
    sampler?: Partial<SamplerDescriptor>
  }
): Promise<WrenTexture>
```

**Loading pipeline:**
1. `fetch()` the image URL
2. `createImageBitmap()` for off-main-thread decode (with `{ premultiplyAlpha: 'none', colorSpaceConversion: 'none' }`)
3. Upload to GPU via HAL
4. Generate mipmaps if requested (WebGPU: compute shader or blit chain, WebGL2: `gl.generateMipmap()`)

### AO Textures

Ambient occlusion maps are single-channel (grayscale). Stored as `r8unorm` to save GPU memory (1/4 the size of RGBA).

```ts
const loadAOTexture = async (url: string, device: WrenDevice): Promise<WrenTexture> => {
  return loadTexture(url, device, {
    srgb: false,           // AO is linear data
    generateMipmaps: true,
  })
}
```

AO maps typically use the second UV set (`uv2`). The material's `aoMapIntensity` controls the strength of the effect:

```glsl
// Fragment shader
float ao = texture(u_aoMap, v_uv2).r;
ao = mix(1.0, ao, u_aoIntensity);
litColor *= ao;
```

## KTX2 / Basis Universal Compressed Textures

GPU-compressed textures that are transcoded at runtime to the best format supported by the device.

```ts
interface KTX2Transcoder {
  transcode(buffer: ArrayBuffer): Promise<CompressedTextureData>
  dispose(): void
}

interface CompressedTextureData {
  width: number
  height: number
  format: TextureFormat       // The GPU-native format after transcoding
  mipLevels: CompressedMipLevel[]
}

interface CompressedMipLevel {
  width: number
  height: number
  data: Uint8Array
}

const createKTX2Transcoder = async (options?: {
  wasmUrl?: string
}): Promise<KTX2Transcoder>
```

### KTX2 Transcoding Pipeline

#### Target Format Selection

At device initialization, probe for supported compressed texture formats:

```ts
const selectTranscodeTarget = (device: WrenDevice): TextureFormat => {
  // WebGPU: check adapter features
  // WebGL2: check extensions

  if (device.supports('texture-compression-astc'))   return 'astc-4x4-unorm'  // Best: mobile
  if (device.supports('texture-compression-bc'))     return 'bc7-rgba-unorm'  // Best: desktop
  if (device.supports('texture-compression-etc2'))   return 'etc2-rgba8unorm' // Fallback: Android
  return 'rgba8unorm'  // Fallback: decompress to RGBA (rare)
}
```

Priority order:
1. **ASTC 4x4** — Best quality/compression on modern mobile (iOS, recent Android, Apple Silicon Macs)
2. **BC7** — Best quality on desktop GPUs (Windows, Linux, Intel Macs)
3. **ETC2** — Guaranteed on all WebGL2 devices, lower quality than BC7/ASTC
4. **RGBA8** — Uncompressed fallback (if no compressed format is available)

#### Worker-Based Transcoding

Like Draco, the Basis Universal WASM transcoder runs in a Web Worker:

```
Main Thread                          Worker Thread
──────────                           ─────────────
createKTX2Transcoder() ──────────►   Load basis_transcoder.wasm (~200KB)
                      ◄──────────   Module ready

transcode(ktx2Buffer)  ──────────►  Parse KTX2 container
  (transfer)                        Read supercompression scheme (ETC1S or UASTC)
                                    Transcode each mip level to target format
                      ◄──────────  Return CompressedTextureData
                        (transfer)
```

#### Codec Selection Guide

| Use Case | Codec | Compression | Quality |
|----------|-------|-------------|---------|
| Diffuse/albedo | ETC1S | ~0.5-1 bpp | Good, some block artifacts |
| Normal maps | UASTC | ~4-6 bpp (with Zstd) | Near-lossless |
| UI/text | UASTC | ~4-6 bpp | Near-lossless |
| Bulk backgrounds | ETC1S | ~0.5-1 bpp | Good enough |

### GPU Upload

#### Uncompressed Textures (WebGL2)

```ts
// WebGL2 backend
gl.texImage2D(GL.TEXTURE_2D, 0, GL.SRGB8_ALPHA8, width, height, 0, GL.RGBA, GL.UNSIGNED_BYTE, pixels)
gl.generateMipmap(GL.TEXTURE_2D)
```

#### Uncompressed Textures (WebGPU)

```ts
device.queue.writeTexture(
  { texture: gpuTexture },
  pixels,
  { bytesPerRow: width * 4 },
  { width, height }
)
// Mipmap generation via blit chain or compute shader
```

#### Compressed Textures (WebGL2)

```ts
for (let level = 0; level < mipLevels.length; level++) {
  const mip = mipLevels[level]
  gl.compressedTexImage2D(GL.TEXTURE_2D, level, glInternalFormat, mip.width, mip.height, 0, mip.data)
}
```

#### Compressed Textures (WebGPU)

```ts
for (let level = 0; level < mipLevels.length; level++) {
  const mip = mipLevels[level]
  device.queue.writeTexture(
    { texture: gpuTexture, mipLevel: level },
    mip.data,
    { bytesPerRow: computeBytesPerRow(mip.width, format) },
    { width: mip.width, height: mip.height }
  )
}
```

### Texture Memory Budget

| Texture Type | 1024x1024 | 2048x2048 |
|--------------|-----------|-----------|
| RGBA8 (uncompressed) | 4 MB | 16 MB |
| BC7 | 1 MB | 4 MB |
| ASTC 4x4 | 1 MB | 4 MB |
| ETC2 | 1 MB | 4 MB |
| ETC1S (Basis) | ~0.5 MB | ~2 MB |
| With mipmaps | +33% | +33% |

KTX2/Basis textures use 4-8x less GPU memory than uncompressed, which is critical on mobile.

### Sampler Cache

Samplers are deduplicated by their descriptor. Most scenes use only 2-3 unique sampler configurations:

```ts
const samplerCache = new Map<string, GPUSamplerHandle>()

const getOrCreateSampler = (device: WrenDevice, desc: SamplerDescriptor): GPUSamplerHandle => {
  const key = `${desc.minFilter}_${desc.magFilter}_${desc.mipmapFilter}_${desc.wrapU}_${desc.wrapV}_${desc.maxAnisotropy}`
  let sampler = samplerCache.get(key)
  if (!sampler) {
    sampler = device.createSampler(desc)
    samplerCache.set(key, sampler)
  }
  return sampler
}
```

### Texture Disposal

```ts
const disposeTexture = (texture: WrenTexture, device: WrenDevice): void => {
  if (texture.gpuTexture) {
    device.destroyTexture(texture.gpuTexture)
    texture.gpuTexture = null
  }
  texture.source = null  // Release CPU-side data
}
```
