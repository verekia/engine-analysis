# React Bindings (lynx-react)

## Overview

`lynx-react` provides a declarative React API for Lynx, inspired by React Three Fiber (R3F). It uses a custom React reconciler to map JSX elements to Lynx scene graph objects, enabling idiomatic React patterns for 3D scenes.

```tsx
import { Canvas, useFrame } from 'lynx-react'

const App = () => (
    <Canvas camera={{ position: [5, 5, 3], fov: 60 }}>
        <ambientLight intensity={0.3} />
        <directionalLight position={[10, 10, 10]} castShadow />
        <mesh position={[0, 0, 1]}>
            <boxGeometry args={[1, 1, 1]} />
            <lambertMaterial color="tomato" />
        </mesh>
        <OrbitControls />
    </Canvas>
)
```

## Architecture

```
┌───────────────────────────────────────────────┐
│ React Application                              │
│  <Canvas>                                      │
│    <mesh> <group> <directionalLight> ...       │
├───────────────────────────────────────────────┤
│ lynx-react Reconciler (react-reconciler)       │
│  Maps JSX → Lynx objects                       │
│  Handles props → mutations                     │
│  Manages lifecycle (mount/update/unmount)       │
├───────────────────────────────────────────────┤
│ Lynx Core                                      │
│  Scene graph, renderer, materials, etc.        │
└───────────────────────────────────────────────┘
```

## The `<Canvas>` Component

The root component that creates the Lynx renderer, scene, camera, and render loop:

```typescript
interface CanvasProps {
    children: React.ReactNode
    camera?: {
        position?: [number, number, number]
        target?: [number, number, number]
        fov?: number
        near?: number
        far?: number
    }
    shadows?: boolean | { cascades?: number, mapSize?: number }
    postProcess?: {
        bloom?: Partial<BloomConfig>
        toneMapping?: 'aces' | 'reinhard' | 'none'
        exposure?: number
    }
    style?: React.CSSProperties
    className?: string
    onCreated?: (state: LynxState) => void
    frameloop?: 'always' | 'demand'  // default 'always'
    dpr?: number | [number, number]  // device pixel ratio
}

const Canvas: React.FC<CanvasProps> = ({ children, camera, shadows, postProcess, ...props }) => {
    const containerRef = useRef<HTMLDivElement>(null)
    const stateRef = useRef<LynxState>()

    useEffect(() => {
        const container = containerRef.current!
        const canvas = document.createElement('canvas')
        container.appendChild(canvas)

        // Initialize Lynx
        const renderer = createRenderer(canvas, {
            shadows: shadows !== false,
            postProcess,
        })
        const scene = createScene()
        const cam = createPerspectiveCamera({
            fov: camera?.fov ?? 60,
            near: camera?.near ?? 0.1,
            far: camera?.far ?? 500,
        })
        if (camera?.position) vec3Set(cam.position, ...camera.position)

        const state: LynxState = {
            renderer,
            scene,
            camera: cam,
            canvas,
            size: { width: container.clientWidth, height: container.clientHeight },
            clock: { elapsed: 0, delta: 0 },
        }
        stateRef.current = state

        // Start reconciler
        const root = reconciler.createContainer(state, 0, null, false, null, '', null, null)
        reconciler.updateContainer(
            <LynxContext.Provider value={state}>{children}</LynxContext.Provider>,
            root,
            null,
            () => {},
        )

        // Render loop
        let lastTime = performance.now()
        const loop = (now: number) => {
            state.clock.delta = (now - lastTime) / 1000
            state.clock.elapsed += state.clock.delta
            lastTime = now

            // Run useFrame callbacks
            for (const cb of frameCallbacks) cb(state, state.clock.delta)

            renderer.render(scene, cam)
            state.animationFrame = requestAnimationFrame(loop)
        }
        state.animationFrame = requestAnimationFrame(loop)

        // Resize observer
        const ro = new ResizeObserver(([entry]) => {
            const { width, height } = entry.contentRect
            renderer.setSize(width, height)
            cam.aspect = width / height
            cam.updateProjectionMatrix()
            state.size = { width, height }
        })
        ro.observe(container)

        props.onCreated?.(state)

        return () => {
            cancelAnimationFrame(state.animationFrame!)
            ro.disconnect()
            reconciler.updateContainer(null, root, null, () => {})
            renderer.dispose()
            container.removeChild(canvas)
        }
    }, [])

    // Re-render reconciler on children change
    useEffect(() => {
        if (!stateRef.current) return
        // Reconciler handles diffing automatically
    })

    return <div ref={containerRef} style={{ width: '100%', height: '100%', ...props.style }} className={props.className} />
}
```

