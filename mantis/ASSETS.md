# Asset Pipeline

## Overview

Mantis loads 3D models via glTF 2.0 and textures via standard image formats or
KTX2 (Basis Universal). Heavy decoding work (Draco mesh decompression, Basis
texture transcoding) runs in Web Workers to keep the main thread free for
rendering.

```
Asset loading pipeline:

  glTF file ──→ Parse JSON ──→ Extract binary buffers
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
              Draco decode     Image decode     KTX2 transcode
              (Worker)         (main thread     (Worker)
                               createImageBitmap)
                    │                │                │
                    └────────────────┼────────────────┘
                                     ▼
                          Y-up → Z-up transform
                                     │
                                     ▼
                          Build scene nodes:
                          Meshes, Skeletons,
                          Animations, BVH
                                     │
                                     ▼
                          Upload to GPU:
                          Buffers, Textures,
                          Pipelines (compile)
```

## glTF 2.0 Loader

### Supported Features

| Feature | Status |
|---|---|
| Meshes (indexed, non-indexed) | Full |
| Draco mesh compression | Full (via `KHR_draco_mesh_compression`) |
| Scenes and nodes | Full (hierarchy preserved) |
| Materials (PBR metallic-roughness) | Mapped to Lambert (see below) |
| Textures (PNG, JPEG, WebP) | Full |
| KTX2 textures | Full (via `KHR_texture_basisu`) |
| Skinning (joints + weights) | Full (up to 4 influences per vertex) |
| Skeletal animations | Full (translation, rotation, scale tracks) |
| Morph targets | Not supported (v1) |
| Cameras | Ignored (user provides camera) |
| Extensions | Draco, Basis, custom `_materialIndex` attribute |

### PBR → Lambert Conversion

glTF uses PBR metallic-roughness materials. Mantis maps these to Lambert:

```
glTF PBR                    →  Mantis Lambert
─────────────────────────────  ─────────────────────
baseColorFactor              →  color
baseColorTexture             →  colorMap
occlusionTexture             →  aoMap
emissiveFactor               →  palette emissive (if present)
metallicFactor               →  (ignored)
roughnessFactor              →  (ignored)
normalTexture                →  (ignored — Lambert has no normal mapping)
metallicRoughnessTexture     →  (ignored)
alphaMode: "BLEND"           →  transparent: true
alphaMode: "MASK"            →  transparent: true + alpha cutoff
doubleSided                  →  side: 'both'
```

This lossy conversion is intentional — Mantis targets stylized visuals where
Lambert is sufficient.

### Custom `_materialIndex` Attribute

Models authored for Mantis include a custom vertex attribute:

```json
{
  "meshes": [{
    "primitives": [{
      "attributes": {
        "POSITION": 0,
        "NORMAL": 1,
        "_materialIndex": 2
      }
    }]
  }]
}
```

The loader detects `_materialIndex` in the attribute map and includes it in the
geometry's vertex buffer. The attribute is a `Uint8Array` with one byte per
vertex, each value indexing into the material's palette.

### Y-up → Z-up Coordinate Conversion

glTF uses Y-up, right-handed coordinates. Mantis uses Z-up, right-handed. The
conversion is applied at load time — no runtime cost.

**Rotation matrix:**
```
┌ 1  0  0 ┐
│ 0  0 -1 │   (Y-up → Z-up: swap Y and Z, negate new Y)
└ 0  1  0 ┘
```

Applied to:
- All vertex positions: `[x, y, z]` → `[x, -z, y]`
- All vertex normals: same transform
- All animation translation keyframes: same transform
- All animation rotation keyframes: quaternion conjugation with the rotation
- All joint inverse bind matrices: pre-multiply with the rotation matrix
- Node transforms in the scene hierarchy: pre-multiply

This is a one-time O(n) pass over the data during loading. The transform is
baked into the vertex data and matrices — nothing is stored or applied at
runtime.

```typescript
const convertYUpToZUp = (positions: Float32Array) => {
  for (let i = 0; i < positions.length; i += 3) {
    const y = positions[i + 1]
    const z = positions[i + 2]
    positions[i + 1] = -z
    positions[i + 2] = y
  }
}

const convertQuatYUpToZUp = (quats: Float32Array) => {
  // Rotation quat for Y-up → Z-up: 90° around X axis
  // q_convert = [sin(45°), 0, 0, cos(45°)] = [0.7071, 0, 0, 0.7071]
  const cx = 0.7071067811865476
  const cw = 0.7071067811865476

  for (let i = 0; i < quats.length; i += 4) {
    const qx = quats[i], qy = quats[i + 1], qz = quats[i + 2], qw = quats[i + 3]
    // q_converted = q_convert × q_original × q_convert_inverse
    quats[i]     = cw * qx + cx * qw
    quats[i + 1] = cw * qy - cx * qz
    quats[i + 2] = cw * qz + cx * qy
    quats[i + 3] = cw * qw - cx * qx
  }
}
```

## Draco Mesh Decompression

Draco-compressed meshes are decoded in a Web Worker using the official
`draco3d` WASM decoder.

### Worker Architecture

```
Main Thread                           Worker Thread
─────────────                         ─────────────
loadGLTF(url)
  ├── parse JSON
  ├── detect Draco extension
  ├── postMessage({ type: 'decode',
  │     buffer: dracoBuffer })  ──→  onmessage
  │                                    ├── decoderModule.decode(buffer)
  │                                    ├── extract positions, normals,
  │                                    │   uvs, indices, _materialIndex
  │                                    └── postMessage({ positions, ... },
  │                                          [positions.buffer, ...])
  │                                          ↑ Transferable (zero-copy)
  ├── await response              ←──┘
  ├── apply Y→Z-up conversion
  └── create Geometry + upload to GPU
```

