# Auth / Infra Patterns — Caesar → Daiton

The distilled, self-contained reference for the auth front door, the tenant model,
the FastAPI module skeleton, and the IaC. Read this before lifting any Caesar
auth, middleware, or module.

## The auth front door — four layers

Caesar's auth is a clean, layered pipeline. Daiton lifted the *plumbing* but
diverged on the *tenancy model* and *naming*. Know both.

### 1. JWT/JWKS validator (Cognito identity-only)

`ch-backend/src/shared/auth/jwt_validator.py` — the canonical validator, used
byte-identically by Caesar and Serenity.

```python
# ch-backend/src/shared/auth/jwt_validator.py  (shape, not verbatim)
from functools import lru_cache

def get_jwks_url() -> str:
    # f"https://cognito-idp.{region}.amazonaws.com/{COGNITO_USER_POOL_ID}/.well-known/jwks.json"
    ...

def get_issuer() -> str:
    # f"https://cognito-idp.{region}.amazonaws.com/{COGNITO_USER_POOL_ID}"
    ...

@lru_cache(maxsize=1)
def _fetch_jwks() -> dict:
    # GET get_jwks_url(); cache the keys in-memory (1h LRU). Never re-fetch per-request.
    ...

def verify_cognito_token(token: str, audience: str | None) -> dict:
    # 1. decode header → kid → find the matching JWKS key
    # 2. verify signature with the JWKS public key (PyJWT/cryptography)
    # 3. verify iss == get_issuer()
    # 4. verify aud == audience (or client_id) when configured
    # 5. verify exp + clock-skew tolerance
    # 6. return the claims dict
    ...

def extract_user_info(claims: dict) -> dict:
    # project the claims into a flat dict: sub, email, tenant_id (custom:tenant_id), role, ...
    ...
```

