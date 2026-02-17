# Orbit Controls

Camera controls for orbiting around a target point, with zoom and pan. Similar to three.js's `OrbitControls` but built for Wren's z-up coordinate system.

## State

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

## Input Handling

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

## Update Loop

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
