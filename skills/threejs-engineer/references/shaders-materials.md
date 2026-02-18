# Shaders & Materials Guide

## When to Use Custom Shaders

| Scenario | Recommended Approach |
|----------|---------------------|
| Standard PBR look | `MeshStandardMaterial` / `MeshPhysicalMaterial` |
| PBR with small tweaks | `onBeforeCompile` to inject shader chunks |
| Fully custom effect | `ShaderMaterial` or `RawShaderMaterial` |
| Particle systems | `ShaderMaterial` with `Points` or GPGPU |

## ShaderMaterial Integration Pattern

```ts
import * as THREE from 'three'
import vertexShader from './shaders/effect/vertex.glsl?raw'
import fragmentShader from './shaders/effect/fragment.glsl?raw'

const material = new THREE.ShaderMaterial({
  vertexShader,
  fragmentShader,
  uniforms: {
    u_time: { value: 0 },
    u_resolution: { value: new THREE.Vector2(window.innerWidth, window.innerHeight) },
    u_mouse: { value: new THREE.Vector2(0, 0) },
    u_noiseScale: { value: 1.5 },
    u_intensity: { value: 0.8 },
    u_texture: { value: null },
  },
  transparent: true,
  side: THREE.DoubleSide,
})

// Update per frame
function tick() {
  material.uniforms.u_time.value = clock.getElapsedTime()
  // ...
}
```

## Common Uniform Conventions

| Uniform | Type | Purpose |
|---------|------|---------|
| `u_time` | `float` | Elapsed time in seconds |
| `u_resolution` | `vec2` | Canvas resolution |
| `u_mouse` | `vec2` | Normalized mouse position (0-1) |
| `u_noiseScale` | `float` | Noise frequency multiplier |
| `u_intensity` | `float` | Effect strength |
| `u_color` | `vec3` | Base color |
| `u_texture` | `sampler2D` | Input texture |

## GLSL Utility Functions

### Noise Functions

```glsl
// Classic 2D value noise
float hash(vec2 p) {
  return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453123);
}

float noise(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  f = f * f * (3.0 - 2.0 * f); // smoothstep

  float a = hash(i);
  float b = hash(i + vec2(1.0, 0.0));
  float c = hash(i + vec2(0.0, 1.0));
  float d = hash(i + vec2(1.0, 1.0));

  return mix(mix(a, b, f.x), mix(c, d, f.x), f.y);
}

// Fractal Brownian Motion
float fbm(vec2 p) {
  float value = 0.0;
  float amplitude = 0.5;
  float frequency = 1.0;
  for (int i = 0; i < 6; i++) {
    value += amplitude * noise(p * frequency);
    frequency *= 2.0;
    amplitude *= 0.5;
  }
  return value;
}
```

### 2D Rotation

```glsl
vec2 rotate2D(vec2 p, float angle) {
  float s = sin(angle);
  float c = cos(angle);
  return vec2(p.x * c - p.y * s, p.x * s + p.y * c);
}
```

### Remap / Range Conversion

```glsl
float remap(float value, float inMin, float inMax, float outMin, float outMax) {
  return outMin + (outMax - outMin) * (value - inMin) / (inMax - inMin);
}
```

## Common Creative Shader Patterns

### Vertex Displacement

```glsl
// vertex.glsl
uniform float u_time;
uniform float u_noiseScale;
uniform float u_intensity;

varying vec2 vUv;
varying float vDisplacement;

// include noise functions here

void main() {
  vUv = uv;

  float displacement = fbm(position.xz * u_noiseScale + u_time * 0.3) * u_intensity;
  vDisplacement = displacement;

  vec3 newPosition = position + normal * displacement;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(newPosition, 1.0);
}
```

### Glow / Halo Effect

```glsl
// fragment.glsl - Fresnel-based rim glow
uniform vec3 u_color;
uniform float u_intensity;

varying vec3 vNormal;
varying vec3 vViewDir;

void main() {
  float fresnel = pow(1.0 - dot(vNormal, vViewDir), 3.0);
  vec3 glow = u_color * fresnel * u_intensity;
  gl_FragColor = vec4(glow, fresnel);
}
```

