# Scene Graph Comparison Across 9 Implementations

## Universal Agreement

All 9 implementations agree on these fundamental scene graph principles:

### 1. Hierarchical Transform System
Every implementation uses a parent-child node hierarchy where:
- Each node has a local transform (position, rotation, scale)
- World transforms are computed by concatenating parent transforms
- The pattern is: `worldMatrix = parent.worldMatrix × localMatrix`

### 2. Coordinate System
8 out of 9 implementations use **Z-up, right-handed** coordinates:
- +X = right
- +Y = forward
- +Z = up
- Cross product: X × Y = Z

The exception is **bonobo**, which currently uses a flat entity list but plans to add hierarchy as opt-in.

### 3. Dirty Flag Propagation
All implementations mark nodes and their descendants as dirty when transforms change:
- Setting position/rotation/scale sets a dirty flag
- Dirty flags propagate recursively to all children
- World matrices are only recomputed for dirty nodes during update phase

### 4. Node Types
All implementations support these standard node types:
- **Group/Node** (base container with transform)
- **Mesh** (geometry + material)
- **SkinnedMesh** (mesh + skeleton for animation)
- **Camera** (projection + view)
- **DirectionalLight** (shadow-casting light)
- **AmbientLight** (global illumination)

### 5. Bone Attachment
All implementations (except bonobo v1) support attaching static meshes to skeleton bones by simply parenting the mesh to a bone node—no special API required.

## Key Variations

### Storage Strategy

**Flat Structure-of-Arrays (SoA):**
- **bonobo**: Deliberately flat entity list with no hierarchy by default. All data in contiguous typed arrays indexed by entity ID. Future v2 will add opt-in hierarchy.
- **hyena**: Z-up hierarchy with standard object graph.
- **mantis**: SoA storage for transforms (positions, rotations, scales, world matrices all in separate Float32Arrays) with breadth-first traversal order cached.

**Traditional Object Graph:**
- **caracal, fennec, lynx, rabbit, shark, wren**: Standard JavaScript objects with parent/child references in memory.

### Matrix Storage

**Separate SoA Arrays:**
- **bonobo**: `worldMatrices: Float32Array(MAX * 16)` - all matrices in one flat buffer
- **mantis**: Similar approach with pre-allocated capacity and doubling growth
- **hyena**: SoA with positions/rotations/scales/matrices in parallel arrays

**Per-Node Storage:**
- **caracal, fennec, lynx, rabbit, shark, wren**: Each node object owns its matrices

### Update Strategy

**Breadth-First (Parents Before Children):**
- **mantis**: Pre-computed breadth-first traversal order, rebuilt only when hierarchy changes
- **hyena**: SoA traversal in breadth-first order

**Depth-First Recursion:**
- **caracal, fennec, lynx, rabbit, shark, wren**: Recursive depth-first traversal during update

**No Traversal (Flat):**
- **bonobo v1**: Linear iteration over all entities, each computes its own matrix independently

### Dirty Flag Implementation

**Two-Level Flags:**
- **caracal, fennec, shark**: Separate `_dirtyLocal` (local matrix needs recompute) and `_dirtyWorld` (world matrix needs recompute)
- **hyena, mantis**: Single dirty flag per node

**Early Exit Optimization:**
- **mantis, hyena, fennec**: Check if already dirty before propagating to avoid O(n²) cascading
- Others propagate unconditionally

### Frustum Culling Integration

**AABB Per Node:**
- **caracal, fennec, hyena, lynx, mantis, rabbit, shark, wren**: Each mesh node has local AABB + world AABB
- **bonobo**: Uses bounding spheres instead (center + radius)

**Hierarchical Culling:**
- **caracal, lynx, shark**: If parent's AABB is outside frustum, skip entire subtree
- **fennec, mantis**: Flat render list approach—frustum test happens on pre-collected mesh list

### Contiguous Matrix Storage for GPU Upload

**Single Flat Buffer:**
- **bonobo**: All world matrices packed for single GPU buffer write
- **caracal**: Contiguous `worldMatrixBuffer` for efficient GPU upload
- **mantis**: Pre-allocated flat matrix pool

**Individual Node Matrices:**
- **fennec, hyena, lynx, rabbit, shark, wren**: Matrices uploaded per-object via dynamic uniform buffer offsets

## Implementation Breakdown by Approach

### Approach 1: Flat SoA with No Hierarchy (Performance-First)
**Count: 1**
- **bonobo**

**Characteristics:**
- Maximum performance for static scenes
- Cache-friendly linear iteration
- Automatic instancing opportunities
- No parent-child transform propagation in v1
- Opt-in hierarchy planned for v2

**Best For:**
- High entity counts (50,000+)
- Mostly independent objects
- Minimal hierarchical relationships

### Approach 2: SoA with Hierarchy (Cache-Friendly + Hierarchy)
**Count: 2**
- **mantis**, **hyena**

**Characteristics:**
- Contiguous typed arrays for cache efficiency
- Pre-computed breadth-first traversal order
- Minimal per-frame allocation
- Hierarchy changes are more expensive (rebuild traversal order)

**Best For:**
- Scenes with moderate hierarchy changes
- Mobile/performance-critical scenarios
- Predictable memory usage

### Approach 3: Traditional Object Graph (Flexibility-First)
**Count: 6**
- **caracal**, **fennec**, **lynx**, **rabbit**, **shark**, **wren**

**Characteristics:**
- Familiar JavaScript object model
- Easy to inspect/debug
- Flexible hierarchy modifications
- More allocation overhead

