# HTML Overlay

## Overview

The HTML overlay system renders DOM elements at 3D positions on top of the WebGL/WebGPU canvas. This is similar to Three.js's `CSS3DRenderer` or Drei's `<Html>` component, but built as a core feature with efficient world-to-screen projection.

Use cases:
- Health bars above characters
- Name tags / labels
- Tooltip popups at 3D positions
- Mini-maps or HUD elements anchored to world objects
- Interactive UI panels in 3D space

## Architecture

```
┌─────────────────────────────────────────────┐
│              Overlay Container               │
│   (position: absolute, pointer-events: none) │
│                                              │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│   │ Overlay 1 │  │ Overlay 2 │  │ Overlay 3 │ │
│   │ (health)  │  │ (label)   │  │ (tooltip) │ │
│   └──────────┘  └──────────┘  └──────────┘ │
│                                              │
├──────────────────────────────────────────────┤
│              <canvas> (WebGL/WebGPU)         │
└──────────────────────────────────────────────┘
```

The overlay container is a `<div>` positioned absolutely over the canvas with `pointer-events: none`. Individual overlay elements opt into pointer events as needed.

## World-to-Screen Projection

The core operation: convert a 3D world position to 2D screen coordinates.

```typescript
const worldToScreen = (
  worldPos: Vec3,
  camera: Camera,
  canvasWidth: number,
  canvasHeight: number,
  out: { x: number, y: number, visible: boolean }
): void => {
  // Project world position to clip space
  // clipPos = projectionMatrix × viewMatrix × worldPos
  const vp = camera._viewProjectionMatrix
  const x = worldPos.x * vp[0] + worldPos.y * vp[4] + worldPos.z * vp[8] + vp[12]
  const y = worldPos.x * vp[1] + worldPos.y * vp[5] + worldPos.z * vp[9] + vp[13]
  const z = worldPos.x * vp[2] + worldPos.y * vp[6] + worldPos.z * vp[10] + vp[14]
  const w = worldPos.x * vp[3] + worldPos.y * vp[7] + worldPos.z * vp[11] + vp[15]

  // Behind camera check
  if (w <= 0) {
    out.visible = false
    return
  }

  // Perspective divide → NDC [-1, 1]
  const ndcX = x / w
  const ndcY = y / w
  const ndcZ = z / w

  // Check if within view frustum
  if (ndcX < -1.2 || ndcX > 1.2 || ndcY < -1.2 || ndcY > 1.2 || ndcZ < -1 || ndcZ > 1) {
    out.visible = false
    return
  }

  // NDC → screen pixels
  out.x = (ndcX * 0.5 + 0.5) * canvasWidth
  out.y = (1 - (ndcY * 0.5 + 0.5)) * canvasHeight  // Flip Y (screen Y is top-down)
  out.visible = true
}
```

The slight over-range check (±1.2 instead of ±1.0) provides a small buffer so overlays don't pop in/out at the exact screen edge.

## HTMLOverlay System

```typescript
interface HTMLOverlaySystem {
  readonly container: HTMLDivElement    // The overlay container element

  // Create a new overlay
  create(options: OverlayOptions): HTMLOverlay

  // Remove an overlay
  remove(overlay: HTMLOverlay): void

  // Update all overlay positions (called each frame)
  update(camera: Camera): void

  // Clean up
  dispose(): void
}

interface OverlayOptions {
  // 3D position (updated each frame from node or static position)
  position?: Vec3                     // Static world position
  node?: Node                         // Track a scene node's world position
  offset?: Vec3                       // Offset from node position (in world space)

  // The DOM element to display
  element: HTMLElement

  // Behavior
  center?: boolean                    // Center element on projected point (default: true)
  occlude?: boolean                   // Hide when behind other objects (default: false)
  distanceScale?: boolean             // Scale element based on distance (default: false)
  minDistance?: number                 // Don't show if closer than this (default: 0)
  maxDistance?: number                 // Don't show if farther than this (default: Infinity)
  zIndexRange?: [number, number]      // Map depth to z-index range (default: [16777271, 0])
  pointerEvents?: boolean             // Enable pointer events (default: false)
}

interface HTMLOverlay {
  options: OverlayOptions
  readonly element: HTMLElement
  readonly visible: boolean           // Currently visible on screen
  readonly screenX: number            // Current screen X
  readonly screenY: number            // Current screen Y

  // Update position at runtime
  setPosition(pos: Vec3): void
  setNode(node: Node): void

  // Show/hide
  show(): void
  hide(): void

  // Clean up
  dispose(): void
}
```

## Update Loop

Each frame, all overlays are batch-updated:

