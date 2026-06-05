---
name: using-the-daiton-brain
description: Use when an agent needs durable company knowledge — who/what/when about people, companies, deals, projects, decisions, or facts that should persist across sessions and be shared with the rest of the team's agents and humans. Triggers on cross-session memory, "what do we know about X", "remember this for next time", looking up an entity by a Slack/Salesforce/email id, checking a fact's history or source, or recording a new fact with provenance. Daiton is the shared, source-backed company brain (a bitemporal knowledge graph) exposed over MCP; reach for it before answering from a private scratchpad or guessing.
---

# Using the Daiton Brain

## Overview

Daiton is your company's shared **brain** — a bitemporal knowledge graph of entities (people, organizations, projects, deals, decisions…) and facts about them, each fact carrying a **source** and a **time**. It's exposed as a remote MCP server, so every agent on every platform and every human reads and writes the *same* truth.

**Core principle:** Most agents keep a private, per-session scratchpad that no one else can see. Daiton is the layer above that — when something matters beyond this conversation, **read it from Daiton and write it back to Daiton** so the next agent (and the next person) inherits it.

**Two things make Daiton different from a notebook, and you must respect both:**
1. **Provenance is sacred.** A fact only earns a place in the brain if it has a real source. Never invent one.
2. **IDs are the currency.** Entities and facts are addressed by stable IDs. Resolve names to IDs before you act, and surface IDs back to the user so they can act on them too.

## When to use

Reach for Daiton when:
- The question is about durable company knowledge: "what do we know about Acme Corp?", "who owns the Q3 migration?", "what did we decide about pricing?"
- You need memory that outlives this session, or that another agent/teammate should see.
- You're handed an external id (Slack user id, Salesforce account id, email address) and need the canonical entity behind it.
- You learned something worth keeping — a decision, a status change, a new relationship — and it has a real source you can point to.
- You need the *history* of a fact: what was true as of some past date, or what superseded what.

**When NOT to use:**
- Throwaway, in-conversation scratch state that no one else needs → keep it local.
- Secrets, credentials, or anything you wouldn't want every agent and teammate in the org to read.
- Pure computation, formatting, or tasks with no knowledge-lookup component.

## Connect

**MCP server URL:** `https://mcp.daiton.ai/mcp/`

Two auth modes — pick the one your platform supports (the per-platform install guide tells you which):

| Mode | Use when | How |
|---|---|---|
| **OAuth / Dynamic Client Registration (DCR)** | The client speaks OAuth to remote MCP servers (Claude.ai, ChatGPT, Manus, and most "add a connector by URL" UIs) | Point the client at the URL; it runs the OAuth/DCR handshake and the user logs in once. No token to paste. |
| **Opaque bearer token (`dai_…`)** | The client takes a static header (Claude Code, Cursor, Copilot, CLIs, scripts) | Send `Authorization: Bearer dai_xxxxxxxx`. Get the token from the Daiton dashboard → **Settings → Agent tokens**. Treat it like a password; never commit it. |

A `dai_` token stamps every write with *which agent* wrote it — that's how Daiton attributes provenance across the fleet. Use a distinct token per agent/integration so the brain knows who said what.

Verify the connection any time with `health_check` (no arguments) — it returns server status without touching data.

## The read flow

**Always resolve a name to an id first.** Tools that operate on a specific entity or fact take **ids**, not free-text names. The path is: *name → search → id → query*.

1. **Resolve.** `search_entities(query="Acme Corp")` → ranked candidates, each with an `entity_id`. Pick the right one (disambiguate by type/aliases if several match).
2. **Read facts.** Feed that `entity_id` to:
   - `query_facts(entity_id=…)` — facts where this entity is the subject.
   - `query_facts_about(entity_id=…)` — facts that *mention* this entity (it appears anywhere in the fact, not just as subject). Use this for "everything we know touching X".
   - `get_entity(entity_id=…)` — the entity record itself (type, attributes, aliases).
3. **Resolve across sources with aliases.** When you hold an external id from another system — a Slack user id, a Salesforce account id, an email — use `find_entity_by_alias(alias="U07ABC123")` (or the email, the Salesforce id, …) to jump straight to the canonical Daiton entity. This is how you **dedup across sources**: the same person showing up from Slack, email, and the CRM resolves to one entity.
4. **Read history (bitemporal).** To ask "what was true *then*", pass a timestamp: `query_facts(entity_id=…, as_of="2026-01-15T00:00:00Z")`. This returns the state of knowledge as of that date — superseded facts, point-in-time values, "what did we believe in January". Without `as_of`, you get the current truth.

> Rule of thumb: if a tool wants an id and you only have a name, you skipped step 1. Go back and `search_entities` (or `find_entity_by_alias`) first.

## The write flow

There are two write paths. Choosing the right one — and respecting provenance — is the whole game.

### `upsert_entity` — the light path (no source required)

Use `upsert_entity` to create or update an **entity** (the node): a new person, a company you just heard of, an attribute on an existing entity. It's idempotent — give it a name/type (and an `entity_id` if you're updating a known one) and it creates-or-merges. No `source_object_id` needed; entities are containers, not claims.

```text
upsert_entity(name="Acme Corp", type="Organization", attributes={"domain": "acme.com"})
```

### `assert_fact` — the heavy path (REAL provenance REQUIRED)

