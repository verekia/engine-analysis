# ASSETS.md - Asset Loading Pipeline

## Primary Format

**Decision: glTF 2.0 as the only 3D model format** (universal agreement)

- `.gltf` (JSON) + `.bin` (binary) or `.glb` (single-file binary) supported
- Full support for meshes, materials, textures, hierarchies, animations, skeletons
- All PBR materials are converted to Lambert at import time (intentional lossy downgrade)
- Custom `_materialindex` vertex attribute recognized automatically

## Compression Support

### Draco Mesh Compression

**Decision: Worker pool for parallel decoding** (Fennec, Hyena, Lynx, Mantis approach)

2-4 workers based on `navigator.hardwareConcurrency`. Parallel decoding is 2-3x faster for models with 10+ Draco-compressed meshes.

```typescript
const gltf = await loadGLTF(url, device, {
  draco: { wasmUrl: '/draco_decoder.wasm' },
})
```

**Why worker pool over single worker (Caracal, Rabbit, Shark, Wren)?**

A single worker is adequate for simple models (<10 meshes), but the target games may load complex scenes with many compressed meshes. Worker pool scales automatically and the implementation overhead is modest:
- Pool of 2-4 workers: ~8MB memory
- Task queue with round-robin dispatch
- Workers initialized lazily on first Draco request
- Workers can be terminated when idle

### KTX2/Basis Universal Textures

**Decision: Priority format detection** (universal agreement)

All implementations use the same priority order based on device capabilities:
1. ASTC 4x4 (mobile: iOS, modern Android, Apple Silicon)
2. BC7 (desktop: Windows, Linux, Intel Macs)
3. ETC2 (older Android, WebGL2 fallback)
4. RGBA8 (uncompressed fallback)

Format probed via `device.features` (WebGPU) or WebGL extension checks.

**KTX2 transcoding** also runs in Web Workers with WASM (Basis Universal transcoder).

### WASM Module Loading

**Decision: User provides WASM URLs** (Caracal, Fennec, Lynx approach - best for CDN distribution)

```typescript
const gltf = await loadGLTF(url, device, {
  draco: { wasmUrl: 'https://cdn.example.com/draco_decoder.wasm' },
  ktx2: { wasmUrl: 'https://cdn.example.com/basis_transcoder.wasm' },
})
```

Modules are loaded lazily on first use and cached for subsequent loads. This keeps the main bundle small (~0 bytes for decoders until needed) and lets users host WASM files on their own CDN.

## Material Mapping

**Decision: PBR to Lambert with unlit extension support** (7/8 implementations agree)

| glTF Property | Engine Property |
|--------------|-----------------|
| `baseColorFactor` | `LambertMaterial.color` |
| `baseColorTexture` | `LambertMaterial.map` |
| `occlusionTexture` | `LambertMaterial.aoMap` |
| `emissiveFactor` | Palette emissive entry |
| `KHR_materials_unlit` | Switch to `BasicMaterial` |
| `alphaMode: "BLEND"` | `material.transparent = true` |
| `alphaMode: "MASK"` | `material.alphaTest = alphaCutoff` |
| `doubleSided: true` | `material.side = 'double'` |
| `metallicFactor`, `roughnessFactor` | Ignored |
| `normalTexture` | Ignored |

Normal maps and PBR-specific textures are intentionally dropped. Lambert does not use them, and the target aesthetic does not need them.

## Y-up to Z-up Conversion

**Decision: Vertex data conversion at load time** (Lynx, Mantis, Shark, Wren approach)

Convert vertex positions, normals, and animation keyframes directly during import:
- Swap Y and Z components
- Negate the new Y (to maintain handedness)
- Apply to all position, normal, and animation track data

### Why Vertex Conversion Over Root Transform (Caracal, Fennec, Hyena, Rabbit)?

Root transform (apply -90deg X rotation to root node) is simpler to implement but:
- Leaves animation data in glTF space, which causes subtle bugs with custom bone queries
- Adds an extra node to the scene graph
- Mixes coordinate spaces (some data in Y-up, some in Z-up)

Baking the conversion into vertex data at load time is a one-time cost and eliminates coordinate space confusion for the rest of the engine's lifetime.

## Return Type

**Decision: Flat lists + scene hierarchy** (8/8 active implementations agree)

```typescript
interface GLTFResult {
  scene: Group                    // Root node with full hierarchy
  meshes: Mesh[]                  // Flat list for direct access
  skinnedMeshes: SkinnedMesh[]    // Flat list
  animations: AnimationClip[]     // Flat list
  skeletons: Skeleton[]           // Flat list
  dispose(): void                 // Cleanup all GPU resources
}
```

Users typically need both: the scene hierarchy for `scene.add(gltf.scene)` and the flat lists for finding specific meshes or starting animations.

## Caching

**Decision: URL-based promise cache** (Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit approach)

```typescript
const cache = new Map<string, Promise<GLTFResult>>()
```

Same URL returns the same Promise. Prevents duplicate network requests and duplicate GPU uploads when multiple React components load the same model.

Texture deduplication is also applied: if two materials reference the same texture URI, only one GPU texture is created.

## Disposal

**Decision: Explicit disposal required, React auto-disposes on unmount** (universal agreement)

```typescript
// Imperative
const gltf = await loadGLTF(url, device)
// ... later
gltf.dispose()  // Frees GPU buffers, textures, materials
```

In React bindings, `<primitive object={gltf.scene} />` automatically disposes the GLTF result when the component unmounts. This prevents GPU memory leaks.

## Texture Loading

### Standard Images (PNG, JPEG, WebP)

Use `createImageBitmap()` for off-main-thread decoding. This is universally supported and avoids blocking the render loop.

### Mipmap Generation

- **KTX2 files**: Pre-computed mipmaps included in the file - upload all levels directly
- **Standard images**: Generate mipmaps on the GPU after upload
  - WebGPU: Compute shader mipmap generation
  - WebGL2: `gl.generateMipmap()`

### Sampler Caching

Samplers are cached by their configuration (min/mag filter, wrap mode). Typical scenes use 2-3 unique samplers.

## Performance Characteristics

| Phase | Single Worker | Worker Pool (4) |
|-------|--------------|-----------------|
| Fetch + JSON parse | ~100ms | ~100ms |
| Draco decode (50 meshes) | ~400ms | ~150ms |
| KTX2 transcode (20 textures) | ~300ms | ~120ms |
| GPU upload | ~50ms | ~50ms |
| **Total** | **~850ms** | **~420ms** |

Worker pool provides ~2x speedup for complex models.

## Memory Budget

| Resource | Size |
|----------|------|
| Draco WASM | ~300KB |
| Basis WASM | ~200KB |
| Worker pool (4 workers) | ~8MB |
| ASTC texture (1024x1024) | ~1MB GPU |
| Uncompressed texture (1024x1024) | ~4MB GPU |

WASM modules are loaded once per application lifecycle and shared across all loads.
