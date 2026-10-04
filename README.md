# Arkestra skills

[Agent Skills](https://agentskills.io) for building things for [Arkestra](https://arkestra.app), the macOS live-visuals app. They work with Claude Code, Codex, Cursor and other tools that support skills.

| Skill | What it does |
|---|---|
| [`arkestra-plugin`](skills/arkestra-plugin/SKILL.md) | Build, debug and publish Arkestra plugins: `.arkplugin` bundles with a manifest, a `main.js` script and Metal compute kernels, running as video effects or generator layers. Covers when to use a plugin instead of an ISF shader, the script API, Metal rules, common patterns, depth/vision/Core ML services, and the test-and-publish loop. |

## Install

With the [skills CLI](https://github.com/vercel-labs/skills):

```sh
npx skills add itzaks/arkestra-skills
```

Or copy the skill folder into your tool's skills directory yourself:

```sh
git clone https://github.com/itzaks/arkestra-skills
cp -R arkestra-skills/skills/arkestra-plugin ~/.claude/skills/
```

For claude.ai, zip the `skills/arkestra-plugin` folder and upload it in the Skills section of your settings.

## Use

Ask your agent for a plugin, for example "Make an Arkestra plugin that leaves glowing trails behind moving objects". The skill tells it how to set up the project and test it against the running app. You'll get the most out of it with:

- **Arkestra running.** Plugins in `~/Documents/Arkestra/Plugins (Dev)/` hot-reload on every save, and Arkestra writes a status file the agent reads to see errors.
- **Optionally, Arkestra's MCP server** (MCP Server panel in Arkestra), so the agent can open a test project, capture frames and check performance itself:
  `claude mcp add --transport http arkestra http://localhost:9315/mcp`
- **Optionally, the `ark` CLI** (`@arkestra/plugin`) for validating and publishing, once it's released on npm.

## License

MIT
