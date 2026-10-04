---
name: arkestra-plugin
description: Build, debug and publish plugins for Arkestra, the macOS live-visuals app. A plugin is an `.arkplugin` bundle (manifest.json + main.js + Metal compute kernels) that runs as a video effect or a generator layer. Use when the user wants to create or change an Arkestra plugin, write an Arkestra Metal kernel or `ctx` script, use depth, face/hand/body tracking or Core ML models in a plugin, fix a plugin's status or validation errors, or pack and publish one to the Arkestra Marketplace.
---

# Arkestra plugins

An Arkestra plugin is a folder ending in `.arkplugin`:

```
<Plugin Name>/                 project folder (notes, README, dist/ live here)
  <slug>.arkplugin/            the bundle: only this folder ships
    manifest.json              identity, parameters, pipelines, capabilities
    main.js                    setup(ctx) once, render(ctx) every frame
    kernels/<name>.metal       Metal compute kernels, compiled by Arkestra
    preview.png                optional 16:9 thumbnail for the plugin grid
```

`main.js` runs in JavaScriptCore on Arkestra's render thread (ES5-style, no `import`/`require`, no DOM, no network).
It never touches Metal itself: `setup` declares textures and passes, and `render` writes uniform numbers and
records dispatches. Arkestra compiles the kernels, runs the frame plan, and turns every manifest parameter into a
real Arkestra control that the user can map to LFOs, sequencers, MIDI, audio and the beat clock.

## Step 1: should this be a plugin at all?

Decide this before writing code, and tell the user what you chose and why.

- **Rhythm and modulation are Arkestra's job, not the plugin's.** Don't build beat sync, "every N bars", random
  jumps over time, envelopes or audio reactivity into the code. Expose a clear parameter (`size`, `density`, `seed`,
  `reset`…) and let the user map it. Continuous motion (`speed * ctx.dt`) is fine.
- **A per-pixel image is an ISF shader's job.** If the idea is a pure function of pixel position, time and inputs
  (color grading, distortion, kaleidoscopes, noise, patterns, simple feedback), an ISF `.fs` shader is simpler and
  has far more examples. With the Arkestra MCP tools: `create_shader`, then `update_custom_shader`.
- **A plugin is for the parts a fragment shader can't reach:**
  - state kept in GPU memory between frames (reaction-diffusion, fluids, cellular automata, particles in textures)
    in your own formats and resolutions;
  - many passes, or a variable number of them (blur/bloom pyramids, jump-flood distance fields, iterative solvers);
  - scatter writes, pixel sorting, histograms and reductions;
  - computing something expensive once and reusing it (rebuild a LUT or distance field only when a parameter changes);
  - combining several video inputs;
  - live text (`rasterize`), depth maps, face/hand/body tracking and Core ML models (see the references).

If the user asks for a plugin that an ISF shader would do better, say so once, then build what they asked for.

## Step 2: create the project

Pick whichever fits the user's setup. Without other instructions, use option 1: it needs nothing but file access.

1. **Arkestra's development folder** (no tools needed): `~/Documents/Arkestra/Plugins (Dev)/`. Arkestra watches it,
   loads new plugins by itself and **hot-reloads on every save**. It also writes `<slug>.status.txt` next to each
   bundle after every reload. If the folder has an `AGENTS.md`, Arkestra wrote it for this app version: read it,
   because it wins over this skill where the two disagree.
2. **The `ark` CLI** (Node 20.15+, npm package `@arkestra/plugin`):
   `npx -p @arkestra/plugin ark create my-plugin --template effect` (or `generator`, `feedback`), then
   `npx -p @arkestra/plugin ark dev` in the folder. `ark dev` links the bundle into the running app and streams
   console output, problems and reloads. If npm can't find the package, it isn't released yet: use option 1. Don't start from the `audio-reactive` or `codec-glitch` templates yet: their
   services aren't in the runtime (see "Not available yet").
3. **The New Plugin button** in Arkestra's Plugins tab, or the MCP tool `create_plugin`, both of which write a
   starter into the development folder.

In the development folder use the id `dev.local.<slug>`. Pick a real reverse-DNS id (`io.yourname.<slug>`) before
publishing, and never change it after that: it's the plugin's identity in saved projects.

Write `manifest.json` **last**, after the files it points to: Arkestra loads the bundle as soon as a valid manifest exists.

## Step 3: the minimal working effect

`manifest.json`
```json
{
  "id": "dev.local.pixelate",
  "name": "Pixelate",
  "version": "0.1.0",
  "apiVersion": 1,
  "kind": "effect",
  "category": "glitch",
  "description": "Snaps the image to a grid of square cells",
  "icon": "square.grid.3x3",
  "entry": "main.js",
  "pipelines": {
    "main": { "type": "compute", "source": "kernels/effect.metal", "function": "effectKernel" }
  },
  "parameters": [
    { "key": "cells", "type": "float", "label": "Pixel Size", "default": 24, "min": 1, "max": 128, "step": 1 }
  ]
}
```

`main.js`
```js
var pass;

function setup(ctx) {
  pass = ctx.computePass("main", {
    textures: { src: ctx.input(), dst: ctx.output },   // keys = kernel argument names
    uniforms: { cells: 24 }                            // order = Uniforms struct order
  });
}

function render(ctx) {
  pass.uniforms.cells = ctx.params.cells;
  ctx.dispatch(pass, ctx.size[0], ctx.size[1]);        // one thread per output pixel
}
```

