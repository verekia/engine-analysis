# Controls — Orbit Controls

## Overview

Fennec ships orbit controls for camera manipulation: orbit (rotate around target), pan (translate target), and zoom (adjust distance). The implementation supports both pointer (mouse) and touch input, with inertia for smooth feel.

## API

```typescript
interface OrbitControlsOptions {
  target?: Vec3               // Point to orbit around (default: [0,0,0])
  distance?: number           // Initial distance from target (default: 10)
  minDistance?: number         // Zoom limit (default: 0.1)
  maxDistance?: number         // Zoom limit (default: 1000)
  minPolarAngle?: number      // Elevation limit in radians (default: 0, straight up along Z)
  maxPolarAngle?: number      // Elevation limit in radians (default: π, straight down)
  enableDamping?: boolean     // Smooth inertia (default: true)
  dampingFactor?: number      // Damping strength (default: 0.1)
  rotateSpeed?: number        // Rotation sensitivity (default: 1.0)
  panSpeed?: number           // Pan sensitivity (default: 1.0)
  zoomSpeed?: number          // Zoom sensitivity (default: 1.0)
  enableRotate?: boolean      // Default: true
  enablePan?: boolean         // Default: true
  enableZoom?: boolean        // Default: true
}

interface OrbitControls {
  readonly camera: Camera
  target: Vec3
  enabled: boolean

  update(deltaTime: number): void
  dispose(): void
}

const createOrbitControls = (camera: Camera, canvas: HTMLCanvasElement, options?: OrbitControlsOptions): OrbitControls
```

## Spherical Coordinate System

The camera position is stored in spherical coordinates relative to the target. Since Fennec is Z-up:

```
Camera position = target + sphericalToCartesian(distance, azimuth, elevation)

Where (Z-up convention):
  azimuth  = rotation around Z axis (0 = looking along +Y)
  elevation = angle from the XY plane toward +Z (0 = level, π/2 = top-down)
```

```typescript
const sphericalToCartesian = (distance: number, azimuth: number, elevation: number): Vec3 => {
  const cosElev = Math.cos(elevation)
  return vec3.fromValues(
    distance * cosElev * Math.sin(azimuth),   // X
    distance * cosElev * Math.cos(azimuth),   // Y (forward when azimuth=0)
    distance * Math.sin(elevation),            // Z (up)
  )
}
```

## Input Handling

```typescript
// Pointer events (unified mouse + touch)
const onPointerDown = (e: PointerEvent) => {
  canvas.setPointerCapture(e.pointerId)
  if (e.button === 0) state.mode = 'rotate'        // Left click → orbit
  else if (e.button === 1) state.mode = 'pan'       // Middle click → pan
  else if (e.button === 2) state.mode = 'pan'       // Right click → pan
  state.prevX = e.clientX
  state.prevY = e.clientY
}

const onPointerMove = (e: PointerEvent) => {
  if (state.mode === 'none') return
  const dx = e.clientX - state.prevX
  const dy = e.clientY - state.prevY
  state.prevX = e.clientX
  state.prevY = e.clientY

  if (state.mode === 'rotate') {
    state.azimuthDelta -= dx * rotateSpeed * 0.005
    state.elevationDelta += dy * rotateSpeed * 0.005
  } else if (state.mode === 'pan') {
    // Pan in screen-aligned plane
    const panX = -dx * panSpeed * state.distance * 0.001
    const panY = dy * panSpeed * state.distance * 0.001
    // Convert screen pan to world-space offset
    const right = camera.getWorldRight()
    const up = camera.getWorldUp()
    vec3.scaleAndAdd(state.panDelta, state.panDelta, right, panX)
    vec3.scaleAndAdd(state.panDelta, state.panDelta, up, panY)
  }
}

const onWheel = (e: WheelEvent) => {
  e.preventDefault()
  state.zoomDelta += e.deltaY * zoomSpeed * 0.001
}
```

## Touch Support

Touch gestures map to the same operations:

| Gesture | Action |
|---------|--------|
| One finger drag | Orbit (rotate) |
| Two finger drag | Pan |
| Pinch | Zoom |

```typescript
const onTouchMove = (e: TouchEvent) => {
  if (e.touches.length === 1) {
    // Single touch → rotate (same as left-click drag)
  } else if (e.touches.length === 2) {
    // Two touches → pan + pinch zoom
    const dx = (e.touches[0].clientX + e.touches[1].clientX) / 2 - state.prevTouchCenter.x
    const dy = (e.touches[0].clientY + e.touches[1].clientY) / 2 - state.prevTouchCenter.y
    // Pan from center movement

    const dist = Math.hypot(
      e.touches[0].clientX - e.touches[1].clientX,
      e.touches[0].clientY - e.touches[1].clientY,
    )
    const prevDist = state.prevPinchDistance
    state.zoomDelta += (prevDist - dist) * zoomSpeed * 0.005
    state.prevPinchDistance = dist
  }
}
```

## Damping Update

```typescript
const updateOrbitControls = (controls: OrbitControls, dt: number) => {
  const state = controls._state
  const damping = controls.dampingFactor

  // Apply deltas with damping
  state.azimuth += state.azimuthDelta
  state.elevation += state.elevationDelta
  vec3.add(controls.target, controls.target, state.panDelta)
  state.distance *= (1 + state.zoomDelta)

  // Clamp
  state.elevation = Math.max(controls.minPolarAngle - Math.PI / 2,
    Math.min(controls.maxPolarAngle - Math.PI / 2, state.elevation))
  state.distance = Math.max(controls.minDistance, Math.min(controls.maxDistance, state.distance))

  // Damping (exponential decay)
  if (controls.enableDamping) {
    state.azimuthDelta *= (1 - damping)
    state.elevationDelta *= (1 - damping)
    vec3.scale(state.panDelta, state.panDelta, 1 - damping)
    state.zoomDelta *= (1 - damping)
  } else {
    state.azimuthDelta = 0
    state.elevationDelta = 0
    vec3.set(state.panDelta, 0, 0, 0)
    state.zoomDelta = 0
  }

  // Compute camera position from spherical coords
  const offset = sphericalToCartesian(state.distance, state.azimuth, state.elevation)
  vec3.add(controls.camera.position, controls.target, offset)
  controls.camera.lookAt(controls.target)
}
```
