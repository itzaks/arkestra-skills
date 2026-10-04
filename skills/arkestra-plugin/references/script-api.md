# Script API (`ctx`)

`main.js` defines global functions that Arkestra calls with the same `ctx` object:

```js
function setup(ctx) { … }    // once per instance, after the kernels compiled; may allocate
function render(ctx) { … }   // every frame; must not allocate
```

`render` is required. A throw while evaluating the file or in `setup` is a load error: the instance passes its input
through (effects) or draws nothing (generators), and the error shows up in the status file, problems and console.

Only the manifest `entry` file is evaluated. There's no module system: put helpers in the same file.

## Always available

| Member | Where | What |
|---|---|---|
| `ctx.params.<key>` | anywhere | Current parameter value. `float`: number in real units, snapped to `step`, clamped to `[min, max]`. `bool`: 0/1. `select`: option index. `color`: `Float32Array(3)` RGB 0…1. `point2d`: `Float32Array(2)`. `text`: string. `texture` params aren't here, use `ctx.input(key)`. |
| `ctx.changed(key)` | render | True on the first frame after a `text` parameter changed, and on the very first frame. |
| `ctx.size` | anywhere | `Float32Array(2)`: render width, height in pixels. |
| `ctx.time`, `ctx.dt` | render | Seconds since start, seconds since the last frame. |
| `ctx.frame` | render | Frame counter. |
| `ctx.beat.bpm`, `ctx.beat.phase(i)`, `ctx.beat.divisions` | render | Arkestra's beat clock. `phase(i)` is 0…1 inside division `i` of `divisions` (labels such as "1 Bar", "1/4"). Prefer letting the user map a parameter. |
| `ctx.random(seed)` | create in setup | Deterministic RNG: `rng.next()` returns [0, 1) without allocating. |
| `ctx.state` | anywhere | Plain JSON-compatible object saved with the project (≤ 64 KiB) and kept across hot reloads unless the manifest says `"hotReload": "reset"`. Don't write it every frame. |
| `console.log/warn/error` | anywhere | Goes to the plugin console (`ark dev`, `get_plugin_console`, the developer panel). Throttle it: never log every frame. |

## Texture handles

Handles are plain numbers. Resolve them in `setup` and reuse them.

| Handle | Meaning |
|---|---|
| `ctx.output` | The plugin's output: `bgra8Unorm` at render size. Write float4 RGBA; Metal does the swizzle. |
| `ctx.input()` / `ctx.input("input")` | The incoming video (effects). For generators it's a 1×1 transparent texture. |
| `ctx.input("<key>")` | A `texture` parameter (another track, a camera, a file). Throws if no such parameter exists. |

## Resources (setup only)

```js
tmp    = ctx.texture("blurH", { format: "rgba16f", scale: 0.5 });          // contents not kept between frames
field  = ctx.persistentTexture("field", { format: "r32f" });               // kept between frames
state  = ctx.persistentTexture("state", { format: "rgba32f", pingPong: true });  // { read, write }
```

- Formats: `rgba16f` (default), `rgba32f`, `rgba8Unorm`, `bgra8Unorm`, `r16f`, `r32f`, `rg16f`, `rg32f`.
- Size: `scale` relative to the render size (0 < scale ≤ 2, default 1), or absolute `width` and `height` (≤ 8192).
  Scaled textures follow the render size when it changes.
- Persistent textures start cleared to 0 and are cleared again after a resize. Pass `clearOnResize: false` to keep
  undefined contents instead.
- Up to 16 textures by default (a ping-pong pair counts as one), and 256 MiB of texture memory.

## Passes

```js
pass = ctx.computePass("pipelineName", {      // a key of manifest "pipelines"
  textures: { src: ctx.input(), dst: ctx.output },
  buffers:  { rects: ctx.vision.faces.rects },   // apiVersion 2 host data views only
  uniforms: { amount: 1, color: [1, 1, 1] }      // numbers or arrays of numbers
});
```

- Returns `{ id, uniforms }`. `pass.uniforms.<name>` has a setter: assign numbers, or assign a param view/array to an
  array uniform (`pass.uniforms.color = ctx.params.color` copies the values, no allocation).
- You can't add uniform names after creation (the object is sealed).
- Make one pass per (pipeline, binding set). To alternate bindings (ping-pong between two atlases, say), make two
  passes in `setup` and pick one in `render`.

## Recording the frame (render)

| Call | Does |
|---|---|
| `ctx.dispatch(pass, x, y?, z?)` | Runs the kernel over an x×y×z grid of threads (fractions round up). Arkestra picks threadgroups. |
| `ctx.swap(pingPong)` | Flips `read`/`write` for every pass recorded after it. Fine to call many times per frame (Jacobi loops). |
| `ctx.blit(src, dst?)` | Copies (and scales) one texture into another; `dst` defaults to `ctx.output`. |

Commands run in recording order on the GPU, after `render` returns. Up to 128 commands per frame by default.
The output reaches the rest of Arkestra's chain one frame later.

## Text atlases (capability `rasterize`)

```js
atlas = ctx.atlas("title", { width: 2048, height: 512 });   // setup only
// render, only when the text changed:
if (ctx.changed("message")) ctx.rasterize(atlas, { text: ctx.params.message, font: "Helvetica-Bold", align: "center" });
```

- `atlas.texture`: a handle to bind like any texture. `r8Unorm` coverage (1 = ink), row 0 at the top. Empty until
  the first raster lands.
- `atlas.generation` bumps when a rasterize finished, `atlas.bounds` is the ink box (x0, y0, x1, y1 in 0…1, y down),
  `atlas.fontSize` is the fitted size in pixels.
- The spec is a string or `{ text, font (PostScript name), align: "left"|"center"|"right", maxSize (px, 0 = atlas
  height), padding (fraction of the height), lineHeight (multiple, 0 = font default) }`. Text wraps and auto-fits.
- It runs off the render thread; the newest request per atlas wins. Call it on change, never every frame.
- `width`/`height` 16…`maxAtlasSize`; at most `maxAtlases` atlases.

## apiVersion 2 services

`ctx.depth`, `ctx.vision` and `ctx.model(key, opts)` exist only with `"apiVersion": 2` and the matching capability.
See [analysis-services.md](analysis-services.md).

## Not available yet

`ctx.drawPass`, `ctx.drawPrimitives`, `ctx.buffer`, `ctx.persistentBuffer`, `ctx.bufferView`, `ctx.bufferViewU32`,
`ctx.texture3D`, `ctx.upload3D`, `ctx.asset`, `ctx.assetTexture`, `ctx.audio`, `ctx.midi`, `ctx.history`, `ctx.codec`,
and the `resize(ctx)` hook. They appear in the spec and in the CLI's type definitions, but the app doesn't run them yet.
