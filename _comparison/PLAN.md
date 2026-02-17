# PLAN.md - Cross-Engine Comparison

This document compares the high-level plans and design philosophies across all 9 engine implementations.

---

## Universal Agreement

All 9 implementations share these core goals and principles:

### Performance Target
- **2000+ draw calls at 60 fps on mobile devices** is the explicit performance target
- Raw draw call throughput prioritized over other considerations

### Dual Backend Support
- **WebGPU as primary target**, WebGL2 as mandatory fallback
- WebGPU recognized as the future; WebGL2 for compatibility
- No implementation treats WebGL2 as the primary target

### Coordinate System
- **Z-up, right-handed coordinate system** (X=right, Y=forward, Z=up)
- Universally adopted across all implementations
- GLTF Y-up assets converted to Z-up at import time

### React Bindings
- All engines include **React bindings** inspired by React Three Fiber
- Custom `react-reconciler` based implementation
- Declarative scene graph construction via JSX

### Material System
- **Two core materials**: Basic (unlit) and Lambert (diffuse)
- Material index system for per-vertex color variation
- Vertex colors supported

### Transparency
- **Weighted Blended Order-Independent Transparency (WBOIT)** universally adopted
- Single-pass, no sorting required
- Eliminates traditional transparent sorting artifacts

### Shadows
- **3-cascade Cascaded Shadow Maps (CSM)**
- PCF filtering for smooth shadows
- Designed for ~200m worlds

### Animation
- **GPU skinning** for skeletal animation
- Crossfade blending between clips
- Bone attachment support

### Asset Loading
- **glTF 2.0** as primary format
- **Draco** mesh compression via WASM worker
- **KTX2/Basis Universal** texture transcoding via WASM worker
- Y-up to Z-up conversion at load time

### Spatial Queries
- **SAH (Surface Area Heuristic) BVH** for raycasting
- Flat array layout for cache efficiency
- Triangle-level intersection

### Code Style
- TypeScript throughout
- No semicolons
- `const` arrow functions preferred over `function` declarations
- Minimal abstractions

---

## Key Variations

### 1. Architecture Philosophy

**Three Distinct Approaches:**

#### Flat Typed Array (SoA) Approach
- **Implementations**: Bonobo
- **Strategy**: Store all entity data in pre-allocated flat `Float32Array` buffers
- **Benefits**: Zero GC, cache-friendly, minimal abstraction
- **Trade-offs**: Less flexible for complex scene hierarchies

#### Modular Package Structure
- **Implementations**: Fennec, Caracal, Lynx, Mantis
- **Strategy**: Multiple npm packages (core, materials, loaders, animation, etc.)
- **Benefits**: Tree-shakeable, clear separation of concerns
- **Trade-offs**: More complex dependency management

#### Monolithic Structure
- **Implementations**: Hyena, Rabbit, Shark, Wren
- **Strategy**: Single cohesive package with internal modules
- **Benefits**: Simpler distribution, easier to understand
- **Trade-offs**: Less granular tree-shaking

### 2. Scene Graph Handling

**Scene Graph vs Entity System:**

#### Full Scene Graph (9 implementations)
- **Who**: All except Bonobo default behavior
- **Details**: Parent-child hierarchy, groups, dirty flag propagation
- **Use case**: Complex scenes with logical hierarchies

#### Flat Entity List with Optional Hierarchy
- **Who**: Bonobo
- **Details**: Flat list by default, parent-child as opt-in feature
- **Rationale**: Simpler mental model, easier automatic instancing
- **Trade-off**: More work for complex hierarchical scenes

### 3. Performance Strategy Priority

**What gets optimized first:**

#### Automatic Instancing Priority
- **Implementations**: Bonobo (primary focus)
- **Strategy**: Automatic merging of N entities with same geometry+material
- **Benefit**: Dramatic draw call reduction without user intervention

#### Draw Call Sorting Priority
- **Implementations**: All 9 implementations
- **Strategy**: Sort by pipeline→material→geometry to minimize state changes
- **Benefit**: Efficient even without instancing

#### Render Bundles (WebGPU)
- **Implementations**: All support when available
- **Strategy**: Pre-record draw commands for static geometry
- **Benefit**: Near-zero JS overhead for static scenes

### 4. React Integration Approach

**Two strategies:**

#### Thin React Layer (No Custom Reconciler)
- **Who**: Bonobo explicitly
- **Details**: Components call engine methods via `useEffect`
- **Benefits**: Simpler, less magic, easier to debug
- **Trade-offs**: Less declarative, more manual lifecycle management

#### Custom Reconciler (R3F-style)
- **Who**: 8 implementations (Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren)
- **Details**: Full `react-reconciler` integration
- **Benefits**: Fully declarative, automatic lifecycle management
- **Trade-offs**: More complexity, harder to debug

### 5. Scope and Features

**What's included vs. out of scope:**

#### Minimal Feature Set (v1)
- **Who**: Bonobo (explicitly minimal)
- **Excludes**: Scene graph hierarchy (optional), physics, shadows (v2), glTF loader (separate), skeletal animation (v2), post-processing beyond bloom, text rendering
- **Philosophy**: Ship only absolute essentials

#### Full Feature Set
- **Who**: 8 implementations
- **Includes**: Full scene graph, shadows, glTF loading, skeletal animation, multiple post-processing effects
- **Philosophy**: Production-ready from v1

