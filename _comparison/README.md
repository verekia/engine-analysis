# Engine Implementation Comparison Analysis

This directory contains detailed comparisons of all 9 engine implementations across 18 key topics. Each comparison document identifies areas of **universal agreement** and **key variations** to help you cherry-pick the best approaches for your implementation.

## Overview

All 18 comparison documents follow the same structure:
- **Universal Agreement** - What ALL implementations agree on
- **Key Variations** - Major differences categorized by approach
- **Implementation Breakdown** - Detailed analysis with counts
- **Recommendations** - Guidance for selecting approaches

## High-Level Findings

### What EVERYONE Agrees On

Across all 9 implementations, there is **near-universal consensus** on:

1. **Performance Target**: 2000 draw calls @ 60fps on recent mobile devices
2. **Graphics APIs**: WebGPU-first with WebGL2 fallback
3. **Coordinate System**: Z-up, right-handed (8/9 implementations)
4. **Materials**: Two types - BasicMaterial (unlit) and LambertMaterial (Lambert diffuse)
5. **Transparency**: Weighted Blended OIT (Order-Independent Transparency)
6. **Shadows**: 3-cascade CSM (Cascaded Shadow Maps)
7. **Bloom**: Unreal-style progressive downsample/upsample
8. **BVH**: SAH-based (Surface Area Heuristic) with flat array layout
9. **Animation**: GPU skinning with up to 4 bone influences per vertex
10. **Assets**: glTF 2.0, Draco mesh compression, KTX2/Basis texture compression
11. **React Bindings**: Declarative JSX following React Three Fiber patterns
12. **Math Library**: Float32Array-backed, zero-allocation APIs
13. **Scene Graph**: Hierarchical parent-child transforms with dirty propagation
14. **Frustum Culling**: Standard 6-plane frustum test against bounding volumes

### Areas of Greatest Variation

The implementations **diverge most significantly** on:

#### 1. **Architecture Philosophy** (See: ARCHITECTURE.md)
- **Data Layout**: Flat SoA (3 engines) vs Object Graph (6 engines)
- **Module Structure**: Monolithic (2) vs Multi-package (7)
- **Scene Graph**: Flat entity list (1) vs Tree structure (8)

#### 2. **Performance Strategies** (See: PERFORMANCE.md)
- **Instancing**: Auto-instancing (1 engine: bonobo) vs Manual/none (8)
- **Sort Algorithm**: Radix sort (6) vs Native sort (3)
- **Render Bundles**: Always (1), Conditionally (3), Never (5)

#### 3. **React Integration** (See: REACT.md)
- **Reconciler**: Custom reconciler (8) vs Plain useEffect (1: bonobo)
- **Complexity**: 2000+ lines (8) vs 50 lines (1)

#### 4. **Material System** (See: MATERIALS.md)
- **Palette Size**: 16 entries (2), 32 entries (5), 64 entries (1), unlimited (1)
- **Emissive Output**: MRT (4) vs Alpha channel (4) vs Separate pass (1)

#### 5. **WebGL2 Fallback** (See: RENDERER.md)
- **State Caching**: Full caching (5) vs Minimal (3) vs None (1)
- **Uniform Strategy**: UBOs (8) vs Separate uniforms (1)

#### 6. **Shadow Maps** (See: LIGHTING-SHADOWS.md)
- **Storage**: Texture Atlas (5) vs Texture Array (4)
- **Resolution**: 1024px (5), 2048px (3), variable (1)
- **PCF Kernel**: 3×3 (5) vs 5×5 (2) vs 9×9 (1) vs adaptive (1)

#### 7. **Orbit Controls** (See: CONTROLS.md)
- **Angle Convention**: Polar (4) vs Elevation (5)
- **Damping**: Velocity-based (4) vs Delta-based (3) vs None (2)

#### 8. **API Design** (See: API.md)
- **Initialization**: Factory functions (5) vs Constructors (4)
- **Configuration**: Explicit methods (7) vs Constructor options (2)

## Comparison Documents

### Core Architecture
- **[PLAN.md](./PLAN.md)** (10K) - High-level design philosophy and feature scope
- **[ARCHITECTURE.md](./ARCHITECTURE.md)** (14K) - Module structure, data layout, frame lifecycle
- **[RENDERER.md](./RENDERER.md)** (17K) - WebGPU/WebGL abstraction, render pipeline, state management

