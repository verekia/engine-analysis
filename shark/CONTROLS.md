# Orbit Controls Architecture

## Overview

Orbit controls let the user rotate, pan, and zoom a camera around a target point using mouse or touch input. The implementation is adapted for Shark's Z-up coordinate system.

## Parameters

```typescript
interface OrbitControlsConfig {
  // Target
  target?: Vec3                 // Orbit center (default: [0, 0, 0])

  // Rotation
  enableRotate?: boolean        // Allow rotation (default: true)
  rotateSpeed?: number          // Rotation sensitivity (default: 1.0)
  minPolarAngle?: number        // Min angle from Z-up axis in radians (default: 0.01)
  maxPolarAngle?: number        // Max angle from Z-up axis in radians (default: PI - 0.01)
  minAzimuthAngle?: number      // Min horizontal angle (default: -Infinity)
  maxAzimuthAngle?: number      // Max horizontal angle (default: Infinity)

  // Zoom
  enableZoom?: boolean          // Allow zoom (default: true)
  zoomSpeed?: number            // Zoom sensitivity (default: 1.0)
  minDistance?: number           // Minimum distance to target (default: 0.1)
  maxDistance?: number           // Maximum distance to target (default: Infinity)

  // Pan
  enablePan?: boolean           // Allow panning (default: true)
  panSpeed?: number             // Pan sensitivity (default: 1.0)

  // Damping
  enableDamping?: boolean       // Smooth camera movement (default: true)
  dampingFactor?: number        // Damping smoothness 0..1 (default: 0.1)

  // Auto-rotate
  autoRotate?: boolean          // Slowly rotate around target (default: false)
  autoRotateSpeed?: number      // Rotation speed in rad/s (default: 0.5)
}
```

## Spherical Coordinates (Z-up)

Standard orbit controls use spherical coordinates. Since Shark uses Z-up, the mapping is:

```
x = r * sin(polar) * cos(azimuth)
y = r * sin(polar) * sin(azimuth)
z = r * cos(polar)

Where:
  r       = distance from target
  polar   = angle from +Z axis (0 = directly above, PI = directly below)
  azimuth = angle around Z axis in XY plane (0 = +X direction)
```

This differs from Y-up orbit controls where polar is measured from +Y.

```typescript
interface SphericalCoords {
  radius: number      // Distance to target
  polar: number       // Angle from Z axis [minPolar, maxPolar]
  azimuth: number     // Angle around Z axis [minAzimuth, maxAzimuth]
}

const sphericalToCartesian = (s: SphericalCoords, out: Vec3): Vec3 => {
  const sinPolar = Math.sin(s.polar)
  out[0] = s.radius * sinPolar * Math.cos(s.azimuth)
  out[1] = s.radius * sinPolar * Math.sin(s.azimuth)
  out[2] = s.radius * Math.cos(s.polar)
  return out
}

const cartesianToSpherical = (v: Vec3, out: SphericalCoords): SphericalCoords => {
  out.radius = Math.sqrt(v[0] * v[0] + v[1] * v[1] + v[2] * v[2])
  out.polar = Math.acos(Math.max(-1, Math.min(1, v[2] / out.radius)))
  out.azimuth = Math.atan2(v[1], v[0])
  return out
}
```

## Input Handling

### Mouse Events

| Input | Action |
|-------|--------|
| Left button drag | Rotate (azimuth + polar) |
| Right button drag | Pan (XY plane at target depth) |
| Scroll wheel | Zoom (change radius) |
| Middle button drag | Zoom (alternative) |

### Touch Events

| Input | Action |
|-------|--------|
| 1-finger drag | Rotate |
| 2-finger pinch | Zoom |
| 2-finger drag | Pan |

### Input Event Processing

