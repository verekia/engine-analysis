# Asset Loading Comparison

This document compares asset loading pipelines across all 9 engine implementations.

## Universal Agreement

All implementations agree on:

### Primary Format
- **glTF 2.0** as the primary 3D model format (.gltf + .bin or .glb binary)
- Full support for meshes, materials, textures, hierarchies, animations, and skeletons
- Y-up to Z-up coordinate conversion applied at load time (no runtime cost)

### Compression Support
- **Draco mesh compression** (`KHR_draco_mesh_compression` extension)
- **KTX2/Basis Universal texture compression** (`KHR_texture_basisu` extension)
- Both decoders run in Web Workers via WASM modules to avoid blocking the main thread

### Decoding Strategy
- **Off-main-thread decoding**: Heavy work (Draco, Basis) in Web Workers with transferable ArrayBuffers
- **WASM modules loaded lazily**: First load initializes decoder, subsequent loads reuse it
- **Parallel processing**: Multiple meshes/textures decoded concurrently across worker pool

### Custom Attributes
- **`_materialindex` vertex attribute**: All implementations recognize this custom glTF attribute for material index systems

### Texture Loading
- **Standard images** (PNG, JPEG, WebP): `createImageBitmap()` for off-main-thread decode
- **Mipmap generation**: Automatic (GPU) or pre-provided in KTX2
- **Texture caching**: Deduplicate textures referenced by multiple materials

---

## Key Variations

### 1. Implementation Status

**Minimal (1 implementation):**
- **Bonobo**: Manual vertex/index array import only in v1, glTF loader deferred to v2

**Full implementation (8 implementations):**
- Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren all have production-ready glTF loaders

### 2. Material Mapping Strategy

**PBR → Lambert (7 implementations):**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark**: Map `baseColorFactor` → Lambert color, `baseColorTexture` → color map
- Ignore metallic/roughness (not needed for stylized rendering)

**PBR → Lambert with approximation (1 implementation):**
- **Wren**: Maps `baseColorFactor` to Lambert, adjusts brightness based on `metallicFactor`

**Minimal mapping (1 implementation):**
- **Bonobo**: No material mapping (deferred to v2)

### 3. Draco Worker Pool Size

**Single worker (4 implementations):**
- **Caracal, Rabbit, Shark, Wren**: One persistent Draco worker
- Sequential decoding of multiple meshes

**Worker pool (4 implementations):**
- **Fennec, Hyena, Lynx, Mantis**: 2-4 workers based on `navigator.hardwareConcurrency`
- Parallel decoding of multiple meshes
- Faster for models with many Draco-compressed primitives

### 4. KTX2 Target Format Selection Priority

**All implementations agree on priority order:**
1. **ASTC 4×4** (mobile: iOS, modern Android, Apple Silicon)
2. **BC7** (desktop: Windows, Linux, Intel Macs)
3. **ETC2** (older Android, WebGL2 fallback)
4. **RGBA8** (uncompressed fallback)

**Differences:**
- **S3TC/BC1-BC3 fallback**: Some implementations (Caracal, Lynx, Mantis) check for S3TC before ETC2 on older desktop GPUs
- **Format probing**: All implementations query device capabilities (`GPUAdapter.features` or WebGL extensions)

### 5. Basis Transcoder Architecture

**Texture array preference (6 implementations):**
- **Fennec, Hyena, Lynx, Mantis, Shark, Wren**: Use texture arrays when available (cleaner shader code)

**Atlas fallback (2 implementations):**
- **Caracal, Rabbit**: Pack cascades/levels into single 2D atlas if texture arrays unavailable

### 6. Y-up to Z-up Conversion Approach

**Root node rotation (4 implementations):**
- **Caracal, Fennec, Hyena, Rabbit**: Apply -90° X-axis rotation to root group node
- Child transforms remain in glTF space, rotation propagates through hierarchy

**Vertex data conversion (4 implementations):**
- **Lynx, Mantis, Shark, Wren**: Convert vertex positions, normals, and animation keyframes directly
- Baked into geometry data at load time

**Hybrid (1 implementation):**
- **Wren**: Both approaches mentioned in docs (root transform + vertex conversion for some cases)

### 7. glTF Return Type Structure

**Flat lists + scene hierarchy (8 implementations):**
```typescript
interface GLTFResult {
  scene: Group                    // Root node
  meshes: Mesh[]                  // Flat list
  skinnedMeshes: SkinnedMesh[]    // Flat list
  animations: AnimationClip[]     // Flat list
  skeletons: Skeleton[]           // Flat list
}
```

