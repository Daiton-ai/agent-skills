# Install the Daiton Brain on Manus

**Packaging:** Manus consumes Agent Skills as **`SKILL.md`** and connects to remote MCP servers as **connectors**. Because both are open standards, the install is essentially "add the connector + use the same `SKILL.md` verbatim" — there is almost nothing Manus-specific beyond the connector wiring. This is the cleanest case of the build-once thesis: the core playbook drops in unchanged.

## 1. Add the Daiton connector

In Manus, add a remote MCP connector pointing at:

```
https://mcp.daiton.ai/mcp/
```

Manus speaks **OAuth / Dynamic Client Registration** to remote MCP servers, so add it by URL and complete the one-time login when prompted — no token to paste. (If your Manus setup uses a static header instead, supply `Authorization: Bearer dai_…` from **Daiton → Settings → Agent tokens**.)

## 2. Add the skill (verbatim)

Use the core skill **as-is** — copy `../using-the-daiton-brain/SKILL.md` into your Manus skills location unchanged. The frontmatter (`name`, `description`) and body are platform-neutral and need no edits for Manus.

```bash
cp ../using-the-daiton-brain/SKILL.md ./using-the-daiton-brain.md   # then add to Manus as a Skill
```

## 3. Confirm

Ask Manus to run `health_check`. A healthy response confirms the connector; the skill then governs the read flow (resolve name → id → query), the write rules (`assert_fact` needs a real source), and the labeled-pill presentation convention.

## Auth mode

Manus uses **OAuth / DCR** by default (add-by-URL, log in once). Falls back to the **opaque `dai_` bearer token** if your deployment takes a static header. See the **Connect** section of the core skill.
