# Asset Pipeline

## Overview

Kestrel supports GLTF 2.0 as its primary asset format, with Draco mesh compression and KTX2/Basis Universal texture compression. All heavy decoding runs in Web Workers to keep the main thread responsive.

## GLTF Loader

### Supported Features

| Feature | Status |
|---|---|
| glTF 2.0 JSON (.gltf) | Supported |
| glTF Binary (.glb) | Supported |
| Meshes (indexed, non-indexed) | Supported |
| Multiple primitives per mesh | Supported |
| Node hierarchy | Supported |
| Skeletal animation (skins) | Supported |
| Animation clips | Supported |
| PBR materials → Lambert conversion | Supported |
| Vertex colors | Supported |
| `_materialindex` vertex attribute | Supported (custom) |
| Draco compression (KHR_draco_mesh_compression) | Supported |
| KTX2 textures (KHR_texture_basisu) | Supported |
| Morph targets | Not supported |
| Cameras | Not supported (use Kestrel cameras) |
| Extensions: KHR_materials_unlit | Supported → BasicMaterial |

### Coordinate System Conversion

GLTF uses Y-up, right-handed. Kestrel uses Z-up, right-handed. On import, a root rotation is applied:

```
Y-up to Z-up rotation matrix:
[1  0  0  0]
[0  0  1  0]     // Y → Z
[0 -1  0  0]     // Z → -Y
[0  0  0  1]
```

This rotation is applied to:
- All mesh vertex positions and normals
- All node transforms
- All animation position channels
- All skeleton bind poses

The rotation is baked into the data at load time — no runtime cost.

### Material Mapping

GLTF PBR materials are converted to Kestrel materials:

| GLTF PBR Property | Kestrel Mapping |
|---|---|
| `baseColorFactor` | `LambertMaterial.color` |
| `baseColorTexture` | `LambertMaterial.colorTexture` |
| `occlusionTexture` | `LambertMaterial.aoTexture` |
| `emissiveFactor` | Material index emissive (if `_materialindex` present) |
| `KHR_materials_unlit` | `BasicMaterial` |
| `alphaMode: "BLEND"` | `transparent: true` |
| `alphaMode: "MASK"` | Alpha test (discard below cutoff) |
| `doubleSided: true` | `side: 'both'` |

PBR-specific properties (metallic, roughness, normal maps) are ignored — Kestrel is Lambert-only by design.

### Custom: `_materialindex` Vertex Attribute

Models can include a custom vertex attribute `_MATERIALINDEX` (accessor of type SCALAR, componentType UNSIGNED_BYTE or UNSIGNED_SHORT) in their mesh primitives:

```json
{
  "meshes": [{
    "primitives": [{
      "attributes": {
        "POSITION": 0,
        "NORMAL": 1,
        "_MATERIALINDEX": 2
      }
    }]
  }]
}
```

The GLTF loader reads this attribute and passes it through as a vertex buffer attribute. The material's `materialIndexMap` then maps each index to a color and emissive value.

### Loading API

```typescript
const loader = new GLTFLoader()

// Load a model
const result = await loader.load('/models/character.glb')

// Result structure
interface GLTFResult {
  scene: Group                    // root node hierarchy
  meshes: Mesh[]                  // all meshes (flat list)
  skinnedMeshes: SkinnedMesh[]   // skinned meshes
  animations: AnimationClip[]    // all animation clips
  skeletons: Skeleton[]          // all skeletons
}

// Add to scene
scene.add(result.scene)

// Play an animation
const mixer = result.skinnedMeshes[0].mixer
mixer.play(result.animations.find(a => a.name === 'walk')!, { loop: true })
```

### Loading Flow

```
GLTFLoader.load(url):
  1. Fetch .glb/.gltf file
  2. Parse JSON + extract binary buffers
  3. Apply Y→Z coordinate conversion to all transforms
  4. For each mesh primitive:
     a. If Draco-compressed → send to DracoDecoder worker
     b. Otherwise → read vertex/index data directly from buffers
     c. Read _materialindex attribute if present
     d. Create Geometry from vertex data
  5. For each material:
     a. Map GLTF PBR properties to Kestrel material
     b. If textures use KTX2 → send to BasisDecoder worker
     c. Otherwise → load images directly
  6. For each skin:
     a. Build Skeleton from joint hierarchy
     b. Read inverse bind matrices
     c. Create SkinnedMesh binding skeleton to mesh
  7. For each animation:
     a. Create AnimationClip with channels targeting bone indices
  8. Build node hierarchy as Group/Mesh/SkinnedMesh tree
  9. Return GLTFResult
```

## Draco Decoder

Draco is a mesh compression format that typically achieves 10–20× compression ratios on geometry data.

### Architecture

```
Main Thread                          Worker Thread
─────────────                        ─────────────
GLTFLoader detects                   DracoDecoder
KHR_draco_mesh_compression           ├─ Loads draco_decoder.wasm (once)
      │                              ├─ Receives compressed buffer
      ├── postMessage(buffer) ──────►├─ Decodes positions, normals, uvs, indices
      │                              ├─ Decodes custom attributes (_materialindex)
      │◄── postMessage(result) ──────├─ Transfers decoded ArrayBuffers back
      │    (Transferable)            └─ (zero-copy transfer)
      ▼
Creates Geometry from
decoded vertex data
```

