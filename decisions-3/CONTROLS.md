# CONTROLS.md - Camera Controls

## Orbit Controls

**Decision: Class-based OrbitControls with elevation angle convention** (Fennec, Hyena, Mantis approach)

### Core Operations (Universal Agreement)

- **Orbit**: Rotate camera around target point (left mouse / 1-finger touch)
- **Pan**: Move target point in screen-aligned plane (right mouse / 2-finger drag)
- **Zoom**: Change distance from target (mouse wheel / pinch gesture)

### Angle Convention

**Decision: Elevation angle from XY plane** (Fennec, Hyena, Mantis, Rabbit, Wren - 5/8 prefer this)

```
elevation > 0  = looking down from above (camera above target)
elevation < 0  = looking up from below (camera below target)
elevation = 0  = camera at same height as target
```

**Why elevation over polar angle (Lynx, Shark)?**

Elevation is more intuitive for game developers: "positive = higher" makes immediate sense. Polar angle (measured from +Z axis down) requires mental inversion. The math is equivalent; it is purely a naming/convention question, and the majority choice is elevation.

### Spherical to Cartesian (Z-up)

```typescript
const cosElev = Math.cos(elevation)
camera.position[0] = target[0] + distance * cosElev * Math.cos(azimuth)
camera.position[1] = target[1] + distance * cosElev * Math.sin(azimuth)
camera.position[2] = target[2] + distance * Math.sin(elevation)
```

### Damping

**Decision: Velocity-based damping** (Fennec, Hyena, Mantis, Wren approach)

```typescript
azimuthVelocity *= (1 - dampingFactor)
elevationVelocity *= (1 - dampingFactor)
zoomVelocity *= (1 - dampingFactor)

azimuth += azimuthVelocity
elevation += elevationVelocity
distance += zoomVelocity
```

Velocity-based damping provides momentum: the camera continues gliding after the user releases the mouse. This feels more physically realistic than delta-based damping (Lynx, Rabbit, Shark) which stops more abruptly.

**Default damping factor**: `0.1` (Fennec, Hyena, Rabbit, Shark, Wren converge on this value)

### Constraints

```typescript
interface OrbitControlsOptions {
  target: Vec3            // Look-at point (default: [0, 0, 0])
  distance: number        // Initial distance (default: 10)
  minDistance: number      // Zoom-in limit (default: 0.1)
  maxDistance: number      // Zoom-out limit (default: Infinity)
  minElevation: number    // Prevent looking through ground (default: -PI/2 + 0.01)
  maxElevation: number    // Prevent flipping at zenith (default: PI/2 - 0.01)
  minAzimuth: number      // Horizontal constraint (default: -Infinity = no limit)
  maxAzimuth: number      // Horizontal constraint (default: Infinity = no limit)
  dampingFactor: number   // Inertia (default: 0.1)
  rotateSpeed: number     // Orbit sensitivity (default: 1.0)
  panSpeed: number        // Pan sensitivity (default: 1.0)
  zoomSpeed: number       // Zoom sensitivity (default: 1.0)
  enableDamping: boolean  // Enable/disable damping (default: true)
  enabled: boolean        // Enable/disable all input (default: true)
}
```

Elevation is clamped to `[-PI/2 + epsilon, PI/2 - epsilon]` to prevent gimbal lock at the poles. Epsilon of `0.01` is sufficient.

### Input Handling

**Mouse events** (standard mapping, universal agreement):
- Left button + drag: orbit
- Right button + drag: pan
- Middle button + drag: pan
- Wheel: zoom

**Touch events** (universal agreement):
- 1-finger drag: orbit
- 2-finger drag: pan
- 2-finger pinch: zoom (via distance between touch points)

### Pan Implementation

All implementations agree: pan along the camera's right and up vectors, scaled by distance from target:

```typescript
const panScale = distance * panSpeed * pixelDelta / canvasHeight
target[0] += right[0] * panX + up[0] * panY
target[1] += right[1] * panX + up[1] * panY
target[2] += right[2] * panX + up[2] * panY
```

Distance-scaled pan ensures consistent screen-space motion regardless of zoom level.

### API

**Decision: Class-based** (Fennec, Hyena, Lynx, Mantis approach)

```typescript
const controls = new OrbitControls(camera, canvas, options)
controls.update(deltaTime)  // Call each frame
controls.dispose()          // Remove event listeners
```

Class-based is preferred over factory functions for controls because:
- Controls have mutable state (velocities, enabled flag, target)
- Lifecycle management (event listener setup/teardown) maps naturally to constructor/dispose
- More ergonomic for imperative use: `controls.target.set(5, 0, 0)`

### Programmatic Control

**Decision: Support animated transitions** (Hyena, Mantis, Rabbit, Shark approach)

```typescript
controls.setTarget([5, 0, 0], { animate: true, duration: 1.0 })
controls.setDistance(20, { animate: true, duration: 0.5 })
```

Animated transitions use exponential easing for smooth camera movement. This is useful for cinematic intros, focus-on-object effects, and UI-driven camera control.

### React Integration

```tsx
<OrbitControls
  target={[0, 0, 0]}
  minDistance={1}
  maxDistance={50}
  enableDamping
/>
```

The React component creates OrbitControls internally and syncs props. It auto-disposes on unmount.

## Future Controls (Not v1)

The architecture supports additional control types without changes to the camera system:
- First-person controls (WASD + mouse look)
- Fly controls (6DOF)
- Third-person follow camera

These are not v1 requirements but the OrbitControls implementation serves as the pattern for future control types.