## Custom Reconciler

The reconciler maps React element types to Lynx objects:

```typescript
import Reconciler from 'react-reconciler'

// Element type → Lynx constructor mapping
const catalogue: Record<string, (...args: unknown[]) => unknown> = {
    mesh: createMesh,
    group: createGroup,
    skinnedMesh: createSkinnedMesh,
    directionalLight: createDirectionalLight,
    ambientLight: () => null, // handled as scene property
    perspectiveCamera: createPerspectiveCamera,

    // Geometries (attached to parent mesh as .geometry)
    boxGeometry: createBoxGeometry,
    sphereGeometry: createSphereGeometry,
    planeGeometry: createPlaneGeometry,
    cylinderGeometry: createCylinderGeometry,
    coneGeometry: createConeGeometry,
    capsuleGeometry: createCapsuleGeometry,
    circleGeometry: createCircleGeometry,

    // Materials (attached to parent mesh as .material)
    basicMaterial: createBasicMaterial,
    lambertMaterial: createLambertMaterial,
}

const reconciler = Reconciler({
    supportsMutation: true,
    supportsPersistence: false,

    createInstance(type: string, props: Record<string, unknown>, _root: LynxState) {
        const constructor = catalogue[type]
        if (!constructor) throw new Error(`Unknown element type: ${type}`)

        // Handle 'args' prop (constructor arguments)
        const args = (props.args as unknown[]) ?? []
        const instance = constructor(...args)

        // Apply props
        applyProps(instance, props)

        return instance
    },

    createTextInstance() {
        throw new Error('Text nodes not supported in Lynx scene')
    },

    appendInitialChild(parent: LynxNode, child: LynxNode) {
        attachChild(parent, child)
    },

    appendChild(parent: LynxNode, child: LynxNode) {
        attachChild(parent, child)
    },

    removeChild(parent: LynxNode, child: LynxNode) {
        detachChild(parent, child)
    },

    insertBefore(parent: LynxNode, child: LynxNode, before: LynxNode) {
        attachChild(parent, child, before)
    },

    prepareUpdate(_instance: unknown, _type: string, oldProps: Record<string, unknown>, newProps: Record<string, unknown>) {
        // Return changed props
        const changes: Array<[string, unknown]> = []
        for (const key of Object.keys(newProps)) {
            if (key === 'children' || key === 'args') continue
            if (oldProps[key] !== newProps[key]) {
                changes.push([key, newProps[key]])
            }
        }
        return changes.length > 0 ? changes : null
    },

    commitUpdate(instance: unknown, updatePayload: Array<[string, unknown]>) {
        for (const [key, value] of updatePayload) {
            applyProp(instance, key, value)
        }
    },

    appendChildToContainer(container: LynxState, child: LynxNode) {
        if (isSceneNode(child)) container.scene.add(child)
    },

    removeChildFromContainer(container: LynxState, child: LynxNode) {
        if (isSceneNode(child)) container.scene.remove(child)
    },

    // ... other required methods (getRootHostContext, etc.)
})
```

### Child Attachment Logic

Geometries and materials attach to their parent mesh as properties, not as scene children:

```typescript
const attachChild = (parent: LynxNode, child: LynxNode, before?: LynxNode) => {
    if (isGeometry(child)) {
        // Geometry attaches as parent.geometry
        parent.geometry = child
    } else if (isMaterial(child)) {
        // Material attaches as parent.material
        parent.material = child
    } else if (isSceneNode(child)) {
        // Scene nodes attach as children
        if (before) {
            parent.addBefore(child, before)
        } else {
            parent.add(child)
        }
    }
}
```

## Prop Application

