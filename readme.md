#Raging Sea

This project relies heavily on custom shaders.

A general walkthrough of the pertinent code is given below:

To begin, the starting imports are made.
```JavaScript
import './style.css'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import * as dat from 'dat.gui'
```

Next the base of the scene was added.
```JavaScript
const gui = new dat.GUI({ width: 340 })
const canvas = document.querySelector('canvas.webgl')
const scene = new THREE.Scene()
```

Next the size of the scene was set.
```JavaScript
const sizes = {
    width: window.innerWidth,
    height: window.innerHeight
}
```

Next the camera was installed.
```JavaScript
const camera = new THREE.PerspectiveCamera(75, sizes.width / sizes.height, 0.1, 100)
camera.position.set(1, 1, 1)
scene.add(camera)
```

Next the camera controls were added.
```JavaScript
const controls = new OrbitControls(camera, canvas)
controls.enableDamping = true
```

Next the renderer was added to the scene.
```JavaScript
const renderer = new THREE.WebGLRenderer({
    canvas: canvas
})
renderer.setSize(sizes.width, sizes.height)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
```

Next an event listener was placed on the window to handle resizing events.
```JavaScript
window.addEventListener('resize', () =>
{
    // Update sizes
    sizes.width = window.innerWidth
    sizes.height = window.innerHeight

    // Update camera
    camera.aspect = sizes.width / sizes.height
    camera.updateProjectionMatrix()

    // Update renderer
    renderer.setSize(sizes.width, sizes.height)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})
```

Next the tick function is created to carry out updates and animations.
```JavaScript
/**
 * Animate
 */
const clock = new THREE.Clock()

const tick = () =>
{
    const elapsedTime = clock.getElapsedTime()

    // Update controls
    controls.update()

    // Render
    renderer.render(scene, camera)

    // Call tick again on the next frame
    window.requestAnimationFrame(tick)
}

tick()
```

Next the object is added to the scene.
```JavaScript
const waterGeometry = new THREE.PlaneGeometry(2, 2, 128, 128)
const waterMaterial = new THREE.MeshBasicMaterial()
const water = new THREE.Mesh(waterGeometry, waterMaterial)
water.rotation.x = - Math.PI * 0.5
scene.add(water)
```

Starting from this intial plane this project will make it motion in a wave like manner as well as colorize it.
To accomplish this first the MeshBasicMaterial is changed to ShaderMaterial.
```JavaScript
const waterMaterial = new THREE.ShaderMaterial();
```
Next a new folder is created in the src folder called shaders and inside that folder another new folder named water is created. Inside the water folder two new files are created (not shown), fragment.glsl and vertex.glsl. Next the vertex.glsl file is set up.
```JavaScript
void main()
{
    vec4 modelMatrix = modelMatrix * vec4(position, 1.0);
    vec4 viewPosition = viewMatrix * modelPosition;
    vec4 projectedPosition = projectionMatrix * viewPosition;

    gl_Position = projectedPosition;
}
```

Next the fragment.glsl file was setup.
```JavaScript
void main()
{
    gl_FragColor = vec4(0.5, 0.8,1.0,1.0);
}
```

Next the two files are imported into script.js.
```JavaScript
import waterVertexShader from "./shaders/water/vertex.glsl";
import waterFragmentShader from "./shaders/water/fragment.glsl";
```

Next the shaders were used inside of the material.
```JavaScript
const waterMaterial = new THREE.ShaderMaterial({
  vertexShader: waterVertexShader,
  fragmentShader: waterFragmentShader,
});
```

At this point a blue plane should be rendered to the screen. Sin will be used to create waves on the plane. This will be done by moving the y value of the model position with sin() inside the vertex shader.
```JavaScript
vec4 modelPosition = modelMatrix * vec4(position, 1.0);
    modelPosition.y += sin(modelPosition.x);
```

Uniforms were used to control the waves elevation and frequency. This was done by adding a new uniform inside the waterMaterial inside script.js.
```JavaScript
  uniforms: {
    uBigWavesElevation: { value: 0.2 },
  },
```