`kernels/effect.metal` (Arkestra prepends `#include <metal_stdlib>` and `using namespace metal;`)
```metal
struct Uniforms {
    float cells;
};

kernel void effectKernel(texture2d<float, access::sample> src [[texture(0)]],
                         texture2d<float, access::write> dst [[texture(1)]],
                         constant Uniforms &uniforms [[buffer(0)]],
                         uint2 gid [[thread_position_in_grid]]) {
    if (gid.x >= dst.get_width() || gid.y >= dst.get_height()) return;
    constexpr sampler s(filter::nearest, address::clamp_to_edge);
    float2 size = float2(dst.get_width(), dst.get_height());
    float cell = max(uniforms.cells, 1.0);
    float2 snapped = (floor(float2(gid) / cell) + 0.5) * cell;
    dst.write(src.sample(s, snapped / size), gid);
}
```

A generator uses `"kind": "generator"`, a category from `simulation | signal | composition`, and binds only `ctx.output`.

## The rules that break plugins most often

1. **Create everything in `setup`.** `ctx.computePass`, `ctx.texture`, `ctx.persistentTexture`, `ctx.atlas` and
   `ctx.model` throw if called from `render`.
2. **`render` must not allocate.** No object or array literals, closures or string building per frame. Write into
   the pre-made `pass.uniforms`. A throw, or going over the script budget (about 3 ms) repeatedly, bypasses the
   plugin for the session until the next save.
3. **The `Uniforms` struct must match the `uniforms` object exactly**: same order, floats only. Array values are
   packed tightly, so use `packed_float2` / `packed_float3` / `packed_float4`, never `float2`/`float3`/`float4`. Bools
   and select indexes arrive as floats (`uniforms.flag > 0.5`).
4. **Bind by name.** `textures` and `buffers` keys must equal the kernel's argument names. The `[[texture(n)]]` /
   `[[buffer(n)]]` indexes only need to be unique. Uniforms always go in `constant Uniforms &uniforms`.
5. **Always bounds-check `gid`** against the texture you write.
6. **Something must write `ctx.output` every frame** (a dispatch or `ctx.blit`).
7. **Ping-pong textures:** read `tex.read`, write `tex.write`, then `ctx.swap(tex)` after the dispatch.
8. **Only these file types may live inside the bundle:** `.json .js .metal .png .jpg .jpeg .cube .txt .svg .csv`.
   Notes, READMEs and `.ts` go next to it.
9. **Parameter keys** match `^[a-z][A-Za-z0-9_]*$`, can't be `input`/`output` or start with `_`, and are permanent once
   published (saved mappings attach to them). There is no int type: use `float` with `"step": 1`. With `step`,
   `min`, `max` and `default` must be multiples of it.
10. **Don't read the beat for rhythm.** `ctx.beat.phase()` exists, but a mapped parameter is almost always better.

## Step 4: test and iterate

After every save, check the result. Don't assume a save worked.

- **Development folder:** read `<slug>.status.txt` next to the bundle. It says `ok`, or `error` with one
  `file:line:column message` line per problem (manifest and Metal compile errors). A broken save keeps the last good
  version running.
- **`ark` CLI** (if installed): `ark validate` runs the same manifest rules as the app and the Marketplace.
  `ark dev` streams problems and `console.log` output live.
- **Arkestra MCP tools** (if connected): `open_plugin_test_project` → `capture_output` (look at the result) →
  `get_plugin_problems` / `get_plugin_console` → `get_plugin_performance` → `save_plugin_preview` once it looks good.
  `open_plugin_test_project` replaces the open project, so ask the user before passing `force: true`.

See [references/testing-and-publishing.md](references/testing-and-publishing.md) for the full loop, MCP setup and publishing.

## Not available yet

The manifest spec and the CLI's `.d.ts` describe more than Arkestra runs today. Don't use these: they either fail
validation or throw at runtime. If an idea depends on one, say so and suggest what's possible now.

- render/draw pipelines (`"type": "render"`, `ctx.drawPass`, `ctx.drawPrimitives`): use compute kernels that stamp
  shapes per pixel;
- `ctx.buffer`, `ctx.persistentBuffer`, `ctx.bufferView` (CPU-built data, particle buffers): keep state in textures;
- `ctx.asset`, `ctx.assetTexture`, the optional `resize(ctx)` hook;
- the `audio`, `midi`, `history`, `texture3d` and `codec` capabilities (`ctx.audio`, `ctx.midi`, `ctx.history`,
  `ctx.codec`): let the user map parameters to audio and MIDI in Arkestra instead.

## References

Read only what the task needs:

- [references/script-api.md](references/script-api.md): every `ctx` member that works today, texture formats, handles.
- [references/manifest.md](references/manifest.md): all manifest fields, parameter types, `visibleWhen`, capabilities, budgets.
- [references/metal.md](references/metal.md): kernel conventions, coordinates, sampling, uniform packing.
- [references/patterns.md](references/patterns.md): feedback, simulations, multi-pass, compute-on-change, live text.
- [references/analysis-services.md](references/analysis-services.md): apiVersion 2 depth, vision (faces, hands,
  bodies) and Hugging Face Core ML models.
- [references/testing-and-publishing.md](references/testing-and-publishing.md): status files, `ark` CLI, MCP tools,
  packing, Marketplace and selling.
