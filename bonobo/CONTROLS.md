# Controls

## Status: Not Yet Specified

Camera controls are not yet fully specified for v1.

## Requirements

Basic orbit controls for 3D camera interaction:

- Orbit (rotate around target)
- Zoom (dolly in/out)
- Pan (move target)
- Mouse and touch support
- Smooth damping

## Planned Features (v1)

### Orbit Controls

Similar to Three.js OrbitControls:

```ts
interface OrbitControlsProps {
  target: [number, number, number]  // Look-at point
  distance: number                  // Distance from target
  minDistance?: number
  maxDistance?: number
  azimuth: number                   // Horizontal rotation (radians)
  elevation: number                 // Vertical rotation (radians)
  minElevation?: number             // Prevent flipping
  maxElevation?: number
  enableDamping?: boolean
  dampingFactor?: number
  enablePan?: boolean
  panSpeed?: number
  enableZoom?: boolean
  zoomSpeed?: number
}
```

### Input Handling

- **Left mouse drag**: Orbit (rotate camera around target)
- **Right mouse drag**: Pan (move target in screen space)
- **Mouse wheel**: Zoom (change distance from target)
- **Touch**: Two-finger pinch/drag for zoom/orbit

### React Integration

```tsx
function OrbitControls({
  target = [0, 0, 0],
  distance = 10,
  azimuth = 0,
  elevation = Math.PI / 4,
  enableDamping = true,
  dampingFactor = 0.05,
}: OrbitControlsProps) {
  const world = useWorld()

  useEffect(() => {
    // Set up event listeners
    // Update camera on drag/wheel
    // Apply damping in animation loop
  }, [/* ... */])

  return null
}
```

Usage:

```tsx
<Canvas>
  <OrbitControls
    target={[0, 0, 0]}
    distance={10}
    enableDamping
  />
  {/* Scene content */}
</Canvas>
```

### Implementation

Orbit controls compute camera position from spherical coordinates:

```ts
function computeCameraPosition(
  target: vec3,
  distance: number,
  azimuth: number,
  elevation: number
): vec3 {
  const x = target[0] + distance * Math.cos(elevation) * Math.cos(azimuth)
  const y = target[1] + distance * Math.cos(elevation) * Math.sin(azimuth)
  const z = target[2] + distance * Math.sin(elevation)
  return [x, y, z]
}
```

Then update the camera:

```ts
world.setCamera({
  eye: computeCameraPosition(target, distance, azimuth, elevation),
  target,
  up: [0, 0, 1],  // Z-up
})
```

### Damping

Smooth interpolation for natural feel:

```ts
// In animation loop
currentAzimuth += (targetAzimuth - currentAzimuth) * dampingFactor
currentElevation += (targetElevation - currentElevation) * dampingFactor
currentDistance += (targetDistance - currentDistance) * dampingFactor
```

## Additional Controls (Future)

### First-Person Controls

- WASD movement
- Mouse look
- Sprint, crouch, jump

### Fly Controls

- Free 6DOF camera movement
- No gravity, no ground constraint

### Third-Person Controls

- Follow a target entity
- Collision avoidance (raycast to prevent camera clipping through walls)

## Touch Gestures

- Single-finger drag: Orbit
- Two-finger drag: Pan
- Two-finger pinch: Zoom
- Support for mobile/tablet

## Accessibility

- Keyboard navigation (arrow keys for orbit)
- Configurable sensitivity
- Invert Y-axis option

## Implementation Priority

v1 feature, needed for basic interactivity. Start with orbit controls, add others as needed.
