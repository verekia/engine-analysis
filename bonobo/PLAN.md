# Bonobo — Lightweight High-Performance WebGPU Engine with R3F-like DX

## Executive Summary

Bonobo is a new engine designed around the 95% of performance wins (instancing, sorting, culling, zero GC) without the complexity of WASM/C. TypeScript only, easy to understand, modify, and maintain.

## Design Principles

1. **Automatic performance** — instancing, sorting, culling happen without user opt-in
2. **TypeScript only** — no C, no WASM, no build toolchain beyond bundling
3. **Thin React layer** — no custom reconciler, no fake Three.js objects
4. **Flat by default** — no deep scene graph; parent-child transforms are opt-in
5. **WebGPU first** — WebGL2 as fallback, not the primary target
6. **Zero per-frame allocations** — pre-allocated typed arrays and scratch math

## Core Performance Strategy

The core thesis: **data layout and batching strategy matter more than language choice**. Flat typed arrays in TypeScript, automatic instancing, draw call sorting, and frustum culling get you 90%+ of the performance of a WASM engine — with full debuggability, one language, and a fraction of the complexity.

### Performance Wins (ranked by impact)

1. **Automatic Instancing** — N entities with same geometry + material = 1 draw call
2. **Draw Call Sorting** — Minimize GPU state changes (pipeline → material → geometry)
3. **Frustum Culling** — Cull 30-70% of entities in typical scenes
4. **WebGPU First** — Dramatically lower CPU overhead per draw call
5. **Flat Typed Array Storage** — Zero GC, cache-friendly, GPU-uploadable
6. **Dirty Flags** — Only recompute matrices for entities that changed
7. **Zero Per-Frame Allocations** — Pre-allocated scratch variables for all math

## What's Explicitly Out of Scope (for v1)

- **Scene graph hierarchy** — flat entity list only. Parent-child can be added later as an opt-in layer
- **Physics** — use Rapier or cannon-es alongside
- **Shadows** — significant complexity, add in v2
- **glTF loader** — import meshes as raw vertex/index arrays. A glTF adapter can be a separate package
- **Skeletal animation** — add in v2 once the core is solid
- **Post-processing** — except bloom (single MRT pass), keep it minimal
- **Text rendering** — use HTML/CSS overlays or SDF text as a separate package

## Architecture Overview

See ARCHITECTURE.md for detailed architecture design.

## File Structure

```
src/
  engine/
    world.ts            World class: entity storage, render loop, spawn/despawn
    renderer.ts         Renderer interface + WebGPU implementation
    webgl-renderer.ts   WebGL2 fallback
    geometry.ts         Primitive generators (box, sphere, custom)
    material.ts         Material types and registry
    camera.ts           Camera state + matrix computation
    math.ts             mat4/vec3 operations on typed arrays
    culling.ts          Frustum plane extraction + sphere test
    sorting.ts          Sort key packing + draw command merging
    shaders.ts          WGSL shaders (static, textured pipelines)
    webgl-shaders.ts    GLSL 300 es shaders
    types.ts            Shared types and constants
    index.ts            Barrel exports

  react/
    context.ts          WorldContext + useWorld hook
    canvas.tsx          <Canvas> component
    mesh.tsx            <Mesh> component
    camera.tsx          <Camera> component
    light.tsx           <Light> component
    index.ts            Barrel exports

  index.ts              Package entry point
```
