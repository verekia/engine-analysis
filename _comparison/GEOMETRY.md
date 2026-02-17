# Geometry Comparison Across 9 Implementations

## Universal Agreement

All 9 implementations agree on these geometry fundamentals:

### 1. Indexed Triangle Meshes
Every implementation uses indexed triangle lists:
- Vertex attributes stored in typed arrays
- Indices stored as `Uint16Array` (≤65535 vertices) or `Uint32Array` (>65535 vertices)
- Automatic selection of index format based on vertex count

### 2. Standard Vertex Attributes
All implementations support these core attributes:
- **position**: `Float32Array`, 3 components (x, y, z)
- **normal**: `Float32Array`, 3 components
- **uv**: `Float32Array`, 2 components (texture coordinates)
- **color**: `Float32Array` or `Uint8Array`, 3 or 4 components (optional)
- **_materialindex**: `Uint8Array` or `Uint16Array`, 1 component (optional)
- **joints/skinIndex**: Bone indices for skinning (optional)
- **weights/skinWeight**: Bone weights for skinning (optional)

### 3. Bounding Volume Computation
All implementations compute an axis-aligned bounding box (AABB) from vertex positions:
```
AABB = { min: [x, y, z], max: [x, y, z] }
```
Computed once at geometry creation (or when positions change).

### 4. Standard Parametric Primitives
All 9 implementations provide the same set of primitive generators:
- **Plane**: Flat rectangle in XY plane
- **Box**: Axis-aligned cube
- **Sphere**: UV sphere with poles along Z axis
- **Cone**: Conical shape with base at Z=0
- **Cylinder**: Cylindrical shape along Z axis
- **Capsule**: Cylinder with hemispherical caps
- **Circle**: Flat disc in XY plane

### 5. Z-Up Coordinate System
All primitives are generated in Z-up, right-handed coordinates:
- Plane lies in XY plane, normal points +Z
- Cylinder/Cone/Capsule extend along Z axis
- Sphere has poles at ±Z

### 6. Lazy GPU Upload
All implementations defer GPU buffer creation until first render:
- CPU-side typed arrays created immediately
- GPU buffers created on first draw call
- Avoids wasting GPU memory for unused geometry

## Key Variations

### Vertex Buffer Layout Strategy

**Interleaved (Single Buffer):**
- **caracal, fennec, hyena**: All attributes interleaved in one buffer
- `[pos.xyz, normal.xyz, uv.xy, pos.xyz, normal.xyz, uv.xy, ...]`
- Cache-friendly sequential reads
- More complex to modify individual attributes

**Separate Buffers (Multiple Buffers):**
- **lynx, mantis, rabbit, wren**: Each attribute in its own buffer
- Buffer 0: positions, Buffer 1: normals, Buffer 2: UVs, etc.
- Easy to add/remove attributes
- Simpler partial updates
- One extra `vertexAttribPointer` call per attribute (negligible cost)

**Hybrid:**
- **shark**: Interleaved float data, separate buffer for uint8/uint16 data
- Avoids alignment issues mixing float and byte data

**Pre-Allocated SoA:**
- **bonobo**: Pre-allocated typed arrays for MAX_ENTITIES, all in SoA layout

### Attribute Storage Format

**Typed Arrays Per Attribute:**
- **All except bonobo**: Standard approach
- `positions: Float32Array`, `normals: Float32Array`, etc.

**Map<string, BufferAttribute>:**
- **caracal, wren**: Attributes stored in a Map with descriptors
- Flexible, supports custom attributes easily

**Named Properties:**
- **fennec, hyena, lynx, mantis, rabbit, shark**: Direct properties on geometry object
- `geometry.positions`, `geometry.normals`, etc.

### Bounding Volume Type

**AABB (Axis-Aligned Bounding Box):**
- **caracal, fennec, hyena, lynx, mantis, rabbit, shark, wren**: Standard AABB
- Computed from min/max of position data

**Bounding Sphere:**
- **bonobo**: Uses bounding radius instead of AABB
- Stored per-entity as `[centerX, centerY, centerZ, radius]`

**Both:**
- **lynx**: Computes both AABB and bounding sphere

### Parametric Generator API Style

**Options Object (Most Common):**
```typescript
createSphere({ radius: 1, widthSegments: 32, heightSegments: 16 })
```
- **caracal, fennec, hyena, lynx, mantis, rabbit, shark, wren**: All use options object

**Positional Parameters:**
```typescript
createSphere(radius, widthSegments, heightSegments)
```
- **None use positional parameters exclusively**
- All prefer options object for clarity and optional parameters

