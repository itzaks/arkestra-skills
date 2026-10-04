# Patterns

Short, working shapes for the things plugins are for. Kernels follow [metal.md](metal.md).

## Feedback / trails (ping-pong)

```js
var trails, pass;
function setup(ctx) {
  trails = ctx.persistentTexture("trails", { format: "rgba16f", pingPong: true });
  pass = ctx.computePass("trail", {
    textures: { src: ctx.input(), history: trails.read, next: trails.write, dst: ctx.output },
    uniforms: { decay: 0.92 }
  });
}
function render(ctx) {
  pass.uniforms.decay = ctx.params.decay;
  ctx.dispatch(pass, ctx.size[0], ctx.size[1]);
  ctx.swap(trails);          // this frame's "next" is next frame's "history"
}
```

The kernel samples `history`, mixes in `src`, writes the result to both `next` and `dst`. The pass keeps its
bindings: `swap` changes which physical texture `read`/`write` mean, so one pass works every frame.

## Simulation with N steps per frame

Reaction-diffusion, cellular automata, Jacobi solvers. A `steps` parameter capped by its `max` keeps the cost bounded.

```js
var sim, step, show;
function setup(ctx) {
  sim  = ctx.persistentTexture("sim", { format: "rg32f", scale: 0.5, pingPong: true });
  step = ctx.computePass("step", { textures: { prev: sim.read, next: sim.write }, uniforms: { feed: 0.055, kill: 0.062 } });
  show = ctx.computePass("show", { textures: { state: sim.read, dst: ctx.output }, uniforms: { color: [1, 1, 1] } });
}
function render(ctx) {
  var p = ctx.params, w = ctx.size[0] * 0.5, h = ctx.size[1] * 0.5;
  step.uniforms.feed = p.feed;
  step.uniforms.kill = p.kill;
  for (var i = 0; i < p.steps; i++) { ctx.dispatch(step, w, h); ctx.swap(sim); }
  show.uniforms.color = p.color;
  ctx.dispatch(show, ctx.size[0], ctx.size[1]);
}
```

Every dispatch counts against `passesPerFrame` (128 by default), so keep `steps` `max` well under it.
Seed the state from the input or from a `reset` parameter (a bool the user can map to a trigger) in the kernel.

## Multi-pass blur / bloom

Separate horizontal and vertical passes over scaled transient textures:

```js
function setup(ctx) {
  a = ctx.texture("a", { scale: 0.5 });
  b = ctx.texture("b", { scale: 0.5 });
  bright = ctx.computePass("bright", { textures: { src: ctx.input(), dst: a }, uniforms: { threshold: 0.7 } });
  blurH  = ctx.computePass("blur",   { textures: { src: a, dst: b }, uniforms: { dir: [1, 0], radius: 8 } });
  blurV  = ctx.computePass("blur",   { textures: { src: b, dst: a }, uniforms: { dir: [0, 1], radius: 8 } });
  combine = ctx.computePass("combine", { textures: { src: ctx.input(), glow: a, dst: ctx.output }, uniforms: { amount: 1 } });
}
```

Two passes may use the same pipeline with different bindings, as `blurH`/`blurV` do. For a pyramid, make one texture
per level (`scale: 0.5, 0.25, …`) in `setup`.

## Compute on change, reuse every frame

Expensive work (a distance field, a pattern, a LUT) goes into a persistent texture and is rebuilt only when its inputs
change. Compare with the previous values: no allocation needed.

```js
var lastSeed = -1, lastDensity = -1;
function render(ctx) {
  var p = ctx.params;
  if (p.seed !== lastSeed || p.density !== lastDensity) {
    lastSeed = p.seed; lastDensity = p.density;
    build.uniforms.seed = p.seed;
    build.uniforms.density = p.density;
    ctx.dispatch(build, 1024, 1024);       // writes the persistent "field" texture
  }
  ctx.dispatch(draw, ctx.size[0], ctx.size[1]);   // samples "field" every frame
}
```

A persistent texture is cleared after a resize, so give it a fixed `width`/`height` when its contents don't depend on
the render size, or also rebuild when `ctx.size` changes.

## Scatter writes

A kernel may write any pixel of its output, not only `gid`: dispatch one thread per *source* element (a particle
texel, a grid cell) and write where it lands. Clear the target first with its own pass, since writes from different
threads to one pixel race (the last one wins).

## Particles in textures

Keep positions and velocities in an `rgba32f` ping-pong texture (one texel per particle, e.g. 512×512 for 262k
particles via `width`/`height`). An `update` pass integrates them, a `clear` pass empties a splat texture, a `splat`
pass scatters one thread per particle into it, and a `show` pass tone-maps it into `ctx.output`. Draw passes, which
would make this simpler, aren't available yet.

## Using another track as input

Declare `{ "key": "mask", "type": "texture", "label": "Mask" }` and bind `ctx.input("mask")`. The user picks the source
in Arkestra. Until they do, it's the plugin's own input (effects) or a 1×1 transparent texture (generators).

## Live text

Declare `"capabilities": { "rasterize": { "maxAtlases": 2 } }` and a `text` parameter. Rasterize when
`ctx.changed("message")` is true, then sample `atlas.texture` (an `r8` coverage mask, y down) in a kernel. Users
can bind the text parameter to OSC from its antenna button, so any script can push text in. Use two atlases taking
turns for transitions: rasterize the next message into the hidden one, and start the transition when its
`generation` bumps.

## Seeded randomness

```js
var rng;
function setup(ctx) { rng = ctx.random(1234); }
function render(ctx) { pass.uniforms.jitter = rng.next(); }
```

For variety the user controls, expose a `seed` float (`"step": 1`) and hash it in the kernel instead.

## Throttled logging

```js
var nextLog = 0;
function render(ctx) {
  if (ctx.time > nextLog) { nextLog = ctx.time + 1; console.log("count", ctx.vision.faces.count); }
}
```
