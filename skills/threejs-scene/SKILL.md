---
name: Three.js Scene Setup
description: This skill should be used when the user asks to "set up a Three.js scene", "create a Three.js renderer", "configure WebGLRenderer", "add lights to a Three.js scene", "set up Three.js camera", "configure shadows in Three.js", "handle window resize in Three.js", "add fog to Three.js scene", "Three.js environment map", "Three.js boilerplate", "Three.js render loop", or "set up Three.js project from scratch". Provides expert guidance on WebGLRenderer, scene hierarchy, cameras, lighting, shadows, resize handling, and render loops for Three.js r160+ ESM.
version: 0.1.0
---

# Three.js Scene Setup

## Overview

This skill covers the foundational setup of a Three.js r160+ ESM application: WebGLRenderer, scene, cameras, lighting, shadows, environment maps, resize handling, and render loops. All code uses ESM syntax — `import * as THREE from 'three'` and addons from `'three/addons/...'`.

## Renderer Setup

```js
import * as THREE from 'three';

const renderer = new THREE.WebGLRenderer({
  antialias: true,
  powerPreference: 'high-performance',
});
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // cap at 2× — never uncapped
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);
```

**Critical rules:**
- `shadowMap.enabled` must be set **before** the first `render()` call — it cannot be toggled after.
- `outputColorSpace = THREE.SRGBColorSpace` must be set explicitly — textures loaded by `TextureLoader` are SRGB by default, but uniforms are Linear; mismatched color spaces cause washed-out or over-saturated rendering.
- Always cap `devicePixelRatio` at 2. Values of 3–4× (common on modern phones) cause severe GPU load with negligible visual improvement.
- For `canvas`-embedded renderers, pass the `canvas` element: `new THREE.WebGLRenderer({ canvas: el, antialias: true })`.

## Scene & Background

```js
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);

// Linear fog (starts at near, fully opaque at far)
scene.fog = new THREE.Fog(0x1a1a2e, 10, 100);

// Exponential-squared fog (more physically accurate)
scene.fog = new THREE.FogExp2(0x1a1a2e, 0.05);
```

Environment maps provide both IBL (image-based lighting) for PBR materials and a visible background. Load HDR with `RGBELoader` and convert with `PMREMGenerator`:

```js
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

const pmrem = new THREE.PMREMGenerator(renderer);
pmrem.compileEquirectangularShader();

new RGBELoader().load('/envmap.hdr', (hdr) => {
  const envMap = pmrem.fromEquirectangular(hdr).texture;
  scene.environment = envMap;  // IBL for all MeshStandardMaterial
  scene.background = envMap;   // visible skybox
  hdr.dispose();
  pmrem.dispose();
});
```

## Cameras

For the full camera API and frustum culling notes, see `references/camera-types.md`.

```js
// 3D perspective (most scenes)
const camera = new THREE.PerspectiveCamera(
  75,                                      // fov in degrees
  window.innerWidth / window.innerHeight,  // aspect ratio
  0.1,                                     // near — avoid < 0.01 to prevent z-fighting
  1000                                     // far
);
camera.position.set(0, 2, 5);
camera.lookAt(0, 0, 0);

// 2D / isometric orthographic
const half = window.innerWidth / window.innerHeight;
const ortho = new THREE.OrthographicCamera(-half, half, 1, -1, 0.1, 100);
```

Always call `camera.updateProjectionMatrix()` after changing `aspect`, `fov`, `near`, or `far`.

## Lighting Rigs

### Minimal Realistic Rig (PBR)

```js
// Hemisphere light — sky/ground gradient, essentially free ambient
const hemi = new THREE.HemisphereLight(0xffffff, 0x444444, 1.0);
scene.add(hemi);

// Key directional light with shadows
const sun = new THREE.DirectionalLight(0xfff4e0, 2.0);
sun.position.set(5, 10, 5);
sun.castShadow = true;
sun.shadow.mapSize.set(2048, 2048);
sun.shadow.camera.near = 0.1;
sun.shadow.camera.far = 50;
sun.shadow.camera.left   = -10;
sun.shadow.camera.right  =  10;
sun.shadow.camera.top    =  10;
sun.shadow.camera.bottom = -10;
sun.shadow.bias = -0.001; // prevents shadow acne — tune per scene scale
scene.add(sun);
```

