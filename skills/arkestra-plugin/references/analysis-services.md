# Analysis services (apiVersion 2)

Depth, Vision tracking and Core ML models. They need `"apiVersion": 2` plus the matching capability. Without the
capability, reading `ctx.depth` or `ctx.vision` throws an error that names the manifest line to add.

General rules:

- Analysis never blocks rendering. Results trail the video by one or more frames (`age`, in seconds).
- Everything reads as empty until the first result lands. Use `ready` (or `count`) to fade in, and keep the plugin
  working without results. On first use a model may download for a while.
- Arrays are **host data views**: fixed-capacity typed arrays the host rewrites in place. Read them like
  `ctx.params` in `render` (no allocation), or bind them to kernel buffer arguments by name. Only the first `count`
  entries mean anything, so pass `count` as a uniform.
- Service textures are read-only: bind them as kernel inputs, never as `dst` or a `blit` target.
- Coordinates are normalised 0…1 in the same space as `uv` computed from `gid` (multiply by the texture size to get
  pixels). Left/right labels name the subject's own side.

## Depth (`depth`)

```json
"apiVersion": 2,
"capabilities": { "depth": { "source": "input" } }
```

| Member | Meaning |
|---|---|
| `ctx.depth.texture` | `r16Float` relative inverse depth (larger = nearer), same orientation as `ctx.input()` |
| `ctx.depth.range` | `Float32Array(2)` smoothed min/max: normalise with `(d - range[0]) / (range[1] - range[0])` |
| `ctx.depth.ready` | 1 once the first map landed |
| `age`, `generation`, `width`, `height` | seconds since the newest map, a counter, the map size |

Sample it with a linear sampler at the output `uv`; its resolution differs from the output. Quality follows
Arkestra's app-wide depth setting, and inference is shared with point clouds and splats.

## Vision (`vision`)

```json
"apiVersion": 2,
"capabilities": {
  "vision": { "source": "input", "faces": {}, "hands": { "maxHands": 4 } }
}
```

Request only the detectors you use; each costs CPU/Neural Engine time. Each takes `rate` (0.5–60 Hz, 15 default).

Every detector has `count`, `truncated`, `reported`, `age`, `generation`, and `ids` (stable per tracked subject,
0 = untracked). `smoothed.*` views are One-Euro-filtered copies of the raw ones: use them for drawing.

| Detector | Max | Views (floats unless noted) |
|---|---|---|
| `faces` | 64 | `rects` (x, y, w, h) · `ids` (Int32) · `angles` (roll, yaw, pitch, radians) · `confidence` · `quality` (with `captureQuality: true`) · `smoothed.rects` |
| `faceLandmarks` | 64 | `rects` · `ids` · `landmarks` (76 × (x, y) per face, unused 0) · `points` (Int32, 65 or 76) · `openness` (left eye, right eye, mouth, 0…1) · `smoothed.landmarks` |
| `hands` | `maxHands` | `ids` · `chirality` (Int32: 0 left, 1 right, −1 unknown) · `confidence` · `joints` (21 × (x, y, confidence)) · `smoothed.joints` (21 × (x, y, valid)) |
| `body` | 32 | `ids` · `confidence` · `joints` (19 × (x, y, confidence)) · `smoothed.joints` |
| `humans` | 64 | `rects` · `ids` · `confidence` · `upperBody` (Int32) · `smoothed.rects` |

Hand joint order: wrist; thumb CMC, MP, IP, tip; index MCP, PIP, DIP, tip; then middle, ring, little the same way.
Body joint order: nose, left eye, right eye, left ear, right ear, neck, left shoulder, right shoulder, left elbow,
right elbow, left wrist, right wrist, root, left hip, right hip, left knee, right knee, left ankle, right ankle.
A joint with confidence 0 wasn't observed.

Drawing tracked data (no draw passes yet, so a per-pixel kernel loops over the entries):

```js
pass = ctx.computePass("overlay", {
  textures: { src: ctx.input(), dst: ctx.output },
  buffers: { rects: ctx.vision.faces.smoothed.rects, joints: ctx.vision.hands.smoothed.joints },
  uniforms: { faceCount: 0, handCount: 0 }
});
// render:
pass.uniforms.faceCount = ctx.vision.faces.count;
pass.uniforms.handCount = ctx.vision.hands.count;
```

```metal
kernel void overlay(texture2d<float, access::sample> src [[texture(0)]],
                    texture2d<float, access::write> dst [[texture(1)]],
                    constant Uniforms &uniforms [[buffer(0)]],
                    device const float *rects [[buffer(1)]],
                    device const float *joints [[buffer(2)]],
                    uint2 gid [[thread_position_in_grid]]) { … rects[i * 4 + 0..3], joints[(h * 21 + j) * 3 + 0..2] … }
```

Keep the per-pixel loops short: use `maxHands` and only the detectors you need.

## Core ML models (`models`)

Plugins can run Core ML models from public Hugging Face repos. The host downloads, verifies, compiles, caches and
schedules them; the plugin stays network-free.

```json
"apiVersion": 2,
"capabilities": { "models": { "maxTotalMiB": 200 } },
"models": [{
  "key": "seg",
  "repo": "apple/coreml-detr-semantic-segmentation",
  "revision": "7c771f8867a479d1441ac5fb0a8de31feea76bb6",
  "path": "DETRResnet50SemanticSegmentationF16.mlpackage",
  "computeUnits": "neuralEngine",
  "rate": 10
}]
```

| Field | Rules |
|---|---|
| `key` | parameter-key grammar, unique; the name used in `ctx.model(key)` |
| `repo` | public `owner/name` (not gated or private) |
| `revision` | full 40-character commit sha, never a branch |
| `path` | `.mlpackage` or `.mlmodel` inside the repo |
| `computeUnits` | `neuralEngine` (default), `cpu`, `gpu` (competes with rendering) |
| `rate` | target inference rate, 0.5–60 Hz (15) |
| `normalize` | for MultiArray image inputs: `{ mean, std, scale }` |
| `temporalBlend` | 0 ≤ x < 1: blends each new texture output with the previous one |

At most 4 models; their total download size must fit `maxTotalMiB`. Look up the real sha and file path on the Hub
(the repo's commit history and file list) and don't guess them.

```js
function setup(ctx) {
  seg  = ctx.model("seg", { input: ctx.input(), rate: 10 });   // setup only
  mask = seg.output("semanticPredictions");                      // texture handle
  // probs = seg.data("classLabelProbs", 1000);                  // Float32Array view for small outputs
}
```

- `ready`, `state` (0 idle, 1 downloading, 2 compiling, 3 loading, 4 ready, 5 failed), `progress` (0…1), `age`,
  `generation`, `inferenceMs`, `runs`.
- `seg.labels()` returns class names once loaded. It allocates: call it once (when `ready` turns 1), not every frame.
- Supported I/O: one image input (or one Float16/32 MultiArray `[1,C,H,W]`/`[C,H,W]` with C ≤ 4), fixed or enumerated
  size. Image or ≤ 4-channel MultiArray outputs become textures through `output(name)`; small outputs (≤ 4096
  values, class probabilities) become data views through `data(name, count)`. Flexible size ranges, several required
  inputs and stateful models fail to load, with a console error naming the feature.
- Output textures are in plugin orientation and sample as empty until the first result.

Installing a plugin that uses depth, vision or models shows the user a consent sheet listing what it analyses and
downloads.