```typescript
const onPointerDown = (e: PointerEvent) => {
  if (e.button === 0) state = 'rotate'
  else if (e.button === 2) state = 'pan'
  prevPointer = [e.clientX, e.clientY]
  canvas.setPointerCapture(e.pointerId)
}

const onPointerMove = (e: PointerEvent) => {
  const dx = e.clientX - prevPointer[0]
  const dy = e.clientY - prevPointer[1]
  prevPointer = [e.clientX, e.clientY]

  if (state === 'rotate') {
    // Accumulate rotation delta (applied during update with damping)
    azimuthDelta -= dx * rotateSpeed * 0.01
    polarDelta -= dy * rotateSpeed * 0.01
  } else if (state === 'pan') {
    const distance = spherical.radius
    const panX = dx * panSpeed * distance * 0.002
    const panY = dy * panSpeed * distance * 0.002
    panDelta[0] -= panX
    panDelta[1] -= panY
  }
}

const onWheel = (e: WheelEvent) => {
  e.preventDefault()
  if (e.deltaY > 0) zoomDelta *= 1.0 + zoomSpeed * 0.1
  else zoomDelta /= 1.0 + zoomSpeed * 0.1
}
```

## Update Loop

The controls update each frame, applying damping to accumulated deltas:

```typescript
const update = (controls: OrbitControls, camera: Camera) => {
  const { spherical, target, config } = controls

  // Apply rotation deltas
  spherical.azimuth += controls.azimuthDelta
  spherical.polar += controls.polarDelta

  // Clamp polar angle (prevent flipping at poles)
  spherical.polar = Math.max(
    config.minPolarAngle,
    Math.min(config.maxPolarAngle, spherical.polar),
  )

  // Clamp azimuth angle (if constrained)
  if (isFinite(config.minAzimuthAngle) && isFinite(config.maxAzimuthAngle)) {
    spherical.azimuth = Math.max(
      config.minAzimuthAngle,
      Math.min(config.maxAzimuthAngle, spherical.azimuth),
    )
  }

  // Apply zoom
  spherical.radius *= controls.zoomDelta
  spherical.radius = Math.max(config.minDistance, Math.min(config.maxDistance, spherical.radius))

  // Apply pan (in camera's local frame)
  const panOffset = computePanOffset(controls.panDelta, camera, spherical)
  vec3Add(target, target, panOffset)

  // Auto-rotate
  if (config.autoRotate) {
    spherical.azimuth += config.autoRotateSpeed * (1 / 60)
  }

  // Compute camera position from spherical coords
  const offset = vec3Create()
  sphericalToCartesian(spherical, offset)
  vec3Add(camera.position, target, offset)
  camera.lookAt(target)

  // Apply damping (decay deltas toward zero)
  if (config.enableDamping) {
    controls.azimuthDelta *= 1 - config.dampingFactor
    controls.polarDelta *= 1 - config.dampingFactor
    controls.zoomDelta = 1 + (controls.zoomDelta - 1) * (1 - config.dampingFactor)
    vec3Scale(controls.panDelta, controls.panDelta, 1 - config.dampingFactor)
  } else {
    controls.azimuthDelta = 0
    controls.polarDelta = 0
    controls.zoomDelta = 1
    vec3Set(controls.panDelta, 0, 0, 0)
  }
}
```

## Pan Computation

Pan moves the target in the camera's local XY plane:

```typescript
const computePanOffset = (panDelta: Vec2, camera: Camera, spherical: SphericalCoords): Vec3 => {
  // Camera's right vector (local X)
  const right: Vec3 = [camera.viewMatrix[0], camera.viewMatrix[4], camera.viewMatrix[8]]

  // Camera's up vector (local Y)
  const up: Vec3 = [camera.viewMatrix[1], camera.viewMatrix[5], camera.viewMatrix[9]]

  // Pan in world space
  const offset = vec3Create()
  vec3ScaleAndAdd(offset, offset, right, panDelta[0])
  vec3ScaleAndAdd(offset, offset, up, -panDelta[1]) // Negate Y for screen coords
  return offset
}
```

## Lifecycle

```typescript
// Create
const controls = new OrbitControls(camera, canvas, {
  target: [0, 0, 0],
  enableDamping: true,
  dampingFactor: 0.1,
  minDistance: 1,
  maxDistance: 100,
})

// Update each frame
controls.update()

// Cleanup
controls.dispose()
```

## React Integration

```tsx
import { OrbitControls } from '@shark3d/react'

const Scene = () => (
  <Canvas>
    <OrbitControls target={[0, 0, 0]} minDistance={1} maxDistance={50} enableDamping />
    {/* scene objects */}
  </Canvas>
)
```

The React component attaches to the default camera and canvas from context, calls `update()` via `useFrame`, and `dispose()` on unmount.
