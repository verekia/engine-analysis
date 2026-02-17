# Asset Pipeline — GLTF, Draco, KTX2/Basis

## Overview

Lynx's asset pipeline handles loading 3D models and textures from standard formats with maximum performance:

```
                      ┌────────────────────┐
                      │   GLTFLoader        │
                      │   parse JSON/GLB    │
                      │   resolve refs      │
                      └────┬─────────┬──────┘
                           │         │
              ┌────────────▼──┐  ┌───▼──────────────┐
              │ Draco Worker  │  │ KTX2/Basis Worker │
              │ decode mesh   │  │ transcode texture │
              │ (WASM)        │  │ (WASM)            │
              └────────┬──────┘  └──────┬────────────┘
                       │                │
              ┌────────▼────────────────▼──────────┐
              │         Asset Assembler             │
              │  Create GPU buffers + textures      │
              │  Build scene graph from GLTF nodes  │
              │  Setup materials + animations       │
              └────────────────────────────────────┘
```

## GLTF Loader

### Supported Features

| Feature | Support |
|---|---|
| glTF 2.0 JSON + separate buffers | Yes |
| GLB (binary) | Yes |
| Embedded base64 buffers | Yes |
| Meshes with multiple primitives | Yes |
| Node hierarchy / scene graph | Yes |
| PBR materials (mapped to Lambert) | Yes |
| Skeletal animations | Yes |
| Morph targets | No (v1) |
| Cameras | Basic |
| Extensions: KHR_draco_mesh_compression | Yes |
| Extensions: KHR_texture_basisu (KTX2) | Yes |
| Extensions: KHR_materials_unlit (→ BasicMaterial) | Yes |

### Coordinate Conversion

GLTF uses **Y-up, right-handed**. Lynx uses **Z-up, right-handed**. On import, apply the conversion:

```typescript
// Y-up → Z-up rotation: rotate -90° around X axis
// This swaps Y and Z: (x, y, z) → (x, z, -y) but we keep handedness
const GLTF_TO_LYNX = mat4FromRotationX(-Math.PI / 2)

const convertPosition = (pos: Float32Array): void => {
    // [x, y, z] → [x, z, -y]
    for (let i = 0; i < pos.length; i += 3) {
        const y = pos[i + 1]
        pos[i + 1] = pos[i + 2]
        pos[i + 2] = -y
    }
}

const convertNormal = (norm: Float32Array): void => {
    // Same transformation as positions (direction transform)
    convertPosition(norm)
}

const convertQuaternion = (q: Float32Array): void => {
    // [x, y, z, w] → [x, z, -y, w]
    for (let i = 0; i < q.length; i += 4) {
        const y = q[i + 1]
        q[i + 1] = q[i + 2]
        q[i + 2] = -y
    }
}
```

### Loader API

```typescript
interface GLTFResult {
    scene: Group                    // root node with full hierarchy
    scenes: Group[]                 // all scenes in the file
    animations: AnimationClip[]     // all animation clips
    meshes: Mesh[]                  // flat list of all meshes
    skeletons: Skeleton[]           // all skeletons
}

interface LoaderOptions {
    dracoDecoderPath?: string       // path to draco_decoder.wasm
    ktx2TranscoderPath?: string     // path to basis_transcoder.wasm
    materialOverrides?: Record<string, Partial<MaterialConfig>>
}

const loadGLTF = async (url: string, options?: LoaderOptions): Promise<GLTFResult> => {
    const response = await fetch(url)
    const isGLB = url.endsWith('.glb')

    let json: GLTFJson
    let binaryChunk: ArrayBuffer | null = null

    if (isGLB) {
        const buffer = await response.arrayBuffer()
        const parsed = parseGLB(buffer)
        json = parsed.json
        binaryChunk = parsed.binary
    } else {
        json = await response.json()
    }

    // Resolve all buffer URIs
    const buffers = await resolveBuffers(json, binaryChunk, url)

    // Decode Draco-compressed meshes (parallel, in worker)
    const meshPromises = decodeMeshes(json, buffers, options)

    // Transcode KTX2 textures (parallel, in worker)
    const texturePromises = transcodeTextures(json, buffers, options)

    // Wait for all async decoding
    const [meshData, textureData] = await Promise.all([
        Promise.all(meshPromises),
        Promise.all(texturePromises),
    ])

    // Assemble scene graph
    return assembleScene(json, meshData, textureData)
}
```

### GLB Parsing