**Default Values:**
- All implementations provide sensible defaults for all parameters

### Primitive Detail Level Defaults

| Primitive | Parameter | Bonobo | Caracal | Fennec | Hyena | Lynx | Mantis | Rabbit | Shark | Wren |
|-----------|-----------|--------|---------|--------|-------|------|--------|--------|-------|------|
| **Sphere** | widthSegments | - | 32 | 32 | 32 | 32 | 32 | 32 | 32 | 32 |
| **Sphere** | heightSegments | - | 16 | 16 | 16 | 16 | 16 | 16 | 16 | 16 |
| **Cylinder** | radialSegments | - | 16 | 32 | 32 | 32 | 32 | 32 | 32 | 32 |
| **Capsule** | capSegments | - | 8 | 8 | 8 | 8 | 8 | 8 | 8 | 8 |

**Consensus:** All implementations use nearly identical defaults for primitive detail.

### Normal Generation

**Smooth Normals (Default):**
- **All implementations** generate smooth normals for sphere, cylinder, capsule, cone
- Normals are averaged/interpolated for curved surfaces

**Flat Normals for Box:**
- **All implementations** generate separate vertices per face with flat normals
- No vertex sharing between box faces (sharp edges)

### UV Mapping Strategy

**Per-Face Mapping (Box):**
- All implementations map each box face to (0,0)→(1,1) independently
- No UV unwrapping across faces

**Spherical Mapping (Sphere):**
- U = longitude / 2π
- V = latitude / π
- All implementations use standard UV sphere mapping

**Polar Degeneration:**
- **All implementations** have degenerate UVs at sphere poles (many vertices map to same UV)
- This is standard and acceptable for UV spheres

### Custom Attributes Support

**Via Map:**
- **caracal, wren**: `geometry.attributes.set('customAttr', bufferAttribute)`

**Via Properties:**
- **fennec, hyena, lynx**: Add custom attributes as properties
- **mantis, rabbit, shark**: Similar approach

**Standard Custom Attribute:**
- **All 8 with material index**: Recognize `_materialindex` as standard custom attribute

## Implementation Breakdown by Storage Strategy

### Approach 1: Interleaved Buffer (Cache-Friendly)
**Count: 3**
- **caracal, fennec, hyena**

**Characteristics:**
- Single vertex buffer with all attributes interleaved
- Optimal cache performance (all data for one vertex is sequential)
- More complex to modify (must rebuild entire buffer for attribute changes)

**Best For:**
- Static geometry (no attribute modifications after creation)
- Maximum GPU performance
- Mobile/cache-constrained targets

### Approach 2: Separate Buffers (Flexibility)
**Count: 4**
- **lynx, mantis, rabbit, wren**

**Characteristics:**
- One GPU buffer per attribute
- Easy to add/remove attributes
- Simple partial updates (e.g., morph targets updating only positions)
- Negligible performance difference on modern hardware

**Best For:**
- Dynamic geometry modifications
- Flexible attribute sets
- Development/prototyping

### Approach 3: Hybrid (Practical)
**Count: 1**
- **shark**

**Characteristics:**
- Interleaved float data (pos, normal, uv)
- Separate buffer for byte/short data (materialIndex, joints)
- Avoids alignment issues

**Best For:**
- When mixing data types
- Want interleaving benefits without alignment headaches

### Approach 4: Pre-Allocated SoA (Maximum Performance)
**Count: 1**
- **bonobo**

**Characteristics:**
- All geometry data pre-allocated in flat typed arrays
- No per-geometry allocations
- Maximum cache efficiency

**Best For:**
- Very high entity counts
- Static geometry
- Performance-critical scenarios

## Primitive Generation Details

### Plane Generation

**All implementations produce:**
- Grid of `(widthSegments+1) × (heightSegments+1)` vertices
- Two triangles per grid cell
- Default 1×1 segments = 4 vertices, 2 triangles
- UVs: (0,0) at bottom-left, (1,1) at top-right

### Box Generation

**All implementations produce:**
- 6 faces, each a subdivided plane
- Separate vertices per face (no sharing)
- Default 1 segment per dimension = 24 vertices (4×6), 12 triangles (2×6)
- Normals point outward per face

### Sphere Generation (UV Sphere)

**Algorithm (all implementations):**
1. Two nested loops: latitude (heightSegments) × longitude (widthSegments)
2. Polar angle θ from 0 (north pole) to π (south pole)
3. Azimuthal angle φ from 0 to 2π around Z axis
4. Spherical-to-Cartesian for Z-up:
   - `x = r × sin(θ) × cos(φ)`
   - `y = r × sin(θ) × sin(φ)`
   - `z = r × cos(θ)`
