# HTML-OVERLAY - Design Decisions

## Core Architecture

**Decision: Overlay container div with CSS transform positioning (universal agreement)**

```html
<div style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; overflow: hidden;">
  <!-- overlay elements positioned via CSS transforms -->
</div>
```

Container sits on top of canvas. `pointer-events: none` prevents blocking canvas mouse events. Individual elements opt-in to interactivity with `pointer-events: auto`.

## Projection Pipeline

**Decision: Standard world-to-screen projection (universal agreement)**

```typescript
// 1. World position → clip space
const clip = vec4TransformMat4(tempVec4, [wx, wy, wz, 1], viewProjectionMatrix)

// 2. Behind camera check
if (clip[3] <= 0) { hide(); return }

// 3. Perspective divide → NDC
const ndcX = clip[0] / clip[3]
const ndcY = clip[1] / clip[3]

// 4. Frustum check (with margin to prevent edge popping)
if (Math.abs(ndcX) > 1.2 || Math.abs(ndcY) > 1.2) { hide(); return }

// 5. NDC → screen pixels (Y-flip for CSS coordinates)
const screenX = (ndcX * 0.5 + 0.5) * canvasWidth
const screenY = (1 - (ndcY * 0.5 + 0.5)) * canvasHeight
```

## Positioning via CSS Transforms

**Decision: GPU-accelerated transforms with will-change hint**

```typescript
element.style.transform = `translate(${screenX}px, ${screenY}px) translate(-50%, -50%)`
element.style.willChange = 'transform'
```

`translate3d` or `translate` for GPU compositing. The second `translate(-50%, -50%)` centers the element on the projected point using percentage-based offset (element's own size).

## API Design

**Decision: Manager/system class with add/remove/update**

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit (6/9)

```typescript
const overlay = createOverlayManager(canvas)

const handle = overlay.add({
  element: document.createElement('div'),
  position: [0, 0, 2],       // static world position
  // OR
  node: characterMesh,         // track a scene node
  offset: [0, 0, 2.5],        // offset above tracked point
  center: true,                // center element on projected point
  occlude: false,              // enable occlusion testing
  distanceFactor: 0,           // 0 = no scaling, >0 = scale by factor/distance
  pointerEvents: false,        // allow click-through by default
})

// Per-frame update (called by engine)
overlay.update(camera, canvasWidth, canvasHeight)

// Cleanup
overlay.remove(handle)
overlay.dispose()
```

## Position Tracking

**Decision: Static position + node tracking + offset**

- Sources: Caracal, Fennec, Hyena, Mantis, Rabbit, Shark (6/8)

Three modes:
1. **Static**: Fixed world position `[x, y, z]`
2. **Node tracking**: Follow a scene node's world position (extracted from world matrix)
3. **Offset**: Additional world-space offset added to tracked position (e.g., `[0, 0, 2.5]` for 2.5 units above)

Bone tracking (Rabbit, Wren) not in v1 — use bone attachment via scene graph instead (attach an empty node to a bone, track that node).

## Centering

**Decision: Boolean center flag**

- Sources: Caracal, Fennec, Hyena, Mantis, Rabbit (5/9)
- Rejected: Anchor vector (Lynx, Shark, Wren) — more flexible but overkill for most use cases

```typescript
center: true   // centers element on projected point (default)
center: false  // top-left corner at projected point
```

Implemented via CSS `translate(-50%, -50%)` which auto-adjusts to element size.

## Occlusion Testing

**Decision: Raycast-based occlusion (opt-in, disabled by default)**

- Sources: Caracal, Fennec, Wren (3/9)
- Rejected: Depth buffer readback (Hyena, Lynx, Mantis) — GPU→CPU stall, 1 frame latency

```typescript
// When occlude: true
const ray = createRayFromCameraToPoint(camera, worldPosition)
const hit = scene.raycast(ray, distanceToOverlay)
const isOccluded = hit !== null && hit.distance < overlayDistance

if (isOccluded) {
  element.style.display = 'none'  // or opacity fade
}
```

Raycast-based occlusion:
- No GPU readback overhead
- Works consistently across WebGL2 and WebGPU
- ~0.05ms per overlay element
- Best for <20 overlays with occlusion (typical use case)

Disabled by default because most overlays (labels, tooltips) don't need occlusion.

## Distance-Based Scaling

**Decision: Factor-based scaling**

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark (7/9)

```typescript
distanceFactor: 100  // scale = distanceFactor / distance
// distanceFactor: 0 means no scaling (fixed screen size)
```

When `distanceFactor > 0`, element scales as `distanceFactor / distance`, making it appear consistent in 3D space. When 0, element maintains fixed screen size regardless of distance.

## Z-Index Management

**Decision: Depth-based auto z-index**

- Sources: All 8 active implementations agree

```typescript
// Closer elements get higher z-index
const normalizedDepth = (clipW - near) / (far - near)
const zIndex = Math.floor((1 - normalizedDepth) * maxZIndex)
element.style.zIndex = String(zIndex)
```

Configurable z-index range (default: `[0, 16777271]`).

## DOM Update Optimization

**Decision: Dirty checking to minimize DOM writes**

- Sources: Caracal, Fennec, Mantis (3/9)

```typescript
// Only update DOM if position changed by > 0.5 pixels
if (Math.abs(newX - prevX) > 0.5 || Math.abs(newY - prevY) > 0.5) {
  element.style.transform = `translate(${newX}px, ${newY}px) translate(-50%, -50%)`
  prevX = newX
  prevY = newY
}
```

Also: batch all DOM reads before DOM writes to avoid layout thrashing.

## Staggered Occlusion Checks

**Decision: Check occlusion every N frames or stagger across elements**

- Sources: Caracal, Fennec (2/9)

For 20 overlays with occlusion: instead of raycasting all 20 every frame, stagger 5 per frame across 4 frames. Reduces per-frame occlusion cost from ~1ms to ~0.25ms.

## Visibility Handling

```typescript
// Behind camera: display: none
// Off-screen: display: none (with ±1.2 frustum margin)
// Occluded: display: none or opacity transition
// Visible: display: block/flex with computed transform
```

`display: none` avoids compositing cost for hidden elements.

## React Integration

**Decision: `<Html>` component with React portal**

- Sources: Caracal, Fennec, Hyena, Lynx, Mantis, Rabbit, Shark (7/9)

```tsx
<Html position={[0, 0, 2]} center occlude distanceFactor={100}>
  <div className="health-bar">
    <span>HP: {health}</span>
  </div>
</Html>
```

Implementation: `ReactDOM.createPortal(children, overlayContainer)`. React context flows through to overlay content, so hooks and state work normally inside `<Html>`.

## Performance Budget

- Projection per overlay: <0.001ms
- DOM update per overlay: <0.0001ms
- Raycast occlusion per overlay: ~0.05ms
- Total for 50 overlays (no occlusion): <0.15ms
- Total for 20 overlays (with occlusion): ~0.5ms

Well within frame budget.
