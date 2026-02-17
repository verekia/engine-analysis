# Interaction

## Orbit Controls

### Overview

`OrbitControls` provides a familiar orbit camera for inspecting 3D scenes. The camera orbits around a target point, with support for rotation, panning, and zooming via mouse/touch input.

### Behavior

| Input | Action |
|-------|--------|
| Left mouse / one-finger drag | Orbit (rotate around target) |
| Right mouse / two-finger drag | Pan (move target and camera) |
| Scroll wheel / pinch | Zoom (move closer/farther from target) |
| Middle mouse drag | Pan (alternative) |

### Configuration

```typescript
interface OrbitControlsConfig {
  target?: Vec3               // orbit center, default [0, 0, 0]
  minDistance?: number         // min zoom distance, default 0.1
  maxDistance?: number         // max zoom distance, default Infinity
  minPolarAngle?: number      // min elevation (from Z-up), default 0.01
  maxPolarAngle?: number      // max elevation, default Math.PI - 0.01
  enableDamping?: boolean     // smooth deceleration, default true
  dampingFactor?: number      // damping strength, default 0.1
  rotateSpeed?: number        // rotation sensitivity, default 1.0
  panSpeed?: number           // pan sensitivity, default 1.0
  zoomSpeed?: number          // zoom sensitivity, default 1.0
  enableRotate?: boolean      // default true
  enablePan?: boolean         // default true
  enableZoom?: boolean        // default true
}
```

### Implementation

The orbit controls work in spherical coordinates (radius, polar angle θ, azimuthal angle φ) relative to the target point, adapted for Z-up:

```
Camera position = target + sphericalToCartesian(radius, theta, phi)

Where (Z-up, right-handed):
  x = radius * sin(theta) * cos(phi)
  y = radius * sin(theta) * sin(phi)
  z = radius * cos(theta)

theta = polar angle from Z axis (0 = looking down, π = looking up)
phi = azimuthal angle in XY plane
```

```typescript
class OrbitControls {
  camera: Camera
  domElement: HTMLElement

  target: Vec3
  private _spherical: { radius: number, theta: number, phi: number }
  private _sphericalDelta: { theta: number, phi: number }
  private _panOffset: Vec3
  private _zoomDelta: number

  constructor(camera: Camera, domElement: HTMLElement, config?: OrbitControlsConfig) {
    this.camera = camera
    this.domElement = domElement
    this.target = config?.target ?? new Vec3(0, 0, 0)

    // Initialize spherical from current camera position
    const offset = new Vec3().subVectors(camera.position, this.target)
    this._spherical = {
      radius: offset.length(),
      theta: Math.acos(offset.z / offset.length()),
      phi: Math.atan2(offset.y, offset.x),
    }

    this._attachListeners()
  }

  update(dt: number): void {
    // Apply damping
    if (this.enableDamping) {
      this._spherical.theta += this._sphericalDelta.theta * this.dampingFactor
      this._spherical.phi += this._sphericalDelta.phi * this.dampingFactor
      this._sphericalDelta.theta *= (1 - this.dampingFactor)
      this._sphericalDelta.phi *= (1 - this.dampingFactor)
    } else {
      this._spherical.theta += this._sphericalDelta.theta
      this._spherical.phi += this._sphericalDelta.phi
      this._sphericalDelta.theta = 0
      this._sphericalDelta.phi = 0
    }

    // Clamp polar angle
    this._spherical.theta = Math.max(
      this.minPolarAngle,
      Math.min(this.maxPolarAngle, this._spherical.theta),
    )

    // Apply zoom
    this._spherical.radius *= (1 + this._zoomDelta)
    this._spherical.radius = Math.max(
      this.minDistance,
      Math.min(this.maxDistance, this._spherical.radius),
    )
    this._zoomDelta *= (1 - this.dampingFactor)

    // Apply pan
    this.target.add(this._panOffset)
    this._panOffset.scale(1 - this.dampingFactor)

    // Compute camera position from spherical coords
    const { radius, theta, phi } = this._spherical
    this.camera.position.set(
      this.target.x + radius * Math.sin(theta) * Math.cos(phi),
      this.target.y + radius * Math.sin(theta) * Math.sin(phi),
      this.target.z + radius * Math.cos(theta),
    )

    // Look at target
    this.camera.lookAt(this.target)
  }

  dispose(): void {
    this._detachListeners()
  }
}
```