```typescript
const applyProp = (instance: unknown, key: string, value: unknown) => {
    if (key === 'position' && Array.isArray(value)) {
        vec3Set(instance.position, value[0], value[1], value[2])
    } else if (key === 'rotation' && Array.isArray(value)) {
        // Euler angles [x, y, z] → quaternion
        quatFromEuler(instance.quaternion, value[0], value[1], value[2])
    } else if (key === 'scale' && Array.isArray(value)) {
        vec3Set(instance.scale, value[0], value[1], value[2])
    } else if (key === 'scale' && typeof value === 'number') {
        vec3Set(instance.scale, value, value, value)
    } else if (key === 'color') {
        instance.color = parseColor(value)
    } else if (key === 'castShadow' || key === 'receiveShadow') {
        instance[key] = value
    } else if (key === 'visible') {
        instance.visible = value
    } else if (key === 'onClick' || key === 'onPointerOver' || key === 'onPointerOut') {
        // Event handlers stored for raycaster
        instance.__handlers = instance.__handlers ?? {}
        instance.__handlers[key] = value
    } else {
        // Generic prop assignment
        instance[key] = value
    }
}
```

## Hooks

### useFrame

Called every frame with the current state and delta time:

```typescript
type FrameCallback = (state: LynxState, delta: number) => void

const frameCallbacks = new Set<FrameCallback>()

const useFrame = (callback: FrameCallback, priority = 0) => {
    const ref = useRef(callback)
    ref.current = callback

    useEffect(() => {
        const wrappedCb: FrameCallback = (state, delta) => ref.current(state, delta)
        frameCallbacks.add(wrappedCb)
        return () => { frameCallbacks.delete(wrappedCb) }
    }, [])
}

// Usage
const SpinningBox = () => {
    const meshRef = useRef<Mesh>(null)
    useFrame((_, delta) => {
        if (meshRef.current) {
            meshRef.current.rotation[2] += delta // rotate around Z
        }
    })
    return (
        <mesh ref={meshRef}>
            <boxGeometry args={[1, 1, 1]} />
            <lambertMaterial color="royalblue" />
        </mesh>
    )
}
```

### useLynx

Access the Lynx state (renderer, scene, camera, etc.):

```typescript
const LynxContext = createContext<LynxState | null>(null)

const useLynx = (): LynxState => {
    const ctx = useContext(LynxContext)
    if (!ctx) throw new Error('useLynx must be used within <Canvas>')
    return ctx
}

// Usage
const CameraLogger = () => {
    const { camera } = useLynx()
    useFrame(() => {
        console.log('Camera at:', camera.position)
    })
    return null
}
```

### useLoader

Load assets with caching and suspense support:

```typescript
const loaderCache = new Map<string, Promise<unknown>>()

const useLoader = <T,>(
    loaderFn: (url: string, options?: unknown) => Promise<T>,
    url: string,
    options?: unknown,
): T => {
    const key = `${url}:${JSON.stringify(options)}`

    if (!loaderCache.has(key)) {
        loaderCache.set(key, loaderFn(url, options))
    }

    const promise = loaderCache.get(key)!
    // Throw promise for Suspense
    if (promise instanceof Promise) {
        throw promise.then(result => {
            loaderCache.set(key, result as Promise<unknown>)
        })
    }

    return promise as T
}

// Usage with Suspense
const Character = () => {
    const gltf = useLoader(loadGLTF, '/models/character.glb')
    return <primitive object={gltf.scene} />
}

const App = () => (
    <Canvas>
        <Suspense fallback={null}>
            <Character />
        </Suspense>
    </Canvas>
)
```

### useAnimations

Manage skeletal animations:

```typescript
const useAnimations = (gltf: GLTFResult, rootRef?: RefObject<Group>) => {
    const mixerRef = useRef<AnimationMixer>()

    useEffect(() => {
        const root = rootRef?.current ?? gltf.scene
        mixerRef.current = createAnimationMixer(root)
        return () => mixerRef.current?.dispose()
    }, [gltf])

    useFrame((_, delta) => {
        mixerRef.current?.update(delta)
    })

    const actions = useMemo(() => {
        const map: Record<string, AnimationAction> = {}
        for (const clip of gltf.animations) {
            map[clip.name] = mixerRef.current!.clipAction(clip)
        }
        return map
    }, [gltf])

    return { mixer: mixerRef.current, actions }
}

// Usage
const AnimatedCharacter = () => {
    const gltf = useLoader(loadGLTF, '/models/character.glb')
    const { actions } = useAnimations(gltf)

    useEffect(() => {
        actions['Idle']?.play()
    }, [actions])

    return <primitive object={gltf.scene} />
}
```