### 6. TypeScript Only vs. Multi-Language

**All Engines: TypeScript Only**
- Universal agreement: No C, no WASM for core engine (only for asset decoding)
- Rationale: One language, full debuggability, easier maintenance
- Core thesis: Data layout matters more than language choice

---

## Implementation Breakdown

### By Primary Performance Strategy

| Strategy | Count | Implementations |
|----------|-------|----------------|
| Sort-based rendering | 9 | All |
| Automatic instancing | 1 | Bonobo (unique emphasis) |
| Render bundles | 9 | All (when WebGPU available) |
| Frustum culling | 9 | All |
| Dirty flag transforms | 9 | All |

### By Architecture Pattern

| Pattern | Count | Implementations |
|---------|-------|----------------|
| Modular packages | 4 | Fennec, Caracal, Lynx, Mantis |
| Monolithic package | 4 | Hyena, Rabbit, Shark, Wren |
| Flat SoA engine | 1 | Bonobo |

### By React Integration Style

| Style | Count | Implementations |
|-------|-------|----------------|
| Custom reconciler | 8 | Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren |
| Thin wrapper | 1 | Bonobo |

### By Feature Completeness (v1)

| Scope | Count | Implementations |
|-------|-------|----------------|
| Full-featured | 8 | Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren |
| Minimal core | 1 | Bonobo |

---

## Naming Conventions

All implementations use **animal names**, following a fun and memorable pattern:

- **Bonobo**: Great ape (intelligent, social)
- **Caracal**: Medium cat (agile, fast)
- **Fennec**: Desert fox (small, efficient)
- **Hyena**: Scavenger (efficient, performant)
- **Lynx**: Wild cat (sharp, precise)
- **Mantis**: Insect (predator, fast reactions)
- **Rabbit**: Small mammal (fast, agile)
- **Shark**: Apex predator (efficient hunter)
- **Wren**: Small bird (lightweight, agile)

---

## Core Thesis - Universal Agreement

**"Data layout and batching strategy matter more than language choice."**

Every implementation agrees:
- Flat typed arrays beat object-oriented abstractions
- Automatic draw call sorting beats manual optimization
- Zero per-frame allocations beat garbage collection
- WebGPU's lower overhead beats WebGL2's driver validation

The core performance wins come from:
1. Structure-of-Arrays (SoA) data layout
2. Pre-allocated typed array pools
3. Draw call sorting by composite keys
4. Frustum culling before rendering
5. Zero allocations in the render loop

These architectural decisions matter infinitely more than switching from TypeScript to C/WASM.

---

## What Sets Each Implementation Apart

### Bonobo
- **Unique**: Flat entity system by default, optional scene graph
- **Focus**: Automatic instancing as primary optimization
- **Philosophy**: 95% of performance wins without WASM complexity

### Caracal
- **Unique**: Explicit about backend abstraction at resource level
- **Focus**: Lowest-overhead GPU abstraction
- **Philosophy**: WebGPU-modeled interface for both backends

### Fennec
- **Unique**: Modular package structure with clear dependencies
- **Focus**: Clean separation of concerns
- **Philosophy**: Import only what you need

### Hyena
- **Unique**: Emphasizes render bundles and command buffer reuse
- **Focus**: Static scene optimization
- **Philosophy**: Pre-record everything possible

### Lynx
- **Unique**: Most detailed architecture documentation
- **Focus**: Zero-overhead abstraction, explicit over magic
- **Philosophy**: Static by default, dynamic by opt-in

### Mantis
- **Unique**: Ring buffer for per-object uniforms
- **Focus**: Command buffer pattern with radix sort
- **Philosophy**: Eliminate GC at structural level

### Rabbit
- **Unique**: Hardware Abstraction Layer (HAL) terminology
- **Focus**: WebGPU-modeled descriptors
- **Philosophy**: Mobile-first performance

### Shark
- **Unique**: Flat typed arrays for all core data
- **Focus**: Sort-once, draw-many pattern
- **Philosophy**: Zero-allocation render loop

### Wren
- **Unique**: Sort-based renderer with 64-bit encoded keys
- **Focus**: O(n) radix sort for optimal ordering
- **Philosophy**: Performance is architecture, not optimization

---

## Recommendations for Cherry-Picking

### If You Want Maximum Performance
- **Draw call sorting**: Universal across all (implement this first)
- **Render bundles**: Use when WebGPU available (8 engines)
- **Automatic instancing**: Bonobo's approach for same-material objects
- **Ring buffer**: Mantis's per-object uniform strategy

### If You Want Simplicity
- **Thin React layer**: Bonobo's useEffect-based approach
- **Monolithic package**: Easier to understand than many packages
- **Minimal features**: Start with Bonobo's scope, add features later

### If You Want Modularity
- **Package structure**: Fennec, Caracal, Lynx, Mantis approach
- **Tree-shakeable**: Pay only for what you import
- **Clear dependencies**: One-way dependency flow

### If You Want Mobile Performance
- **MSAA resolution**: All use 4x MSAA by default
- **Ring buffer**: Minimize per-object allocations
- **BVH culling**: SAH-based for efficient raycasting

### If You Want Developer Experience
- **Custom reconciler**: Full R3F-style declarative API (8 engines)
- **Automatic disposal**: Resources cleaned up on unmount
- **TypeScript**: Full type safety throughout