Use `assert_fact` to record a **claim** about the world: a status, a relationship, a decision, a value. A fact is only worth trusting because of where it came from, so `assert_fact` **requires**:
- `source_object_id` — a pointer to the **genuine source** the claim came from (the meeting/transcript/message/document object in Daiton that says it). 
- `asserter_entity_id` — **who** is making the assertion (the agent or person on whose authority this goes in).

**The provenance rule — non-negotiable:**

> **Only assert a fact when you have a real source for it. Never fabricate, guess, or back-fill a `source_object_id` to satisfy the argument.**

A made-up source is worse than no fact — it poisons the brain for every agent and human downstream, and it's the one thing Daiton exists to prevent. If you *believe* something but can't point to where it came from, you have two honest options:
- **Don't assert it.** Leave it out, or surface it to the user as an unverified observation.
- **Create the source first.** If the claim genuinely originates from a real artifact you have (a transcript, an email, a doc), ingest/register that artifact so it has a `source_object_id`, *then* assert against it. The source must actually exist and actually contain the claim.

Red flags that mean STOP:
- "I'll just put the conversation id / a placeholder as the source." → No. The source must be the artifact that *contains the claim*.
- "It's obviously true, so the source doesn't matter." → It does. Unsourced-but-true and fabricated-source are both forbidden; pick an honest option above.
- "I'll assert it now and someone can add the source later." → No. Source-at-assertion is the contract.

### Merge vs link vs group — collapsing and relating entities

When the graph has duplicates or related entities, use the right verb (each has an inverse for when you get it wrong):

| You have… | Use | Inverse | Meaning |
|---|---|---|---|
| **Two records of the *same* thing in the *same* layer** (e.g. "Acme Corp" and "Acme Corporation", both from the CRM) | `merge_entities` | `unmerge_entities` | **Collapse** the dupes into one entity. Destructive-ish: they become one node. |
| **The *same* real thing seen across *different* layers/sources** (the Slack person ↔ the Salesforce contact ↔ the email identity) | `link_entities` | `unlink_entities` | Assert `same_as` *across* layers without collapsing — each source keeps its own node, but they're known to be the same. |
| **Several *distinct* entities that belong together** (a team, a deal's accounts, a project's people) | `group_entities` | `ungroup_entities` | Relate them as a set; they stay separate entities. |

Rule of thumb: **merge = "these are literally the same record, fuse them" (within a layer); link = "these are the same real-world thing, but keep the per-source nodes" (across layers); group = "these belong together but are different things."** If unsure between merge and link, prefer `link` — it's reversible without data loss.

`refresh_from_source(source_object_id=…)` re-pulls a source and re-derives its facts (use after a connector updates upstream).

## Presentation convention — ALWAYS surface IDs as labeled pills

Daiton entity and fact IDs are how the user (and you, next time) act on the brain — to open it in the UI, correct it, cite it, or feed it back to a tool. **Never bury a raw UUID inside a sentence.** Always surface it as a copyable, **labeled** pill so it's obvious what the id *is*.

Render every id as `` `<label>` → `<id>` ``:

> Acme's renewal owner is **Dana Lee** (`entity_id` → `ent_7f3a…`), per the Jan 14 QBR.
> Recorded: Acme moved to the Enterprise plan (`fact_id` → `fct_91be…`, `source_object_id` → `src_22aa…`).

Good (labeled, copyable):
```text
- `entity_id`        → `ent_7f3a2c10-…`
- `fact_id`          → `fct_91be0d44-…`
- `source_object_id` → `src_22aa9f31-…`
```

Bad (raw id buried in prose):
```text
The owner is Dana Lee with id 7f3a2c10 and the fact is 91be0d44.
```

When you list several facts or entities, put the id pill next to **each** one. The label tells the reader which tool that id plugs into next; the value is the thing they copy. This is non-negotiable formatting — it's what turns "the agent said something" into "the user can act on it."

## Quick reference

| Goal | Tool(s) |
|---|---|
| Check the server is up | `health_check` |
| Name → entity | `search_entities` |
| External id (Slack/SFDC/email) → entity | `find_entity_by_alias` |
| The entity record | `get_entity` |
| Facts where entity is the subject | `query_facts` |
| Facts that mention the entity | `query_facts_about` |
| State as of a past date | `query_facts(…, as_of=…)` |
| Create/update an entity (no source) | `upsert_entity` |
| Record a claim (**real source required**) | `assert_fact` |
| Fuse same-layer dupes / undo | `merge_entities` / `unmerge_entities` |
| Same thing across layers / undo | `link_entities` / `unlink_entities` |
| Relate a set / undo | `group_entities` / `ungroup_entities` |
| Re-derive facts from a source | `refresh_from_source` |

## Common mistakes

- **Passing a name where an id is required.** Resolve with `search_entities` / `find_entity_by_alias` first.
- **Asserting a fact with a fabricated or placeholder source.** Forbidden — don't assert it, or create the real source first.
- **Using `assert_fact` to create an entity, or `upsert_entity` to record a claim.** Entities (nodes) vs facts (claims) are different paths.
- **Merging across layers.** Cross-source "same person" is `link_entities` (keeps per-source nodes), not `merge_entities`.
- **Dropping IDs into prose.** Always surface them as labeled pills so the user can copy and act.
- **Re-deriving truth from scratch each session.** If you already wrote it to Daiton, read it back next time instead of re-asking.