```typescript
const updateOverlays = (system: HTMLOverlaySystem, camera: Camera) => {
  const canvas = system._engine.canvas
  const width = canvas.clientWidth
  const height = canvas.clientHeight

  for (const overlay of system._overlays) {
    // Determine world position
    let worldPos: Vec3

    if (overlay.options.node) {
      worldPos = overlay.options.node.worldPosition
      if (overlay.options.offset) {
        worldPos = vec3Add(_v0, worldPos, overlay.options.offset)
      }
    } else if (overlay.options.position) {
      worldPos = overlay.options.position
    } else {
      continue
    }

    // Distance check
    const distance = vec3Distance(worldPos, camera.worldPosition)
    if (distance < overlay.options.minDistance || distance > overlay.options.maxDistance) {
      overlay._setVisible(false)
      continue
    }

    // Project to screen
    worldToScreen(worldPos, camera, width, height, _screenResult)

    if (!_screenResult.visible) {
      overlay._setVisible(false)
      continue
    }

    overlay._setVisible(true)
    overlay._screenX = _screenResult.x
    overlay._screenY = _screenResult.y

    // Apply CSS transform (using translate3d for GPU-accelerated positioning)
    const el = overlay.element
    let transform = `translate3d(${_screenResult.x}px, ${_screenResult.y}px, 0)`

    if (overlay.options.center) {
      transform += ' translate(-50%, -50%)'
    }

    if (overlay.options.distanceScale) {
      const scale = Math.max(0.1, 1 / (distance * 0.1))
      transform += ` scale(${scale})`
    }

    el.style.transform = transform

    // Z-index based on depth (closer = higher z-index)
    if (overlay.options.zIndexRange) {
      const [max, min] = overlay.options.zIndexRange
      const t = Math.max(0, Math.min(1, distance / camera.far))
      el.style.zIndex = String(Math.round(max - t * (max - min)))
    }
  }
}
```

### Performance Optimization

- **`translate3d`**: Uses GPU-accelerated CSS transform (not `left`/`top`)
- **`will-change: transform`**: Set on overlay elements to hint browser for optimization
- **Batch reads, batch writes**: All world-to-screen projections computed first, then all DOM updates applied (avoids layout thrashing)
- **Visibility toggle via `display: none`**: Hidden overlays don't participate in layout
- **Dirty check**: Only update DOM if position actually changed (threshold: 0.5px)

```typescript
// Only update DOM if position changed significantly
const dx = Math.abs(overlay._screenX - overlay._prevScreenX)
const dy = Math.abs(overlay._screenY - overlay._prevScreenY)

if (dx > 0.5 || dy > 0.5) {
  el.style.transform = transform
  overlay._prevScreenX = overlay._screenX
  overlay._prevScreenY = overlay._screenY
}
```

## Occlusion

When `occlude: true`, overlays are hidden when the tracked position is behind scene geometry. This uses a single raycast from the camera to the overlay position:

```typescript
if (overlay.options.occlude) {
  const dir = vec3Normalize(_v0, vec3Sub(_v0, worldPos, camera.worldPosition))
  const dist = vec3Distance(worldPos, camera.worldPosition)

  raycaster.set(camera.worldPosition, dir)
  const hit = raycaster.intersectScene(scene)

  if (hit && hit.distance < dist - 0.1) {
    // Something is between camera and overlay target
    overlay._setVisible(false)
    continue
  }
}
```

Occlusion raycasting can be expensive with many overlays. Recommendations:
- Only enable for overlays that need it
- Use a throttled update (every 2-3 frames) for occlusion checks
- Use scene-level BVH for fast broad-phase rejection

## Container Setup

```typescript
const createOverlayContainer = (canvas: HTMLCanvasElement): HTMLDivElement => {
  const container = document.createElement('div')

  // Match canvas position and size
  container.style.position = 'absolute'
  container.style.top = '0'
  container.style.left = '0'
  container.style.width = '100%'
  container.style.height = '100%'
  container.style.overflow = 'hidden'
  container.style.pointerEvents = 'none' // Don't block canvas interactions

  // Insert after canvas in the same parent
  canvas.parentElement?.appendChild(container)

  return container
}
```

## Usage Example

```typescript
// Create overlay system
const overlays = createHTMLOverlaySystem(engine)

// Create a health bar above a character
const healthBar = document.createElement('div')
healthBar.className = 'health-bar'
healthBar.innerHTML = '<div class="health-fill" style="width: 75%"></div>'

const overlay = overlays.create({
  node: characterMesh,
  offset: new Vec3(0, 0, 2.5), // 2.5 units above character (Z-up)
  element: healthBar,
  center: true,
  maxDistance: 50, // Hide beyond 50 meters
  pointerEvents: false,
})

// Update health bar
const setHealth = (percent: number) => {
  healthBar.querySelector('.health-fill').style.width = `${percent}%`
}

// Remove when character dies
overlay.dispose()
```

## React Integration

The HTML overlay integrates naturally with the React bindings (see [react-bindings.md](react-bindings.md)):

```tsx
const HealthBar = ({ node, health }: { node: Node, health: number }) => (
  <Overlay node={node} offset={[0, 0, 2.5]} center maxDistance={50}>
    <div className="health-bar">
      <div className="health-fill" style={{ width: `${health}%` }} />
    </div>
  </Overlay>
)
```

The `<Overlay>` React component manages the overlay lifecycle and renders its children into the overlay container using a React portal.
