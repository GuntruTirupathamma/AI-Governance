---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 21 of 35
Previous: DAY20.md - Cost Governance and Budget Enforcement
Next: DAY22.md - Real-Time Alerting and Notification Channels
---

# Day 21: Multi-Tenant Isolation and Data Segregation

## Table of contents

- [The problem with single-tenant assumptions](#the-problem)
- [What multi-tenant isolation means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 21.1 - The tenant registry](#step-211)
- [Step 21.2 - Tenant context and the org header](#step-212)
- [Step 21.3 - Namespacing cache keys by organization](#step-213)
- [Step 21.4 - Org-scoped database columns](#step-214)
- [Step 21.5 - Threading org_id through the audit system](#step-215)
- [Step 21.6 - Cross-tenant access checks](#step-216)
- [Step 21.7 - Org-aware secret resolution](#step-217)
- [Step 21.8 - Wiring org_id through streaming and the orchestrator](#step-218)
- [Step 21.9 - Abuse detection for cross-tenant probing](#step-219)
- [Step 21.10 - Tests](#step-2110)
- [What you built today](#what-you-built)
- [Day 21 checklist](#checklist)

<a id="the-problem"></a>
## The problem with single-tenant assumptions

Every day so far has built on `X-User-ID` and `X-User-Role` (Day 4). Those two headers identify a *person* and a *role*, and twenty days of governance - rate limits, budgets, secrets, audit - have been keyed off them. Nothing identifies which **organization** that person belongs to, because until now this system has only ever had one.

That assumption breaks the moment a second customer signs up. Say Acme Corp and Globex Inc both deploy this platform. Each has an analyst named `alice` - same username, different companies, completely unrelated people. Walk through what happens today:

- Day 9's streaming counters write to `cost:user:alice:2026-06-13` and `stream:active:alice`. Acme's Alice and Globex's Alice share both keys. Globex's token spend inflates Acme's daily total, and if Globex's Alice has an active stream, Acme's Alice gets `429 Too Many Concurrent Streams` for a slot she never opened.
- Day 20's `budget:user:alice:2026-06-13` and `budget:user:alice:2026-06` are the same story - one shared budget meter for two unrelated accounts. Whichever org's Alice spends first exhausts the cap for both.
- Day 20's `GET /billing/usage/{user_id}` endpoint trusts `X-User-Role: admin` and nothing else. Acme's admin can pass `alice` (or any user_id they can guess) and read Globex's spend, limits, and remaining budget. There is no check that the user_id being queried belongs to the caller's organization - because the concept of "the caller's organization" doesn't exist yet.
- Day 17's audit log is one global hash chain, one table, no tenant column. An Acme admin investigating "did someone tamper with our audit trail" would, if given any query access at all, see Globex's security events mixed into the same chain.
- Day 19's `AGENT_SECRET_SCOPES` hands every `research` agent the same vault-resolved API keys regardless of which customer's agent is asking. If Acme and Globex each have their own third-party API key for the same provider, today's vault has no way to give each org its own.

None of this is a bug in any single day's code - each day solved a real problem for a single tenant. The gap is structural: there has never been a dimension above `user_id` that says *which organization does this belong to*, so nothing can be partitioned, and nothing can be checked at a boundary.

Day 21 adds that dimension - `org_id` - as a first-class concept that flows through identity, caching, the database, and authorization, layered onto the twenty days of work that already exist without rewriting any of it.

<a id="what-it-means-here"></a>
## What multi-tenant isolation means here

Multi-tenant isolation in this system means four things, in increasing order of how much they touch:

1. **A tenant registry.** A small, explicit list of known organizations - like Day 14's `AGENT_REGISTRY` or Day 19's `SECRET_REGISTRY` - so an `org_id` can be validated instead of trusted blindly.
2. **A tenant context on every request.** A new `X-Org-ID` header, read alongside Day 4's `X-User-ID` / `X-User-Role`, validated against the registry, and rejected with a clear error if unknown. Fail closed: an unrecognized org is not "treated as default", it's a 403.
3. **Namespaced storage.** Every Redis key built in Days 9, 18, and 20 - cost counters, budget totals, abuse signal windows, active-stream locks - gets an `org:{org_id}:` segment, so two orgs with identically-named users never collide. Every database row written in Days 17 and 9 - audit log entries and stream sessions - gets an `org_id` column, so queries can be scoped.
4. **A checked boundary.** Cross-org reads - the exact gap in Day 20's `/billing/usage/{user_id}` - are checked against a stored user-to-org mapping and rejected with a new audit event, `TENANT_ACCESS_DENIED`, which Day 18's abuse detector now also watches for.

What this is *not*: a rewrite of the data model into per-tenant databases or schemas, a multi-region or sharding story, or a billing/plan-tier system (Day 21 adds a `tier` field to the tenant record because it's nearly free to do so, but nothing enforces tier-based limits yet - that's future work). The goal is the minimum set of changes that makes "Acme's data and Globex's data cannot leak into each other" true across everything built through Day 20.

<a id="what-youll-build"></a>
## What you will build today

- `backend/tenancy/tenant_schema.py` - `Tenant`, `TenantTier`, and the `TENANT_REGISTRY` of known organizations.
- `backend/tenancy/context.py` - `TenantContext`, the `get_tenant_context()` FastAPI dependency that reads `X-Org-ID` and validates it.
- `backend/tenancy/user_directory.py` - a tiny Redis-backed mapping from `user_id` to the `org_id` that user belongs to, written on every authenticated request and read at the cross-tenant boundary.
- Updates to `backend/billing/budget_tracker.py`, `backend/cache/redis_client.py` callers, `backend/security/abuse_thresholds.py` (Day 18), and Day 9's stream counters - every cache key gains an `org:{org_id}:` segment.
- A migration adding `org_id` columns to `audit_log` (Day 17) and `stream_sessions` (Day 9), both indexed, both defaulting to `"default"` for existing rows.
- Updates to `backend/audit/audit_schema.py` and `backend/audit/audit_logger.py` (Day 17) - `AuditEvent`, `AuditRecord`, and every `AuditLogger` method now carry `org_id`.
- A new `AuditEventType.TENANT_ACCESS_DENIED` and the cross-tenant check it represents, applied to Day 20's billing router and Day 19's secret-scopes endpoint.
- Updates to `backend/secrets/vault.py` and `CredentialBroker` (Day 19) - org-aware secret resolution with a global fallback.
- Updates to `GovernedStreamManager` (Day 9) and `AgentOrchestrator` (Day 14) - `org_id` threaded alongside `user_id`/`user_role` everywhere.
- A new `ABUSE_THRESHOLDS` entry for repeated `TENANT_ACCESS_DENIED` events.
- Tests under `tests/tenancy/`, plus updates to `tests/billing/`, `tests/audit/`, and `tests/secrets/`.

<a id="step-211"></a>
## Step 21.1 - The tenant registry

Like Day 14's agent registry and Day 19's secret registry, tenants start as an explicit, in-code list. No self-service signup yet - that's a product feature for a later day. Today, an `org_id` is either in this registry or it doesn't exist.

```python
# backend/tenancy/tenant_schema.py

from dataclasses import dataclass
from enum import Enum


class TenantTier(str, Enum):
    """
    Reserved for future tier-based limits (e.g. higher BUDGET_LIMITS
    for enterprise tenants). Day 21 only records the tier - nothing
    reads it yet.
    """
    STANDARD = "standard"
    ENTERPRISE = "enterprise"


@dataclass(frozen=True)
class Tenant:
    org_id: str
    name: str
    tier: TenantTier


TENANT_REGISTRY: dict[str, Tenant] = {
    "acme": Tenant(org_id="acme", name="Acme Corp", tier=TenantTier.ENTERPRISE),
    "globex": Tenant(org_id="globex", name="Globex Inc", tier=TenantTier.STANDARD),

    # "default" exists so that every example from Days 1-20 - which never
    # sent an X-Org-ID header - keeps working without modification. New
    # integrations should always send a real org_id; "default" is a
    # migration aid, not a recommended tenant.
    "default": Tenant(org_id="default", name="Default Organization", tier=TenantTier.STANDARD),
}


def get_tenant(org_id: str) -> Tenant | None:
    return TENANT_REGISTRY.get(org_id)


def is_known_tenant(org_id: str) -> bool:
    return org_id in TENANT_REGISTRY


def all_org_ids() -> list[str]:
    return list(TENANT_REGISTRY.keys())
```

The `"default"` entry deserves a second look, because it's the difference between "fail closed" and "silently merge everyone into one tenant." It exists purely so that a request with no `X-Org-ID` header - which describes every request made by Days 1 through 20's code and curl examples - still resolves to a *known, single* tenant rather than being rejected outright. The isolation guarantee Day 21 cares about is: **two different known orgs never share data**. A caller that never sends `X-Org-ID` is, by definition, not claiming to be any particular org, so it lands in `"default"` alongside every other caller that also didn't send one. The moment a deployment has two real customers, both get assigned a real `org_id` and `"default"` becomes unused.

<a id="step-212"></a>
## Step 21.2 - Tenant context and the org header

Day 4 introduced `X-User-ID` and `X-User-Role`, read with `.get(..., "anonymous")` / `.get(..., "employee")` fallbacks - permissive by design, because Day 4 explicitly deferred real authentication. `X-Org-ID` follows the same pattern for the *fallback*, but not for *validation*: an org_id that doesn't exist in `TENANT_REGISTRY` is a 403, not a guess.

```python
# backend/tenancy/context.py

from dataclasses import dataclass

from fastapi import Request, HTTPException

from backend.tenancy.tenant_schema import is_known_tenant


@dataclass(frozen=True)
class TenantContext:
    """
    The three identity dimensions every governed request now carries:
    which organization, which user within it, and what role that user
    holds. Built once per request by get_tenant_context() and passed
    down to routers, the orchestrator, and the stream manager.
    """
    org_id: str
    user_id: str
    user_role: str


def get_tenant_context(request: Request) -> TenantContext:
    org_id = request.headers.get("X-Org-ID", "default")
    user_id = request.headers.get("X-User-ID", "anonymous")
    user_role = request.headers.get("X-User-Role", "employee")

    if not is_known_tenant(org_id):
        raise HTTPException(
            status_code=403,
            detail=f"Unknown organization: '{org_id}'. Contact an administrator to provision this tenant.",
        )

    return TenantContext(org_id=org_id, user_id=user_id, user_role=user_role)
```

Wire it into a router as a dependency, the same way Day 4's headers were read inline:

```python
# backend/routers/billing.py (excerpt - full update in Step 21.6)

from fastapi import Depends
from backend.tenancy.context import TenantContext, get_tenant_context

@router.get("/billing/usage")
async def get_my_usage(ctx: TenantContext = Depends(get_tenant_context)):
    return await _usage_payload(ctx.org_id, ctx.user_id, ctx.user_role)
```

Every router touched from here on replaces its old `request.headers.get("X-User-ID", ...)` / `request.headers.get("X-User-Role", ...)` pair with a single `ctx: TenantContext = Depends(get_tenant_context)`, gaining `ctx.org_id` for free and a 403 on bad org headers it didn't have before.

<a id="step-213"></a>
## Step 21.3 - Namespacing cache keys by organization

This is the step that actually stops Acme's Alice and Globex's Alice from sharing a budget. The rule is mechanical: **every Redis key that contains a `user_id` also gets an `org_id` segment, placed before the user_id.** Three call sites change.

**Day 9's per-stream counters** (`backend/llm/governed_client.py`):

```python
# Before (Day 9)
await redis_incr(f"cost:user:{self.user_id}:{today}")
await redis_incr(f"cost:model:{self.model_id}:{today}")
active_key = f"stream:active:{self.user_id}"

# After (Day 21)
await redis_incr(f"cost:org:{self.org_id}:user:{self.user_id}:{today}")
await redis_incr(f"cost:model:{self.model_id}:{today}")  # model-level totals stay global by design
active_key = f"stream:active:org:{self.org_id}:user:{self.user_id}"
```

The model-level counter (`cost:model:{model_id}:{date}`) is *intentionally* left global - it answers "how much does `gpt-4o` cost us across all tenants today", a question that's useful precisely because it isn't tenant-scoped. Not every key needs an org segment; only the ones that represent a per-tenant resource (budget, concurrency, abuse signal) do.

**Day 20's budget tracker** (`backend/billing/budget_tracker.py`):

```python
# backend/billing/budget_tracker.py (updated)

from dataclasses import dataclass
from datetime import datetime, timezone

from backend.cache.redis_client import redis_incrbyfloat, redis_get_float

DAILY_TTL_SECONDS = 26 * 3600
MONTHLY_TTL_SECONDS = 32 * 24 * 3600


@dataclass
class BudgetSpend:
    daily_usd: float
    monthly_usd: float


def _daily_key(org_id: str, user_id: str, date: str) -> str:
    return f"budget:org:{org_id}:user:{user_id}:{date}"


def _monthly_key(org_id: str, user_id: str, month: str) -> str:
    return f"budget:org:{org_id}:user:{user_id}:{month}"


async def record_cost(org_id: str, user_id: str, cost_usd: float) -> None:
    now = datetime.now(timezone.utc)
    date_str = now.strftime("%Y-%m-%d")
    month_str = now.strftime("%Y-%m")

    await redis_incrbyfloat(_daily_key(org_id, user_id, date_str), cost_usd, ttl=DAILY_TTL_SECONDS)
    await redis_incrbyfloat(_monthly_key(org_id, user_id, month_str), cost_usd, ttl=MONTHLY_TTL_SECONDS)


async def get_spend(org_id: str, user_id: str) -> BudgetSpend:
    now = datetime.now(timezone.utc)
    date_str = now.strftime("%Y-%m-%d")
    month_str = now.strftime("%Y-%m")

    daily = await redis_get_float(_daily_key(org_id, user_id, date_str))
    monthly = await redis_get_float(_monthly_key(org_id, user_id, month_str))
    return BudgetSpend(daily_usd=daily, monthly_usd=monthly)
```

`_daily_key` and `_monthly_key` now take `org_id` as their first argument. Every caller of `record_cost()` and `get_spend()` - the billing router, `BudgetEnforcer`, `GovernedStreamManager` - gains an `org_id` argument too. The `redis_incrbyfloat` / `redis_get_float` helpers from Day 20 don't change at all; only the key strings passed to them do.

**Day 20's budget enforcer** (`backend/billing/budget_enforcer.py`) follows the same shape - `check()` gains `org_id`:

```python
# backend/billing/budget_enforcer.py (signature change only; logic from Day 20 unchanged)

class BudgetEnforcer:
    async def check(self, org_id: str, user_id: str, user_role: str) -> BudgetDecision:
        spend = await budget_tracker.get_spend(org_id, user_id)
        limits = limits_for_role(user_role)
        # ... monthly-then-daily comparison from Day 20, unchanged ...
```

**Day 18's abuse signal windows** (`backend/security/abuse_tracker.py`) get the same treatment - the sliding-window sorted-set keys built from `event_type` and `actor_id` now include `org_id`:

```python
# backend/security/abuse_tracker.py (key construction, updated)

def _signal_key(org_id: str, event_type: str, actor_id: str) -> str:
    return f"abuse:signal:org:{org_id}:{event_type}:{actor_id}"
```

This matters for the same collision reason as budgets: without it, three `SECRET_ACCESS_DENIED` events from Globex's Alice and two from Acme's Alice would sum to five against a shared `SUSPEND` threshold of five - suspending a user who, from their own organization's point of view, only tripped the detector twice.

<a id="step-214"></a>
## Step 21.4 - Org-scoped database columns

Two tables need an `org_id` column: `audit_log` (Day 17) and `stream_sessions` (Day 9). Both get the same treatment - a new `String(64)` column, indexed, `nullable=False`, with a server-side default of `"default"` so the migration backfills every existing row without breaking foreign data.

```python
# backend/database/models.py (additions)

class AuditLogEntry(Base):
    __tablename__ = "audit_log"

    id           = Column(Integer, primary_key=True, autoincrement=True)
    event_id     = Column(String(36), unique=True, nullable=False)
    event_type   = Column(String(64), nullable=False, index=True)
    org_id       = Column(String(64), nullable=False, default="default", index=True)  # new
    actor_id     = Column(String(128), nullable=False, index=True)
    actor_role   = Column(String(64), nullable=True)
    session_id   = Column(String(128), nullable=True, index=True)
    resource_id  = Column(String(256), nullable=True)
    outcome      = Column(String(32), nullable=False)
    reason       = Column(Text, nullable=True)
    payload      = Column(Text, nullable=False, default="{}")
    timestamp    = Column(DateTime, nullable=False, index=True)
    content_hash = Column(String(64), nullable=False)
    chain_hash   = Column(String(64), nullable=False, unique=True)

    __table_args__ = (
        Index("ix_audit_session_type", "session_id", "event_type"),
        Index("ix_audit_org_type", "org_id", "event_type"),  # new
    )


class StreamSession(Base):
    __tablename__ = "stream_sessions"

    stream_id    = Column(String(64),  primary_key=True)
    org_id       = Column(String(64),  nullable=False, default="default", index=True)  # new
    user_id      = Column(String(256), nullable=False, index=True)
    user_role    = Column(String(64),  nullable=False)
    model_id     = Column(String(256), nullable=False)
    session_id   = Column(String(256), nullable=True)
    prompt_hash  = Column(String(64),  nullable=False)
    outcome      = Column(String(32),  nullable=False)
    tokens_used  = Column(Integer,     nullable=False, default=0)
    cost_usd     = Column(Float,       nullable=False, default=0.0)
    retracted    = Column(Boolean,     nullable=False, default=False)
    duration_ms  = Column(Integer,     nullable=False, default=0)
    scan_id      = Column(String(64),  nullable=True)
    context_used = Column(Boolean,     nullable=False, default=False)
    started_at   = Column(DateTime(timezone=True), default=lambda: datetime.now(timezone.utc), index=True)
```

```bash
alembic revision --autogenerate -m "add_org_id_to_audit_log_and_stream_sessions"
alembic upgrade head
```

A note on the hash chain: Day 17's `chain_hash` links every audit record to the previous one, globally, in insertion order - and Day 21 deliberately does **not** split that into one chain per org. A single global chain is *stronger* tamper evidence (an attacker would have to rewrite history across every tenant's records to forge it undetected), and `org_id` is just another column covered by `content_hash` like `actor_id` or `resource_id` already were. What changes is the *query layer*: `AuditVerifier.verify_chain()` still walks the one global chain, but a new `get_audit_events(org_id, ...)` helper (Step 21.5) filters by `org_id` for anything an org-scoped caller is allowed to read.

<a id="step-215"></a>
## Step 21.5 - Threading org_id through the audit system

Day 17's `AuditEvent` and `AuditRecord` models, and every method on `AuditLogger`, gain `org_id`. This is mechanical but touches a lot of call sites - the pattern is the same one Day 18 used when it added `_emit()`: change the shared model and writer once, then update each `AuditLogger` method's signature.

```python
# backend/audit/audit_schema.py (additions)

class AuditEventType(str, Enum):
    # ... all existing members from Day 17 unchanged ...

    # Tenancy (new)
    TENANT_ACCESS_DENIED = "tenancy.access_denied"


class AuditEvent(BaseModel):
    event_type:  AuditEventType
    org_id:      str = "default"          # new - which tenant this event belongs to
    actor_id:    str
    actor_role:  Optional[str] = None
    session_id:  Optional[str] = None
    resource_id: Optional[str] = None
    outcome:     str = "unknown"
    reason:      Optional[str] = None
    payload:     dict[str, Any] = Field(default_factory=dict)


class AuditRecord(BaseModel):
    id:           int
    event_id:     str
    event_type:   str
    org_id:       str                      # new
    actor_id:     str
    actor_role:   Optional[str]
    session_id:   Optional[str]
    resource_id:  Optional[str]
    outcome:      str
    reason:       Optional[str]
    payload:      dict[str, Any]
    timestamp:    datetime
    content_hash: str
    chain_hash:   str
```

`AuditWriter.append()` (Day 17) needs one change: `org_id` is included in the payload that gets hashed into `content_hash`, and written to the new column:

```python
# backend/audit/audit_writer.py (excerpt, updated)

def append(self, event: AuditEvent) -> AuditRecord:
    # ... existing id/timestamp/previous-hash lookup unchanged ...

    content = {
        "event_type": event.event_type.value,
        "org_id": event.org_id,            # new - part of the hashed content
        "actor_id": event.actor_id,
        "actor_role": event.actor_role,
        "session_id": event.session_id,
        "resource_id": event.resource_id,
        "outcome": event.outcome,
        "reason": event.reason,
        "payload": event.payload,
        "timestamp": timestamp.isoformat(),
    }
    content_hash = _hash_content(content)
    chain_hash = _hash_content({"previous": previous_chain_hash, "content_hash": content_hash})

    row = AuditLogEntry(
        event_id=event_id,
        event_type=event.event_type.value,
        org_id=event.org_id,               # new
        actor_id=event.actor_id,
        # ... remaining fields unchanged from Day 17 ...
        content_hash=content_hash,
        chain_hash=chain_hash,
    )
    # ... persist and return AuditRecord, now including org_id ...
```

`AuditLogger`'s per-domain methods (Day 17, extended in Days 18-20) each take `org_id` as their second parameter, right after `event_type` is implied:

```python
# backend/audit/audit_logger.py (signature updates - bodies call _emit() exactly as in Day 18)

class AuditLogger:
    def session_created(self, org_id: str, session_id: str, user_id: str, role: str, goal: str):
        self._emit(AuditEvent(
            event_type=AuditEventType.SESSION_CREATED,
            org_id=org_id, actor_id=user_id, actor_role=role,
            session_id=session_id, outcome="created",
            payload={"goal": goal},
        ))

    def session_closed(self, org_id: str, session_id: str, user_id: str, role: str, outcome: str):
        self._emit(AuditEvent(
            event_type=AuditEventType.SESSION_CLOSED,
            org_id=org_id, actor_id=user_id, actor_role=role,
            session_id=session_id, outcome=outcome,
        ))

    # secret_access_granted, secret_access_denied, secret_leak_blocked (Day 19),
    # budget_warning, budget_exceeded (Day 20), and every other method gain the
    # same leading org_id parameter. The bodies are unchanged from prior days
    # other than passing org_id through to AuditEvent.

    def tenant_access_denied(self, org_id: str, user_id: str, role: str, target_user_id: str, resource: str):
        self._emit(AuditEvent(
            event_type=AuditEventType.TENANT_ACCESS_DENIED,
            org_id=org_id, actor_id=user_id, actor_role=role,
            resource_id=target_user_id, outcome="blocked",
            reason=f"cross-tenant access to {resource} denied",
            payload={"requested_resource": resource, "target_user_id": target_user_id},
        ))
```

Day 18's `_emit()` wrapper - which writes the record and then feeds Day 18's abuse detector - is unchanged in shape; it now just receives an `AuditEvent` that carries `org_id`, and Step 21.3's namespaced abuse-signal keys mean the detector counts per-org automatically.

Finally, the read side. `AuditVerifier.verify_chain()` (Day 17) still walks every row, but a new helper gives org-scoped callers a filtered view:

```python
# backend/audit/audit_logger.py (new helper)

def get_audit_events(org_id: str, event_type: Optional[str] = None, limit: int = 100) -> list[AuditRecord]:
    db = SessionLocal()
    try:
        query = db.query(AuditLogEntry).filter(AuditLogEntry.org_id == org_id)
        if event_type:
            query = query.filter(AuditLogEntry.event_type == event_type)
        rows = query.order_by(AuditLogEntry.id.desc()).limit(limit).all()
        return [AuditRecordOut.from_orm(r) for r in rows]
    finally:
        db.close()
```

<a id="step-216"></a>
## Step 21.6 - Cross-tenant access checks

The user directory and the boundary check come together here. The directory is intentionally tiny - one Redis `SET`, written on every request that carries a `TenantContext`, read whenever a request asks about *another* user_id:

```python
# backend/tenancy/user_directory.py

from backend.cache.redis_client import get_redis

DIRECTORY_TTL_SECONDS = 90 * 24 * 3600  # 90 days - refreshed on every request


async def record_user_org(org_id: str, user_id: str) -> None:
    """
    Idempotently record that user_id belongs to org_id. Called once per
    request via get_tenant_context's caller (see billing router below).
    Last writer wins, which is fine: a user_id legitimately changing
    orgs is an admin action, not an attack, and this directory is a
    convenience lookup, not the source of truth for authorization
    decisions about the *caller* (the X-Org-ID header is).
    """
    r = await get_redis()
    await r.set(f"tenancy:user_org:{user_id}", org_id, ex=DIRECTORY_TTL_SECONDS)


async def lookup_user_org(user_id: str) -> str | None:
    r = await get_redis()
    value = await r.get(f"tenancy:user_org:{user_id}")
    return value.decode() if value else None
```

Day 20's billing router gets two changes: it now depends on `TenantContext`, records the caller's `(user_id, org_id)` pair, and - for the admin "look up someone else's usage" endpoint - checks the target user_id's recorded org against the caller's org before answering.

```python
# backend/routers/billing.py (updated)

from fastapi import APIRouter, Depends, HTTPException

from backend.tenancy.context import TenantContext, get_tenant_context
from backend.tenancy.user_directory import record_user_org, lookup_user_org
from backend.audit.audit_logger import AuditLogger
from backend.billing import budget_tracker
from backend.billing.budget_schema import limits_for_role

router = APIRouter()
_audit = AuditLogger()


async def _usage_payload(org_id: str, user_id: str, user_role: str) -> dict:
    spend = await budget_tracker.get_spend(org_id, user_id)
    limits = limits_for_role(user_role)
    return {
        "org_id": org_id,
        "user_id": user_id,
        "daily": {
            "spend_usd": round(spend.daily_usd, 4),
            "limit_usd": limits.daily_usd,
            "remaining_usd": round(max(limits.daily_usd - spend.daily_usd, 0.0), 4),
        },
        "monthly": {
            "spend_usd": round(spend.monthly_usd, 4),
            "limit_usd": limits.monthly_usd,
            "remaining_usd": round(max(limits.monthly_usd - spend.monthly_usd, 0.0), 4),
        },
    }


@router.get("/billing/usage")
async def get_my_usage(ctx: TenantContext = Depends(get_tenant_context)):
    await record_user_org(ctx.org_id, ctx.user_id)
    return await _usage_payload(ctx.org_id, ctx.user_id, ctx.user_role)


@router.get("/billing/usage/{user_id}")
async def get_user_usage(user_id: str, ctx: TenantContext = Depends(get_tenant_context)):
    await record_user_org(ctx.org_id, ctx.user_id)

    if ctx.user_role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required")

    target_org = await lookup_user_org(user_id)
    if target_org is not None and target_org != ctx.org_id:
        _audit.tenant_access_denied(ctx.org_id, ctx.user_id, ctx.user_role, user_id, "billing/usage")
        raise HTTPException(status_code=403, detail="Cannot view usage for a user outside your organization")

    # target_org is None only for a user_id that has never made a request
    # (no directory entry yet) - treated as "not in any org we know about
    # yet", which is allowed: it can't be a cross-tenant leak of real data
    # because no real data exists for that user_id under any org.
    return await _usage_payload(ctx.org_id, user_id, ctx.user_role)
```

The same dependency and directory check apply to Day 19's admin endpoint:

```python
# backend/routers/security.py (updated)

@router.get("/security/secret-scopes")
async def get_secret_scopes(ctx: TenantContext = Depends(get_tenant_context)):
    await record_user_org(ctx.org_id, ctx.user_id)

    if ctx.user_role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required")

    # AGENT_SECRET_SCOPES (Day 19) is still global config, not per-org data,
    # so there's nothing to filter by org here - admins from any org can see
    # which *categories* of secrets each agent type may access. What's
    # org-scoped is the *values* (Step 21.7), which this endpoint never
    # returns anyway.
    return {agent_type: sorted(s.value for s in secrets) for agent_type, secrets in AGENT_SECRET_SCOPES.items()}
```

This is the fix for the exact gap described in the problem statement: before Day 21, `GET /billing/usage/{user_id}` with `X-User-Role: admin` returned any user's spend. After Day 21, it returns 403 plus a `TENANT_ACCESS_DENIED` audit event the moment the target user_id is known to belong to a different `org_id` than the caller's `X-Org-ID`.

<a id="step-217"></a>
## Step 21.7 - Org-aware secret resolution

Day 19's `vault.get_secret(name)` reads one environment variable per `SecretName`. Day 21 extends it to check an org-specific variable first, falling back to the global one - so Acme and Globex can each bring their own key for the same logical secret, while a deployment with only one tenant keeps working unchanged.

```python
# backend/secrets/vault.py (updated)

import os
from backend.secrets.secret_schema import SecretName, SECRET_REGISTRY


def _env_var_name(secret: SecretName, org_id: str | None = None) -> str:
    base = SECRET_REGISTRY[secret].env_var  # e.g. "OPENAI_API_KEY"
    if org_id and org_id != "default":
        return f"{org_id.upper()}_{base}"   # e.g. "ACME_OPENAI_API_KEY"
    return base


def get_secret(secret: SecretName, org_id: str | None = None) -> str | None:
    """
    Resolve a secret value, preferring an org-specific environment variable
    (<ORG>_<BASE_VAR>) and falling back to the global <BASE_VAR>. A tenant
    that hasn't configured its own credential transparently shares the
    operator's default - same behavior as Day 19 for single-tenant setups.
    """
    if org_id:
        org_specific = os.environ.get(_env_var_name(secret, org_id))
        if org_specific:
            return org_specific
    return os.environ.get(_env_var_name(secret))


def is_configured(secret: SecretName, org_id: str | None = None) -> bool:
    return get_secret(secret, org_id) is not None


def all_known_values(org_id: str | None = None) -> set[str]:
    """Used by secret_scanner.py (Day 19) to build its exact-match list."""
    values = set()
    for name in SecretName:
        value = get_secret(name, org_id)
        if value:
            values.add(value)
    return values
```

`CredentialBroker.get_secret()` (Day 19) gains `org_id` and passes it straight through:

```python
# backend/secrets/credential_broker.py (updated)

class CredentialBroker:
    def get_secret(self, agent_type: str, secret: SecretName, org_id: str = "default") -> str:
        allowed = AGENT_SECRET_SCOPES.get(agent_type, set())
        if secret not in allowed:
            _audit.secret_access_denied(org_id, agent_type, secret.value, reason="not in agent's scope")
            raise SecretAccessDenied(f"{agent_type} is not permitted to access {secret.value}")

        value = vault.get_secret(secret, org_id)
        if value is None:
            _audit.secret_access_denied(org_id, agent_type, secret.value, reason="not configured")
            raise SecretAccessDenied(f"{secret.value} is not configured for org '{org_id}'")

        _audit.secret_access_granted(org_id, agent_type, secret.value)
        return value
```

And `GovernedAgent.get_secret()` (Day 19) - already a thin pass-through - adds the one field it was missing:

```python
# backend/agents/base_agent.py (updated)

class GovernedAgent:
    def get_secret(self, secret: SecretName) -> str:
        return _credential_broker.get_secret(self.agent_type.value, secret, self.org_id)
```

`self.org_id` is new on `GovernedAgent` - set by the orchestrator when it spawns the agent (Step 21.8). `secret_scanner.py`'s `redact_secrets()` (Day 19) calls `all_known_values(org_id)` instead of `all_known_values()`, so outbound-message scanning redacts the calling org's secrets - including any org-specific overrides - without needing to know about other tenants' values at all.

<a id="step-218"></a>
## Step 21.8 - Wiring org_id through streaming and the orchestrator

`GovernedStreamManager` (Day 9) takes `org_id` as a constructor argument, defaulting to `"default"` so existing call sites that haven't been updated keep working:

```python
# backend/llm/governed_client.py (updated)

class GovernedStreamManager:
    def __init__(
        self,
        user_id: str,
        user_role: str,
        model_id: str,
        prompt: str,
        org_id: str = "default",          # new
        session_id: str | None = None,
        system_prompt: str | None = None,
        context: dict | None = None,
        db_session=None,
    ):
        self.user_id = user_id
        self.user_role = user_role
        self.org_id = org_id              # new
        self.model_id = model_id
        self.prompt = prompt
        self.session_id = session_id
        self.system_prompt = system_prompt
        self.context = context
        self.db_session = db_session

    async def stream(self):
        # Step 0 (Day 20): budget check, now org-scoped
        decision = await self._budget_enforcer.check(self.org_id, self.user_id, self.user_role)
        if decision.action == BudgetAction.BLOCK:
            yield StreamBudgetExceededEvent(
                org_id=self.org_id, period=decision.period,
                spend_usd=decision.spend, limit_usd=decision.limits,
            ).to_sse()
            _audit.budget_exceeded(self.org_id, self.user_id, self.user_role, decision.period, decision.spend)
            return

        # ... Day 9's concurrent-slot check, pre-stream governance, etc. ...
        # _acquire_stream_slot() now uses the org-namespaced key from Step 21.3:
        #   f"stream:active:org:{self.org_id}:user:{self.user_id}"

        # ... on completion/retraction/disconnect ...
        await budget_tracker.record_cost(self.org_id, self.user_id, actual_cost)
        await redis_incr(f"cost:org:{self.org_id}:user:{self.user_id}:{today}")
```

`StreamBudgetExceededEvent` (Day 20) gains `org_id` as a field, like every other stream event dataclass. `StreamSession` rows (Step 21.4) are written with `org_id=self.org_id`.

The orchestrator (Day 14) threads `org_id` from the inbound request down to every agent it spawns and every audit call it makes:

```python
# backend/orchestrator/orchestrator.py (updated)

class AgentOrchestrator:
    async def orchestrate(self, goal: str, org_id: str, user_id: str, user_role: str, context: dict | None = None):
        session_id = str(uuid.uuid4())
        _audit.session_created(org_id, session_id, user_id, user_role, goal)

        try:
            # ... policy check (Day 5), as before ...
            for entry in self._select_agents(goal):
                agent = entry.agent_class(
                    agent_type=entry.agent_type,
                    org_id=org_id,          # new - flows into GovernedAgent.get_secret()
                    session_id=session_id,
                    user_id=user_id,
                )
                _audit.agent_spawn_allowed(org_id, session_id, user_id, user_role, entry.agent_type.value)
                # ... run agent, message bus checks (Days 16, 19) unchanged ...
        finally:
            _audit.session_closed(org_id, session_id, user_id, user_role, outcome="completed")
```

And the FastAPI entry point that kicks off orchestration replaces its two-header read with `TenantContext`:

```python
# backend/routers/agents.py (updated)

@router.post("/agents/run")
async def run_agents(body: RunRequest, ctx: TenantContext = Depends(get_tenant_context)):
    await record_user_org(ctx.org_id, ctx.user_id)
    result = await orchestrator.orchestrate(
        goal=body.goal, org_id=ctx.org_id, user_id=ctx.user_id, user_role=ctx.user_role,
    )
    return result
```

<a id="step-219"></a>
## Step 21.9 - Abuse detection for cross-tenant probing

A caller that repeatedly guesses other organizations' user_ids against `/billing/usage/{user_id}` - fishing for one that resolves - is a behavioral signal Day 18's detector should know about. One new entry in `ABUSE_THRESHOLDS`:

```python
# backend/security/abuse_thresholds.py (addition)

ABUSE_THRESHOLDS = [
    # ... all entries from Days 18 and 20 unchanged ...

    AbuseThreshold(
        event_type=AuditEventType.TENANT_ACCESS_DENIED,
        window_seconds=3600,
        count=3,
        action=AbuseAction.SUSPEND,
        description="3 cross-tenant access attempts in 1 hour - likely org_id enumeration",
    ),
]
```

Three is deliberately low compared to Day 18's other thresholds (`SECRET_ACCESS_DENIED` warns at 3 and suspends at 5). A legitimate admin might fat-finger one wrong user_id; three in an hour, every one of them belonging to a *different* organization than the caller's, isn't a typo pattern - it's enumeration. Because Step 21.3 namespaced the abuse signal keys by `org_id`, this threshold is also evaluated per-org: an attacker can't dilute their count by also holding a legitimate account in a different tenant.

<a id="step-2110"></a>
## Step 21.10 - Tests

```python
# tests/tenancy/test_tenant_context.py

import pytest
from fastapi import HTTPException
from fastapi.testclient import TestClient

from backend.main import app
from backend.tenancy.context import get_tenant_context
from backend.tenancy.tenant_schema import TENANT_REGISTRY

client = TestClient(app)


def test_known_org_resolves():
    ctx = get_tenant_context.__wrapped__ if hasattr(get_tenant_context, "__wrapped__") else get_tenant_context
    # Direct unit test via a fake Request-like object is overkill here;
    # exercise it through a real endpoint instead.
    resp = client.get("/billing/usage", headers={
        "X-Org-ID": "acme", "X-User-ID": "alice", "X-User-Role": "employee",
    })
    assert resp.status_code == 200
    assert resp.json()["org_id"] == "acme"


def test_unknown_org_rejected():
    resp = client.get("/billing/usage", headers={
        "X-Org-ID": "initech", "X-User-ID": "alice", "X-User-Role": "employee",
    })
    assert resp.status_code == 403


def test_missing_org_header_defaults_to_default_tenant():
    resp = client.get("/billing/usage", headers={
        "X-User-ID": "alice", "X-User-Role": "employee",
    })
    assert resp.status_code == 200
    assert resp.json()["org_id"] == "default"
```

```python
# tests/tenancy/test_cache_namespacing.py

import pytest
from backend.billing import budget_tracker


class TestCacheNamespacing:
    async def test_same_user_id_different_orgs_dont_collide(self):
        await budget_tracker.record_cost("acme", "alice", 10.00)
        await budget_tracker.record_cost("globex", "alice", 3.00)

        acme_spend = await budget_tracker.get_spend("acme", "alice")
        globex_spend = await budget_tracker.get_spend("globex", "alice")

        assert acme_spend.daily_usd == pytest.approx(10.00)
        assert globex_spend.daily_usd == pytest.approx(3.00)

    async def test_budget_enforcer_is_org_scoped(self):
        from backend.billing.budget_enforcer import BudgetEnforcer, BudgetAction

        enforcer = BudgetEnforcer()
        # Acme's "alice" (employee) blows her monthly cap...
        await budget_tracker.record_cost("acme", "alice2", 150.00)
        acme_decision = await enforcer.check("acme", "alice2", "employee")
        assert acme_decision.action == BudgetAction.BLOCK

        # ...but Globex's identically-named user is unaffected.
        globex_decision = await enforcer.check("globex", "alice2", "employee")
        assert globex_decision.action == BudgetAction.OK
```

```python
# tests/tenancy/test_cross_tenant_billing.py

from fastapi.testclient import TestClient
from backend.main import app
from backend.audit.audit_logger import AuditLogger, AuditEventType
from backend.database.session import SessionLocal
from backend.database.models import AuditLogEntry

client = TestClient(app)


class TestCrossTenantBilling:
    def test_admin_cannot_view_other_org_user_usage(self):
        # Establish that "bob" belongs to globex.
        client.get("/billing/usage", headers={
            "X-Org-ID": "globex", "X-User-ID": "bob", "X-User-Role": "employee",
        })

        # An acme admin tries to look bob up.
        resp = client.get("/billing/usage/bob", headers={
            "X-Org-ID": "acme", "X-User-ID": "admin1", "X-User-Role": "admin",
        })
        assert resp.status_code == 403

        db = SessionLocal()
        try:
            record = (
                db.query(AuditLogEntry)
                .filter(AuditLogEntry.event_type == AuditEventType.TENANT_ACCESS_DENIED.value)
                .filter(AuditLogEntry.org_id == "acme")
                .first()
            )
        finally:
            db.close()

        assert record is not None
        assert record.resource_id == "bob"

    def test_admin_can_view_same_org_user_usage(self):
        client.get("/billing/usage", headers={
            "X-Org-ID": "acme", "X-User-ID": "carol", "X-User-Role": "employee",
        })

        resp = client.get("/billing/usage/carol", headers={
            "X-Org-ID": "acme", "X-User-ID": "admin1", "X-User-Role": "admin",
        })
        assert resp.status_code == 200
        assert resp.json()["org_id"] == "acme"
```

```python
# tests/secrets/test_org_scoped_vault.py

import os
from backend.secrets import vault
from backend.secrets.secret_schema import SecretName


class TestOrgScopedVault:
    def test_org_specific_secret_overrides_global(self, monkeypatch):
        monkeypatch.setenv("OPENAI_API_KEY", "sk-global-default")
        monkeypatch.setenv("ACME_OPENAI_API_KEY", "sk-acme-only")

        assert vault.get_secret(SecretName.OPENAI_API_KEY, org_id="acme") == "sk-acme-only"
        assert vault.get_secret(SecretName.OPENAI_API_KEY, org_id="globex") == "sk-global-default"
        assert vault.get_secret(SecretName.OPENAI_API_KEY) == "sk-global-default"
```

```python
# tests/audit/test_org_scoped_audit.py

from backend.audit.audit_logger import AuditLogger, get_audit_events


class TestOrgScopedAudit:
    def test_get_audit_events_filters_by_org(self):
        audit = AuditLogger()
        audit.session_created("acme", "sess-acme-1", "alice", "employee", "test goal")
        audit.session_created("globex", "sess-globex-1", "alice", "employee", "test goal")

        acme_events = get_audit_events("acme", event_type="session.created", limit=10)
        assert all(e.org_id == "acme" for e in acme_events)
        assert any(e.session_id == "sess-acme-1" for e in acme_events)
        assert not any(e.session_id == "sess-globex-1" for e in acme_events)
```

Run them:

```bash
pytest tests/tenancy/ tests/secrets/test_org_scoped_vault.py tests/audit/test_org_scoped_audit.py -v
```

`test_admin_cannot_view_other_org_user_usage` is the test that proves the fix for the problem statement's central claim - before Step 21.6's check existed, this request returned `200` with Globex's real spend numbers.

<a id="what-you-built"></a>
## What you built today

A `TENANT_REGISTRY` of known organizations, each with a name and reserved tier, and `TenantContext` - a third identity dimension alongside Day 4's `user_id`/`user_role`, read from a new `X-Org-ID` header and rejected outright if it names an org that doesn't exist. Every Redis key built across Days 9, 18, and 20 - cost counters, budget totals, abuse signal windows, active-stream locks - now carries an `org:{org_id}:` segment, so two tenants with identically-named users no longer share a budget, a concurrency slot, or an abuse counter. Both `audit_log` (Day 17) and `stream_sessions` (Day 9) gained an indexed `org_id` column, threaded through `AuditEvent`, `AuditRecord`, and every `AuditLogger` method, with the global hash chain intact and a new `get_audit_events(org_id, ...)` for scoped reads. A tiny Redis-backed `user_directory` records which org each user_id belongs to, and Day 20's `GET /billing/usage/{user_id}` - previously answerable for *any* user_id by *any* admin - now returns `403` plus a new `TENANT_ACCESS_DENIED` audit event the moment the target belongs to a different org than the caller. `vault.get_secret()` and `CredentialBroker` (Day 19) now check an org-specific environment variable before falling back to the global one, so each tenant can bring its own third-party credentials. `GovernedStreamManager` and `AgentOrchestrator` thread `org_id` everywhere `user_id` already flowed. And a new `ABUSE_THRESHOLDS` entry suspends a caller after three cross-tenant access denials in an hour - org enumeration now looks like the abuse signal it is.

<a id="checklist"></a>
## Day 21 checklist

- [ ] `backend/tenancy/tenant_schema.py` defines `Tenant`, `TenantTier`, and `TENANT_REGISTRY` including a `"default"` migration-aid tenant
- [ ] `backend/tenancy/context.py`'s `get_tenant_context()` reads `X-Org-ID` (default `"default"`), `X-User-ID`, `X-User-Role`, and raises `403` for an unknown org
- [ ] `backend/tenancy/user_directory.py` records and looks up which `org_id` a `user_id` belongs to via Redis
- [ ] Every Day 9/18/20 Redis key that includes a `user_id` (`cost:user:...`, `budget:user:...`, `abuse:signal:...`, `stream:active:...`) now also includes `org:{org_id}:`
- [ ] `audit_log` and `stream_sessions` have indexed, non-null `org_id` columns, backfilled to `"default"`
- [ ] `AuditEvent`, `AuditRecord`, and every `AuditLogger` method (including Day 19's and Day 20's) take `org_id`; `content_hash` covers it; the global hash chain is unchanged
- [ ] `AuditEventType` includes `TENANT_ACCESS_DENIED`, with `AuditLogger.tenant_access_denied()`
- [ ] `GET /billing/usage/{user_id}` returns `403` and logs `TENANT_ACCESS_DENIED` when the target user_id belongs to a different org than the caller
- [ ] `vault.get_secret()` checks `<ORG>_<SECRET>` before falling back to `<SECRET>`; `CredentialBroker.get_secret()` and `GovernedAgent.get_secret()` pass `org_id` through
- [ ] `GovernedStreamManager` and `AgentOrchestrator` accept and propagate `org_id` to every cache key, audit call, and spawned agent
- [ ] `ABUSE_THRESHOLDS` includes a `TENANT_ACCESS_DENIED` entry (SUSPEND at 3 per hour), evaluated per-org
- [ ] All tests in `tests/tenancy/`, plus the updated tests in `tests/billing/`, `tests/audit/`, and `tests/secrets/`, pass

---

[Back to repo](https://github.com/GuntruTirupathamma/AI-Governance) - Previous: [DAY20.md](./DAY20.md) - Next: DAY22.md - Real-Time Alerting and Notification Channels
