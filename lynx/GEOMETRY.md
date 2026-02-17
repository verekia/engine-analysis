# Geometry System

The geometry system in Lynx manages vertex data on the CPU side and handles upload to GPU buffers for rendering. All geometry in Lynx is represented through `BufferGeometry`, a flat container of typed-array attribute buffers and an optional index buffer.

---

## 1. BufferGeometry

`BufferGeometry` is the core geometry container. It holds a set of named vertex attributes as typed arrays, an optional index buffer, and metadata needed for draw calls and bounding volume computation.

### Attribute Layout

Each attribute is described by a descriptor and stored in its own typed array (separate buffer strategy). Lynx does not interleave attributes into a single buffer. This trades a small amount of GPU bandwidth for significant flexibility: attributes can be added, removed, or replaced independently without rebuilding adjacent data.

| Attribute         | Type          | Components | Required |
|-------------------|---------------|------------|----------|
| `position`        | `Float32Array`| 3 (vec3)   | Yes      |
| `normal`          | `Float32Array`| 3 (vec3)   | Yes      |
| `uv`              | `Float32Array`| 2 (vec2)   | Yes      |
| `color`           | `Float32Array`| 3 or 4 (vec3/vec4) | No |
| `_materialindex`  | `Uint8Array`  | 1 (uint8)  | No       |
| `joints`          | `Uint16Array` | 4 (uvec4)  | No       |
| `weights`         | `Float32Array`| 4 (vec4)   | No       |

The `_materialindex` attribute is an engine-internal attribute used for multi-material meshes. It stores a per-vertex material slot index as an unsigned byte, allowing up to 256 material slots per geometry. The `joints` and `weights` attributes are present only on skinned meshes.

### Interfaces

```typescript
interface AttributeDescriptor {
  name: string
  size: number          // components per vertex: 1, 2, 3, or 4
  type: AttributeType   // Float32, Uint8, Uint16, Uint32
  normalized: boolean
}

const enum AttributeType {
  Float32 = 0,
  Uint8   = 1,
  Uint16  = 2,
  Uint32  = 3,
}

interface BufferAttribute {
  descriptor: AttributeDescriptor
  data: TypedArray        // Float32Array | Uint8Array | Uint16Array | Uint32Array
  gpuBuffer: GPUBuffer | WebGLBuffer | null
  needsUpload: boolean
}

interface BufferGeometry {
  id: number
  attributes: Map<string, BufferAttribute>
  index: Uint16Array | Uint32Array | null
  indexBuffer: GPUBuffer | WebGLBuffer | null
  vertexCount: number
  indexCount: number
  aabb: AABB
  refCount: number
  isDynamic: boolean
  needsUpload: boolean
}
```

### AABB Computation

When a `BufferGeometry` is created or when the position attribute is replaced, an axis-aligned bounding box is computed by iterating over the position data:

```typescript
const computeAABB = (positions: Float32Array): AABB => {
  const min: Vec3 = [Infinity, Infinity, Infinity]
  const max: Vec3 = [-Infinity, -Infinity, -Infinity]

  for (let i = 0; i < positions.length; i += 3) {
    const x = positions[i]
    const y = positions[i + 1]
    const z = positions[i + 2]
    if (x < min[0]) min[0] = x
    if (y < min[1]) min[1] = y
    if (z < min[2]) min[2] = z
    if (x > max[0]) max[0] = x
    if (y > max[1]) max[1] = y
    if (z > max[2]) max[2] = z
  }

  return { min, max }
}
```

### Vertex and Index Count

`vertexCount` is derived from the position attribute: `positions.length / 3`. `indexCount` is `index.length` when an index buffer is present, or `0` otherwise. These values are passed directly to draw calls (`drawElements` / `drawArrays` or their WebGPU equivalents).

### Separate vs Interleaved Buffers

Lynx uses **separate** attribute buffers. Each attribute lives in its own typed array and maps to its own GPU buffer. This means:

- Adding an attribute (e.g., vertex colors for debug visualization) requires no changes to existing buffers.
- Removing an attribute (e.g., stripping UVs from a shadow-only geometry) is a simple map deletion.
- Partial updates are straightforward: uploading only the changed attribute buffer.

The cost is one additional `vertexAttribPointer` / vertex buffer binding per attribute, which is negligible on modern hardware.

---

## 2. Parametric Geometries

