# Architecture

## Overview

Wren is a lightweight 3D game engine for the web, targeting both WebGL2 and WebGPU. It prioritizes raw draw-call throughput, minimal JavaScript overhead, and a developer experience inspired by three.js — but built from scratch around a sort-based renderer designed to sustain 2000+ draw calls at 60 fps on mobile.

## Design Principles

1. **Performance is architecture, not optimization.** Every system — from the HAL to the scene graph — is designed around minimizing per-frame allocations, state changes, and JS-to-GPU API calls.
2. **WebGPU-first abstraction, WebGL2 always works.** The HAL models WebGPU concepts (pipelines, bind groups, command encoders). The WebGL2 backend emulates them. Advanced WebGPU features (render bundles, compute-based culling) are used when available but never required.
3. **Transparency that works.** Weighted Blended OIT eliminates sort-order bugs. No manual renderOrder hacks.
4. **Z-up, right-handed.** Consistent with Blender, Unreal, and most game-art pipelines.
5. **Minimal API surface.** Fewer concepts to learn, fewer things that can break.

## Coordinate System

- **Z is up**, Y is forward, X is right
- Right-handed coordinate system
- Consistent with Blender default orientation

## Key Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Rendering strategy** | Sort-based renderer with 64-bit encoded render keys | Minimizes state changes. Radix sort is O(n). Enables opaque front-to-back and transparent back-to-front in a single sorted pass. |
| **HAL design** | WebGPU-modeled abstraction | WebGPU's explicit pipeline/bind-group model maps cleanly to WebGL2 state. Avoids lowest-common-denominator limitations. |
| **Shader authoring** | Dual GLSL 300 es / WGSL sources with shared logic via `#include`-style preprocessing | Avoids runtime transpilation cost. Shader variants are small (Basic, Lambert, Skinned, Shadow, Bloom). |
| **Transparency** | Weighted Blended OIT (McGuire-Bavoil 2013) | Single-pass, no sorting, works on both backends. Good enough for game-style transparency. |
| **Shadows** | 3-cascade CSM, 1024px per cascade, PCF 5x5 | Optimal for 200m worlds. Texel-snapped projections prevent shimmer. |
| **Bloom** | Per-vertex emissive + Unreal-style progressive downsample/upsample | 5-level mip chain, ~10 fullscreen passes at decreasing resolution. Very cheap. |
| **BVH** | SAH-built flat-array BVH in typed arrays | Cache-friendly, zero-GC traversal, comparable to three-mesh-bvh. |
| **Animation** | Linear blend skinning on GPU, CPU pose interpolation | Simple, fast, sufficient for game characters. Crossfade via dual-pose blending. |
| **Scene graph** | Dirty-flag transform propagation, flat world-matrix cache | Only recomputes matrices when transforms change. O(1) world matrix lookup. |
| **React bindings** | Custom `react-reconciler` host config | Same approach as R3F. Maps JSX to scene graph mutations. Decoupled from render loop. |
| **Math** | Custom z-up right-handed math (Float32Array-backed) | No allocation per operation. All ops mutate destination. Z-up baked in, not converted. |
| **WebGPU advantages** | Render bundles for static geometry, compute-based frustum culling | Optional enhancements. Engine works fully without them. |

## Performance Budget

| Metric | Target |
|--------|--------|
| Draw calls at 60fps (mobile) | 2000+ |
| JS overhead per draw call | < 5 microseconds |
| Scene graph traversal (1000 nodes) | < 0.5ms |
| BVH raycast (100k triangles) | < 0.3ms |
| Bloom post-process | < 1ms at 1080p |
| Total frame budget | < 16.6ms |

## System Components

- **HAL (Hardware Abstraction Layer)** - Unified WebGL2/WebGPU device API
- **Sort-Based Renderer** - Render keys, sorting, state caching, draw submission
- **Scene Graph** - Nodes, groups, transforms, bone attachment
- **Materials** - Basic, Lambert, vertex colors, material index system, emissive
- **Geometry** - Parametric primitives, vertex layout, GLTF, Draco
- **Textures** - Color, AO, KTX2/Basis Universal
- **Lighting & Shadows** - Directional, ambient, 3-cascade CSM
- **Skeletal Animation** - Clips, mixer, crossfade, bone attachment
- **Bloom** - Per-vertex emissive, Unreal-style post-process bloom
- **Transparency** - Weighted Blended OIT
- **Raycasting** - SAH BVH, flat-array traversal
- **Culling & MSAA** - Frustum culling, MSAA, performance strategies
- **Controls** - Orbit controls
- **HTML Overlay** - DOM overlay system
- **React Bindings** - Reconciler, hooks, JSX catalog

## Dependencies

| Dependency | Purpose | Size |
|------------|---------|------|
| `basis_universal` (WASM) | KTX2/Basis texture transcoding | ~200KB |
| `draco3d` (WASM) | Draco mesh decompression | ~300KB |
| `react-reconciler` | React bindings (peer dep) | ~15KB |
| **Total core (no React, no WASM)** | | **< 50KB gzipped** |

All WASM decoders are loaded lazily on first use.
