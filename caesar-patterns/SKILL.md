---
name: caesar-patterns
description: >-
  Distilled, self-contained blueprint of Caesar Health's reusable patterns for
  Daiton agents fork-and-adapting a Caesar UI, auth, infra, design-system, or
  data-sync surface — WITHOUT the Caesar repo cloned. Use whenever building or
  changing a Daiton frontend surface (catalog, entity/fact list or detail,
  master-detail, graph, agent, inbox, settings, users), wiring EntityConfig /
  EntityTable / EntityDetails / MasterDetailLayout / breadcrumbs / an entity card
  / the list toolbar, converging on Caesar's design system, lifting Caesar auth
  (Cognito JWT / JWKS / RBAC ladder / tenant middleware / audit), standing up
  FastAPI module skeleton, or building a re-fetch / webhook-replay / fetch-from-
  source recovery flow — even when the user never says "Caesar". Daiton is a
  fork-and-adapt of Caesar (ADR-011 "Caesar is the blueprint"); Caesar already
  solved most of these, so reach for this BEFORE building from scratch or
  half-measuring one surface at a time. This is the repo-free pocket reference;
  for the full internal lift methodology see the caesar-lift skill.
---

# Caesar Patterns — the fork-and-adapt blueprint (repo-free)

## What this is

Daiton is a **fork-and-adapt of Caesar Health** — ADR-011: *"Caesar is the
blueprint — fork-and-adapt, don't rebuild."* Caesar is a mature clinical EMR
monorepo (`ch-front-end/` React + TS + Vite + Tailwind v4 + shadcn/Radix +
TanStack Router/Query/Table; `ch-backend/` FastAPI + SQLAlchemy 2.0 + Alembic +
Cognito + ECS Fargate; `ch-infra/` Terraform). The stacks match Daiton almost
exactly, so most lifts are mechanical.

**The problem this skill solves:** Daiton cloud agents frequently need to lift a
Caesar pattern but **don't have the Caesar repo cloned** (it's private, and the
internal `caesar-lift` skill points at local paths like
`~/Development/ch-caesar-health` that don't exist in the cloud). This skill is the
**self-contained pocket reference**: the distilled pattern, the key Caesar source
path, a short excerpt, and how to adapt it into Daiton — enough to act without the
repo. It is deliberately selective (patterns + pointers), not a repo dump.

> **Companion (internal only):** the `caesar-lift` skill carries the full lift
> *methodology* — the five non-negotiables, the 8-step lift loop, render-verify,
> provenance headers. This skill carries the *patterns*. If you have access to
> `caesar-lift`, use both; if not (cloud, no internal skills), this skill alone is
> enough to identify and adapt the right pattern.

## The five non-negotiables (how to lift, not just what)

1. **Check Caesar first.** If a surface/feature exists in Caesar, lift it. Building
   from scratch when Caesar solved it is wasted effort and creates a second,
   inconsistent way to do the same thing.
2. **Lift the CONCEPT, not just the component.** Porting files without adopting the
   *mechanism* (the config-driven registry, the 6-layer tenant stack) is the #1
   trap. The mechanism is the value.
3. **Apply across the board.** A pattern lifted onto one surface but not its
   siblings drifts and reads as a bug. Consistency *is* the deliverable.
4. **Render-verify every UI change.** Typecheck-green is necessary, not sufficient.
   Actually render the page (drive a headless browser / CDP against the real local
   backend, seed dev auth, navigate to a real record) and screenshot it. A change
   can be real on disk but invisible to the user.
