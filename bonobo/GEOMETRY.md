# Geometry

## Geometry Interface

```ts
interface Geometry {
  id: number
  vertexBuffer: GPUBuffer       // interleaved [pos, normal, color, ...]
  indexBuffer: GPUBuffer
  indexCount: number
  indexFormat: 'uint16' | 'uint32'
  boundingRadius: number        // for frustum culling
}
```

## Geometry Registry

Geometries are stored in a registry within the World:

```ts
// Inside World class
geometryRegistry: Map<geometryId, Geometry>
```

Entities reference geometries by ID:

```ts
geometryIds: Uint32Array(MAX_ENTITIES)
```

## Vertex Format

Interleaved vertex layout for cache efficiency:

```
Position:  vec3<f32>  (12 bytes)
Normal:    vec3<f32>  (12 bytes)
Color:     vec4<f32>  (16 bytes, optional)
UV:        vec2<f32>  (8 bytes, for textured materials)
                      ──────────────
                      48 bytes/vertex (with color + UV)
```

## Primitive Generators

Built-in primitive generators for common shapes:

### Box

```ts
function createBox(
  width: number,
  height: number,
  depth: number
): Geometry
```

### Sphere

```ts
function createSphere(
  radius: number,
  widthSegments: number,
  heightSegments: number
): Geometry
```

### Plane

```ts
function createPlane(
  width: number,
  height: number,
  widthSegments?: number,
  heightSegments?: number
): Geometry
```

### Cone

```ts
function createCone(
  radius: number,
  height: number,
  radialSegments: number
): Geometry
```

### Cylinder

```ts
function createCylinder(
  radiusTop: number,
  radiusBottom: number,
  height: number,
  radialSegments: number
): Geometry
```

### Capsule

```ts
function createCapsule(
  radius: number,
  height: number,
  radialSegments: number
): Geometry
```

### Circle

```ts
function createCircle(
  radius: number,
  segments: number
): Geometry
```

## Custom Geometry

Generic geometry creation from raw vertex/index data:

```ts
function createGeometry(
  vertices: Float32Array,   // interleaved vertex data
  indices: Uint16Array | Uint32Array
): Geometry
```

This enables importing meshes from external sources (glTF, OBJ, etc.) as raw arrays.

## Bounding Volume

Each geometry stores a bounding radius for frustum culling:

```ts
boundingRadius: number  // Radius of bounding sphere in local space
```

Combined with entity world position to compute world-space bounding sphere:

```ts
// Per entity
boundingSpheres: Float32Array(MAX * 4)  // cx, cy, cz, radius
```

The bounding sphere is updated when the entity's transform changes.

## Index Format

Supports both 16-bit and 32-bit indices:

```ts
indexFormat: 'uint16' | 'uint32'
```

- Use `uint16` for meshes with < 65,536 vertices (most cases)
- Use `uint32` for very high-poly meshes
