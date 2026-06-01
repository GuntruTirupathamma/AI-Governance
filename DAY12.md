# 🏢 DAY 12 — Multi-Tenant Isolation: Namespace All Data by org_id

> **Series:** AI Governance Engineering — Zero to Production  **Author:** GuntruTirupathamma  **Day:** 12 of 35  **Previous:** [DAY11.md](https://github.com/GuntruTirupathamma/AI-Governance/blob/main/DAY11.md) — Dashboard: Real-Time Governance Metrics with WebSockets  **Next:** DAY13.md — Alerting: Fire PagerDuty/Slack on Governance Threshold Breaches

---

## 📌 Table of Contents

1. [Why Multi-Tenancy Is a Governance Requirement](#why-multitenancy)
2. [What You'll Build Today](#what-youll-build)
3. [The Multi-Tenant Architecture](#multitenancy-architecture)
4. [Step 12.1 — Organization Model and Tenant Registry](#step-121)
5. [Step 12.2 — Tenant Resolution Middleware](#step-122)
6. [Step 12.3 — Database Migration: org_id on All Tables](#step-123)
7. [Step 12.4 — Tenant-Scoped Query Helpers](#step-124)
8. [Step 12.5 — Tenant-Scoped Policy Rules](#step-125)
9. [Step 12.6 — Tenant-Scoped RAG Isolation](#step-126)
10. [Step 12.7 — Tenant-Scoped Metrics and Cost Tracking](#step-127)
11. [Step 12.8 — Tenant Management Endpoints](#step-128)
12. [Step 12.9 — Tenant-Scoped Redis Keys](#step-129)
13. [Step 12.10 — Testing Multi-Tenant Isolation](#step-1210)
14. [What You Built Today](#what-you-built)
15. [Day 12 Checklist](#checklist)

---

## ⚡ Why Multi-Tenancy Is a Governance Requirement

Right now the governance system is single-tenant. Every user, every policy check, every audit record, every RAG document, every cost counter — all of it shares the same namespace. This works for one organization. The moment you serve two organizations, it breaks.

### The Four Scenarios That Make Multi-Tenancy Non-Negotiable

```
Scenario 1 — The Data Leakage Incident
  Acme Corp and Beta Inc both use your governance platform.
  Acme's HR team indexes their salary data into the RAG system.
  Beta Inc's analyst asks: "What are the salary bands?"
  
  Without org_id isolation: ChromaDB returns Acme's salary chunks to Beta.
  The retrieval filter has no concept of "your data vs their data."
  Acme has a data breach. Beta Inc has data they shouldn't see.
  
  With org_id isolation: every ChromaDB chunk carries org_id metadata.
  Beta's query filter includes org_id="beta-inc".
  Acme's salary chunks are never returned. Breach prevented.

Scenario 2 — The Audit Confusion
  Acme Corp's compliance officer opens the audit log dashboard.
  She sees 4,000 policy check records.
  1,200 of them are from Beta Inc's users.
  
  Without org_id: she cannot tell which records belong to her org.
  Her compliance report is wrong. She files incorrect regulatory data.
  
  With org_id: GET /audit/logs automatically scopes to her org.
  She sees exactly 2,800 records — all Acme's. Compliance report is correct.

Scenario 3 — The Policy Collision
  Acme Corp adds a custom policy rule: "Block all prompts mentioning competitors."
  The rule fires on "How does our product compare to BetaCorp?"
  
  Without org_id on policy rules: this rule now applies to ALL tenants.
  Beta Inc's users can no longer mention their own company name in prompts.
  Beta Inc's IT manager calls to report the bug.
  
  With org_id: Acme's custom rule is scoped to org_id="acme-corp".
  Beta Inc's policy engine never loads Acme's rules. No collision.

Scenario 4 — The Bill Dispute
  End of month. Acme Corp gets billed $8,400 for LLM usage.
  Acme's CFO asks: "Can you show us our usage breakdown?"
  
  Without org_id on cost records: total cost = $12,200 across all tenants.
  You cannot separate Acme's $8,400 from Beta's $3,800.
  You cannot produce an itemized bill. You lose the contract.
  
  With org_id on Redis cost keys: cost:org:acme-corp:gpt-4o:2026-05-31 = 8400
  Itemized invoice generated in seconds.
  Acme's CFO is satisfied. Contract renewed.

```

**Multi-tenancy is not a feature. It is the boundary that makes governance data trustworthy.**

---

## 🎯 What You'll Build Today

```
BEFORE (Day 11):
  All data in one namespace — no concept of "which organization"
  Policy rules shared across all users (no per-tenant customization)
  RAG documents accessible to all users regardless of org
  Audit logs mixed across all organizations
  Cost counters global — cannot produce per-org invoices
  Metrics dashboard shows all orgs combined
  Rate limits per user but not per org (orgs can't have org-level quotas)
  No tenant onboarding or management endpoints

AFTER (Day 12):
  ✅ Organization model (tenant registry with plan, status, settings)
  ✅ TenantResolutionMiddleware — extracts org_id from X-Org-ID header
  ✅ Database migration — org_id column on 8 tables (NOT NULL with default)
  ✅ TenantScopedDB helper — auto-applies org_id to all queries
  ✅ Tenant-scoped policy rules — each org has its own rule set
  ✅ Tenant-scoped RAG — documents isolated by org_id in ChromaDB metadata
  ✅ Tenant-scoped audit logs — every record carries org_id
  ✅ Tenant-scoped cost keys — Redis keys namespaced by org_id
  ✅ Tenant-scoped metrics — WebSocket snapshot filtered by org
  ✅ Tenant-scoped rate limits — org-level quotas on top of user limits
  ✅ POST /tenants/          — register a new organization
  ✅ GET  /tenants/{org_id}  — tenant profile and settings
  ✅ PATCH /tenants/{org_id} — update plan, quota, settings
  ✅ GET  /tenants/{org_id}/usage — current month usage and cost
  ✅ 12+ tests verifying tenant isolation at every layer

```

---

## 🏛️ The Multi-Tenant Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     MULTI-TENANT REQUEST FLOW                            │
│                                                                          │
│  Incoming Request                                                        │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │              TenantResolutionMiddleware                         │    │
│  │                                                                 │    │
│  │  1. Extract X-Org-ID header                                     │    │
│  │  2. Validate org exists in Organization table                   │    │
│  │  3. Check org is active (not suspended/deleted)                 │    │
│  │  4. Attach org_id to request.state.org_id                       │    │
│  │  5. Check org-level rate limit (monthly API quota)              │    │
│  │                                                                 │    │
│  │  Missing/invalid X-Org-ID → 400 Bad Request                     │    │
│  │  Org suspended → 403 Forbidden                                  │    │
│  │  Org over quota → 429 Too Many Requests                         │    │
│  └──────────────────────────┬──────────────────────────────────────┘    │
│                             │ org_id resolved                            │
│                             ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │              Route Handler                                      │    │
│  │                                                                 │    │
│  │  org_id = request.state.org_id                                  │    │
│  │                                                                 │    │
│  │  Every DB query:   .where(Model.org_id == org_id)               │    │
│  │  Every Redis key:  f"org:{org_id}:..."                          │    │
│  │  Every ChromaDB:   filter={"org_id": org_id}                    │    │
│  │  Every audit log:  AuditEvent(org_id=org_id, ...)               │    │
│  │  Every cost key:   f"cost:org:{org_id}:model:..."               │    │
│  └──────────────────────────┬──────────────────────────────────────┘    │
│                             │                                            │
│                             ▼                                            │
│                    Tenant-scoped response                                │
│                    (only that org's data)                                │
│                                                                          │
│  Data Isolation by Layer:                                                │
│                                                                          │
│  ┌──────────────────┬────────────────────────────────────────────────┐  │
│  │ Layer            │ Isolation Mechanism                            │  │
│  ├──────────────────┼────────────────────────────────────────────────┤  │
│  │ PostgreSQL       │ WHERE org_id = ? on every query                │  │
│  │ Redis (cache)    │ cache:org:{org_id}:{hash} key prefix           │  │
│  │ Redis (rate)     │ rate:org:{org_id}:... key prefix               │  │
│  │ Redis (cost)     │ cost:org:{org_id}:model:... key prefix         │  │
│  │ ChromaDB         │ metadata filter: {"org_id": org_id}            │  │
│  │ Metrics          │ Snapshot scoped to org_id at collection time   │  │
│  │ Audit log        │ org_id column on AuditEvent, PolicyCheck, etc. │  │
│  └──────────────────┴────────────────────────────────────────────────┘  │
│                                                                          │
│  Organization Plans:                                                     │
│    free:       100 LLM calls/month, 1GB RAG storage, 1 admin user       │
│    starter:  1,000 LLM calls/month, 5GB RAG storage, 5 users           │
│    pro:     10,000 LLM calls/month, 20GB RAG storage, 20 users         │
│    enterprise: unlimited calls, unlimited storage, unlimited users      │
└──────────────────────────────────────────────────────────────────────────┘

org_id Format:
  Slug format: lowercase letters, numbers, hyphens only.
  Examples: "acme-corp", "beta-inc", "medical-ai-2026"
  Max 64 characters.
  Immutable after creation.

Request Headers (required for all API calls):
  X-Org-ID:   acme-corp       ← which organization
  X-User-ID:  alice           ← which user within that org
  X-User-Role: analyst        ← role within that org

```

---

## ⚙️ Step 12.1 — Organization Model and Tenant Registry

```python
# backend/database/models.py  (add Organization model)
from sqlalchemy import (
    Column, String, Boolean, Integer, DateTime,
    JSON, Enum as SAEnum, Text
)
from datetime import datetime, timezone
import enum

from backend.database.db import Base


class OrgPlan(str, enum.Enum):
    free       = "free"
    starter    = "starter"
    pro        = "pro"
    enterprise = "enterprise"


class OrgStatus(str, enum.Enum):
    active    = "active"
    suspended = "suspended"
    deleted   = "deleted"
    trial     = "trial"


class Organization(Base):
    """
    Tenant registry. One row per organization using the governance platform.
    
    This table is the source of truth for:
      - Is this org active?
      - What plan are they on?
      - What are their quotas?
      - Who are their admins?
    
    org_id is a human-readable slug (e.g. "acme-corp"), not a UUID.
    This makes debugging logs far easier: org_id="acme-corp" in a log line
    is immediately meaningful. UUIDs are not.
    """
    __tablename__ = "organizations"

    org_id          = Column(String(64),  primary_key=True)   # slug: "acme-corp"
    name            = Column(String(256), nullable=False)     # "Acme Corporation"
    plan            = Column(SAEnum(OrgPlan), nullable=False, default=OrgPlan.free)
    status          = Column(SAEnum(OrgStatus), nullable=False, default=OrgStatus.trial)
    admin_email     = Column(String(512), nullable=False)

    # Quotas (override plan defaults if set)
    monthly_llm_call_limit   = Column(Integer, nullable=True)   # None = use plan default
    monthly_cost_limit_usd   = Column(Integer, nullable=True)   # None = use plan default
    max_rag_documents        = Column(Integer, nullable=True)

    # Feature flags per tenant
    features        = Column(JSON, nullable=True)
    # e.g. {"rag_enabled": true, "streaming_enabled": false, "custom_policies": true}

    # Metadata
    created_at      = Column(DateTime(timezone=True),
                             default=lambda: datetime.now(timezone.utc), nullable=False)
    updated_at      = Column(DateTime(timezone=True),
                             default=lambda: datetime.now(timezone.utc),
                             onupdate=lambda: datetime.now(timezone.utc))
    suspended_at    = Column(DateTime(timezone=True), nullable=True)
    suspended_reason = Column(Text, nullable=True)

    @property
    def is_active(self) -> bool:
        return self.status in [OrgStatus.active, OrgStatus.trial]

    @property
    def effective_llm_limit(self) -> int:
        """Monthly LLM call limit based on plan or override."""
        if self.monthly_llm_call_limit is not None:
            return self.monthly_llm_call_limit
        return {
            OrgPlan.free:       100,
            OrgPlan.starter:    1_000,
            OrgPlan.pro:        10_000,
            OrgPlan.enterprise: 999_999_999,
        }.get(self.plan, 100)

    @property
    def effective_cost_limit_usd(self) -> float:
        """Monthly cost limit in USD based on plan or override."""
        if self.monthly_cost_limit_usd is not None:
            return float(self.monthly_cost_limit_usd)
        return {
            OrgPlan.free:       5.0,
            OrgPlan.starter:    50.0,
            OrgPlan.pro:        500.0,
            OrgPlan.enterprise: 99_999.0,
        }.get(self.plan, 5.0)
```

Add `org_id` to all existing models:

```python
# backend/database/models.py  (add org_id to existing models)

# On PolicyCheck, AuditEvent, OutputScanResult,
# IndexedDocument, RetrievalEvent, StreamSession,
# AIModel, PolicyRule — add this column:

org_id = Column(
    String(64),
    nullable=False,
    default="default",   # Existing data gets "default" org
    index=True,          # Always indexed — every query filters by it
)

# After migration, the "default" org is created in the Organization table.
# All existing rows are assigned to it.
# New rows always carry a real org_id from the middleware.
```

---

## ⚙️ Step 12.2 — Tenant Resolution Middleware

```python
# backend/middleware/tenant_middleware.py
"""
TenantResolutionMiddleware: resolves the org_id on every request.

Runs BEFORE the rate limiter and route handlers.
Every subsequent layer can trust request.state.org_id is valid.

This is the single point where org_id enters the request lifecycle.
No route handler should extract org_id from headers directly.
They all read request.state.org_id set here.
"""
import logging
from fastapi import Request, Response
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
from sqlalchemy import select

from backend.cache.redis_client import cache_get_json, cache_set_json

logger = logging.getLogger(__name__)

# Paths that don't need org resolution (public endpoints)
TENANT_BYPASS_PATHS = {
    "/",
    "/health",
    "/health/",
    "/docs",
    "/openapi.json",
    "/redoc",
    "/tenants",      # Tenant registration itself doesn't need existing org
}

ORG_CACHE_TTL = 60   # Cache org validation for 60 seconds


class TenantResolutionMiddleware(BaseHTTPMiddleware):
    """
    Resolves X-Org-ID header to a validated Organization record.
    
    On every request:
      1. Read X-Org-ID header
      2. Check Redis cache for org validation (60s TTL)
      3. On cache miss: query PostgreSQL organizations table
      4. Verify org is active
      5. Attach org_id to request.state.org_id
      6. Check org-level monthly quota
    
    Returns 400 if X-Org-ID is missing or invalid.
    Returns 403 if org is suspended.
    Returns 429 if org has exceeded monthly quota.
    """

    def __init__(self, app, db_session_factory):
        super().__init__(app)
        self.db_factory = db_session_factory

    async def dispatch(self, request: Request, call_next) -> Response:
        # Skip tenant resolution for public paths
        if request.url.path in TENANT_BYPASS_PATHS:
            request.state.org_id = "default"
            return await call_next(request)

        # ── Extract X-Org-ID header ────────────────────────────────
        org_id = request.headers.get("X-Org-ID", "").strip()
        if not org_id:
            return JSONResponse(
                status_code=400,
                content={
                    "error":   "missing_org_id",
                    "message": "X-Org-ID header is required. "
                               "Set it to your organization slug (e.g. 'acme-corp').",
                }
            )

        # ── Validate org_id format ─────────────────────────────────
        import re
        if not re.match(r'^[a-z0-9][a-z0-9\-]{0,62}[a-z0-9]$', org_id):
            return JSONResponse(
                status_code=400,
                content={
                    "error":   "invalid_org_id_format",
                    "message": "X-Org-ID must be lowercase alphanumeric with hyphens, "
                               "2-64 characters (e.g. 'acme-corp').",
                }
            )

        # ── Redis cache lookup for org validation ──────────────────
        cache_key = f"org:validation:{org_id}"
        cached    = await cache_get_json(cache_key)

        if cached:
            if cached.get("status") == "suspended":
                return JSONResponse(
                    status_code=403,
                    content={
                        "error":   "org_suspended",
                        "message": f"Organization '{org_id}' is suspended. "
                                   f"Reason: {cached.get('suspended_reason', 'Contact support.')}",
                    }
                )
            if not cached.get("exists"):
                return JSONResponse(
                    status_code=400,
                    content={
                        "error":   "org_not_found",
                        "message": f"Organization '{org_id}' is not registered. "
                                   f"Register at POST /tenants/",
                    }
                )
        else:
            # ── Cache miss: query PostgreSQL ───────────────────────
            try:
                async with self.db_factory() as db:
                    from backend.database.models import Organization
                    result = await db.execute(
                        select(Organization).where(Organization.org_id == org_id)
                    )
                    org = result.scalar_one_or_none()
            except Exception as e:
                logger.error(f"Tenant resolution DB error: {e}")
                # Fail open for DB errors — don't block legitimate traffic
                request.state.org_id = org_id
                return await call_next(request)

            if not org:
                await cache_set_json(
                    cache_key, {"exists": False}, ttl_seconds=ORG_CACHE_TTL
                )
                return JSONResponse(
                    status_code=400,
                    content={
                        "error":   "org_not_found",
                        "message": f"Organization '{org_id}' is not registered.",
                    }
                )

            if not org.is_active:
                await cache_set_json(
                    cache_key,
                    {
                        "exists":           True,
                        "status":           org.status.value,
                        "suspended_reason": org.suspended_reason,
                    },
                    ttl_seconds=ORG_CACHE_TTL,
                )
                return JSONResponse(
                    status_code=403,
                    content={
                        "error":   "org_suspended",
                        "message": f"Organization '{org_id}' is {org.status.value}.",
                    }
                )

            # Cache valid org
            await cache_set_json(
                cache_key,
                {
                    "exists":     True,
                    "status":     org.status.value,
                    "plan":       org.plan.value,
                    "llm_limit":  org.effective_llm_limit,
                    "cost_limit": org.effective_cost_limit_usd,
                },
                ttl_seconds=ORG_CACHE_TTL,
            )

        # ── Check org-level monthly quota ──────────────────────────
        from datetime import datetime, timezone
        month    = datetime.now(timezone.utc).strftime("%Y-%m")
        cost_key = f"cost:org:{org_id}:total:{month}"
        from backend.cache.redis_client import redis_get
        current_cost = float(await redis_get(cost_key) or 0.0)

        org_data   = cached or {"cost_limit": 500.0}
        cost_limit = org_data.get("cost_limit", 500.0)

        if current_cost >= cost_limit:
            return JSONResponse(
                status_code=429,
                content={
                    "error":         "org_monthly_quota_exceeded",
                    "message":       f"Organization '{org_id}' has exceeded its monthly "
                                     f"AI spend limit of ${cost_limit:.2f}.",
                    "current_spend": current_cost,
                    "limit_usd":     cost_limit,
                    "resets":        "1st of next month",
                }
            )

        # ── Attach org_id to request state ─────────────────────────
        request.state.org_id = org_id

        # Add org_id to response headers for debugging
        response = await call_next(request)
        response.headers["X-Org-ID"] = org_id
        return response
```

Register in `main.py`:

```python
# backend/main.py  (add tenant middleware — must run before rate limiter)
from backend.middleware.tenant_middleware import TenantResolutionMiddleware
from backend.database.db import AsyncSessionLocal

# Add BEFORE RateLimitMiddleware (middlewares run in reverse registration order)
app.add_middleware(TenantResolutionMiddleware, db_session_factory=AsyncSessionLocal)
app.add_middleware(RateLimitMiddleware)

# Middleware execution order (outermost first):
#   TenantResolutionMiddleware → RateLimitMiddleware → Route Handler
```

---

## ⚙️ Step 12.3 — Database Migration: org_id on All Tables

```bash
# Generate migration
alembic revision --autogenerate -m "add_org_id_to_all_tables_and_organizations_table"
```

The auto-generated migration will add `org_id` columns. Review and enhance it:

```python
# alembic/versions/xxxx_add_org_id_to_all_tables.py
from alembic import op
import sqlalchemy as sa


def upgrade() -> None:
    # ── Create organizations table ─────────────────────────────────
    op.create_table(
        "organizations",
        sa.Column("org_id",           sa.String(64),  primary_key=True),
        sa.Column("name",             sa.String(256), nullable=False),
        sa.Column("plan",             sa.String(32),  nullable=False, server_default="free"),
        sa.Column("status",           sa.String(32),  nullable=False, server_default="trial"),
        sa.Column("admin_email",      sa.String(512), nullable=False),
        sa.Column("monthly_llm_call_limit", sa.Integer, nullable=True),
        sa.Column("monthly_cost_limit_usd", sa.Integer, nullable=True),
        sa.Column("max_rag_documents",      sa.Integer, nullable=True),
        sa.Column("features",         sa.JSON,   nullable=True),
        sa.Column("created_at",       sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column("updated_at",       sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column("suspended_at",     sa.DateTime(timezone=True), nullable=True),
        sa.Column("suspended_reason", sa.Text,   nullable=True),
    )

    # ── Insert default organization for existing data ──────────────
    op.execute("""
        INSERT INTO organizations (org_id, name, plan, status, admin_email)
        VALUES ('default', 'Default Organization', 'enterprise', 'active', 'admin@localhost')
        ON CONFLICT DO NOTHING
    """)

    # ── Add org_id to all governance tables ───────────────────────
    tables = [
        "policy_checks",
        "audit_events",
        "output_scan_results",
        "indexed_documents",
        "retrieval_events",
        "stream_sessions",
        "ai_models",
        "policy_rules",
        "metrics_snapshots",
    ]
    for table in tables:
        # Add column (nullable first to avoid issues with existing rows)
        op.add_column(table, sa.Column("org_id", sa.String(64), nullable=True))
        # Backfill existing rows with "default"
        op.execute(f"UPDATE {table} SET org_id = 'default' WHERE org_id IS NULL")
        # Now make it NOT NULL
        op.alter_column(table, "org_id", nullable=False, server_default="default")
        # Add index for fast tenant queries
        op.create_index(f"ix_{table}_org_id", table, ["org_id"])


def downgrade() -> None:
    tables = [
        "policy_checks", "audit_events", "output_scan_results",
        "indexed_documents", "retrieval_events", "stream_sessions",
        "ai_models", "policy_rules", "metrics_snapshots",
    ]
    for table in tables:
        op.drop_index(f"ix_{table}_org_id", table)
        op.drop_column(table, "org_id")
    op.drop_table("organizations")
```

```bash
# Run the migration
alembic upgrade head

# Verify tables
docker exec -it governance-postgres psql -U postgres -d ai_governance \
  -c "SELECT column_name, data_type FROM information_schema.columns WHERE table_name='audit_events' AND column_name='org_id';"
# Should show: org_id | character varying
```

---

## ⚙️ Step 12.4 — Tenant-Scoped Query Helpers

```python
# backend/tenancy/scoped_db.py
"""
TenantScopedDB: a thin wrapper around SQLAlchemy AsyncSession
that automatically adds WHERE org_id = ? to every query.

Usage:
    scoped = TenantScopedDB(db, org_id="acme-corp")
    # This automatically adds .where(Model.org_id == "acme-corp")
    checks = await scoped.get_all(PolicyCheck)
    check  = await scoped.get_by_id(PolicyCheck, "check-uuid")
    await scoped.add(PolicyCheck(org_id=scoped.org_id, ...))
"""
import logging
from typing import Type, TypeVar, Optional
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

logger = logging.getLogger(__name__)
T = TypeVar("T")


class TenantScopedDB:
    """
    Every query through this class is automatically filtered by org_id.
    
    This is the primary defense against cross-tenant data access.
    A bug in route code (forgetting to add .where(org_id=...)) is caught
    here at the data layer.
    
    Pattern: use TenantScopedDB for ALL data access in route handlers.
    Never use the raw db session directly for tenant-owned data.
    """

    def __init__(self, db: AsyncSession, org_id: str):
        self.db     = db
        self.org_id = org_id

        if not org_id or org_id == "unknown":
            raise ValueError(
                f"TenantScopedDB created with invalid org_id='{org_id}'. "
                f"This is a bug — org_id must be resolved by middleware first."
            )

    async def get_all(
        self,
        model:   Type[T],
        limit:   int = 100,
        offset:  int = 0,
        **filters,
    ) -> list[T]:
        """Get all records of a model type, scoped to this org."""
        query = select(model).where(model.org_id == self.org_id)
        for field, value in filters.items():
            query = query.where(getattr(model, field) == value)
        query = query.offset(offset).limit(limit)
        result = await self.db.execute(query)
        return result.scalars().all()

    async def get_by_id(
        self,
        model:      Type[T],
        primary_key: str,
        pk_column:  str = None,
    ) -> Optional[T]:
        """
        Get a single record by primary key, scoped to this org.
        Raises PermissionError if the record exists but belongs to another org.
        """
        # First fetch by primary key (ignoring org)
        pk_col = pk_column or list(model.__table__.primary_key.columns)[0].name
        result = await self.db.execute(
            select(model).where(getattr(model, pk_col) == primary_key)
        )
        record = result.scalar_one_or_none()

        if record is None:
            return None

        # Verify it belongs to this org
        record_org = getattr(record, "org_id", None)
        if record_org and record_org != self.org_id:
            logger.warning(
                f"Cross-tenant access attempt: "
                f"requested org={self.org_id}, record org={record_org}, "
                f"model={model.__name__}, pk={primary_key}"
            )
            raise PermissionError(
                f"Record '{primary_key}' does not belong to org '{self.org_id}'"
            )

        return record

    async def count(self, model: Type[T], **filters) -> int:
        """Count records scoped to this org."""
        from sqlalchemy import func
        query = select(func.count()).select_from(model).where(
            model.org_id == self.org_id
        )
        for field, value in filters.items():
            query = query.where(getattr(model, field) == value)
        result = await self.db.execute(query)
        return result.scalar() or 0

    def add(self, record: T) -> T:
        """
        Add a record, ensuring org_id is set correctly.
        Overwrites any org_id on the record with the scoped org_id.
        This prevents a user from saving data under a different org.
        """
        if hasattr(record, "org_id"):
            record.org_id = self.org_id
        self.db.add(record)
        return record


def get_org_id(request) -> str:
    """
    Extract org_id from request state (set by TenantResolutionMiddleware).
    Use this in every route handler as the single source of org_id.
    """
    org_id = getattr(request.state, "org_id", None)
    if not org_id:
        raise ValueError(
            "org_id not found in request.state. "
            "TenantResolutionMiddleware must run before this handler."
        )
    return org_id
```

Update all route handlers to use `TenantScopedDB`:

```python
# backend/routers/policies.py  (updated check_policy with tenant scoping)
from backend.tenancy.scoped_db import TenantScopedDB, get_org_id

@router.post("/check", response_model=PolicyCheckResponse)
async def check_policy(
    request_body: PolicyCheckRequest,
    request:      Request,
    db:           AsyncSession = Depends(get_db),
):
    org_id = get_org_id(request)
    scoped = TenantScopedDB(db, org_id)

    # ── Cache key now includes org_id ──────────────────────────────
    cache_input = f"{org_id}:{request_body.prompt}:{request_body.user_role}:{request_body.model_id}"
    cache_key   = f"cache:org:{org_id}:policy:{hashlib.sha256(cache_input.encode()).hexdigest()}"

    cached = await cache_get_json(cache_key)
    if cached:
        cached["cache_hit"] = True
        return PolicyCheckResponse(**cached)

    # ── Load only THIS ORG's policy rules ─────────────────────────
    rules = await scoped.get_all(PolicyRule, enabled=True)

    # ... rest of policy logic unchanged ...

    # ── Write audit record with org_id ────────────────────────────
    scoped.add(PolicyCheck(
        check_id    = check_id,
        org_id      = org_id,   # Explicit — TenantScopedDB also sets this
        model_id    = request_body.model_id,
        user_id     = request_body.user_id,
        user_role   = request_body.user_role,
        prompt_hash = prompt_hash,
        violations  = violations,
        allowed     = allowed,
    ))
    await db.commit()
```

---

## ⚙️ Step 12.5 — Tenant-Scoped Policy Rules

```python
# backend/routers/policies.py  (tenant-scoped rule management)
from backend.tenancy.scoped_db import TenantScopedDB, get_org_id


@router.post("/rules", summary="Create a custom policy rule for this org")
async def create_policy_rule(
    rule:    PolicyRuleCreate,
    request: Request,
    db:      AsyncSession = Depends(get_db),
):
    """
    Creates a policy rule scoped to the calling organization.
    
    Acme Corp's rules only fire for Acme Corp's policy checks.
    Beta Inc never sees or is affected by Acme's rules.
    
    System-wide default rules (org_id="default") apply to all orgs
    UNLESS the org has overridden them with their own version.
    """
    org_id = get_org_id(request)
    scoped = TenantScopedDB(db, org_id)

    # Check if org already has a rule with this rule_id
    existing_rules = await scoped.get_all(PolicyRule)
    existing_ids   = {r.rule_id for r in existing_rules}
    if rule.rule_id in existing_ids:
        from fastapi import HTTPException
        raise HTTPException(
            status_code=409,
            detail=f"Rule '{rule.rule_id}' already exists for org '{org_id}'."
        )

    new_rule = scoped.add(PolicyRule(
        rule_id             = rule.rule_id,
        name                = rule.name,
        description         = rule.description,
        pattern             = rule.pattern,
        action              = rule.action,
        risk_score_increment = rule.risk_score_increment,
        applies_to_roles    = rule.applies_to_roles,
        enabled             = True,
    ))
    await db.commit()
    return new_rule


@router.get("/rules", summary="List policy rules for this org")
async def list_policy_rules(
    include_defaults: bool = True,
    request:          Request = None,
    db:               AsyncSession = Depends(get_db),
):
    """
    Returns this org's custom rules, plus system-wide default rules
    (if include_defaults=True).
    
    Custom rules always override default rules with the same rule_id.
    """
    org_id = get_org_id(request)
    scoped = TenantScopedDB(db, org_id)

    org_rules  = await scoped.get_all(PolicyRule)

    if include_defaults:
        # Load default rules (org_id="default")
        default_scoped = TenantScopedDB(db, "default")
        default_rules  = await default_scoped.get_all(PolicyRule)

        # Org rules override defaults with the same rule_id
        org_rule_ids = {r.rule_id for r in org_rules}
        merged = list(org_rules) + [
            r for r in default_rules if r.rule_id not in org_rule_ids
        ]
        return merged

    return org_rules
```

---

## ⚙️ Step 12.6 — Tenant-Scoped RAG Isolation

```python
# backend/rag/governed_rag.py  (updated with org_id isolation)

class GovernedRAG:
    """
    All RAG operations are now org-scoped.
    Documents indexed by Acme Corp are never returned to Beta Inc.
    """

    async def index_document(
        self,
        content:    str,
        title:      str,
        indexed_by: str,
        org_id:     str,           # ← NEW required parameter
        clearance:  str = "internal",
        source:     str = "",
        metadata:   dict = None,
        ttl_days:   int = None,
    ) -> dict:
        # ... existing scan logic ...

        # ── Store in ChromaDB with org_id in metadata ──────────────
        metadatas = [
            {
                "doc_id":      doc_id,
                "chunk_index": i,
                "clearance":   clearance,
                "org_id":      org_id,     # ← KEY isolation field
                "indexed_by":  indexed_by,
                "source":      source,
                "title":       title,
                "expires_at":  expires_at.isoformat(),
                **(metadata or {}),
            }
            for i in range(len(raw_chunks))
        ]
        # ... rest of indexing logic ...

    async def retrieve(
        self,
        query:      str,
        user_id:    str,
        user_role:  str,
        org_id:     str,           # ← NEW required parameter
        session_id: str = None,
        k:          int = None,
    ) -> dict:
        allowed_clearances = get_allowed_clearances(user_role)
        effective_k        = k or get_k_for_role(user_role)
        now_iso            = datetime.now(timezone.utc).isoformat()

        # ── Cache key scoped by org_id ─────────────────────────────
        query_hash = self._hash(query)
        cache_key  = f"cache:org:{org_id}:rag:{user_role}:{query_hash}"

        cached = await cache_get_json(cache_key)
        if cached:
            return {**cached, "from_cache": True}

        # ── ChromaDB query with org_id + clearance filter ──────────
        collection = self._get_collection()
        try:
            query_embedding = await self.embeddings.aembed_query(query)
            results = collection.query(
                query_embeddings = [query_embedding],
                n_results        = effective_k,
                where            = {
                    "$and": [
                        {"org_id":     {"$eq":  org_id}},           # ← Tenant isolation
                        {"clearance":  {"$in":  allowed_clearances}},
                        {"expires_at": {"$gt":  now_iso}},
                    ]
                },
                include = ["documents", "metadatas", "distances"],
            )
        except Exception as e:
            logger.error(f"ChromaDB query failed: {e}")
            return {"chunks": [], "doc_ids": [], "from_cache": False, "filtered_count": 0}

        # ... rest of retrieval, re-scan, cache, audit logic unchanged ...
```

Update RAG endpoints to pass org_id:

```python
# backend/routers/rag.py
@router.post("/index")
async def index_text(body: IndexTextRequest, request: Request, db: AsyncSession = Depends(get_db)):
    org_id = get_org_id(request)
    user_id = request.headers.get("X-User-ID", "anonymous")
    rag = GovernedRAG(db_session=db)
    result = await rag.index_document(
        content=body.content, title=body.title,
        indexed_by=user_id, org_id=org_id,          # ← Pass org_id
        clearance=body.clearance,
    )
    await db.commit()
    return result


@router.post("/query")
async def query_rag(body: QueryRequest, request: Request, db: AsyncSession = Depends(get_db)):
    org_id   = get_org_id(request)
    user_id  = request.headers.get("X-User-ID", "anonymous")
    user_role = request.headers.get("X-User-Role", "employee")
    rag = GovernedRAG(db_session=db)
    result = await rag.retrieve(
        query=body.query, user_id=user_id,
        user_role=user_role, org_id=org_id,          # ← Pass org_id
    )
    await db.commit()
    return result
```

---

## ⚙️ Step 12.7 — Tenant-Scoped Metrics and Cost Tracking

```python
# backend/metrics/collector.py  (updated with org_id scoping)

class MetricsCollector:
    """
    MetricsCollector now accepts an org_id parameter.
    All aggregations are scoped to that org only.
    """

    def __init__(self, db_session, org_id: str = "default"):
        self.db     = db_session
        self.org_id = org_id

    async def collect(self, period_hours: int = 24) -> GovernanceMetricsSnapshot:
        """Collect metrics scoped to self.org_id."""
        since = datetime.now(timezone.utc) - timedelta(hours=period_hours)
        # All sub-collectors now filter by self.org_id
        policy      = await self._collect_policy(since)
        llm         = await self._collect_llm(since)
        # ...

    async def _collect_policy(self, since: datetime) -> PolicyMetrics:
        result = await self.db.execute(
            select(PolicyCheck).where(
                PolicyCheck.org_id    == self.org_id,    # ← Scoped
                PolicyCheck.checked_at >= since,
            )
        )
        checks = result.scalars().all()
        # ... rest of aggregation unchanged ...

    async def _collect_llm(self, since: datetime) -> LLMMetrics:
        result = await self.db.execute(
            select(AuditEvent).where(
                AuditEvent.org_id      == self.org_id,   # ← Scoped
                AuditEvent.action_type == "llm_call",
                AuditEvent.timestamp   >= since,
            )
        )
        # ...

        # Cost keys are also org-scoped
        today = datetime.now(timezone.utc).strftime("%Y-%m-%d")
        r     = await get_redis()
        cost_by_model: dict = {}
        if r:
            # ← Keys now include org_id
            model_keys = await r.keys(f"cost:org:{self.org_id}:model:*:{today}")
            for key in model_keys:
                model_id = key.split(":")[4]
                cost_by_model[model_id] = round(float(await redis_get(key) or 0.0), 4)
        # ...
```

Update the WebSocket endpoint to scope by org:

```python
# backend/routers/metrics.py
@router.websocket("/ws/metrics")
async def metrics_websocket(websocket: WebSocket):
    """
    Org-scoped live metrics WebSocket.
    Client must pass X-Org-ID as query parameter:
      ws://localhost:8000/metrics/ws/metrics?org_id=acme-corp
    """
    org_id = websocket.query_params.get("org_id", "default")
    # Validate org_id before accepting connection
    # ...

    await ws_manager.connect(websocket)
    try:
        while True:
            await asyncio.sleep(PUSH_INTERVAL)
            async with AsyncSessionLocal() as db:
                # ← Scoped collector
                collector = MetricsCollector(db_session=db, org_id=org_id)
                snapshot  = await collector.collect(period_hours=PERIOD_HOURS)
            await ws_manager.send_personal(websocket, snapshot.to_json())
    except WebSocketDisconnect:
        ws_manager.disconnect(websocket)
```

---

## ⚙️ Step 12.8 — Tenant Management Endpoints

```python
# backend/routers/tenants.py
from fastapi import APIRouter, Depends, HTTPException, Request
from pydantic import BaseModel, Field, EmailStr
from typing import Optional
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from datetime import datetime, timezone

from backend.database.db import get_db
from backend.database.models import Organization, OrgPlan, OrgStatus
from backend.cache.redis_client import redis_delete, redis_get

router = APIRouter()


class TenantCreate(BaseModel):
    org_id:      str   = Field(..., pattern=r'^[a-z0-9][a-z0-9\-]{0,62}[a-z0-9]$')
    name:        str   = Field(..., min_length=2, max_length=256)
    admin_email: str   = Field(...)
    plan:        str   = Field("free")


class TenantUpdate(BaseModel):
    name:                    Optional[str]   = None
    plan:                    Optional[str]   = None
    monthly_llm_call_limit:  Optional[int]   = None
    monthly_cost_limit_usd:  Optional[int]   = None
    features:                Optional[dict]  = None


@router.post("/", summary="Register a new organization (tenant)")
async def create_tenant(body: TenantCreate, db: AsyncSession = Depends(get_db)):
    """
    Register a new organization. Creates the tenant record and
    seeds their account with the default policy rules.
    
    org_id is permanent and cannot be changed after creation.
    """
    # Check for duplicate
    existing = await db.execute(
        select(Organization).where(Organization.org_id == body.org_id)
    )
    if existing.scalar_one_or_none():
        raise HTTPException(
            status_code=409,
            detail=f"Organization '{body.org_id}' already exists."
        )

    try:
        plan = OrgPlan(body.plan)
    except ValueError:
        raise HTTPException(
            status_code=422,
            detail=f"Invalid plan '{body.plan}'. Valid: free, starter, pro, enterprise."
        )

    org = Organization(
        org_id      = body.org_id,
        name        = body.name,
        plan        = plan,
        status      = OrgStatus.trial,
        admin_email = body.admin_email,
        features    = {
            "rag_enabled":       True,
            "streaming_enabled": True,
            "custom_policies":   plan in [OrgPlan.pro, OrgPlan.enterprise],
        },
    )
    db.add(org)

    # Seed default policy rules for the new org
    from backend.routers.policies import seed_default_rules
    await seed_default_rules(db, org_id=body.org_id)

    await db.commit()
    return {
        "org_id":     org.org_id,
        "name":       org.name,
        "plan":       org.plan.value,
        "status":     org.status.value,
        "llm_limit":  org.effective_llm_limit,
        "cost_limit": org.effective_cost_limit_usd,
        "created_at": org.created_at.isoformat() if org.created_at else None,
    }


@router.get("/{org_id}", summary="Get tenant profile and settings")
async def get_tenant(org_id: str, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Organization).where(Organization.org_id == org_id)
    )
    org = result.scalar_one_or_none()
    if not org:
        raise HTTPException(status_code=404, detail=f"Organization '{org_id}' not found.")
    return {
        "org_id":                 org.org_id,
        "name":                   org.name,
        "plan":                   org.plan.value,
        "status":                 org.status.value,
        "admin_email":            org.admin_email,
        "llm_limit_monthly":      org.effective_llm_limit,
        "cost_limit_monthly_usd": org.effective_cost_limit_usd,
        "features":               org.features,
        "created_at":             org.created_at.isoformat() if org.created_at else None,
    }


@router.patch("/{org_id}", summary="Update tenant plan or settings")
async def update_tenant(
    org_id: str,
    body:   TenantUpdate,
    db:     AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Organization).where(Organization.org_id == org_id)
    )
    org = result.scalar_one_or_none()
    if not org:
        raise HTTPException(status_code=404, detail=f"Organization '{org_id}' not found.")

    if body.name:                   org.name = body.name
    if body.plan:                   org.plan = OrgPlan(body.plan)
    if body.monthly_llm_call_limit: org.monthly_llm_call_limit = body.monthly_llm_call_limit
    if body.monthly_cost_limit_usd: org.monthly_cost_limit_usd = body.monthly_cost_limit_usd
    if body.features:               org.features = {**(org.features or {}), **body.features}

    # Invalidate the org validation cache
    await redis_delete(f"org:validation:{org_id}")

    await db.commit()
    return {"updated": True, "org_id": org_id}


@router.get("/{org_id}/usage", summary="Current month usage and cost for a tenant")
async def get_tenant_usage(org_id: str, db: AsyncSession = Depends(get_db)):
    """
    Returns the current month's usage for billing and quota monitoring.
    """
    from datetime import datetime, timezone
    month = datetime.now(timezone.utc).strftime("%Y-%m")
    today = datetime.now(timezone.utc).strftime("%Y-%m-%d")

    # Cost from Redis
    total_cost = float(await redis_get(f"cost:org:{org_id}:total:{month}") or 0.0)

    # LLM call count from audit events this month
    from backend.database.models import AuditEvent
    from datetime import timedelta
    since  = datetime.now(timezone.utc).replace(day=1, hour=0, minute=0, second=0)
    result = await db.execute(
        select(AuditEvent).where(
            AuditEvent.org_id      == org_id,
            AuditEvent.action_type == "llm_call",
            AuditEvent.timestamp   >= since,
        )
    )
    llm_calls = len(result.scalars().all())

    # Get org limits
    org_result = await db.execute(
        select(Organization).where(Organization.org_id == org_id)
    )
    org = org_result.scalar_one_or_none()
    if not org:
        raise HTTPException(status_code=404, detail=f"Organization '{org_id}' not found.")

    return {
        "org_id":              org_id,
        "month":               month,
        "llm_calls_used":      llm_calls,
        "llm_calls_limit":     org.effective_llm_limit,
        "llm_calls_remaining": max(0, org.effective_llm_limit - llm_calls),
        "cost_usd_used":       round(total_cost, 4),
        "cost_usd_limit":      org.effective_cost_limit_usd,
        "cost_usd_remaining":  round(max(0.0, org.effective_cost_limit_usd - total_cost), 4),
        "plan":                org.plan.value,
    }


# Register in main.py:
# app.include_router(tenants.router, prefix="/tenants", tags=["Tenant Management"])
```

---

## ⚙️ Step 12.9 — Tenant-Scoped Redis Keys

Every Redis key must now include `org_id`. This ensures cost counters, caches, and rate limits are fully isolated between tenants.

```python
# backend/tenancy/tenant_keys.py
"""
Centralized Redis key generation for multi-tenant isolation.
All Redis keys must go through these functions.
Never construct Redis keys manually in route handlers.
"""


def policy_cache_key(org_id: str, cache_hash: str) -> str:
    return f"cache:org:{org_id}:policy:{cache_hash}"


def llm_cache_key(org_id: str, cache_hash: str) -> str:
    return f"cache:org:{org_id}:llm:{cache_hash}"


def rag_cache_key(org_id: str, user_role: str, query_hash: str) -> str:
    return f"cache:org:{org_id}:rag:{user_role}:{query_hash}"


def model_cache_key(org_id: str, model_id: str) -> str:
    return f"cache:org:{org_id}:model:{model_id}"


def rate_limit_user_key(org_id: str, user_id: str, window: str) -> str:
    return f"rate:org:{org_id}:user:{user_id}:{window}"


def rate_limit_ip_key(org_id: str, ip: str, window: str) -> str:
    return f"rate:org:{org_id}:ip:{ip}:{window}"


def cost_model_key(org_id: str, model_id: str, date: str) -> str:
    return f"cost:org:{org_id}:model:{model_id}:{date}"


def cost_user_key(org_id: str, user_id: str, date: str) -> str:
    return f"cost:org:{org_id}:user:{user_id}:{date}"


def cost_total_key(org_id: str, month: str) -> str:
    return f"cost:org:{org_id}:total:{month}"


def org_validation_key(org_id: str) -> str:
    return f"org:validation:{org_id}"


def stream_active_key(org_id: str, user_id: str) -> str:
    return f"stream:org:{org_id}:active:{user_id}"


def metrics_stat_key(org_id: str, stat: str) -> str:
    return f"metrics:org:{org_id}:{stat}"
```

---

## ⚙️ Step 12.10 — Testing Multi-Tenant Isolation

```python
# tests/unit/test_multitenancy.py
import pytest
from unittest.mock import AsyncMock, patch


@pytest.mark.unit
@pytest.mark.governance
class TestTenantIsolation:

    async def test_org_a_cannot_retrieve_org_b_documents(
        self, client
    ):
        """
        GOVERNANCE: RAG retrieval must be scoped to the calling org.
        Org A's documents must never appear in Org B's query results.
        """
        # Register two orgs
        await client.post("/tenants/", json={
            "org_id": "org-alpha", "name": "Org Alpha",
            "admin_email": "admin@alpha.com", "plan": "pro"
        })
        await client.post("/tenants/", json={
            "org_id": "org-beta", "name": "Org Beta",
            "admin_email": "admin@beta.com", "plan": "pro"
        })

        # Org A indexes a confidential document
        with patch("backend.rag.governed_rag.GovernedRAG.index_document") as mock_idx:
            mock_idx.return_value = {
                "indexed": True, "doc_id": "doc-alpha-001",
                "chunk_count": 3, "scan_status": "clean",
                "scan_findings": [], "risk_score": 0,
                "clearance": "internal", "expires_at": "2030-01-01",
            }
            await client.post("/rag/index",
                json={"content": "Alpha Corp salary: $500,000/yr",
                      "title": "Alpha Salaries", "clearance": "internal"},
                headers={"X-Org-ID": "org-alpha", "X-User-ID": "alice",
                         "X-User-Role": "employee"},
            )

        # Org B queries — must NOT get Org A's data
        with patch("backend.rag.governed_rag.GovernedRAG.retrieve") as mock_ret:
            mock_ret.return_value = {
                "chunks": [], "doc_ids": [],
                "from_cache": False, "filtered_count": 0, "chunks_count": 0,
            }
            response = await client.post("/rag/query",
                json={"query": "What are the salary bands?"},
                headers={"X-Org-ID": "org-beta", "X-User-ID": "bob",
                         "X-User-Role": "employee"},
            )
            assert response.status_code == 200
            # Verify retrieve() was called with org-beta (not org-alpha)
            call_kwargs = mock_ret.call_args.kwargs
            assert call_kwargs.get("org_id") == "org-beta"

    async def test_org_a_policy_rules_do_not_affect_org_b(
        self, client
    ):
        """
        GOVERNANCE: Custom policy rules are scoped to the creating org.
        Org A's rules must never fire for Org B's requests.
        """
        # Create orgs
        for org_id in ["rule-org-a", "rule-org-b"]:
            await client.post("/tenants/", json={
                "org_id": org_id, "name": org_id,
                "admin_email": f"admin@{org_id}.com", "plan": "pro"
            })

        # Org A creates a custom rule blocking "competitor"
        await client.post("/policies/rules",
            json={
                "rule_id":              "org-a-competitor-block",
                "name":                 "Block competitor mentions",
                "pattern":              r"(?i)\bcompetitor\b",
                "action":               "block",
                "risk_score_increment": 50,
                "applies_to_roles":     ["employee"],
            },
            headers={"X-Org-ID": "rule-org-a", "X-User-ID": "admin",
                     "X-User-Role": "admin"},
        )

        # Org B asks a question with "competitor" — should NOT be blocked
        response = await client.post("/policies/check",
            json={
                "prompt":     "How does our product compare to the competitor?",
                "user_role":  "employee",
                "model_id":   "test-model",
                "user_id":    "bob",
                "session_id": "s1",
            },
            headers={"X-Org-ID": "rule-org-b", "X-User-ID": "bob",
                     "X-User-Role": "employee"},
        )
        assert response.status_code == 200
        assert response.json()["allowed"] == True, (
            "Org B was blocked by Org A's rule — cross-tenant policy leak!"
        )

    async def test_missing_org_id_header_returns_400(self, client):
        """Every request (except public paths) requires X-Org-ID."""
        response = await client.post("/policies/check",
            json={"prompt": "hello", "user_role": "employee",
                  "model_id": "test", "user_id": "alice", "session_id": "s1"},
            # No X-Org-ID header
        )
        assert response.status_code == 400
        assert "X-Org-ID" in response.json()["message"]

    async def test_suspended_org_returns_403(self, client, db_session):
        """A suspended organization cannot make API calls."""
        from backend.database.models import Organization, OrgStatus
        # Create and immediately suspend an org
        org = Organization(
            org_id="suspended-org", name="Suspended",
            plan="free", status=OrgStatus.suspended,
            admin_email="x@x.com", suspended_reason="Payment failed",
        )
        db_session.add(org)
        await db_session.commit()

        response = await client.post("/policies/check",
            json={"prompt": "hello", "user_role": "employee",
                  "model_id": "test", "user_id": "alice", "session_id": "s1"},
            headers={"X-Org-ID": "suspended-org"},
        )
        assert response.status_code == 403
        assert "suspended" in response.json()["error"]

    async def test_tenant_usage_endpoint_scoped_to_org(self, client):
        """GET /tenants/{org_id}/usage returns only that org's usage."""
        # Create org
        await client.post("/tenants/", json={
            "org_id": "usage-test-org", "name": "Usage Test",
            "admin_email": "a@b.com", "plan": "starter"
        })
        response = await client.get("/tenants/usage-test-org/usage")
        assert response.status_code == 200
        data = response.json()
        assert data["org_id"]          == "usage-test-org"
        assert "llm_calls_used"        in data
        assert "llm_calls_limit"       in data
        assert "cost_usd_used"         in data
        assert "cost_usd_remaining"    in data

    async def test_redis_keys_are_org_scoped(self):
        """All Redis key constructors include org_id."""
        from backend.tenancy.tenant_keys import (
            policy_cache_key, llm_cache_key, cost_model_key, rate_limit_user_key
        )
        org_a = "acme-corp"
        org_b = "beta-inc"
        hash_ = "abc123"

        # Same hash, different orgs → different keys
        assert policy_cache_key(org_a, hash_) != policy_cache_key(org_b, hash_)
        assert llm_cache_key(org_a, hash_)    != llm_cache_key(org_b, hash_)
        assert cost_model_key(org_a, "gpt-4o", "2026-05-31") != \
               cost_model_key(org_b, "gpt-4o", "2026-05-31")

        # Keys contain org_id
        assert org_a in policy_cache_key(org_a, hash_)
        assert org_b in policy_cache_key(org_b, hash_)

    async def test_org_b_cost_does_not_appear_in_org_a_metrics(
        self, client, e2e_redis
    ):
        """
        Cost counters must be org-scoped.
        Org B's LLM spend must not show up in Org A's cost report.
        """
        from backend.tenancy.tenant_keys import cost_model_key
        from datetime import datetime, timezone

        today = datetime.now(timezone.utc).strftime("%Y-%m-%d")

        # Simulate Org B spending $10
        await e2e_redis.set(cost_model_key("org-b", "gpt-4o", today), "10.00")

        # Org A's cost should still be 0
        org_a_key = cost_model_key("org-a", "gpt-4o", today)
        org_a_cost = float(await e2e_redis.get(org_a_key) or 0.0)
        assert org_a_cost == 0.0, "Org B's cost leaked into Org A's counter"
```

---

## 🏁 What You Built Today

```
Week 3 Progress:
  Day 11 → Dashboard: real-time governance metrics with WebSockets
  Day 12 → Multi-tenant isolation: namespace all data by org_id  ← TODAY
  Day 13 → Alerting: fire PagerDuty/Slack on threshold breaches

New capabilities added:
  ✅ Organization model with plan, status, quotas, feature flags
  ✅ TenantResolutionMiddleware — resolves org_id on every request
  ✅ org_id added to 9 PostgreSQL tables (migration with backfill)
  ✅ "default" org created for backward compatibility
  ✅ TenantScopedDB helper — auto-scopes all queries
  ✅ Cross-tenant access detection (PermissionError on mismatch)
  ✅ Tenant-scoped policy rules (each org has its own rule set)
  ✅ Tenant-scoped RAG (ChromaDB filter: org_id=$eq=acme-corp)
  ✅ Tenant-scoped metrics (MetricsCollector takes org_id)
  ✅ Tenant-scoped WebSocket (org_id in query param)
  ✅ Centralized Redis key generation (tenant_keys.py)
  ✅ org-namespaced cache, rate limit, cost, stream keys
  ✅ POST /tenants/           — register a new organization
  ✅ GET  /tenants/{org_id}   — tenant profile
  ✅ PATCH /tenants/{org_id}  — update plan/settings
  ✅ GET  /tenants/{org_id}/usage — monthly usage + billing data
  ✅ Org-level monthly quota enforcement (cost limit per org)
  ✅ 8 tests verifying isolation at DB, Redis, ChromaDB, and middleware layers

```

### The Complete Governance Stack After Day 12

```
EVERY REQUEST NOW CARRIES AN org_id THROUGH ALL LAYERS

  Request: X-Org-ID: acme-corp, X-User-ID: alice, X-User-Role: analyst
       │
  [TenantResolution]  → Validates acme-corp, checks quota, sets state.org_id
       │
  [RateLimiter]       → Key: rate:org:acme-corp:user:alice:2026-05-31T14
       │
  [PolicyEngine]      → Loads only acme-corp's rules from DB
       │               → Cache key: cache:org:acme-corp:policy:{hash}
       │
  [RAGRetrieval]      → ChromaDB filter: {org_id: "acme-corp", clearance: [...]}
       │               → Cache key: cache:org:acme-corp:rag:analyst:{hash}
       │
  [LLMCall]           → Cost key: cost:org:acme-corp:model:gpt-4o:2026-05-31
       │
  [OutputScanner]     → Audit record: org_id="acme-corp"
       │
  [CostTracker]       → Updates cost:org:acme-corp:total:2026-05
       │
  [AuditLogger]       → AuditEvent(org_id="acme-corp", ...)
       │
  Response + X-Org-ID: acme-corp header

  Beta Inc's data:    → Never visible to Acme Corp
  Acme Corp's rules:  → Never fire for Beta Inc
  Acme Corp's costs:  → Never counted in Beta Inc's invoice

```

---

## ✅ Day 12 Checklist

Before moving to Day 13, verify all of these:

- [ ] `alembic upgrade head` created `organizations` table and added `org_id` to all 9 tables
- [ ] `POST /tenants/` creates a new org with correct plan limits
- [ ] `GET /tenants/acme-corp` returns org profile and quota settings
- [ ] Request without `X-Org-ID` header returns `400 Bad Request`
- [ ] Request with invalid org_id format returns `400` with format hint
- [ ] Request with unregistered org_id returns `400 org_not_found`
- [ ] Request with suspended org returns `403 org_suspended`
- [ ] Request with org over monthly cost quota returns `429`
- [ ] `POST /policies/check` with `X-Org-ID: org-a` creates PolicyCheck with `org_id="org-a"`
- [ ] Org A's custom policy rule does NOT fire for Org B's requests
- [ ] RAG retrieve() called with correct `org_id` from request context
- [ ] Redis keys all contain `org_id` (verify with `KEYS cache:org:*`)
- [ ] `GET /tenants/org-a/usage` shows only Org A's LLM calls and cost
- [ ] WebSocket `ws://localhost:8000/metrics/ws/metrics?org_id=acme-corp` shows only acme-corp metrics
- [ ] All existing unit tests still pass: `pytest -m unit` → 0 failures
- [ ] New tenant tests pass: `pytest tests/unit/test_multitenancy.py -v`
- [ ] Coverage stays above 90%: `pytest --cov=backend --cov-fail-under=90`

---

## 📚 What to Read Tonight

| Resource | Topic |
|---|---|
| [Stripe's Multi-Tenant Architecture](https://stripe.com/blog/idempotency) | How Stripe handles tenant isolation at scale |
| [PostgreSQL Row-Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) | DB-level tenant isolation (alternative to WHERE clause) |
| [OWASP — Insecure Direct Object Reference](https://owasp.org/www-community/attacks/Insecure_Direct_Object_Reference) | Why tenant isolation prevents IDOR attacks |
| [Redis Key Naming](https://redis.io/learn/howtos/patterns/key-naming) | Best practices for namespace design |
| [SOC 2 — Logical Access](https://www.aicpa-cima.com/resources/landing/soc-2) | Why tenant isolation is a SOC 2 requirement |
| [EU AI Act — Article 53](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A52021PC0206) | Data governance requirements for AI systems |

---

## 🔮 Coming Up

```
Day 13 → Alerting: fire PagerDuty/Slack on governance threshold breaches
Day 14 → Model versioning and canary deployments with governance
Day 15 → Week 3 Integration Tests: full multi-tenant E2E suite
```

> Day 13 is the alerting layer.
> The metrics dashboard (Day 11) makes governance data visible.
> Day 13 makes it actionable: when the toxicity block rate crosses 10%,
> a Slack message fires to the #ai-governance channel.
> When a tenant exceeds 80% of their monthly cost quota,
> their billing admin gets an email.
> When the hallucination rate drops below 0.50 entailment score,
> PagerDuty wakes up the on-call engineer.
>
> Monitoring without alerting is just a dashboard nobody watches.

---

*Day 12 complete. The governance system now knows who it is talking to.*

*Every request carries an org_id. Every byte of data belongs to exactly one tenant. Acme Corp's salaries stay inside Acme Corp. Beta Inc's policy rules fire only for Beta Inc. The audit log knows which organization every action belongs to.*

*This is the architecture that makes the system sellable as a B2B platform. Without this day, you have an internal tool. With this day, you have a product.*

---

**Repository:** [GuntruTirupathamma/AI-Governance](https://github.com/GuntruTirupathamma/AI-Governance)  **Series:** AI Governance Engineering from Scratch  **Next:** `DAY13.md` → Alerting: Fire PagerDuty/Slack on Governance Threshold Breaches
