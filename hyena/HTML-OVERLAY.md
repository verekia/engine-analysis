# HTML Overlay

## Overview

The HTML overlay system renders DOM elements at 3D world positions on top of the WebGL/WebGPU canvas. This is useful for health bars, name tags, tooltips, UI markers, and any HTML content that should track a 3D object.

Similar to three.js CSS3DRenderer or Drei's `<Html>` component, but simpler and more performant.

## Architecture

```
┌─────────────────────────────────────────────┐
│  Container (position: relative)              │
│  ┌───────────────────────────────────────┐  │
│  │  <canvas> (WebGL/WebGPU)              │  │
│  │  z-index: 0                            │  │
│  └───────────────────────────────────────┘  │
│  ┌───────────────────────────────────────┐  │
│  │  Overlay <div> (position: absolute)   │  │
│  │  pointer-events: none                  │  │
│  │  z-index: 1                            │  │
│  │  top: 0, left: 0, width: 100%         │  │
│  │                                        │  │
│  │  ┌──────────┐  ┌──────────┐           │  │
│  │  │ Label A   │  │ HP Bar   │           │  │
│  │  │ (CSS      │  │ (CSS     │           │  │
│  │  │ transform)│  │ transform│           │  │
│  │  └──────────┘  └──────────┘           │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

The overlay div sits on top of the canvas with `pointer-events: none`. Individual overlay elements can opt into pointer events with `pointer-events: auto`.

## World-to-Screen Projection

Each frame, overlay elements' 3D positions are projected to screen coordinates:

```typescript
const projectToScreen = (
  worldPosition: Vec3,
  viewProjectionMatrix: Mat4,
  canvasWidth: number,
  canvasHeight: number
): { x: number, y: number, z: number, visible: boolean } => {
  // Multiply world position by view-projection matrix
  const clip = vec4.transformMat4(pool.vec4(), [worldPosition[0], worldPosition[1], worldPosition[2], 1], viewProjectionMatrix)

  // Perspective divide
  const ndc = {
    x: clip[0] / clip[3],
    y: clip[1] / clip[3],
    z: clip[2] / clip[3],
  }

  // Check if behind camera
  if (clip[3] <= 0 || ndc.z < 0 || ndc.z > 1) {
    return { x: 0, y: 0, z: ndc.z, visible: false }
  }

  // NDC [-1, 1] → screen pixels
  const screenX = (ndc.x * 0.5 + 0.5) * canvasWidth
  const screenY = (1 - (ndc.y * 0.5 + 0.5)) * canvasHeight  // flip Y

  return { x: screenX, y: screenY, z: ndc.z, visible: true }
}
```

## CSS Transform Application

Overlay elements are positioned using `transform: translate()` rather than `top`/`left` to leverage GPU-accelerated compositing:

```typescript
const updateOverlayElement = (
  element: HTMLElement,
  screen: { x: number, y: number, z: number, visible: boolean },
  options: OverlayOptions
) => {
  if (!screen.visible) {
    element.style.display = 'none'
    return
  }

  element.style.display = ''
  element.style.transform = `translate(${screen.x}px, ${screen.y}px) translate(-50%, -50%)`

  // Optional: scale by distance
  if (options.scaleWithDistance) {
    const scale = 1.0 / screen.z
    element.style.transform += ` scale(${scale})`
  }

  // Optional: z-ordering by depth
  if (options.zIndex) {
    element.style.zIndex = String(Math.floor((1 - screen.z) * 10000))
  }
}
```

The `translate(-50%, -50%)` centers the element on the projected point.

## Occlusion Testing

Optional depth-based occlusion hides overlay elements that are behind scene geometry:

### Strategy: Depth Buffer Readback

```typescript
const testOcclusion = (
  screen: { x: number, y: number, z: number },
  depthBuffer: Float32Array,  // readback from GPU
  canvasWidth: number
): boolean => {
  const px = Math.floor(screen.x)
  const py = Math.floor(screen.y)
  const depthAtPixel = depthBuffer[py * canvasWidth + px]

  // If the overlay's depth is greater than the scene depth, it's occluded
  return screen.z > depthAtPixel + 0.001  // small bias
}
```

**Performance note**: Depth readback (`readPixels`/`readBuffer`) is expensive because it stalls the GPU-CPU pipeline. Mitigation strategies:

1. **Read at reduced resolution**: Read a 1/4 or 1/8 resolution depth buffer.
2. **Async readback**: On WebGPU, use `mapAsync` on a staging buffer to avoid stalling. On WebGL2, use `gl.readPixels` into a PBO with `gl.fenceSync` and read back one frame later.
3. **Skip frames**: Only test occlusion every 2–3 frames. Overlay visibility changes are rarely noticeable at 30Hz.
4. **Disable by default**: Occlusion is opt-in per overlay element.

### Strategy: Approximate Raycast

For fewer overlay elements, cast a ray from the camera toward the overlay's world position and check if it hits any scene geometry closer than the overlay:

```typescript
const ray = camera.worldPositionToRay(overlay.worldPosition)
const hits = scene.raycast(ray, { maxDistance: overlay.distanceToCamera, firstHitOnly: true })
const occluded = hits.length > 0 && hits[0].distance < overlay.distanceToCamera - 0.1
```

This avoids depth readback entirely but costs a raycast per overlay element. Good for <20 overlays.

## API

### Imperative API

```typescript
const overlay = new HtmlOverlay(engine)