Each parametric geometry is a factory function that returns a fully populated `BufferGeometry`. All functions produce indexed geometry with `position`, `normal`, and `uv` attributes.

### Function Signatures

```typescript
const createPlaneGeometry = (
  width: number,
  depth: number,
  widthSegments?: number,   // default: 1
  depthSegments?: number    // default: 1
): BufferGeometry

const createBoxGeometry = (
  width: number,
  height: number,
  depth: number,
  widthSegments?: number,   // default: 1
  heightSegments?: number,  // default: 1
  depthSegments?: number    // default: 1
): BufferGeometry

const createSphereGeometry = (
  radius: number,
  widthSegments?: number,   // default: 32
  heightSegments?: number,  // default: 16
  phiStart?: number,        // default: 0
  phiLength?: number,       // default: 2 * Math.PI
  thetaStart?: number,      // default: 0
  thetaLength?: number      // default: Math.PI
): BufferGeometry

const createConeGeometry = (
  radius: number,
  height: number,
  radialSegments?: number,  // default: 32
  heightSegments?: number,  // default: 1
  openEnded?: boolean       // default: false
): BufferGeometry

const createCylinderGeometry = (
  radiusTop: number,
  radiusBottom: number,
  height: number,
  radialSegments?: number,  // default: 32
  heightSegments?: number,  // default: 1
  openEnded?: boolean       // default: false
): BufferGeometry

const createCapsuleGeometry = (
  radius: number,
  height: number,           // height of the cylindrical section (total = height + 2 * radius)
  capSegments?: number,     // default: 8
  radialSegments?: number   // default: 32
): BufferGeometry

const createCircleGeometry = (
  radius: number,
  segments?: number,        // default: 32
  thetaStart?: number,      // default: 0
  thetaLength?: number      // default: 2 * Math.PI
): BufferGeometry
```

### Orientation Summary

All geometries follow the Z-up right-handed convention:

| Geometry   | Primary Plane / Axis | Normal Direction      |
|------------|---------------------|-----------------------|
| Plane      | XY plane            | +Z                    |
| Box        | Centered at origin  | Face normals outward  |
| Sphere     | Poles along Z axis  | Radial outward        |
| Cone       | Tip at +Z           | Surface outward       |
| Cylinder   | Along Z axis        | Surface outward       |
| Capsule    | Along Z axis        | Surface outward       |
| Circle     | XY plane            | +Z                    |

---

## 3. Geometry Generation Details

### Index Buffer Type Selection

All factory functions select the index buffer type based on vertex count:

```typescript
const createIndexBuffer = (indices: number[], vertexCount: number): Uint16Array | Uint32Array => {
  if (vertexCount <= 65535) {
    return new Uint16Array(indices)
  }
  return new Uint32Array(indices)
}
```

This keeps index buffers as small as possible (2 bytes per index) for typical meshes while supporting high-resolution meshes that exceed the 16-bit vertex limit.

### Plane Generation

The plane lies on the XY plane with the normal pointing in the +Z direction. Vertices are laid out in a grid:

```typescript
const createPlaneGeometry = (
  width: number,
  depth: number,
  widthSegments = 1,
  depthSegments = 1
): BufferGeometry => {
  const widthHalf = width / 2
  const depthHalf = depth / 2
  const gridX = widthSegments
  const gridY = depthSegments
  const gridX1 = gridX + 1
  const gridY1 = gridY + 1
  const segmentWidth = width / gridX
  const segmentDepth = depth / gridY

  const positions: number[] = []
  const normals: number[] = []
  const uvs: number[] = []
  const indices: number[] = []

  for (let iy = 0; iy < gridY1; iy++) {
    const y = iy * segmentDepth - depthHalf
    for (let ix = 0; ix < gridX1; ix++) {
      const x = ix * segmentWidth - widthHalf
      positions.push(x, y, 0)
      normals.push(0, 0, 1)
      uvs.push(ix / gridX, 1 - iy / gridY)
    }
  }

  for (let iy = 0; iy < gridY; iy++) {
    for (let ix = 0; ix < gridX; ix++) {
      const a = ix + gridX1 * iy
      const b = ix + gridX1 * (iy + 1)
      const c = (ix + 1) + gridX1 * (iy + 1)
      const d = (ix + 1) + gridX1 * iy
      indices.push(a, b, d)
      indices.push(b, c, d)
    }
  }

  // ... build BufferGeometry from typed arrays
}
```