Next that uBigWavesElevation uniform is retrieved by the vertex shader for use there.
```JavaScript
uniform float uBigWavesElevation;
void main()
{
    
    vec4 modelPosition = modelMatrix * vec4(position, 1.0);
    modelPosition.y += sin(modelPosition.x) * uBigWavesElevation;
```

Next an elevation variable was created in order to colorize the waves by using height and depth with shadows later.
```JavaScript
   float elevation = sin(modelPosition.x) * uBigWavesElevation;
```

Next I added uBigWavesElevation into Dat.GUI.
```JavaScript
gui
  .add(waterMaterial.uniforms.uBigWavesElevation, "value")
  .min(0)
  .max(1)
  .step(0.001)
  .name("waveElevation");
```

Next the wave frequency was controlled to prevent uniform waves on the plane. To do this another uniform was added.
```JavaScript
  uniforms: {
    uBigWavesElevation: { value: 0.2 },
    uBigWavesFrequency: { value: new THREE.Vector2(4,1.5)}
  },
```

Next the new uniform is imported into the vertex file (not shown) and it is used to set the frequency of the x.
```JavaScript
void main()
{   
    vec4 modelPosition = modelMatrix * vec4(position, 1.0);
    float elevation = sin(modelPosition.x * uBigWavesFrequency.x) * uBigWavesElevation;
    modelPosition.y += elevation;
```

Next the z axis is added to the elevation variable.
```JavaScript
    float elevation = sin(modelPosition.x * uBigWavesFrequency.x) * sin(modelPosition.z * uBigWavesFrequency.y) * uBigWavesElevation;
```

Now there are waves occuring on both the x and z axis with different frequencies. Next more tweaks were added to Dat.GUI for the frequencies of x and y.
```JavaScript
gui
  .add(waterMaterial.uniforms.uBigWavesFrequency.value, "x")
  .min(0)
  .max(10)
  .step(0.001)
  .name("waveFrequencyX");
gui
  .add(waterMaterial.uniforms.uBigWavesFrequency.value, "y")
  .min(0)
  .max(10)
  .step(0.001)
  .name("waveFrequencyY");
```

Next the process of animating the plane is started by using the elepased time to offeset te value in the sin(). To do this a new uniform was created to send the time to the shader.
```JavaScript
      uTime: { value: 0},
```

Now the water material is updated in the animation tick function.
```JavaScript
waterMaterial.uniforms.uTime.value = elapsedTime;
```

Next the uTime is imported into vertex.glsl (not shown) and then used inside both sin functions.
```JavaScript
   float elevation = sin(modelPosition.x * uBigWavesFrequency.x + uTime) * sin(modelPosition.z * uBigWavesFrequency.y + uTime) * uBigWavesElevation;
```

At this point the plane is animated and moving. Next the speed of the waves is controlled - the same speed is used for both axis. To do this a new uniform is created.
```JavaScript
    uBigWavesSpeed: { value: 0.75 },
```

uBigWaveSpeed is now fetched inside vertex.glsl (not shown) and used there.
```JavaScript
    float elevation = sin(modelPosition.x * uBigWavesFrequency.x + uTime * uBigWavesSpeed) * sin(modelPosition.z * uBigWavesFrequency.y + uTime * uBigWavesSpeed) * uBigWavesElevation;
```

Next the speed is added as a tweat to Dat.GUI (not shown). After that colors were added to the wave using two colors, one for depth and one for height. To start this process a debutObject must first be created to have a container to put the color values in.
```JavaScript
const debugObject = {};
```

Next the two color properies were added to the debugObject.
```JavaScript
debugObject.depthColor = '#0000ff';
debugObject.surfaceColor = '#8888ff';
```

To gain access to the color properties new uniforms were created using the .Color() class.
```JavaScript
    uDepthColor: { value: new THREE.Color(debugObject.depthColor) },
    uSurfaceColor: { value: new THREE.Color(debugObject.surfaceColor) },
```

