# Controls

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
