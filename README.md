# 3D Engine Implementation Proposals - Analysis Repository

This repository contains 9 different implementation proposals for a minimal high-performance 3D engine, each designed by different AI agents.

## Structure

Each implementation is organized in a standardized format with 18 markdown files for easy cross-comparison:

### Implementation Directories
- **bonobo** - Flat entity system with automatic instancing
- **caracal** - [View PLAN.md for approach]
- **fennec** - [View PLAN.md for approach]
- **hyena** - [View PLAN.md for approach]
- **lynx** - [View PLAN.md for approach]
- **mantis** - [View PLAN.md for approach]
- **rabbit** - [View PLAN.md for approach]
- **shark** - [View PLAN.md for approach]
- **wren** - [View PLAN.md for approach]

### Standardized File Structure

Every implementation contains these 18 files:

1. **PLAN.md** - Executive summary and overall approach
2. **ARCHITECTURE.md** - High-level architecture and core design decisions
3. **RENDERER.md** - WebGL/WebGPU abstraction layer (HAL/backend)
4. **SCENE-GRAPH.md** - Scene graph, transform hierarchy, parent/child relationships
5. **MATERIALS.md** - Material system (Basic, Lambert, vertex colors, emissive)
6. **GEOMETRY.md** - Geometry primitives (plane, box, sphere, cone, cylinder, capsule, circle)
7. **ANIMATION.md** - Skeletal animation, cross-fading, bone attachment
8. **ASSETS.md** - Asset loading (GLTF, Draco, KTX2, Basis texture decoding)
9. **LIGHTING-SHADOWS.md** - Lighting (directional, ambient) and cascading shadow maps
10. **POST-PROCESSING.md** - Post-processing effects (Unreal Bloom, MSAA)
11. **TRANSPARENCY.md** - Transparency rendering approach
12. **RAYCASTING-BVH.md** - Raycasting and BVH spatial acceleration
13. **CONTROLS.md** - Orbit controls and camera interaction
14. **REACT.md** - React bindings (React Three Fiber-like)
15. **HTML-OVERLAY.md** - HTML overlay system (CSS3DRenderer/Drei Html-like)
16. **PERFORMANCE.md** - Performance optimization strategies (2000 draw calls @ 60fps target)
17. **MATH.md** - Math utilities and coordinate system (Z-up, right-handed)
18. **API.md** - API design philosophy and usage examples

## Comparing Implementations

To compare different approaches on a specific topic, simply open the same filename across different directories. For example:

- **Compare rendering strategies**: Look at all `RENDERER.md` files
- **Compare animation systems**: Look at all `ANIMATION.md` files
- **Compare transparency approaches**: Look at all `TRANSPARENCY.md` files
- **Compare React bindings**: Look at all `REACT.md` files

## Engine Requirements

All implementations target these specifications:
- WebGL and WebGPU support with fallbacks
- Basic and Lambert materials
- GLTF with Draco decoding
- Color textures, AO textures, KTX2 with Basis decoding
- Per-vertex bloom, vertex colors, material-index-based properties
- Efficient MSAA
- Raycasting and BVH (three-mesh-bvh quality)
- Skinned meshes with skeletal animations and cross-fading
- Frustum culling
- Proper transparency handling
- High performance: 2000 draw calls @ 60fps on recent phones
- Orbit controls
- Parametrized geometry primitives
- Directional and ambient lighting
- Cascading shadow maps for 200x200m low-poly worlds
- Z-up, right-handed coordinate system
- Parent/child scene graph with Groups
- Bone attachment for static meshes (e.g., sword to hand)
- HTML overlay feature
- React bindings similar to React Three Fiber
- TypeScript, no semicolons, prefer const arrow functions

## Documentation

See `STANDARDIZATION.md` for details on the standardized file structure and how the reorganization was performed.