### Sphere Generation (UV Sphere, Z-Up)

The sphere is generated as a UV sphere with poles along the Z axis. Latitude rings sweep from the south pole (-Z) to the north pole (+Z) and longitude slices sweep around the Z axis.

```typescript
const createSphereGeometry = (
  radius: number,
  widthSegments = 32,
  heightSegments = 16,
  phiStart = 0,
  phiLength = Math.PI * 2,
  thetaStart = 0,
  thetaLength = Math.PI
): BufferGeometry => {
  const widthSegs = Math.max(3, Math.floor(widthSegments))
  const heightSegs = Math.max(2, Math.floor(heightSegments))
  const thetaEnd = Math.min(thetaStart + thetaLength, Math.PI)

  const positions: number[] = []
  const normals: number[] = []
  const uvs: number[] = []
  const indices: number[] = []
  const grid: number[][] = []

  let index = 0

  // Generate vertices.
  // theta sweeps from top (thetaStart) to bottom (thetaEnd) — polar angle from +Z.
  // phi sweeps around the Z axis starting at phiStart.
  for (let iy = 0; iy <= heightSegs; iy++) {
    const row: number[] = []
    const v = iy / heightSegs

    // theta: angle from +Z pole (0 = north pole, PI = south pole)
    const theta = thetaStart + v * thetaLength

    for (let ix = 0; ix <= widthSegs; ix++) {
      const u = ix / widthSegs

      // phi: angle around Z axis in the XY plane
      const phi = phiStart + u * phiLength

      // Convert spherical (r, theta, phi) to Cartesian (x, y, z) for Z-up:
      //   x = r * sin(theta) * cos(phi)
      //   y = r * sin(theta) * sin(phi)
      //   z = r * cos(theta)
      const sinTheta = Math.sin(theta)
      const cosTheta = Math.cos(theta)
      const sinPhi = Math.sin(phi)
      const cosPhi = Math.cos(phi)

      const x = radius * sinTheta * cosPhi
      const y = radius * sinTheta * sinPhi
      const z = radius * cosTheta

      positions.push(x, y, z)

      // Normal is the normalized position vector (unit sphere direction).
      normals.push(sinTheta * cosPhi, sinTheta * sinPhi, cosTheta)

      uvs.push(u, 1 - v)

      row.push(index++)
    }

    grid.push(row)
  }

  // Generate indices.
  // Each quad in the grid becomes two triangles, except at the poles
  // where degenerate triangles are skipped.
  for (let iy = 0; iy < heightSegs; iy++) {
    for (let ix = 0; ix < widthSegs; ix++) {
      const a = grid[iy][ix + 1]
      const b = grid[iy][ix]
      const c = grid[iy + 1][ix]
      const d = grid[iy + 1][ix + 1]

      // Skip degenerate triangles at the north pole (first row).
      if (iy !== 0 || thetaStart > 0) {
        indices.push(a, b, d)
      }

      // Skip degenerate triangles at the south pole (last row).
      if (iy !== heightSegs - 1 || thetaEnd < Math.PI) {
        indices.push(b, c, d)
      }
    }
  }

  const vertexCount = positions.length / 3
  const positionArray = new Float32Array(positions)
  const normalArray = new Float32Array(normals)
  const uvArray = new Float32Array(uvs)
  const indexArray = createIndexBuffer(indices, vertexCount)

  return buildBufferGeometry(positionArray, normalArray, uvArray, indexArray)
}
```

**Algorithm summary for the sphere:**

1. Two nested loops iterate over latitude (`iy` from 0 to `heightSegs`) and longitude (`ix` from 0 to `widthSegs`).
2. The polar angle `theta` ranges from `thetaStart` (default 0, the +Z pole) to `thetaStart + thetaLength` (default PI, the -Z pole).
3. The azimuthal angle `phi` ranges from `phiStart` (default 0) to `phiStart + phiLength` (default 2PI) around the Z axis.
4. Spherical-to-Cartesian conversion uses the Z-up convention: `x = r sin(theta) cos(phi)`, `y = r sin(theta) sin(phi)`, `z = r cos(theta)`.
5. Normals are simply the unit direction from the origin (the position divided by the radius).
6. A 2D grid of vertex indices is built. Each cell in the grid maps to a quad, which is split into two triangles. Degenerate pole triangles (where an entire row of vertices collapses to a single point) are omitted.