const label = overlay.create({
  worldPosition: [10, 20, 5],          // 3D world position
  element: document.createElement('div'), // DOM element to position
  occlude: false,                         // default: no occlusion test
  scaleWithDistance: false,               // default: fixed screen size
  center: true,                           // center element on projected point
})

// Update position (e.g., following a mesh)
engine.onFrame(() => {
  label.worldPosition = mesh.getWorldPosition(pool.vec3())
})

// Remove
overlay.remove(label)
```

### React API

```tsx
import { Html } from '@hyena/react'

// Basic usage — positioned at parent's world position
const NameTag = () => (
  <group position={[10, 20, 5]}>
    <Html>
      <div className="nametag">Player 1</div>
    </Html>
  </group>
)

// With offset and options
const HealthBar = () => (
  <mesh position={[0, 0, 0]}>
    <boxGeometry />
    <lambertMaterial />
    <Html
      position={[0, 0, 2.5]}       // offset above the mesh
      occlude                        // enable occlusion testing
      center                         // center on projected point
      style={{ pointerEvents: 'auto' }}  // enable click-through
    >
      <div className="health-bar">
        <div className="fill" style={{ width: '75%' }} />
      </div>
    </Html>
  </mesh>
)
```

The `<Html>` component:
- Renders its children into a portal inside the overlay div
- Inherits world position from its parent `<group>` or `<mesh>`
- Applies the `position` prop as a local offset
- Updates projection in `useFrame`

## Performance

- **DOM updates**: Only `transform` is changed each frame (GPU-composited, no layout reflow).
- **Projection**: Simple matrix multiplication per overlay element (<0.01ms each).
- **Typical count**: 5–50 overlay elements. At this count, the total per-frame cost is negligible.
- **Batch updates**: All overlay transforms are updated in a single `requestAnimationFrame` callback after the 3D render, using a `documentFragment` for batch DOM writes if needed.
- **CSS `will-change: transform`** is set on overlay elements to hint the browser to promote them to GPU layers.

## Styling

Overlay elements are standard DOM elements with standard CSS:

```css
.nametag {
  color: white;
  font-size: 14px;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.8);
  white-space: nowrap;
  user-select: none;
}

.health-bar {
  width: 60px;
  height: 6px;
  background: rgba(0, 0, 0, 0.5);
  border-radius: 3px;
  overflow: hidden;
}

.health-bar .fill {
  height: 100%;
  background: #4CAF50;
  transition: width 0.2s;
}
```
