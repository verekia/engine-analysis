# HTML Overlay

## Overview

Mantis provides an HTML overlay system that renders DOM elements at 3D scene
positions on top of the WebGL/WebGPU canvas. This is similar to Three.js's
CSS3DRenderer or Drei's `<Html>` component, but designed for performance on
mobile.

Use cases:
- Health bars above characters
- Name labels floating over objects
- Tooltips at interaction points
- UI panels anchored to world positions

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  Container div                   │
│  ┌─────────────────────────────────────────────┐ │
│  │           WebGL/WebGPU <canvas>             │ │
│  │                                              │ │
│  │       3D scene rendered here                 │ │
│  │                                              │ │
│  └─────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────┐ │
│  │       Overlay div (pointer-events: none)     │ │
│  │                                              │ │
│  │  ┌──────┐   ┌────────┐   ┌──────────┐      │ │
│  │  │Label │   │HP: 100 │   │ Tooltip  │      │ │
│  │  └──────┘   └────────┘   └──────────┘      │ │
│  │  (absolute positioned, CSS transform)        │ │
│  │                                              │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

The overlay div sits on top of the canvas via CSS stacking. Each overlay
element is absolutely positioned and moved via `CSS transform: translate()` for
GPU-accelerated compositing — no layout reflows.

## 3D → 2D Projection

World positions are projected to screen coordinates using the camera's
view-projection matrix:

```typescript
const projectToScreen = (
  worldPos: Vec3,
  vpMatrix: Mat4,
  canvasWidth: number,
  canvasHeight: number,
): { x: number, y: number, visible: boolean } => {
  // World → clip space
  const clipX = vpMatrix[0] * worldPos[0] + vpMatrix[4] * worldPos[1] + vpMatrix[8]  * worldPos[2] + vpMatrix[12]
  const clipY = vpMatrix[1] * worldPos[0] + vpMatrix[5] * worldPos[1] + vpMatrix[9]  * worldPos[2] + vpMatrix[13]
  const clipZ = vpMatrix[2] * worldPos[0] + vpMatrix[6] * worldPos[1] + vpMatrix[10] * worldPos[2] + vpMatrix[14]
  const clipW = vpMatrix[3] * worldPos[0] + vpMatrix[7] * worldPos[1] + vpMatrix[11] * worldPos[2] + vpMatrix[15]

  // Behind camera check
  if (clipW <= 0) return { x: 0, y: 0, visible: false }

  // Clip → NDC
  const ndcX = clipX / clipW
  const ndcY = clipY / clipW
  const ndcZ = clipZ / clipW

  // Frustum check (outside [-1, 1] in any axis)
  if (ndcX < -1.2 || ndcX > 1.2 || ndcY < -1.2 || ndcY > 1.2 || ndcZ < -1 || ndcZ > 1) {
    return { x: 0, y: 0, visible: false }
  }

  // NDC → screen pixels
  const screenX = (ndcX * 0.5 + 0.5) * canvasWidth
  const screenY = (1 - (ndcY * 0.5 + 0.5)) * canvasHeight  // flip Y for screen coords

  return { x: screenX, y: screenY, visible: true }
}
```

The slight out-of-bounds tolerance (±1.2 instead of ±1.0) prevents elements
from popping in/out at the screen edges.

## DOM Element Positioning

Each overlay element uses CSS `transform` for positioning:

```typescript
const updateOverlayElement = (
  element: HTMLElement,
  screenPos: { x: number, y: number, visible: boolean },
  center: boolean,
) => {
  if (!screenPos.visible) {
    element.style.display = 'none'
    return
  }

  element.style.display = ''

  // Use transform for GPU-composited positioning (no layout reflow)
  const tx = center ? `calc(${screenPos.x}px - 50%)` : `${screenPos.x}px`
  const ty = center ? `calc(${screenPos.y}px - 50%)` : `${screenPos.y}px`
  element.style.transform = `translate(${tx}, ${ty})`
}
```

**Why CSS `transform`?**
- Browser composits `transform` changes on the GPU — no layout reflow
- Moving 100 overlay elements costs < 0.01 ms total (just style writes)
- The browser batches all style changes and paints once

**CSS setup for overlay elements:**
```css
.mantis-overlay-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;    /* let clicks pass through to canvas */
  overflow: hidden;
}

.mantis-overlay-element {
  position: absolute;
  top: 0;
  left: 0;
  will-change: transform;
  pointer-events: auto;   /* enable interaction on the element itself */
}
```

## Occlusion

Without occlusion testing, labels appear through walls. Mantis provides two
occlusion strategies:

### Strategy 1: Depth-Based (GPU Readback)

Read the depth buffer at the projected screen position and compare with the
overlay's world-space depth:

```typescript
const isOccluded = (screenX: number, screenY: number, worldDepth: number): boolean => {
  // Read depth at pixel (async — 1 frame latency)
  const depthAtPixel = depthReadback[screenY * width + screenX]
  return worldDepth > depthAtPixel
}
```

