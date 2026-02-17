# CONTROLS - Design Decisions

## Primary Control Type

**Decision: Orbit controls as the primary camera interaction (universal agreement)**

Three core operations:
- **Orbit**: Rotate camera around target point
- **Pan**: Translate target point in screen-aligned plane
- **Zoom**: Change distance from target (dolly, not FOV zoom)

## Input Mapping

**Decision: Standard mouse/touch mapping (universal agreement)**

| Input | Action |
|-------|--------|
| Left mouse drag / 1-finger touch | Orbit (rotate) |
| Right mouse drag / 2-finger drag | Pan (translate) |
| Mouse wheel / pinch gesture | Zoom (distance) |
| Middle mouse drag | Pan (alternative) |

## Angle Convention

**Decision: Elevation angle from XY plane (intuitive for game developers)**

- Sources: Fennec, Hyena, Mantis, Rabbit, Wren (5/9)
- Rejected: Polar angle from Z-axis (Lynx, Shark — mathematically conventional but less intuitive)

```typescript
// Elevation: 0 = horizontal, +π/2 = looking up, -π/2 = looking down
// Azimuth: angle around Z axis (0 = +X direction)

x = distance * cos(elevation) * cos(azimuth)
y = distance * cos(elevation) * sin(azimuth)
z = distance * sin(elevation)
```

Positive elevation = looking up is more intuitive for game developers than counting down from the pole.

## Damping

**Decision: Velocity-based damping with exponential decay**

- Sources: Fennec, Hyena, Mantis, Wren (4/9)
- Rejected: Delta-based damping (Lynx, Rabbit, Shark) — simpler but less physically realistic

```typescript
// Store velocities for each axis
azimuthVelocity *= (1 - dampingFactor)
elevationVelocity *= (1 - dampingFactor)
distanceVelocity *= (1 - dampingFactor)

azimuth += azimuthVelocity
elevation += elevationVelocity
distance += distanceVelocity
```

Velocity-based damping allows momentum to continue after input stops, providing a smooth, physics-like feel. Better for orbit controls where users expect the camera to "coast" after a drag gesture.

## Damping Factor

**Decision: 0.1 (most popular default)**

- Sources: Fennec, Hyena, Rabbit, Shark, Wren (5/9)
- Range: 0.0 (no damping) to 1.0 (instant stop)
- 0.1 provides responsive but smooth camera motion

## Elevation Limits

**Decision: Clamp to [-π/2 + ε, π/2 - ε] to prevent gimbal lock**

```typescript
const minElevation = -Math.PI / 2 + 0.01  // slightly above looking straight down
const maxElevation = Math.PI / 2 - 0.01   // slightly below looking straight up
```

Prevents the camera from flipping when passing through poles.

## Distance Limits

**Decision: Configurable min/max with sensible defaults**

```typescript
minDistance: 0.1,    // prevent clipping into target
maxDistance: 1000,   // prevent losing sight of scene
```

## API Style

**Decision: Factory function**

- Sources: Rabbit, Shark, Wren (3/9 — aligns with overall API philosophy of factory functions)

```typescript
const controls = createOrbitControls(camera, canvas, {
  target: [0, 0, 0],
  distance: 10,
  azimuth: 0,
  elevation: Math.PI / 6,
  minDistance: 0.1,
  maxDistance: 1000,
  minElevation: -Math.PI / 2 + 0.01,
  maxElevation: Math.PI / 2 - 0.01,
  dampingFactor: 0.1,
  rotateSpeed: 1.0,
  panSpeed: 1.0,
  zoomSpeed: 1.0,
  enabled: true,
})

// Per-frame update
controls.update(deltaTime)

// Cleanup
controls.dispose()
```

## Pan Implementation

**Decision: Screen-space pan scaled by distance (universal agreement)**

```typescript
// Pan in screen-aligned plane
const right = camera.getRightVector()
const up = camera.getUpVector()
const panScale = distance * panSpeed * pixelDelta / canvasHeight

target[0] += right[0] * panX * panScale + up[0] * panY * panScale
target[1] += right[1] * panX * panScale + up[1] * panY * panScale
target[2] += right[2] * panX * panScale + up[2] * panY * panScale
```

Pan speed scales with distance from target so that screen-space motion feels consistent regardless of zoom level.

## Touch Support

**Decision: Full multi-touch with gesture recognition**

- Sources: All specified implementations agree on touch support

```typescript
// Touch state machine:
// 1 finger: orbit
// 2 fingers: pan + pinch-zoom simultaneously
// Track finger IDs to handle finger add/remove during gesture
```

Pinch distance computed from two-finger separation. Simultaneous pan + zoom from two-finger center movement + separation change.

## Programmatic Control

**Decision: Animated transitions via setTarget/setPosition**

- Sources: Hyena, Mantis, Rabbit, Shark (4/9)

```typescript
controls.setTarget([10, 0, 5], { animate: true, duration: 0.5 })
controls.setDistance(20, { animate: true, duration: 0.5 })
controls.setAzimuth(Math.PI / 4, { animate: true, duration: 0.5 })
```

Smooth interpolation (ease-out curve) for programmatic camera moves. Non-animated calls update instantly.

## React Integration

```tsx
<OrbitControls
  target={[0, 0, 0]}
  minDistance={1}
  maxDistance={100}
  enableDamping
  dampingFactor={0.1}
/>
```

Refs available for imperative control:
```tsx
const controlsRef = useRef()
controlsRef.current?.setTarget([10, 0, 5], { animate: true })
```

## Event Callbacks

```typescript
controls.onChange = () => { /* camera moved */ }
controls.onStart = () => { /* interaction started */ }
controls.onEnd = () => { /* interaction ended */ }
```

## Future Controls (out of scope for v1)

- First-person controls (WASD + mouse look)
- Fly controls (6DOF)
- Third-person follow camera
- Keyboard navigation for accessibility