**Daiton adaptation:** Already lifted to
`backend/src/shared/auth/jwt_validator.py`, **pointed at Daiton's OWN Cognito
pool** (`COGNITO_USER_POOL_ID` — never Caesar's). Daiton's fork additionally
surfaces `custom:tenant_id` + `custom:role`. **No realignment needed — already
lifted verbatim.**

### 2. Tenant-context middleware (the deliberate divergence)

**Caesar** read tenancy from an `X-Organization-ID` **request header** — a second
trust boundary beyond the JWT. **Daiton** reads `custom:tenant_id` off the
**signed JWT claim** instead — one trust boundary, smaller attack surface.

```python
# Daiton: backend/src/shared/middleware/tenant_context_middleware.py
# (Caesar's is X-Organization-ID header-based; this is Daiton's JWT-claim variant)
async def tenant_context_middleware(request: Request, call_next):
    # 1. extract JWT from Authorization header
    # 2. verify_cognito_token(token, audience)
    # 3. claims = extract_user_info(...)
    # 4. request.state.tenant_id   = claims["tenant_id"]   # custom:tenant_id
    #    request.state.user_id     = claims["sub"]
    #    request.state.role        = claims["role"]
    #    request.state.is_super_admin = claims.get("is_super_admin", False)
    #    request.state.user_context = UserContext(...)
    return await call_next(request)
```

> **Load-bearing invariant:** everything downstream reads
> `request.state.{tenant|user}_id`. The middleware shape is
> `CORS → TenantContext → RateLimit(MCP) → Audit` (Daiton trims Caesar's
> `CorrelationId → RequestLogging → BodyCache → GZip` middleware to essentials).

### 3. RBAC ladder + permission strings

`ch-backend/src/shared/auth/rbac.py` + `dependencies.py`. The values are the
contract:

```python
# ch-backend/src/shared/auth/rbac.py  (Caesar + Daiton share this ladder)
from enum import IntEnum

class RoleLevel(IntEnum):
    SUPER_ADMIN = 100   # platform-wide; NOT a tenant role — set via is_super_admin flag,
                        # never assigned per-tenant, can never be created through RBAC.
    ADMIN       = 60
    STAFF       = 30
    VIEWER      = 10
    AUDITOR     = 10

# Enforcement helpers (ch-backend/src/shared/auth/dependencies.py):
#   require_role(min_level)        — FastAPI dependency; 403 if request.state.role < min_level
#   require_permission(perm)       — checks a permission string against the role
#   require_super_admin()          — request.state.is_super_admin must be True
```

**Permission strings** (e.g. `canPromoteToCanonical`, `canAnswerQuestion`,
`canOverrideAuthority`, `canEditSourceConfig`, `canEditLLMConfig`) are a
**contract surfaced to the frontend via `/me`** so the UI can disable controls;
actual enforcement is `require_role()` / `require_permission()` dependencies on
each endpoint.

> **Naming false-friend:** there's an `entity_roles` table in Daiton
> (`EntityRole` + `RoleStatus`) — these are **graph primitives, not auth**. It
> records that "JPMorgan plays the roles Vendor + Counterparty" in the company
> graph. It has nothing to do with user permissions.

### 4. HIPAA-grade audit middleware

`ch-backend/src/shared/middleware/audit_middleware.py`. Writes `access_logs` with
a `PHI_RESOURCES` frozenset (patients/calls/appointments/encounters…) and
sensitivity-aware payload redaction.

```python
# ch-backend/src/shared/middleware/audit_middleware.py  (shape)
PHI_RESOURCES = frozenset({"patients", "calls", "appointments", "encounters", ...})

async def audit_middleware(request: Request, call_next):
    response = await call_next(request)
    if request.method in {"POST","PUT","PATCH","DELETE"} or request.url.path in PHI_RESOURCES:
        # write an access_log row: user_id, organization_id, method, path,
        #   target_uuid (resource_id), event_type (action), timestamp,
        #   payload (redacted if PHI resource)
        ...
    return response
```

**Daiton adaptation:** Forked → writes `audit_events` (renamed/reshaped:
`organization_id→tenant_id`, `user_id→actor_id`, `resource_id→target_uuid`,
`action→event_type`). Resources are graph primitives (entities/facts/predicates/
episodes) instead of PHI. The redaction logic carries over.

## The tenancy model — the ONE axis where Daiton diverged

This is the most important thing to understand before lifting any Caesar auth.
Get it wrong and you'll build an org-membership system that doesn't fit Daiton.

| | Caesar (canonical) | Daiton (alone) |
|---|---|---|
| **Naming** | `organization` / `org_id` (~1856 refs) | `tenant` / `tenant_id` |
| **Tables** | real `organizations`, `users`, `user_organizations` (membership w/ role+status) | **NONE** — no `organizations`, no `users`, no membership table |
| **Header/claim** | `X-Organization-ID` **header** | `custom:tenant_id` **JWT claim** |
| **Context object** | `UserOrgContext` dataclass | `UserContext` / `request.state.tenant_id` |
| **Membership check** | `UserOrganization` row lookup | none — "the tenant IS the org" |
| **Isolation** | DB-level (org-scoped queries) | **app-layer** 6-layer stack, **NO Postgres RLS** |

**Do NOT lift Caesar's org/membership CRUD.** Daiton has no `organizations`/
`users`/`user_organizations` tables and won't get them without a substrate-wide
rename (deferred). The fork dependency file says it explicitly: *"Stripped
Caesar's User ORM lookup … Daiton has no concept of 'org' as a separate entity
(the tenant IS the org)."*

### Daiton's 6-layer app-layer tenant-scoping stack

`backend/src/database/tenant_context.py`. The real isolation — six layers, all
must hold:

1. **JWT** — `custom:tenant_id` claim on the signed token.
2. **Contextvar** — `with_tenant(tenant_id)` / `current_tenant_id()`, **fail-closed** (raises if no tenant set).
3. **Decorator** — `@require_tenant` on every service/repository method.
4. **SQL** — every query parametrizes `WHERE tenant_id = :t` (+ a dev/test `assert_tenant_scoped` regex that fails the test suite if a query lacks it).
5. **Neo4j** — `tenant_id` property on every node.
6. **Graphiti** — `group_id = org_{tenant_id}`.

> When you lift more auth from Caesar: **keep the JWT-claim boundary** (not
> headers), **keep app-layer tenant scoping** + the 6-layer stack; do **not**
> introduce Caesar's org tables. The bounded re-alignment (per Caesar-naming
> convention) is the membership model + header alias, not the substrate-wide
> rename.

### The tenant-switch chokepoint (Daiton improvement)

Caesar's tenant-switch is **inline** in `get_current_user_org_context`: super-admin
bypass (all orgs) else validate `UserOrganization` membership for
`X-Organization-ID`. Daiton **factored it out** into
`resolve_active_tenant(home_tenant_id, requested_tenant_id, is_super_admin)` that
**fails closed** — a header-requested tenant is honored only for verified
super-admins, else the JWT-home tenant wins. This is the single source of truth
across Daiton's two auth seams (`TenantContextMiddleware` for REST +
`AsgiAgentAuth` for `/mcp`). **This is a place Daiton improved the substrate and
could be lifted back into the family.**

## The FastAPI module skeleton (the feature-module quartet)

Identical across Caesar, Serenity, Content Stash AI, and Limelite V3. Daiton
diverged at the `modules/` layer (see below).

```
src/
├── main.py              # app-factory + LIFO middleware stack
├── core/                # config (pydantic-settings), security, logger
├── shared/
│   ├── auth/            # jwt_validator, rbac, dependencies
│   ├── middleware/      # tenant_context, audit, (rate_limit, correlation)
│   ├── services/        # cognito_service, storage_service, email_service, resource_service
│   ├── repositories/    # base repository pattern
│   └── schemas/         # shared Pydantic schemas
├── database/            # base.py (declarative base), session.py, models/
└── modules/<feature>/   # THE QUARTET — one dir per feature:
    ├── api/             #   FastAPI router (APIRouter), the HTTP boundary
    ├── models/          #   SQLAlchemy ORM models
    ├── services/        #   business logic (the orchestration layer)
    └── repositories/    #   data access (query layer)
```