**Caveats:**
- Requires GPU → CPU depth buffer readback (expensive on mobile)
- One frame of latency (read frame N, use result in frame N+1)
- Best for few overlay elements (< 10)

### Strategy 2: Raycast-Based (CPU)

Cast a ray from the camera to the overlay's world position. If the ray hits
any opaque object before reaching the overlay, it's occluded:

```typescript
const isOccluded = (cameraPos: Vec3, worldPos: Vec3): boolean => {
  const direction = vec3Sub(worldPos, cameraPos, tempVec)
  const distance = vec3Length(direction)
  vec3Normalize(direction, direction)

  const ray = { origin: cameraPos, direction }
  const hit = engine.raycast(ray, { maxDistance: distance - 0.1 })

  return hit !== null
}
```

**Advantages:**
- No GPU readback needed
- Pixel-exact occlusion
- Works well for < 20 overlay elements (< 0.1 ms total)

### Default: No Occlusion

For many games, overlay elements represent UI that should always be visible
(health bars, name tags). Occlusion is opt-in per element.

## Imperative API

```typescript
// Create an overlay
const overlay = engine.createHtmlOverlay({
  worldPosition: [10, 5, 3],
  center: true,                   // center element on projected position
  occlude: false,                 // disable occlusion testing
})

// Set DOM content
const div = document.createElement('div')
div.innerHTML = '<span class="hp-bar">HP: 100</span>'
overlay.setContent(div)

// Update position (e.g., following a moving character)
engine.onFrame(() => {
  overlay.worldPosition = character.position
})

// Remove
overlay.dispose()
```

## React Integration

The `<Html>` component makes overlays declarative:

```tsx
import { Html } from 'mantis/react'

const HealthBar = ({ hp }: { hp: number }) => (
  <mesh position={[0, 0, 2]}>
    <sphereGeometry args={{ radius: 0.5 }} />
    <lambertMaterial color={[1, 0, 0]} />

    {/* Positioned relative to parent mesh, offset by [0, 0, 1] */}
    <Html position={[0, 0, 1.5]} center occlude="raycast">
      <div className="health-bar">
        <div className="health-fill" style={{ width: `${hp}%` }} />
      </div>
    </Html>
  </mesh>
)
```

### `<Html>` Props

```typescript
interface HtmlProps {
  position?: [number, number, number]  // offset from parent's world position
  center?: boolean                      // center on projected point (default true)
  occlude?: boolean | 'raycast'         // occlusion mode
  zIndexRange?: [number, number]        // z-index range for depth sorting
  style?: React.CSSProperties          // applied to wrapper div
  className?: string
  distanceFactor?: number               // scale with distance (0 = fixed size)
  children: React.ReactNode
}
```

### Portal-Based Rendering

`<Html>` renders its children into a React portal targeting the overlay
container div. This means:
- React context (themes, state) flows through to overlay content
- Overlay DOM is separate from the canvas (proper layering)
- Standard React event handling works on overlay elements

```typescript
// Simplified Html component implementation
const Html = ({ children, position, center = true, occlude = false, ...props }) => {
  const meshRef = useParentMesh()
  const overlayContainer = useOverlayContainer()
  const wrapperRef = useRef<HTMLDivElement>(null)

  useFrame(() => {
    if (!wrapperRef.current || !meshRef.current) return

    // Compute world position = parent world pos + offset
    const worldPos = vec3Pool.acquire()
    const parentMatrix = meshRef.current.worldMatrix
    vec3Set(worldPos, position?.[0] ?? 0, position?.[1] ?? 0, position?.[2] ?? 0)
    vec3TransformMat4(worldPos, parentMatrix, worldPos)

    // Project to screen
    const screen = projectToScreen(worldPos, camera.vpMatrix, width, height)

    // Occlusion check
    if (occlude && screen.visible) {
      screen.visible = !isOccluded(camera.position, worldPos)
    }

    // Update DOM position
    updateOverlayElement(wrapperRef.current, screen, center)
  })

  return createPortal(
    <div ref={wrapperRef} className="mantis-overlay-element" {...props}>
      {children}
    </div>,
    overlayContainer
  )
}
```

## Performance

| Operation | Cost |
|---|---|
| Project 1 position (3D → 2D) | < 0.001 ms |
| Update 1 DOM element transform | < 0.0001 ms |
| 100 overlay elements total | < 0.15 ms |
| Raycast occlusion (1 test) | ~0.05 ms |
| Depth readback (GPU → CPU) | ~0.5 ms (1 frame latency) |

For games with < 50 visible overlays, the total cost is negligible.

## Limitations

- DOM elements cannot be depth-tested against the 3D scene without explicit
  occlusion (they always render on top by default)
- CSS animations on overlay elements work but add to layout cost
- Very large numbers of overlays (> 200) may cause DOM performance issues —
  consider pooling/recycling elements
- Mobile browsers may have compositing limitations with many `will-change`
  elements — test on target devices
