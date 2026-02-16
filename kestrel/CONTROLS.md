# Orbit Controls

## Overview

`OrbitControls` provides pointer and touch-driven camera control: orbiting around a target point, panning, and zooming. Supports damping (inertia) for smooth feel.

## Coordinate System

Since Kestrel uses Z-up:
- **Orbit horizontal** (azimuth): rotates around the Z axis
- **Orbit vertical** (elevation): rotates from the XY plane toward +Z
- **Pan**: moves the target point in the camera's local XZ plane (screen-space horizontal/vertical)

## Configuration

```typescript
interface OrbitControlsOptions {
  target?: [number, number, number]    // orbit center — default [0, 0, 0]
  distance?: number                    // initial distance from target — default 10
  minDistance?: number                 // zoom-in limit — default 0.1
  maxDistance?: number                 // zoom-out limit — default Infinity

  // Angle limits (radians)
  minElevation?: number               // min vertical angle — default 0.05 (just above ground)
  maxElevation?: number               // max vertical angle — default π/2 - 0.05 (just below zenith)

  // Azimuth limits (undefined = unlimited rotation)
  minAzimuth?: number                 // default -Infinity (free rotation)
  maxAzimuth?: number                 // default Infinity

  // Behavior
  damping?: number                    // inertia factor — default 0.1 (0 = no damping, 1 = instant)
  rotateSpeed?: number                // orbit sensitivity — default 1.0
  panSpeed?: number                   // pan sensitivity — default 1.0
  zoomSpeed?: number                  // zoom sensitivity — default 1.0

  // Enable/disable
  enableRotate?: boolean              // default true
  enablePan?: boolean                 // default true
  enableZoom?: boolean                // default true
}
```

## State

```typescript
interface OrbitState {
  // Spherical coordinates (Z-up)
  azimuth: number        // horizontal angle around Z axis (radians)
  elevation: number      // vertical angle from XY plane toward Z (radians)
  distance: number       // distance from target

  // Target point
  target: Vec3           // world-space point we orbit around

  // Velocity (for damping)
  azimuthVelocity: number
  elevationVelocity: number
  distanceVelocity: number
  panVelocity: Vec3
}
```

## Spherical to Camera Position

```typescript
const updateCamera = (state: OrbitState, camera: Camera) => {
  // Spherical → Cartesian (Z-up)
  const x = state.distance * Math.cos(state.elevation) * Math.cos(state.azimuth)
  const y = state.distance * Math.cos(state.elevation) * Math.sin(state.azimuth)
  const z = state.distance * Math.sin(state.elevation)

  camera.position.set(
    state.target.x + x,
    state.target.y + y,
    state.target.z + z,
  )

  camera.lookAt(state.target)
}
```

## Input Handling

### Pointer Events

```
Left mouse drag    → Orbit (rotate azimuth + elevation)
Right mouse drag   → Pan
Middle mouse drag  → Pan
Scroll wheel       → Zoom (adjust distance)
```

### Touch Events

```
1-finger drag      → Orbit
2-finger drag      → Pan
2-finger pinch     → Zoom
```

### Implementation

```typescript
const onPointerDown = (event: PointerEvent) => {
  if (event.button === 0) mode = 'rotate'
  else if (event.button === 1 || event.button === 2) mode = 'pan'
  lastX = event.clientX
  lastY = event.clientY
  canvas.setPointerCapture(event.pointerId)
}

const onPointerMove = (event: PointerEvent) => {
  const dx = event.clientX - lastX
  const dy = event.clientY - lastY
  lastX = event.clientX
  lastY = event.clientY

  if (mode === 'rotate') {
    state.azimuthVelocity -= dx * options.rotateSpeed * 0.01
    state.elevationVelocity += dy * options.rotateSpeed * 0.01
  } else if (mode === 'pan') {
    // Pan in camera-local XY plane
    const panX = -dx * options.panSpeed * state.distance * 0.001
    const panY = dy * options.panSpeed * state.distance * 0.001

    // Compute camera's right and up vectors
    const right = pool.v3()
    const up = pool.v3()
    getCameraRightUp(camera, right, up)

    state.panVelocity.x += right[0] * panX + up[0] * panY
    state.panVelocity.y += right[1] * panX + up[1] * panY
    state.panVelocity.z += right[2] * panX + up[2] * panY
  }
}

const onWheel = (event: WheelEvent) => {
  const zoomDelta = event.deltaY * options.zoomSpeed * 0.001
  state.distanceVelocity += state.distance * zoomDelta
}
```

## Update Loop (Damping)

Called every frame:

```typescript
const update = (dt: number) => {
  const d = options.damping

  // Apply velocity to state
  state.azimuth += state.azimuthVelocity
  state.elevation += state.elevationVelocity
  state.distance += state.distanceVelocity
  state.target.x += state.panVelocity.x
  state.target.y += state.panVelocity.y
  state.target.z += state.panVelocity.z

  // Clamp
  state.elevation = clamp(state.elevation, options.minElevation, options.maxElevation)
  state.distance = clamp(state.distance, options.minDistance, options.maxDistance)
  if (options.minAzimuth !== -Infinity) {
    state.azimuth = clamp(state.azimuth, options.minAzimuth, options.maxAzimuth)
  }

  // Damp velocities
  state.azimuthVelocity *= (1 - d)
  state.elevationVelocity *= (1 - d)
  state.distanceVelocity *= (1 - d)
  state.panVelocity.x *= (1 - d)
  state.panVelocity.y *= (1 - d)
  state.panVelocity.z *= (1 - d)

  // Zero out tiny velocities to prevent eternal micro-movement
  if (Math.abs(state.azimuthVelocity) < 1e-6) state.azimuthVelocity = 0
  if (Math.abs(state.elevationVelocity) < 1e-6) state.elevationVelocity = 0
  if (Math.abs(state.distanceVelocity) < 1e-6) state.distanceVelocity = 0

  // Update camera
  updateCamera(state, camera)
}
```

## Usage

### Imperative

```typescript
const controls = new OrbitControls(engine.camera, engine.canvas, {
  target: [0, 0, 0],
  distance: 15,
  damping: 0.1,
  minDistance: 2,
  maxDistance: 100,
  maxElevation: Math.PI / 2 - 0.1,
})

// In the render loop (automatically connected via engine):
engine.onFrame((dt) => controls.update(dt))
```

### React

```tsx
import { OrbitControls } from '@kestrel/react'

<Canvas>
  <OrbitControls target={[0, 0, 0]} damping={0.1} />
  {/* ... scene */}
</Canvas>
```

## Programmatic Camera Control

```typescript
// Animate to a new target/position
controls.setTarget([10, 20, 5], { animate: true, duration: 1.0 })
controls.setDistance(20, { animate: true, duration: 0.5 })
controls.setAzimuth(Math.PI / 4, { animate: true })

// Instant jump
controls.setTarget([0, 0, 0])
controls.setDistance(10)

// Disable interaction temporarily
controls.enabled = false
```

## Disposal

```typescript
controls.dispose()
// Removes all event listeners from the canvas
```
