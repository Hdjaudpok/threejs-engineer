# 3D Pipeline & Asset Management

## Blender-to-Web Pipeline

### Export Settings (Blender → glTF/GLB)

1. **Format**: glTF Binary (.glb) for single-file convenience
2. **Apply transforms** before export (Ctrl+A → All Transforms)
3. **Polycount targets**:
   - Hero model: 10k-50k triangles
   - Background props: 500-5k triangles
   - Total scene: < 200k triangles for mobile
4. **Bake lighting** into lightmaps/vertex colors when possible
5. **Normal maps** instead of high-poly detail (bake from high-poly)
6. **Texture packing**: combine metalness + roughness into single texture (glTF convention: green = roughness, blue = metalness)

### Optimization Checklist

- [ ] Remove hidden/invisible faces
- [ ] Decimate where possible without visual loss
- [ ] Apply modifiers before export
- [ ] Remove unused materials and textures
- [ ] Check scale (1 unit = 1 meter in Three.js)
- [ ] Clean up mesh names for easy traversal in code
- [ ] Bake AO / lightmaps for static objects

## Compression

### Draco (Geometry Compression)

Reduces geometry file size by 50-90%.

```ts
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js'

const dracoLoader = new DRACOLoader()
dracoLoader.setDecoderPath('/draco/')
// Use CDN: 'https://www.gstatic.com/draco/versioned/decoders/1.5.7/'

const gltfLoader = new GLTFLoader()
gltfLoader.setDRACOLoader(dracoLoader)
```

CLI compression with gltf-transform:
```bash
npx gltf-transform draco input.glb output.glb
```

### KTX2/Basis (Texture Compression)

GPU-compressed textures, 4-6x smaller, faster GPU upload.

```ts
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js'

const ktx2Loader = new KTX2Loader()
ktx2Loader.setTranscoderPath('/basis/')
ktx2Loader.detectSupport(renderer)

// Use with glTF
gltfLoader.setKTX2Loader(ktx2Loader)
```

CLI conversion:
```bash
npx gltf-transform ktx2 input.glb output.glb --slots "baseColor,normal,emissive"
```

### Full Optimization Pipeline

```bash
# Install gltf-transform
npm install -g @gltf-transform/cli

# Optimize: dedup, compress textures, compress geometry
npx gltf-transform optimize input.glb output.glb
# Or step by step:
npx gltf-transform dedup input.glb temp.glb
npx gltf-transform ktx2 temp.glb temp2.glb
npx gltf-transform draco temp2.glb output.glb
```

## Asset Folder Structure

```
public/
├── assets/
│   ├── models/
│   │   ├── hero.glb
│   │   ├── environment.glb
│   │   └── props/
│   │       ├── chair.glb
│   │       └── table.glb
│   ├── textures/
│   │   ├── matcap-gold.png
│   │   ├── noise.png
│   │   └── gradient.png
│   ├── envMaps/
│   │   ├── studio.hdr
│   │   └── sunset.hdr
│   └── draco/
│       └── (draco decoder files)
```

## Loading Manager with Progress

### Basic Loading Overlay

```ts
const loadingManager = new THREE.LoadingManager()
const overlay = document.querySelector<HTMLElement>('.loading-overlay')!
const progressBar = document.querySelector<HTMLElement>('.loading-bar')!

loadingManager.onProgress = (url, loaded, total) => {
  const progress = loaded / total
  progressBar.style.transform = `scaleX(${progress})`
}

loadingManager.onLoad = () => {
  // All assets loaded - transition to scene
  gsap.to(overlay, {
    opacity: 0,
    duration: 0.8,
    onComplete: () => {
      overlay.style.display = 'none'
      startExperience()
    },
  })
}

loadingManager.onError = (url) => {
  console.error(`Failed to load: ${url}`)
}

// Pass manager to all loaders
const gltfLoader = new GLTFLoader(loadingManager)
const textureLoader = new THREE.TextureLoader(loadingManager)
```

### Loading Overlay HTML/CSS

```html
<div class="loading-overlay">
  <div class="loading-content">
    <p class="loading-text">Loading...</p>
    <div class="loading-bar-container">
      <div class="loading-bar"></div>
    </div>
  </div>
</div>
```

```css
.loading-overlay {
  position: fixed;
  inset: 0;
  background: #000;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
  transition: opacity 0.5s;
}

.loading-bar-container {
  width: 200px;
  height: 2px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 1px;
  overflow: hidden;
}

.loading-bar {
  width: 100%;
  height: 100%;
  background: #fff;
  transform-origin: left;
  transform: scaleX(0);
  transition: transform 0.3s;
}
```

### Advanced Resource Loader Class

```ts
interface AssetDefinition {
  name: string
  type: 'gltf' | 'texture' | 'envMap' | 'hdr'
  path: string
}

class ResourceLoader {
  private manager: THREE.LoadingManager
  private gltfLoader: GLTFLoader
  private textureLoader: THREE.TextureLoader
  private rgbeLoader: RGBELoader
  private assets: Map<string, any> = new Map()

  constructor(onProgress?: (progress: number) => void) {
    this.manager = new THREE.LoadingManager()
    this.manager.onProgress = (_, loaded, total) => {
      onProgress?.(loaded / total)
    }

    this.gltfLoader = new GLTFLoader(this.manager)
    this.textureLoader = new THREE.TextureLoader(this.manager)
    this.rgbeLoader = new RGBELoader(this.manager)
  }

  async loadAll(definitions: AssetDefinition[]): Promise<Map<string, any>> {
    const promises = definitions.map((def) => this.loadAsset(def))
    await Promise.all(promises)
    return this.assets
  }

  private async loadAsset(def: AssetDefinition): Promise<void> {
    return new Promise((resolve, reject) => {
      switch (def.type) {
        case 'gltf':
          this.gltfLoader.load(def.path, (gltf) => {
            this.assets.set(def.name, gltf)
            resolve()
          }, undefined, reject)
          break
        case 'texture':
          this.textureLoader.load(def.path, (texture) => {
            this.assets.set(def.name, texture)
            resolve()
          }, undefined, reject)
          break
        case 'hdr':
          this.rgbeLoader.load(def.path, (envMap) => {
            this.assets.set(def.name, envMap)
            resolve()
          }, undefined, reject)
          break
      }
    })
  }

  get<T>(name: string): T {
    return this.assets.get(name) as T
  }
}
```

## LOD Strategies

### Manual LOD

```ts
const lod = new THREE.LOD()

// High detail (close)
lod.addLevel(highDetailMesh, 0)
// Medium detail
lod.addLevel(mediumDetailMesh, 15)
// Low detail (far)
lod.addLevel(lowDetailMesh, 30)

scene.add(lod)
// LOD updates automatically based on camera distance
```

### Dynamic Detail

```ts
// Reduce particle count on mobile
const isMobile = /Android|iPhone|iPad/i.test(navigator.userAgent)
const particleCount = isMobile ? 5000 : 50000

// Skip expensive effects on mobile
if (!isMobile) {
  composer.addPass(ssaoPass)
}
```

## Environment Maps

```ts
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js'

const rgbeLoader = new RGBELoader()
rgbeLoader.load('/assets/envMaps/studio.hdr', (envMap) => {
  envMap.mapping = THREE.EquirectangularReflectionMapping
  scene.environment = envMap  // Apply to all PBR materials
  scene.background = envMap   // Optional: use as background
})
```
