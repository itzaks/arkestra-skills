# manifest.json

One JSON object. Unknown keys are errors (they catch typos), so don't invent fields.

## Top-level fields

| Field | Req. | Rules |
|---|---|---|
| `id` | ✅ | Lowercase reverse-DNS, at least three parts: `^[a-z0-9]+(\.[a-z0-9-]+){2,}$`, ≤ 128 chars. `dev.local.<slug>` while developing. `app.arkestra.*` is reserved. **Never change it once shared.** |
| `name` | ✅ | 1–48 characters. |
| `version` | ✅ | `MAJOR.MINOR.PATCH`, no pre-release suffix. Bump it for every published release. |
| `apiVersion` | ✅ | `1`, or `2` for depth, vision and models. |
| `kind` | ✅ | `"effect"` (processes the incoming video) or `"generator"` (a layer that makes its own image). |
| `category` | — | Effect: `glitch` · `time` · `color` · `trace`. Generator: `simulation` · `signal` · `composition`. |
| `description` | — | ≤ 140 characters, shown under the name. |
| `icon` | — | An SF Symbol name, e.g. `"wand.and.stars"`, `"wind"`, `"circle.hexagongrid"`. |
| `author` | — | Display only. |
| `entry` | — | Script path, default `"main.js"`. |
| `pipelines` | — | `{ "<name>": { "type": "compute", "source": "kernels/x.metal", "function": "kernelName" } }`. Several pipelines may share one source file. |
| `parameters` | — | Ordered list (the order is the inspector order). |
| `capabilities` | — | Services and their load-time sizes (below). |
| `models` | — | apiVersion 2 only, with the `models` capability. See analysis-services.md. |
| `budgets` | — | Requested limits, clamped by the host (below). |
| `requiresHighPrecisionInput` | — | Effects only: receive a 16-bit float input instead of 8-bit. |
| `hotReload` | — | `"preserve"` (default: persistent textures and `ctx.state` survive a dev reload) or `"reset"`. |
| `remix` | — | `false` hides "Open as development copy" (consider it for paid plugins). Default `true`. |
| `updateURL` | — | https URL of a self-hosted update document, for plugins sold or hosted outside the Marketplace. |
| `marketplaceSlug` | — | Must equal the Marketplace listing slug if present. |

Paths are relative to the bundle root, use `/`, and can't contain `..` or start with `/`.

## Parameters

Common fields: `key` (required), `type` (required), `label` (1–32 chars, defaults to the key), `default`,
`hidden` (true: not shown but still mappable), `visibleWhen`.

`key`: `^[a-z][A-Za-z0-9_]{0,63}$`, unique, not `input`/`output`, not starting with `_`. Keys are the identity
saved projects and mappings use: **renaming one breaks users' projects**. Labels can change freely.

| Type | Fields | Script sees |
|---|---|---|
| `float` | `min` (0), `max` (1), `default` (real units, default `min`), `step`, `stepLabels` | number, clamped to `[min, max]` |
| `bool` | `default` (false) | 0 / 1 |
| `select` | `options` (1–64 unique strings), `default` (index) | index |
| `color` | `default` `[r, g, b]` 0…1 | `Float32Array(3)` |
| `point2d` | `min`/`max` `[x, y]` (default `[0,0]`/`[1,1]`), `default` `[x, y]` | `Float32Array(2)` |
| `text` | `default` (string), `maxLength` 1–1024 (256) | string; not mappable, but bindable to OSC |
| `texture` | none | not in `ctx.params`; use `ctx.input("<key>")` |

`float` details:
- There's no int type: integers are `float` with `"step": 1`.
- With `step`, `min`, `max` and `default` must each be a whole multiple of `step`.
- `stepLabels` turns a stepped float into a mappable dropdown. It needs `step` and `min: 0`, and exactly
  `round(max / step) + 1` labels.
- Floats are what users map to LFOs, audio and MIDI, so prefer a float over a bool/select when a value could be
  modulated.

### visibleWhen

Shows a control only while another parameter has some value. Purely cosmetic: hidden parameters keep rendering.

```json
"visibleWhen": { "key": "mode", "notEquals": 0 }
"visibleWhen": { "all": [ { "key": "mode", "in": [1, 2] }, { "key": "enabled", "equals": true } ] }
"visibleWhen": { "any": [ { "key": "glitch", "greaterThan": 0 }, { "key": "surge", "greaterThan": 0 } ] }
```

Operators: `equals`, `notEquals`, `in`, `notIn`, `greaterThan`, `lessThan`. Groups `all`/`any` nest up to 3 deep.
The referenced key must be another `float`, `bool`, `select` or `point2d` parameter. Floats compare in real units,
selects by index.

## Capabilities

An object; each key enables one service and sets its sizes. Leave it out when you don't need any.

| Capability | apiVersion | Config |
|---|---|---|
| `rasterize` | 1+ | `maxAtlases` 1–4 (1), `maxAtlasSize` 256–4096 (2048) |
| `depth` | 2 | `source`: `"input"` (effects) or a `texture` parameter key |
| `vision` | 2 | `source` as for depth; one key per detector: `faces`, `faceLandmarks`, `hands`, `body`, `humans`; each `{}` or `{ "rate": 15 }`; `hands` takes `maxHands` 1–64; `faces`/`faceLandmarks` take `captureQuality` |
| `models` | 2 | `maxTotalMiB` 1–2048 (600), together with the top-level `models` array |
| `highPrecisionInput` | 1+ | `{}`, effects only |

`audio`, `midi`, `history`, `texture3d` and `codec` are in the spec but have no runtime yet: don't declare them.
Generators have no input, so their `depth`/`vision` `source` must name a `texture` parameter.

## Budgets

Optional; the host uses `min(request, hard cap)`.

| Key | Default | Hard cap |
|---|---|---|
| `scriptMs` | 3 | 6 |
| `gpuMs` | 4 | 8 |
| `memoryMiB` | 256 | 1024 |
| `passesPerFrame` | 128 | 256 |
| `textures` | 16 | 32 |

Repeatedly going over `scriptMs` or `gpuMs` bypasses the plugin for the session.

## Bundle limits

- Allowed files: `.json .js .metal .png .jpg .jpeg .cube .txt .svg .csv`. No dotfiles, `.md`, `.ts`, binaries,
  `.metallib` or symlinks.
- Whole bundle ≤ 32 MiB, one file ≤ 16 MiB, all `.js` ≤ 1 MiB, each `.metal` ≤ 512 KiB.