Next the two properties were added to Dat.GUI and updated using an onChange function.
```JavaScript
gui
  .addColor(debugObject, "depthColor")
  .name("depthColor")
  .onChange(() => {
    waterMaterial.uniforms.uDepthColor.value.set(debugObject.depthColor);
  });
gui
  .addColor(debugObject, "surfaceColor")
  .name("surfaceColor")
  .onChange(() => {
    waterMaterial.uniforms.uSurfaceColor.value.set(debugObject.surfaceColor);
  });
```

Next those colors are retrieved inside of the fragment shader for use there.
```JavaScript
uniform vec3 uDepthColor;
uniform vec3 uSurfaceColor;

void main()
{
    gl_FragColor = vec4(uDepthColor,1.0);
}
```

Now that the depthColor is linked to the plane a mix was created bewteen the depth and surface colors according to the elevation of the wave. To do this varying is used to send information from the vertex file into the fragment file.
```JavaScript
varying float vElevation;

    vElevation = elevation;
```

Next the varying was retrieved inside the fragment and then used.
```JavaScript
varying float vElevation;

void main()
{
    gl_FragColor = vec4(uDepthColor,1.0);
}
```

Next the uDepthColor and uSuraceColor were mixed.
```JavaScript
void main()
{
    vec3 color = mix(uDepthColor, uSurfaceColor, vElevation);

    gl_FragColor = vec4(color,1.0);
}
```

To improve the contrast between the elevations two new uniforms are added to offset the colors.
```JavaScript
    uColorOffset: { value: 0.25 },
    uColorMultiplier: { value: 2 },
```

Then the tweaks are added to Dat.GUI (not shown). Next the uniforms were imported and used inside the fragment file.
```JavaScript
uniform float uColorOffset;
uniform float uColorMultiplier;

void main()
{

    float mixStrength = (vElevation + uColorOffset) * uColorMultiplier;
    vec3 color = mix(uDepthColor, uSurfaceColor, mixStrength );
```

At this point the plane is animated with waves and the colors can be contrasted using the interface. Next small waves are added using perlin noise to make the waves change in time inside the vertex file.
```JavaScript
// Classic Perlin 3D Noise 
// by Stefan Gustavson
//
vec4 permute(vec4 x)
{
    return mod(((x*34.0)+1.0)*x, 289.0);
}
vec4 taylorInvSqrt(vec4 r)
{
    return 1.79284291400159 - 0.85373472095314 * r;
}
vec3 fade(vec3 t)
{
    return t*t*t*(t*(t*6.0-15.0)+10.0);
}

float cnoise(vec3 P)
{
    vec3 Pi0 = floor(P); // Integer part for indexing
    vec3 Pi1 = Pi0 + vec3(1.0); // Integer part + 1
    Pi0 = mod(Pi0, 289.0);
    Pi1 = mod(Pi1, 289.0);
    vec3 Pf0 = fract(P); // Fractional part for interpolation
    vec3 Pf1 = Pf0 - vec3(1.0); // Fractional part - 1.0
    vec4 ix = vec4(Pi0.x, Pi1.x, Pi0.x, Pi1.x);
    vec4 iy = vec4(Pi0.yy, Pi1.yy);
    vec4 iz0 = Pi0.zzzz;
    vec4 iz1 = Pi1.zzzz;

    vec4 ixy = permute(permute(ix) + iy);
    vec4 ixy0 = permute(ixy + iz0);
    vec4 ixy1 = permute(ixy + iz1);

    vec4 gx0 = ixy0 / 7.0;
    vec4 gy0 = fract(floor(gx0) / 7.0) - 0.5;
    gx0 = fract(gx0);
    vec4 gz0 = vec4(0.5) - abs(gx0) - abs(gy0);
    vec4 sz0 = step(gz0, vec4(0.0));
    gx0 -= sz0 * (step(0.0, gx0) - 0.5);
    gy0 -= sz0 * (step(0.0, gy0) - 0.5);

    vec4 gx1 = ixy1 / 7.0;
    vec4 gy1 = fract(floor(gx1) / 7.0) - 0.5;
    gx1 = fract(gx1);
    vec4 gz1 = vec4(0.5) - abs(gx1) - abs(gy1);
    vec4 sz1 = step(gz1, vec4(0.0));
    gx1 -= sz1 * (step(0.0, gx1) - 0.5);
    gy1 -= sz1 * (step(0.0, gy1) - 0.5);

    vec3 g000 = vec3(gx0.x,gy0.x,gz0.x);
    vec3 g100 = vec3(gx0.y,gy0.y,gz0.y);
    vec3 g010 = vec3(gx0.z,gy0.z,gz0.z);
    vec3 g110 = vec3(gx0.w,gy0.w,gz0.w);
    vec3 g001 = vec3(gx1.x,gy1.x,gz1.x);
    vec3 g101 = vec3(gx1.y,gy1.y,gz1.y);
    vec3 g011 = vec3(gx1.z,gy1.z,gz1.z);
    vec3 g111 = vec3(gx1.w,gy1.w,gz1.w);

    vec4 norm0 = taylorInvSqrt(vec4(dot(g000, g000), dot(g010, g010), dot(g100, g100), dot(g110, g110)));
    g000 *= norm0.x;
    g010 *= norm0.y;
    g100 *= norm0.z;
    g110 *= norm0.w;
    vec4 norm1 = taylorInvSqrt(vec4(dot(g001, g001), dot(g011, g011), dot(g101, g101), dot(g111, g111)));
    g001 *= norm1.x;
    g011 *= norm1.y;
    g101 *= norm1.z;
    g111 *= norm1.w;

    float n000 = dot(g000, Pf0);
    float n100 = dot(g100, vec3(Pf1.x, Pf0.yz));
    float n010 = dot(g010, vec3(Pf0.x, Pf1.y, Pf0.z));
    float n110 = dot(g110, vec3(Pf1.xy, Pf0.z));
    float n001 = dot(g001, vec3(Pf0.xy, Pf1.z));
    float n101 = dot(g101, vec3(Pf1.x, Pf0.y, Pf1.z));
    float n011 = dot(g011, vec3(Pf0.x, Pf1.yz));
    float n111 = dot(g111, Pf1);

    vec3 fade_xyz = fade(Pf0);
    vec4 n_z = mix(vec4(n000, n100, n010, n110), vec4(n001, n101, n011, n111), fade_xyz.z);
    vec2 n_yz = mix(n_z.xy, n_z.zw, fade_xyz.y);
    float n_xyz = mix(n_yz.x, n_yz.y, fade_xyz.x); 
    return 2.2 * n_xyz;
}
```

