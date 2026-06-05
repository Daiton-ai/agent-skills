# Install the Daiton Brain on GitHub Copilot

**Packaging:** Copilot (in VS Code) reads MCP servers from a workspace **`.vscode/mcp.json`** and reads always-on guidance from **`.github/copilot-instructions.md`**. So the Daiton install is two files: the MCP config (the connection) + a copilot-instructions block (a condensed pointer to the playbook). The full playbook lives in `../using-the-daiton-brain/SKILL.md`; the instructions file keeps Daiton in Copilot's context and defers to it.

## 1. MCP config (the connection)

Create `.vscode/mcp.json` in your workspace. Use an `inputs` prompt so the token is entered at runtime rather than committed:

```json
{
  "inputs": [
    {
      "type": "promptString",
      "id": "daiton-token",
      "description": "Daiton agent token (dai_…)",
      "password": true
    }
  ],
  "servers": {
    "daiton": {
      "type": "http",
      "url": "https://mcp.daiton.ai/mcp/",
      "headers": {
        "Authorization": "Bearer ${input:daiton-token}"
      }
    }
  }
}
```

Get the `dai_` token from **Daiton → Settings → Agent tokens**. VS Code will prompt for it on first use and keep it out of the committed file.

## 2. Instructions (the playbook pointer)

Create or append to `.github/copilot-instructions.md`:

```markdown
## Daiton company brain

When a task needs durable company knowledge (who/what/when about people, companies, deals,
decisions) or cross-session memory, use the `daiton` MCP tools rather than guessing.

- Resolve names to ids first: `search_entities` (or `find_entity_by_alias` for a Slack/
  Salesforce/email id), then `query_facts` / `query_facts_about` / `get_entity`. Use
  `query_facts(as_of=…)` for historical state.
- `upsert_entity` is the light path (no source). `assert_fact` REQUIRES a real
  `source_object_id` + `asserter_entity_id` — never fabricate a source; if you can't cite
  one, don't assert it.
- merge = same record within a layer; link = same real-world thing across layers; group =
  distinct things that belong together.
- ALWAYS surface ids as labeled, copyable pills: `entity_id` → `…`, `fact_id` → `…`.
```

For the full rules and the merge-vs-link table, this defers to `../using-the-daiton-brain/SKILL.md`.

## 3. Confirm

Open Copilot Chat in **Agent** mode (MCP tools are available there), and ask it to run `health_check`. A healthy response confirms the connection.

## Auth mode

GitHub Copilot uses the **opaque `dai_` bearer token** via the `headers` block above.