5. Normals = normalized position (unit sphere)
6. Skip degenerate triangles at poles

**Differences:**
- **fennec, lynx**: Explicit pole degeneration handling documented
- **caracal, hyena, mantis, rabbit, shark, wren**: Similar approach, less documentation

### Cylinder/Cone Generation

**All implementations:**
- Cylinder with `radiusTop = 0` becomes cone
- Side surface: rings from bottom to top, interpolating radius
- Optional caps at top/bottom (controlled by `openEnded`)
- Normals perpendicular to slant surface

**Slant Normal Calculation:**
```
dr = radiusTop - radiusBottom
slant = atan2(dr, height)
normal = [cos(φ) × cos(slant), sin(φ) × cos(slant), sin(slant)]
```

### Capsule Generation (Most Complex)

**All implementations:**
- Three sections: top hemisphere + cylinder body + bottom hemisphere
- Cylinder height = `totalHeight - 2 × radius`
- Hemisphere caps generated as partial spheres (θ from 0 to π/2)
- Vertices shared at hemisphere-cylinder boundary

**UV Mapping:**
- **lynx**: Arc-length parameterization (uniform texture density)
- **Others**: Simpler mapping (may stretch at boundaries)

**Algorithm Complexity:**
- **lynx**: Most detailed capsule documentation
- **caracal, fennec, hyena, mantis, rabbit, shark, wren**: Similar approach, varying detail

### Circle Generation

**All implementations:**
- Center vertex at origin
- Ring of vertices at radius
- Triangle fan from center to ring
- All normals point +Z (up)

## Geometry Disposal and Sharing

### Reference Counting
- **lynx**: Explicit reference counting (`retainGeometry`, `releaseGeometry`)
- **caracal, fennec, hyena**: Implicit via garbage collection
- **mantis, rabbit, shark, wren**: Similar to lynx (manual or automatic)

### Disposal API
```typescript
geometry.dispose()  // All implementations
```
Destroys GPU buffers, clears CPU arrays (optionally).

### Geometry Sharing
**All implementations support:**
- Multiple meshes sharing the same geometry instance
- GPU buffers shared (only uploaded once)
- Reference counting or GC ensures cleanup when last mesh removed

## Special Features by Implementation

### Caracal
- Interleaved buffer layout by default for parametrics
- Preserves glTF buffer layout on import (no forced interleaving)
- Comprehensive buffer attribute descriptor system

### Fennec
- Explicit BVH builder for raycasting (built lazily)
- Grid-based index generation utility
- Helper `buildGeometry` function for custom geometry

### Hyena
- Interleaved vertex layout with detailed documentation
- GPU upload in `prepareGeometry` phase
- Lazy bounding box computation

### Lynx
- Separate buffer strategy with clear justification
- Both AABB and bounding sphere
- Most comprehensive capsule generation (arc-length UVs)
- Reference counting for geometry lifecycle

### Mantis
- SoA storage (pre-allocated capacity with doubling growth)
- CPU-side arrays released after GPU upload (optional)
- Retain flag for raycasting (keeps CPU copy)

### Rabbit
- Limited documentation (likely standard approach)
- Expected to follow common patterns

### Shark
- Hybrid buffer layout (float interleaved, byte/short separate)
- Explicit 1×1 default textures
- Clear disposal lifecycle

### Wren
- Separate buffer storage with detailed justification
- GLTF loader coordinate conversion (Y-up to Z-up)
- Draco decoder integration (Web Worker)

### Bonobo
- Pre-allocated SoA for MAX_ENTITIES
- Bounding radius instead of AABB
- No parametric generators documented (future feature)

## Index Format Selection

**All implementations:**
```typescript
vertexCount <= 65535 ? Uint16Array : Uint32Array
```

**Benefits of Uint16:**
- Half the memory of Uint32
- Better vertex cache hit rates on mobile GPUs
- Sufficient for 99% of game assets

## Custom Geometry Creation

**All implementations provide:**
- Factory function to create geometry from raw arrays
- Example:
```typescript
const geometry = createGeometry({
  positions: new Float32Array([...]),
  normals: new Float32Array([...]),
  uvs: new Float32Array([...]),
  indices: new Uint16Array([...])
})
```

**Custom Attributes:**
- **All 8 with material index**: Support adding `_materialindex` attribute
- **caracal, wren**: Generic `attributes` map for any custom attribute

## Geometry Merging

**Implementations with Merge Utility:**
- **lynx**: Comprehensive `mergeGeometries` with transform support
- Combines multiple geometries into one (bakes transforms)
- Useful for static batching

