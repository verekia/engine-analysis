# Standardized Documentation Structure

This document defines the standardized file structure that all engine implementation proposals should follow.

## Standard Files

Every implementation should contain the following markdown files:

1. **PLAN.md** - Executive summary and overall approach
2. **ARCHITECTURE.md** - High-level architecture and core design decisions
3. **RENDERER.md** - WebGL/WebGPU abstraction layer (HAL/backend)
4. **SCENE-GRAPH.md** - Scene graph, transform hierarchy, parent/child relationships, Groups
5. **MATERIALS.md** - Material system (Basic, Lambert, vertex colors, emissive per material index)
6. **GEOMETRY.md** - Geometry primitives (plane, box, sphere, cone, cylinder, capsule, circle)
7. **ANIMATION.md** - Skeletal animation, cross-fading, bone attachment for static meshes
8. **ASSETS.md** - Asset loading (GLTF, Draco decoding, KTX2, Basis texture decoding)
9. **LIGHTING-SHADOWS.md** - Lighting system (directional, ambient) and cascading shadow maps
10. **POST-PROCESSING.md** - Post-processing effects (Unreal Bloom, MSAA)
11. **TRANSPARENCY.md** - Transparency rendering approach (must work without headaches)
12. **RAYCASTING-BVH.md** - Raycasting and BVH spatial acceleration (three-mesh-bvh level)
13. **CONTROLS.md** - Orbit controls and camera interaction
14. **REACT.md** - React bindings similar to React Three Fiber
15. **HTML-OVERLAY.md** - HTML overlay system (CSS3DRenderer/Drei Html-like)
16. **PERFORMANCE.md** - Performance optimization strategies (2000 draw calls @ 60fps target)
17. **MATH.md** - Math utilities and coordinate system (Z-up, right-handed)
18. **API.md** - API design philosophy and usage examples

## Mapping from Current Files

### File Mergers

Some implementations split these topics into multiple files. These should be merged:

- `BLOOM.md` + `POST_PROCESSING.md` → **POST-PROCESSING.md**
- `LIGHTING.md` + `SHADOWS.md` → **LIGHTING-SHADOWS.md**
- `BVH-RAYCASTING.md` or `CULLING-RAYCASTING.md` → **RAYCASTING-BVH.md**
- `SPATIAL.md` (if about raycasting/BVH) → **RAYCASTING-BVH.md**
- `INTERACTION.md` (if about controls) → **CONTROLS.md**
- `GPU-ABSTRACTION.md` or `HAL.md` → **RENDERER.md**
- `RENDERING.md` (general) → **RENDERER.md**

### File Renamings

Some implementations use different naming conventions:

- `SCENE_GRAPH.md` → **SCENE-GRAPH.md** (use hyphens)
- `HTML_OVERLAY.md` → **HTML-OVERLAY.md** (use hyphens)
- `POST_PROCESSING.md` → **POST-PROCESSING.md** (use hyphens)
- `REACT-BINDINGS.md` → **REACT.md** (shorter)
- `API-REFERENCE.md` → **API.md** (shorter)
- `ASSET-LOADING.md` → **ASSETS.md** (shorter)
- Numbered files (e.g., `01-hal.md`, `02-renderer.md`) → Use standard names without numbers

## Purpose

This standardization allows for easy comparison of different approaches by topic:
- Want to compare rendering strategies? Look at all `RENDERER.md` files
- Want to compare animation systems? Look at all `ANIMATION.md` files
- Want to understand transparency approaches? Look at all `TRANSPARENCY.md` files

Each file should be self-contained enough to understand that aspect of the implementation independently.