### Implementation

```typescript
// draco.worker.ts
import { DracoDecoderModule } from '@aspect/draco3d'

let decoderModule: DracoDecoderModule

self.onmessage = async (event: MessageEvent) => {
  const { id, buffer, attributes } = event.data

  if (!decoderModule) {
    decoderModule = await createDracoDecoder()
  }

  const decoder = new decoderModule.Decoder()
  const decoderBuffer = new decoderModule.DecoderBuffer()
  decoderBuffer.Init(new Int8Array(buffer), buffer.byteLength)

  const geometryType = decoder.GetEncodedGeometryType(decoderBuffer)
  const mesh = new decoderModule.Mesh()
  decoder.DecodeBufferToMesh(decoderBuffer, mesh)

  // Extract standard attributes
  const positions = extractAttribute(decoder, mesh, decoderModule.POSITION)
  const normals = extractAttribute(decoder, mesh, decoderModule.NORMAL)
  const uvs = extractAttribute(decoder, mesh, decoderModule.TEX_COORD)
  const indices = extractIndices(decoder, mesh)

  // Extract custom attributes (like _materialindex)
  const customAttrs: Record<string, Float32Array> = {}
  for (const [name, id] of Object.entries(attributes)) {
    const attr = decoder.GetAttributeByUniqueId(mesh, id)
    if (attr) customAttrs[name] = extractAttribute(decoder, mesh, attr)
  }

  // Transfer buffers (zero-copy)
  const transferables = [positions.buffer, normals.buffer, uvs.buffer, indices.buffer,
    ...Object.values(customAttrs).map(a => a.buffer)]

  self.postMessage({ id, positions, normals, uvs, indices, customAttrs }, transferables)
}
```

### WASM Source

The Draco decoder WASM is bundled from Google's official `draco3d` npm package (~300KB). It's loaded lazily on first use.

## Basis / KTX2 Decoder

KTX2 is a container format for Basis Universal compressed textures. Basis can be transcoded to any GPU compressed format at runtime, giving:
- Small file sizes (Basis compression)
- Native GPU decompression (no CPU decode at render time)
- Cross-platform support (different GPUs support different formats)

### Format Selection

The decoder selects the optimal GPU format based on device capabilities:

| GPU | Transcoded Format | Notes |
|---|---|---|
| Desktop (NVIDIA/AMD/Intel) | BC7 | Highest quality, universally supported |
| Apple Silicon (Mac/iPhone) | ASTC 4×4 | Native, best quality on Apple |
| Android (Adreno/Mali) | ASTC 4×4 | Widely supported on modern Android |
| Android (older) | ETC2 | Fallback for older devices |
| WebGL2 (no compressed) | RGBA8 | Full decompress (last resort) |

Format detection:

```typescript
const detectBestFormat = (device: Device): BasisFormat => {
  if (device.capabilities.astc) return 'astc-4x4-unorm'
  if (device.capabilities.bc7) return 'bc7-rgba-unorm'
  if (device.capabilities.etc2) return 'etc2-rgba8unorm'
  return 'rgba8unorm'  // fallback: full decompress
}
```

### Worker Architecture

```
Main Thread                          Worker Thread
─────────────                        ─────────────
GLTFLoader or                        BasisDecoder
TextureLoader detects .ktx2          ├─ Loads basis_transcoder.wasm (once)
      │                              ├─ Receives KTX2 buffer + target format
      ├── postMessage({              ├─ Parses KTX2 header
      │     buffer,                  ├─ Transcodes Basis data → target format
      │     format                   ├─ Handles mipmaps if present
      │   }) ──────────────────────► ├─ Transfers compressed ArrayBuffer back
      │                              └─ (zero-copy transfer)
      │◄── postMessage(result) ──────┘
      ▼
Creates GpuTexture from
compressed data (uploaded
directly to GPU — no CPU
decompression needed)
```

### Loading API

```typescript
// Direct KTX2 texture loading
const texture = await TextureLoader.loadKTX2('/textures/brick.ktx2')

// Or as part of GLTF (automatic)
const gltf = await gltfLoader.load('/models/scene.glb')
// KTX2 textures in the GLTF are automatically decoded
```

### WASM Source

The Basis transcoder WASM is from Binomial's official `basis_universal` builds (~200KB). Loaded lazily on first KTX2 texture request.

## Resource Management

### Caching

Loaded resources are cached by URL to prevent duplicate loads:

```typescript
const cache = new Map<string, Promise<any>>()

const load = async (url: string) => {
  if (cache.has(url)) return cache.get(url)
  const promise = doLoad(url)
  cache.set(url, promise)
  return promise
}
```

### Disposal

```typescript
// Manual disposal
gltfResult.scene.traverse(obj => {
  if (obj instanceof Mesh) {
    obj.geometry.dispose()   // releases GPU vertex/index buffers
    obj.material.dispose()   // releases GPU uniform buffer, bind group
  }
})

// In React: automatic on unmount
```

Textures use reference counting — disposed only when no material references them.

## Progress and Error Handling

```typescript
const gltf = await loader.load('/models/big_scene.glb', {
  onProgress: (loaded, total) => {
    console.log(`${Math.round(loaded / total * 100)}%`)
  },
  onError: (error) => {
    console.error('Failed to load:', error)
  }
})
```
