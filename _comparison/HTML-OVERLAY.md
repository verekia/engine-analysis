# HTML Overlay Comparison Across 9 Implementations

This document compares HTML overlay systems for positioning DOM elements at 3D scene positions across bonobo, caracal, fennec, hyena, lynx, mantis, rabbit, shark, and wren.

## Universal Agreement

All implementations that specify overlays (8 out of 9) agree on:

### Core Architecture
- **Overlay container div** positioned absolutely on top of canvas with `pointer-events: none`
- **Individual overlay elements** positioned at projected 3D coordinates
- **CSS transforms** for positioning (not `left`/`top`) to leverage GPU compositing
- **`translate3d()` or `translate()` + `translate(-50%, -50%)`** for centering

### Projection Pipeline
All use identical world-to-screen projection:
1. Multiply world position by view-projection matrix → clip space
2. Perspective divide (clip / w) → NDC (normalized device coordinates)
3. Check if behind camera (w ≤ 0)
4. Check frustum bounds (NDC outside [-1, 1] range)
5. Map NDC to screen pixels with Y-flip for CSS coordinates

### Essential Features
- **Center alignment** option (default: true) - centers element on projected point
- **Visibility culling** - hide elements behind camera or off-screen
- **Z-index based on depth** - closer elements render on top
- **Distance-based scaling** (optional) - elements scale with distance from camera

### Performance Optimizations
All implementations use:
- **GPU-accelerated transforms** (`transform: translate()`)
- **`will-change: transform`** hint for browser compositing
- **Batch DOM updates** - all position writes in single frame pass
- **Display none for hidden elements** - avoid compositing cost

## Key Variations

### 1. Implementation Status

**Not Yet Implemented (1 implementation)**
- **Bonobo**: Explicitly out of scope for v1, planned for v2

