# Re-fetch / Replay / Fetch-from-Source Patterns — Caesar → Daiton

The distilled, self-contained reference for the data-sync patterns behind
Daiton's **Fetch-from-Recall recovery flow** and every connector's data-freshness
guarantee. This is the pattern cluster that prevents silent data loss.

## The cardinal rule — never webhook-only

> **Every 3rd-party that emits events (calls, messages, recordings, webhooks —
> Retell, Recall, Fathom, Gmail, Slack, …) MUST have all three legs. A webhook
> alone is NOT an integration.**

Webhook-only integrations **silently lose data** — when a delivery is missed,
there's no recovery. This bug has shipped twice in the family. The three legs:

1. **Live webhook** — idempotent, signature-verified, **raw-first** (store before
   comprehend), so nothing in-flight is lost.
2. **Reconcile poll / `/sync`** — a cursor/watermark catch-up that re-pulls from
   the provider, so a dropped or never-delivered webhook is ALWAYS recovered.
   Overlap with the webhook is harmless — idempotency dedups.
3. **Backfill job** — a durable, resumable initial pull of history.

> **Caesar's Retell pattern is the exemplar:** raw-store `webhook_events` keyed by
> `call_id` + a `sync_calls(org_id)` reconcile — webhook + reconcile, both
> idempotent. (Note: Caesar has `scripts/test_scribe_latency.py` and similar
> replay-test harnesses; the idempotent-upsert shape is the reusable core, not a
> single `replay_webhooks.py` script.)

## The idempotent upsert key (the spine of every sync)

```python
# The five load-bearing parts — all five, or the mirror drifts.
# 1. external_id column  — the provider's stable id = the idempotency key (DB-unique index)
# 2. find_by_external_id — consulted on EVERY sync decision: found → update, missing → create
# 3. "settled" gate      — skip already-final rows (don't reprocess terminal records)
# 4. local-first ordering — for OUTBOUND create: write local FIRST → POST remote → stamp external_id
# 5. orphan-not-delete   — a local row absent from a complete remote listing is soft-deactivated, never hard-deleted

# ch-backend/src/modules/phone/repositories/call_repository.py
class CallRepository:
    def find_by_external_id(self, org_id, external_id) -> Call | None: ...

# ch-backend/src/modules/phone/services/call_service.py
def sync_one(provider_call: dict, org_id) -> Call:
    call = repo.find_by_external_id(org_id, provider_call["call_id"])
    if call:
        if service.is_call_complete(call):   # settled gate — skip finals
            return call
        # update fields from provider_call
        ...
    else:
        # create (inbound-originated object)
        call = Call(external_id=provider_call["call_id"], org_id=org_id, ...)
        repo.add(call)
    return call
```

**The "settled" gate** is the non-obvious part — don't refetch/reprocess terminal
records. Caesar's `is_call_complete = summary AND sentiment AND call_successful is
not None` (i.e. the post-call webhook has arrived). It's *not* a status enum by
default; it's "is this row's authoritative state fully landed?".

## Outbound create ordering (load-bearing — prevents orphaned remote objects)

For a local entity whose twin lives in a remote system (a Retell call, a Cognito
user, a Retell voice agent), an edit must travel out. **Local-first ordering:**

```python
# ch-backend/src/modules/phone/services/call_service.py — create_outbound_call
def create_outbound_call(db, org_id, ...) -> Call:
    # 1. write the local row FIRST (external_id null)
    call = Call(org_id=org_id, status="initiated", external_id=None, ...)
    db.add(call)
    db.flush()                          # so call.id exists

    # 2. POST remote, passing the LOCAL id as provider-side metadata
    provider_call = telephony.create_call(
        metadata={"internal_call_id": str(call.id), "org_id": str(org_id), ...}
    )
    #    ^ the inbound webhook correlates back via this metadata

    # 3. stamp the returned remote id onto external_id
    call.external_id = provider_call["call_id"]
    db.commit()
    return call
    # On POST failure: the local row survives (mark status="error"), re-raise.
    # It is NOT abandoned mid-flight.
