# Geometry Architecture

## Vertex Format

### Standard Vertex Layout

All geometries use a consistent, interleaved vertex format. Attributes not present in a specific geometry are omitted (the pipeline's vertex buffer layout adapts accordingly).

```typescript
interface Geometry {
  // Attribute data (typed arrays)
  positions: Float32Array       // vec3 — required
  normals: Float32Array | null  // vec3 — required for Lambert
  uvs: Float32Array | null      // vec2 — for textured materials
  colors: Float32Array | null   // vec4 — vertex colors (RGBA)
  materialIndices: Uint8Array | null  // uint8 — per-vertex material index
  joints: Uint8Array | null     // uvec4 — bone indices (skinning)
  weights: Float32Array | null  // vec4 — bone weights (skinning)

  // Index data
  indices: Uint16Array | Uint32Array  // Triangle indices

  // Bounding volume
  boundingBox: AABB             // Computed from positions

  // GPU resources (created on first use)
  _vertexBuffer: GpuBuffer | null
  _indexBuffer: GpuBuffer | null
  _vertexCount: number
  _indexCount: number
}
```

### Vertex Buffer Layout

Attributes are interleaved in a single vertex buffer for cache efficiency:

```
Stride = 3*4 + 3*4 + 2*4 + 4*4 + 1 + 4*1 + 4*4 = 65 bytes (padded to 68)
         pos   norm  uv    color  matIdx joints weights

With only position + normal + uv (most common):
Stride = 3*4 + 3*4 + 2*4 = 32 bytes (nice cache alignment)
```

When a geometry has fewer attributes, the stride shrinks accordingly. The vertex buffer layout in the pipeline descriptor matches the geometry's actual attributes.

### Interleaving Strategy

```typescript
const interleaveVertexData = (geometry: Geometry): Float32Array => {
  const vertexCount = geometry.positions.length / 3
  const hasNormals = geometry.normals !== null
  const hasUvs = geometry.uvs !== null
  const hasColors = geometry.colors !== null
  // ... etc

  const floatsPerVertex = 3                          // position
    + (hasNormals ? 3 : 0)
    + (hasUvs ? 2 : 0)
    + (hasColors ? 4 : 0)
    // materialIndex and joints handled separately (non-float)

  const data = new Float32Array(vertexCount * floatsPerVertex)
  let offset = 0

  for (let i = 0; i < vertexCount; i++) {
    // Position
    data[offset++] = geometry.positions[i * 3]
    data[offset++] = geometry.positions[i * 3 + 1]
    data[offset++] = geometry.positions[i * 3 + 2]
    // Normal
    if (hasNormals) {
      data[offset++] = geometry.normals![i * 3]
      data[offset++] = geometry.normals![i * 3 + 1]
      data[offset++] = geometry.normals![i * 3 + 2]
    }
    // UV
    if (hasUvs) {
      data[offset++] = geometry.uvs![i * 2]
      data[offset++] = geometry.uvs![i * 2 + 1]
    }
    // Color
    if (hasColors) {
      data[offset++] = geometry.colors![i * 4]
      data[offset++] = geometry.colors![i * 4 + 1]
      data[offset++] = geometry.colors![i * 4 + 2]
      data[offset++] = geometry.colors![i * 4 + 3]
    }
  }
  return data
}
```

For `materialIndices` and `joints` (uint8 data), a second vertex buffer is used to avoid alignment issues in the interleaved float buffer.

## AABB Computation

```typescript
const computeAABB = (positions: Float32Array): AABB => {
  const min = [Infinity, Infinity, Infinity]
  const max = [-Infinity, -Infinity, -Infinity]
  for (let i = 0; i < positions.length; i += 3) {
    min[0] = Math.min(min[0], positions[i])
    min[1] = Math.min(min[1], positions[i + 1])
    min[2] = Math.min(min[2], positions[i + 2])
    max[0] = Math.max(max[0], positions[i])
    max[1] = Math.max(max[1], positions[i + 1])
    max[2] = Math.max(max[2], positions[i + 2])
  }
  return { min, max }
}
```

Computed once when geometry is created or positions are updated.

## Parametric Geometries

All parametric geometries generate position, normal, UV, and index data. They are created once and immutable (changing parameters requires creating a new geometry).

### PlaneGeometry

```typescript
interface PlaneGeometryParams {
  width?: number        // default: 1
  height?: number       // default: 1
  widthSegments?: number  // default: 1
  heightSegments?: number // default: 1
}
```

Generates a flat quad in the XY plane (Z-up), facing +Z. Grid of `(widthSegments+1) * (heightSegments+1)` vertices.

### BoxGeometry

```typescript
interface BoxGeometryParams {
  width?: number        // X extent, default: 1
  height?: number       // Y extent, default: 1
  depth?: number        // Z extent, default: 1
  widthSegments?: number
  heightSegments?: number
  depthSegments?: number
}
```

6 faces, each a subdivided plane. Normals face outward. UVs are per-face (no wrapping).

### SphereGeometry

```typescript
interface SphereGeometryParams {
  radius?: number           // default: 0.5
  widthSegments?: number    // longitude, default: 32
  heightSegments?: number   // latitude, default: 16
  phiStart?: number         // horizontal start angle
  phiLength?: number        // horizontal sweep angle (default: 2π)
  thetaStart?: number       // vertical start angle
  thetaLength?: number      // vertical sweep angle (default: π)
}
```

UV sphere generated using spherical coordinates. Poles are degenerate (single vertex shared by all triangles).

### ConeGeometry

```typescript
interface ConeGeometryParams {
  radius?: number           // base radius, default: 0.5
  height?: number           // default: 1
  radialSegments?: number   // default: 32
  heightSegments?: number   // default: 1
  openEnded?: boolean       // if true, no base cap (default: false)
}
```

Apex at `(0, 0, height)`, base centered at origin. This is a special case of CylinderGeometry with `topRadius = 0`.

### CylinderGeometry

```typescript
interface CylinderGeometryParams {
  topRadius?: number        // default: 0.5
  bottomRadius?: number     // default: 0.5
  height?: number           // default: 1
  radialSegments?: number   // default: 32
  heightSegments?: number   // default: 1
  openEnded?: boolean       // if true, no caps (default: false)
}
```

Centered at origin, extends from `-height/2` to `+height/2` along Z axis.

### CapsuleGeometry

```typescript
interface CapsuleGeometryParams {
  radius?: number           // default: 0.5
  height?: number           // total height including caps, default: 1
  capSegments?: number      // hemisphere subdivisions, default: 8
  radialSegments?: number   // default: 32
}
```

A cylinder with hemisphere caps. The cylinder section height is `height - 2 * radius`. Generated by stitching a top hemisphere, cylinder body, and bottom hemisphere.

### CircleGeometry

```typescript
interface CircleGeometryParams {
  radius?: number           // default: 0.5
  segments?: number         // default: 32
  thetaStart?: number       // start angle, default: 0
  thetaLength?: number      // sweep angle, default: 2π
}
```

A flat disc in the XY plane, facing +Z. Center vertex at origin, fan of triangles.

## Geometry Disposal

When a geometry is no longer needed, its GPU buffers must be released:

```typescript
const dispose = (geometry: Geometry, device: GpuDevice) => {
  if (geometry._vertexBuffer) device.destroyBuffer(geometry._vertexBuffer)
  if (geometry._indexBuffer) device.destroyBuffer(geometry._indexBuffer)
  geometry._vertexBuffer = null
  geometry._indexBuffer = null
}
```

The React bindings handle this automatically on component unmount.

## Index Format Selection

- **Uint16** for geometries with ≤ 65535 vertices (vast majority of game assets)
- **Uint32** for larger geometries (auto-detected from vertex count)

Uint16 is preferred as it halves index buffer memory and improves vertex post-transform cache hit rates on mobile GPUs.
