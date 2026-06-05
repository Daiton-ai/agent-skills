# Daiton Agent Skills

**Connect any AI agent to your company's shared brain.**

[Daiton](https://daiton.ai) is a bitemporal knowledge graph — a **shared, source-backed company brain** that every agent *and* every person reads and writes. It's exposed as a remote **MCP server** at `https://mcp.daiton.ai/mcp/`. This repo packages Daiton as an **Agent Skill** so any MCP-capable platform can use it with a one-step install.

## Why this exists

Most AI agents keep a private, per-session scratchpad that no other agent or teammate can see — so the same knowledge gets re-derived, re-asked, and re-trusted on every call, and two agents can silently believe two different things. Daiton is the layer *above* that scratchpad: one governed source of truth, with provenance and history, that the whole fleet shares.

**MCP and Agent Skills (`SKILL.md`) are open standards.** That means you build the integration **once** — a single MCP server plus a single playbook skill — and re-skin it per platform. The thesis of this repo: **be the Daiton skill on every agent platform.** Every file in `install/` is a thin wrapper around the same core skill; only the packaging differs.

## What's in here

```
agent-skills/
├── README.md                         ← you are here
├── LICENSE                           ← MIT (placeholder)
├── CONTRIBUTING.md
├── using-the-daiton-brain/
│   └── SKILL.md                      ← THE core, reusable playbook (~90% of the value)
└── install/                          ← thin per-platform wrappers (packaging only)
    ├── claude-code.md
    ├── cursor.md
    ├── manus.md
    ├── chatgpt.md
    ├── claude-ai.md
    └── github-copilot.md
```

The **core skill** (`using-the-daiton-brain/SKILL.md`) is the heart: how to connect, the read flow (resolve name → id → query facts, with alias-based dedup and bitemporal `as_of` lookups), the write rules (`upsert_entity` light path vs `assert_fact` with **mandatory real provenance**; merge vs link vs group), and the presentation convention (always surface IDs as labeled, copyable pills). The per-platform files only describe how to *package and connect* it on each host.

## Install per platform

| Platform | Packaging | Auth mode | Guide |
|---|---|---|---|
| **Claude Code** | Plugin: `.mcp.json` + `SKILL.md` (+ optional slash commands) | `dai_` bearer token | [install/claude-code.md](install/claude-code.md) |
| **Cursor** | `mcp.json` + Project Rule (`.cursor/rules/`) | `dai_` bearer token | [install/cursor.md](install/cursor.md) |
| **Manus** | Connector + `SKILL.md` (verbatim) | OAuth / DCR | [install/manus.md](install/manus.md) |
| **ChatGPT** | Connector + custom GPT / App instructions | OAuth | [install/chatgpt.md](install/chatgpt.md) |
| **Claude.ai** | Uploaded Skill + remote Connector | OAuth / DCR | [install/claude-ai.md](install/claude-ai.md) |
| **GitHub Copilot** | `.vscode/mcp.json` + `copilot-instructions.md` | `dai_` bearer token | [install/github-copilot.md](install/github-copilot.md) |

New platform? If it speaks MCP, you usually need only (1) the server URL `https://mcp.daiton.ai/mcp/` with one of the two auth modes, and (2) the core `SKILL.md` (or a condensed pointer to it in whatever "instructions" surface the platform exposes). Copy the closest `install/*.md` as a template.

## Get this repo

```bash
git clone https://github.com/daiton-ai/agent-skills.git
```

> The repo slug `daiton-ai/agent-skills` is a **placeholder** — if it's renamed before/after publishing, it's a single find/replace across this repo.

## The #1 discovery move: register in the MCP Registry

The single highest-leverage distribution step is **listing the Daiton MCP server in the public [MCP Registry](https://github.com/modelcontextprotocol/registry)** (the open, community catalog of MCP servers). Once Daiton is in the registry, MCP-native clients and "add a connector" UIs can **discover and install it without anyone copy-pasting a URL** — it becomes a search result instead of a private link. Registering there is how "be the Daiton skill on every platform" turns from a per-user manual setup into ambient, pull-based discovery. Do this first; the per-platform installs above are the fallback for hosts that don't yet pull from the registry.

## Auth, in one line

- **OAuth / DCR** — for clients that add MCP servers by URL and log the user in (Claude.ai, ChatGPT, Manus). No token to paste.
- **Opaque `dai_` bearer token** — for clients that take a static `Authorization` header (Claude Code, Cursor, Copilot, scripts). Get it from **Daiton → Settings → Agent tokens**; treat it like a password; use a distinct token per agent so writes are attributed correctly.

## License

MIT — see [LICENSE](LICENSE). (Placeholder; confirm before publishing.)
