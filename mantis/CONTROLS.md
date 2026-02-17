# Orbit Controls

## Overview

Mantis provides orbit controls for camera navigation — orbiting around a target
point, panning, and zooming. The controls are Z-up aware: azimuth rotates
around the Z axis, and elevation measures the angle from the XY ground plane
toward the Z axis.

## Coordinate Convention

```
      Z (up)
      │
      │    ╱ elevation (phi)
      │   ╱
      │  ╱
      │ ╱  ← camera at distance d
      │╱
  ────●──── → X
     ╱│
    ╱  │
  Y    │
       │
   azimuth (theta): rotation around Z axis in the XY plane
```

- **Azimuth (θ):** Angle around Z axis, measured from +X toward +Y. Range:
  unrestricted (wraps) or clamped.
- **Elevation (φ):** Angle from XY plane toward +Z. Range: [-π/2 + ε, π/2 - ε]
  (prevents gimbal lock at poles).
- **Distance (d):** Distance from target to camera.

Camera position from spherical coordinates:
```
x = target.x + d × cos(φ) × cos(θ)
y = target.y + d × cos(φ) × sin(θ)
z = target.z + d × sin(φ)
```

## Input Mapping

| Input | Action |
|---|---|
| Left mouse drag / 1-finger touch drag | Orbit (rotate azimuth + elevation) |
| Right mouse drag / 2-finger touch drag | Pan (move target in screen plane) |
| Scroll wheel / pinch gesture | Zoom (change distance) |
| Middle mouse drag | Pan (alternative) |

## Configuration

```typescript
interface OrbitControlsOptions {
  target?: Vec3              // default [0, 0, 0]
  distance?: number          // default 10
  azimuth?: number           // default 0 (radians)
  elevation?: number         // default π/6 (30°)

  // Limits
  minDistance?: number        // default 1
  maxDistance?: number        // default 100
  minElevation?: number      // default -π/2 + 0.01 (just above -Z)
  maxElevation?: number      // default π/2 - 0.01 (just below +Z)
  minAzimuth?: number        // default -Infinity (unrestricted)
  maxAzimuth?: number        // default Infinity (unrestricted)

  // Sensitivity
  orbitSpeed?: number         // default 1.0 (radians per 300px drag)
  panSpeed?: number           // default 1.0
  zoomSpeed?: number          // default 1.0

  // Damping (inertia)
  enableDamping?: boolean     // default true
  dampingFactor?: number      // default 0.08 (0 = no inertia, 1 = no damping)

  // Touch
  enableTouch?: boolean       // default true
}
```

## Implementation

### State

```typescript
interface OrbitControlsState {
  azimuth: number
  elevation: number
  distance: number
  target: Vec3

  // Velocities (for damping)
  azimuthVelocity: number
  elevationVelocity: number
  distanceVelocity: number
  panVelocityX: number
  panVelocityY: number

  // Interaction tracking
  isDragging: boolean
  dragButton: number        // 0 = left, 2 = right
  lastPointerX: number
  lastPointerY: number
  lastPinchDistance: number
}
```

### Update Loop

Called each frame before scene graph update:

```typescript
const updateOrbitControls = (controls: OrbitControls, camera: Camera, dt: number) => {
  const { state, options } = controls

  // Apply damping to velocities
  if (options.enableDamping && !state.isDragging) {
    const decay = 1 - options.dampingFactor

    state.azimuthVelocity *= decay
    state.elevationVelocity *= decay
    state.distanceVelocity *= decay
    state.panVelocityX *= decay
    state.panVelocityY *= decay

    // Apply damped velocities
    state.azimuth += state.azimuthVelocity
    state.elevation += state.elevationVelocity
    state.distance += state.distanceVelocity

    // Apply pan velocity in screen space
    applyPan(controls, state.panVelocityX, state.panVelocityY)

    // Kill near-zero velocities to avoid infinite drift
    if (Math.abs(state.azimuthVelocity) < 1e-5) state.azimuthVelocity = 0
    if (Math.abs(state.elevationVelocity) < 1e-5) state.elevationVelocity = 0
    if (Math.abs(state.distanceVelocity) < 1e-5) state.distanceVelocity = 0
    if (Math.abs(state.panVelocityX) < 1e-5) state.panVelocityX = 0
    if (Math.abs(state.panVelocityY) < 1e-5) state.panVelocityY = 0
  }

  // Clamp elevation
  state.elevation = Math.max(options.minElevation, Math.min(options.maxElevation, state.elevation))

  // Clamp azimuth (if restricted)
  if (isFinite(options.minAzimuth)) {
    state.azimuth = Math.max(options.minAzimuth, Math.min(options.maxAzimuth, state.azimuth))
  }

  // Clamp distance
  state.distance = Math.max(options.minDistance, Math.min(options.maxDistance, state.distance))

  // Compute camera position from spherical coordinates
  const cosElev = Math.cos(state.elevation)
  const sinElev = Math.sin(state.elevation)
  const cosAzim = Math.cos(state.azimuth)
  const sinAzim = Math.sin(state.azimuth)

  camera.position = [
    state.target[0] + state.distance * cosElev * cosAzim,
    state.target[1] + state.distance * cosElev * sinAzim,
    state.target[2] + state.distance * sinElev,
  ]

  // Look at target (Z-up)
  mat4LookAt(camera.viewMatrix, camera.position, state.target, VEC3_UP)
}
```

### Event Handlers

