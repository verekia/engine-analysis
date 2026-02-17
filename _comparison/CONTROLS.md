# Controls Comparison Across 9 Implementations

This document compares orbit camera controls implementations across bonobo, caracal, fennec, hyena, lynx, mantis, rabbit, shark, and wren.

## Universal Agreement

All implementations that specify controls (8 out of 9) agree on:

### Core Functionality
- **Orbit controls** as the primary camera interaction mode for 3D scenes
- **Three core operations**: Orbit (rotate around target), Pan (move target), Zoom (change distance)
- **Z-up coordinate system** for all implementations (aligned with engine architecture)
- **Spherical coordinates** for camera positioning (distance, azimuth, elevation/polar)

### Input Mapping
All implementations use identical mouse/touch mappings:
- **Left mouse drag / 1-finger touch**: Orbit (rotate around target)
- **Right mouse drag / 2-finger drag**: Pan (move target)
- **Mouse wheel / pinch gesture**: Zoom (change distance)
- **Middle mouse drag**: Pan (alternative)

### Essential Features
- **Damping/inertia** for smooth camera movement (all enable by default)
- **Distance limits** (min/max distance from target)
- **Elevation/polar angle limits** to prevent camera flipping at poles
- **Touch support** with pinch-to-zoom and multi-finger gestures

### Spherical to Cartesian Conversion (Z-up)
All implementations use the same mathematical model:
```typescript
x = distance * cos(elevation) * cos(azimuth)
y = distance * cos(elevation) * sin(azimuth)
z = distance * sin(elevation)
```

## Key Variations

### 1. Specification Status

**Not Yet Specified (1 implementation)**
- **Bonobo**: Marked as "Not Yet Specified" for v1, with planned features for future versions

**Minimal Specification (1 implementation)**
- **Caracal**: Brief mention only, no detailed specification

**Full Specification (7 implementations)**
- **Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren**: Complete implementations with detailed documentation

### 2. Angle Convention Terminology

**Polar Angle (from Z-axis down) - 4 implementations**
- **Lynx, Shark**: Use "polar angle" measured from +Z axis (0 = looking down, π = looking up)
- Matches traditional spherical coordinate convention
- Range: typically [0.01, π - 0.01] to avoid gimbal lock

**Elevation Angle (from XY plane up) - 5 implementations**
- **Fennec, Hyena, Mantis, Rabbit, Wren**: Use "elevation" measured from XY plane toward +Z
- Range: typically [-π/2 + ε, π/2 - ε]
- More intuitive for game developers (positive = looking up)

### 3. Damping Implementation

**Velocity-based damping (4 implementations)**
- **Fennec, Hyena, Mantis, Wren**: Store velocities separately, apply exponential decay
```typescript
velocity *= (1 - dampingFactor)
value += velocity
```
- Allows momentum to continue after input stops
- More physically realistic feel

**Delta-based damping (3 implementations)**
- **Lynx, Rabbit, Shark**: Accumulate deltas, apply with damping factor
```typescript
value += delta * dampingFactor
delta *= (1 - dampingFactor)
```
- Simpler implementation
- Similar visual result to velocity-based

### 4. Azimuth Constraints

**Unrestricted rotation (default) - 6 implementations**
- **Fennec, Hyena, Lynx, Mantis, Rabbit, Wren**: Allow unlimited horizontal rotation by default
- Optional min/max azimuth for constrained scenarios

**No azimuth limits mentioned - 2 implementations**
- **Shark, Bonobo**: Don't specify azimuth constraints in documentation

### 5. Damping Factor Defaults

- **0.05**: Bonobo (planned)
- **0.08**: Lynx, Mantis
- **0.1**: Fennec, Hyena, Rabbit, Shark, Wren

Most implementations converge on 0.08-0.1 as the sweet spot for responsive but smooth controls.

### 6. API Design

**Class-based (4 implementations)**
- **Fennec, Hyena, Lynx, Mantis**: Object-oriented API with `new OrbitControls()`
```typescript
const controls = new OrbitControls(camera, canvas, options)
controls.update(dt)
controls.dispose()
```

**Factory function (3 implementations)**
- **Rabbit, Shark, Wren**: Functional API with `createOrbitControls()`
```typescript
const controls = createOrbitControls(camera, canvas, options)
```

**React Component (all specified implementations)**
All 7 implementations with full specs provide a React wrapper:
```tsx
<OrbitControls target={[0,0,0]} enableDamping />
```

### 7. Advanced Features

**Programmatic camera control (4 implementations)**
- **Hyena, Mantis, Rabbit, Shark**: Support animated transitions
```typescript
controls.setTarget([x, y, z], { animate: true, duration: 1.0 })
```

**Auto-rotate (1 implementation)**
- **Shark**: Includes auto-rotate feature for slowly rotating around target

**Enabled/disabled state (most implementations)**
- **Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren**: Support `enabled` property

### 8. Pan Implementation

**Screen-space pan (all implementations)**
All use camera's right and up vectors to pan in screen-aligned plane:
```typescript
const right = camera.getRightVector()
const up = camera.getUpVector()
target += right * panX + up * panY
```

**Pan scaling by distance (all implementations)**
Pan speed scales with distance from target so screen motion feels consistent:
```typescript
const panScale = distance * panSpeed * pixelDelta
```

## Implementation Breakdown

### By Completeness
- **Minimal/Not Specified**: Bonobo (future), Caracal (brief mention)
- **Standard Implementation**: Fennec, Rabbit, Wren (core features only)
- **Enhanced Implementation**: Hyena, Lynx, Mantis, Shark (programmatic control, animations)

### By Angle Convention
- **Polar (from Z-axis)**: Lynx (4), Shark (2)
- **Elevation (from XY-plane)**: Fennec (5), Hyena, Mantis, Rabbit, Wren

### By Damping Style
- **Velocity-based**: Fennec (4), Hyena, Mantis, Wren
- **Delta-based**: Lynx (3), Rabbit, Shark

### By API Style
- **Class-based**: Fennec (4), Hyena, Lynx, Mantis
- **Factory function**: Rabbit (3), Shark, Wren
- **React-only**: Bonobo (planned)

## Cherry-Picking Guide

### For Simplest Implementation
**Use Wren or Rabbit**: Clean factory functions, minimal API surface, velocity-based damping

### For Most Features
**Use Hyena or Mantis**: Programmatic camera control, animated transitions, extensive configuration

### For Best Touch Support
**Use Fennec or Lynx**: Detailed touch event handling with pinch distance tracking

### For Angle Convention
- **Choose Elevation** (Fennec, Hyena, Mantis, Rabbit, Wren) if: Targeting game developers, more intuitive "looking up/down"
- **Choose Polar** (Lynx, Shark) if: Maintaining mathematical convention, integrating with existing spherical systems

### For Damping Feel
- **Velocity-based** (Fennec, Hyena, Mantis, Wren): Better for physics-like momentum
- **Delta-based** (Lynx, Rabbit, Shark): Simpler code, slightly snappier response

### For React Integration
All specified implementations offer `<OrbitControls>` component. Hyena and Mantis additionally support refs for imperative control within React.

## Implementation Priority

**All implementations agree**: Orbit controls are essential for v1, required for basic 3D scene interaction.

**Future controls** mentioned by several implementations (lower priority):
- First-person controls (WASD + mouse look)
- Fly controls (6DOF movement)
- Third-person follow camera
- Keyboard navigation for accessibility