### Scene & Content
- **[SCENE-GRAPH.md](./SCENE-GRAPH.md)** (10K) - Transform hierarchy, update strategies, storage
- **[MATERIALS.md](./MATERIALS.md)** (14K) - Material system, palettes, shader variants
- **[GEOMETRY.md](./GEOMETRY.md)** (17K) - Buffer layouts, primitives, bounding volumes

### Animation & Assets
- **[ANIMATION.md](./ANIMATION.md)** (11K) - Skeletal animation, mixing, bone matrices
- **[ASSETS.md](./ASSETS.md)** (13K) - glTF loading, compression, worker strategies

### Lighting & Effects
- **[LIGHTING-SHADOWS.md](./LIGHTING-SHADOWS.md)** (20K) - Lambert shading, CSM, PCF filtering
- **[POST-PROCESSING.md](./POST-PROCESSING.md)** (9K) - Bloom, MSAA, tone mapping
- **[TRANSPARENCY.md](./TRANSPARENCY.md)** (12K) - OIT techniques, weight functions

### Interaction & Queries
- **[RAYCASTING-BVH.md](./RAYCASTING-BVH.md)** (16K) - BVH construction, traversal, ray tests
- **[CONTROLS.md](./CONTROLS.md)** (7K) - Orbit camera, input handling, damping

### Developer Experience
- **[REACT.md](./REACT.md)** (10K) - React bindings, reconcilers, hooks
- **[HTML-OVERLAY.md](./HTML-OVERLAY.md)** (11K) - 3D-positioned DOM elements, occlusion
- **[API.md](./API.md)** (15K) - API design patterns, initialization, configuration

### Foundation
- **[PERFORMANCE.md](./PERFORMANCE.md)** (11K) - Optimization strategies, frame budgets
- **[MATH.md](./MATH.md)** (12K) - Math library design, coordinate systems

## How to Use This Analysis

### For Architects
Start with **PLAN.md** and **ARCHITECTURE.md** to understand the high-level design philosophies and choose between SoA vs Object Graph, monolithic vs modular, etc.

### For Rendering Engineers
Focus on **RENDERER.md**, **MATERIALS.md**, **LIGHTING-SHADOWS.md** to understand GPU abstraction, shader management, and shadow techniques.

### For Performance Engineers
Study **PERFORMANCE.md**, **RAYCASTING-BVH.md**, **SCENE-GRAPH.md** to understand optimization strategies and data structure choices.

### For DX (Developer Experience) Engineers
Review **REACT.md**, **API.md**, **HTML-OVERLAY.md** to understand different approaches to developer-facing APIs.

### For Feature Implementers
Each topic document has a **"Recommendations for Cherry-Picking"** section that guides you toward the best approach for your specific needs (simplicity, performance, features, etc.).

## Key Insights

### Bonobo's Unique Position
Bonobo stands out as the **most minimal** implementation:
- Only 72K total (vs 124K-526K for others)
- No custom React reconciler (50 lines vs 2000+)
- Flat SoA architecture with auto-instancing
- Many features marked as "v2" or out of scope

### Lynx's Comprehensiveness
Lynx is the **most detailed** implementation:
- 526K total documentation (7.3× larger than bonobo)
- Most comprehensive feature set
- Extensive code examples and implementation details

### Common Ground
Despite variations, **80%+ of decisions** are shared across all implementations:
- Core rendering techniques (OIT, CSM, Bloom)
- Asset formats (glTF, Draco, KTX2)
- Performance targets (2000 draws @ 60fps)
- React-based developer experience

### Innovation Highlights
Unique approaches worth considering:
- **Bonobo**: Auto-instancing, flat SoA, minimal React integration
- **Caracal**: Stackless BVH traversal with skip pointers
- **Fennec**: Dual Kawase blur for bloom
- **Hyena**: Depth-aware PCF with 9×9 kernel
- **Lynx**: Largest material palette (64 entries)
- **Rabbit**: mat4x3 bone matrices (25% space savings)
- **Wren**: Fastest bloom (0.22ms at 1080p)

## Next Steps

1. Read **PLAN.md** to understand the different architectural philosophies
2. For each feature you're implementing, read the corresponding comparison document
3. Use the **"Recommendations for Cherry-Picking"** sections to guide your decisions
4. Refer back to the original implementation files (in the engine directories) for code examples

Each comparison is designed to help you make informed decisions without bias - just facts, trade-offs, and categorized approaches.