## Event System

Pointer events are handled via raycasting on the canvas:

```typescript
const setupEvents = (state: LynxState) => {
    const raycaster = createRaycaster()

    const getInteractableMeshes = (): Mesh[] => {
        const meshes: Mesh[] = []
        state.scene.traverse(node => {
            if (isMesh(node) && node.__handlers) {
                meshes.push(node)
            }
        })
        return meshes
    }

    let hoveredMesh: Mesh | null = null

    state.canvas.addEventListener('pointermove', (e: PointerEvent) => {
        const ray = state.camera.screenToRay(e.offsetX, e.offsetY)
        const hit = raycaster.intersectFirst(ray, getInteractableMeshes())
        const mesh = hit?.mesh ?? null

        if (mesh !== hoveredMesh) {
            hoveredMesh?.__handlers?.onPointerOut?.({ mesh: hoveredMesh })
            mesh?.__handlers?.onPointerOver?.({ mesh, point: hit!.point })
            hoveredMesh = mesh
        }
    })

    state.canvas.addEventListener('click', (e: MouseEvent) => {
        const ray = state.camera.screenToRay(e.offsetX, e.offsetY)
        const hit = raycaster.intersectFirst(ray, getInteractableMeshes())
        hit?.mesh?.__handlers?.onClick?.({ mesh: hit.mesh, point: hit.point })
    })
}
```

## Custom Components with extend()

Register custom Lynx objects for use in JSX:

```typescript
const extend = (components: Record<string, (...args: unknown[]) => unknown>) => {
    Object.assign(catalogue, components)
}

// Usage
import { createCustomEffect } from './my-effect'
extend({ customEffect: createCustomEffect })

// Now usable in JSX
const Scene = () => (
    <Canvas>
        <customEffect intensity={0.5} />
    </Canvas>
)
```

## The `<primitive>` Element

Mount an existing Lynx object (e.g., from GLTF loader) into the React tree:

```tsx
// Wraps an existing Lynx object without recreating it
const Primitive = ({ object, ...props }) => {
    // The reconciler recognizes 'primitive' type and uses the object directly
    // Props are applied to the existing object
    return <primitive object={object} {...props} />
}

// Usage
const LoadedModel = () => {
    const gltf = useLoader(loadGLTF, '/models/scene.glb')
    return <primitive object={gltf.scene} position={[0, 0, 0]} />
}
```

## Complete Example

```tsx
import { Canvas, useFrame, useLoader, OrbitControls, Html } from 'lynx-react'
import { loadGLTF } from 'lynx'

const Character = () => {
    const gltf = useLoader(loadGLTF, '/models/knight.glb')
    const { actions } = useAnimations(gltf)
    const ref = useRef<Group>(null)

    useEffect(() => { actions['Run']?.play() }, [actions])

    return (
        <group ref={ref}>
            <primitive object={gltf.scene} />
            <Html position={[0, 0, 2.5]} anchor={[0.5, 1]}>
                <div className="name-tag">Sir Lancelot</div>
            </Html>
        </group>
    )
}

const Ground = () => (
    <mesh rotation={[0, 0, 0]} receiveShadow>
        <planeGeometry args={[200, 200]} />
        <lambertMaterial color="#5a8a3c" />
    </mesh>
)

const App = () => (
    <Canvas
        shadows
        camera={{ position: [10, 10, 8], fov: 50 }}
        postProcess={{ bloom: { enabled: true, threshold: 0.9, strength: 0.5 } }}
    >
        <ambientLight intensity={0.3} color="#b4c6e0" />
        <directionalLight
            position={[20, 30, 25]}
            intensity={1.2}
            castShadow
            shadow-mapSize={2048}
        />
        <Suspense fallback={null}>
            <Character />
        </Suspense>
        <Ground />
        <OrbitControls target={[0, 0, 1]} maxPolarAngle={Math.PI * 0.45} />
    </Canvas>
)
```

## Package Exports

```typescript
// lynx-react/src/index.ts
export { Canvas } from './Canvas'
export { useFrame } from './hooks/useFrame'
export { useLynx } from './hooks/useLynx'
export { useLoader } from './hooks/useLoader'
export { useAnimations } from './hooks/useAnimations'
export { extend } from './reconciler'
export { OrbitControls } from './components/OrbitControls'
export { Html } from './components/Html'
```