```typescript
const GLB_MAGIC = 0x46546C67   // 'glTF'
const GLB_VERSION = 2
const CHUNK_JSON = 0x4E4F534A  // 'JSON'
const CHUNK_BIN = 0x004E4942   // 'BIN\0'

interface GLBParsed {
    json: GLTFJson
    binary: ArrayBuffer | null
}

const parseGLB = (buffer: ArrayBuffer): GLBParsed => {
    const view = new DataView(buffer)
    const magic = view.getUint32(0, true)
    if (magic !== GLB_MAGIC) throw new Error('Not a valid GLB file')

    const version = view.getUint32(4, true)
    if (version !== GLB_VERSION) throw new Error(`Unsupported GLB version: ${version}`)

    let offset = 12 // skip header
    let json: GLTFJson | null = null
    let binary: ArrayBuffer | null = null

    while (offset < buffer.byteLength) {
        const chunkLength = view.getUint32(offset, true)
        const chunkType = view.getUint32(offset + 4, true)
        const chunkData = buffer.slice(offset + 8, offset + 8 + chunkLength)

        if (chunkType === CHUNK_JSON) {
            json = JSON.parse(new TextDecoder().decode(chunkData))
        } else if (chunkType === CHUNK_BIN) {
            binary = chunkData
        }

        offset += 8 + chunkLength
    }

    if (!json) throw new Error('GLB missing JSON chunk')
    return { json, binary }
}
```

## Draco Mesh Decompression

### Worker Architecture

Draco decoding uses a dedicated Web Worker to avoid blocking the main thread:

```typescript
// Main thread
const dracoWorkerPool = createWorkerPool('draco-worker.js', navigator.hardwareConcurrency || 4)

const decodeDracoMesh = async (
    compressedData: ArrayBuffer,
    attributes: DracoAttributeMap,
): Promise<DecodedMesh> => {
    return dracoWorkerPool.dispatch({
        type: 'decode',
        data: compressedData,
        attributes,
    })
}
```

### Worker Implementation

```typescript
// draco-worker.ts
import { DracoDecoderModule } from './draco_decoder.js'

let decoder: DracoDecoder | null = null

const init = async (wasmPath: string) => {
    const module = await DracoDecoderModule({ wasmBinary: await fetch(wasmPath).then(r => r.arrayBuffer()) })
    decoder = new module.Decoder()
}

self.onmessage = async (e: MessageEvent) => {
    if (e.data.type === 'init') {
        await init(e.data.wasmPath)
        self.postMessage({ type: 'ready' })
        return
    }

    if (e.data.type === 'decode') {
        const { data, attributes } = e.data
        const buffer = new decoder!.DecoderBuffer()
        buffer.Init(new Int8Array(data), data.byteLength)

        const geometryType = decoder!.GetEncodedGeometryType(buffer)
        let dracoGeometry: DracoMesh | DracoPointCloud

        if (geometryType === decoder!.TRIANGULAR_MESH) {
            dracoGeometry = new decoder!.Mesh()
            decoder!.DecodeBufferToMesh(buffer, dracoGeometry)
        } else {
            throw new Error('Only triangle meshes supported')
        }

        // Extract attributes
        const result: DecodedMesh = {
            indices: extractIndices(decoder!, dracoGeometry),
            positions: extractAttribute(decoder!, dracoGeometry, 'POSITION'),
        }

        if (attributes.NORMAL !== undefined) {
            result.normals = extractAttribute(decoder!, dracoGeometry, 'NORMAL')
        }
        if (attributes.TEXCOORD_0 !== undefined) {
            result.uvs = extractAttribute(decoder!, dracoGeometry, 'TEXCOORD_0')
        }
        if (attributes._materialindex !== undefined) {
            result.materialIndices = extractAttribute(decoder!, dracoGeometry, '_materialindex')
        }

        decoder!.destroy(dracoGeometry)
        buffer.destroy()

        self.postMessage({ type: 'decoded', result }, [
            result.positions.buffer,
            result.indices.buffer,
            ...(result.normals ? [result.normals.buffer] : []),
            ...(result.uvs ? [result.uvs.buffer] : []),
        ])
    }
}
```

### Worker Pool

```typescript
interface WorkerPool {
    dispatch: <T>(message: unknown) => Promise<T>
    terminate: () => void
}

const createWorkerPool = (workerUrl: string, size: number): WorkerPool => {
    const workers: Worker[] = []
    const idle: Worker[] = []
    const queue: Array<{ message: unknown, resolve: (value: unknown) => void }> = []

    for (let i = 0; i < size; i++) {
        const worker = new Worker(workerUrl, { type: 'module' })
        workers.push(worker)
        idle.push(worker)
    }

    const processQueue = () => {
        while (idle.length > 0 && queue.length > 0) {
            const worker = idle.pop()!
            const task = queue.shift()!

            worker.onmessage = (e: MessageEvent) => {
                task.resolve(e.data.result)
                idle.push(worker)
                processQueue()
            }

            worker.postMessage(task.message)
        }
    }

    return {
        dispatch: <T>(message: unknown): Promise<T> =>
            new Promise(resolve => {
                queue.push({ message, resolve: resolve as (v: unknown) => void })
                processQueue()
            }),
        terminate: () => workers.forEach(w => w.terminate()),
    }
}
```

