# Geometry & Buffer Layout

## Geometry Structure

A `Geometry` is a container for vertex attributes and an optional index buffer. It describes the raw shape data without any material or transform.

```typescript
interface Geometry {
  readonly id: number

  // Vertex attributes
  attributes: Map<string, BufferAttribute>

  // Index buffer (optional — if absent, uses non-indexed drawing)
  index: BufferAttribute | null

  // Computed bounding volumes (calculated from position attribute)
  boundingBox: AABB | null
  boundingSphere: Sphere | null

  // GPU resource handles (set after upload)
  _vao: GPUBufferHandle | null       // WebGL VAO / WebGPU vertex state
  _vertexBuffers: GPUBufferHandle[]  // One per attribute (or interleaved)
  _indexBuffer: GPUBufferHandle | null

  // Methods
  computeBoundingBox(): AABB
  computeBoundingSphere(): Sphere
  upload(device: GPUDevice): void
  dispose(device: GPUDevice): void
}
```

## Buffer Attributes

Each attribute wraps a typed array with metadata:

```typescript
interface BufferAttribute {
  name: string                 // Shader attribute name
  array: Float32Array | Uint16Array | Uint32Array | Uint8Array
  itemSize: number             // Components per vertex (1, 2, 3, or 4)
  count: number                // Number of vertices
  normalized: boolean          // Normalize integer types to [0,1]
  stride: number               // Bytes between consecutive elements (0 = tightly packed)
  offset: number               // Byte offset within interleaved buffer
  needsUpload: boolean         // Dirty flag for re-upload
}
```

### Standard Attribute Names

| Name | Item Size | Type | Description |
|------|-----------|------|-------------|
| `position` | 3 | Float32 | Vertex position (x, y, z) |
| `normal` | 3 | Float32 | Vertex normal |
| `uv` | 2 | Float32 | Texture coordinates |
| `color` | 3 or 4 | Float32 or Uint8 | Vertex color (RGB or RGBA) |
| `_materialindex` | 1 | Float32 | Material index (integer stored as float for GLSL compatibility) |
| `skinIndex` | 4 | Uint16 | Bone indices (4 bones per vertex) |
| `skinWeight` | 4 | Float32 | Bone weights (4 per vertex, sum to 1.0) |

### Interleaved vs. Separate Buffers

By default, attributes are stored in separate buffers for simplicity. However, the geometry system supports interleaved buffers for better cache performance:

```typescript
// Separate buffers (default)
// position: [x0,y0,z0, x1,y1,z1, ...]  → Buffer 0
// normal:   [nx0,ny0,nz0, ...]          → Buffer 1
// uv:       [u0,v0, u1,v1, ...]         → Buffer 2

// Interleaved buffer (optional, for performance)
// [x0,y0,z0, nx0,ny0,nz0, u0,v0, x1,y1,z1, nx1,ny1,nz1, u1,v1, ...]
// All in Buffer 0, with stride = 32 bytes, varying offsets
```

For parametric geometries, Caracal uses interleaved buffers by default since the layout is known at generation time. For loaded geometry (glTF), it preserves whatever layout the file provides.

## Vertex Layout Descriptor

The vertex layout defines how attributes map to shader inputs. This is needed for WebGPU pipeline creation:

```typescript
interface VertexLayout {
  buffers: VertexBufferLayout[]
}

interface VertexBufferLayout {
  stride: number
  stepMode: 'vertex' | 'instance'
  attributes: VertexAttributeLayout[]
}

interface VertexAttributeLayout {
  shaderLocation: number   // Matches @location(N) in WGSL / layout(location=N) in GLSL
  offset: number           // Byte offset within buffer
  format: VertexFormat     // 'float32x3', 'float32x2', 'uint16x4', etc.
}
```

### Standard Shader Locations

| Location | Attribute | Format |
|----------|-----------|--------|
| 0 | `position` | float32x3 |
| 1 | `normal` | float32x3 |
| 2 | `uv` | float32x2 |
| 3 | `color` | float32x3 or float32x4 |
| 4 | `_materialindex` | float32 |
| 5 | `skinIndex` | uint16x4 |
| 6 | `skinWeight` | float32x4 |

## Index Buffer

Most geometries use indexed drawing to share vertices between triangles. Caracal supports:
- `Uint16Array` — up to 65,535 vertices (default, most efficient)
- `Uint32Array` — for meshes exceeding 65K vertices

The geometry automatically selects the index type based on vertex count.

## Parametric Geometry Generators

All generators return a `Geometry` with `position`, `normal`, and `uv` attributes. They use interleaved buffers for cache efficiency.

### PlaneGeometry

A flat rectangle in the XY plane (Z-up means the plane faces Z+):

