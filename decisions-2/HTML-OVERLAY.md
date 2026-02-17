# HTML-OVERLAY.md - Decisions

## Decision: CSS Transform Positioning, Raycast Occlusion, Boolean Center, Distance Factor Scaling

### Core Architecture

Universal agreement across all 8 active implementations:

- **Overlay container div** positioned absolutely on top of canvas with `pointer-events: none`
- **Individual overlay elements** positioned via CSS `transform: translate3d()`
- **GPU-accelerated compositing** via `will-change: transform`
- **Batch DOM writes** per frame (read all positions, then write all transforms)

### World-to-Screen Projection

Universal pipeline:

```typescript
// 1. Clip space
const clip = vec4TransformMat4(tempVec4, [wx, wy, wz, 1.0], viewProjectionMatrix)

// 2. Behind camera check
if (clip[3] <= 0) { element.style.display = 'none'; return }

// 3. NDC
const ndcX = clip[0] / clip[3]
const ndcY = clip[1] / clip[3]

// 4. Frustum check (with margin to prevent popping)
if (Math.abs(ndcX) > 1.2 || Math.abs(ndcY) > 1.2) { hide; return }

// 5. Screen coordinates (Y-flip for CSS)
const screenX = (ndcX * 0.5 + 0.5) * canvasWidth
const screenY = (1 - (ndcY * 0.5 + 0.5)) * canvasHeight
```

### Centering: Boolean Flag

**Chosen**: Boolean `center` flag (5/9: Caracal, Fennec, Hyena, Mantis, Rabbit)
**Rejected**: Anchor vector (3/9: Lynx, Shark, Wren) - more flexible but rarely needed for the target use cases (health bars, labels, tooltips)

```typescript
if (center) {
  element.style.transform = `translate(${x}px, ${y}px) translate(-50%, -50%)`
} else {
  element.style.transform = `translate(${x}px, ${y}px)`
}
```

The second `translate(-50%, -50%)` uses percentages to center based on the element's own size, which CSS handles natively without measuring.

### Position Tracking: Static + Node

**Chosen**: Support both static world position and scene node tracking (6/9: Caracal, Fennec, Hyena, Mantis, Rabbit, Shark)

```typescript
interface OverlayOptions {
  position?: Vec3          // Static world position
  node?: Node              // Track a scene node's world position
  offset?: Vec3            // Offset from tracked position (default: [0, 0, 0])
  center?: boolean         // Center element on point (default: true)
  occlude?: boolean        // Enable occlusion testing (default: false)
  distanceFactor?: number  // Distance-based scaling (default: 0, disabled)
  pointerEvents?: boolean  // Allow clicks (default: false)
}
```

When tracking a node, the overlay reads the node's world position each frame. Offset is applied in world space (e.g., `offset: [0, 0, 2.5]` for 2.5 units above the tracked point).

### Occlusion Testing: Raycast-Based

**Chosen**: Raycast from camera to overlay position (3/9: Caracal, Fennec, Wren)
**Rejected**: Depth buffer readback (4/9: Hyena, Lynx, Mantis, Rabbit) - causes GPU pipeline stall and 1 frame of latency

Raycast occlusion:

```typescript
if (overlay.occlude) {
  const ray = createRayFromCameraToPoint(camera, worldPosition)
  const hit = scene.raycast(ray, distance - 0.1)
  if (hit) {
    element.style.display = 'none'
    return
  }
}
```

Cost: ~0.05ms per overlay. For the typical case of <20 overlays, total cost is <1ms, well within budget.

Disabled by default (opt-in per overlay) since most overlays (labels, tooltips) don't need occlusion.

### Distance-Based Scaling

**Chosen**: Distance factor approach (5/9: Caracal, Fennec, Hyena, Rabbit, Shark)

```typescript
if (distanceFactor > 0) {
  const scale = distanceFactor / distance
  element.style.transform += ` scale(${scale})`
}
```

This makes overlays appear at a consistent "world size" - closer elements are larger, distant ones smaller. Useful for health bars above characters.

### Z-Index Management: Depth-Based Auto

**Chosen**: Automatic z-index based on depth (8/9 implementations)

```typescript
const normalizedDepth = (distance - nearPlane) / (farPlane - nearPlane)
const zIndex = Math.round(maxZIndex * (1 - normalizedDepth))
element.style.zIndex = String(zIndex)
```

Closer elements render on top. Z-index range: 0 to 16777271 (CSS integer max).

### Pointer Events

- Container: `pointer-events: none` (prevents blocking canvas input)
- Individual elements: opt-in via `pointer-events: auto`

```typescript
if (overlay.pointerEvents) {
  element.style.pointerEvents = 'auto'
}
```

### Performance Optimizations

**Dirty checking** (3/9: Caracal, Fennec, Mantis):

```typescript
if (Math.abs(newX - prevX) > 0.5 || Math.abs(newY - prevY) > 0.5) {
  element.style.transform = newTransform
  prevX = newX
  prevY = newY
}
```

Skip DOM writes when the projected position hasn't changed by at least 0.5 pixels. Reduces DOM mutations significantly for static or slow-moving objects.

**Staggered occlusion** (2/9: Caracal, Fennec): Check occlusion every 3-5 frames or stagger across overlays to amortize raycast cost.

**Display none for hidden elements**: Avoid compositing cost entirely for off-screen or occluded elements.

### React Integration

```tsx
<Html position={[0, 0, 2]} center occlude>
  <div className="health-bar">HP: 100</div>
</Html>

<Html node={characterRef} offset={[0, 0, 2.5]} distanceFactor={100}>
  <div>Player Name</div>
</Html>
```

Implemented via `ReactDOM.createPortal()` into the overlay container. React context flows through to overlay content.

### API (Imperative)

```typescript
const overlaySystem = createOverlaySystem(canvas)

const label = overlaySystem.add(element, {
  node: meshNode,
  offset: [0, 0, 2],
  center: true,
  occlude: true,
})

// Per frame (called by engine automatically if integrated)
overlaySystem.update(camera, canvasWidth, canvasHeight)

// Cleanup
label.remove()
overlaySystem.dispose()
```

### Performance Budget

- Projection per overlay: <0.001ms
- DOM update per overlay: <0.0001ms
- Raycast occlusion per overlay: ~0.05ms
- Total for 20 overlays with occlusion: <1.1ms
- Total for 50 overlays without occlusion: <0.1ms
