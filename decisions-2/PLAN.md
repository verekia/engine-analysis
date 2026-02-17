# PLAN.md - Decisions

## Decision: Full-Featured Monolithic Package

**Chosen approach**: Single cohesive package with full v1 feature set (Hyena/Shark/Wren style), not Bonobo's minimal/deferred approach.

### Rationale

The prompt is explicit: shadows, skeletal animation, glTF loading, React bindings, HTML overlays, and BVH raycasting are all v1 requirements, not stretch goals. Bonobo's strategy of deferring half the feature set to v2 doesn't match the specification.

A monolithic single-package distribution (used by 5/9 implementations) is preferred over a multi-package monorepo (4/9) because:

- Simpler distribution: one `npm install`, one version number, zero cross-package version drift
- The engine scope is deliberately minimal (two materials, one light type) - the tree-shaking benefit of multiple packages is marginal
- Multi-package setups add build complexity disproportionate to the codebase size
- React bindings ship as a separate subpath export (`engine/react`) since they have a peer dependency on React

### Architecture Philosophy

- **WebGPU-first abstraction** with WebGL2 fallback (universal agreement)
- **TypeScript throughout**, no semicolons, const arrow functions (as specified)
- **Zero per-frame allocations** in the render loop
- **Data layout over language**: SoA typed arrays for hot data, OOP wrappers for the public API
- **2000 draw calls at 60fps on mobile** as the hard performance target

### What's In Scope for v1

Everything in the original prompt:

- Dual backend (WebGPU + WebGL2)
- Basic + Lambert materials with material index palette
- glTF 2.0 with Draco + KTX2/Basis
- Vertex colors, per-vertex emissive, bloom
- MSAA
- SAH BVH raycasting
- Skeletal animation with crossfade and bone attachment
- Frustum culling
- WBOIT transparency
- Orbit controls
- All 7 parametric primitives
- Directional + ambient light
- 3-cascade CSM shadows
- Z-up right-handed coordinate system
- Parent/child scene graph with Groups
- HTML overlay
- React bindings (custom reconciler)

### What's Explicitly Out of Scope

- PBR materials (Lambert is sufficient for the target aesthetic)
- Point lights, spot lights
- Morph targets / blend shapes
- Physics
- Text rendering
- Instancing (the prompt says "no instancing" - performance comes from low per-draw overhead)

### Sources

Draws primarily from: Wren (sort-based architecture, monolithic), Mantis (ring buffer, radix sort, SoA), Fennec (bloom, MSAA integration, React), Lynx (comprehensive documentation, scene graph), with specific cherry-picks from other implementations noted per topic.