**Multiple scenes support (3 implementations):**
- **Lynx, Rabbit, Shark**: Also return `scenes: Group[]` array for glTF files with multiple scenes

### 8. Asset Caching Strategy

**URL-based cache (6 implementations):**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit**: `Map<string, Promise<GLTFResult>>`
- Same URL returns same Promise (prevents duplicate loads)

**Geometry/texture deduplication (8 implementations):**
- All implementations share GPU buffers and textures when primitives reference the same data
- Multiple `Mesh` instances share single `Geometry` object

**No automatic caching (2 implementations):**
- **Shark, Wren**: User responsible for caching if needed

### 9. Disposal API

**Explicit disposal required (8 implementations):**
```typescript
gltfResult.dispose()  // or
gltfResult.meshes.forEach(m => m.dispose())
```
- No automatic garbage collection of GPU resources
- User must manually dispose when assets no longer needed

**React bindings auto-dispose (2 implementations):**
- **Hyena, Fennec**: React components automatically dispose on unmount

---

## Implementation Breakdown

### By Feature Completeness

| Feature | Bonobo | Caracal | Fennec | Hyena | Lynx | Mantis | Rabbit | Shark | Wren |
|---------|--------|---------|--------|-------|------|--------|--------|-------|------|
| glTF 2.0 JSON | ❌ v2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| GLB binary | ❌ v2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Draco compression | ❌ v2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| KTX2/Basis | ❌ v2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Material mapping | ❌ v2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Animation import | ❌ v2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Skeleton import | ❌ v2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Worker pool | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |

### By Compression Support Quality

**Production-grade (8 implementations):**
- All active implementations have robust Draco and KTX2 support
- WASM modules loaded from official sources (Google Draco, Binomial Basis)
- Proper error handling and fallback to uncompressed

**Best worker utilization:**
- **Fennec, Hyena, Lynx, Mantis**: Worker pools scale with CPU cores
- Parallel decode of complex models with many primitives

### By Texture Memory Efficiency

**All implementations achieve 4-8× texture compression:**
- RGBA8 uncompressed: 4 MB per 1024×1024 texture
- BC7/ASTC compressed: 1 MB per 1024×1024 texture
- Critical for mobile GPU memory budgets

**Mipmap handling:**
- **Pre-computed mipmaps** (KTX2): All implementations upload all levels
- **Runtime generation**: WebGPU compute shader (Fennec, Lynx, Mantis, Wren) or `gl.generateMipmap()` (all WebGL2 backends)

### By Material Mapping Sophistication

**Simple mapping (7 implementations):**
- `baseColorFactor` → `LambertMaterial.color`
- `baseColorTexture` → `LambertMaterial.map`
- `occlusionTexture` → `LambertMaterial.aoMap`
- `emissiveFactor` → `Material.emissive` or palette emissive

**Unlit extension support (6 implementations):**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Wren**: `KHR_materials_unlit` → `BasicMaterial`

**Alpha mode support (8 implementations):**
- `alphaMode: "BLEND"` → transparent material
- `alphaMode: "MASK"` → alpha cutoff discard
- `doubleSided: true` → render both faces

### By Coordinate Conversion Strategy

**Root transform (pros: simple, cons: animation in glTF space):**
- **Caracal, Fennec, Hyena, Rabbit**: One rotation at root, minimal code

**Vertex conversion (pros: fully baked, cons: more complex):**
- **Lynx, Mantis, Shark, Wren**: Convert positions, normals, animation tracks
- No runtime coordinate space mixing

---

## Performance Characteristics

### Loading Times (example: 5MB glTF with 50 meshes, 20 textures)

| Phase | Single Worker | Worker Pool | Notes |
|-------|---------------|-------------|-------|
| Fetch + parse | ~100ms | ~100ms | Network + JSON parsing |
| Draco decode | ~400ms | ~150ms | Worker pool 3× faster |
| KTX2 transcode | ~300ms | ~120ms | Worker pool 2.5× faster |
| GPU upload | ~50ms | ~50ms | Not parallelizable |
| **Total** | **~850ms** | **~420ms** | Worker pool 2× faster overall |

**Worker pool benefit scales with asset complexity:**
- Single mesh: no difference
- 10+ Draco meshes: significant speedup
- 50+ meshes: ~2-3× faster with worker pool

### Memory Overhead

**WASM modules (loaded once per app lifecycle):**
- Draco decoder: ~300 KB
- Basis transcoder: ~200 KB
- Total: ~500 KB (amortized across all loads)

