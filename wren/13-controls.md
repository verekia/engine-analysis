# 13 — Orbit Controls & HTML Overlay

## Orbit Controls

Camera controls for orbiting around a target point, with zoom and pan. Similar to three.js's `OrbitControls` but built for Wren's z-up coordinate system.

### State

```ts
interface OrbitControls {
  camera: CameraNode
  target: Float32Array         // [x, y, z] — orbit center
  enabled: boolean

  // Spherical coordinates (z-up)
  azimuth: number              // Horizontal angle (radians), 0 = +Y direction
  polar: number                // Angle from +Z axis (radians), 0 = top, π = bottom
  distance: number             // Distance from target

  // Limits
  minDistance: number           // Default: 0.1
  maxDistance: number           // Default: Infinity
  minPolar: number             // Default: 0.01 (avoid gimbal lock at pole)
  maxPolar: number             // Default: π - 0.01
  enablePan: boolean           // Default: true
  enableZoom: boolean          // Default: true
  enableRotate: boolean        // Default: true

  // Damping
  dampingFactor: number        // 0 = instant, 0.9 = very smooth. Default: 0.1
  rotateSensitivity: number    // Default: 1.0
  zoomSensitivity: number      // Default: 1.0
  panSensitivity: number       // Default: 1.0

  // Methods
  update(dt: number): void
  dispose(): void
}
```

### Input Handling

```ts
const createOrbitControls = (camera: CameraNode, canvas: HTMLCanvasElement): OrbitControls => {
  const controls = { /* ... initial state ... */ }

  // Velocity accumulators (for damping)
  let azimuthVelocity = 0
  let polarVelocity = 0
  let zoomVelocity = 0
  let panVelocity = [0, 0]

  // Pointer tracking
  let pointerDown = false
  let pointerButton = -1
  let prevX = 0, prevY = 0

  const onPointerDown = (e: PointerEvent) => {
    pointerDown = true
    pointerButton = e.button
    prevX = e.clientX
    prevY = e.clientY
    canvas.setPointerCapture(e.pointerId)
  }

  const onPointerMove = (e: PointerEvent) => {
    if (!pointerDown) return
    const dx = e.clientX - prevX
    const dy = e.clientY - prevY
    prevX = e.clientX
    prevY = e.clientY

    if (pointerButton === 0) {
      // Left click: rotate
      azimuthVelocity -= dx * 0.01 * controls.rotateSensitivity
      polarVelocity += dy * 0.01 * controls.rotateSensitivity
    } else if (pointerButton === 2 || pointerButton === 1) {
      // Right/middle click: pan
      panVelocity[0] -= dx * 0.002 * controls.distance * controls.panSensitivity
      panVelocity[1] += dy * 0.002 * controls.distance * controls.panSensitivity
    }
  }

  const onPointerUp = (e: PointerEvent) => {
    pointerDown = false
    canvas.releasePointerCapture(e.pointerId)
  }

  const onWheel = (e: WheelEvent) => {
    e.preventDefault()
    const zoomDelta = e.deltaY > 0 ? 1.1 : 0.9
    zoomVelocity *= zoomDelta
    if (zoomVelocity === 0) zoomVelocity = zoomDelta
  }

  // Touch support
  let touchStartDistance = 0
  let touchStartAzimuth = 0
  let touchStartPolar = 0

  const onTouchStart = (e: TouchEvent) => {
    if (e.touches.length === 1) {
      // Single finger: rotate
      prevX = e.touches[0].clientX
      prevY = e.touches[0].clientY
      pointerDown = true
      pointerButton = 0
    } else if (e.touches.length === 2) {
      // Two fingers: pinch zoom + pan
      touchStartDistance = getTouchDistance(e.touches)
    }
  }

  // Register events
  canvas.addEventListener('pointerdown', onPointerDown)
  canvas.addEventListener('pointermove', onPointerMove)
  canvas.addEventListener('pointerup', onPointerUp)
  canvas.addEventListener('wheel', onWheel, { passive: false })
  canvas.addEventListener('contextmenu', e => e.preventDefault())

  return controls
}
```

### Update Loop

