# HTML Overlay

## Overview

The HTML overlay renders DOM elements at 3D scene positions, on top of the canvas. This is useful for labels, health bars, nameplates, tooltips, and other UI that should track 3D objects.

Similar to `CSS3DRenderer` in three.js or `<Html>` in Drei, but simpler and more focused.

## Architecture

```
┌─────────────────────────────────────┐
│ Overlay container (position:absolute)│  ← Covers the canvas
│   ┌─────────┐  ┌──────────────┐     │
│   │ Label A  │  │ Health bar B │     │  ← Positioned HTML elements
│   └─────────┘  └──────────────┘     │
├─────────────────────────────────────┤
│ <canvas> (WebGL2/WebGPU)            │  ← 3D rendering
└─────────────────────────────────────┘
```

## Implementation

```typescript
class HtmlOverlay {
  private _container: HTMLDivElement
  private _entries: Map<number, OverlayEntry>
  private _camera: Camera | null
  private _tempVec4: Float32Array

  constructor(canvas: HTMLCanvasElement) {
    // Create overlay container positioned over the canvas
    this._container = document.createElement('div')
    this._container.style.position = 'absolute'
    this._container.style.top = '0'
    this._container.style.left = '0'
    this._container.style.width = '100%'
    this._container.style.height = '100%'
    this._container.style.pointerEvents = 'none'
    this._container.style.overflow = 'hidden'

    canvas.parentElement!.style.position = 'relative'
    canvas.parentElement!.appendChild(this._container)

    this._entries = new Map()
    this._tempVec4 = new Float32Array(4)
  }

  // Add an HTML element at a 3D position
  add(
    id: number,
    element: HTMLElement,
    worldPosition: Vec3,
    options?: OverlayOptions,
  ): void {
    element.style.position = 'absolute'
    element.style.pointerEvents = options?.pointerEvents ? 'auto' : 'none'
    element.style.willChange = 'transform' // hint for browser compositing

    this._container.appendChild(element)
    this._entries.set(id, {
      element,
      worldPosition,
      occlude: options?.occlude ?? false,
      zIndexRange: options?.zIndexRange ?? [16777271, 0],
      center: options?.center ?? true,
    })
  }

  // Remove an overlay element
  remove(id: number): void {
    const entry = this._entries.get(id)
    if (entry) {
      this._container.removeChild(entry.element)
      this._entries.delete(id)
    }
  }

  // Update positions each frame
  update(camera: Camera, canvasWidth: number, canvasHeight: number): void {
    const vp = camera.viewProjectionMatrix

    for (const [, entry] of this._entries) {
      const { worldPosition, element, center } = entry

      // Project world position to clip space
      const x = worldPosition.x
      const y = worldPosition.y
      const z = worldPosition.z
      const e = vp.elements

      const clipX = e[0]*x + e[4]*y + e[8]*z + e[12]
      const clipY = e[1]*x + e[5]*y + e[9]*z + e[13]
      const clipW = e[3]*x + e[7]*y + e[11]*z + e[15]

      // Behind camera check
      if (clipW <= 0) {
        element.style.display = 'none'
        continue
      }

      // NDC to screen coordinates
      const ndcX = clipX / clipW
      const ndcY = clipY / clipW

      // NDC [-1,1] → screen pixels
      // Note: Y is flipped (NDC Y up, screen Y down)
      const screenX = (ndcX * 0.5 + 0.5) * canvasWidth
      const screenY = (1 - (ndcY * 0.5 + 0.5)) * canvasHeight

      element.style.display = ''
      const tx = center ? 'translate(-50%, -50%)' : ''
      element.style.transform = `translate(${screenX}px, ${screenY}px) ${tx}`

      // Depth-based z-index (nearer = higher z-index)
      const depth = clipW
      const [maxZ, minZ] = entry.zIndexRange
      const normalizedDepth = Math.max(0, Math.min(1, (depth - 0.1) / 1000))
      element.style.zIndex = String(Math.round(maxZ - normalizedDepth * (maxZ - minZ)))
    }
  }

  // Attach to a node (updates worldPosition from node each frame)
  attachToNode(id: number, node: Node, offset?: Vec3): void {
    const entry = this._entries.get(id)
    if (!entry) return

    entry.trackedNode = node
    entry.trackOffset = offset ?? new Vec3()
  }

  // Pre-update: sync tracked node positions
  syncTrackedNodes(): void {
    for (const [, entry] of this._entries) {
      if (entry.trackedNode) {
        entry.worldPosition.copyFrom(entry.trackedNode.worldMatrix.getTranslation())
        if (entry.trackOffset) {
          entry.worldPosition.add(entry.trackOffset)
        }
      }
    }
  }

  dispose(): void {
    this._container.remove()
    this._entries.clear()
  }
}
```

## Overlay Options

```typescript
interface OverlayOptions {
  occlude?: boolean           // hide when behind other objects (uses depth readback)
  pointerEvents?: boolean     // allow mouse interaction with this element
  center?: boolean            // center the element on the projected point (default true)
  zIndexRange?: [number, number] // z-index range for depth sorting
}
```

## Occlusion (Optional)

For occlusion testing (hiding labels behind walls), we can read the depth buffer at the projected screen position and compare with the element's depth:

```typescript
// Approximate occlusion via depth buffer readback
// This is a per-frame GPU readback — use sparingly (only for a few elements)
const testOcclusion = (
  screenX: number,
  screenY: number,
  elementDepth: number,
  depthBuffer: Float32Array, // readback of depth texture
  canvasWidth: number,
): boolean => {
  const px = Math.floor(screenX)
  const py = Math.floor(screenY)
  const idx = py * canvasWidth + px
  const sceneDepth = depthBuffer[idx]
  return elementDepth > sceneDepth + 0.001 // element is behind scene geometry
}
```

For WebGPU, depth readback uses `copyTextureToBuffer` + `mapAsync`. For WebGL2, `gl.readPixels` on the depth attachment. Both are async and have a frame of latency, which is acceptable for UI elements.

## Usage Example

```typescript
const overlay = new HtmlOverlay(canvas)

// Create a label
const label = document.createElement('div')
label.textContent = 'Player 1'
label.className = 'player-label'

// Add at a world position
overlay.add(1, label, new Vec3(10, 5, 2))

// Or attach to a node (tracks automatically)
overlay.add(2, healthBar, new Vec3(), { pointerEvents: true })
overlay.attachToNode(2, playerMesh, new Vec3(0, 0, 2.5)) // offset 2.5m above

// Each frame (called by the engine)
overlay.syncTrackedNodes()
overlay.update(camera, canvas.width, canvas.height)
```

## CSS Styling

The overlay elements are standard DOM elements — they can be styled with CSS:

```css
.player-label {
  color: white;
  font-size: 14px;
  text-shadow: 0 1px 3px rgba(0,0,0,0.8);
  white-space: nowrap;
  user-select: none;
}
```

## Performance Notes

- `transform: translate()` is used instead of `left`/`top` to keep positioning on the GPU compositor layer
- `will-change: transform` hints the browser to promote the element to its own compositing layer
- `pointer-events: none` on the container avoids interfering with canvas mouse events
- For large numbers of overlay elements (50+), consider batching position updates or using a single canvas-based label renderer instead