### Capsule Generation (Z-Up)

A capsule is a cylinder with hemisphere caps on both ends. The cylindrical section runs along the Z axis, and the caps are generated similarly to the sphere but covering only one hemisphere each.

```typescript
const createCapsuleGeometry = (
  radius: number,
  height: number,
  capSegments = 8,
  radialSegments = 32
): BufferGeometry => {
  const radSegs = Math.max(3, Math.floor(radialSegments))
  const capSegs = Math.max(1, Math.floor(capSegments))
  const halfHeight = height / 2

  const positions: number[] = []
  const normals: number[] = []
  const uvs: number[] = []
  const indices: number[] = []

  // Total height rings: capSegs (top cap) + 1 (shared equator row for cylinder top)
  //                   + 0 or more cylinder body rows
  //                   + 1 (shared equator row for cylinder bottom) + capSegs (bottom cap)
  // Simplified: top hemisphere + cylinder body + bottom hemisphere.

  // The total number of latitude rings:
  //   top cap:    capSegs + 1 rows  (from +Z pole down to cylinder top)
  //   cylinder:   2 rows            (top ring and bottom ring — the equator rows)
  //   bottom cap: capSegs + 1 rows  (from cylinder bottom down to -Z pole)
  //
  // Merged shared rows, the total unique rows:
  //   (capSegs + 1) + 0 + (capSegs + 1) = 2 * capSegs + 2
  //   but the cylinder top row is shared with the last row of the top cap,
  //   and the cylinder bottom row is shared with the first row of the bottom cap,
  //   so we only add the two cylinder body equator rows if height > 0.

  const totalRows = capSegs * 2 + 2  // +2 for the two equator/cylinder-edge rings
  let index = 0
  const grid: number[][] = []

  // Compute the total V range for UV mapping.
  // The total arc length along the profile:
  //   top cap arc = radius * (PI/2)
  //   cylinder    = height
  //   bottom cap  = radius * (PI/2)
  const arcTop = radius * Math.PI * 0.5
  const arcCyl = height
  const arcBot = radius * Math.PI * 0.5
  const totalArc = arcTop + arcCyl + arcBot

  // Generate vertices row by row.
  for (let iy = 0; iy <= totalRows; iy++) {
    const row: number[] = []
    let nz: number   // normal z component
    let nxy: number   // normal xy scale
    let z: number     // vertex z position
    let v: number     // texture v coordinate (0 at top, 1 at bottom)

    if (iy <= capSegs) {
      // --- Top hemisphere cap ---
      // Polar angle from +Z pole: 0 to PI/2
      const t = iy / capSegs
      const theta = t * (Math.PI / 2)
      nz = Math.cos(theta)
      nxy = Math.sin(theta)
      z = halfHeight + radius * nz
      v = (arcTop * t) / totalArc
    } else if (iy <= capSegs + 1) {
      // --- Cylinder bottom edge (top cap ended, cylinder begins) ---
      nz = 0
      nxy = 1
      z = -halfHeight
      v = (arcTop + arcCyl) / totalArc
    } else {
      // --- Bottom hemisphere cap ---
      const t = (iy - capSegs - 1) / capSegs
      const theta = Math.PI / 2 + t * (Math.PI / 2)
      nz = Math.cos(theta)
      nxy = Math.sin(theta)
      z = -halfHeight + radius * nz
      v = (arcTop + arcCyl + arcBot * t) / totalArc
    }

    for (let ix = 0; ix <= radSegs; ix++) {
      const u = ix / radSegs
      const phi = u * Math.PI * 2

      const cosPhi = Math.cos(phi)
      const sinPhi = Math.sin(phi)

      const nx = nxy * cosPhi
      const ny = nxy * sinPhi

      positions.push(
        (nxy * radius) * cosPhi,
        (nxy * radius) * sinPhi,
        z
      )
      normals.push(nx, ny, nz)
      uvs.push(u, 1 - v)

      row.push(index++)
    }

    grid.push(row)
  }

  // Generate indices from the grid, same approach as the sphere.
  for (let iy = 0; iy < totalRows; iy++) {
    for (let ix = 0; ix < radSegs; ix++) {
      const a = grid[iy][ix + 1]
      const b = grid[iy][ix]
      const c = grid[iy + 1][ix]
      const d = grid[iy + 1][ix + 1]

      // Skip degenerate triangle at the top pole.
      if (iy !== 0) {
        indices.push(a, b, d)
      }

      // Skip degenerate triangle at the bottom pole.
      if (iy !== totalRows - 1) {
        indices.push(b, c, d)
      }
    }
  }

  const vertexCount = positions.length / 3
  const positionArray = new Float32Array(positions)
  const normalArray = new Float32Array(normals)
  const uvArray = new Float32Array(uvs)
  const indexArray = createIndexBuffer(indices, vertexCount)

  return buildBufferGeometry(positionArray, normalArray, uvArray, indexArray)
}
```