5. **Provenance header on every lifted file** (ADR-011):
   ```ts
   // Lifted from Caesar Health: ch-front-end/src/components/<path>
   // Caesar SHA at lift time: <short-sha>   ·   Adapted for Daiton: <one line>
   ```
   This makes future re-syncs (porting Caesar's bug fixes) auditable.

## The lift loop (mechanics)

1. **Locate** the Caesar source (paths in the sections below). Read it whole.
2. **Copy** into the mirroring Daiton path (`frontend/src/...` or `backend/src/...`).
3. **Rewrite imports** — Caesar's `@/` alias → Daiton's; swap the handful of
   `@radix-ui/react-icons` for `lucide-react`.
4. **Widen Caesar's EMR type unions to Daiton's domain.** Caesar's entities are
   clinical (Patient/Medication/Encounter…); Daiton's eight are
   **Person · Organization · Meeting · ActionItem · Topic · Decision · Fact ·
   Source**. Replace EMR literal unions and EMR design tokens with Daiton's.
5. **Typecheck** — `cd frontend && npm run typecheck` (`tsc -b --noEmit`) must
   exit 0.
6. **RENDER-VERIFY** (non-negotiable #4) — screenshot against the real backend.
7. **Provenance header** (#5).

## The four pattern domains

| Domain | Pattern summary | Deep dive |
|---|---|---|
| **A. Entity / list UI** | Config-driven spine: `EntityConfig<T>` → `EntityTable` + `EntityDetails` + `EntityCardBase` + `MasterDetailLayout` + breadcrumbs + list toolbar. One config per entity type drives every surface. | [`references/entity-ui-patterns.md`](references/entity-ui-patterns.md) |
| **B. Auth / infra** | Cognito JWT/JWKS validator → tenant-context middleware (JWT claim, not header) → RBAC ladder → HIPAA audit. The feature-module FastAPI skeleton. | [`references/auth-infra-patterns.md`](references/auth-infra-patterns.md) |
| **C. Design system** | shadcn/Radix primitives + Tailwind v4 + `cn()` + TanStack Router/Query/Table; stack is byte-compatible with Daiton. | [`references/design-system-patterns.md`](references/design-system-patterns.md) |
| **D. Re-fetch / replay / fetch-from-source** | The webhook + reconcile + backfill trinity (never webhook-only); idempotent upsert keyed by `external_id`; raw-first ingest; `refresh_from_source` re-derivation. | [`references/refetch-replay-patterns.md`](references/refetch-replay-patterns.md) |

Read the section below for the 60-second orientation, then open the matching
`references/*.md` for the actual excerpts and adaptation steps before you build.

---

## A. The entity / list UI system (60-second orientation)

> Full detail: [`references/entity-ui-patterns.md`](references/entity-ui-patterns.md)

**The through-line:** *one `EntityConfig<T>` per entity type drives every surface*
— its table columns, its detail sections, its card, its graph node, its chat
citation — from a single declaration. Add a new entity type = write one config,
not touch six surfaces.

**The Caesar source map (the files that matter):**

| Caesar file | What it is |
|---|---|
| `ch-front-end/src/types/entity-config.ts` | The `EntityConfig<T>` master type + the lower-tier config types (`EntityTableConfig`, `EntityDetailsConfig`, `EntityCardConfig`). |
| `ch-front-end/src/components/data-table/entity-table.tsx` (~990 lines) | `EntityTable<T>` — the generic server-side-paginated table with toolbar, sort, faceted filters, view-options, card mode. |
| `ch-front-end/src/components/layout/entity-details.tsx` (244) | `EntityDetails<T>` — the section-model detail body (`info` / `metadata` / `actions` / `custom`). |
| `ch-front-end/src/components/layout/master-detail-layout.tsx` (189) | Two-panel responsive master-detail (mobile overlay, collapsible list, compact card mode). |
| `ch-front-end/src/components/emr-core/cards/base/entity-card-base.tsx` | `EntityCardBase` — one generic record card; ~34 domain cards built on it. |
| `ch-front-end/src/context/navigation-provider.tsx` + `components/navigation/breadcrumb-navigation.tsx` | The breadcrumb trail/stack provider. |
| `ch-front-end/src/features/contacts/{index.tsx, components/*}` | **THE template** — the worked example of wiring table + columns + filters + detail + card together. |

**The config type spine** (distilled from `entity-config.ts`):

```ts
// The lower tiers are the load-bearing convention Caesar actually ships.
type EntityTableConfig<T> = {
  data: T[]; columns: ColumnDef<T>[]; totalCount: number;
  selectedItem?: T; onSelectItem?: (t: T) => void; getItemId: (t: T) => string;
  searchConfig?: SearchConfig; filters?: FilterConfig<T>[]; urlState: TableUrlState;
  cardConfig?: EntityCardConfig<T>; /* + loading flags, compact mode */
};
type EntityDetailsConfig<T> = {
  entity?: T; emptyState?: ReactNode;
  sections: ({ type: 'info'; label: string; value: ...; format?: 'date'|'badge'|'text'|'custom' }
           | { type: 'metadata' } | { type: 'actions'; buttons: ... }
           | { type: 'custom'; render: (t: T) => ReactNode })[];
};
type EntityCardConfig<T> = {
  getName: (t: T) => string; getTimestamp?: (t: T) => string;
  getCategories?: (t: T) => string[]; getSubtitle?: (t: T) => string;
  getStatusDisplay?: (t: T) => { label: string; variant: BadgeVariant };
};
// The MASTER — one per entity type. Factory functions return the per-surface config.
type EntityConfig<T> = {
  name: string; namePlural: string; icon: ReactNode;
  table?: (props) => Omit<EntityTableConfig<T>, 'data' | 'totalCount' | ...>;
  details?: (props) => Omit<EntityDetailsConfig<T>, 'entity'>;
  // Daiton extends with: graphNode, graphEdge, chatCitation, chatMention, linkerPreview
};
```

**The `EntityCardBase` prop contract** (the anatomy that matters):

```text
icon            ReactNode   — leading type glyph
title           string      — primary line
subtitle?       string      — secondary line (muted)
status?         string + statusVariant  — a Badge (default/secondary/destructive/outline)
reason?         string      — a separated explanatory line (line-clamped)
metadata?       {label,value}[]  — key/value rows
children?       ReactNode   — domain-specific body
aiGenerated? + confidence?  — a subtle "AI (NN%)" footer  ← trust signal, first-class
widgets?        ('add'|'edit'|'remove')[] + onWidgetAction  — inline row actions
onClick?        — make the whole card a click target
size            'compact' | 'standard' | 'detailed'  ← progressive disclosure
```

**The worked example** (`features/contacts/index.tsx` — copy this shape):

```tsx
<EntityPageLayout headerConfig={{ icon: Users, title: 'Contacts', /*…*/ }}>
  <MasterDetailLayout
    mode={mode}
    listPanel={
      <EntityTable
        data={contacts} columns={columns} totalCount={total}
        selectedItem={selectedContact} onSelectItem={handleSelectContact}
        getItemId={getItemId}
        searchConfig={SEARCH_CONFIG} filters={CONTACT_FILTERS}
        urlState={urlState} compact={mode === 'master-detail'}
        cardConfig={{ getName, getTimestamp, getCategories, getSubtitle }}
      />
    }
    detailPanel={selectedContact && <ContactDetailView contact={selectedContact} /*…*/ />}
  />
</EntityPageLayout>
```

**Honest nuance:** even in Caesar, the *master* `EntityConfig<T>` factory is
**aspirational** — there is no `useEntityConfig` hook and no
`EntityConfig<Contact>` instance in the Caesar tree. What Caesar *actually ships
and reuses heavily* is the **lower tier**: every feature hand-composes
`EntityTable` + `EntityDetails` fed per-feature `ColumnDef[]` + section arrays + a
`cardConfig`. The generic **components** are the load-bearing convention; the
master factory is the formalization on top. Don't wait for a perfect registry —
wire the components per feature the Caesar way.

**Daiton adaptation:** Daiton has *already ported* `EntityTable`,
`MasterDetailLayout`, `EntityPageLayout`, navigation-provider,
breadcrumb-navigation, and json-viewer — but the `EntityConfig` registry is
**vendored but dormant** (`registerEntityConfig` is never called), `EntityDetails`
is ported but **unused**, and there is **no card model**. The highest-leverage
moves: (1) author one config per Daiton entity type and call
`registerEntityConfig('person', personConfig)` at module load; (2) lift
`EntityCardBase` verbatim into `frontend/src/components/entity-card/` and build
~4 thin domain cards (`PersonCard`, `OrgCard`, `MeetingCard`, `FactCard`) — the
`aiGenerated`+`confidence` footer is a perfect fit for Daiton's extraction/ER
confidence; (3) retire the `Sheet` popover for fact detail in favor of a routed
`EntityDetails`. See the deep dive for the adaptation table.

---

## B. Auth / infra (60-second orientation)

> Full detail: [`references/auth-infra-patterns.md`](references/auth-infra-patterns.md)

**The front door — Caesar's auth plumbing, adapted:**

| Layer | Caesar source | Daiton adaptation |
|---|---|---|
| **JWT/JWKS validator** | `ch-backend/src/shared/auth/jwt_validator.py` — `get_jwks_url()`/`get_issuer()` from `COGNITO_USER_POOL_ID`+region, 1h JWKS LRU cache, `verify_cognito_token` + `extract_user_info`. | Already lifted to `backend/src/shared/auth/jwt_validator.py`, **pointed at Daiton's OWN Cognito pool** (never Caesar's), additionally surfaces `custom:tenant_id` + `custom:role`. **No realignment needed.** |
| **Tenant-context middleware** | Caesar read tenancy from an `X-Organization-ID` **header**. | **Deliberate divergence:** Daiton reads `custom:tenant_id` off the **signed JWT claim** (one trust boundary, not two) → `request.state.tenant_id`. |
| **RBAC ladder** | `ch-backend/src/shared/auth/rbac.py` + `dependencies.py`. | Same ladder: `super_admin=100 → admin=60 → staff/viewer/auditor=10`; enforced via `require_role`/`require_permission`/`require_super_admin` in `shared/auth/dependencies.py`. |
| **HIPAA audit middleware** | `ch-backend/src/shared/middleware/audit_middleware.py` — writes `access_logs` with a `PHI_RESOURCES` frozenset. | Forked → writes `audit_events`; renames `organization_id→tenant_id`, `user_id→actor_id`, `target_uuid←resource_id`; resources are graph primitives (entities/facts/predicates). |

**The RBAC ladder (the values are the contract):**

```python
# Caesar + Daiton share this ladder (ch-backend/src/shared/auth/rbac.py)
class RoleHierarchy:
    ROLE_HIERARCHY = {
        "super_admin": 100,   # platform-wide; NOT a tenant role; set via is_super_admin flag
        "admin": 60,
        "staff": 30,
        "viewer": 10,
        "auditor": 10,
    }
# super_admin can never be created through RBAC — it's a platform flag.
# Per-role permission strings (canPromoteToCanonical, canAnswerQuestion, ...)
# are surfaced to the frontend via /me so the UI can disable controls;
# actual enforcement is require_role() / require_permission() deps on each endpoint.
```

**The FastAPI module skeleton** (identical across the family):

```
src/
├── core/            # config, security, logger
├── shared/
│   ├── auth/        # jwt_validator, rbac, dependencies, (tenant_override, super_admin in Daiton)
│   ├── middleware/  # tenant_context, audit, (rate_limit in Daiton)
│   ├── services/    # cognito_service, storage_service, email_service, resource_service
│   ├── repositories/
│   └── schemas/
├── database/        # base, session, models
└── modules/<feature>/   # THE QUARTET: api + models + services + repositories
                          # e.g. modules/users/{api,models,services}/,
                          #      modules/organizations/{api,models,services}/
```

**The one axis where Daiton diverged (important):** Caesar/Serenity/Content-Stash
all use `organization` naming — real `organizations`/`users`/`user_organizations`
tables, header `X-Organization-ID`, `UserOrgContext`. **Daiton alone** uses
`tenant` — a bare `tenant_id` **string** on every row (no `tenants`/`Organization`
table, no membership table), isolation enforced in **application code** (6-layer
stack), **no Postgres RLS**. So **do not** lift Caesar's org/membership CRUD — it
doesn't match Daiton's model. The 6-layer stack (JWT → contextvar
`with_tenant`/`current_tenant_id`, fail-closed → `@require_tenant` → parametrized
SQL `WHERE tenant_id = :t` → Neo4j `tenant_id` property → Graphiti
`group_id = org_{tenant_id}`) is the real isolation. See the deep dive.

---

## C. Design system primitives (60-second orientation)

> Full detail: [`references/design-system-patterns.md`](references/design-system-patterns.md)

**The stack is byte-compatible with Daiton** — lifts are mechanical:

| Layer | Caesar | Daiton | Compatible? |
|---|---|---|---|
| **UI primitives** | shadcn/ui + `@radix-ui` | shadcn/ui + `@radix-ui` | ✓ |
| **Styling** | Tailwind **v4** + `cn()` (clsx + tailwind-merge) | Tailwind v4 + `cn()` | ✓ |
| **Icons** | mostly `lucide-react` (some `@radix-ui/react-icons`) | `lucide-react` only | ✓ (swap the handful of Radix icons on port) |
| **Data** | TanStack Table / Query / Router | TanStack Table / Query / Router | ✓ |
| **State** | Zustand (scoped stores) | Zustand | ✓ |

**The only adaptation rule:** Caesar's `@/` import alias → Daiton's alias; replace
any `@radix-ui/react-icons` import with the `lucide-react` equivalent. Everything
else (`Card`/`Badge`/`Button`/`Separator`/`ScrollArea`/`Sheet`/`Dialog`/`Command`)
exists identically in Daiton. When you see Caesar design tokens
(`var(--emr-...)`), swap them for Daiton's tokens.

---

## D. Re-fetch / replay / fetch-from-source (60-second orientation)

> Full detail: [`references/refetch-replay-patterns.md`](references/refetch-replay-patterns.md)

This is the pattern cluster behind Daiton's **Fetch-from-Recall recovery flow**
and every connector's data-freshness guarantee.

**The cardinal rule — never webhook-only (every 3rd party, no exceptions):**

> Every provider that emits events (calls, messages, recordings, webhooks — Retell,
> Recall, Fathom, Gmail, Slack, …) MUST have **all three legs**. A webhook alone is
> NOT an integration — it silently loses data when a delivery is missed.

1. **Live webhook** — idempotent, signature-verified, **raw-first** (store before
   comprehend).
2. **Reconcile poll / `/sync`** — a cursor/watermark catch-up that re-pulls from
   the provider, so a dropped/never-delivered webhook is ALWAYS recovered. Overlap
   with the webhook is harmless — idempotency dedups.
3. **Backfill job** — a durable, resumable initial pull of history.

**Caesar's Retell pattern is the exemplar:** raw-store `webhook_events` keyed by
`call_id` + a `sync_calls(org_id)` reconcile — webhook + reconcile, both
idempotent. Source: `ch-backend/src/modules/phone/services/call_service.py`
(`sync_calls_from_telephony` = plan→apply with per-record try/except+rollback;
`is_call_complete` = the "settled" gate that skips final rows),
`ch-backend/src/modules/phone/api/agents.py` (`PATCH /agents/{id}/config` =
PATCH-remote-then-re-persist), `ch-backend/src/modules/phone/repositories/call_repository.py`
(`find_by_external_id`).

**The idempotent upsert key (the spine of every sync):**

```python
# external_id = the provider's stable id = the idempotency key (DB-unique index).
# find_by_external_id consulted on EVERY sync decision: found → update, missing → create.
# Never blind-insert. Never hard-delete an absent row — soft-deactivate (is_active=False).
call = repo.find_by_external_id(provider_call_id)
if call:
    # update — but skip if "settled" (e.g. is_call_complete = summary AND sentiment AND ...)
    ...
else:
    # create — LOCAL FIRST: db.add(call) → flush() → POST remote → stamp external_id
    ...
```

**Outbound create ordering (load-bearing — prevents orphaned remote objects):**
write the local row FIRST (`external_id` null) → `flush()` so it has an id → POST
remote, passing the **local id as provider-side metadata**
(`metadata={"internal_call_id": str(call.id), ...}`) so the inbound webhook
correlates back → stamp `row.external_id = remote["id"]`. On POST failure, the
local row survives (mark `error`); it is not abandoned mid-flight. **Reverse
ordering orphans the remote object.**

**Re-derive facts from a source (Daiton's `refresh_from_source`):** the same
shape, inverted — instead of pulling from a provider, re-pull a registered source
artifact and re-run extraction so its derived facts supersede bi-temporally.
Idempotent on the `(tenant_id, connector, source_object_id, source_object_version)`
tuple; the version axis (`updated_at`) lets updates re-flow and supersede.

See the deep dive for the snapshot-catalog vs incremental-ledger decision tree and
the full recipe for each.

---

## When NOT to lift from Caesar

- **The brain/knowledge substrate.** Daiton deliberately *diverged* from Caesar's
  feature-module quartet into flat pipeline-stage modules (`ingest`, `extract`,
  `retrieval`, `entity_resolution`, `knowledge_store`, `trust`) and a single
  `substrate.py` facade (Postgres + Neo4j + pgvector + Graphiti). Don't cage the
  open-ontology, predicate-fact, bitemporal brain in Caesar's per-type models.
- **Org/membership CRUD.** Daiton has no `organizations`/`users`/`user_organizations`
  tables — "the tenant IS the org." Don't lift Caesar's membership model.
- **Clinical/EMR-specific logic.** Caesar's `EncounterEngine`, clinical card
  renderers, PHI resource lists — de-medicalize, don't copy.

## Quick reference — which Caesar file for which job

| You're building… | Start from (Caesar) |
|---|---|
| A new entity list/table | `ch-front-end/src/components/data-table/entity-table.tsx` + `features/contacts/index.tsx` (template) |
| An entity detail / reading pane | `ch-front-end/src/components/layout/entity-details.tsx` + `features/contacts/components/contact-detail-view.tsx` |
| Master-detail page | `ch-front-end/src/components/layout/master-detail-layout.tsx` |
| An entity card | `ch-front-end/src/components/emr-core/cards/base/entity-card-base.tsx` + `cards/base/entity-list-card.tsx` |
| Breadcrumbs / nav | `ch-front-end/src/context/navigation-provider.tsx` + `components/navigation/breadcrumb-navigation.tsx` |
| List toolbar / filters / view-options | `ch-front-end/src/components/data-table/{toolbar,pagination,column-header,faceted-filter,view-options}.tsx` |
| JWT auth | `ch-backend/src/shared/auth/{jwt_validator,rbac,dependencies}.py` |
| Tenant context | `ch-backend/src/shared/middleware/audit_middleware.py` + Daiton's `tenant_context_middleware.py` (JWT-claim variant) |
| A new FastAPI feature module | `ch-backend/src/modules/users/{api,models,services}/` (the quartet template) |
| File/S3 storage | `ch-backend/src/shared/services/storage_service.py` |
| A provider sync (webhook + reconcile) | `ch-backend/src/modules/phone/services/call_service.py` (`sync_calls_from_telephony`) + `modules/phone/api/agents.py` |
| A snapshot catalog (models/voices/agents) | `ch-backend/src/modules/phone/api/{voices,agents}.py` (`/sync` + `*SyncResponse`) |
| ECS deploy reliability | `ch-infra/modules/emr-service-ecs/main.tf` (healthCheck + autoscaling + circuit-breaker) |

> **Stack note:** Caesar repo paths use `ch-front-end/` and `ch-backend/`. Daiton
> uses `frontend/` (or `apps/frontend/`) and `backend/` (or `apps/backend/`). Map
> the prefix when you port.
