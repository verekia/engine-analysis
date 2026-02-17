# CONTROLS — Final Decision

## Primary Interaction

**Decision: Orbit controls as the only camera interaction** (universal agreement)

Three operations:
- **Orbit**: rotate camera around a target point
- **Pan**: move the target point in the camera's screen plane
- **Zoom**: change distance from target (dolly)

## Angle Convention

**Decision: Elevation angle from XY plane** (universal agreement)

- Elevation 0 = horizontal (looking at horizon)
- Elevation +π/2 = looking straight down
- Elevation -π/2 = looking straight up
- Azimuth increases counter-clockwise when viewed from above

This is more intuitive for game developers than the polar-from-Z-axis convention: positive elevation means "looking down," which matches the mental model.

### Spherical to Cartesian (Z-up)

```typescript
const x = target[0] + distance * Math.cos(elevation) * Math.cos(azimuth)
const y = target[1] + distance * Math.cos(elevation) * Math.sin(azimuth)
const z = target[2] + distance * Math.sin(elevation)
```

## Damping

**Decision: Velocity-based damping with exponential decay** (universal agreement)

```typescript
const dampingFactor = 0.1  // Default

// Each frame:
velocity *= Math.pow(1 - dampingFactor, deltaTime * 60)
azimuth += velocity.azimuth * deltaTime
elevation += velocity.elevation * deltaTime
distance += velocity.distance * deltaTime
```

Damping factor of 0.1 provides smooth deceleration without feeling sluggish. The `deltaTime * 60` normalization ensures consistent feel regardless of frame rate.

## Input Mapping

### Mouse

| Input | Action |
|-------|--------|
| Left drag | Orbit (rotate around target) |
| Right drag / Middle drag | Pan (move target) |
| Scroll wheel | Zoom (change distance) |

### Touch

| Input | Action |
|-------|--------|
| 1-finger drag | Orbit |
| 2-finger drag | Pan |
| Pinch | Zoom |

Touch gesture recognition uses pointer count to distinguish orbit from pan/zoom.

## Pan Implementation

**Decision: Screen-space pan scaled by distance** (universal agreement)

Pan delta is computed in screen space and projected into the camera's local right/up vectors, scaled by distance from target:

```typescript
const panSpeed = distance * Math.tan(fov / 2) * 2 / canvasHeight
const right = getCameraRight(camera)
const up = getCameraUp(camera)
target[0] += (right[0] * -dx + up[0] * dy) * panSpeed
target[1] += (right[1] * -dx + up[1] * dy) * panSpeed
target[2] += (right[2] * -dx + up[2] * dy) * panSpeed
```

Scaling by distance ensures pan speed feels consistent regardless of zoom level.

## Constraints

```typescript
const controls = createOrbitControls(camera, canvas, {
  target: [0, 0, 0],
  minDistance: 0.1,              // Minimum zoom distance
  maxDistance: 1000,             // Maximum zoom distance
  minElevation: -Math.PI / 2,   // How far down (default: no limit)
  maxElevation: Math.PI / 2,    // How far up (default: no limit)
  dampingFactor: 0.1,
  enabled: true,
})
```

Elevation is clamped to prevent gimbal lock at the poles (a small epsilon buffer prevents reaching exactly ±π/2).

Azimuth is unrestricted by default (full 360° rotation). Optional `minAzimuth`/`maxAzimuth` for restricted rotation.

## API

**Decision: Factory function** (consistent with rest of API)

```typescript
const controls = createOrbitControls(camera, canvas, {
  target: [0, 0, 0],
  dampingFactor: 0.1,
  minDistance: 1,
  maxDistance: 100,
})

// Per-frame update
controls.update(deltaTime)

// Programmatic control
controls.setTarget([10, 0, 5], { animate: true, duration: 0.5 })
controls.setPosition([20, -15, 10], { animate: true, duration: 0.5 })

// State
controls.enabled = false  // Disable interaction
controls.target           // Current target position (readonly Vec3)

// Events
controls.onChange(() => { /* camera moved */ })
controls.onStart(() => { /* interaction started */ })
controls.onEnd(() => { /* interaction ended */ })

// Cleanup
controls.dispose()  // Removes event listeners
```

## Animated Transitions

`setTarget()` and `setPosition()` with `{ animate: true }` smoothly interpolate from the current state to the new state over the specified duration using ease-out interpolation.

## React Integration

```tsx
<OrbitControls target={[0, 0, 0]} minDistance={1} maxDistance={100} />
```

The React component creates controls on mount, updates props reactively, and disposes on unmount.

## Event Listener Cleanup

`controls.dispose()` removes all mouse, touch, and wheel event listeners from the canvas. Essential to prevent memory leaks when controls are created/destroyed dynamically.
