# Geometry — Parametric Geometries

## Overview

Fennec provides parametric geometry generators for common primitives: **plane, box, sphere, cone, cylinder, capsule, and circle**. All generators produce indexed geometry with positions, normals, and UVs. The output is a `Geometry` object ready for GPU upload.

## Geometry Object

```typescript
interface Geometry {
  readonly id: number
  readonly positions: Float32Array     // vec3 per vertex
  readonly normals: Float32Array       // vec3 per vertex
  readonly uvs: Float32Array           // vec2 per vertex
  readonly indices: Uint16Array | Uint32Array  // Uint16 if <= 65535 verts, else Uint32
  readonly vertexCount: number
  readonly indexCount: number
  readonly boundingBox: AABB

  // GPU handles (created on first use)
  _positionBuffer: GPUBufferHandle | null
  _normalBuffer: GPUBufferHandle | null
  _uvBuffer: GPUBufferHandle | null
  _indexBuffer: GPUBufferHandle | null
  _vao: WebGLVertexArrayObject | null  // WebGL2 only
  _bvh: BVHData | null                // Built on demand for raycasting

  dispose(): void
}
```

## Coordinate System Note

All geometries are generated in Fennec's **Z-up, right-handed** coordinate system. "Height" parameters refer to the Z axis. "Width" is X, "depth" is Y.

## Plane Geometry

A flat plane in the XY plane (facing Z-up).

```typescript
interface PlaneGeometryOptions {
  width?: number       // X extent (default: 1)
  height?: number      // Y extent (default: 1)
  widthSegments?: number   // Subdivisions along X (default: 1)
  heightSegments?: number  // Subdivisions along Y (default: 1)
}

const createPlaneGeometry = (options?: PlaneGeometryOptions): Geometry
```

```
widthSegments=3, heightSegments=2:

Y
^  v6----v7----v8----v9
|  | \   | \   | \   |
|  |  \  |  \  |  \  |
|  v3----v4----v5----v5
|  | \   | \   | \   |
|  |  \  |  \  |  \  |
|  v0----v1----v2----v3
+----------------------------> X
```

- Normals: all point `[0, 0, 1]` (Z-up)
- UVs: `(0,0)` at bottom-left, `(1,1)` at top-right

## Box Geometry

An axis-aligned box centered at the origin.

```typescript
interface BoxGeometryOptions {
  width?: number        // X extent (default: 1)
  depth?: number        // Y extent (default: 1)
  height?: number       // Z extent (default: 1)
  widthSegments?: number
  depthSegments?: number
  heightSegments?: number
}

const createBoxGeometry = (options?: BoxGeometryOptions): Geometry
```

- 6 faces, each a subdivided plane
- Each face has its own set of vertices (no sharing) for correct normals
- Normals: outward-facing per face
- UVs: each face maps `(0,0)` to `(1,1)`

## Sphere Geometry

A UV sphere centered at the origin.

```typescript
interface SphereGeometryOptions {
  radius?: number          // Default: 0.5
  widthSegments?: number   // Longitude segments (default: 32)
  heightSegments?: number  // Latitude segments (default: 16)
  phiStart?: number        // Horizontal start angle (default: 0)
  phiLength?: number       // Horizontal sweep (default: 2π)
  thetaStart?: number      // Vertical start angle (default: 0)
  thetaLength?: number     // Vertical sweep (default: π)
}

const createSphereGeometry = (options?: SphereGeometryOptions): Geometry
```

Generation:

```typescript
const createSphereGeometry = (opts?: SphereGeometryOptions): Geometry => {
  const radius = opts?.radius ?? 0.5
  const wSeg = opts?.widthSegments ?? 32
  const hSeg = opts?.heightSegments ?? 16
  const phiStart = opts?.phiStart ?? 0
  const phiLength = opts?.phiLength ?? Math.PI * 2
  const thetaStart = opts?.thetaStart ?? 0
  const thetaLength = opts?.thetaLength ?? Math.PI

  const vertCount = (wSeg + 1) * (hSeg + 1)
  const positions = new Float32Array(vertCount * 3)
  const normals = new Float32Array(vertCount * 3)
  const uvs = new Float32Array(vertCount * 2)

  let idx = 0
  for (let y = 0; y <= hSeg; y++) {
    const v = y / hSeg
    const theta = thetaStart + v * thetaLength

    for (let x = 0; x <= wSeg; x++) {
      const u = x / wSeg
      const phi = phiStart + u * phiLength

      // Z-up sphere: swap Y and Z from standard UV sphere
      const px = -radius * Math.sin(theta) * Math.cos(phi)
      const py = -radius * Math.sin(theta) * Math.sin(phi)
      const pz = radius * Math.cos(theta)

      positions[idx * 3] = px
      positions[idx * 3 + 1] = py
      positions[idx * 3 + 2] = pz

      const nx = px / radius
      const ny = py / radius
      const nz = pz / radius
      normals[idx * 3] = nx
      normals[idx * 3 + 1] = ny
      normals[idx * 3 + 2] = nz

      uvs[idx * 2] = u
      uvs[idx * 2 + 1] = v

      idx++
    }
  }

  // Generate triangle indices
  const indices = generateGridIndices(wSeg, hSeg, vertCount)

  return buildGeometry(positions, normals, uvs, indices)
}
```

