# PLAN - Design Decisions

## Executive Summary

A high-performance, minimal 3D game engine with Three.js-like developer experience, targeting 2000 draw calls at 60fps on recent mobile devices. Full-featured from v1.

## Core Philosophy

**Decision: Full-featured v1 with production-ready scope**

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren (8/9 implementations)
- Rejected: Bonobo's minimal v1 approach (defers too many core features — shadows, animation, glTF, BVH — to v2)
- Rationale: The requirements explicitly list shadows, skeletal animation, glTF, BVH raycasting, and React bindings. Deferring these means v1 cannot be used for real games

## Architecture Philosophy

**Decision: Monolithic single package with internal modules**

- Sources: Hyena, Rabbit, Shark, Wren, Bonobo (5/9)
- Rejected: Multi-package monorepo (Fennec, Caracal, Lynx, Mantis)
- Rationale: Simpler distribution, single version number, easier to understand. Multi-package adds build complexity that isn't justified for a focused game engine. Internal modules still provide separation of concerns. React bindings can be a separate entry point

## Performance Target

**Decision: 2000 draw calls at 60fps on recent phones (16.6ms budget)**

- Sources: All 9 implementations agree
- Strategy: Sort-based rendering as primary optimization, no automatic instancing
- Per-draw JS overhead target: <2-3μs

## Instancing Strategy

**Decision: No automatic instancing**

- Sources: 8/9 implementations agree (all except Bonobo)
- Rationale: The prompt explicitly states "no instancing" — the engine should be so performant at individual draw calls that instancing isn't needed for most cases. Per-draw overhead optimized to <2-3μs makes 2000 unique draws feasible within budget

## Dual Backend Support

**Decision: WebGPU-first with WebGL2 fallback**

- Sources: All 9 implementations agree
- WebGPU is the primary target; GPU abstraction modeled after WebGPU concepts
- WebGL2 is the translation layer, not the design target
- Auto-detect WebGPU with graceful fallback

## Coordinate System

**Decision: Z-up, right-handed**

- Sources: All 9 implementations agree
- X = right, Y = forward, Z = up
- Matches Blender, Unreal Engine
- glTF Y-up assets converted at import time

## Code Style

**Decision: TypeScript, no semicolons, const arrow functions**

- Sources: Universal agreement from requirements
- Strict mode TypeScript throughout
- `const` arrow functions preferred over `function` declarations
- Code should be minimal, well-documented, and understandable

## Feature Scope (v1)

All of the following are in-scope for v1:

- WebGL2 and WebGPU backends
- Basic and Lambert materials
- Material index system with per-vertex color/emissive
- Vertex colors
- glTF 2.0 with Draco mesh compression
- KTX2/Basis Universal texture compression
- Color textures, AO textures
- Per-vertex bloom (Unreal-style)
- 4x MSAA
- SAH-based BVH raycasting (three-mesh-bvh quality)
- GPU skinned meshes with skeletal animation and crossfade
- Frustum culling
- Weighted Blended OIT transparency
- Orbit controls
- Parametric geometry primitives (plane, box, sphere, cone, cylinder, capsule, circle)
- Directional light + ambient light
- 3-cascade CSM for 200×200m low-poly worlds
- Parent/child scene graph with Groups
- Bone attachment for static meshes
- HTML overlay system
- React bindings (R3F-style)
- ACES filmic tone mapping

## What's Out of Scope for v1

- Point lights, spot lights
- PBR materials (Lambert is sufficient for stylized/low-poly)
- Morph targets / blend shapes
- Physics integration
- Text rendering
- Particle systems
- Post-processing beyond bloom + tone mapping
- PCSS / VSM / ESM shadow techniques
- Multiple shadow-casting lights
