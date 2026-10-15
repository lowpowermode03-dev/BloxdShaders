# Bloxd WGSL Shader Notes

A community-written reference for experimenting with Bloxd's WGSL shader-pack API. This document collects the observed pack structure, shader callbacks, built-in data types and helpers, code examples, and a short WGSL primer.

> **Status:** Community research, not official documentation. Some behavior—especially macro availability and scene-depth support—may be incomplete or unverified. Test shaders in a safe environment and treat examples as starting points.

## Contents

- [Shader pack structure](#shader-pack-structure)
- [WGSL language](#wgsl-language)
- [Macros](#macros)
- [Data structures](#data-structures)
- [Built-in API](#built-in-api)
- [Shader callbacks](#shader-callbacks)
- [Examples](#examples)
- [Credits](#credits)
- [API references](#api-references)
- [Quick WGSL tutorial](#quick-wgsl-tutorial)

---

<details>

<summary>

## Macros

</summary>

Macros are compile-time switches. Depending on which macros are defined, shader code and available fields may differ.
```wgsl
BLOXD_STRICT // "use strict" of shaders, prevent stuff like `discard;` from being used
BLOXD_LIGHTING // bloxd lighting toggle
BLOXD_ALPHA // is shader accepting alpha values
BLOXD_SCENE_DEPTH // not too sure
//they are either defined or not
//parse directive exists and are used like so

#ifdef MACRO
  // runs code inside here if `MACRO` is defined
#else
  // runs code inside here if `MACRO` isn't defined
#endif
// Similar in shape to if (...) { ... } else { ... } in JavaScript.

#ifndef MACRO // Compiles the block only when MACRO is not defined.
```

</details>

<details>

<summary>

## Data structures

</summary>

These structures define the data passed into and returned from shader callbacks.
```wgsl
// === INTERNAL ===

struct BloxdPackInputs {
    time: f32,
    originOffset: vec3i,
    sunDirection: vec3f,
    eyePosition: vec3f,
}
@group(3) @binding(0) var<uniform> bloxdPackInputs: BloxdPackInputs;

// === EXTERNAL ===

struct BloxdVertexIn {
    position: vec3f,     // noa-local: world minus bloxdOriginOffset()
    normal: vec3f,
    blockId: u32,        // the block whose face this is, by its block ID, the same across the face
}

struct BloxdVertex {
    position: vec3f,     // noa-local
}

struct BloxdSurfaceIn {
    colour: vec3f,       // texture x vertex colour (tint x AO, baked together by the mesher)
    alpha: f32,
    normal: vec3f,
    position: vec3f,     // noa-local; see bloxdWorldPosition and bloxdFaceBlock
    blockId: u32,        // as in BloxdVertexIn
}

struct BloxdSurface {
    colour: vec3f,
#ifdef BLOXD_ALPHA
    alpha: f32,
#endif
    emission: vec3f,     // added after lighting, then fogged
}

struct BloxdLightIn {
    litColour: vec3f,    // the engine's lit result for this fragment, from the surface hook's colour: before emission and fog
    engineColour: vec3f, // the same after fog, bloxdApplyFog(litColour, l); bloxdDefaultLight adds the emission
    directional: f32,    // summed Lambert from the scene's directional lights (0 when off)
    ambient: vec3f,      // the material's ambient times the scene's
    blockLight: vec3f,   // voxel block light, tone-mapped; 0 without BLOXD_LIGHTING
    skyLight: f32,       // voxel sky light, tone-mapped; 1 without BLOXD_LIGHTING
    ambientLightColour: vec3f, // white without BLOXD_LIGHTING
    skyLightColour: vec3f,     // white without BLOXD_LIGHTING
    heldLight: vec3f,    // held light here, tone-mapped like blockLight, before the ambient boost; 0 out of reach
                         // or without BLOXD_LIGHTING
    fogColour: vec3f,    // the game's fog colour. Bloxd's games always have fog, pushed far out when the player
                         // turns it off; black only in a scene with no fog at all
    fogVisibility: f32,  // 1 = no fog, 0 = fully fogged
    position: vec3f,     // as in BloxdSurfaceIn
    normal: vec3f,
    blockId: u32,        // as in BloxdVertexIn
}

struct BloxdLight {
    colour: vec3f,
}

// Returned by bloxdOpaqueBehind(), which surface and light hooks may call where BLOXD_SCENE_DEPTH is defined.
struct BloxdOpaqueBehind {
    found: bool,         // false where nothing opaque is behind (sky)
    position: vec3f,     // noa-local; meaningless when !found
    distance: f32,       // from the eye; meaningless when !found
}
```

</details>

<details>

<summary>

## Built-in API

</summary>
  
These helpers expose engine-provided values and default behavior. Call the default functions, then modify their returned values when needed.
```wgsl
// === HELPER API ===

// bloxdTime() returns to 0 every this many seconds.
const bloxdTimeWrap: f32 = 3600;

// Seconds, returning to 0 every bloxdTimeWrap.
fn bloxdTime() -> f32 {
    return bloxdPackInputs.time;
}

// Whole blocks: a noa-local position plus this is a world position.
fn bloxdOriginOffset() -> vec3i {
    return bloxdPackInputs.originOffset;
}

// The camera's position, noa-local.
fn bloxdEyePosition() -> vec3f {
    return bloxdPackInputs.eyePosition;
}

// Unit, towards the sun, in world orientation; zero when the sky has no sun.
fn bloxdSunDirection() -> vec3f {
    return bloxdPackInputs.sunDirection;
}

// A noa-local position in world coordinates. In f32, so it loses fine detail far from the world's origin.
fn bloxdWorldPosition(position: vec3f) -> vec3f {
    return position + vec3f(bloxdOriginOffset());
}

// The world block the face at a fragment's position and normal belongs to, exact anywhere: half a block back inside it.
fn bloxdFaceBlock(position: vec3f, normal: vec3f) -> vec3i {
    return vec3i(floor(position - normal * 0.5)) + bloxdOriginOffset();
}

// Fogs colour as the engine fogs this fragment.
fn bloxdApplyFog(colour: vec3f, l: BloxdLightIn) -> vec3f {
    return mix(l.fogColour, colour, l.fogVisibility);
}

// === BLOXD SHADER ===

fn bloxdDefaultVertex(v: BloxdVertexIn) -> BloxdVertex {
    return BloxdVertex(v.position);
}

fn bloxdDefaultSurface(s: BloxdSurfaceIn) -> BloxdSurface {
#ifdef BLOXD_ALPHA
    return BloxdSurface(s.colour, s.alpha, vec3f(0.0));
#else
    return BloxdSurface(s.colour, vec3f(0.0));
#endif
}

fn bloxdDefaultLight(s: BloxdSurface, l: BloxdLightIn) -> BloxdLight {
    return BloxdLight(l.engineColour + s.emission * l.fogVisibility);
}
```

</details>

<details>

<summary>

## Shader callbacks

</summary>

Bloxd invokes these callbacks during rendering. Each callback must return the appropriate output structure.

```wgsl
fn bloxdTerrainVertex(vertex: bloxdVertexIn) -> bloxdVertex {}
fn bloxdTerrainSurface(vertex: bloxdSurfaceIn) -> bloxdSurface {}
fn bloxdTerrainLight(surface: bloxdSurface, light: bloxdLightIn) -> bloxdLight {}

fn bloxdLiquidVertex(vertex: bloxdVertexIn) -> bloxdVertex {}
fn bloxdLiquidSurface(vertex: bloxdSurfaceIn) -> bloxdSurface {}
fn bloxdLiquidLight(surface: bloxdSurface, light: bloxdLightIn) -> bloxdLight {}
```

</details>

<details>

<summary>
  

## Example
---

</summary>

```wgsl
fn bloxdTerrainSurface(s: bloxdSurfaceIn) -> bloxdSurface {
    return bloxdDefaultSurface(s);
}
// Passes through the default surface unchanged.
```
```wgsl
fn bloxdTerrainSurface(s: bloxdSurfaceIn) -> bloxdSurface {
    var result = bloxdDefaultSurface(s);
    result.colour = vec3f(0.0) + 0.5 * result.colour;
    return result;
}
// Scales the surface colour to make it darker.
// Adjust the multiplier to change the brightness.
```
```wgsl
fn bloxdTerrainSurface(s: bloxdSurfaceIn) -> bloxdSurface {
    var result = bloxdDefaultSurface(s);
    let pulse = 0.5 + 0.5 * sin(s.time * 3.0); //what if its greater than 3.0 ?
    result.colour = vec3f(pulse, 0.0, 0.0);
    return result;
}
// Produces a pulsing red animation.
```
### Full example
```wgsl
fn bloxdTerrainSurface(s: bloxdSurfaceIn) -> bloxdSurface {
    var result = bloxdDefaultSurface(s);

    // Convert the local position to world coordinates.
    let worldPos = s.position + vec3f(s.originOffset);

    // Create a repeating wave pattern.
    let pattern = sin(worldPos.x * 2.0 + s.time * 3.0)
                * cos(worldPos.z * 2.0 + s.time * 2.0);

    // Map the wave to a colour range.
    let wave = pattern * 0.5 + 0.5;
    result.colour = vec3f(wave, 0.2, 1.0 - wave);

    // Add a matching emission glow.
    result.emission = vec3f(wave * 0.5, 0.0, wave);

    return result;
}
```

</details>

<details>

<summary>


## Credits
---

</summary>

- `GlitchHunter` — shader documentation and research
- `_Xenon_` — shader creation and early experimentation
- `Cannoli` — research into shader activation and required data
- `ObiloxYT` — shader behavior and storage research
### People who followed or contributed to the research
- all members in credits
- `Ocelote` `Tridentify`  //previous interest
- `Crownix` //wanted to be pinged for findings like these
- `nervousking` `SUSSYPRIME` //seen it be developed
- `KingCobra` //wanted to see shaders working

</details>

</details>

<details>
  
<summary>
  
# API references

</summary>

<details>

<summary>

## API version 1 (legacy reference)

</summary>

```wgsl
// Engine-provided and read-only to packs. apiVersion 1.
struct bloxd_VertexIn {
    position: vec3f,     // noa-local world space (world minus originOffset)
    normal: vec3f,
    originOffset: vec3i, // whole blocks
    time: f32,           // seconds, wrapped every 3600 s (part of the API)
}

#ifndef BLOXD_STRICT
struct bloxd_Vertex {
    position: vec3f,     // noa-local
}
#endif

struct bloxd_SurfaceIn {
    colour: vec3f,       // texture x vertex colour (tint x AO, baked together by the mesher in v1)
    alpha: f32,
    normal: vec3f,
    position: vec3f,     // noa-local
    originOffset: vec3i, // a face's world block is vec3i(floor(position - normal * 0.5)) + originOffset
    eyePosition: vec3f,  // noa-local
    time: f32,
    sunDirection: vec3f, // unit, towards the sun, in world orientation; zero when the sky has no sun
}

struct bloxd_Surface {
    colour: vec3f,
#ifdef BLOXD_ALPHA
    alpha: f32,
#endif
    emission: vec3f,     // added after lighting, then fogged
}

struct bloxd_LightIn {
    engineColour: vec3f, // the engine's result for this fragment, from the surface hook's colour
    directional: f32,    // summed Lambert from the scene's directional lights (0 when off)
    ambient: vec3f,      // the material's ambient times the scene's
#ifdef BLOXD_LIGHTING
    blockLight: vec3f,   // voxel block light, tone-mapped
    skyLight: f32,       // voxel sky light, tone-mapped
    ambientLightColour: vec3f,
    skyLightColour: vec3f,
    heldLight: vec3f,    // held light here, tone-mapped like blockLight, before the ambient boost; 0 out of reach
#endif
    fogColour: vec3f,
    fogFactor: f32,      // 1 = no fog, 0 = fully fogged
    position: vec3f,     // as in bloxd_SurfaceIn
    normal: vec3f,
    originOffset: vec3i,
    eyePosition: vec3f,
    time: f32,
    sunDirection: vec3f,
}

struct bloxd_Light {
    colour: vec3f,
}

#ifndef BLOXD_STRICT
fn bloxd_defaultVertex(v: bloxd_VertexIn) -> bloxd_Vertex {
    return bloxd_Vertex(v.position);
}
#endif

fn bloxd_defaultSurface(s: bloxd_SurfaceIn) -> bloxd_Surface {
#ifdef BLOXD_ALPHA
    return bloxd_Surface(s.colour, s.alpha, vec3f(0.0));
#else
    return bloxd_Surface(s.colour, vec3f(0.0));
#endif
}

fn bloxd_defaultLight(s: bloxd_Surface, l: bloxd_LightIn) -> bloxd_Light {
    return bloxd_Light(l.engineColour + s.emission * l.fogFactor);
}

fn bloxd_terrain_vertex(v: bloxd_VertexIn) -> bloxd_Vertex {}
fn bloxd_terrain_surface(v: bloxd_SurfaceIn) -> bloxd_Surface {}
fn bloxd_terrain_light(s: bloxd_Surface, l: bloxd_LightIn) -> bloxd_Light {}

fn bloxd_liquid_vertex(v: bloxd_VertexIn) -> bloxd_Vertex {}
fn bloxd_liquid_surface(v: bloxd_SurfaceIn) -> bloxd_Surface {}
fn bloxd_liquid_light(s: bloxd_Surface, l: bloxd_LightIn) -> bloxd_Light {}
```

</details>

<details>

<summary>

## API version 2

</summary>

```wgsl
// Engine-provided and read-only to packs. apiVersion 2.

struct BloxdPackInputs {
    time: f32,
    originOffset: vec3i,
    sunDirection: vec3f,
    eyePosition: vec3f,
}
@group(3) @binding(0) var<uniform> bloxdPackInputs: BloxdPackInputs;

// Per frame: the same for every vertex and fragment of a frame. The functions read the engine's uniforms, so any
// function can call them, not only a hook.

// bloxdTime() returns to 0 every this many seconds.
const bloxdTimeWrap: f32 = 3600;

// Seconds, returning to 0 every bloxdTimeWrap.
fn bloxdTime() -> f32 {
    return bloxdPackInputs.time;
}

// Whole blocks: a noa-local position plus this is a world position.
fn bloxdOriginOffset() -> vec3i {
    return bloxdPackInputs.originOffset;
}

// The camera's position, noa-local.
fn bloxdEyePosition() -> vec3f {
    return bloxdPackInputs.eyePosition;
}

// Unit, towards the sun, in world orientation; zero when the sky has no sun.
fn bloxdSunDirection() -> vec3f {
    return bloxdPackInputs.sunDirection;
}

// A noa-local position in world coordinates. In f32, so it loses fine detail far from the world's origin.
fn bloxdWorldPosition(position: vec3f) -> vec3f {
    return position + vec3f(bloxdOriginOffset());
}

// The world block the face at a fragment's position and normal belongs to, exact anywhere: half a block back inside it.
fn bloxdFaceBlock(position: vec3f, normal: vec3f) -> vec3i {
    return vec3i(floor(position - normal * 0.5)) + bloxdOriginOffset();
}

struct BloxdVertexIn {
    position: vec3f,     // noa-local: world minus bloxdOriginOffset()
    normal: vec3f,
    blockId: u32,        // the block whose face this is, by its block ID, the same across the face
}

struct BloxdVertex {
    position: vec3f,     // noa-local
}

struct BloxdSurfaceIn {
    colour: vec3f,       // texture x vertex colour (tint x AO, baked together by the mesher)
    alpha: f32,
    normal: vec3f,
    position: vec3f,     // noa-local; see bloxdWorldPosition and bloxdFaceBlock
    blockId: u32,        // as in BloxdVertexIn
}

struct BloxdSurface {
    colour: vec3f,
#ifdef BLOXD_ALPHA
    alpha: f32,
#endif
    emission: vec3f,     // added after lighting, then fogged
}

struct BloxdLightIn {
    litColour: vec3f,    // the engine's lit result for this fragment, from the surface hook's colour: before emission and fog
    engineColour: vec3f, // the same after fog, bloxdApplyFog(litColour, l); bloxdDefaultLight adds the emission
    directional: f32,    // summed Lambert from the scene's directional lights (0 when off)
    ambient: vec3f,      // the material's ambient times the scene's
    blockLight: vec3f,   // voxel block light, tone-mapped; 0 without BLOXD_LIGHTING
    skyLight: f32,       // voxel sky light, tone-mapped; 1 without BLOXD_LIGHTING
    ambientLightColour: vec3f, // white without BLOXD_LIGHTING
    skyLightColour: vec3f,     // white without BLOXD_LIGHTING
    heldLight: vec3f,    // held light here, tone-mapped like blockLight, before the ambient boost; 0 out of reach
                         // or without BLOXD_LIGHTING
    fogColour: vec3f,    // the game's fog colour. Bloxd's games always have fog, pushed far out when the player
                         // turns it off; black only in a scene with no fog at all
    fogVisibility: f32,  // 1 = no fog, 0 = fully fogged
    position: vec3f,     // as in BloxdSurfaceIn
    normal: vec3f,
    blockId: u32,        // as in BloxdVertexIn
}

struct BloxdLight {
    colour: vec3f,
}

// Returned by bloxdOpaqueBehind(), which surface and light hooks may call where BLOXD_SCENE_DEPTH is defined.
struct BloxdOpaqueBehind {
    found: bool,         // false where nothing opaque is behind (sky)
    position: vec3f,     // noa-local; meaningless when !found
    distance: f32,       // from the eye; meaningless when !found
}

fn bloxdDefaultVertex(v: BloxdVertexIn) -> BloxdVertex {
    return BloxdVertex(v.position);
}

fn bloxdDefaultSurface(s: BloxdSurfaceIn) -> BloxdSurface {
#ifdef BLOXD_ALPHA
    return BloxdSurface(s.colour, s.alpha, vec3f(0.0));
#else
    return BloxdSurface(s.colour, vec3f(0.0));
#endif
}

// Fogs colour as the engine fogs this fragment.
fn bloxdApplyFog(colour: vec3f, l: BloxdLightIn) -> vec3f {
    return mix(l.fogColour, colour, l.fogVisibility);
}

fn bloxdDefaultLight(s: BloxdSurface, l: BloxdLightIn) -> BloxdLight {
    return BloxdLight(l.engineColour + s.emission * l.fogVisibility);
}

fn bloxdTerrainVertex(vertex: bloxdVertexIn) -> bloxdVertex {}
fn bloxdTerrainSurface(vertex: bloxdSurfaceIn) -> bloxdSurface {}
fn bloxdTerrainLight(surface: bloxdSurface, light: bloxdLightIn) -> bloxdLight {}

fn bloxdLiquidVertex(vertex: bloxdVertexIn) -> bloxdVertex {}
fn bloxdLiquidSurface(vertex: bloxdSurfaceIn) -> bloxdSurface {}
fn bloxdLiquidLight(surface: bloxdSurface, light: bloxdLightIn) -> bloxdLight {}
```

</details>

</details>

<details>
  
<summary>

# Quick WGSL tutorial

</summary>

---

Expected Knowledge:
- Standard JS knowledge
- A Understanding of Types

## JavaScript vs. WGSL

```js
var a = 1;
let c = 3;
function d(e) { return e * 2; }
```

```wgsl
var a: f32 = 1;
let c: f32 = 3;
fn d(e: f32) -> f32 { return e * 2; }
//note how you need variable, argument, and return types stated
```

## Syntax

```wgsl
// For
for (var i = 0; i < 10; i++) { ... }

// If
if (i < 5) {
  ...
} else if (i > 7) {
  ..
} else {
  ...
}

// While
var j = 0;
while (j < 5) {...}

// Loop
var k = 0;
loop {
  k++;
  if (...) {
    break; // Break
  }
  break if (...); // Break if
  
}

continue
for (var i = 0; i < 10; ++i) {
  if (i % 2 == 1) {
    continue; // Continue
  }
  ...
  continuing { // Continuing
    // continue goes here
    ...
  }
}

if (v < 0.5) {
  discard; // Discard
}
// discard exits the shader. It can only be used in a fragment shader.

// Switch
var a : i32;
let x : i32 = generateValue();
switch x {
  case 0: {      // The colon is optional
    a = 1;
  }
  default {      // The default need not appear last
    a = 2;
  }
  case 1, 2, {   // Multiple selector values can be used
    a = 3;
  }
  case 3, {      // The trailing comma is optional
    a = 4;
  }
  case 4 {
    a = 5;
  }
}
// switch only works with u32 or i32 and cases must be constants.
```
Unlike JavaScript, WGSL is strongly typed: values have specific types, and many operations require compatible types. Type inference can reduce how often you need to write those types explicitly.

## Type inference
so instead of having to declare all your types, it can infer your type
- Simple
```wgsl
fn f() -> bool { return false; }

// 32 Bit
1i   // i32
1u   // u32
1f   // f32
1.0f // f32

// 16 Bit
1h   // f16
1.0h // f16

// Compile Type
1    // AbstractInt
1.0  // AbstractFloat

// Inference
f() // inferred as bool
```
- Vector
```wgsl
let a = vec4<f32>(1, 2, 3, 4);
let b = vec4f(1, 2, 3, 4);
// both are same type
```

## Types
- Simple Types
```wgsl
i32  // 32 bit signed int
u32  // 32 bit unsigned int
f32  // 32 bit float
bool // boolean

// COMPILE ONLY
AbstractInt // undecided int type
AbstractFloat // undecided float type
```
- Read/Write
```wgsl
var a = 1 // is equivalent of `let` in JS
let b = 1 // is equivalent of `const` in JS

const c = 1 // is a compile time constant
```
- Vectors
```wgsl
// Declare
let a = vec4<f32>(1, 2, 3, 4); // 4 f32 s
let b = vec4<f32>(4, 3, 2, 1); // 4 f32 s

let uint = vec3<u32>(3.4, 5.6, 7.8); // 3 u32 s
let sint = vec2<i32>(1, -2); // 2 i32 s

// Accessing
a.z;   // via x,y,z,w
a.b;   // via r,g,b,a
a[2];  // via array element accessors

// Swizzles
a.zx;   // via x,y,z,w
a.br;   // via r,g,b,a
vec2<f32>(a[2], a[0]);
a.xzz // even repeat elements

// Insertion
vec4f(1, 2, 3, 4);
vec2f(2, 3);
vec4f(1, a.xx, 4);
vec4f(1, a.yz, 4);
vec4f(a.xyz, 4);
vec4f(1, a.yzw);
// all the same type
// it just inserts the vector into the longer vector

// Operations
a + b; // vec4f(5, 5, 5, 5)
a * b; // vec4f(4, 6, 6, 4)
a - b; // vec4f(-3, -1, 1, 3)
mix(a, b) // vec4f(1, 2, 2, 1)
```
- Matrix
```wgsl
// `mat[numVectors]x[vectorSize]<[type]>`
let a: mat4x4<f32> = ...
let b: mat4x4f = ...
let c = mat4x4f(...);
let v = vec4f(1, 2, 3, 4);

a[2];   // vec4f of the 3rd vector of a
a * v;  // vec4f and the result of a * b ( * being used as dot product )
```
- Array
```wgsl
// `array<[type],[numElements]>`
let a = array<f32, 5>;   // an array of five f32s
let b = array(vec3f(1,2,3), vec3f(4,5,6), vec3f(7,8,9));
```
- Function
```wgsl
// `fn [name]([params]) -> [returnType] {[body]}`
fn add(a: f32, b: f32) -> f32 {
  return a + b;
}
```
## Type conversion
Since it is a strongly typed language you can't auto convert one to another
```wgsl
let a = 1;     // a is an i32
let b = 2.0;   // b is a f32
let c = a + b; // ERROR can't add an i32 to an f32
```
Convert the integer explicitly:
```wgsl
let a = 1; // a is an i32
let b = 2.0; // b is a f32
let c = f32(a) + b; // ok
let d = 1 + 2.0; // d is a f32 // AbstInt + AbstFloat = AbstFloat = f32 //then becomes decided
```

For the complete language reference, continue with:
- https://webgpufundamentals.org/webgpu/lessons/webgpu-wgsl.html
- https://webgpufundamentals.org/webgpu/lessons/webgpu-wgsl-function-reference.html
This section is only a quick introduction; consult the references above for complete syntax and built-in functions.
---

</details>
