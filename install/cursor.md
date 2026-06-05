# Install the Daiton Brain on Cursor

**Packaging:** Cursor reads MCP servers from an `mcp.json`, and reads always-on guidance from **Project Rules** (`.cursor/rules/`, the modern replacement for a single `.cursorrules` file). So the Daiton install is two pieces: an **MCP config** (the connection) + a **Rule** (a condensed pointer to the playbook). The full playbook lives in `../using-the-daiton-brain/SKILL.md`; the Rule keeps Daiton top-of-mind and defers to it.

## 1. MCP config (the connection)

Create `.cursor/mcp.json` (project-scoped) — or `~/.cursor/mcp.json` for all projects:

```json
{
  "mcpServers": {
    "daiton": {
      "url": "https://mcp.daiton.ai/mcp/",
      "headers": {
        "Authorization": "Bearer dai_xxxxxxxxxxxxxxxx"
      }
    }
  }
}
```

Replace `dai_…` with your token from **Daiton → Settings → Agent tokens**. Because `mcp.json` may be committed, prefer a per-developer token and keep `.cursor/mcp.json` out of version control if your team shares the repo. Then open **Cursor → Settings → MCP** and confirm the `daiton` server shows its tools as connected.

## 2. Rule (the playbook pointer)

Create `.cursor/rules/daiton-brain.mdc`:

```markdown
---
description: Use the Daiton company brain for durable, shared company knowledge
alwaysApply: true
---
When a request needs durable company knowledge (who/what/when about people, companies,
deals, decisions) or cross-session memory, use the `daiton` MCP tools instead of guessing
or relying on private context.

- Resolve names to ids first: `search_entities` (or `find_entity_by_alias` for a Slack/
  Salesforce/email id), then `query_facts` / `query_facts_about` / `get_entity`.
- `upsert_entity` is the light path (no source). `assert_fact` REQUIRES a real
  `source_object_id` + `asserter_entity_id` — never fabricate a source; if you can't cite
  one, don't assert it.
- merge = same record within a layer; link = same real-world thing across layers; group =
  distinct things that belong together.
- ALWAYS surface ids as labeled, copyable pills: `entity_id` → `…`, `fact_id` → `…`.
```

For the full read/write rules, presentation convention, and the merge-vs-link table, the Rule defers to `../using-the-daiton-brain/SKILL.md`. (Legacy alternative: a single root `.cursorrules` file with the same body — Project Rules are preferred where supported.)

## Auth mode

Cursor uses the **opaque `dai_` bearer token** via the `headers` block above.