**Shadow frustum**: Tightly fit `left/right/top/bottom/far` to the shadow-receiving area. An oversized frustum wastes the entire shadow map resolution — a 2048×2048 map covering 100 m² gives the same effective resolution as a 256×256 map covering 10 m².

### Point & Spot Lights

```js
// PointLight — omnidirectional point source
const point = new THREE.PointLight(0xff6600, 3, 20, 2);
// color, intensity, distance (0=infinite), decay (2=physically correct)
point.castShadow = true;
point.shadow.mapSize.set(512, 512); // 6-face cube map; keep small for perf

// SpotLight — cone-shaped directional source
const spot = new THREE.SpotLight(0xffffff, 5, 30, Math.PI / 6, 0.3, 2);
// color, intensity, distance, angle, penumbra (0–1), decay
spot.position.set(2, 8, 2);
spot.target.position.set(0, 0, 0);
scene.add(spot, spot.target); // must add both mesh and target to scene
```

`decay = 2` matches the physical inverse-square falloff (Three.js default since r155 when `useLegacyLights = false`).

For complete patterns including `RectAreaLight` and `LightProbe`, see `references/lighting-guide.md`.

## Resize Handler

```js
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  // If using EffectComposer, also call: composer.setSize(w, h)
});
```

For canvas-embedded renderers, observe the parent element with `ResizeObserver` instead of `window.resize`.

## Render Loop

```js
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();     // seconds since last frame — use for motion
  const elapsed = clock.getElapsedTime(); // total seconds — use for oscillation

  // controls.update(delta)  — if using DampingControls
  // mixer.update(delta)     — if using AnimationMixer
  // rapierWorld.step()      — if using physics

  renderer.render(scene, camera);
}
animate();
```

Use `clock.getDelta()` for all frame-rate-independent animation. Never use fixed multipliers like `object.rotation.y += 0.01` — this ties rotation speed to the monitor refresh rate.

## Mesh Shadows

Enable both `castShadow` and `receiveShadow` on meshes:

```js
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(20, 20),
  new THREE.MeshStandardMaterial({ color: 0x888888 })
);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);

const box = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x4488ff })
);
box.castShadow = true;
box.receiveShadow = true;
scene.add(box);
```

## Disposal & Cleanup

Every `Geometry`, `Material`, and `Texture` created with `new` must be manually disposed. Three.js does **not** garbage-collect GPU resources.

```js
function disposeMesh(mesh) {
  scene.remove(mesh);
  mesh.geometry.dispose();
  const mats = Array.isArray(mesh.material) ? mesh.material : [mesh.material];
  for (const mat of mats) {
    mat.dispose();
    // Dispose texture maps
    for (const key of ['map','normalMap','roughnessMap','metalnessMap','emissiveMap']) {
      if (mat[key]) mat[key].dispose();
    }
  }
}

// On app teardown
renderer.dispose();
renderer.forceContextLoss(); // releases WebGL context
```

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `devicePixelRatio` uncapped | `Math.min(window.devicePixelRatio, 2)` |
| `shadowMap.enabled` set after first render | Set before `animate()` call |
| Missing `camera.updateProjectionMatrix()` | Call after any camera prop change |
| Oversized shadow frustum | Tightly fit `shadow.camera` bounds to shadow-receiving area |
| No disposal on scene clear | Implement `disposeMesh()` for all removed objects |
| Mixing SRGB/Linear textures | Set `texture.colorSpace = THREE.SRGBColorSpace` on color textures |
| `renderer.render()` called twice per frame | Ensure single render call in animation loop |

## Additional Resources

- **`references/camera-types.md`** — PerspectiveCamera vs OrthographicCamera, depth precision, logarithmic depth buffer
- **`references/lighting-guide.md`** — RectAreaLight, LightProbe, baked lighting, performance tiers
