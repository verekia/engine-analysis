# HTML Overlay

## Overview

The HTML overlay system renders DOM elements positioned at 3D world coordinates on top of the WebGL/WebGPU canvas. This is similar to drei's `<Html>` component or three.js CSS3DRenderer, but simpler and more focused.

Use cases:
- Health bars above characters
- Item labels / tooltips
- UI markers and waypoints
- Name tags

## Architecture

```
┌──────────────────────────┐
│   Canvas (WebGL/WebGPU)  │  z-index: 0
├──────────────────────────┤
│   Overlay Container      │  z-index: 1, pointer-events: none
│  ┌────────┐ ┌──────────┐│
│  │ Label  │ │ HP Bar   ││  Absolutely positioned DOM elements
│  └────────┘ └──────────┘│
└──────────────────────────┘
```

The overlay container is a `<div>` that sits on top of the canvas with `pointer-events: none` (so it doesn't block canvas input). Individual overlay items can opt in to `pointer-events: auto` if they need to be clickable.

## OverlayManager

```typescript
interface OverlayItem {
  id: number
  worldPosition: Vec3           // 3D position to track
  element: HTMLElement           // The DOM element to position
  offsetX?: number               // Pixel offset from projected point
  offsetY?: number
  occlude?: boolean              // Hide when behind other objects (default: false)
  zIndexRange?: [number, number] // Auto z-index based on distance
  onVisibilityChange?: (visible: boolean) => void
  _visible: boolean
  _screenX: number
  _screenY: number
}

interface OverlayManager {
  readonly container: HTMLDivElement

  add(item: OverlayItemOptions): OverlayItem
  remove(item: OverlayItem): void
  update(camera: Camera, viewportWidth: number, viewportHeight: number): void
  dispose(): void
}
```

## Setup

```typescript
const createOverlayManager = (canvasContainer: HTMLElement): OverlayManager => {
  const container = document.createElement('div')
  container.style.position = 'absolute'
  container.style.top = '0'
  container.style.left = '0'
  container.style.width = '100%'
  container.style.height = '100%'
  container.style.overflow = 'hidden'
  container.style.pointerEvents = 'none'
  container.style.zIndex = '1'
  canvasContainer.style.position = 'relative'
  canvasContainer.appendChild(container)

  const items: OverlayItem[] = []

  return { container, items, add, remove, update, dispose }
}
```

## Projection

Each frame, world positions are projected to screen coordinates:

```typescript
const updateOverlay = (manager: OverlayManager, camera: Camera, vpWidth: number, vpHeight: number) => {
  const vp = camera.viewProjectionMatrix

  for (const item of manager.items) {
    const pos = item.worldPosition

    // Project to clip space
    const clipX = vp[0] * pos[0] + vp[4] * pos[1] + vp[8] * pos[2] + vp[12]
    const clipY = vp[1] * pos[0] + vp[5] * pos[1] + vp[9] * pos[2] + vp[13]
    const clipZ = vp[2] * pos[0] + vp[6] * pos[1] + vp[10] * pos[2] + vp[14]
    const clipW = vp[3] * pos[0] + vp[7] * pos[1] + vp[11] * pos[2] + vp[15]

    // Behind camera check
    if (clipW <= 0) {
      if (item._visible) {
        item.element.style.display = 'none'
        item._visible = false
        item.onVisibilityChange?.(false)
      }
      continue
    }

    // NDC → screen
    const ndcX = clipX / clipW
    const ndcY = clipY / clipW

    const screenX = (ndcX * 0.5 + 0.5) * vpWidth + (item.offsetX ?? 0)
    const screenY = (1 - (ndcY * 0.5 + 0.5)) * vpHeight + (item.offsetY ?? 0)

    // Frustum check (is it on screen?)
    const onScreen = ndcX >= -1.2 && ndcX <= 1.2 && ndcY >= -1.2 && ndcY <= 1.2

    if (!onScreen) {
      if (item._visible) {
        item.element.style.display = 'none'
        item._visible = false
        item.onVisibilityChange?.(false)
      }
      continue
    }

    // Show and position
    if (!item._visible) {
      item.element.style.display = ''
      item._visible = true
      item.onVisibilityChange?.(true)
    }

    // Only update DOM if position changed (avoid layout thrashing)
    if (Math.abs(screenX - item._screenX) > 0.5 || Math.abs(screenY - item._screenY) > 0.5) {
      item.element.style.transform = `translate(${screenX}px, ${screenY}px)`
      item._screenX = screenX
      item._screenY = screenY
    }

    // Distance-based z-index
    if (item.zIndexRange) {
      const dist = clipW // W = view-space Z distance
      const [zNear, zFar] = item.zIndexRange
      const t = Math.max(0, Math.min(1, dist / 200))
      item.element.style.zIndex = String(Math.round(zFar + (zNear - zFar) * (1 - t)))
    }
  }
}
```

## Occlusion

Optional occlusion testing hides overlay items that are behind geometry. This uses raycasting from the camera to the overlay's world position:

```typescript
const checkOcclusion = (item: OverlayItem, camera: Camera, scene: Scene) => {
  if (!item.occlude) return true // Always visible

  const ray = createRay(camera.getWorldPosition(), item.worldPosition)
  const dist = vec3.distance(camera.getWorldPosition(), item.worldPosition)
  const hits = raycastScene(ray, scene)

  // If closest hit is closer than the overlay position, it's occluded
  return hits.length === 0 || hits[0].distance >= dist - 0.1
}
```

Occlusion checks are expensive (raycasting), so they're throttled:
- Run every 3-5 frames instead of every frame
- Use a staggered schedule so different items check on different frames

## Usage

```typescript
const overlay = createOverlayManager(canvasContainer)

// Add a label above a character
const label = document.createElement('div')
label.textContent = 'Player 1'
label.className = 'player-label'

const overlayItem = overlay.add({
  worldPosition: character.position,  // Tracked automatically
  element: label,
  offsetY: -40,                       // 40px above the projected point
  occlude: true,
})

// Remove when done
overlay.remove(overlayItem)
```

## Performance Considerations

- DOM updates are batched: `transform` is the only property changed per frame (triggers compositing, not layout)
- Elements behind the camera or off-screen are hidden (`display: none`) to avoid rendering cost
- Use `will-change: transform` on overlay items for GPU-composited movement
- Keep overlay element count reasonable (< 100 visible at once)
- Position uses `translate()` instead of `left/top` to stay on the compositor thread

## CSS Integration

The overlay container uses `pointer-events: none` globally, but individual items can be interactive:

```css
.player-label {
  pointer-events: auto;  /* Clickable */
  cursor: pointer;
  position: absolute;
  left: 0;
  top: 0;
  transform-origin: center center;
  /* Centering: offset by half width/height */
  margin-left: -50%;
  white-space: nowrap;
  will-change: transform;
}
```