## KTX2/Basis Universal Texture Transcoding

### Format Selection

Basis Universal can transcode to various GPU-compressed formats. The best format depends on the device:

```typescript
interface TextureFormatSupport {
    astc: boolean   // ASTC (mobile — iOS, modern Android)
    bc7: boolean    // BC7 (desktop — Windows, macOS, Linux)
    etc2: boolean   // ETC2 (older Android, WebGL2 fallback)
    s3tc: boolean   // BC1/BC3 (older desktop fallback)
}

const detectTextureFormats = (device: GalDevice): TextureFormatSupport => ({
    astc: device.supportsFormat('astc-4x4-unorm'),
    bc7: device.supportsFormat('bc7-rgba-unorm'),
    etc2: device.supportsFormat('etc2-rgba8unorm'),
    s3tc: device.supportsFormat('bc1-rgba-unorm'),
})

const selectTranscodeTarget = (support: TextureFormatSupport, hasAlpha: boolean): BasisFormat => {
    if (support.astc) return 'ASTC_4x4'            // best quality, mobile
    if (support.bc7) return 'BC7'                    // best quality, desktop
    if (hasAlpha) {
        if (support.s3tc) return 'BC3'               // alpha, desktop fallback
        if (support.etc2) return 'ETC2_RGBA'          // alpha, mobile fallback
    } else {
        if (support.s3tc) return 'BC1'               // no alpha, desktop
        if (support.etc2) return 'ETC2_RGB'          // no alpha, mobile
    }
    return 'RGBA32'                                   // uncompressed fallback
}
```

### KTX2 Worker

```typescript
// ktx2-worker.ts
import { BasisModule } from './basis_transcoder.js'

let transcoder: BasisTranscoder | null = null

self.onmessage = async (e: MessageEvent) => {
    if (e.data.type === 'init') {
        const module = await BasisModule({
            wasmBinary: await fetch(e.data.wasmPath).then(r => r.arrayBuffer()),
        })
        module.initializeBasis()
        transcoder = new module.BasisFile()
        self.postMessage({ type: 'ready' })
        return
    }

    if (e.data.type === 'transcode') {
        const { ktx2Data, targetFormat } = e.data

        // Parse KTX2 container
        const ktx2 = parseKTX2(new Uint8Array(ktx2Data))

        // Open Basis data
        transcoder!.open(ktx2.basisData)
        const images = transcoder!.getNumImages()
        const levels = transcoder!.getNumLevels(0)
        const hasAlpha = transcoder!.getHasAlpha()

        const mipmaps: TranscodedLevel[] = []

        for (let level = 0; level < levels; level++) {
            const width = transcoder!.getImageWidth(0, level)
            const height = transcoder!.getImageHeight(0, level)
            const size = transcoder!.getImageTranscodedSizeInBytes(0, level, targetFormat)
            const data = new Uint8Array(size)

            if (!transcoder!.transcodeImage(data, 0, level, targetFormat, 0, 0)) {
                throw new Error(`Basis transcode failed at level ${level}`)
            }

            mipmaps.push({ width, height, data })
        }

        transcoder!.close()

        self.postMessage(
            { type: 'transcoded', mipmaps, hasAlpha },
            mipmaps.map(m => m.data.buffer),
        )
    }
}
```

## Standard Texture Loading

For non-KTX2 textures (PNG, JPEG):

```typescript
const loadTexture = async (
    device: GalDevice,
    url: string,
    options?: { generateMipmaps?: boolean, sRGB?: boolean },
): Promise<GalTexture> => {
    const response = await fetch(url)
    const blob = await response.blob()
    const bitmap = await createImageBitmap(blob, {
        colorSpaceConversion: 'none',
        premultiplyAlpha: 'none',
    })

    const texture = device.createTexture({
        width: bitmap.width,
        height: bitmap.height,
        format: options?.sRGB ? 'rgba8unorm-srgb' : 'rgba8unorm',
        usage: 'sampled' | 'copy-dst',
        mipLevels: options?.generateMipmaps ? Math.floor(Math.log2(Math.max(bitmap.width, bitmap.height))) + 1 : 1,
    })

    device.writeTexture(texture, bitmap)

    if (options?.generateMipmaps) {
        device.generateMipmaps(texture)
    }

    return texture
}
```