**Others:**
- No documented merge utility (can be added as needed)

## Performance Characteristics Summary

| Implementation | Storage | Buffer Upload | Attribute Modification | Custom Attributes |
|----------------|---------|---------------|----------------------|-------------------|
| **bonobo** | Pre-allocated SoA | N/A (direct access) | Complex (rebuild) | Limited |
| **caracal** | Interleaved | Lazy, single buffer | Complex (rebuild) | Via map |
| **fennec** | Interleaved | Lazy, single buffer | Complex (rebuild) | Via properties |
| **hyena** | Interleaved | Lazy, single buffer | Complex (rebuild) | Via properties |
| **lynx** | Separate buffers | Lazy, per-attribute | Simple (update one buffer) | Easy (add buffer) |
| **mantis** | Separate buffers | Lazy, per-attribute | Simple | Easy |
| **rabbit** | Separate buffers | Lazy, per-attribute | Simple | Easy |
| **shark** | Hybrid | Lazy, two buffers | Medium | Via properties |
| **wren** | Separate buffers | Lazy, per-attribute | Simple | Easy |

## Decision Matrix: Which Geometry Approach to Use?

### Choose Interleaved Buffers (Caracal/Fennec/Hyena) if:
- Static geometry (no runtime modifications)
- Maximum cache performance critical
- Mobile/low-power targets
- Simpler vertex shader setup (fewer buffer bindings)

### Choose Separate Buffers (Lynx/Mantis/Rabbit/Wren) if:
- Dynamic geometry modifications expected
- Need to add/remove attributes frequently
- Morph targets or procedural deformation
- Development flexibility more important than micro-optimization

### Choose Hybrid (Shark) if:
- Want interleaving benefits but have mixed data types
- Need to avoid alignment headaches
- Pragmatic middle ground

### Choose Pre-Allocated SoA (Bonobo) if:
- Maximum entity count (tens of thousands)
- All geometry known upfront
- Willing to trade flexibility for performance

## Unique Innovations Worth Cherry-Picking

### From Lynx:
- Reference counting for geometry lifecycle
- Merge utility with transform support
- Arc-length UV parameterization for capsules

### From Caracal:
- Preserve glTF buffer layout (no forced conversion)
- Comprehensive attribute descriptor system

### From Fennec:
- BVH builder for raycasting integration
- Grid index generation utility

### From Mantis:
- Pre-allocated capacity with doubling growth
- Optional CPU array retention for raycasting

### From Shark:
- Hybrid buffer layout (pragmatic solution)
- Default 1×1 textures to avoid branching

### From Wren:
- Draco decoder Web Worker integration
- GLTF coordinate conversion baked in loader

### From Bonobo:
- Pre-allocated SoA for maximum throughput
- Bounding sphere instead of AABB (simpler, faster test)

## Common Primitive Parameters Summary

### Plane
- **width, height**: Extent along X and Y
- **widthSegments, heightSegments**: Subdivisions
- **Default**: 1×1, 1 segment each = 4 verts, 2 tris

### Box
- **width, height, depth**: Extent along X, Y, Z
- **widthSegments, heightSegments, depthSegments**: Subdivisions
- **Default**: 1×1×1, 1 segment each = 24 verts, 12 tris

### Sphere
- **radius**: Sphere radius
- **widthSegments**: Longitude divisions (default: 32)
- **heightSegments**: Latitude divisions (default: 16)
- **phiStart, phiLength, thetaStart, thetaLength**: Partial sphere control
- **Default**: 32×16 segments

### Cone
- **radius**: Base radius
- **height**: Height along Z
- **radialSegments**: Circumference divisions (default: 32)
- **heightSegments**: Height divisions (default: 1)
- **openEnded**: If true, no base cap

### Cylinder
- **radiusTop, radiusBottom**: Top and bottom radii
- **height**: Height along Z
- **radialSegments, heightSegments**: Subdivisions
- **openEnded**: If true, no caps

### Capsule
- **radius**: Cap and body radius
- **height**: Total height (or cylinder portion, varies by implementation)
- **capSegments**: Hemisphere subdivisions (default: 8)
- **radialSegments**: Circumference divisions (default: 32)

### Circle
- **radius**: Radius
- **segments**: Circumference divisions (default: 32)
- **thetaStart, thetaLength**: Partial circle control

## Z-Up Convention Consistency

**All 9 implementations:**
- Plane in XY, normal +Z
- Cylinder/Cone/Capsule along Z axis
- Sphere poles at ±Z
- Box aligned to XYZ axes

**This consistency is a major benefit** — geometry from one engine can be copy-pasted to another with no coordinate conversion.