**Algorithm summary for the capsule:**

1. The capsule profile is divided into three sections along Z: a top hemisphere cap, a straight cylindrical section, and a bottom hemisphere cap.
2. The top hemisphere sweeps the polar angle `theta` from 0 (the +Z pole) to PI/2 (the equator). At each ring, the Z position is `halfHeight + radius * cos(theta)` and the XY radius is `radius * sin(theta)`.
3. The cylindrical section connects the bottom of the top hemisphere (`z = halfHeight`) to the top of the bottom hemisphere (`z = -halfHeight`). This is represented by two rings at the equator edges. Normals point radially outward with `nz = 0`.
4. The bottom hemisphere sweeps `theta` from PI/2 to PI (the -Z pole). The Z position is `-halfHeight + radius * cos(theta)`.
5. At each ring, `radialSegments + 1` vertices are placed around the Z axis using the azimuthal angle `phi` from 0 to 2PI.
6. UV mapping uses arc-length parameterization along the profile so that texture density is uniform across the caps and the cylindrical body.
7. Indexing follows the same grid-based quad-to-triangle approach as the sphere, skipping degenerate triangles at the two poles.

### Cylinder and Cone

`createConeGeometry` is implemented as a special case of `createCylinderGeometry` with `radiusTop = 0`:

```typescript
const createConeGeometry = (
  radius: number,
  height: number,
  radialSegments = 32,
  heightSegments = 1,
  openEnded = false
): BufferGeometry => {
  return createCylinderGeometry(0, radius, height, radialSegments, heightSegments, openEnded)
}
```

The cylinder algorithm generates rings along the Z axis from `-height/2` to `+height/2`. At each ring, the radius is linearly interpolated between `radiusTop` and `radiusBottom`. If `openEnded` is false, disc caps are generated at the top and bottom as triangle fans.

### Box Generation

The box is built from six independent plane faces, each oriented to face outward. Each face is a subdivided quad (controlled by segment parameters). Vertices are not shared between faces so that normals remain discontinuous at edges.

### Circle Generation

A circle is a triangle fan on the XY plane with center vertex at the origin and edge vertices arranged at radius distance. The normal for all vertices is `(0, 0, 1)`.

---

## 4. GPU Buffer Upload

### Lazy Upload

Geometry does not create GPU buffers at construction time. Buffers are created and populated on the first frame the geometry is actually rendered. This avoids wasting GPU memory on geometries that may be constructed but never drawn (e.g., LOD levels not yet visible).

```typescript
const ensureGPUBuffers = (geometry: BufferGeometry, backend: RenderBackend): void => {
  if (!geometry.needsUpload) return

  for (const [name, attr] of geometry.attributes) {
    if (attr.gpuBuffer === null || attr.needsUpload) {
      attr.gpuBuffer = backend.createVertexBuffer(attr.data, geometry.isDynamic)
      attr.needsUpload = false
    }
  }

  if (geometry.index !== null && (geometry.indexBuffer === null || geometry.needsUpload)) {
    geometry.indexBuffer = backend.createIndexBuffer(geometry.index, geometry.isDynamic)
  }

  geometry.needsUpload = false
}
```

### Static vs Dynamic

By default, geometries are static. The GPU buffer is created with a `STATIC_DRAW` usage hint (WebGL2) or `GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST` without frequent re-upload expectations (WebGPU). When `isDynamic` is `true`, the buffer is created with `DYNAMIC_DRAW` (WebGL2) or mapped for frequent writes (WebGPU). Dynamic geometry is used for:

- Morph targets blended on CPU
- Procedural geometry updated each frame (e.g., terrain deformation, ribbon trails)
- Particle system vertex buffers

### Vertex Binding

On **WebGL2**, a Vertex Array Object (VAO) is created for each geometry-shader pair. The VAO records the `vertexAttribPointer` calls for each attribute, binding the attribute's GPU buffer with its descriptor (size, type, normalized, stride = 0, offset = 0 since buffers are separate):