### Input Handling

```typescript
// Mouse events
private _onPointerDown = (e: PointerEvent) => {
  this.domElement.setPointerCapture(e.pointerId)
  this._pointers.set(e.pointerId, { x: e.clientX, y: e.clientY, button: e.button })
}

private _onPointerMove = (e: PointerEvent) => {
  const prev = this._pointers.get(e.pointerId)
  if (!prev) return

  const dx = e.clientX - prev.x
  const dy = e.clientY - prev.y
  prev.x = e.clientX
  prev.y = e.clientY

  if (this._pointers.size === 1) {
    if (prev.button === 0 && this.enableRotate) {
      // Left button: orbit
      const factor = 2 * Math.PI * this.rotateSpeed / this.domElement.clientHeight
      this._sphericalDelta.phi -= dx * factor
      this._sphericalDelta.theta -= dy * factor
    } else if ((prev.button === 2 || prev.button === 1) && this.enablePan) {
      // Right/middle button: pan
      this._pan(dx, dy)
    }
  } else if (this._pointers.size === 2) {
    // Two-finger: pan + pinch zoom
    this._handleTwoFingerGesture()
  }
}

private _onWheel = (e: WheelEvent) => {
  if (!this.enableZoom) return
  e.preventDefault()
  const delta = e.deltaY > 0 ? 0.05 : -0.05
  this._zoomDelta += delta * this.zoomSpeed
}

private _pan = (dx: number, dy: number) => {
  // Pan in the camera's local XY plane
  const distance = this._spherical.radius
  const fovFactor = 2 * Math.tan(this.camera.fov / 2) * distance
  const panX = -dx * fovFactor / this.domElement.clientHeight * this.panSpeed
  const panY = dy * fovFactor / this.domElement.clientHeight * this.panSpeed

  // Camera right and up vectors (Z-up world)
  const right = _tempVec3a.set(1, 0, 0).applyQuat(this.camera.rotation)
  const up = _tempVec3b.set(0, 0, 1) // world up for Z-up

  this._panOffset.addScaled(right, panX)
  this._panOffset.addScaled(up, panY)
}
```

### Touch Gestures

Touch events are handled through the Pointer Events API (unified mouse/touch):

- **One finger**: orbit rotation
- **Two fingers**: simultaneous pan (center of fingers) + pinch zoom (distance between fingers)
- Touch events use `setPointerCapture` for reliable tracking

## HTML Overlay

### Overview

The HTML overlay renders DOM elements at 3D scene positions, on top of the canvas. This is useful for labels, health bars, nameplates, tooltips, and other UI that should track 3D objects.

Similar to `CSS3DRenderer` in three.js or `<Html>` in Drei, but simpler and more focused.

### Architecture

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

### Implementation

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

### Overlay Options

```typescript
interface OverlayOptions {
  occlude?: boolean           // hide when behind other objects (uses depth readback)
  pointerEvents?: boolean     // allow mouse interaction with this element
  center?: boolean            // center the element on the projected point (default true)
  zIndexRange?: [number, number] // z-index range for depth sorting
}
```

### Occlusion (Optional)

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

### Usage Example

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

### CSS Styling

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

### Performance Notes

- `transform: translate()` is used instead of `left`/`top` to keep positioning on the GPU compositor layer
- `will-change: transform` hints the browser to promote the element to its own compositing layer
- `pointer-events: none` on the container avoids interfering with canvas mouse events
- For large numbers of overlay elements (50+), consider batching position updates or using a single canvas-based label renderer instead
