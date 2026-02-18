# Camera Types in Three.js

## PerspectiveCamera

```js
const camera = new THREE.PerspectiveCamera(fov, aspect, near, far);
```

| Parameter | Typical Value | Effect |
|-----------|--------------|--------|
| `fov` | 45–75° | Wider = more peripheral distortion; narrower = telephoto compression |
| `aspect` | `innerWidth / innerHeight` | Must match render target aspect ratio |
| `near` | 0.1 | Minimum render distance. Values < 0.01 cause z-fighting. |
| `far` | 100–10000 | Objects beyond this are clipped |

### Depth Buffer Precision

Depth precision is non-linear — most precision is near the camera, least near `far`. To maximise precision:
- Keep `near` as large as possible (never `0.001` unless forced)
- Keep `far` as small as possible
- The ratio `far/near` determines precision: `1000/0.1 = 10000:1` uses full precision; `10000/0.001 = 10M:1` causes visible z-fighting

**Logarithmic depth buffer** (for scenes spanning many orders of magnitude):
```js
const renderer = new THREE.WebGLRenderer({ logarithmicDepthBuffer: true });
// All shaders must also use #include <logdepthbuf_pars_vertex> etc.
// Not compatible with some post-processing passes
```

### Dynamic FOV Adjustment

```js
// Vertical FOV stays constant — horizontal FOV changes with aspect
// For portrait layouts, you may want to maintain horizontal FOV:
function updateFOV(camera, baseFOV, baseAspect) {
  const baseH = 2 * Math.tan((baseFOV * Math.PI / 180) / 2);
  const baseW = baseH * baseAspect;
  const currentH = baseW / camera.aspect;
  camera.fov = 2 * Math.atan(currentH / 2) * (180 / Math.PI);
  camera.updateProjectionMatrix();
}
```

### Camera Shake

```js
const basePos = new THREE.Vector3(0, 2, 5);
const shake = new THREE.Vector3();

function animate() {
  const t = clock.getElapsedTime();
  const intensity = 0.05;
  shake.set(
    (Math.random() - 0.5) * intensity,
    (Math.random() - 0.5) * intensity,
    0
  );
  camera.position.copy(basePos).add(shake);
}
```

---

## OrthographicCamera

```js
const camera = new THREE.OrthographicCamera(left, right, top, bottom, near, far);
```

Orthographic projection has no perspective — objects appear the same size regardless of distance. Use for:
- 2D/UI overlays
- Isometric 3D games
- Technical drawings, CAD views
- Shadow camera (Three.js uses this internally for `DirectionalLight`)

```js
// Maintain aspect ratio on resize
function makeOrthoCamera(width, height, zoom = 5) {
  const aspect = width / height;
  return new THREE.OrthographicCamera(
    -zoom * aspect,  // left
     zoom * aspect,  // right
     zoom,           // top
    -zoom,           // bottom
    0.1, 1000
  );
}

window.addEventListener('resize', () => {
  const aspect = window.innerWidth / window.innerHeight;
  camera.left   = -zoom * aspect;
  camera.right  =  zoom * aspect;
  camera.top    =  zoom;
  camera.bottom = -zoom;
  camera.updateProjectionMatrix();
});
```

### Isometric Setup

```js
const iso = new THREE.OrthographicCamera(-10, 10, 10, -10, 0.1, 1000);
iso.position.set(10, 10, 10);
iso.lookAt(0, 0, 0);
// Exact isometric angle: Math.atan(1/Math.sqrt(2)) ≈ 35.26°
```

---

## CubeCamera (Reflections)

```js
// Capture a real-time environment map for reflective surfaces
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256, {
  format: THREE.RGBFormat,
  generateMipmaps: true,
  minFilter: THREE.LinearMipmapLinearFilter,
});
const cubeCamera = new THREE.CubeCamera(0.1, 100, cubeRenderTarget);
scene.add(cubeCamera);

const reflectiveMat = new THREE.MeshStandardMaterial({
  metalness: 1,
  roughness: 0,
  envMap: cubeRenderTarget.texture,
});

// In animate(): update the cube camera from the reflective object's position
function animate() {
  reflectiveMesh.visible = false; // hide self during capture
  cubeCamera.position.copy(reflectiveMesh.position);
  cubeCamera.update(renderer, scene);
  reflectiveMesh.visible = true;
  renderer.render(scene, camera);
}
```

**Performance note**: `cubeCamera.update()` renders 6 faces per call — very expensive. Only call it every N frames for slow-moving objects:
```js
if (frameCount % 10 === 0) cubeCamera.update(renderer, scene);
```

---

## ArrayCamera (Multi-view)

`ArrayCamera` renders multiple sub-cameras in one pass — used for VR split-screen or multi-viewport UIs:

```js
const subCameras = [
  Object.assign(new THREE.PerspectiveCamera(90, 0.5, 0.1, 1000), {
    viewport: new THREE.Vector4(0, 0, window.innerWidth / 2, window.innerHeight)
  }),
  Object.assign(new THREE.PerspectiveCamera(90, 0.5, 0.1, 1000), {
    viewport: new THREE.Vector4(window.innerWidth / 2, 0, window.innerWidth / 2, window.innerHeight)
  }),
];
const arrayCam = new THREE.ArrayCamera(subCameras);
renderer.render(scene, arrayCam);
```

---

## Frustum Culling

Three.js automatically culls objects outside the camera frustum. To debug culling:

```js
const frustum = new THREE.Frustum();
const mat = new THREE.Matrix4();

mat.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
frustum.setFromProjectionMatrix(mat);

// Test if a point is inside the frustum
const visible = frustum.containsPoint(new THREE.Vector3(0, 0, 0));

// Test if a bounding sphere is visible
const sphere = new THREE.Sphere(mesh.position, 1);
const inView = frustum.intersectsSphere(sphere);
```

Disable frustum culling on an object:
```js
mesh.frustumCulled = false; // always render regardless of camera
```

Use `frustumCulled = false` sparingly — it bypasses a major performance optimization.