```typescript
const createVAO = (
  gl: WebGL2RenderingContext,
  geometry: BufferGeometry,
  program: WebGLProgram
): WebGLVertexArrayObject => {
  const vao = gl.createVertexArray()!
  gl.bindVertexArray(vao)

  for (const [name, attr] of geometry.attributes) {
    const location = gl.getAttribLocation(program, name)
    if (location === -1) continue

    gl.bindBuffer(gl.ARRAY_BUFFER, attr.gpuBuffer)
    gl.enableVertexAttribArray(location)

    const glType = attributeTypeToGL(attr.descriptor.type)

    if (attr.descriptor.type === AttributeType.Uint8 ||
        attr.descriptor.type === AttributeType.Uint16 ||
        attr.descriptor.type === AttributeType.Uint32) {
      if (!attr.descriptor.normalized) {
        gl.vertexAttribIPointer(location, attr.descriptor.size, glType, 0, 0)
      } else {
        gl.vertexAttribPointer(location, attr.descriptor.size, glType, true, 0, 0)
      }
    } else {
      gl.vertexAttribPointer(
        location, attr.descriptor.size, glType, attr.descriptor.normalized, 0, 0
      )
    }
  }

  if (geometry.indexBuffer !== null) {
    gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, geometry.indexBuffer)
  }

  gl.bindVertexArray(null)
  return vao
}
```

On **WebGPU**, a vertex buffer layout array is constructed from the attribute descriptors and passed to the render pipeline descriptor. Each attribute maps to a separate `GPUVertexBufferLayout` entry with `arrayStride` equal to `descriptor.size * bytesPerComponent` and a single attribute at offset 0:

```typescript
const buildVertexBufferLayouts = (
  geometry: BufferGeometry
): GPUVertexBufferLayout[] => {
  const layouts: GPUVertexBufferLayout[] = []
  let shaderLocation = 0

  for (const [name, attr] of geometry.attributes) {
    const format = attributeDescriptorToGPUFormat(attr.descriptor)
    const bytesPerComponent = bytesForType(attr.descriptor.type)
    const stride = attr.descriptor.size * bytesPerComponent

    layouts.push({
      arrayStride: stride,
      stepMode: 'vertex',
      attributes: [{
        shaderLocation: shaderLocation++,
        offset: 0,
        format,
      }],
    })
  }

  return layouts
}
```

---

## 5. Geometry Sharing

Geometries are reference-counted so that multiple meshes can share the same vertex and index data without duplication.

```typescript
const retainGeometry = (geometry: BufferGeometry): void => {
  geometry.refCount++
}

const releaseGeometry = (geometry: BufferGeometry, backend: RenderBackend): void => {
  geometry.refCount--

  if (geometry.refCount <= 0) {
    // Free GPU buffers.
    for (const [, attr] of geometry.attributes) {
      if (attr.gpuBuffer !== null) {
        backend.deleteBuffer(attr.gpuBuffer)
        attr.gpuBuffer = null
      }
    }
    if (geometry.indexBuffer !== null) {
      backend.deleteBuffer(geometry.indexBuffer)
      geometry.indexBuffer = null
    }
  }
}
```

When a `Mesh` component is added to an entity, it calls `retainGeometry`. When the component is removed or the entity is destroyed, it calls `releaseGeometry`. This ensures GPU memory is reclaimed as soon as no mesh references the geometry.

A typical sharing pattern:

```typescript
const sharedSphere = createSphereGeometry(1, 32, 16)

// Both meshes reference the same geometry and GPU buffers.
const meshA = createMesh(sharedSphere, materialRed)
const meshB = createMesh(sharedSphere, materialBlue)

// When meshA is destroyed, refCount drops to 1 — buffers remain.
// When meshB is destroyed, refCount drops to 0 — GPU buffers are freed.
```

---

## 6. Merging

The merge utility combines multiple geometries into a single `BufferGeometry`, reducing draw calls for static scene elements that share the same material.

### Interface

```typescript
interface MergeEntry {
  geometry: BufferGeometry
  transform: Mat4   // 4x4 transformation matrix
}

const mergeGeometries = (entries: MergeEntry[]): BufferGeometry
```

### Algorithm

1. **Compute totals.** Sum vertex counts and index counts across all entries to pre-allocate output arrays.