```ts
const updateOrbitControls = (controls: OrbitControls, dt: number): void => {
  const damping = 1 - controls.dampingFactor

  // Apply velocities with damping
  controls.azimuth += azimuthVelocity
  controls.polar += polarVelocity
  azimuthVelocity *= damping
  polarVelocity *= damping

  // Clamp polar angle
  controls.polar = Math.max(controls.minPolar, Math.min(controls.maxPolar, controls.polar))

  // Apply zoom
  if (zoomVelocity !== 0) {
    controls.distance *= zoomVelocity
    controls.distance = Math.max(controls.minDistance, Math.min(controls.maxDistance, controls.distance))
    zoomVelocity = 1 + (zoomVelocity - 1) * damping
    if (Math.abs(zoomVelocity - 1) < 0.001) zoomVelocity = 0
  }

  // Apply pan (in camera-local XZ plane, since Z is up)
  if (panVelocity[0] !== 0 || panVelocity[1] !== 0) {
    // Right vector in world space
    const rightX = Math.cos(controls.azimuth)
    const rightY = Math.sin(controls.azimuth)
    // Up vector is always Z
    controls.target[0] += rightX * panVelocity[0]
    controls.target[1] += rightY * panVelocity[0]
    controls.target[2] += panVelocity[1]
    panVelocity[0] *= damping
    panVelocity[1] *= damping
  }

  // Convert spherical to Cartesian (z-up)
  const sinPolar = Math.sin(controls.polar)
  const cosPolar = Math.cos(controls.polar)
  const sinAzimuth = Math.sin(controls.azimuth)
  const cosAzimuth = Math.cos(controls.azimuth)

  const x = controls.target[0] + controls.distance * sinPolar * cosAzimuth
  const y = controls.target[1] + controls.distance * sinPolar * sinAzimuth
  const z = controls.target[2] + controls.distance * cosPolar

  setPosition(controls.camera, x, y, z)

  // Look at target (z-up)
  lookAt(controls.camera, controls.target, [0, 0, 1])
}
```

## HTML Overlay

Renders DOM elements at 3D scene positions, projected onto a 2D overlay div that sits on top of the canvas. Similar to three.js's CSS3DRenderer or drei's `<Html>` component.

### Architecture

```
┌─────────────────────────────────┐
│  Container div (position:relative) │
│  ┌────────────────────────────┐ │
│  │  Canvas (3D scene)         │ │
│  └────────────────────────────┘ │
│  ┌────────────────────────────┐ │
│  │  Overlay div               │ │  ← position:absolute, pointer-events:none
│  │  (same size as canvas)     │ │
│  │  ┌──────┐   ┌──────┐      │ │
│  │  │Label │   │HP bar│      │ │  ← Individual overlays have pointer-events:auto
│  │  └──────┘   └──────┘      │ │
│  └────────────────────────────┘ │
└─────────────────────────────────┘
```

### API

```ts
interface HtmlOverlay {
  container: HTMLDivElement         // The overlay container (created automatically)

  add(element: HTMLElement, options: OverlayOptions): OverlayHandle
  remove(handle: OverlayHandle): void
  update(camera: CameraNode): void  // Call each frame to reproject
  dispose(): void
}

interface OverlayOptions {
  // Position source (one of these):
  position?: Float32Array           // Static world position [x, y, z]
  node?: SceneNode                  // Track a scene node's world position
  bone?: { skeleton: Skeleton, boneName: string }  // Track a bone

  // Projection options
  center?: boolean                  // Center the element on the projected point. Default: true
  occlude?: boolean                 // Hide when behind other objects (via depth). Default: false
  zIndexRange?: [number, number]    // Z-index range for depth ordering. Default: [0, 1000]
  distanceFactor?: number           // If set, scale element by 1/distance * factor
  sprite?: boolean                  // If true, always face camera (no 3D rotation). Default: true
}

interface OverlayHandle {
  element: HTMLElement
  options: OverlayOptions
  visible: boolean
}
```

### Projection

Each frame, every overlay element is projected from 3D world space to 2D screen coordinates:

