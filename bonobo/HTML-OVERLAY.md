# HTML Overlay

## Status: Out of Scope for v1

HTML overlay system (CSS3DRenderer / Drei Html-like) is out of scope for v1.

## Requirements (Future)

For v2, add support for HTML overlays positioned in 3D space:

- HTML elements that follow 3D positions (like Drei's `<Html>` component)
- CSS 3D transforms for positioning
- Automatic occlusion (hide when behind objects)
- Screen-space UI anchoring

## Planned Features (v2)

### 3D-Positioned HTML

```tsx
<Html position={[0, 0, 1]} center>
  <div className="label">
    Player Name
  </div>
</Html>
```

Rendered as a CSS 3D transformed div that follows the 3D position.

### Occlusion

Option to hide HTML elements when occluded by 3D geometry:

```tsx
<Html position={[0, 0, 1]} occlude>
  <div>Only visible when not behind objects</div>
</Html>
```

Implementation: Raycast from camera to HTML element position, hide if intersects geometry.

### Screen-Space Anchoring

Anchor HTML to screen corners/edges:

```tsx
<Html screenPosition="top-left">
  <div>HUD Element</div>
</Html>
```

### Portal

Render HTML to a specific DOM container (for multi-canvas setups):

```tsx
<Html portal={document.getElementById('ui-container')}>
  <div>UI Content</div>
</Html>
```

## Implementation Strategy

### CSS 3D Transforms

Use CSS 3D transforms to position HTML elements:

```ts
function updateHtmlPosition(
  element: HTMLElement,
  worldPos: vec3,
  camera: Camera
) {
  // Project 3D position to screen space
  const screenPos = project(worldPos, camera.viewProjection)

  // Apply CSS transform
  element.style.transform = `
    translate3d(${screenPos.x}px, ${screenPos.y}px, 0)
    scale(${screenPos.z})  // Perspective scaling
  `
}
```

### Occlusion Detection

Raycast from camera to HTML element position:

```ts
function isOccluded(position: vec3, camera: Camera): boolean {
  const ray = {
    origin: camera.eye,
    direction: normalize(sub(position, camera.eye))
  }
  const hit = raycast(ray)
  if (!hit) return false

  const distToHtml = distance(camera.eye, position)
  return hit.distance < distToHtml
}
```

Hide element if occluded:

```ts
element.style.visibility = isOccluded(position, camera) ? 'hidden' : 'visible'
```

### Performance

- Update HTML positions only for visible elements
- Throttle updates to 60fps (or sync with render loop)
- Use CSS `will-change` for better compositing

## Alternative: Canvas-Based UI

Instead of HTML overlays, consider canvas-based UI (like Dear ImGui):

- Draw UI elements directly in WebGPU/WebGL
- Full control over rendering
- Better performance for many UI elements
- No DOM overhead

Libraries:
- [imgui-js](https://github.com/flyover/imgui-js) — Dear ImGui port to JS
- Custom canvas-based UI toolkit

## Use Cases

### 3D Labels

Player names, object tooltips, annotations:

```tsx
<Html position={playerPosition} center>
  <div className="player-label">{playerName}</div>
</Html>
```

### HUD Elements

Health bars, minimaps, score displays (screen-space anchored):

```tsx
<Html screenPosition="top-left">
  <div className="health-bar">
    <div style={{ width: `${health}%` }} />
  </div>
</Html>
```

### Interactive UI

Buttons, menus, forms (rendered in 3D space or screen-space):

```tsx
<Html position={[0, 0, 2]}>
  <button onClick={handleClick}>Interact</button>
</Html>
```

## Text Rendering Alternative

For v1, if text is needed:
- Use HTML/CSS overlays positioned manually (outside the engine)
- Use SDF (Signed Distance Field) text rendering (separate package)
- Use texture-based text (render text to texture, display on quad)

## Implementation Priority

v2 feature. For v1, use standard HTML/CSS outside the canvas for UI.