**Best For:**
- Rapid development
- Dynamic scene changes
- Complex hierarchical scenes

## Specialized Features

### Bone Attachment Special Cases

**Standard (Bone is a Node):**
- **caracal, hyena, lynx, rabbit, shark, wren**: Bones are regular scene nodes; `addChild(bone, mesh)` just works

**Dedicated BoneAttachment Node:**
- **fennec**: Special `BoneAttachment` node type that overrides world matrix computation to follow a bone

**No Special Support:**
- **bonobo v1**: No skeleton system yet
- **mantis**: Bones are nodes (standard approach)

### Euler Angles

**Quaternion-Only:**
- **All implementations** store rotation as quaternions internally

**Euler Support:**
- **caracal**: Dedicated `Euler` object synced with quaternion
- **shark**: `setRotationFromEuler()` helper methods
- Others provide conversion utilities but don't store Euler

### Visibility Flags

**Node-Level Visible Flag:**
- **All implementations** have `node.visible: boolean`
- When false, node and all children skip rendering

**Frustum Culling Toggle:**
- **caracal, shark**: `frustumCulled: boolean` to disable per-node
- **fennec, lynx**: Similar per-node override
- **mantis**: Global setting or per-node flag

### lookAt() Helper

**Z-Up Aware:**
- **caracal**: Handles degenerate case (forward ≈ Z)
- **fennec, hyena, lynx, shark, wren**: Z-up lookAt implementation
- **bonobo**: No lookAt in v1

### Transform Setter API

**Direct Property Mutation:**
- **caracal, shark, wren**: `node.position.set(x, y, z)` with Proxy or setters to auto-mark dirty

**Explicit Methods:**
- **fennec, mantis**: Production uses `setPosition(node, x, y, z)` to avoid Proxy overhead
- Debug mode may use Proxy for convenience

## Traversal Patterns

### Update Traversal (World Matrix Computation)
- **Depth-first recursive:** caracal, fennec, lynx, rabbit, shark, wren
- **Breadth-first iterative:** mantis, hyena
- **Flat iteration:** bonobo

### Render List Building
- **Recursive traverse then cull:** caracal, fennec, lynx, shark
- **Flat list maintained:** mantis, hyena (rebuild on add/remove)
- **Hybrid:** rabbit, wren (traverse to collect, then sort/cull)

### Node Lookup
- **Recursive find by name:** All except bonobo
- **Map-based lookup:** mantis (O(1) name lookup via Map)
- **Linear scan:** bonobo (no names in v1)

## Performance Characteristics Summary

| Implementation | Storage | Update Cost (1000 dirty nodes) | Hierarchy Change Cost | Memory Overhead |
|----------------|---------|-------------------------------|----------------------|-----------------|
| **bonobo** | SoA flat | Lowest (linear scan) | N/A (no hierarchy) | Lowest |
| **mantis** | SoA hierarchy | Low (breadth-first cache) | Medium (rebuild order) | Low |
| **hyena** | SoA hierarchy | Low (breadth-first) | Medium | Low |
| **caracal** | Object graph | Medium (recursive) | Low (just add/remove) | Medium |
| **fennec** | Object graph | Medium (recursive + early exit) | Low | Medium |
| **lynx** | Object graph | Medium (recursive) | Low | Medium |
| **rabbit** | Object graph | Medium | Low | Medium |
| **shark** | Object graph | Medium | Low | Medium |
| **wren** | Object graph | Medium | Low | Medium |

## Decision Matrix: Which Approach to Use?

### Choose Flat SoA (Bonobo-style) if:
- Maximum entity count (>10,000 independent objects)
- Minimal parent-child relationships
- Performance is critical over flexibility
- You're comfortable with manual instancing

### Choose SoA Hierarchy (Mantis/Hyena-style) if:
- Need hierarchy but want SoA cache benefits
- Moderate hierarchy changes (mostly static structure)
- Mobile or memory-constrained targets
- Predictable allocation patterns important

### Choose Object Graph (Caracal/Fennec/Lynx/Rabbit/Shark/Wren-style) if:
- Rapid prototyping and development speed
- Frequent hierarchy modifications
- Complex nested scenes (UI, skeletal rigs, etc.)
- Developer experience and debuggability matter

## Unique Implementation Details Worth Noting

### Caracal
- Contiguous matrix storage for GPU upload efficiency
- Explicit dirty flag propagation to avoid Proxy overhead
- Built-in frustum culling at graph level

### Fennec
- Dedicated `BoneAttachment` node type
- Production fast path (explicit setters) vs. debug Proxy mode
- Flat mesh list rebuilt on add/remove for fast iteration

### Lynx
- Most comprehensive documentation of the 9
- Clear separation of concerns (transform vs. culling vs. rendering)
- Explicit reference counting for geometry sharing

### Mantis
- Pre-computed breadth-first traversal order
- Doubling capacity growth strategy for SoA arrays
- O(1) name lookup via Map

### Hyena
- Structure-of-Arrays with full hierarchy support
- Dirty flag stops propagation if already dirty (optimization)

### Bonobo
- Deliberately flat for maximum performance
- Future v2 will add opt-in parent-child as additional typed arrays
- Targets 50,000 entities with automatic instancing

### Rabbit
- (Note: Limited documentation, likely follows standard object graph pattern)

### Shark
- Lazy world matrix computation (only when accessed)
- Flat matrix storage in a single pool with views per node
- Node ID = index into storage pool

### Wren
- Separate buffer storage for vertex attributes (not interleaved)
- Coordinate system conversion baked into GLTF loader
- Dual-output emissive for bloom integration
