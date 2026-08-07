# bowmark plugin

The [Bowmark](https://bowmark.ai) plugin for **Claude Code**, **Codex**, and every client that reads **[Agent Plugins](https://agent-plugins.org/) 1.0.0** — installs the `bowmark` skill *and* auto-wires the hosted MCP server (`https://api.bowmark.ai/mcp`) in one step, so there's no separate `claude mcp add` / `config.toml` edit.

This package is the source of a one-way mirror to the public **[github.com/bowmark-ai/plugin](https://github.com/bowmark-ai/plugin)** (pushed on each `plugin-v*` release by [`release-plugin.yml`](../../.github/workflows/release-plugin.yml)) — same machinery as the skill mirror. The private monorepo can't be a public marketplace, but nothing here is private: only the public HTTP MCP URL and the public skill.

**One tree, three manifests, one payload.** Claude Code reads `.claude-plugin/`, Codex reads `.codex-plugin/` + `.agents/plugins/`, and an Agent Plugins client reads the root `plugin.json` + `mcp.json`. All three bundle the same skill and the same hosted MCP; nothing is duplicated but the manifest. Adding the third cost two files because the spec's conventional layout (`skills/<name>/SKILL.md`) is the layout this package already had.

**The Agent Plugins spec deliberately says nothing about installation** — no registry, no install command, no URL scheme; §"Client-managed installation, distribution, enablement, updates, and user interface are outside the portable specification." What is portable is the *directory*, so distribution is the same git repo it already was, and each host keeps its own way in.

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

## Install — an Agent Plugins client (ChatGPT, Cursor, Copilot, VS Code, Kiro, …)

There is no one command, because the spec defines no install path. Every host resolves a plugin from a **git repo or a local directory** and then reads the root `plugin.json`. Two shapes cover all of them:

```sh
# repo-level: anyone working in this checkout gets it
git clone https://github.com/bowmark-ai/plugin plugins/bowmark

# personal: the host's own plugin directory, e.g. Codex
git clone https://github.com/bowmark-ai/plugin ~/.codex/plugins/bowmark
```

Hosts with a marketplace UI (Cursor's Settings → Plugins → Team Marketplaces → Import from Repo, Codex's `/plugins` browser) point at `bowmark-ai/plugin` and read the manifests in-tree.

Any of the three routes gets you the skill + the MCP (tools `mcp__bowmark__get_library`, `mcp__bowmark__run`) and a card in the host's plugin directory.

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
├── plugin.json              # Agent Plugins 1.0.0 manifest — metadata ONLY, no component lists
├── mcp.json                 # Agent Plugins MCP config (streamable-http) — NOT the same file as .mcp.json
├── skills/
│   └── bowmark/
│       └── SKILL.md         # MIRROR of packages/skill/bowmark/SKILL.md — do NOT edit here
├── package.json             # release-please component `plugin` (stripped from the mirror)
└── sync-skill.sh            # regenerates the skill mirror
```

**Three MCP declarations, three shapes, and none of them can be shared.** Claude declares it **inline** in `.claude-plugin/plugin.json`; Codex points its manifest at `.mcp.json` (`{ name: { url } }`); Agent Plugins reads the root `mcp.json` (`{ $schema, mcpServers: { name: { type: "streamable-http", url } } }` — `type` is REQUIRED and is `streamable-http`, not the `http` the other two spell it). **`mcp.json` and `.mcp.json` are different files for different hosts** and the leading dot is the only thing telling them apart, so read the path twice before editing either.

`plugin.json` at the root carries **metadata only** — the schema is `additionalProperties: false` and has no `skills` or `mcpServers` field, unlike both sibling manifests. Components are found by convention (`skills/`, `mcp.json`), so there is nothing to keep in sync there and nothing to add when a skill is added.

**Each channel gets its own published destination** — `…/mcp/claude-code-plugin`, `…/mcp/codex-plugin`, `…/mcp/agent-plugins`, never the bare `…/mcp`. One segment carries both the install attribution and the platform pin. The first two **pin** their platform because a host-specific manifest knows its host at install time; **`agent-plugins` does not pin**, because the entire premise of the format is that six named hosts and anything else conformant read the same manifest — a pin there would assert one of six. Keep all three distinct when editing: pointing two at one segment would hand one host the other's operating text. Detection and the segment list live in [`apps/api/README.md`](https://github.com/Metroxe/bowmark/blob/main/apps/api/README.md#per-platform-instructions).

## The bundled skill is a mirror — edit canonical, then sync

`skills/bowmark/SKILL.md` mirrors `packages/skill/bowmark/SKILL.md`, the source of truth (minus its release-please version line — see `sync-skill.sh`). A plugin install needs the file in-tree, so it can't reference the separate skill package. **Never edit the copy directly.** After changing the canonical skill:

```sh
bash packages/plugin/sync-skill.sh           # copy canonical → plugin
bash packages/plugin/sync-skill.sh --check    # verify in sync (CI runs this)
```

CI fails any PR that changes the canonical skill without re-syncing.

## Versioning

release-please owns `package.json` `version` (component `plugin`, tag `plugin-v*`) and mirrors it into **all three** manifests — `.claude-plugin/plugin.json`, `.codex-plugin/plugin.json` and the root `plugin.json` — at `$.version` via json updaters, so each release republishes the plugin at the new version on every host. **A fourth manifest means a fourth `extra-files` entry**; miss it and that host serves a stale version forever with nothing failing. Ship changes under `feat(plugin):` / `fix(plugin):`.

## Validate

```sh
claude plugin validate packages/plugin     # Claude side has a local validator
```

There is no local `codex plugin validate`; the Codex manifests are built to the published spec ([developers.openai.com/codex/plugins](https://developers.openai.com/codex/plugins)) and confirmed on a real Codex install.

The Agent Plugins pair validates against the spec's own published JSON Schemas — the canonical copies live in [`agentplugins/agent-plugins-spec`](https://github.com/agentplugins/agent-plugins-spec) under `schemas/1.0.0/`, and both files were checked green against them (ajv, 2026-08-06). There is no CI gate for this: the schemas are fetched over the network and the fast gate is hermetic. Re-check by hand when either file changes.