**The quartet template** — `ch-backend/src/modules/users/` is the canonical
worked example:

- `modules/users/models/user.py` — `User(Base)` SQLAlchemy model.
- `modules/users/repositories/user_repository.py` — `UserRepository` with `find_by_*`, `create`, `update`.
- `modules/users/services/user_service.py` — business logic, calls the repository.
- `modules/users/api/users.py` — `APIRouter` with `@router.get/post/...`, depends on `require_role` / `require_org_role`, returns Pydantic response models via `response_model=`.

**`response_model` discipline (the contract-safety rule):** Caesar ships **256
`response_model=`** declarations, 0 `None` — every body-returning route declares
its response schema. This makes the OpenAPI schema codegen-ready (frontend TS
types generated from it). **Daiton routes are all `response_model=None` today**
(deliberate, for sparse `?fields=`) — and its hand-mirrored frontend types are
**already drifting**. The durable fix is to restore `response_model` (re-express
`?fields=` as a subset-projection) then codegen. See `design-system-patterns.md`
for the codegen side.

### Daiton's divergence at `modules/`

Daiton **abandoned the quartet** in favor of **flat, pipeline-stage modules**
(`ingest`, `extract`, `retrieval`, `entity_resolution`, `knowledge_store`,
`trust`) — files like `resolver.py`, `normalize.py`, `merge_policy.py`, zero
`api/services/models` subdirs. Routers live separately under `api/rest/*.py` as
`build_*_router()` factories. Storage is subsumed into a single
`database/substrate.py` facade (Postgres + Neo4j + pgvector + Graphiti).

> This is a **deliberate architectural divergence** (Daiton is a brain, not a
> CRUD app). Don't force the quartet back onto the pipeline modules. But the
> *auth/core/shared/database* skeleton above is identical and safe to lift from.

## The one app-factory + LIFO middleware stack

All family members build one FastAPI app and stack the same cross-cutting
middleware (LIFO order matters):

```python
# ch-backend/src/main.py  (shape — Caesar/Serenity)
def create_app() -> FastAPI:
    app = FastAPI(title=...)
    # LIFO stack — LAST added runs FIRST on the way in:
    app.add_middleware(GZipMiddleware)
    app.add_middleware(BodyCacheMiddleware)       # cache request body for audit/re-read
    app.add_middleware(AuditMiddleware)
    app.add_middleware(RequestLoggingMiddleware)
    app.add_middleware(CorrelationIdMiddleware)
    app.add_middleware(ReadonlyGuard)             # block writes in read-only mode
    app.add_middleware(CORSMiddleware, ...)        # FIRST on the way in, LAST added
    # ... include routers ...
    return app
```

**Daiton** keeps the shape, trims/renames to
`CORS → TenantContext → RateLimit(MCP) → Audit`. The shared contract:
**everything downstream reads `request.state.{tenant|user}_id`.**

## IaC — ECS reliability envelope (the ops dimension)

`ch-infra/` — 20 Terraform modules. The relevant one for Daiton's runtime gaps:

**`ch-infra/modules/emr-service-ecs/main.tf`** — the full ECS reliability envelope
that Daiton's hand-written task-defs lack:

- **Container `healthCheck`**: `CMD-SHELL curl -f localhost:<port><path>`, `interval=30`, `timeout=5`, `retries=3`, **`startPeriod=120`** (120s per LiveKit guidance so in-flight conversations aren't killed during startup).
- **ALB target-group `health_check`** in front of it.
- **Deployment safety**: `deployment_maximum_percent` / `deployment_minimum_healthy_percent` + `deployment_circuit_breaker { enable = true, rollback = true }`.
- **Autoscaling**: `aws_appautoscaling_target` + CPU and memory target-tracking policies. Prod: `desired_count=2`, `min=2`, `max=5`.
- Command: `python -m src.entrypoints agent`.

**`ch-infra/prod.tfvars`** carries the prod counts. Daiton deploys via
`aws ecs update-service --force-new-deployment` (not `terraform apply`, because
the service has `lifecycle { ignore_changes = [task_definition, desired_count] }`
and TF state is local) — but it should adopt the reliability *settings* from this
module. The #1 ops gap: Daiton's `scribe-agent-taskdef.json` has **no
`healthCheck`, no autoscaling, no circuit breaker** — a long-lived worker can
silently zombie (ECS shows `RUNNING` / health `UNKNOWN`, nothing replaces it).

## Footguns flagged in the secrets audit (fix while here)

- `ENVIRONMENT` defaults to `"dev"` (`core/config.py`) → the no-auth dev bypass is active unless a deploy sets `ENVIRONMENT=prod`. Add a fail-closed startup guard.
- FastAPI `/docs` `/redoc` `/openapi.json` are public in all envs — disable in prod.
- Secrets resolve at task START from SSM SecureString on a **specific KMS CMK**; a param encrypted under a different key fails silently at task start. `put-secret-value` alone does **NOT** restart — you must `--force-new-deployment`.
