---
name: Three.js Shaders & Materials
description: This skill should be used when the user asks to "write a GLSL shader in Three.js", "create a ShaderMaterial", "add custom shader to Three.js", "Three.js uniforms", "GLSL vertex shader", "GLSL fragment shader", "Three.js post-processing", "Three.js EffectComposer", "onBeforeCompile Three.js", "Three.js shader chunks", "extend MeshStandardMaterial with GLSL", "Three.js RawShaderMaterial", "Three.js custom material", "Three.js noise shader", "Three.js dissolve effect", "Three.js outline shader", or "Three.js custom pass". Provides expert guidance on GLSL authoring, ShaderMaterial, RawShaderMaterial, uniforms, onBeforeCompile, post-processing, and shader chunk extensions for Three.js r160+ ESM.
version: 0.1.0
---

# Three.js Shaders & Materials

## Overview

This skill covers writing GLSL shaders in Three.js using `ShaderMaterial`, `RawShaderMaterial`, and `onBeforeCompile`. It also covers the `EffectComposer` post-processing pipeline. All examples target Three.js r160+ ESM with GLSL ES 3.00 where noted.

## ShaderMaterial vs RawShaderMaterial

| | `ShaderMaterial` | `RawShaderMaterial` |
|---|---|---|
| **Injected uniforms** | Yes — `modelViewMatrix`, `projectionMatrix`, `normalMatrix`, etc. | None — you declare everything |
| **GLSL version** | GLSL ES 1.00 (Three.js manages) | You declare `#version 300 es` |
| **Built-in chunks** | Available via `#include <...>` | Not available |
| **Use when** | Extending existing pipeline | Full control, GLSL ES 3 features |

## ShaderMaterial

```js
import * as THREE from 'three';

const mat = new THREE.ShaderMaterial({
  uniforms: {
    uTime:  { value: 0 },
    uColor: { value: new THREE.Color(0x4488ff) },
    uTex:   { value: null },
  },
  vertexShader: /* glsl */`
    uniform float uTime;
    varying vec2 vUv;
    varying vec3 vNormal;

    void main() {
      vUv = uv;
      vNormal = normalize(normalMatrix * normal);

      vec3 pos = position;
      pos.y += sin(pos.x * 3.0 + uTime) * 0.2; // wave deformation

      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: /* glsl */`
    uniform vec3 uColor;
    uniform sampler2D uTex;
    varying vec2 vUv;
    varying vec3 vNormal;

    void main() {
      float light = dot(vNormal, normalize(vec3(1.0, 2.0, 1.0)));
      light = clamp(light * 0.5 + 0.5, 0.0, 1.0); // half-lambert

      vec4 texColor = texture2D(uTex, vUv);
      gl_FragColor = vec4(uColor * texColor.rgb * light, texColor.a);
    }
  `,
  transparent: false,
  side: THREE.FrontSide,
});