Then cnoise was used to add in the effect.
```JavaScript
   elevation += cnoise(vec3(modelPosition.xz * 3.0, uTime * 0.2));
```

Now the amplitude of the small waves is too high and the big waves cannot be seen so this was fixed next.
```JavaScript
    elevation += cnoise(vec3(modelPosition.xz * 3.0, uTime * 0.2))*0.15;
```

To make realistic waves with rounded troughs and crest the abs() method is used.
```JavaScript
    elevation -= abs(cnoise(vec3(modelPosition.xz * 3.0, uTime * 0.2))*0.15);
```

More pearling noises with smaller frequencies were next added to add a more chaotic look to the waves.
```JavaScript
   for(float i = 1.0; i<=3.0;i++){
    elevation -= abs(cnoise(vec3(modelPosition.xz * 3.0 * i, uTime * 0.2))*0.15 / i);

    }
```

Lastly controls were added in to control the elevation, frequency, speed and iteration count of the small waves.
```JavaScript
    uSmallWavesElevation: {value: 0.15},
    uSmallWavesFrequency: {value: 3.0},
    uSmallWavesWavesSpeed: {value: 0.2},
    uSmallIterations: {value: 4.0},
```

Next they were retrieved in the vertex.
```JavaScript
uniform float uSmallWavesElevation;
uniform float uSmallWavesFrequency;
uniform float uSmallWavesSpeed;
uniform float uSmallWavesIterations;
```

Next they were added into the for loop in place of the hard coded values.
```JavaScript

    for(float i = 1.0; i <= uSmallIterations; i++){
    elevation -= abs(cnoise(vec3(modelPosition.xz * uBigWavesFrequency * i, uTime * 0.2)) * uBigWavesElevation / i);
    }
```

Lastly the controls were added to the GUI interface (not shown).


***End Walkthrough

