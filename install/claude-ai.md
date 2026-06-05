# Install the Daiton Brain on Claude.ai

**Packaging:** Claude.ai takes two open-standard pieces directly: an uploaded **Skill** (the `SKILL.md`) and a remote **Connector** (the MCP server). Nothing is re-skinned — you upload the same skill file and add the same server URL. This is the build-once thesis at its purest on the consumer side.

The reusable playbook is `../using-the-daiton-brain/SKILL.md`.

## 1. Add the Daiton connector

In Claude.ai, add a custom **Connector** pointing at:

```
https://mcp.daiton.ai/mcp/
```

Claude.ai connectors authenticate via **OAuth / Dynamic Client Registration**, so add it by URL and complete the one-time login when prompted — no token to paste.

## 2. Upload the skill

Upload `../using-the-daiton-brain/SKILL.md` as a **Skill** in Claude.ai — verbatim, no edits. Its `name`/`description` frontmatter and body are platform-neutral. Once uploaded, Claude loads the playbook whenever a request matches the description (durable company knowledge, cross-session memory, entity/fact lookups).

## 3. Confirm

Start a chat with the connector enabled and ask Claude to run `health_check`. A healthy response confirms the connector; the uploaded skill governs the read flow, write rules, and the labeled-pill presentation convention.

## Auth mode

Claude.ai uses **OAuth / DCR** (add-by-URL, log in once). See the **Connect** section of the core skill.