// Update uniforms each frame
const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  mat.uniforms.uTime.value = clock.getElapsedTime();
  renderer.render(scene, camera);
}
```

**Available built-in uniforms in ShaderMaterial** (no declaration needed):
- `modelMatrix`, `viewMatrix`, `projectionMatrix`, `modelViewMatrix`
- `normalMatrix` (mat3, inverse-transpose of modelView)
- `cameraPosition` (world-space camera position)

**Available attributes** (no declaration needed):
- `position`, `normal`, `uv`, `uv2`, `color`, `tangent`

## RawShaderMaterial (GLSL ES 3)

```js
const mat = new THREE.RawShaderMaterial({
  glslVersion: THREE.GLSL3,
  uniforms: {
    uTime:             { value: 0 },
    modelViewMatrix:   { value: new THREE.Matrix4() },
    projectionMatrix:  { value: new THREE.Matrix4() },
    normalMatrix:      { value: new THREE.Matrix3() },
  },
  vertexShader: /* glsl */`#version 300 es
    uniform mat4 modelViewMatrix;
    uniform mat4 projectionMatrix;
    uniform mat3 normalMatrix;
    uniform float uTime;

    in vec3 position;
    in vec3 normal;
    in vec2 uv;

    out vec2 vUv;
    out vec3 vNormal;

    void main() {
      vUv = uv;
      vNormal = normalize(normalMatrix * normal);
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */`#version 300 es
    precision highp float;

    in vec2 vUv;
    in vec3 vNormal;
    out vec4 fragColor;

    void main() {
      float light = dot(vNormal, normalize(vec3(1.0, 2.0, 1.0))) * 0.5 + 0.5;
      fragColor = vec4(vec3(light), 1.0);
    }
  `,
});
```

With `RawShaderMaterial`, **all** uniforms and attributes must be declared manually, including `modelViewMatrix` and `projectionMatrix`.

## Uniforms — Types & Update Patterns

```js
uniforms: {
  // Scalars
  uFloat:   { value: 1.0 },
  uInt:     { value: 0 },

  // Vectors
  uVec2: { value: new THREE.Vector2(0.5, 0.5) },
  uVec3: { value: new THREE.Vector3(1, 0, 0) },
  uVec4: { value: new THREE.Vector4(0, 0, 0, 1) },
  uColor: { value: new THREE.Color(0xff0000) },

  // Matrices
  uMat3: { value: new THREE.Matrix3() },
  uMat4: { value: new THREE.Matrix4() },

  // Textures
  uTex:      { value: texture2d },
  uCubeMap:  { value: cubeTexture },
  uData:     { value: dataTexture },

  // Arrays
  uFloatArr: { value: [1.0, 2.0, 3.0] },
  uVecArr:   { value: [new THREE.Vector3(), new THREE.Vector3()] },
}
```

**Update patterns:**
```js
// Safe: mutate the value object (avoids allocation)
mat.uniforms.uColor.value.set(0x00ff00);
mat.uniforms.uVec2.value.set(x, y);
mat.uniforms.uMat4.value.copy(newMatrix);

// Also safe: assign a new value (fine for scalars)
mat.uniforms.uTime.value = clock.getElapsedTime();

// Wrong: replace the uniform object (breaks GLSL binding)
mat.uniforms.uColor = { value: new THREE.Color() }; // DON'T
```

## onBeforeCompile — Extending Built-in Materials

`onBeforeCompile` injects GLSL into Three.js's built-in PBR shaders without full shader rewrite:

```js
const mat = new THREE.MeshStandardMaterial({ color: 0xffffff });

mat.onBeforeCompile = (shader) => {
  // Inject custom uniforms
  shader.uniforms.uTime = { value: 0 };
  mat.userData.shader = shader; // keep reference to update uniforms

  // Inject into vertex shader (before or after specific chunks)
  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    /* glsl */`
      #include <begin_vertex>
      transformed.y += sin(transformed.x * 3.0 + uTime) * 0.1;
    `
  );

  // Inject into fragment shader
  shader.fragmentShader = `
    uniform float uTime;
    ${shader.fragmentShader}
  `.replace(
    '#include <dithering_fragment>',
    /* glsl */`
      #include <dithering_fragment>
      gl_FragColor.rgb = mix(gl_FragColor.rgb, vec3(1.0, 0.0, 0.0), 0.3);
    `
  );
};

// Update the injected uniform each frame
function animate() {
  requestAnimationFrame(animate);
  if (mat.userData.shader) {
    mat.userData.shader.uniforms.uTime.value = clock.getElapsedTime();
  }
  renderer.render(scene, camera);
}
```

For a full list of injectable chunk names, see `references/glsl-reference.md`.

## Texture Uniforms

```js
const loader = new THREE.TextureLoader();
const tex = loader.load('/texture.png');
tex.colorSpace = THREE.SRGBColorSpace; // for color/albedo textures
tex.wrapS = THREE.RepeatWrapping;
tex.wrapT = THREE.RepeatWrapping;
tex.repeat.set(4, 4);

mat.uniforms.uTex.value = tex;
```

For normal maps, roughness, and data textures, use `THREE.LinearSRGBColorSpace` (not SRGB).

## Common GLSL Utility Functions

```glsl
// Remap a value from [inMin, inMax] to [outMin, outMax]
float remap(float v, float inMin, float inMax, float outMin, float outMax) {
  return outMin + (v - inMin) / (inMax - inMin) * (outMax - outMin);
}

// Smooth step (built-in GLSL, listed for reference)
// float s = smoothstep(edge0, edge1, x); // 0 at edge0, 1 at edge1

// Random (hash-based)
float rand(vec2 co) {
  return fract(sin(dot(co, vec2(12.9898, 78.233))) * 43758.5453);
}

// Value noise
float noise(vec2 p) {
  vec2 i = floor(p), f = fract(p);
  f = f * f * (3.0 - 2.0 * f); // smoothstep
  return mix(
    mix(rand(i), rand(i + vec2(1,0)), f.x),
    mix(rand(i + vec2(0,1)), rand(i + vec2(1,1)), f.x),
    f.y
  );
}
```

For Perlin noise, FBM, and advanced GLSL patterns, see `references/glsl-reference.md`.

## Post-Processing with EffectComposer

```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';
import { FXAAShader } from 'three/addons/shaders/FXAAShader.js';

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera)); // always first

// Bloom pass
const bloom = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  1.5,  // strength
  0.4,  // radius
  0.85  // threshold
);
composer.addPass(bloom);

// FXAA antialiasing pass (replaces renderer antialias option when using composer)
const fxaa = new ShaderPass(FXAAShader);
fxaa.uniforms['resolution'].value.set(1 / window.innerWidth, 1 / window.innerHeight);
composer.addPass(fxaa);

// Use composer.render() instead of renderer.render()
function animate() {
  requestAnimationFrame(animate);
  composer.render(); // NOT renderer.render(scene, camera)
}

// Resize
window.addEventListener('resize', () => {
  composer.setSize(window.innerWidth, window.innerHeight);
  fxaa.uniforms['resolution'].value.set(1 / window.innerWidth, 1 / window.innerHeight);
});
```

For custom `ShaderPass` creation and the full post-processing pass list, see `references/post-processing.md`.

## Material Flags Reference

```js
new THREE.ShaderMaterial({
  transparent: true,       // enables alpha blending (sorts draw order)
  opacity: 1.0,
  blending: THREE.NormalBlending,  // NormalBlending, AdditiveBlending, MultiplyBlending
  depthWrite: false,       // disable for transparent/additive effects
  depthTest: true,
  side: THREE.FrontSide,   // FrontSide, BackSide, DoubleSide
  wireframe: false,
  lights: false,           // set true to receive Three.js light uniforms
  fog: false,              // set true to apply scene fog
  clipping: false,         // set true to support clipping planes
});
```

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Uniform `.value` object replaced (not mutated) | Always mutate: `u.value.set(...)`, not `u = {value: ...}` |
| `uTime` not updated every frame | Update in animation loop via `mat.userData.shader.uniforms.uTime.value` |
| `onBeforeCompile` chunk name typo | Check exact chunk names in `references/glsl-reference.md` |
| `EffectComposer` and `renderer.render()` both called | Use only `composer.render()` when composer is active |
| FXAA `resolution` uniform not updated on resize | Update `fxaa.uniforms.resolution` in resize handler |
| Color texture with wrong colorSpace | Use `SRGBColorSpace` for albedo, `LinearSRGBColorSpace` for data/normal |

## Additional Resources

- **`references/glsl-reference.md`** — Built-in chunk names, GLSL ES 3 features, noise functions, lighting equations, precision modifiers
- **`references/post-processing.md`** — Full pass list, custom ShaderPass authoring, render targets, multi-pass pipelines