```

**Reverse ordering (POST before the local row exists) orphans the remote object** —
it exists, but no local row references it, so the correlating webhook has nothing
to find.

## Outbound update — PATCH then re-persist

```python
# ch-backend/src/modules/phone/api/agents.py — PATCH /agents/{id}/config
@router.patch("/agents/{id}/config")
async def update_agent_config(id, config_in, org_id = Depends(require_org)):
    agent = repo.get(id, org_id)
    # 1. PATCH the remote via agent.provider_agent_id
    returned = retell.update_agent(agent.provider_agent_id, config_in)
    # 2. write the RETURNED config back into the local row (remote is authority)
    agent.configuration = returned["config"]   # trust its response, don't assume your body landed
    db.commit()
    return agent
```

The remote is the authority for the post-write shape; trust its response, don't
assume your request body landed verbatim.

## Reconcile = plan → apply, behind a per-provider registry

```python
# ch-backend/src/modules/phone/services/call_service.py — sync_calls_from_telephony
async def sync_calls_from_telephony(org_id, provider="retell"):
    adapter = TelephonyProviderRegistry.get(provider)   # strategy pattern
    remote_items = await adapter.list_calls(org_id)     # complete listing
    local_by_ext = {c.external_id: c for c in repo.list_all(org_id)}

    sync_work = {}  # PLAN first
    for item in remote_items:
        existing = local_by_ext.get(item["call_id"])
        sync_work[item["call_id"]] = None if not existing else existing  # None ⇒ create

    # APPLY concurrently, per-record failure isolated
    async def apply_one(ext_id, existing):
        async with db_session() as db:
            try:
                ... # upsert via find_by_external_id logic
                await db.commit()
            except Exception:
                await db.rollback()   # one bad record doesn't sink the batch
                log.exception(...)
    await asyncio.gather(*[apply_one(k, v) for k, v in sync_work.items()])

    # Orphan-not-delete: local rows whose external_id is absent from a COMPLETE listing
    for ext_id, local in local_by_ext.items():
        if ext_id not in {i["call_id"] for i in remote_items}:
            local.is_active = False   # soft-deactivate, NEVER hard-delete
```

**Key points:**
- Build a diff first (`sync_work`), then apply.
- Tolerate **per-record failure** (try/except + `rollback()` between records).
- Treat a remote `409/400 already-exists` as **success** (an idempotent POST is a feature).
- **Orphan = soft-deactivate** (`is_active=False`), never hard-delete; a later reconcile can reactivate.
- Resolve the adapter from a **per-provider registry** (`TelephonyProviderRegistry.register("retell", ...)`, adapters implement `TelephonyAdapter` ABC) so a second remote is a new strategy, not a new code path.

## Pick the variant first (the non-obvious decision)

```
Bounded set you can fully re-list each refresh?
  YES → SNAPSHOT CATALOG  (models, channels, voices, repos, numbers, agents, pipelines)
  NO  → INCREMENTAL LEDGER / BACKFILL  (high-volume, append-y: calls, messages, events, deals)