**Fully Implemented (8 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark, Wren**: Complete implementations with detailed specs

### 2. API Design

**Manager/System class (6 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit**: Object-oriented API
```typescript
const overlay = createOverlayManager(canvas)
overlay.add({ element, worldPosition, options })
overlay.update(camera, width, height)
```

**Direct function (1 implementation)**
- **Shark**: Functional approach with HtmlAnchor nodes
```typescript
const anchor = new HtmlAnchor({ center: [0.5, 1] })
anchor.element.innerHTML = '<div>Label</div>'
scene.add(anchor)
```

**Not fully specified (1 implementation)**
- **Wren**: Mentioned but implementation details sparse

### 3. Position Tracking

**Static world position (all implementations)**
```typescript
worldPosition: [x, y, z]
```

**Node tracking (7 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark**: Track a scene node's world position
```typescript
node: meshNode  // Automatically follows node's world position
```

**Bone tracking (2 implementations)**
- **Rabbit, Wren**: Track skeleton bone positions
```typescript
bone: { skeleton, boneName: 'head' }
```

**Offset support (8 implementations)**
- **All except Bonobo**: Add offset to tracked position
```typescript
offset: [0, 0, 2.5]  // 2.5 units above tracked point
```

### 4. Centering/Anchoring

**Boolean center flag (5 implementations)**
- **Caracal, Fennec, Hyena, Mantis, Rabbit**: Simple true/false
```typescript
center: true  // Centers element on point
```

**Anchor vector (3 implementations)**
- **Lynx, Shark, Wren**: [x, y] where [0.5, 0.5] = center, [0, 0] = top-left
```typescript
anchor: [0.5, 1.0]  // Bottom-center alignment
```

### 5. Occlusion Testing

**Raycast-based (6 implementations)**
- **Caracal, Fennec, Hyena, Mantis, Rabbit, Wren**: Cast ray from camera to overlay position
```typescript
const ray = camera.rayTo(worldPosition)
const hit = scene.raycast(ray, maxDistance)
const occluded = hit && hit.distance < overlayDistance
```
- Pros: No GPU readback, works on all backends
- Cons: CPU cost per overlay (~0.05ms each)

**Depth buffer readback (4 implementations)**
- **Hyena, Lynx, Mantis, Rabbit**: Read depth texture at projected pixel
```typescript
const sceneDepth = readDepthAtPixel(screenX, screenY)
const occluded = overlayDepth > sceneDepth
```
- Pros: O(1) per overlay
- Cons: GPU→CPU pipeline stall, 1 frame latency

**Both approaches mentioned (3 implementations)**
- **Hyena, Lynx, Mantis**: Document both strategies, let user choose

**Opt-in by default (8 implementations)**
- Occlusion testing disabled by default (expensive), enabled per overlay element

### 6. Distance Scaling

**Distance factor (7 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark**: Scale element by `factor / distance`
```typescript
distanceFactor: 100  // Element at distance 10 has scale 10
```

**Reference distance (2 implementations)**
- **Lynx, Mantis**: Specify distance at which scale = 1.0
```typescript
referenceDistance: 15  // Scale 1.0 at 15 units, larger closer, smaller farther
```

**Min/max scale clamps (2 implementations)**
- **Lynx, Mantis**: Prevent elements from becoming too large or small
```typescript
minScale: 0.3, maxScale: 2.0
```

### 7. Z-Index Management

**Depth-based auto z-index (8 implementations)**
- **All except Bonobo**: Automatically assign z-index based on depth
```typescript
zIndex = maxZ - (normalizedDepth * (maxZ - minZ))
```

**Configurable range (6 implementations)**
- **Caracal, Fennec, Lynx, Mantis, Rabbit, Wren**: Specify z-index range
```typescript
zIndexRange: [0, 16777271]  // or [1000, 0]
```

**Fixed z-index option (2 implementations)**
- **Shark, Rabbit**: Allow manual z-index instead of auto

### 8. Pointer Events

**Container non-interactive (all implementations)**
- Overlay container has `pointer-events: none`
- Prevents blocking canvas mouse events

**Per-element opt-in (8 implementations)**
- Individual elements set `pointer-events: auto` to receive clicks
```typescript
pointerEvents: true  // or element.style.pointerEvents = 'auto'
```

### 9. Performance Optimizations

**Dirty checking (3 implementations)**
- **Caracal, Fennec, Mantis**: Only update DOM if position changed > threshold
```typescript
if (Math.abs(newX - prevX) > 0.5 || Math.abs(newY - prevY) > 0.5) {
  element.style.transform = ...
}
```

**Staggered occlusion checks (2 implementations)**
- **Caracal, Fennec**: Check occlusion every N frames or stagger across elements

**Element pooling mentioned (2 implementations)**
- **Rabbit, Shark**: Suggest pooling for dynamic elements (damage numbers, hit markers)

### 10. React Integration

**`<Html>` component (7 implementations)**
- **Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark**: Declarative React component
```tsx
<Html position={[0, 0, 2]} center occlude>
  <div>Health: 100</div>
</Html>
```

**React portal rendering (7 implementations)**
- Render children into overlay container via `ReactDOM.createPortal()`
- React context flows through to overlay content

**`<Overlay>` variant (1 implementation)**
- **Caracal**: Uses `<Overlay>` instead of `<Html>` naming

**Future/not specified (1 implementation)**
- **Bonobo**: Out of scope for v1

## Implementation Breakdown

### By Completeness
- **Not implemented**: Bonobo (1)
- **Basic implementation**: Wren (1) - minimal specification
- **Standard implementation**: Fennec, Rabbit (2) - core features
- **Advanced implementation**: Caracal, Hyena, Lynx, Mantis, Shark (5) - optimizations, multiple strategies

### By Position Tracking
- **Static + Node**: Caracal, Fennec, Hyena, Mantis, Rabbit, Shark (6)
- **Static + Node + Bone**: Rabbit, Wren (2)
- **Static only**: Lynx (1)

### By Occlusion Strategy
- **Raycast only**: Caracal, Fennec, Wren (3)
- **Depth readback only**: None (0)
- **Both documented**: Hyena, Lynx, Mantis (3)
- **Either mentioned**: Rabbit, Shark (2)

### By Centering Approach
- **Boolean flag**: Caracal, Fennec, Hyena, Mantis, Rabbit (5)
- **Anchor vector**: Lynx, Shark, Wren (3)

### By Distance Scaling
- **Factor only**: Caracal, Fennec, Hyena, Rabbit, Shark (5)
- **Factor + reference + clamps**: Lynx, Mantis (2)
- **Not mentioned**: Wren (1)

## Cherry-Picking Guide

### For Simplest Implementation
**Use Fennec or Rabbit**: Clean API, essential features only, straightforward projection math

### For Most Features
**Use Mantis or Lynx**:
- Both occlusion strategies documented
- Reference distance for intuitive scaling
- Min/max scale clamps
- Comprehensive configuration

### For Best Performance
**Use Caracal or Mantis**:
- Dirty checking to minimize DOM writes
- Staggered occlusion updates
- Batch read/write separation

### For Flexible Positioning
**Use Rabbit or Wren**: Bone tracking support for character-attached UI (health bars above heads)

### For Anchor Control
- **Boolean center**: Simpler, covers most use cases (Caracal, Fennec, Hyena, Mantis, Rabbit)
- **Anchor vector**: More flexible, precise control (Lynx, Shark, Wren)

### For Occlusion Strategy
**Raycast** (Caracal, Fennec, Wren):
- Best for < 20 overlays
- No GPU readback overhead
- Works consistently across backends

**Depth readback** (Hyena, Lynx, Mantis):
- Best for 20+ overlays
- O(1) per overlay
- Requires async readback handling

**Hybrid** (Mantis):
- Document both, let developer choose based on use case

### For React Integration
All implementations with React bindings provide `<Html>` component. Hyena and Mantis offer most comprehensive integration with context flow and portal rendering.

## Common Pitfalls

### CSS Transform Centering
Most implementations use:
```typescript
transform: `translate(${x}px, ${y}px) translate(-50%, -50%)`
```
The second translate uses percentages to center based on element's own size.

### Y-Axis Flip
NDC Y is +1 at top, -1 at bottom. Screen Y is 0 at top, height at bottom:
```typescript
screenY = (1 - (ndcY * 0.5 + 0.5)) * height
```
All implementations handle this correctly.

### Frustum Margin
Most use ±1.2 instead of ±1.0 for frustum check to prevent popping at screen edges.

### Behind Camera Check
Critical to check `clipW <= 0` before perspective divide to avoid rendering elements behind camera.

## Typical Use Cases

All implementations target these scenarios:

1. **Health bars above characters**
   - Track character node + offset upward
   - Distance scaling for consistent size
   - Occlusion to hide through walls

2. **Name tags / labels**
   - Static or node-tracked position
   - Center alignment
   - Auto z-index by depth

3. **Tooltips on hover**
   - Temporary overlays at interaction points
   - No occlusion needed
   - Pointer events for click-through

4. **UI panels anchored to 3D objects**
   - Track moving objects
   - Maintain fixed screen size (no distance scaling)
   - Pointer events enabled for interaction

## Performance Budget

Typical costs (from documentation):
- **Projection per overlay**: < 0.001ms (matrix multiply)
- **DOM update per overlay**: < 0.0001ms (transform write)
- **Raycast occlusion**: ~0.05ms per test
- **Depth readback**: ~0.5ms with 1 frame latency
- **Total for 50 overlays**: < 0.15ms without occlusion, ~0.5ms with occlusion

## Recommendation Summary

- **Getting started**: Fennec or Rabbit (simple, complete)
- **Advanced features**: Mantis or Lynx (comprehensive options)
- **Performance-critical**: Caracal (dirty checking, staggered updates)
- **Character UI**: Rabbit or Wren (bone tracking)
- **Flexible alignment**: Lynx or Shark (anchor vector)
- **React-first**: Hyena or Mantis (best React integration)
