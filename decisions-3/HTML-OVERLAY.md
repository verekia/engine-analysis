# HTML-OVERLAY.md - HTML Overlay System

## Architecture

**Decision: Absolute-positioned overlay container with CSS transform positioning** (universal agreement)

### Container Setup

```html
<div style="position: relative;">
  <canvas />
  <div id="overlay-container" style="
    position: absolute;
    top: 0; left: 0;
    width: 100%; height: 100%;
    pointer-events: none;
    overflow: hidden;
  " />
</div>
```

The overlay container sits on top of the canvas with `pointer-events: none` so mouse events pass through to the canvas below.

### Projection Pipeline (Universal Agreement)

1. Multiply world position by view-projection matrix -> clip space
2. Perspective divide (clip / w) -> NDC (-1 to 1)
3. Check if behind camera (w <= 0) -> hide
4. Check frustum bounds (NDC outside [-1, 1] with margin) -> hide
5. Map NDC to screen pixels with Y-flip for CSS coordinates
6. Apply CSS `transform: translate()` for GPU-accelerated positioning

```typescript
const clip = vec4TransformMat4(tempVec4, [wx, wy, wz, 1], viewProjection)
if (clip[3] <= 0) { hide(); return }  // Behind camera

const ndcX = clip[0] / clip[3]
const ndcY = clip[1] / clip[3]

if (Math.abs(ndcX) > 1.2 || Math.abs(ndcY) > 1.2) { hide(); return }  // Off-screen

const screenX = (ndcX * 0.5 + 0.5) * canvasWidth
const screenY = (1 - (ndcY * 0.5 + 0.5)) * canvasHeight  // Y-flip

element.style.transform = `translate(${screenX}px, ${screenY}px) translate(-50%, -50%)`
```

The `translate(-50%, -50%)` centers the element on the projected point using the element's own dimensions.

## API Design

**Decision: Manager-based API with node tracking** (Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit approach)

```typescript
const overlay = createHTMLOverlaySystem(canvas)

const label = overlay.add({
  element: document.createElement('div'),
  node: meshNode,          // Track a scene node's world position
  offset: [0, 0, 2.5],    // 2.5 units above the node
  center: true,            // Center element on projected point
  occlude: false,          // Whether to test for occlusion
  distanceFactor: 0,       // 0 = no distance scaling
  pointerEvents: false,    // Element is non-interactive
})

// Per-frame update (called automatically by engine)
overlay.update(camera, canvasWidth, canvasHeight)
```

### Position Tracking

- **Static position**: `worldPosition: [x, y, z]`
- **Node tracking**: `node: meshNode` - automatically follows node's world matrix
- **Offset**: Added to tracked position (e.g., health bar 2 units above character)

## Centering

**Decision: Boolean center flag** (Caracal, Fennec, Hyena, Mantis, Rabbit - 5/8 use this)

```typescript
center: true   // translate(-50%, -50%) to center element on point
center: false  // top-left of element at projected point
```

**Why not anchor vector (Lynx, Shark, Wren)?**

Anchor vectors `[0.5, 1.0]` provide more flexibility (e.g., bottom-center alignment), but the vast majority of use cases need either centered (health bars, labels) or top-left (tooltips). A boolean covers both cases simply. For the rare case needing custom anchoring, users can apply CSS transforms to the element itself.

## Occlusion

**Decision: Raycast-based occlusion, opt-in** (Caracal, Fennec, Wren - default for <20 overlays)

Cast a ray from the camera to the overlay's world position. If the ray hits an opaque object before reaching the overlay, the element is occluded.

```typescript
const overlayDist = vec3Distance(camera.position, worldPosition)
const ray = createRay(camera.position, vec3Normalize(vec3Sub(worldPosition, camera.position)))
const hit = raycaster.intersectScene(scene, { near: 0, far: overlayDist })
if (hit && hit.distance < overlayDist - 0.1) {
  element.style.display = 'none'
}
```

