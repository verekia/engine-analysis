# CONTROLS.md - Decisions

## Decision: Elevation Angle Convention, Velocity-Based Damping, Factory Function API

### Core Functionality: Orbit Controls

Universal agreement: orbit controls as the primary camera interaction mode.

Three core operations:
- **Orbit**: Rotate camera around target point
- **Pan**: Move target point in screen-aligned plane
- **Zoom**: Change distance from target

### Input Mapping

Universal agreement across all implementations:

| Input | Action |
|-------|--------|
| Left mouse drag / 1-finger touch | Orbit (rotate) |
| Right mouse drag / 2-finger drag | Pan (move target) |
| Mouse wheel / pinch gesture | Zoom (change distance) |
| Middle mouse drag | Pan (alternative) |

### Angle Convention: Elevation from XY Plane

**Chosen**: Elevation angle measured from XY plane toward +Z (5/9: Fennec, Hyena, Mantis, Rabbit, Wren)
**Rejected**: Polar angle from Z-axis (2/9: Lynx, Shark) - mathematically conventional but less intuitive

Elevation is more intuitive for game developers: positive = looking up, negative = looking down.

Range: `[-PI/2 + epsilon, PI/2 - epsilon]` to prevent gimbal lock at poles.

### Spherical to Cartesian (Z-up)

```typescript
x = target.x + distance * Math.cos(elevation) * Math.cos(azimuth)
y = target.y + distance * Math.cos(elevation) * Math.sin(azimuth)
z = target.z + distance * Math.sin(elevation)
```

### Damping: Velocity-Based

**Chosen**: Velocity-based with exponential decay (4/9: Fennec, Hyena, Mantis, Wren)
**Rejected**: Delta-based damping (3/9: Lynx, Rabbit, Shark) - simpler but lacks momentum feel

```typescript
// During input: accumulate velocity from mouse delta
azimuthVelocity += mouseDeltaX * rotateSpeed

// Every frame (including after input stops):
azimuth += azimuthVelocity
azimuthVelocity *= (1 - dampingFactor)
```

Velocity-based damping provides a physical momentum feel - the camera continues to drift after releasing the mouse, which feels polished and responsive.

### Damping Factor: 0.1

**Chosen**: 0.1 (5/9: Fennec, Hyena, Rabbit, Shark, Wren)

0.08-0.1 is the consensus sweet spot for responsive but smooth controls.

### Distance Limits

```typescript
{
  minDistance: 0.1,     // Prevents camera from reaching target
  maxDistance: 1000,    // Prevents camera from going too far
}
```

### Azimuth: Unrestricted by Default

**Chosen**: Allow unlimited horizontal rotation (6/9: Fennec, Hyena, Lynx, Mantis, Rabbit, Wren)

Optional `minAzimuth` / `maxAzimuth` for constrained scenarios (e.g., turntable product viewer).

### Pan Implementation

Screen-space pan using camera right and up vectors (universal agreement):

```typescript
const right = camera.getRightVector()
const up = camera.getUpVector()
const panScale = distance * panSpeed * pixelDelta
target += right * panDeltaX * panScale + up * panDeltaY * panScale
```

Pan speed scales with distance from target so screen motion feels consistent regardless of zoom level.

### API: Factory Function

**Chosen**: Factory function (3/9: Rabbit, Shark, Wren)
**Also acceptable**: Class-based (4/9: Fennec, Hyena, Lynx, Mantis)

Factory function is consistent with the engine's overall API style (see API.md):

```typescript
const controls = createOrbitControls(camera, canvas, {
  target: [0, 0, 0],
  minDistance: 1,
  maxDistance: 100,
  dampingFactor: 0.1,
  rotateSpeed: 1.0,
  panSpeed: 1.0,
  zoomSpeed: 1.0,
  enableDamping: true,
  enabled: true,
})

// Per frame
controls.update(deltaTime)

// Cleanup
controls.dispose()
```

### Programmatic Camera Control

**Chosen**: Support animated transitions (4/9: Hyena, Mantis, Rabbit, Shark)

```typescript
controls.setTarget([x, y, z], { animate: true, duration: 0.5 })
controls.setDistance(10, { animate: true, duration: 0.5 })
```

Smooth interpolation between current and target camera state, useful for "focus on object" interactions.

### React Integration

```tsx
<OrbitControls
  target={[0, 0, 0]}
  minDistance={1}
  maxDistance={100}
  enableDamping
/>
```

Props map to controls options. `useRef` available for imperative access.

### Touch Support

- **1-finger**: Orbit (rotation)
- **2-finger**: Pan (translation) and pinch zoom (distance)
- Pinch distance tracked between `touchstart` and `touchmove`

### Enabled/Disabled State

`controls.enabled: boolean` - when false, all input is ignored. Useful for modal UI overlays.

### Event Listeners

Attach to the canvas element, clean up on `dispose()`. Use `{ passive: false }` for wheel events to allow `preventDefault()`.