**Worker memory:**
- Each worker: ~2 MB (WASM instance + buffers)
- Pool of 4 workers: ~8 MB
- Released when idle (worker can be terminated)

### Texture Compression Ratios (1024×1024 texture)

| Format | File Size (KTX2) | GPU Memory | Quality |
|--------|------------------|------------|---------|
| RGBA8 | ~2 MB | 4 MB | Perfect |
| BC7 | ~0.8 MB | 1 MB | Excellent |
| ASTC 4×4 | ~0.8 MB | 1 MB | Excellent |
| ETC2 | ~0.8 MB | 1 MB | Good |
| ETC1S (Basis) | ~0.4 MB | 1 MB | Good (slight artifacts) |

**All implementations prioritize ASTC/BC7 for best quality.**

---

## Cherry-Picking Recommendations

### For Fastest Loading (Complex Models)
**Choose**: Fennec, Hyena, Lynx, or Mantis worker pool approach
- Parallel Draco decoding across CPU cores
- 2-3× faster for models with 10+ compressed meshes
- Scales with device capabilities (`navigator.hardwareConcurrency`)

### For Simplest Integration
**Choose**: Rabbit or Shark single-worker approach
- Minimal complexity: one Draco worker, one Basis worker
- Adequate for most use cases (models with <10 meshes)
- Lower memory overhead

### For Best Texture Quality
**All implementations equivalent:**
- All detect best available format (ASTC/BC7/ETC2)
- All transcode KTX2 correctly
- No significant quality differences

### For Memory Efficiency
**Choose**: Lynx or Mantis vertex conversion approach
- Fully baked coordinate conversion (no runtime overhead)
- Slightly smaller scene graph (no extra root rotation node)

**Choose**: Any implementation with texture deduplication
- All active implementations share textures across materials
- Critical for scenes with many material instances

### For React/Framework Integration
**Choose**: Hyena or Fennec React bindings
- Automatic disposal on component unmount
- Prevents GPU memory leaks
- Simpler resource lifecycle management

### For Custom glTF Extensions
**All implementations support custom attributes:**
- `_materialindex` universally recognized
- Easy to extend loaders for additional custom data
- Draco preserves custom attributes (all implementations handle this)

### For Production Robustness
**All active implementations are production-ready:**
- Proper error handling (network, parsing, decode failures)
- Fallback to uncompressed if Draco/Basis unavailable
- Tested on WebGL2 and WebGPU backends

**Standout:**
- **Caracal**: Most detailed documentation of worker protocol
- **Lynx**: Most comprehensive loader options (material overrides, BVH building)
- **Fennec**: Best React integration
- **Mantis**: Clearest memory management examples

### For Minimal Bundle Size
**Choose**: Implementations with optional decoder paths
- **Caracal, Fennec, Lynx**: Allow custom WASM URLs
- Load decoders from CDN instead of bundling
- Reduces initial bundle size by ~500 KB

### For WebGL2 vs WebGPU Parity
**All implementations have backend parity:**
- Same loader works for both backends
- Format selection adapts to device capabilities
- No WebGPU-specific loading advantages (yet)

**Note**: WebGPU has cleaner texture array API, but functionality is identical.

---

## Known Limitations

**Morph targets (blend shapes):**
- **Not supported** in v1 by any implementation
- Deferred to v2 (Bonobo, Caracal, Hyena) or not mentioned
- Rare in stylized/low-poly games

**Cameras:**
- **Ignored** by most implementations (6/8 use engine cameras instead)
- **Basic support** in Lynx (imports but doesn't use)

**Lights:**
- **Not imported** from glTF (all engines use scene-defined lights)
- glTF light extensions (`KHR_lights_punctual`) not supported

**PBR materials:**
- **Converted to Lambert** (lossy but intentional)
- Normal maps, metallic/roughness textures discarded
- Acceptable for target aesthetic (low-poly, stylized)

---

## Implementation Maturity Assessment

**Production-ready (8 implementations):**
- All active implementations handle complex real-world glTF files
- Draco and KTX2 support is robust and well-tested
- Performance is excellent (sub-second loads for typical game assets)

**Deferred to v2 (1 implementation):**
- Bonobo intentionally minimal in v1 (manual geometry import only)

**Best overall:**
- **Fennec**: Best React integration + worker pool
- **Lynx**: Most configuration options + vertex conversion
- **Mantis**: Clearest docs + memory management examples
- **Caracal**: Most detailed worker protocol documentation

**No significant gaps** across active implementations. All are suitable for production use with glTF 2.0 assets.