## AO Texture Support

Ambient Occlusion textures are loaded as single-channel (R8) textures:

```typescript
const loadAOTexture = async (device: GalDevice, url: string): Promise<GalTexture> =>
    loadTexture(device, url, { sRGB: false, generateMipmaps: true })

// In the shader, AO modulates ambient lighting:
// vec3 ambient = uAmbientColor * uAmbientIntensity * texture(uAOMap, vUV).r
```

## Scene Assembly

After all meshes and textures are decoded, assemble the GLTF scene graph into Lynx nodes:

```typescript
const assembleScene = (
    json: GLTFJson,
    meshData: DecodedMesh[],
    textureData: GalTexture[],
): GLTFResult => {
    // Build materials
    const materials = (json.materials ?? []).map((mat, i) => {
        const config: MaterialConfig = {
            color: mat.pbrMetallicRoughness?.baseColorFactor?.slice(0, 3) ?? [1, 1, 1],
            opacity: mat.pbrMetallicRoughness?.baseColorFactor?.[3] ?? 1,
            transparent: (mat.alphaMode === 'BLEND'),
        }

        // Map PBR base color texture → color texture
        const texIdx = mat.pbrMetallicRoughness?.baseColorTexture?.index
        if (texIdx !== undefined) {
            config.colorTexture = textureData[texIdx]
        }

        // AO texture
        const aoIdx = mat.occlusionTexture?.index
        if (aoIdx !== undefined) {
            config.aoTexture = textureData[aoIdx]
        }

        // Unlit → BasicMaterial, otherwise → LambertMaterial
        if (mat.extensions?.KHR_materials_unlit) {
            return createBasicMaterial(config)
        }
        return createLambertMaterial(config)
    })

    // Build node hierarchy
    const nodes = json.nodes.map(node => {
        const group = createGroup()
        if (node.translation) {
            group.position.set(
                node.translation[0],
                node.translation[2],   // Y→Z swap
                -node.translation[1],
            )
        }
        if (node.rotation) {
            group.quaternion.set(
                node.rotation[0],
                node.rotation[2],
                -node.rotation[1],
                node.rotation[3],
            )
        }
        if (node.scale) {
            group.scale.set(node.scale[0], node.scale[2], node.scale[1])
        }
        return group
    })

    // Parent-child relationships
    json.nodes.forEach((node, i) => {
        if (node.children) {
            for (const childIdx of node.children) {
                nodes[i].add(nodes[childIdx])
            }
        }
    })

    // Attach meshes to nodes
    json.nodes.forEach((node, i) => {
        if (node.mesh !== undefined) {
            const gltfMesh = json.meshes[node.mesh]
            for (const prim of gltfMesh.primitives) {
                const geometry = createBufferGeometry(meshData[prim.index])
                const material = materials[prim.material ?? 0]
                const mesh = createMesh(geometry, material)
                nodes[i].add(mesh)
            }
        }
    })

    // Build skeletons and skinned meshes
    const skeletons = buildSkeletons(json, nodes)

    // Build animations
    const animations = buildAnimations(json)

    // Root scene
    const rootScene = createGroup()
    const sceneDef = json.scenes[json.scene ?? 0]
    for (const nodeIdx of sceneDef.nodes) {
        rootScene.add(nodes[nodeIdx])
    }

    return {
        scene: rootScene,
        scenes: json.scenes.map(s => {
            const g = createGroup()
            for (const n of s.nodes) g.add(nodes[n])
            return g
        }),
        animations,
        meshes: nodes.flatMap(n => n.getChildrenOfType('mesh')),
        skeletons,
    }
}
```

## Loading API Usage

```typescript
// Basic GLTF loading
const gltf = await loadGLTF('/models/character.glb', {
    dracoDecoderPath: '/wasm/draco_decoder.wasm',
    ktx2TranscoderPath: '/wasm/basis_transcoder.wasm',
})

scene.add(gltf.scene)

// Play animation
const mixer = createAnimationMixer(gltf.scene)
mixer.play(gltf.animations[0])

// Access specific meshes
const sword = gltf.meshes.find(m => m.name === 'Sword')
```

## Caching and Memory

- **Geometry sharing**: Identical geometries referenced by multiple nodes share the same GPU buffers
- **Texture sharing**: Textures referenced by multiple materials are loaded once
- **Worker reuse**: Draco and KTX2 workers persist across loads (WASM initialization is one-time)
- **Dispose**: `gltf.dispose()` releases all GPU resources (buffers, textures) associated with the loaded model