2. **Allocate output arrays.** Create `Float32Array` for positions, normals, UVs, and a `Uint16Array` or `Uint32Array` for indices based on the total vertex count.

3. **Copy and transform.** For each entry, copy its attribute data into the output arrays at the current offset:
   - **Positions** are transformed by the entry's 4x4 matrix (multiply as a point, `w = 1`).
   - **Normals** are transformed by the inverse-transpose of the upper-left 3x3 of the matrix, then re-normalized. This correctly handles non-uniform scale.
   - **UVs** are copied directly without transformation.
   - **Indices** are copied with an offset equal to the running vertex count so they reference the correct vertices in the merged buffer.

4. **Build output geometry.** Construct a new `BufferGeometry` from the merged arrays.

```typescript
const mergeGeometries = (entries: MergeEntry[]): BufferGeometry => {
  let totalVertices = 0
  let totalIndices = 0

  for (const entry of entries) {
    totalVertices += entry.geometry.vertexCount
    totalIndices += entry.geometry.indexCount
  }

  const outPositions = new Float32Array(totalVertices * 3)
  const outNormals = new Float32Array(totalVertices * 3)
  const outUVs = new Float32Array(totalVertices * 2)
  const outIndices = totalVertices <= 65535
    ? new Uint16Array(totalIndices)
    : new Uint32Array(totalIndices)

  let vertexOffset = 0
  let indexOffset = 0

  for (const { geometry, transform } of entries) {
    const positions = geometry.attributes.get('position')!.data as Float32Array
    const normals = geometry.attributes.get('normal')!.data as Float32Array
    const uvData = geometry.attributes.get('uv')!.data as Float32Array
    const normalMatrix = mat3InverseTranspose(mat4ToMat3(transform))

    // Transform and copy positions.
    for (let i = 0; i < geometry.vertexCount; i++) {
      const si = i * 3
      const di = (vertexOffset + i) * 3
      const px = positions[si]
      const py = positions[si + 1]
      const pz = positions[si + 2]

      outPositions[di]     = transform[0] * px + transform[4] * py + transform[8]  * pz + transform[12]
      outPositions[di + 1] = transform[1] * px + transform[5] * py + transform[9]  * pz + transform[13]
      outPositions[di + 2] = transform[2] * px + transform[6] * py + transform[10] * pz + transform[14]
    }

    // Transform and copy normals.
    for (let i = 0; i < geometry.vertexCount; i++) {
      const si = i * 3
      const di = (vertexOffset + i) * 3
      const nx = normals[si]
      const ny = normals[si + 1]
      const nz = normals[si + 2]

      let tnx = normalMatrix[0] * nx + normalMatrix[3] * ny + normalMatrix[6] * nz
      let tny = normalMatrix[1] * nx + normalMatrix[4] * ny + normalMatrix[7] * nz
      let tnz = normalMatrix[2] * nx + normalMatrix[5] * ny + normalMatrix[8] * nz

      // Re-normalize.
      const len = Math.sqrt(tnx * tnx + tny * tny + tnz * tnz)
      if (len > 0) {
        const invLen = 1 / len
        tnx *= invLen
        tny *= invLen
        tnz *= invLen
      }

      outNormals[di]     = tnx
      outNormals[di + 1] = tny
      outNormals[di + 2] = tnz
    }

    // Copy UVs unchanged.
    outUVs.set(uvData, vertexOffset * 2)

    // Copy indices with vertex offset.
    const srcIndex = geometry.index!
    for (let i = 0; i < geometry.indexCount; i++) {
      outIndices[indexOffset + i] = srcIndex[i] + vertexOffset
    }

    vertexOffset += geometry.vertexCount
    indexOffset += geometry.indexCount
  }

  return buildBufferGeometry(outPositions, outNormals, outUVs, outIndices)
}
```

### Use Cases

- **Static environments.** Merge all static props (rocks, fences, crates) that share the same material into a single geometry. This reduces dozens of draw calls to one.
- **Instanced prefabs.** Merge repeated geometry (e.g., tree trunks, fence posts) at different transforms to avoid per-instance draw overhead when hardware instancing is not available or not worth the complexity.
- **Level-of-detail baking.** Merge simplified geometry pieces into combined LOD meshes for distant rendering.

The merged geometry is a new allocation. The source geometries are not modified and can be released independently if no longer needed.
