# Scene Graph

## Flat by Default

Bonobo uses a **flat entity list** by default, not a hierarchical scene graph. This is a deliberate design choice for performance and simplicity.

### Why Flat?

1. **Performance** — No recursive matrix propagation, no deep tree traversals
2. **Cache-friendly** — All entity data in contiguous typed arrays
3. **Simplicity** — Easier to understand and debug
4. **Automatic instancing** — Easier to batch when entities aren't scattered across a hierarchy

### Entity Storage

All entities are stored in flat typed arrays (Structure of Arrays):

```ts
// Pre-allocated for MAX_ENTITIES (e.g. 50,000)
positions:      Float32Array(MAX * 3)
rotations:      Float32Array(MAX * 3)     // Euler XYZ
scales:         Float32Array(MAX * 3)
worldMatrices:  Float32Array(MAX * 16)
colors:         Float32Array(MAX * 4)     // RGBA
flags:          Uint32Array(MAX)
boundingSpheres: Float32Array(MAX * 4)    // cx, cy, cz, radius
geometryIds:    Uint32Array(MAX)
materialIds:    Uint32Array(MAX)
sortKeys:       Uint32Array(MAX)
```

Each entity is just an index into these arrays. No parent-child relationships tracked by default.

## Parent-Child Transforms (Future)

Parent-child relationships can be added as an **opt-in layer** in v2:

```ts
// Additional typed arrays for hierarchy (opt-in)
parentIds:      Int32Array(MAX)    // -1 = no parent
firstChildIds:  Int32Array(MAX)    // -1 = no children
nextSiblingIds: Int32Array(MAX)    // -1 = last sibling
```

When hierarchy is enabled:
- Entities with a parent propagate their transform from parent's world matrix
- Matrix updates become incremental: mark parent dirty → mark all descendants dirty
- Culling can use parent bounding volumes for early-out

This adds complexity but keeps it optional. Most games don't need deep hierarchies for every entity.

## Groups (Future)

For grouping entities without full parent-child transforms:

```ts
const group = world.createGroup()
group.add(entity1)
group.add(entity2)
group.setPosition(0, 10, 0)  // Offset all children
```

Implemented as a transform offset applied during matrix computation, without recursive propagation.

## Current Implementation (v1)

In v1, there is **no scene graph**. Every entity's world matrix is computed independently from its local TRS (position, rotation, scale):

```ts
for (let i = 0; i < entityCount; i++) {
  if (flags[i] & DIRTY) {
    mat4_from_trs(this.worldMatrices, i, this.positions, this.rotations, this.scales)
    flags[i] &= ~DIRTY
  }
}
```

No parent, no hierarchy, no transform propagation. Just flat, independent entities.