## Cone Geometry

A cone with tip at `Z = height/2` and base at `Z = -height/2`.

```typescript
interface ConeGeometryOptions {
  radius?: number           // Base radius (default: 0.5)
  height?: number           // Z extent (default: 1)
  radialSegments?: number   // Around the circumference (default: 32)
  heightSegments?: number   // Along Z (default: 1)
  openEnded?: boolean       // Skip base cap (default: false)
  thetaStart?: number       // Start angle (default: 0)
  thetaLength?: number      // Sweep angle (default: 2π)
}

const createConeGeometry = (options?: ConeGeometryOptions): Geometry
```

Internally uses the same generator as cylinder with `topRadius = 0`.

## Cylinder Geometry

A cylinder centered at the origin, extending along the Z axis.

```typescript
interface CylinderGeometryOptions {
  radiusTop?: number        // Top radius (default: 0.5)
  radiusBottom?: number     // Bottom radius (default: 0.5)
  height?: number           // Z extent (default: 1)
  radialSegments?: number   // Default: 32
  heightSegments?: number   // Default: 1
  openEnded?: boolean       // Skip top/bottom caps (default: false)
  thetaStart?: number       // Default: 0
  thetaLength?: number      // Default: 2π
}

const createCylinderGeometry = (options?: CylinderGeometryOptions): Geometry
```

The cylinder generator handles:
- Variable top/bottom radii (for cones and tapered shapes)
- Open/closed ends
- Partial sweeps for arcs

Normals on the side surface are interpolated between the top and bottom edge normals for smooth shading on the curved surface.

## Capsule Geometry

A capsule (cylinder with hemispherical caps) centered at the origin, along the Z axis.

```typescript
interface CapsuleGeometryOptions {
  radius?: number           // Cap and cylinder radius (default: 0.5)
  height?: number           // Total height including caps (default: 1)
  capSegments?: number      // Hemisphere segments (default: 8)
  radialSegments?: number   // Around the circumference (default: 32)
}

const createCapsuleGeometry = (options?: CapsuleGeometryOptions): Geometry
```

Construction:
1. Generate a hemisphere for the top cap (Z = height/2 - radius to Z = height/2)
2. Generate the cylinder body (Z = -height/2 + radius to Z = height/2 - radius)
3. Generate a hemisphere for the bottom cap (Z = -height/2 to Z = -height/2 + radius)
4. Stitch vertices at cap-to-cylinder boundaries (shared vertices, interpolated normals)

## Circle Geometry

A flat circle (disc) in the XY plane, facing Z-up.

```typescript
interface CircleGeometryOptions {
  radius?: number           // Default: 0.5
  segments?: number         // Default: 32
  thetaStart?: number       // Default: 0
  thetaLength?: number      // Default: 2π
}

const createCircleGeometry = (options?: CircleGeometryOptions): Geometry
```

- Center vertex at `(0, 0, 0)`
- Ring of vertices at `radius` distance
- Triangle fan from center to ring
- Normals: all `[0, 0, 1]`
- UVs: center = `(0.5, 0.5)`, edge maps around the unit circle

## Utility: Index Generation

For grid-based geometries (plane, sphere), indices are generated from a grid:

```typescript
const generateGridIndices = (cols: number, rows: number, vertexCount: number): Uint16Array | Uint32Array => {
  const indexCount = cols * rows * 6
  const IndexArray = vertexCount > 65535 ? Uint32Array : Uint16Array
  const indices = new IndexArray(indexCount)
  let idx = 0

  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      const a = row * (cols + 1) + col
      const b = a + 1
      const c = (row + 1) * (cols + 1) + col
      const d = c + 1

      indices[idx++] = a
      indices[idx++] = c
      indices[idx++] = b

      indices[idx++] = b
      indices[idx++] = c
      indices[idx++] = d
    }
  }

  return indices
}
```

## Geometry Builder

All generators use a common builder that finalizes the geometry, computes the AABB, and prepares for GPU upload:

```typescript
const buildGeometry = (positions: Float32Array, normals: Float32Array, uvs: Float32Array, indices: Uint16Array | Uint32Array): Geometry => {
  const boundingBox = computeAABBFromPositions(positions)

  return {
    id: generateId(),
    positions,
    normals,
    uvs,
    indices,
    vertexCount: positions.length / 3,
    indexCount: indices.length,
    boundingBox,
    _positionBuffer: null,
    _normalBuffer: null,
    _uvBuffer: null,
    _indexBuffer: null,
    _vao: null,
    _bvh: null,
    dispose() {
      // Delete GPU resources
    },
  }
}
```

## Custom Geometry

Users can create geometry from raw data:

```typescript
const customGeometry = createGeometry({
  positions: new Float32Array([...]),
  normals: new Float32Array([...]),
  uvs: new Float32Array([...]),
  indices: new Uint16Array([...]),
  // Optional additional attributes
  attributes: {
    materialIndex: { data: new Uint8Array([...]), size: 1 },
    color: { data: new Float32Array([...]), size: 4 },
  }
})
```