```ts
const updateOverlays = (overlay: HtmlOverlay, camera: CameraNode): void => {
  const vp = camera.viewProjectionMatrix
  const canvasWidth = overlay.container.clientWidth
  const canvasHeight = overlay.container.clientHeight

  for (const handle of overlay.handles) {
    // Get world position
    let worldPos: Float32Array
    if (handle.options.node) {
      worldPos = getWorldPosition(handle.options.node)
    } else if (handle.options.bone) {
      const bone = handle.options.bone.skeleton.getBoneByName(handle.options.bone.boneName)
      worldPos = getWorldPosition(bone)
    } else {
      worldPos = handle.options.position!
    }

    // Project to clip space
    const clipX = vp[0]*worldPos[0] + vp[4]*worldPos[1] + vp[8]*worldPos[2] + vp[12]
    const clipY = vp[1]*worldPos[0] + vp[5]*worldPos[1] + vp[9]*worldPos[2] + vp[13]
    const clipZ = vp[2]*worldPos[0] + vp[6]*worldPos[1] + vp[10]*worldPos[2] + vp[14]
    const clipW = vp[3]*worldPos[0] + vp[7]*worldPos[1] + vp[11]*worldPos[2] + vp[15]

    // Behind camera
    if (clipW <= 0) {
      handle.element.style.display = 'none'
      handle.visible = false
      continue
    }

    // NDC
    const ndcX = clipX / clipW
    const ndcY = clipY / clipW
    const ndcZ = clipZ / clipW  // 0 to 1 (depth)

    // Screen coordinates
    const screenX = (ndcX * 0.5 + 0.5) * canvasWidth
    const screenY = (1 - (ndcY * 0.5 + 0.5)) * canvasHeight  // Flip Y

    // Apply position
    handle.element.style.display = ''
    handle.element.style.transform = `translate(${screenX}px, ${screenY}px)${
      handle.options.center ? ' translate(-50%, -50%)' : ''
    }`

    // Depth-based z-index
    if (handle.options.zIndexRange) {
      const [min, max] = handle.options.zIndexRange
      handle.element.style.zIndex = String(Math.round(max - ndcZ * (max - min)))
    }

    // Distance-based scaling
    if (handle.options.distanceFactor) {
      const dist = clipW  // W is approximately the distance in view space
      const scale = handle.options.distanceFactor / dist
      handle.element.style.transform += ` scale(${scale})`
    }

    handle.visible = true
  }
}
```

### Occlusion (Optional)

When `occlude: true`, the overlay checks whether the 3D point is behind opaque geometry by reading the depth buffer at the projected screen coordinate:

```ts
// Only works with WebGL2 (readPixels from depth) or WebGPU (buffer readback)
const isOccluded = (ndcZ: number, screenX: number, screenY: number, depthBuffer: Float32Array): boolean => {
  const depthAtPixel = sampleDepthBuffer(depthBuffer, screenX, screenY)
  return ndcZ > depthAtPixel + 0.001  // Small bias
}
```

Depth readback is expensive, so occlusion is opt-in and best limited to a small number of overlays.

### CSS Setup

```ts
const createOverlay = (canvas: HTMLCanvasElement): HtmlOverlay => {
  const container = document.createElement('div')
  container.style.cssText = `
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    overflow: hidden;
  `
  canvas.parentElement!.style.position = 'relative'
  canvas.parentElement!.appendChild(container)

  return {
    container,
    handles: [],
    add: (element, options) => {
      element.style.position = 'absolute'
      element.style.left = '0'
      element.style.top = '0'
      element.style.willChange = 'transform'
      element.style.pointerEvents = 'auto'
      container.appendChild(element)
      const handle = { element, options, visible: true }
      overlay.handles.push(handle)
      return handle
    },
    remove: (handle) => {
      container.removeChild(handle.element)
      overlay.handles.splice(overlay.handles.indexOf(handle), 1)
    },
    update: (camera) => updateOverlays(overlay, camera),
    dispose: () => container.remove(),
  }
}
```

### Usage Example

```ts
// Create a floating label above a character
const label = document.createElement('div')
label.className = 'player-name'
label.textContent = 'Player 1'
label.style.cssText = 'color: white; font-size: 14px; text-shadow: 0 0 4px black;'

overlay.add(label, {
  node: characterMesh,
  center: true,
  distanceFactor: 100,
  zIndexRange: [0, 1000],
})

// Health bar above an enemy, attached to head bone
const healthBar = document.createElement('div')
healthBar.innerHTML = '<div style="width:80%;height:100%;background:red"></div>'
healthBar.style.cssText = 'width:60px;height:6px;background:#333;border-radius:3px;overflow:hidden;'

overlay.add(healthBar, {
  bone: { skeleton: enemySkeleton, boneName: 'head' },
  center: true,
  distanceFactor: 80,
})
```