**Key details:**
- The Draco WASM module is loaded once and cached in the worker
- Decoded typed arrays are transferred (not copied) back to the main thread
  via `Transferable` — this is zero-copy for large meshes
- Multiple Draco meshes in a single glTF are decoded sequentially in the worker
  (Draco is not parallelizable within a single mesh, but multiple meshes could
  use multiple workers if needed)

### Worker Pool

Mantis maintains a pool of 2 workers:

```typescript
const workerPool = new WorkerPool(2)

// Worker 0: Draco decoder (persistent — WASM module stays loaded)
// Worker 1: Basis transcoder (persistent — WASM module stays loaded)
```

Two workers is the sweet spot: enough to decode mesh + texture in parallel,
few enough to avoid mobile device resource pressure.

## KTX2 / Basis Universal Texture Transcoding

KTX2 containers with Basis Universal supercompressed textures are transcoded
to the optimal GPU-native compressed format for the current device.

### Format Selection

```typescript
const selectBasisTarget = (device: GALDevice): BasisFormat => {
  const gl = device.limits

  // Prefer ASTC (mobile) → BC7 (desktop) → ETC2 (older mobile) → RGBA8 (fallback)
  if (gl.astc)  return BasisFormat.ASTC_4x4     // 8 bpp, excellent quality
  if (gl.bc7)   return BasisFormat.BC7           // 8 bpp, desktop
  if (gl.s3tc)  return BasisFormat.BC1           // 4 bpp, desktop (lower quality)
  if (gl.etc2)  return BasisFormat.ETC2_RGBA     // 8 bpp, older Android
  return BasisFormat.RGBA8                        // 32 bpp, uncompressed fallback
}
```

### Transcoding Pipeline

```
Main Thread                           Worker Thread
─────────────                         ─────────────
loadKTX2(url)
  ├── fetch KTX2 file
  ├── detect supercompression type
  ├── postMessage({ type: 'transcode',
  │     ktx2Data, targetFormat })  ──→  onmessage
  │                                       ├── basisModule.transcode(
  │                                       │     ktx2Data, targetFormat)
  │                                       └── postMessage({ mipmaps },
  │                                             [mipmaps[0].buffer, ...])
  ├── await response              ←──────┘
  └── device.writeTexture(...)
```

Basis Universal supports all target formats from a single source — no need to
ship separate texture files for each platform.

### Mipmap Handling

KTX2 files can contain pre-computed mipmaps. If present, all levels are
transcoded and uploaded. If absent, Mantis generates mipmaps on the GPU after
uploading the base level:

- WebGPU: compute shader mipmap generation (fast)
- WebGL2: `gl.generateMipmap()` (driver-implemented)

## Texture Loading (Non-KTX2)

Standard image textures (PNG, JPEG, WebP) are loaded via `createImageBitmap`
for off-main-thread decoding:

```typescript
const loadTexture = async (url: string, device: GALDevice): Promise<GALTexture> => {
  const response = await fetch(url)
  const blob = await response.blob()
  const bitmap = await createImageBitmap(blob, { colorSpaceConversion: 'none' })

  const texture = device.createTexture({
    width: bitmap.width,
    height: bitmap.height,
    format: 'rgba8unorm',
    usage: 'sampled',
    generateMipmaps: true,
  })

  device.writeTexture(texture, bitmap)
  bitmap.close()
  return texture
}
```

`createImageBitmap` decodes JPEG/PNG off the main thread in all modern browsers.

## Loader API

```typescript
// Load a glTF model
const model = await loadGLTF(engine, '/models/character.glb')
// Returns: { meshes, skeleton?, animations?, scene }

// Add to scene
scene.root.add(model.scene)

// Access specific meshes
const body = model.meshes.find(m => m.name === 'Body')

// Play animations (if present)
const mixer = new AnimationMixer(model.skeleton)
mixer.play(model.animations[0])

// Load a texture independently
const tex = await loadTexture(engine, '/textures/ground.ktx2')
const material = new LambertMaterial({ colorMap: tex })
```

### Loading Options

```typescript
const model = await loadGLTF(engine, '/models/scene.glb', {
  // Override material for all meshes
  material: myCustomMaterial,

  // Pre-build mesh BVH for raycasting
  buildBVH: true,

  // Pre-compile shader variants for this model's feature set
  precompileShaders: true,

  // Scale factor applied at load time
  scale: 0.01,  // e.g., if model was authored in centimeters
})
```

## Geometry Construction

After decoding, vertex data is assembled into an interleaved vertex buffer for
optimal GPU access:

```
Interleaved layout (32 bytes per vertex, cache-line aligned):
┌────────────┬────────────┬────────┬──────────────┐
│ position   │ normal     │ uv     │ extra        │
│ float32×3  │ float32×3  │ float32×2 │ (color/idx)│
│ 12 bytes   │ 12 bytes   │ 8 bytes│ varies       │
└────────────┴────────────┴────────┴──────────────┘
```

If vertex colors or material indices are present, the stride extends to
accommodate them. The layout is determined at geometry creation and does not
change.

Index buffers use `Uint16Array` for meshes with < 65,536 vertices (most cases)
and `Uint32Array` for larger meshes. 16-bit indices halve index bandwidth.

## Memory Management

- GPU buffers (vertex, index, texture) are created during loading and persist
  until explicitly disposed
- CPU-side typed arrays (positions, normals, etc.) are released after GPU upload
  unless the user requests them retained (for CPU raycasting or modification)
- Worker WASM modules stay resident — the 1–2 MB memory cost is amortized
  across all loads
- `model.dispose()` frees all GPU resources (buffers, textures, pipelines)
  associated with a loaded model
