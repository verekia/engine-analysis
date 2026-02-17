# HTML-OVERLAY — Final Decision

## Architecture

**Decision: Absolute-positioned overlay container div with CSS transform positioning** (universal agreement)

```html
<div style="position: relative;">
  <canvas />
  <div id="overlay-container" style="position: absolute; inset: 0; pointer-events: none; overflow: hidden;">
    <!-- Overlay elements positioned via CSS transforms -->
  </div>
</div>
```

The overlay container sits directly above the canvas. Individual overlay elements are positioned using CSS `transform: translate(x, y)` based on their projected 3D position.

## Projection Pipeline

**Decision: Standard world-to-screen projection** (universal agreement)

```typescript
// 1. World position → clip space
const clipPos = vec4Transform(viewProjectionMatrix, worldPosition)

// 2. Clip space → NDC (perspective divide)
const ndcX = clipPos[0] / clipPos[3]
const ndcY = clipPos[1] / clipPos[3]

// 3. Check if behind camera
if (clipPos[3] <= 0) { hide element; return }

// 4. Check if outside frustum
if (Math.abs(ndcX) > 1 || Math.abs(ndcY) > 1) { hide element; return }

// 5. NDC → screen pixels
const screenX = (ndcX * 0.5 + 0.5) * canvasWidth
const screenY = (1 - (ndcY * 0.5 + 0.5)) * canvasHeight  // Y flipped
```

## CSS Transform Positioning

**Decision: GPU-accelerated CSS transforms** (universal agreement)

```typescript
element.style.transform = `translate(${screenX}px, ${screenY}px)`
element.style.willChange = 'transform'  // Hint for GPU layer promotion
```

CSS transforms are GPU-accelerated and avoid triggering layout/reflow.

## Centering

**Decision: Boolean center flag** (universal agreement)

```typescript
if (center) {
  element.style.transform = `translate(${screenX - width / 2}px, ${screenY - height / 2}px)`
} else {
  element.style.transform = `translate(${screenX}px, ${screenY}px)`
}
```

- `center: true` — element centered on the projected point (default for labels)
- `center: false` — element top-left corner at the projected point

## Tracking Modes

### Static Position

Fixed world-space position:

```typescript
overlay.add({
  element: labelDiv,
  position: [10, 5, 3],
  center: true,
})
```

### Node Tracking

Follow a scene node with optional offset:

```typescript
overlay.add({
  element: healthBar,
  node: characterMesh,
  offset: [0, 0, 2.5],  // 2.5 units above the character
  center: true,
})
```

The overlay position is computed as `node.worldPosition + offset` each frame.

## Occlusion Testing

**Decision: Raycast-based occlusion, opt-in** (universal agreement)

```typescript
overlay.add({
  element: labelDiv,
  node: characterMesh,
  offset: [0, 0, 2.5],
  occlude: true,  // Enable occlusion testing
})
```

When enabled, a ray is cast from the camera to the overlay's world position. If the ray hits an opaque object closer than the overlay, the element is hidden.

### Performance: Staggered Updates

**Decision: Stagger occlusion checks every 3 frames** (decisions-3 approach)

Not all overlays are tested every frame. They are distributed across frames to spread the cost:

```typescript
if ((frameCount + overlayIndex) % 3 === 0) {
  overlay.occluded = testOcclusion(overlay, camera, scene)
}
```

This reduces the per-frame occlusion cost from N rays to N/3 rays.

## Distance Scaling

**Decision: Optional factor-based distance scaling** (universal agreement)

```typescript
overlay.add({
  element: labelDiv,
  node: mesh,
  distanceScale: true,  // Element scales with distance from camera
})
```

When enabled, the element's CSS `scale` is adjusted based on distance from the camera, creating a perspective-consistent size appearance.

## Depth-Based Z-Index

**Decision: Automatic z-index from projected depth** (universal agreement)

Overlays closer to the camera get higher z-index values, ensuring proper stacking order:

```typescript
element.style.zIndex = String(Math.floor((1 - normalizedDepth) * 10000))
```

## Pointer Events

**Decision: Opt-in per element** (universal agreement)

The overlay container has `pointer-events: none`. Individual elements can opt in:

```typescript
overlay.add({
  element: buttonDiv,
  node: mesh,
  pointerEvents: true,  // Sets pointer-events: auto on element
})
```

## Dirty Checking

**Decision: Skip DOM updates when position change is <0.5px** (decisions-3 approach)

```typescript
if (Math.abs(newX - oldX) < 0.5 && Math.abs(newY - oldY) < 0.5) return
```

Minimizes DOM writes, which can trigger browser compositing overhead even with CSS transforms.

## Visibility Handling

Elements are hidden (`display: none`) when:
- Behind the camera (`clipW <= 0`)
- Outside the frustum
- Occluded by opaque geometry (when `occlude: true`)

## API

```typescript
const overlay = createOverlayManager(canvas)

const label = overlay.add({
  element: labelDiv,
  node: characterMesh,
  offset: [0, 0, 2.5],
  center: true,
  occlude: true,
  distanceScale: false,
  pointerEvents: false,
})

// Update (called internally by engine each frame)
overlay.update(camera, canvasWidth, canvasHeight)

// Remove
overlay.remove(label)

// Cleanup
overlay.dispose()
```

## React Integration

```tsx
import { Html } from 'engine-react'

<Html position={[0, 0, 2.5]} center occlude>
  <div className="player-label">Player Name</div>
</Html>
```

Implemented via `ReactDOM.createPortal` into the overlay container div. Props map directly to overlay options.

## Performance Budget

```
Position projection (20 overlays):    ~0.05ms
DOM updates (20 overlays):            ~0.15ms
Occlusion rays (7 per frame, staggered): ~0.1ms
────────────────────────────────────────────────
Total overlay overhead:               ~0.3ms
```