```

A single feature can need both (e.g. Stripe **products** = catalog; Stripe
**charges** = ledger). Split them.

| | Snapshot catalog | Incremental ledger / backfill |
|---|---|---|
| **For** | bounded picklists re-listed each refresh | unbounded / high-volume records |
| **Store** | one mirror/registry table, natural-key unique `(scope, provider_key)` | reuse `raw_events` (raw-first) + entity projection |
| **Refresh** | re-list all → upsert → **soft-disable rows not seen** | **cursor/watermark** pull of only what changed |
| **Endpoints** | `POST /{resource}/sync` + `GET /{resource}` | + a durable **job** for the initial backfill |
| **Trigger** | manual now, schedule later | webhook (live) + cursor poll (catch-up) + job (backfill) |
| **Caesar exemplar** | `modules/phone/api/{voices,agents}.py` (`/sync` + `*SyncResponse`) | `modules/phone/services/call_service.py` (`sync_calls_from_telephony`) |

### Snapshot catalog recipe

1. **Table**: provider's stable id + unique `(scope, provider_key)` as the upsert conflict target; `enabled`/`active` bool; `last_seen_at`; `raw` jsonb; scope/tenant column.
2. **`sync()` service**: re-list → `INSERT … ON CONFLICT DO UPDATE` per item → **soft-disable** items present locally but absent from a *complete* listing (never on a partial/failed fetch). Return `{added, updated, disabled}`.
3. **Endpoints**: `POST /api/v1/{resource}/sync` (the refresh) + `GET /api/v1/{resource}` (the picklist, `enabled_only` filter). Mirror Caesar's `Sync*Response`.
4. **Validate at the consumer**: anything that *uses* a value validates against the enabled registry with an actionable error.
5. **Schedule later**: point a Lambda/cron at `POST /sync`.

### Incremental ledger recipe

1. **Raw-first**: record each item into `raw_events` via `RawEventStore.record()` (idempotent on `(tenant_id, event_id)`) BEFORE comprehension.
2. **Watermark** in `source_connections.config.cursor_values[...]`; advance only after successful emit+commit (at-least-once).
3. **Idempotency tuple** `(tenant_id, connector, source_object_id, source_object_version)`; the version axis lets updates re-flow and supersede bi-temporally.
4. **Initial backfill = durable, resumable job**; **checkpoint the page cursor per page** so a crash resumes mid-walk.
5. **Ongoing = webhook + reconcile poll**. Overlap is harmless.
6. **Endpoints**: `POST /{resource}/backfill`, `GET /{resource}/backfill`, `POST /{resource}/sync`, `POST /{resource}/refresh-cursor`.

## Re-derive facts from a source (Daiton's `refresh_from_source`)

The same shape, inverted — instead of pulling from a provider, re-pull a
registered source artifact and re-run extraction so its derived facts supersede
bi-temporally. This is the **Fetch-from-Recall recovery flow** primitive:

- A **source** is registered (`register_source` / `source_objects` table) with a
  `connector` + `source_object_id` + content.
- Extraction derives **facts** from it, each carrying the `source_object_id` as
  provenance.
- `refresh_from_source(source_object_id=…)` **re-pulls** the source (re-fetch the
  artifact from its origin — the connector's fetch path), **re-runs extraction**,
  and the new facts **supersede** the old ones bi-temporally (old facts get an
  `invalidated_at`, new facts get a fresh `valid_at`).
- **Idempotent** on the `(tenant_id, connector, source_object_id, source_object_version)`
  tuple — re-running with the same version is a no-op; a changed version
  (`updated_at`/`date_updated`) re-flows and supersedes.

This is the recovery primitive: if a connector dropped or corrupted a delivery,
`refresh_from_source` re-derives the brain's facts from the original artifact.
Pair it with the reconcile/backfill legs so the *source* is always recoverable.

## Webhook signature verification (the Caesar/family standard)

Per-provider signature verification is hand-rolled and strong across the family.
Match the pattern when adding a new provider:

| Provider | Scheme | Source pattern |
|---|---|---|
| **Slack** | HMAC v0 + replay window | `slack/connector.py` |
| **Google (Gmail)** | OIDC JWT + constant-time hash compare | `gmail/connector.py` |
| **Recall** | Svix-scheme verification, fail-closed | `meeting_bot/recall.py` |
| **Generic HMAC** | HMAC with rotation grace | `webhook/receiver.py` |
| **Internal hops** (scribe ingest) | `X-Internal-Secret` shared secret | `scribe_ingest.py` |
| **Cognito user provisioning** | HMAC-signed internal endpoint, 300s replay window, constant-time | `internal_auth.py` (`hmac_signature.py`) |

> **Standardize** on binary-mode CloudEvents (`ce-type`/`ce-source` headers) for
> internal HTTP hops so middleware can route/filter without parsing bodies; use
> structured mode for storage and replay. Never invent custom header names for
> event metadata — the `ce-` prefix is the reserved namespace.

## Common mistakes

- **A webhook-only integration.** The cardinal sin. Add the reconcile leg as a required deliverable, not a follow-up.
- **Blind-insert instead of `find_by_external_id`.** Creates duplicates; always consult the idempotency key first.
- **POST remote before the local row exists.** Orphans the remote object. Always local-first, stamp `external_id` after.
- **Hard-deleting absent items.** Soft-deactivate (`is_active=False`), and only after a *complete* listing — never on a partial/failed fetch.
- **No "settled" gate.** Re-processing terminal records on every sync wastes work and can clobber settled state.
- **Trusting the request body over the remote response.** On outbound update, write the *returned* config back, not the request body.
- **Live-fetching on the read path.** Serve from the table; refresh is a separate write path (`POST /sync`).
- **Building the schedule first.** Manual `/sync` ships first; the Lambda/cron just calls it later.
- **Assuming listed = usable.** If the provider lists more than is usable (e.g. Bedrock inference profiles), probe and store `enabled`.
