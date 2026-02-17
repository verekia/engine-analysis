# API Design & Usage

## Overview

The Lynx API design philosophy was not explicitly documented in the original proposal. This placeholder exists to maintain structural consistency with the standardized documentation format.

## Key Principles

Based on the architecture and implementation details found throughout other documentation files, the API likely follows these principles:

- **Z-up right-handed coordinate system** for consistency with many 3D modeling tools
- **Immutable data structures** where appropriate (e.g., render pipelines, geometry)
- **Backend-agnostic** through the GPU Abstraction Layer (GAL)
- **TypeScript-first** with full type safety
- **Explicit resource management** (manual cleanup via destroy() methods)
- **React bindings** similar to React Three Fiber (see REACT.md)

## Usage Pattern

```typescript
// Initialize renderer with canvas
const renderer = createRenderer(canvas, {
  preferredBackend: 'webgpu', // or 'webgl2'
  msaa: true,
  bloom: true,
  // ... other config
})

// Create scene
const scene = createScene()

// Add objects
const mesh = createMesh(geometry, material)
scene.add(mesh)

// Create camera
const camera = createPerspectiveCamera({
  fov: 60,
  near: 0.1,
  far: 1000,
})

// Render loop
function animate() {
  renderer.render(scene, camera)
  requestAnimationFrame(animate)
}
animate()
```

## See Also

- **ARCHITECTURE.md** - System architecture and component relationships
- **RENDERER.md** - GPU Abstraction Layer (GAL) interfaces
- **REACT.md** - React bindings and declarative API
- **MATERIALS.md** - Material system API
- **GEOMETRY.md** - Geometry primitives API
- **SCENE-GRAPH.md** - Scene graph manipulation API
