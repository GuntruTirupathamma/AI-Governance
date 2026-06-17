---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 25 of 35
Previous: DAY24.md - Compliance Reporting and Evidence Export
Next: DAY26.md - Incident Response Playbooks and Automated Remediation
---

# Day 25: Policy-as-Code - Dynamic Governance Rules

## Table of contents

- [The problem with rules baked into source files](#the-problem)
- [What policy-as-code means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 25.1 - The policy schema and database model](#step-251)
- [Step 25.2 - Alembic migration for governance_policies](#step-252)
- [Step 25.3 - The policy store: DB reads with an in-process cache](#step-253)
- [Step 25.4 - Seeding defaults from the hardcoded dicts](#step-254)
- [Step 25.5 - Drop-in loader replacements](#step-255)
- [Step 25.6 - Wiring the loaders and auditing policy changes](#step-256)
- [Step 25.7 - The policies admin router](#step-257)
- [Step 25.8 - Tests](#step-258)
- [What you built today](#what-you-built)
- [Day 25 checklist](#checklist)

<a id="the-problem"></a>
## The problem with rules baked into source files

`ALERT_RULES` (Day 22), `BUDGET_LIMITS` (Day 20), `ABUSE_THRESHOLDS` (Day 18), `AGENT_SECRET_SCOPES` (Day 19): every governance decision in this system traces back to one of these four structures. They are correct, they are documented, and they are completely inaccessible to anyone without a pull request, a review, and a deployment.

The problem with that is not academic. An incident happens at 02:00. The on-call engineer can see from Day 23's dashboard that `alerts_last_hour.critical` is spiking. They know the fix: lower the `ABUSE_THRESHOLDS` count for `INJECTION_DETECTED` from 5 to 2 so offenders get suspended faster. But the change lives in `backend/security/abuse_schema.py`. The deployment pipeline takes 20 minutes on a good day. By the time the change is live, the incident has been running for half an hour.

Alternatively: a large customer wants their budget limits raised for the quarter-end sprint. That's a two-line change in `BUDGET_LIMITS`. It goes into the same pull-request queue as feature work. The customer waits.

The fix is not to remove the hardcoded dicts: they're the correct defaults and the source of truth during development. The fix is to make the runtime read governance rules from the database, seeded from those dicts, so an admin with the right credentials can update a rule in seconds and every in-flight request sees the new value within 30 seconds.

<a id="what-it-means-here"></a>
## What policy-as-code means here

A single `governance_policies` table stores governance rules as JSON rows, keyed by `(table_name, rule_key)`. Day 22's `ALERT_RULES` becomes the `alert_rules` policy table; Day 20's `BUDGET_LIMITS` becomes the `budget_limits` policy table. Each row holds the full rule as JSON, a version counter, a `notes` field for human-readable change rationale, and a flag marking whether the value is still the seeded default or has been updated by an admin.

A thin `PolicyStore` reads from the database and keeps a 30-second in-process cache per rule. A rule update (via `PUT /policies/{table}/{key}`) writes to the database and drops the cache entry. Every process that reads the rule will see the new value within its next cache expiry cycle, without a restart.

`loaders.py` provides drop-in replacements for the hardcoded dict lookups: `get_alert_rule_dynamic(event_type_value)` and `limits_for_role_dynamic(role)` have the same return types as the originals and the same call sites. Day 22's `dispatch_alert()` and Day 20's `BudgetEnforcer` each change by one import and one function name. Every call site that already exists keeps working.

Day 18's `ABUSE_THRESHOLDS` (a list, not a dict) and Day 19's `AGENT_SECRET_SCOPES` require more design before going dynamic. Their list structure doesn't map as cleanly to a keyed policy table. They remain hardcoded for now, with a note in the schema pointing at where their future tables would sit.

<a id="what-youll-build"></a>
## What you will build today

- `backend/policies/policy_schema.py` - `PolicyTable(str, Enum)`, `PolicyRuleEntry(Base)` SQLAlchemy model, `PolicyUpdate` Pydantic request body.
- An Alembic migration creating `governance_policies` with a unique constraint on `(table_name, rule_key)`.
- `backend/policies/policy_store.py` - `get_policy()`, `list_policies()`, `set_policy()`, `reset_to_default()`, all sync DB operations with a module-level in-process cache.
- `backend/policies/seeders.py` - `seed_defaults()` that writes `ALERT_RULES` and `BUDGET_LIMITS` as JSON rows on first startup, idempotent.
- `backend/policies/loaders.py` - `get_alert_rule_dynamic()` and `limits_for_role_dynamic()`, drop-in replacements for the hardcoded dict lookups.
- One-line wiring changes in `dispatch_alert()` (Day 22) and `BudgetEnforcer` (Day 20/21).
- A new `AuditEventType.POLICY_UPDATED` member, emitted on every rule change.
- `backend/routers/policies.py` - `GET`, `PUT`, and `DELETE` endpoints for policy rules, admin-only via Day 21's `TenantContext`.
- Tests under `tests/policies/`.

<a id="step-251"></a>
## Step 25.1 - The policy schema and database model

```python
# backend/policies/policy_schema.py

from __future__ import annotations

import json
from datetime import datetime
from enum import Enum
from typing import Any

from pydantic import BaseModel
from sqlalchemy import (
    Boolean, Column, DateTime, Integer, String, Text,
    UniqueConstraint, func,
)

from backend.database.base import Base


class PolicyTable(str, Enum):
    ALERT_RULES   = "alert_rules"
    BUDGET_LIMITS = "budget_limits"
    # Future: ABUSE_THRESHOLDS = "abuse_thresholds"
    # Future: AGENT_SECRET_SCOPES = "agent_secret_scopes"


class PolicyRuleEntry(Base):
    """
    One row per governance rule. The combination of (table_name, rule_key)
    is unique: "alert_rules" / "audit.tamper_detected" is one row,
    "budget_limits" / "admin" is another.

    value_json holds the full rule as a JSON string. The schema of that
    JSON is defined by PolicyTable - alert_rules rows must have
    {severity, channels, dedup_window_seconds, description};
    budget_limits rows must have {daily_usd, monthly_usd, warn_ratio, description}.
    Validation happens in the router before any write reaches this table.
    """
    __tablename__ = "governance_policies"

    id           = Column(Integer, primary_key=True, autoincrement=True)
    table_name   = Column(String(64),  nullable=False, index=True)
    rule_key     = Column(String(128), nullable=False, index=True)
    value_json   = Column(Text, nullable=False)
    is_default   = Column(Boolean, default=True, nullable=False)
    version      = Column(Integer, default=1,  nullable=False)
    updated_by   = Column(String(128), nullable=True)
    updated_at   = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
    )
    notes        = Column(Text, nullable=True)

    __table_args__ = (
        UniqueConstraint("table_name", "rule_key", name="uq_policy_table_key"),
    )

    def as_dict(self) -> dict:
        return {
            "table_name":  self.table_name,
            "rule_key":    self.rule_key,
            "value":       json.loads(self.value_json),
            "is_default":  self.is_default,
            "version":     self.version,
            "updated_by":  self.updated_by,
            "updated_at":  self.updated_at.isoformat() if self.updated_at else None,
            "notes":       self.notes,
        }


class PolicyUpdate(BaseModel):
    value: dict[str, Any]
    notes: str | None = None
```

`value_json` is `Text`, not a JSONB column (which PostgreSQL supports but SQLite, the tutorial's default, doesn't). The router validates the JSON shape before writing, so the database doesn't need to enforce structure.

`version` starts at 1 for seeded defaults and increments on every `set_policy()` call. An audit trail of "who changed this rule and to what version" lives in the `audit_log` via `POLICY_UPDATED` events (Step 25.6), not in this table; `governance_policies` only holds the current state.

<a id="step-252"></a>
## Step 25.2 - Alembic migration for governance_policies

```python
# alembic/versions/0010_add_governance_policies.py

"""Add governance_policies table

Revision ID: 0010
Revises: 0009
"""

from alembic import op
import sqlalchemy as sa

revision = "0010"
down_revision = "0009"


def upgrade() -> None:
    op.create_table(
        "governance_policies",
        sa.Column("id",          sa.Integer(),     primary_key=True, autoincrement=True),
        sa.Column("table_name",  sa.String(64),    nullable=False),
        sa.Column("rule_key",    sa.String(128),   nullable=False),
        sa.Column("value_json",  sa.Text(),        nullable=False),
        sa.Column("is_default",  sa.Boolean(),     nullable=False, server_default="1"),
        sa.Column("version",     sa.Integer(),     nullable=False, server_default="1"),
        sa.Column("updated_by",  sa.String(128),   nullable=True),
        sa.Column("updated_at",  sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column("notes",       sa.Text(),        nullable=True),
    )
    op.create_index("ix_policy_table", "governance_policies", ["table_name"])
    op.create_index("ix_policy_key",   "governance_policies", ["rule_key"])
    op.create_unique_constraint(
        "uq_policy_table_key", "governance_policies", ["table_name", "rule_key"]
    )


def downgrade() -> None:
    op.drop_table("governance_policies")
```

```bash
alembic upgrade head
```

<a id="step-253"></a>
## Step 25.3 - The policy store: DB reads with an in-process cache

The cache is a module-level dict mapping `"{table}:{key}"` to `(value, expires_at)` where `expires_at` is a `time.monotonic()` timestamp. A 30-second TTL is short enough to pick up updates within one minute, long enough to keep the database quiet under normal request volumes. When `set_policy()` writes a new value it pops the entry immediately, so the very next read after a policy change sees the updated value without waiting.

```python
# backend/policies/policy_store.py

import json
import time
from typing import Any

from backend.database.session import SessionLocal
from backend.policies.policy_schema import PolicyRuleEntry

CACHE_TTL_SECONDS = 30

_cache: dict[str, tuple[Any, float]] = {}


def _ck(table: str, key: str) -> str:
    return f"{table}:{key}"


def get_policy(table: str, key: str) -> dict | None:
    ck = _ck(table, key)
    if ck in _cache:
        value, expires_at = _cache[ck]
        if time.monotonic() < expires_at:
            return value
        del _cache[ck]

    db = SessionLocal()
    try:
        row = (
            db.query(PolicyRuleEntry)
            .filter(
                PolicyRuleEntry.table_name == table,
                PolicyRuleEntry.rule_key == key,
            )
            .first()
        )
        if row is None:
            return None
        value = json.loads(row.value_json)
        _cache[ck] = (value, time.monotonic() + CACHE_TTL_SECONDS)
        return value
    finally:
        db.close()


def list_policies(table: str) -> list[dict]:
    db = SessionLocal()
    try:
        rows = (
            db.query(PolicyRuleEntry)
            .filter(PolicyRuleEntry.table_name == table)
            .order_by(PolicyRuleEntry.rule_key)
            .all()
        )
        return [r.as_dict() for r in rows]
    finally:
        db.close()


def set_policy(
    table:      str,
    key:        str,
    value:      dict,
    updated_by: str | None = None,
    notes:      str | None = None,
) -> dict:
    db = SessionLocal()
    try:
        row = (
            db.query(PolicyRuleEntry)
            .filter(
                PolicyRuleEntry.table_name == table,
                PolicyRuleEntry.rule_key == key,
            )
            .first()
        )
        if row:
            row.value_json  = json.dumps(value)
            row.is_default  = False
            row.version    += 1
            row.updated_by  = updated_by
            row.notes       = notes
        else:
            row = PolicyRuleEntry(
                table_name=table,
                rule_key=key,
                value_json=json.dumps(value),
                is_default=False,
                updated_by=updated_by,
                notes=notes,
            )
            db.add(row)
        db.commit()
        db.refresh(row)
    finally:
        db.close()

    _cache.pop(_ck(table, key), None)
    return value


def reset_to_default(table: str, key: str) -> None:
    """
    Delete the custom row so the next read returns None and the caller
    falls back to its hardcoded default. The seeder (Step 25.4) must be
    re-run to restore a seeded value, or the rule can be PUT again.
    """
    db = SessionLocal()
    try:
        db.query(PolicyRuleEntry).filter(
            PolicyRuleEntry.table_name == table,
            PolicyRuleEntry.rule_key == key,
        ).delete()
        db.commit()
    finally:
        db.close()
    _cache.pop(_ck(table, key), None)
```

Every path that reads from the database closes the session in a `finally` block. `set_policy()` pops the cache entry as the very last thing it does, after `db.commit()` succeeds, so a failed commit does not leave the cache and the database out of sync.

<a id="step-254"></a>
## Step 25.4 - Seeding defaults from the hardcoded dicts

`seed_defaults()` writes every entry from `ALERT_RULES` and `BUDGET_LIMITS` into `governance_policies` if no row for that key already exists. It's idempotent: running it twice doesn't change anything, and admin-modified rules are left untouched because the `INSERT OR IGNORE` / upsert only fires on missing keys.

```python
# backend/policies/seeders.py

import json

from backend.alerting.alert_schema import ALERT_RULES
from backend.billing.budget_schema import BUDGET_LIMITS
from backend.database.session import SessionLocal
from backend.policies.policy_schema import PolicyRuleEntry, PolicyTable


def _row_exists(db, table: str, key: str) -> bool:
    return (
        db.query(PolicyRuleEntry)
        .filter(
            PolicyRuleEntry.table_name == table,
            PolicyRuleEntry.rule_key == key,
        )
        .count()
        > 0
    )


def seed_defaults() -> dict[str, int]:
    """
    Insert default policy rules from the hardcoded dicts, skipping any
    row that already exists (so admin changes are preserved). Returns
    {table_name: rows_inserted}.
    """
    db = SessionLocal()
    counts: dict[str, int] = {}
    try:
        # --- alert_rules ---
        inserted = 0
        for event_type, rule in ALERT_RULES.items():
            key = event_type.value
            if not _row_exists(db, PolicyTable.ALERT_RULES.value, key):
                db.add(PolicyRuleEntry(
                    table_name  = PolicyTable.ALERT_RULES.value,
                    rule_key    = key,
                    value_json  = json.dumps({
                        "severity":             rule.severity.value,
                        "channels":             [c.value for c in rule.channels],
                        "dedup_window_seconds": rule.dedup_window_seconds,
                        "description":          rule.description,
                    }),
                    is_default=True,
                ))
                inserted += 1
        counts[PolicyTable.ALERT_RULES.value] = inserted

        # --- budget_limits ---
        inserted = 0
        for role, limits in BUDGET_LIMITS.items():
            if not _row_exists(db, PolicyTable.BUDGET_LIMITS.value, role):
                db.add(PolicyRuleEntry(
                    table_name  = PolicyTable.BUDGET_LIMITS.value,
                    rule_key    = role,
                    value_json  = json.dumps({
                        "daily_usd":   limits.daily_usd,
                        "monthly_usd": limits.monthly_usd,
                        "warn_ratio":  limits.warn_ratio,
                        "description": limits.description,
                    }),
                    is_default=True,
                ))
                inserted += 1
        counts[PolicyTable.BUDGET_LIMITS.value] = inserted

        db.commit()
    finally:
        db.close()

    return counts
```

Call this from the application startup sequence:

```python
# backend/main.py (inside the startup event, after alembic upgrade)

from backend.policies.seeders import seed_defaults

@app.on_event("startup")
async def startup() -> None:
    seed_defaults()
    # ... other startup tasks ...
```

`seed_defaults()` runs synchronously at startup. On a database with 50 governance rules seeded for the first time it takes under 100ms, fast enough to not matter during startup. Running it on every request would add unnecessary latency.

<a id="step-255"></a>
## Step 25.5 - Drop-in loader replacements

The hardcoded lookup functions (`get_alert_rule(event_type)` from `alert_schema.py` and `limits_for_role(role)` from `budget_schema.py`) each become a DB-backed alternative in `loaders.py`. The return types are identical: an `AlertRule | None` and a `BudgetLimits`. Every call site that used the originals can switch by changing one import line.

```python
# backend/policies/loaders.py

from backend.alerting.alert_schema import AlertChannel, AlertRule, AlertSeverity
from backend.billing.budget_schema import BUDGET_LIMITS, BudgetLimits
from backend.policies.policy_schema import PolicyTable
from backend.policies.policy_store import get_policy

_FALLBACK_BUDGET = BudgetLimits(
    daily_usd=0.50, monthly_usd=5.00, warn_ratio=0.8,
    description="Emergency fallback - DB read failed or role is unknown",
)


def get_alert_rule_dynamic(event_type_value: str) -> AlertRule | None:
    """
    Reads the alert rule for event_type_value from the policy store.
    Returns None if no rule exists (same contract as Day 22's get_alert_rule()).
    Falls back to the hardcoded ALERT_RULES dict if the DB has no row yet
    (i.e., seed_defaults() hasn't run in this environment).
    """
    data = get_policy(PolicyTable.ALERT_RULES.value, event_type_value)
    if data is not None:
        return AlertRule(
            severity=AlertSeverity(data["severity"]),
            channels=tuple(AlertChannel(c) for c in data["channels"]),
            dedup_window_seconds=data["dedup_window_seconds"],
            description=data["description"],
        )

    # Hardcoded fallback for environments where seed_defaults() hasn't run.
    from backend.alerting.alert_schema import ALERT_RULES, AuditEventType
    try:
        return ALERT_RULES.get(AuditEventType(event_type_value))
    except ValueError:
        return None


def limits_for_role_dynamic(role: str) -> BudgetLimits:
    """
    Reads the budget limits for role from the policy store.
    Falls back to the "unknown" tier if no row exists for this role,
    and to _FALLBACK_BUDGET if even that is missing.
    """
    data = get_policy(PolicyTable.BUDGET_LIMITS.value, role.lower())
    if data is None:
        data = get_policy(PolicyTable.BUDGET_LIMITS.value, "unknown")
    if data is None:
        # Last resort: read the hardcoded dict so startup keeps working
        # before seed_defaults() has been called.
        return BUDGET_LIMITS.get(role.lower(), _FALLBACK_BUDGET)

    return BudgetLimits(
        daily_usd=data["daily_usd"],
        monthly_usd=data["monthly_usd"],
        warn_ratio=data["warn_ratio"],
        description=data["description"],
    )
```

The two-level fallback matters during local development: if a developer runs the service without migrating the database, the hardcoded defaults keep everything working instead of producing a 500 every time an LLM call is made.

<a id="step-256"></a>
## Step 25.6 - Wiring the loaders and auditing policy changes

Two files change. Each change is one import and one function name. Nothing else.

```python
# backend/alerting/dispatcher.py (Day 22 - update the rule lookup)

# BEFORE (Day 22):
from backend.alerting.alert_schema import get_alert_rule
# ...
rule = get_alert_rule(event_type)

# AFTER (Day 25):
from backend.policies.loaders import get_alert_rule_dynamic
# ...
rule = get_alert_rule_dynamic(entry.event_type)
```

```python
# backend/billing/budget_enforcer.py (Day 20/21 - update the limits lookup)

# BEFORE (Day 20):
from backend.billing.budget_schema import limits_for_role
# ...
limits = limits_for_role(user_role)

# AFTER (Day 25):
from backend.policies.loaders import limits_for_role_dynamic as limits_for_role
# ...
limits = limits_for_role(user_role)   # call site unchanged
```

Every other call site that imports from `budget_schema` or `alert_schema` still works; no other file needs touching. For existing tests that monkeypatch `ALERT_RULES` or `BUDGET_LIMITS` directly, add a `policy_store._cache.clear()` call in the test setup or monkeypatch `get_policy` instead.

For the audit trail, `POLICY_UPDATED` is the new event type:

```python
# backend/audit/audit_schema.py (addition to AuditEventType)

    # Policy-as-code (new - Day 25)
    POLICY_UPDATED = "policy.updated"
```

Every `PUT /policies/{table}/{key}` request calls `_emit()` with this type. The payload carries `table`, `key`, and the full new `value` so a compliance report (Day 24) can show exactly what changed and when, signed by the audit chain hash.

<a id="step-257"></a>
## Step 25.7 - The policies admin router

```python
# backend/routers/policies.py

from fastapi import APIRouter, Depends, HTTPException, Path

from backend.audit.audit_logger import _emit
from backend.audit.audit_schema import AuditEvent, AuditEventType
from backend.policies.loaders import get_alert_rule_dynamic
from backend.policies.policy_schema import PolicyTable, PolicyUpdate
from backend.policies.policy_store import (
    get_policy, list_policies, reset_to_default, set_policy,
)
from backend.tenancy.context import TenantContext, get_tenant_context

router = APIRouter(prefix="/policies", tags=["Policies"])

_ALERT_RULE_KEYS   = {"severity", "channels", "dedup_window_seconds", "description"}
_BUDGET_LIMIT_KEYS = {"daily_usd", "monthly_usd", "warn_ratio", "description"}


def _require_admin(ctx: TenantContext) -> None:
    if ctx.user_role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required")


def _validate_value(table: PolicyTable, value: dict) -> None:
    """Reject writes that would produce an unreadable rule."""
    if table == PolicyTable.ALERT_RULES:
        missing = _ALERT_RULE_KEYS - value.keys()
        if missing:
            raise HTTPException(status_code=422, detail=f"alert_rule missing fields: {missing}")
        if value.get("severity") not in ("info", "warning", "critical"):
            raise HTTPException(status_code=422, detail="severity must be info, warning, or critical")
        if not isinstance(value.get("channels"), list):
            raise HTTPException(status_code=422, detail="channels must be a list")
        if not isinstance(value.get("dedup_window_seconds"), int):
            raise HTTPException(status_code=422, detail="dedup_window_seconds must be an integer")

    elif table == PolicyTable.BUDGET_LIMITS:
        missing = _BUDGET_LIMIT_KEYS - value.keys()
        if missing:
            raise HTTPException(status_code=422, detail=f"budget_limit missing fields: {missing}")
        for field in ("daily_usd", "monthly_usd", "warn_ratio"):
            if not isinstance(value.get(field), (int, float)):
                raise HTTPException(status_code=422, detail=f"{field} must be a number")
        if not (0 < value["warn_ratio"] < 1):
            raise HTTPException(status_code=422, detail="warn_ratio must be between 0 and 1")


@router.get("/{table}")
def list_rules(
    table: PolicyTable,
    ctx: TenantContext = Depends(get_tenant_context),
):
    _require_admin(ctx)
    return {"table": table.value, "rules": list_policies(table.value)}


@router.get("/{table}/{key}")
def get_rule(
    table: PolicyTable,
    key:   str,
    ctx:   TenantContext = Depends(get_tenant_context),
):
    _require_admin(ctx)
    value = get_policy(table.value, key)
    if value is None:
        raise HTTPException(status_code=404, detail=f"No rule for {table.value}/{key}")
    return {"table": table.value, "key": key, "value": value}


@router.put("/{table}/{key}")
def update_rule(
    table: PolicyTable,
    key:   str,
    body:  PolicyUpdate,
    ctx:   TenantContext = Depends(get_tenant_context),
):
    _require_admin(ctx)
    _validate_value(table, body.value)

    set_policy(table.value, key, body.value, updated_by=ctx.user_id, notes=body.notes)

    _emit(AuditEvent(
        event_type=AuditEventType.POLICY_UPDATED,
        org_id=ctx.org_id,
        actor_id=ctx.user_id,
        actor_role=ctx.user_role,
        outcome="updated",
        reason=body.notes,
        payload={"table": table.value, "key": key, "value": body.value},
    ))

    return {"status": "updated", "table": table.value, "key": key, "version": "incremented"}


@router.delete("/{table}/{key}")
def reset_rule(
    table: PolicyTable,
    key:   str,
    ctx:   TenantContext = Depends(get_tenant_context),
):
    _require_admin(ctx)
    existing = get_policy(table.value, key)
    if existing is None:
        raise HTTPException(status_code=404, detail=f"No rule for {table.value}/{key}")

    reset_to_default(table.value, key)

    _emit(AuditEvent(
        event_type=AuditEventType.POLICY_UPDATED,
        org_id=ctx.org_id,
        actor_id=ctx.user_id,
        actor_role=ctx.user_role,
        outcome="reset",
        reason="reset to default",
        payload={"table": table.value, "key": key},
    ))

    return {"status": "reset", "table": table.value, "key": key}
```

```python
# backend/main.py (add alongside other routers)
from backend.routers import policies

app.include_router(policies.router)
```

`_validate_value` runs before any database write. An admin who posts `{"severity": "hypercritical"}` or a `budget_limits` row missing `monthly_usd` gets a 422 before anything touches the database or the cache.

<a id="step-258"></a>
## Step 25.8 - Tests

```python
# tests/policies/test_policy_store.py

from backend.policies import policy_store
from backend.policies.policy_store import (
    get_policy, list_policies, reset_to_default, set_policy,
)
from backend.policies.policy_schema import PolicyTable


class TestPolicyStore:
    def setup_method(self):
        policy_store._cache.clear()

    def test_get_policy_returns_none_for_missing_key(self):
        result = get_policy(PolicyTable.ALERT_RULES.value, "event.that.does.not.exist")
        assert result is None

    def test_set_and_get_roundtrip(self):
        value = {"daily_usd": 99.0, "monthly_usd": 500.0, "warn_ratio": 0.75, "description": "test"}
        set_policy(PolicyTable.BUDGET_LIMITS.value, "test_role", value)
        result = get_policy(PolicyTable.BUDGET_LIMITS.value, "test_role")
        assert result == value

    def test_set_invalidates_cache(self):
        set_policy(PolicyTable.BUDGET_LIMITS.value, "cache_test", {"x": 1, "daily_usd": 1, "monthly_usd": 10, "warn_ratio": 0.5, "description": ""})
        get_policy(PolicyTable.BUDGET_LIMITS.value, "cache_test")  # populate cache

        set_policy(PolicyTable.BUDGET_LIMITS.value, "cache_test", {"x": 2, "daily_usd": 2, "monthly_usd": 20, "warn_ratio": 0.5, "description": ""})
        result = get_policy(PolicyTable.BUDGET_LIMITS.value, "cache_test")

        assert result["daily_usd"] == 2  # cache was busted, fresh DB read

    def test_reset_removes_row_and_returns_none(self):
        set_policy(PolicyTable.ALERT_RULES.value, "test.event", {
            "severity": "info", "channels": ["slack"],
            "dedup_window_seconds": 60, "description": "test",
        })
        assert get_policy(PolicyTable.ALERT_RULES.value, "test.event") is not None

        reset_to_default(PolicyTable.ALERT_RULES.value, "test.event")
        assert get_policy(PolicyTable.ALERT_RULES.value, "test.event") is None

    def test_list_policies_returns_all_rows_for_table(self):
        set_policy(PolicyTable.BUDGET_LIMITS.value, "list_test_a", {"daily_usd": 1, "monthly_usd": 10, "warn_ratio": 0.5, "description": ""})
        set_policy(PolicyTable.BUDGET_LIMITS.value, "list_test_b", {"daily_usd": 2, "monthly_usd": 20, "warn_ratio": 0.5, "description": ""})
        rules = list_policies(PolicyTable.BUDGET_LIMITS.value)
        keys = [r["rule_key"] for r in rules]
        assert "list_test_a" in keys
        assert "list_test_b" in keys
```

```python
# tests/policies/test_loaders.py

from backend.policies import policy_store
from backend.policies.loaders import get_alert_rule_dynamic, limits_for_role_dynamic
from backend.policies.policy_schema import PolicyTable
from backend.policies.policy_store import set_policy
from backend.alerting.alert_schema import AlertSeverity


class TestLoaders:
    def setup_method(self):
        policy_store._cache.clear()

    def test_get_alert_rule_dynamic_reads_from_db(self):
        set_policy(PolicyTable.ALERT_RULES.value, "test.alert.event", {
            "severity": "warning",
            "channels": ["slack"],
            "dedup_window_seconds": 120,
            "description": "test rule",
        })

        rule = get_alert_rule_dynamic("test.alert.event")
        assert rule is not None
        assert rule.severity == AlertSeverity.WARNING
        assert rule.dedup_window_seconds == 120

    def test_limits_for_role_dynamic_reads_from_db(self):
        set_policy(PolicyTable.BUDGET_LIMITS.value, "test_engineer", {
            "daily_usd": 42.0,
            "monthly_usd": 500.0,
            "warn_ratio": 0.7,
            "description": "test",
        })

        limits = limits_for_role_dynamic("test_engineer")
        assert limits.daily_usd == 42.0
        assert limits.warn_ratio == 0.7

    def test_limits_for_role_dynamic_falls_back_to_unknown(self):
        limits = limits_for_role_dynamic("nonexistent_role_xyz")
        # Should not raise; returns unknown-tier or hardcoded fallback
        assert limits.daily_usd > 0


class TestPoliciesRouter:
    from fastapi.testclient import TestClient
    from backend.main import app

    client = TestClient(app)
    ADMIN_HEADERS = {"X-Org-ID": "acme", "X-User-ID": "admin1", "X-User-Role": "admin"}

    def test_get_rules_requires_admin(self):
        resp = self.client.get(
            "/policies/alert_rules",
            headers={"X-Org-ID": "acme", "X-User-ID": "alice", "X-User-Role": "employee"},
        )
        assert resp.status_code == 403

    def test_put_rule_validates_shape(self):
        resp = self.client.put(
            "/policies/alert_rules/test.shape",
            json={"value": {"severity": "invalid_level", "channels": [], "dedup_window_seconds": 0, "description": "x"}},
            headers=self.ADMIN_HEADERS,
        )
        assert resp.status_code == 422

    def test_put_rule_updates_and_is_readable(self):
        new_rule = {
            "severity": "critical",
            "channels": ["slack"],
            "dedup_window_seconds": 0,
            "description": "router test",
        }
        put_resp = self.client.put(
            "/policies/alert_rules/router.test.event",
            json={"value": new_rule, "notes": "test run"},
            headers=self.ADMIN_HEADERS,
        )
        assert put_resp.status_code == 200

        get_resp = self.client.get(
            "/policies/alert_rules/router.test.event",
            headers=self.ADMIN_HEADERS,
        )
        assert get_resp.json()["value"]["severity"] == "critical"

    def test_put_rule_emits_policy_updated_audit_event(self):
        from backend.audit.audit_logger import get_audit_events
        from backend.audit.audit_schema import AuditEventType

        self.client.put(
            "/policies/budget_limits/audit_test_role",
            json={"value": {"daily_usd": 5, "monthly_usd": 50, "warn_ratio": 0.8, "description": "x"}, "notes": "audit test"},
            headers=self.ADMIN_HEADERS,
        )

        events = get_audit_events("acme", event_type=AuditEventType.POLICY_UPDATED.value, limit=10)
        matching = [e for e in events if e.payload.get("key") == "audit_test_role"]
        assert matching, "Expected a POLICY_UPDATED event for audit_test_role"
```

Run them:

```bash
pytest tests/policies/ -v
```

<a id="what-you-built"></a>
## What you built today

A `governance_policies` table (keyed by `table_name` and `rule_key`, with `value_json`, `is_default`, `version`, `updated_by`, and `notes`) backed by a migration and a `PolicyStore` with a 30-second in-process cache that invalidates on write. A `seed_defaults()` function that walks `ALERT_RULES` and `BUDGET_LIMITS` and inserts them as JSON rows on first startup without touching any row that already exists. Two loader functions (`get_alert_rule_dynamic`, `limits_for_role_dynamic`) that read from the store and fall back to the hardcoded dicts in environments where the database hasn't been seeded yet. One-line wiring changes in `dispatch_alert()` and `BudgetEnforcer` that switch both to the dynamic path. A new `AuditEventType.POLICY_UPDATED` member emitted on every PUT or DELETE to the policies API, carrying the full new value in its payload for compliance reporting (Day 24). And a `GET`/`PUT`/`DELETE` admin API under `/policies/{table}/{key}` with input validation (shape-checking before any DB write), admin-only enforcement, and cache invalidation on every successful write.

<a id="checklist"></a>
## Day 25 checklist

- [ ] `governance_policies` table exists in the database with `(table_name, rule_key)` unique constraint, `version` integer, `is_default` boolean, and `notes` text; migration `0010` runs via `alembic upgrade head`
- [ ] `policy_store.py`'s `_cache` is a module-level dict; `get_policy()` checks it first, falls back to DB, and writes back to cache on a hit; `set_policy()` pops the entry immediately after a successful DB commit
- [ ] `seed_defaults()` inserts rows for all keys in `ALERT_RULES` and `BUDGET_LIMITS` with `is_default=True`, skips any key that already has a row, and is called from the FastAPI startup event
- [ ] `get_alert_rule_dynamic(event_type_value)` returns an `AlertRule` built from the DB value, or falls back to the hardcoded `ALERT_RULES` dict if no row exists
- [ ] `limits_for_role_dynamic(role)` returns a `BudgetLimits` from DB, falls back to the "unknown" row, then falls back to `_FALLBACK_BUDGET`; it never raises for an unknown role
- [ ] `dispatch_alert()` (Day 22) calls `get_alert_rule_dynamic(entry.event_type)` instead of `get_alert_rule(AuditEventType(entry.event_type))`
- [ ] `BudgetEnforcer` (Day 20/21) calls `limits_for_role_dynamic(user_role)` instead of `limits_for_role(user_role)`
- [ ] `AuditEventType.POLICY_UPDATED = "policy.updated"` is defined in `audit_schema.py`
- [ ] `PUT /policies/{table}/{key}` validates the JSON shape before writing, increments `version`, sets `is_default=False`, calls `_emit()` with `POLICY_UPDATED`, and returns 200
- [ ] `DELETE /policies/{table}/{key}` removes the row (no fallback re-seeding), emits `POLICY_UPDATED` with `outcome="reset"`, and returns 200; a subsequent GET for that key returns 404
- [ ] A non-admin request to any `/policies/*` endpoint returns 403
- [ ] All tests in `tests/policies/` pass

---

[Back to repo](https://github.com/GuntruTirupathamma/AI-Governance) - Previous: [DAY24.md](./DAY24.md) - Next: DAY26.md - Incident Response Playbooks and Automated Remediation