```typescript
const onPointerDown = (controls: OrbitControls, e: PointerEvent) => {
  controls.state.isDragging = true
  controls.state.dragButton = e.button
  controls.state.lastPointerX = e.clientX
  controls.state.lastPointerY = e.clientY
  controls.canvas.setPointerCapture(e.pointerId)
}

const onPointerMove = (controls: OrbitControls, e: PointerEvent) => {
  if (!controls.state.isDragging) return

  const dx = e.clientX - controls.state.lastPointerX
  const dy = e.clientY - controls.state.lastPointerY
  controls.state.lastPointerX = e.clientX
  controls.state.lastPointerY = e.clientY

  if (controls.state.dragButton === 0) {
    // Left button: orbit
    const orbitScale = controls.options.orbitSpeed / 300
    controls.state.azimuthVelocity = -dx * orbitScale
    controls.state.elevationVelocity = dy * orbitScale

    controls.state.azimuth += controls.state.azimuthVelocity
    controls.state.elevation += controls.state.elevationVelocity
  } else if (controls.state.dragButton === 2) {
    // Right button: pan
    const panScale = controls.options.panSpeed * controls.state.distance / 500
    controls.state.panVelocityX = -dx * panScale
    controls.state.panVelocityY = dy * panScale

    applyPan(controls, controls.state.panVelocityX, controls.state.panVelocityY)
  }
}

const onWheel = (controls: OrbitControls, e: WheelEvent) => {
  e.preventDefault()
  const zoomScale = controls.options.zoomSpeed * 0.001
  const delta = e.deltaY * zoomScale * controls.state.distance
  controls.state.distanceVelocity = delta
  controls.state.distance += delta
}
```

### Pan in Screen Space

Pan moves the target in the camera's screen-space plane:

```typescript
const applyPan = (controls: OrbitControls, dx: number, dy: number) => {
  const { azimuth, elevation } = controls.state
  const target = controls.state.target

  // Camera right vector (screen X direction)
  const rightX = -Math.sin(azimuth)
  const rightY = Math.cos(azimuth)

  // Camera up vector in screen space (not world up — screen vertical)
  const upX = -Math.cos(azimuth) * Math.sin(elevation)
  const upY = -Math.sin(azimuth) * Math.sin(elevation)
  const upZ = Math.cos(elevation)

  target[0] += rightX * dx + upX * dy
  target[1] += rightY * dx + upY * dy
  target[2] += upZ * dy
}
```

## Touch Support

Touch events are mapped to the same orbit/pan/zoom actions:

| Gesture | Action |
|---|---|
| 1-finger drag | Orbit |
| 2-finger drag | Pan |
| Pinch | Zoom |

```typescript
const onTouchStart = (controls: OrbitControls, e: TouchEvent) => {
  if (e.touches.length === 1) {
    controls.state.isDragging = true
    controls.state.dragButton = 0  // orbit
    controls.state.lastPointerX = e.touches[0].clientX
    controls.state.lastPointerY = e.touches[0].clientY
  } else if (e.touches.length === 2) {
    controls.state.isDragging = true
    controls.state.dragButton = 2  // pan
    controls.state.lastPinchDistance = touchDistance(e.touches[0], e.touches[1])
    const mid = touchMidpoint(e.touches[0], e.touches[1])
    controls.state.lastPointerX = mid.x
    controls.state.lastPointerY = mid.y
  }
}
```

## Programmatic Control

```typescript
const controls = new OrbitControls(engine, camera, {
  target: [0, 0, 0],
  distance: 20,
  azimuth: Math.PI / 4,
  elevation: Math.PI / 6,
})

// Animate to a new view
controls.setTarget([10, 5, 0], { animate: true, duration: 0.5 })
controls.setDistance(30, { animate: true, duration: 0.5 })
controls.setAzimuth(Math.PI / 2, { animate: true, duration: 0.5 })
controls.setElevation(Math.PI / 4, { animate: true, duration: 0.5 })

// Instant move (no animation)
controls.setTarget([0, 0, 0])
controls.setDistance(10)

// Disable controls temporarily
controls.enabled = false

// Dispose (removes event listeners)
controls.dispose()
```

### Animated Transitions

```typescript
const setTarget = (controls: OrbitControls, newTarget: Vec3, opts?: { animate?: boolean, duration?: number }) => {
  if (opts?.animate) {
    controls.animation = {
      property: 'target',
      from: vec3Copy(vec3Create(), controls.state.target),
      to: vec3Copy(vec3Create(), newTarget),
      duration: opts.duration ?? 0.3,
      elapsed: 0,
      easing: easeOutCubic,
    }
  } else {
    vec3Copy(controls.state.target, newTarget)
  }
}

// In update loop:
if (controls.animation) {
  controls.animation.elapsed += dt
  const t = Math.min(controls.animation.elapsed / controls.animation.duration, 1)
  const eased = controls.animation.easing(t)

  if (controls.animation.property === 'target') {
    vec3Lerp(controls.state.target, controls.animation.from, controls.animation.to, eased)
  }
  // ... similar for distance, azimuth, elevation

  if (t >= 1) controls.animation = null
}
```

## React Integration

```tsx
import { OrbitControls } from 'mantis/react'

const Scene = () => (
  <>
    <OrbitControls
      target={[0, 0, 0]}
      distance={20}
      enableDamping
      dampingFactor={0.08}
      minDistance={5}
      maxDistance={50}
    />
    {/* ... scene content */}
  </>
)
```

The `<OrbitControls>` React component creates an `OrbitControls` instance and
registers it with the engine. It updates when props change and disposes on
unmount.
