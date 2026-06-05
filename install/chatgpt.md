# Install the Daiton Brain on ChatGPT

**Packaging:** ChatGPT reaches external tools through **connectors** (remote MCP servers) and lets you wrap behavior into a **custom GPT** (or a ChatGPT App). So the Daiton install is: add Daiton as a **connector**, then paste the playbook into the GPT's **Instructions** so the model knows how to drive the tools. The connector is the connection; the custom-GPT instructions are the `SKILL.md` adapted to ChatGPT's instruction box.

The reusable playbook is `../using-the-daiton-brain/SKILL.md`.

## 1. Add Daiton as a connector

Add a remote MCP connector pointing at:

```
https://mcp.daiton.ai/mcp/
```

ChatGPT connectors authenticate via **OAuth**, so add it by URL and complete the one-time sign-in when prompted — no token to paste. The Daiton tools (`search_entities`, `query_facts`, `assert_fact`, …) then become available to the model.

## 2. Build a custom GPT (or App) with the playbook as Instructions

Create a custom GPT, enable the Daiton connector on it, and put a condensed version of the core skill in the **Instructions** box (ChatGPT has no separate skills file — the Instructions field *is* where the playbook lives):

```text
You have access to Daiton, the company's shared knowledge graph ("company brain"), via its
connector. Use it for durable company knowledge (who/what/when about people, companies,
deals, decisions) and anything that should persist across chats.

READ: resolve a name to an id first — search_entities, or find_entity_by_alias for a
Slack/Salesforce/email id — then query_facts / query_facts_about / get_entity. Use
query_facts(as_of=…) for historical state.

WRITE: upsert_entity is the light path (no source needed). assert_fact REQUIRES a real
source_object_id + asserter_entity_id — only assert facts you have a genuine source for;
never fabricate or placeholder a source. merge = same record within a layer; link = same
real-world thing across layers; group = distinct things that belong together.

PRESENT: always surface ids to me as labeled, copyable pills, e.g. `entity_id` → `…`,
`fact_id` → `…`. Never bury a raw id in prose.
```

For the complete rules (merge-vs-link table, the full provenance section, the bitemporal `as_of` flow), keep `../using-the-daiton-brain/SKILL.md` as the source of truth and trim it to fit the Instructions length limit.

## 3. Confirm

Ask the GPT to run `health_check`. A healthy response confirms the connector.

## Auth mode

ChatGPT connectors use **OAuth** (add-by-URL, sign in once). See the **Connect** section of the core skill.