Use with `transparent: true` and `blending: THREE.AdditiveBlending` for halo effects.

### Procedural Water Surface

Key ingredients for procedural water:
1. **Vertex displacement**: stack multiple noise layers at different frequencies for waves
2. **Normal derivation**: compute normals from displaced surface via partial derivatives
3. **Reflection**: sample environment map using reflected view vector
4. **Refraction**: offset UV sampling based on view angle
5. **Fresnel**: blend reflection/refraction based on viewing angle

```glsl
// Simplified water fragment
uniform samplerCube u_envMap;
uniform float u_time;

varying vec3 vWorldPosition;
varying vec3 vNormal;
varying vec3 vViewDir;

void main() {
  // Animated normals from noise
  vec3 normal = normalize(vNormal);
  normal.xz += noise(vWorldPosition.xz * 2.0 + u_time) * 0.1;
  normal = normalize(normal);

  // Fresnel
  float fresnel = pow(1.0 - max(dot(normal, vViewDir), 0.0), 4.0);

  // Reflection
  vec3 reflected = reflect(-vViewDir, normal);
  vec3 reflectionColor = textureCube(u_envMap, reflected).rgb;

  // Water color
  vec3 deepColor = vec3(0.0, 0.05, 0.15);
  vec3 shallowColor = vec3(0.0, 0.3, 0.5);

  vec3 waterColor = mix(deepColor, shallowColor, fresnel);
  vec3 finalColor = mix(waterColor, reflectionColor, fresnel * 0.6);

  gl_FragColor = vec4(finalColor, 0.9);
}
```

### Hologram Effect

```glsl
// fragment.glsl
uniform float u_time;
uniform vec3 u_color;

varying vec2 vUv;
varying vec3 vNormal;
varying vec3 vViewDir;

void main() {
  // Scanlines
  float scanline = sin(vUv.y * 300.0 + u_time * 5.0) * 0.5 + 0.5;
  scanline = pow(scanline, 1.5) * 0.3 + 0.7;

  // Fresnel glow
  float fresnel = pow(1.0 - abs(dot(vNormal, vViewDir)), 2.0);

  // Glitch offset
  float glitch = step(0.98, sin(u_time * 50.0 + vUv.y * 20.0));

  vec3 color = u_color * scanline * (0.5 + fresnel * 0.5);
  float alpha = (0.3 + fresnel * 0.7) * scanline;
  alpha += glitch * 0.5;

  gl_FragColor = vec4(color, alpha);
}
```

### GPGPU Particles (Ping-Pong Pattern)

For GPU-driven particle systems:

1. Create two `WebGLRenderTarget` textures (position A / position B)
2. Use a simulation shader (full-screen quad) that reads from A and writes to B
3. Swap targets each frame (ping-pong)
4. Render particles using a `Points` mesh that reads positions from the current texture

```ts
// Simulation material
const simMaterial = new THREE.ShaderMaterial({
  uniforms: {
    u_positions: { value: null }, // previous frame positions texture
    u_time: { value: 0 },
    u_delta: { value: 0 },
  },
  vertexShader: simVertexShader,
  fragmentShader: simFragmentShader,
})

// Render particles
const particlesMaterial = new THREE.ShaderMaterial({
  uniforms: {
    u_positions: { value: null }, // current positions texture
    u_pointSize: { value: 2.0 },
  },
  vertexShader: particlesVertexShader,
  fragmentShader: particlesFragmentShader,
  transparent: true,
  depthWrite: false,
  blending: THREE.AdditiveBlending,
})
```

## onBeforeCompile Pattern

Extend built-in materials without writing a full ShaderMaterial:

```ts
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 })

material.onBeforeCompile = (shader) => {
  shader.uniforms.u_time = { value: 0 }

  // Inject into vertex shader
  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    `
    #include <begin_vertex>
    transformed.y += sin(transformed.x * 5.0 + u_time) * 0.2;
    `
  )

  // Store reference for updates
  material.userData.shader = shader
}

// Update in tick
if (material.userData.shader) {
  material.userData.shader.uniforms.u_time.value = elapsed
}
```
