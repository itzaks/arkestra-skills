# Testing and publishing

## The loop

1. Save a file. Arkestra re-validates the manifest, recompiles the kernels, and swaps every live instance to the new
   version, keeping parameter values by key (and persistent textures and `ctx.state`, unless `"hotReload": "reset"`).
2. If anything fails, the **last good version keeps running**, and the problems are reported.
3. Read the problems, fix them, save again.
4. When it loads cleanly, look at what it renders and check its performance.

A plugin that throws in `render`, runs over its script or GPU budget repeatedly, or faults the GPU is bypassed for the
session. Fixing it and saving reloads it.

## Seeing problems

**Status file (development folder only).** After every reload Arkestra writes `<slug>.status.txt` next to the
bundle in `~/Documents/Arkestra/Plugins (Dev)/<Plugin Name>/`:

```
error: "Pixelate" failed to load at 14:02:11. The last good version (0.1.0) is still running.
Paths are relative to pixelate.arkplugin/.
kernels/effect.metal:14:9 error: use of undeclared identifier 'uv'
```

or `ok: "Pixelate" 0.1.0 loaded at 14:02:11`. Problems can also be warnings. This works with nothing but file
access. Check that the time is after your save; if it isn't, wait a moment and read it again.

**`ark` CLI** (npm package `@arkestra/plugin`; run commands as `npx -p @arkestra/plugin ark <command>` from the
project folder, where `path` defaults to the current folder). If npm can't find the package, it isn't released yet:
rely on the status file and the MCP tools.

| Command | Use |
|---|---|
| `ark validate [--json]` | The same manifest and bundle rules as the app and the Marketplace, as `file:line:column: message [code]`. Run it before every pack/publish. It can't compile Metal; the app does that. |
| `ark dev [--no-launch]` | Links a bundle outside the development folder into the running app and streams console output, problems and reloads. Ctrl-C stops streaming; the link stays. |
| `ark pack` | Validates, then writes `dist/<name>-<version>.arkplugin.zip` plus a `.sha256`. |

**Arkestra MCP tools.** Turn on the MCP server in Arkestra (MCP Server panel; port 9315 by default) and connect the
agent, for Claude Code: `claude mcp add --transport http arkestra http://localhost:9315/mcp`.

| Tool | Use |
|---|---|
| `create_plugin` | `name`, `kind` (`effect`/`generator`): writes a starter project into the development folder. |
| `list_plugins` | Installed and development plugins with status and problem counts. |
| `reload_plugin` | `pluginId` or `path`, `reset?`: force a reload (saving already does this). |
| `get_plugin_problems` | Structured problems with file, line, column. |
| `get_plugin_console` | `console.log` output; pass `nextSince` back as `since` to page. |
| `open_plugin_test_project` | Opens a test project: camera + plugin (effects) or the plugin alone (generators). Replaces the open project; refuses with unsaved changes unless `force: true`, so **ask the user first**. Returns track/effect indexes. |
| `add_plugin_effect` | Adds the plugin to a track in the user's own project instead. |
| `capture_output` | Grabs a frame. Use `tap: "effect"` with the track/effect indexes from above, `delayFrames` for animated results, `stats: true` for numbers. Look at it before calling the plugin done. |
| `list_parameters` / `set_parameter` | Try parameter ranges, especially the extremes. |
| `get_plugin_performance` | Script and GPU ms (p50/p99/max) against the budgets, watchdog trips, memory. |
| `save_plugin_preview` | Captures the current output as `preview.png` (the grid thumbnail). |

`add_effect` doesn't add plugins; use `add_plugin_effect`.

## Before calling it done

- Status `ok` / no problems, no console errors.
- You've looked at a capture, at default parameters and at the extremes of each parameter.
- `get_plugin_performance` is comfortably under budget (script well under 3 ms, GPU under 4 ms at 1080p).
- No allocation in `render`, nothing logged every frame.
- Every parameter has a clear label and a useful range, and rhythmic behaviour is left to mappings.
- A 16:9 `preview.png` exists.

## Publishing

1. Give it a permanent id (`io.yourname.slug`, not `dev.local.*`), a `version`, an `author`, a `description` and an
   `icon`. From now on, never rename the id or parameter keys.
2. `ark validate`, then `ark login` (browser sign-in, stores a publish-only token).
3. `ark publish [--notes "What changed"] [--draft] [--dry-run]`. It validates and packs locally, finds the listing
   by the manifest id (or creates a draft), and uploads the version. Bump `version` for every upload.
4. The rest of the listing (screenshots, video link, pricing) is set on the web dashboard.

Plugins that declare `depth`, `vision` or `models` get badges on the listing ("analyzes video", "downloads a model").
Models must come from public Hugging Face repos pinned to a commit sha.

**Selling.** Arkestra doesn't process payments. A listing is free (the Marketplace hosts the zip) or paid: you set
a price label and a link to your own shop, which delivers the zip from `ark pack` or an
`arkestra://install?url=<https zip>&sha256=<hash>` link. Uploading the same version with `ark publish` gives buyers the
"Arkestra Marketplace" badge and update notices. Consider `"remix": false` for paid plugins, and `updateURL` for a
self-hosted update channel.

Environment for CI: `ARKESTRA_TOKEN` (creator token), `ARKESTRA_API_URL`.
