# Math

## Coordinate System

**Z-up, right-handed coordinate system**

```
      Z (up)
      │
      │
      └─── X
     ╱
    ╱
   Y
```

This is the coordinate system used throughout the rabbit implementation:
- **Z-axis**: points up (vertical)
- **X-axis**: points right
- **Y-axis**: points forward
- **Right-handed**: cross product X × Y = Z

This differs from Three.js which uses Y-up. The Z-up convention is common in CAD, game engines (Unreal), and many 3D modeling tools.

## Math Utilities

The implementation includes standard vector and matrix math utilities:

### Vector Types
- **Vec2**: 2D vectors
- **Vec3**: 3D vectors (position, direction, color)
- **Vec4**: 4D vectors (homogeneous coordinates, quaternions as XYZW)

### Matrix Types
- **Mat3**: 3×3 matrices (normal transforms, 2D transforms)
- **Mat4**: 4×4 matrices (world transforms, view-projection)

### Quaternions
- Used for rotations (gimbal-lock free)
- Interpolation (slerp) for smooth animation transitions

### Common Operations
- Vector arithmetic (add, subtract, scale, dot product, cross product)
- Matrix multiplication, inversion, transposition
- Quaternion multiplication and interpolation
- Transform hierarchies (world matrix = parent world × local)

## Spherical Coordinates

Used by orbit controls, adapted for Z-up:

```typescript
// Spherical to Cartesian (Z-up)
x = radius * sin(theta) * cos(phi)
y = radius * sin(theta) * sin(phi)
z = radius * cos(theta)

// Where:
// theta = polar angle from Z axis (0 = looking down, π = looking up)
// phi = azimuthal angle in XY plane
```

## AABB (Axis-Aligned Bounding Box)

Used for frustum culling and raycasting:

```typescript
interface AABB {
  min: Vec3  // minimum corner
  max: Vec3  // maximum corner
}
```

Operations:
- Union: merge two AABBs
- Transform: apply matrix to AABB (results in new AABB)
- Intersection: test if two AABBs overlap
- Contains point: test if point is inside AABB
- Surface area: used in BVH construction (SAH)
