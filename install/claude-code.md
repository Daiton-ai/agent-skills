# Install the Daiton Brain on Claude Code

**Packaging:** a Claude Code **Plugin** — an `.mcp.json` that points at the Daiton MCP server, the core `SKILL.md` bundled as a skill, and (optionally) a couple of slash commands. One-step install: the plugin carries the connection *and* the playbook together, so the agent gets both the tools and the instructions for using them.

The reusable playbook is `../using-the-daiton-brain/SKILL.md`. This file only adds the Claude-Code-specific packaging.

## Option A — drop-in (fastest)

1. **Add the MCP server.** Create or edit `.mcp.json` at your project root:

   ```json
   {
     "mcpServers": {
       "daiton": {
         "type": "http",
         "url": "https://mcp.daiton.ai/mcp/",
         "headers": {
           "Authorization": "Bearer ${DAITON_API_TOKEN}"
         }
       }
     }
   }
   ```

   Then export your token so it's never written into the repo:

   ```bash
   export DAITON_API_TOKEN=dai_xxxxxxxxxxxxxxxx   # from Daiton → Settings → Agent tokens
   ```

2. **Bundle the skill.** Copy the core skill into your project's skills directory so the agent loads the playbook automatically:

   ```bash
   mkdir -p .claude/skills/using-the-daiton-brain
   cp ../using-the-daiton-brain/SKILL.md .claude/skills/using-the-daiton-brain/SKILL.md
   ```

3. **Confirm.** In a Claude Code session, run `health_check` (no args). A healthy response means you're connected; the bundled skill governs how the agent reads/writes.

## Option B — as a Plugin (shareable, one command for teammates)

Package the connection + skill + slash commands as a plugin directory so anyone on the team installs it in one step.

```
daiton-brain-plugin/
├── .mcp.json                                   # the daiton server block above
├── skills/
│   └── using-the-daiton-brain/
│       └── SKILL.md                            # copied from ../using-the-daiton-brain/SKILL.md
└── commands/                                   # optional slash commands (see below)
    ├── recall.md
    └── remember.md
```

Install it the same way as any Claude Code plugin (add the plugin directory to your Claude Code plugin config / marketplace entry), then set `DAITON_API_TOKEN` in the environment.

### Optional slash commands

`commands/recall.md` — look something up in the brain:

```markdown
---
description: Look up what the company brain knows about a person, company, or topic
---
Resolve "$ARGUMENTS" in Daiton: search_entities to get the entity_id, then query_facts
and query_facts_about for it. Present each result with its `entity_id` / `fact_id` as a
labeled, copyable pill. Follow the using-the-daiton-brain skill.
```

`commands/remember.md` — write a fact back (with provenance):

```markdown
---
description: Record a fact in the company brain (only with a real source)
---
Record this in Daiton: "$ARGUMENTS". Use upsert_entity for any new entity, then assert_fact
ONLY if you have a genuine source_object_id and asserter_entity_id — never fabricate a source.
Surface the new `fact_id` and `source_object_id` as labeled pills. Follow the
using-the-daiton-brain skill.
```

## Auth mode

Claude Code uses the **opaque `dai_` bearer token** (static `Authorization` header). Use a distinct token per machine/agent so writes are attributed correctly. See the **Connect** section of the core skill.
