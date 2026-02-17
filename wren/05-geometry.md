# 05 — Geometry, Parametric Primitives, GLTF & Draco

## Geometry Container

A `Geometry` is a collection of typed-array vertex attributes and an optional index buffer, stored in a format ready for direct GPU upload.

```ts
interface Geometry {
  id: number
  attributes: Map<string, VertexAttribute>
  index: Uint16Array | Uint32Array | null
  vertexCount: number
  indexCount: number
  boundingBox: AABB
  boundingSphere: { center: Float32Array, radius: number }

  // GPU handles (created on first use)
  gpuVertexBuffers: GPUBufferHandle[] | null
  gpuIndexBuffer: GPUBufferHandle | null

  // BVH (built on demand for raycasting)
  bvh: FlatBVH | null
}

interface VertexAttribute {
  name: string                     // 'position', 'normal', 'uv', 'color', '_materialindex', etc.
  data: Float32Array | Uint8Array | Uint16Array | Uint32Array
  itemSize: number                 // Components per vertex (3 for position, 2 for uv, etc.)
  normalized: boolean
  gpuBuffer: GPUBufferHandle | null
}
```

### Standard Attribute Names

| Name | Item Size | Type | Description |
|------|-----------|------|-------------|
| `position` | 3 | Float32 | Vertex position (x, y, z) |
| `normal` | 3 | Float32 | Vertex normal |
| `uv` | 2 | Float32 | Primary UV coordinates |
| `uv2` | 2 | Float32 | Secondary UV (for AO maps) |
| `color` | 3 or 4 | Float32 or Uint8 | Vertex color (RGB or RGBA) |
| `_materialindex` | 1 | Uint8 or Uint16 | Material index (see [04-materials.md](04-materials.md)) |
| `joints` | 4 | Uint8 or Uint16 | Bone indices for skinning |
| `weights` | 4 | Float32 | Bone weights for skinning |

### Vertex Buffer Layout

Attributes are stored in **separate buffers** (struct-of-arrays), not interleaved. This is optimal for:
- Partial updates (e.g., only updating positions for morphing)
- Flexible vertex formats (different meshes have different attributes)
- GPU memory alignment

```
Buffer 0: [pos.x, pos.y, pos.z, pos.x, pos.y, pos.z, ...]  // positions
Buffer 1: [n.x, n.y, n.z, n.x, n.y, n.z, ...]              // normals
Buffer 2: [u, v, u, v, ...]                                   // UVs
Buffer 3: [idx, idx, idx, ...]                                 // material indices
```

## Parametric Geometries

All primitive generators return a `Geometry` with positions, normals, UVs, and indices. Z is up.

### Plane

```ts
const createPlaneGeometry = (options?: {
  width?: number        // Default: 1
  height?: number       // Default: 1
  widthSegments?: number  // Default: 1
  heightSegments?: number // Default: 1
}): Geometry
```

Lies in the XY plane, facing +Z (up). Grid of `(widthSegments+1) * (heightSegments+1)` vertices.

### Box

```ts
const createBoxGeometry = (options?: {
  width?: number        // X extent. Default: 1
  height?: number       // Y extent. Default: 1
  depth?: number        // Z extent. Default: 1
  widthSegments?: number
  heightSegments?: number
  depthSegments?: number
}): Geometry
```

Six faces, each a subdivided quad. Normals face outward.

### Sphere

```ts
const createSphereGeometry = (options?: {
  radius?: number           // Default: 0.5
  widthSegments?: number    // Default: 32
  heightSegments?: number   // Default: 16
  phiStart?: number         // Default: 0
  phiLength?: number        // Default: 2π
  thetaStart?: number       // Default: 0
  thetaLength?: number      // Default: π
}): Geometry
```

UV sphere with poles along Z axis.

### Cone

```ts
const createConeGeometry = (options?: {
  radius?: number           // Default: 0.5
  height?: number           // Default: 1
  radialSegments?: number   // Default: 32
  heightSegments?: number   // Default: 1
  openEnded?: boolean       // Default: false
}): Geometry
```

Tip at +Z, base at -Z (centered at origin).

### Cylinder

```ts
const createCylinderGeometry = (options?: {
  radiusTop?: number        // Default: 0.5
  radiusBottom?: number     // Default: 0.5
  height?: number           // Default: 1
  radialSegments?: number   // Default: 32
  heightSegments?: number   // Default: 1
  openEnded?: boolean       // Default: false
}): Geometry
```

Axis along Z.

### Capsule

```ts
const createCapsuleGeometry = (options?: {
  radius?: number           // Default: 0.5
  height?: number           // Default: 1 (cylinder portion)
  capSegments?: number      // Default: 8
  radialSegments?: number   // Default: 32
}): Geometry
```

Cylinder with hemisphere caps along Z axis. Total height = `height + 2 * radius`.

### Circle

```ts
const createCircleGeometry = (options?: {
  radius?: number           // Default: 0.5
  segments?: number         // Default: 32
  thetaStart?: number       // Default: 0
  thetaLength?: number      // Default: 2π
}): Geometry
```

Flat disc in XY plane, facing +Z.

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

## GPU Upload

Geometry data is uploaded to the GPU lazily on first render:

```ts
const ensureGPUBuffers = (geometry: Geometry, device: WrenDevice): void => {
  if (geometry.gpuVertexBuffers) return  // Already uploaded

  geometry.gpuVertexBuffers = []
  for (const [name, attr] of geometry.attributes) {
    const buffer = device.createBuffer({
      size: attr.data.byteLength,
      usage: ['vertex'],
      data: attr.data,
      label: `${geometry.id}_${name}`,
    })
    attr.gpuBuffer = buffer
    geometry.gpuVertexBuffers.push(buffer)
  }

  if (geometry.index) {
    geometry.gpuIndexBuffer = device.createBuffer({
      size: geometry.index.byteLength,
      usage: ['index'],
      data: geometry.index,
      label: `${geometry.id}_index`,
    })
  }
}
```

## Geometry Disposal

When a geometry is removed from the scene and no longer referenced, its GPU buffers are released:

```ts
const disposeGeometry = (geometry: Geometry, device: WrenDevice): void => {
  if (geometry.gpuVertexBuffers) {
    for (const buf of geometry.gpuVertexBuffers) {
      device.destroyBuffer(buf)
    }
    geometry.gpuVertexBuffers = null
  }
  if (geometry.gpuIndexBuffer) {
    device.destroyBuffer(geometry.gpuIndexBuffer)
    geometry.gpuIndexBuffer = null
  }
  geometry.bvh = null
}
```
