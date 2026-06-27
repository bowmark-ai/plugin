# bowmark plugin

The [Bowmark](https://bowmark.ai) plugin for **Claude Code** and **Codex** — installs the `bowmark` skill *and* auto-wires the hosted MCP server (`https://api.bowmark.ai/mcp`) in one step, so there's no separate `claude mcp add` / `config.toml` edit.

This package is the source of a one-way mirror to the public **[github.com/bowmark-ai/plugin](https://github.com/bowmark-ai/plugin)** (pushed on each `plugin-v*` release by [`release-plugin.yml`](../../.github/workflows/release-plugin.yml)) — same machinery as the skill mirror. The private monorepo can't be a public marketplace, but nothing here is private: only the public HTTP MCP URL and the public skill.

One repo serves both hosts: Claude Code reads `.claude-plugin/`, Codex reads `.codex-plugin/` + `.agents/plugins/`. Both bundle the same skill and the same hosted MCP.

## Install — Claude Code

```sh
claude plugin marketplace add bowmark-ai/plugin
claude plugin install bowmark@bowmark-ai
```

## Install — Codex

```sh
codex plugin marketplace add bowmark-ai/plugin
codex /plugins            # open the plugin browser, then install Bowmark
```

Either way you get the skill + the MCP (tools `mcp__bowmark__ask`, `mcp__bowmark__report_outcome`) and a card in the host's plugin directory.

Other surfaces are unchanged and live elsewhere: skill-only via `npx skills add bowmark-ai/skill` (works on Claude Code, Codex, Cursor, Copilot, OpenCode, …), MCP-only via `claude mcp add` / a Codex `[mcp_servers.bowmark]` block, or point any MCP client straight at the URL. See [bowmark.ai/#install](https://bowmark.ai/#install).

## Layout

```
packages/plugin/
├── .claude-plugin/          # Claude Code
│   ├── marketplace.json     #   marketplace: bowmark-ai (source "./")
│   └── plugin.json          #   plugin: bowmark — inline mcpServers, skills
├── .codex-plugin/           # Codex
│   └── plugin.json          #   plugin: bowmark — mcpServers "./.mcp.json", skills "./skills/"
├── .agents/plugins/
│   └── marketplace.json     # Codex marketplace (source {local, "./"})
├── .mcp.json                # MCP server config Codex's plugin.json references (HTTP, url)
├── skills/
│   └── bowmark/
│       └── SKILL.md         # MIRROR of packages/skill/bowmark/SKILL.md — do NOT edit here
├── package.json             # release-please component `plugin` (stripped from the mirror)
└── sync-skill.sh            # regenerates the skill mirror
```

Claude declares the MCP **inline** in `.claude-plugin/plugin.json`; Codex points its manifest at `.mcp.json`. The two MCP shapes differ (Claude wraps `{ mcpServers: { … } }` / Codex is `{ name: { url } }`), so they're kept separate rather than shared.

## The bundled skill is a mirror — edit canonical, then sync

`skills/bowmark/SKILL.md` mirrors `packages/skill/bowmark/SKILL.md`, the source of truth (minus its release-please version line — see `sync-skill.sh`). A plugin install needs the file in-tree, so it can't reference the separate skill package. **Never edit the copy directly.** After changing the canonical skill:

```sh
bash packages/plugin/sync-skill.sh           # copy canonical → plugin
bash packages/plugin/sync-skill.sh --check    # verify in sync (CI runs this)
```

CI fails any PR that changes the canonical skill without re-syncing.

## Versioning

release-please owns `package.json` `version` (component `plugin`, tag `plugin-v*`) and mirrors it into **both** `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json` `$.version` via json updaters, so each release republishes the plugin at the new version on both hosts. Ship changes under `feat(plugin):` / `fix(plugin):`.

## Validate

```sh
claude plugin validate packages/plugin     # Claude side has a local validator
```

There is no local `codex plugin validate`; the Codex manifests are built to the published spec ([developers.openai.com/codex/plugins](https://developers.openai.com/codex/plugins)) and confirmed on a real Codex install.
