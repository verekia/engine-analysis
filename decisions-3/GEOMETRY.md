# GEOMETRY.md - Geometry and Primitives

## Indexed Triangle Meshes

**Decision: Indexed triangle lists with automatic index format selection** (universal agreement)

- Vertex attributes stored in typed arrays
- Indices: `Uint16Array` for meshes with <=65535 vertices, `Uint32Array` otherwise
- Automatic selection saves 50% index memory for the vast majority of game assets

## Vertex Buffer Layout

**Decision: Separate buffers per attribute** (Lynx, Mantis, Rabbit, Wren approach)

Each vertex attribute gets its own GPU buffer:
- Buffer 0: positions (Float32Array, 3 components)
- Buffer 1: normals (Float32Array, 3 components)
- Buffer 2: UVs (Float32Array, 2 components)
- Buffer 3: colors (Float32Array, 4 components) - optional
- Buffer 4: materialIndex (Uint8Array, 1 component) - optional
- Buffer 5: joints (Uint8Array, 4 components) - optional, skinning
- Buffer 6: weights (Float32Array, 4 components) - optional, skinning

### Why Separate Over Interleaved

Interleaved buffers (Caracal, Fennec, Hyena) offer slightly better cache performance for sequential reads, but separate buffers provide:

- **Simpler attribute management**: Add or remove attributes without rebuilding the entire buffer.
- **Cleaner glTF import**: glTF stores attributes separately. Interleaving requires an extra copy step.
- **Mixed data types without alignment issues**: Uint8 material index data alongside Float32 positions would cause padding in an interleaved layout.
- **Negligible performance difference**: On modern GPUs, the vertex fetch unit handles separate buffers with minimal overhead. The bottleneck is draw call count and state changes, not vertex attribute layout.
- **Partial updates**: Update only positions (e.g., for morph targets) without touching normals or UVs.

The performance difference between interleaved and separate is measurable only in synthetic benchmarks - it is lost in the noise of a real game frame with 2000 draw calls.

## Standard Vertex Attributes

| Attribute | Type | Components | Required | Notes |
|-----------|------|-----------|----------|-------|
| `position` | Float32 | 3 | Yes | x, y, z |
| `normal` | Float32 | 3 | Yes | Normalized surface normal |
| `uv` | Float32 | 2 | Yes | Texture coordinates |
| `color` | Float32 | 4 | No | Vertex color (RGBA, multiplicative) |
| `_materialindex` | Uint8 | 1 | No | Palette index (flat interpolation) |
| `joints` | Uint8 | 4 | No | Bone indices for skinning |
| `weights` | Float32 | 4 | No | Bone weights for skinning |

## Bounding Volume

**Decision: AABB per geometry** (8/9 implementations)

```typescript
interface AABB {
  min: Vec3  // [minX, minY, minZ]
  max: Vec3  // [maxX, maxY, maxZ]
}
```

Computed once at geometry creation from vertex positions. Used for frustum culling and BVH construction. Bounding spheres (Bonobo) are faster to test but looser fits; AABBs are the standard choice and integrate better with BVH construction.

## Parametric Primitives

All 7 primitive generators follow the same pattern:

```typescript
const geometry = createBoxGeometry({ width: 1, height: 1, depth: 1 })
const geometry = createSphereGeometry({ radius: 1, widthSegments: 32, heightSegments: 16 })
```

**Options objects** (not positional parameters) for clarity and optional parameters with sensible defaults.

### Primitive List

All generated in Z-up, right-handed coordinates:

| Primitive | Key Parameters | Default | Vertices (default) | Triangles (default) |
|-----------|---------------|---------|-------------------|-------------------|
| **Plane** | width, height, widthSegs, heightSegs | 1x1, 1x1 segs | 4 | 2 |
| **Box** | width, height, depth, wSegs, hSegs, dSegs | 1x1x1, 1x1x1 segs | 24 | 12 |
| **Sphere** | radius, widthSegs, heightSegs | r=1, 32x16 | ~530 | ~992 |
| **Cone** | radius, height, radialSegs, heightSegs, openEnded | r=1, h=1, 32 segs | ~66 | ~64 |
| **Cylinder** | radiusTop, radiusBottom, height, radialSegs, heightSegs, openEnded | r=1, h=1, 32 segs | ~132 | ~128 |
| **Capsule** | radius, height, capSegs, radialSegs | r=0.5, h=1, 8 cap, 32 radial | ~530 | ~1024 |
| **Circle** | radius, segments | r=1, 32 segs | 33 | 32 |

### Z-Up Convention

- **Plane**: Lies in XY plane, normal points +Z
- **Cylinder/Cone/Capsule**: Extend along Z axis
- **Sphere**: Poles at +Z (north) and -Z (south)
- **Box**: Aligned to XYZ axes

### Normal Generation

- **Sphere, Cylinder, Capsule, Cone**: Smooth normals (averaged/interpolated for curved surfaces)
- **Box**: Flat normals per face (separate vertices per face, no sharing between faces for sharp edges)

### UV Mapping

- **Box**: Each face mapped independently to (0,0)-(1,1)
- **Sphere**: Standard spherical UV mapping (u = longitude/2pi, v = latitude/pi)
- **Cylinder/Cone**: Cylindrical mapping for sides, planar for caps
- **Capsule**: Standard mapping (arc-length parameterization for uniform texture density is not worth the complexity)
- **Circle**: Radial mapping from center

## GPU Buffer Upload

**Decision: Lazy upload on first render** (universal agreement)

CPU-side typed arrays are created immediately when geometry is constructed. GPU buffers are created on the first draw call that uses the geometry. This avoids wasting GPU memory for geometry that is created but never rendered.

## Geometry Sharing

Multiple meshes can share the same geometry instance. GPU buffers are uploaded once and referenced by all meshes using that geometry.

```typescript
const boxGeo = createBoxGeometry()
const mesh1 = createMesh(boxGeo, redMaterial)
const mesh2 = createMesh(boxGeo, blueMaterial)
// One GPU upload, two draw calls
```

## Geometry Disposal

```typescript
geometry.dispose()
```

Destroys GPU buffers and optionally releases CPU arrays. CPU arrays can be retained if raycasting needs them (BVH operates on CPU-side vertex data).

## Custom Geometry

```typescript
const geometry = createGeometry({
  positions: new Float32Array([...]),
  normals: new Float32Array([...]),
  uvs: new Float32Array([...]),
  indices: new Uint16Array([...]),
  // Optional
  colors: new Float32Array([...]),
  materialIndices: new Uint8Array([...]),
})
```

Any additional custom attributes can be added via a generic `attributes` map for extensibility.