```typescript
const plane = createPlaneGeometry({
  width: 10,           // Size along X
  height: 10,          // Size along Y
  widthSegments: 1,    // Subdivisions along X
  heightSegments: 1,   // Subdivisions along Y
})
```

Generates `(widthSegments + 1) × (heightSegments + 1)` vertices. Default: 4 vertices, 2 triangles.

### BoxGeometry

An axis-aligned box centered at origin:

```typescript
const box = createBoxGeometry({
  width: 1,            // Size along X
  height: 1,           // Size along Y (forward)
  depth: 1,            // Size along Z (up)
  widthSegments: 1,
  heightSegments: 1,
  depthSegments: 1,
})
```

Each face is a plane subdivision. Normals point outward per face. With 1 segment per axis: 24 vertices (4 per face × 6 faces), 36 indices.

### SphereGeometry

UV sphere (longitude/latitude parameterization):

```typescript
const sphere = createSphereGeometry({
  radius: 1,
  widthSegments: 32,   // Longitude divisions
  heightSegments: 16,  // Latitude divisions
  phiStart: 0,         // Longitude start angle (radians)
  phiLength: Math.PI * 2,
  thetaStart: 0,       // Latitude start angle
  thetaLength: Math.PI,
})
```

Pole vertices are shared. Total vertices: `(widthSegments + 1) × (heightSegments + 1)`.

### ConeGeometry

A cone with apex at Z+ and base at Z=0:

```typescript
const cone = createConeGeometry({
  radius: 1,           // Base radius
  height: 2,           // Height along Z
  radialSegments: 16,  // Circumference divisions
  heightSegments: 1,   // Vertical divisions
  openEnded: false,    // If true, no base cap
})
```

### CylinderGeometry

A cylinder centered at origin, extending along Z:

```typescript
const cylinder = createCylinderGeometry({
  radiusTop: 1,
  radiusBottom: 1,
  height: 2,
  radialSegments: 16,
  heightSegments: 1,
  openEnded: false,    // If true, no caps
})
```

When `radiusTop !== radiusBottom`, produces a truncated cone.

### CapsuleGeometry

A cylinder with hemispherical caps. Useful for character colliders and pills:

```typescript
const capsule = createCapsuleGeometry({
  radius: 0.5,
  height: 2,           // Total height including caps
  capSegments: 8,      // Hemisphere latitude divisions
  radialSegments: 16,
})
```

The capsule extends along Z. Hemisphere caps are generated as partial spheres joined seamlessly to the cylinder body.

### CircleGeometry

A flat disc in the XY plane:

```typescript
const circle = createCircleGeometry({
  radius: 1,
  segments: 32,
  thetaStart: 0,
  thetaLength: Math.PI * 2,
})
```

With `thetaLength < 2π`, produces a sector/pie shape.

## Custom Attributes

Users can add arbitrary attributes to any geometry for custom shading effects:

```typescript
const geometry = createBoxGeometry({ width: 1, height: 1, depth: 1 })

// Add a custom per-vertex attribute
geometry.attributes.set('a_temperature', createBufferAttribute({
  array: new Float32Array(geometry.attributes.get('position')!.count),
  itemSize: 1,
}))
```

The `_materialindex` attribute is the primary custom attribute used in Caracal's material index system. When loading glTF, the loader looks for a custom attribute named `_MATERIALINDEX` (following glTF naming conventions) and maps it to `_materialindex`.

## Geometry Upload

Geometry buffers are uploaded to the GPU explicitly:

```typescript
// Upload creates GPU buffers and VAO (WebGL) or records vertex layout (WebGPU)
geometry.upload(device)

// After upload, CPU-side arrays can optionally be released to save memory:
geometry.attributes.get('position')!.array = null // free CPU copy
// (Only do this for static geometry that won't be used for raycasting)
```

For raycasting, the CPU-side position and index arrays must be retained. The `BVH` builder reads from these arrays.

## Geometry Disposal

```typescript
geometry.dispose(device)
// Destroys all GPU buffers, VAO, and clears attribute references
```

Disposal is explicit — there's no automatic garbage collection of GPU resources. The scene graph does NOT auto-dispose geometries when nodes are removed, since geometries can be shared between multiple meshes.

## Buffer Sharing

Multiple meshes can share the same geometry:

```typescript
const sharedBox = createBoxGeometry({ width: 1, height: 1, depth: 1 })
sharedBox.upload(device)

const mesh1 = createMesh(sharedBox, redMaterial)
const mesh2 = createMesh(sharedBox, blueMaterial) // same geometry, different material
```

This saves GPU memory and allows the renderer to potentially batch draw calls using the same vertex buffers.