**Why raycast over depth readback?**

Depth readback (Hyena, Lynx, Mantis) is O(1) per overlay but:
- Requires GPU-to-CPU readback (pipeline stall, 1 frame latency)
- WebGPU readback API is asynchronous and complex
- WebGL2 readback via `readPixels` is synchronous and slow

Raycast occlusion is O(log n) per overlay (via BVH), consistent across both backends, and has no pipeline stall. For typical overlay counts (5-20), the total cost is ~0.25-1ms, which is acceptable.

**Occlusion is disabled by default** (8/8 implementations agree it is opt-in). Most overlays do not need occlusion testing. Enable per-element:

```typescript
overlay.add({ element, node, occlude: true })
```

### Staggered Occlusion Updates

**Decision: Check occlusion every 3 frames** (Caracal, Fennec approach)

For overlays with occlusion enabled, stagger the checks to reduce per-frame cost:

```typescript
if (frameCount % 3 === overlayIndex % 3) {
  checkOcclusion(overlay)
}
```

This spreads the raycast cost across frames. With 15 occluded overlays, only 5 are tested per frame.

## Z-Index Management

**Decision: Depth-based auto z-index** (universal agreement)

Closer overlays render on top of farther ones:

```typescript
const normalizedDepth = (clipZ - near) / (far - near)  // 0 = near, 1 = far
const zIndex = Math.floor((1 - normalizedDepth) * 10000)
element.style.zIndex = String(zIndex)
```

## Distance Scaling

**Decision: Optional factor-based scaling** (Caracal, Fennec, Hyena, Rabbit, Shark approach)

```typescript
distanceFactor: 100  // Scale = distanceFactor / distance
```

When `distanceFactor > 0`, the element scales with distance so that it appears roughly the same size in the 3D world. At `distance = distanceFactor`, scale = 1.0. Closer = larger, farther = smaller.

When `distanceFactor = 0` (default), the element maintains its CSS pixel size regardless of camera distance.

## Pointer Events

```typescript
overlay.add({
  element: buttonDiv,
  pointerEvents: true,  // Sets pointer-events: auto on this element
})
```

Individual elements can opt into receiving mouse/touch events by setting `pointer-events: auto`. The overlay container remains `pointer-events: none`.

## Dirty Checking

**Decision: Skip DOM update if position changed less than 0.5px** (Caracal, Fennec, Mantis approach)

```typescript
if (Math.abs(newX - prevX) > 0.5 || Math.abs(newY - prevY) > 0.5) {
  element.style.transform = `translate(${newX}px, ${newY}px) translate(-50%, -50%)`
  prevX = newX
  prevY = newY
}
```

Sub-pixel DOM updates are invisible and waste compositing bandwidth. Skipping them reduces DOM writes by ~70% in typical scenes (most overlays move less than 0.5px per frame when the camera is stationary).

## React Integration

```tsx
<Html position={[0, 0, 2]} center occlude>
  <div className="health-bar">HP: 100</div>
</Html>
```

The `<Html>` component:
1. Creates a DOM element in the overlay container via `ReactDOM.createPortal()`
2. Registers with the overlay system for position tracking
3. React context flows through the portal (state, theme, etc.)
4. Auto-disposes on unmount

Props map to overlay options:
- `position`: world position or `node` tracking
- `center`: boolean center flag
- `occlude`: enable occlusion testing
- `distanceFactor`: distance-based scaling
- `pointerEvents`: enable interaction

## Performance

| Operation | Cost |
|-----------|------|
| Projection per overlay | < 0.001ms |
| DOM transform write | < 0.001ms |
| Raycast occlusion (per test) | ~0.05ms |
| Total (20 overlays, 5 occluded) | ~0.3ms |

HTML overlay updates are one of the cheapest operations in the frame. Even 50 overlays with no occlusion cost < 0.1ms.
