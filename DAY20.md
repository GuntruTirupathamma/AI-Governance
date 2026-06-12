---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 20 of 35
Previous: DAY19.md - Secrets Management and Credential Isolation
Next: DAY21.md - Multi-Tenant Isolation and Data Segregation
---

# Day 20: Cost Governance and Budget Enforcement

## Table of contents

- [The problem with rate limits and no budgets](#the-problem)
- [What cost governance means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 20.1 - Per-role budget limits](#step-201)
- [Step 20.2 - Float-precision cost counters in Redis](#step-202)
- [Step 20.3 - The budget tracker](#step-203)
- [Step 20.4 - The budget enforcer](#step-204)
- [Step 20.5 - New audit events for budget warnings and blocks](#step-205)
- [Step 20.6 - Enforcing budgets in the streaming flow](#step-206)
- [Step 20.7 - Feeding budget events into abuse detection](#step-207)
- [Step 20.8 - Spend visibility: the billing router](#step-208)
- [Step 20.9 - Tests](#step-209)
- [What you built today](#what-you-built)
- [Day 20 checklist](#checklist)

<a id="the-problem"></a>
## The problem with rate limits and no budgets

Day 4 built `ROLE_LIMITS`: a table of how many requests, policy checks, and LLM calls each role can make per hour. Day 9 built governed streaming on top of that, and its own docstring lists "Pre-stream governance (policy, model, budget)" as a responsibility - but if you go looking for the budget check, it isn't there. What Day 9 actually shipped were two Redis counters, `cost:user:{user_id}:{date}` and `cost:model:{model_id}:{date}`, both incremented by 1 on every completed stream. They count *how many billed streams happened*, not *how many dollars were spent*. Nothing reads them, and nothing acts on them.

That gap matters because rate limits and dollar costs aren't the same axis. Two `employee`-role users both under their 100-calls-per-hour limit can have wildly different bills: one asks short questions against `gpt-3.5-turbo`, the other pastes 20,000-token documents into `gpt-4-turbo` with a 2048-token completion every time. Both are "allowed" by Day 4's rules. Only one of them is a budget problem.

Day 18's abuse detector doesn't help here either - by design, it watches for *behavioral* signals (injection attempts, privilege escalation, repeated blocks). A user running large-but-legitimate prompts through an expensive model isn't abusing anything. They're just spending money, and currently nothing in this system can tell you how much, in real time, or stop it once a limit is reached.

Day 20 closes that gap: real per-user dollar totals, updated after every stream with the actual cost Day 9 already computes (`estimate_cost()`), checked against per-role daily and monthly caps *before* the next stream starts.

<a id="what-it-means-here"></a>
## What cost governance means here

- **Budgets are a table, same shape as `ROLE_LIMITS`.** `BUDGET_LIMITS` maps role to a daily cap, a monthly cap, and a warn threshold. Reviewing or changing a role's spending limit is a one-line diff in one file, same as Day 19's `AGENT_SECRET_SCOPES` and Day 4's `ROLE_LIMITS`.

- **The numbers are real dollars, not call counts.** `estimate_cost()` already runs at the end of every stream (Day 9). Day 20 takes that number and accumulates it in Redis with `INCRBYFLOAT`, so `budget:user:{user_id}:{date}` is an actual running total in USD - not a proxy for one.

- **Two horizons, checked in order of how hard they are to recover from.** A daily cap resets at UTC midnight - worst case, a blocked user waits a few hours. A monthly cap that's already exceeded stays exceeded for the rest of the month. The enforcer checks monthly first: if that's blown, the daily number doesn't matter.

- **A warning before a block.** `warn_ratio` (typically 0.8) means a user crosses into "you're close" territory before they hit the wall. The stream still completes - this is a heads-up, logged to the audit trail, not an enforcement action.

- **Blocking happens before the model is called.** If the budget check fails, `GovernedStreamManager` never calls the LLM - no tokens are generated, no cost is incurred for a stream that was going to be rejected anyway. This mirrors Day 9's policy and model-registry checks, which also short-circuit before any tokens flow.

- **Budget exhaustion isn't an abuse signal by itself - repetition is.** One blocked stream because you're out of budget today is just... you're out of budget today. A user (or a misbehaving script) hammering the endpoint after every block is a different problem, and Day 18's abuse detector already knows how to handle "the same actor keeps tripping the same wire" - Day 20 just gives it a new wire to watch.

By the end of today, a stream request from a user who's exhausted their daily or monthly budget gets a single `StreamBudgetExceededEvent` and nothing else - no policy check, no model call, no tokens billed - and an admin (or the user themselves) can see exactly how much of which budget has been spent via a new `/billing/usage` endpoint.

<a id="what-youll-build"></a>
## What you will build today

```
backend/
├── billing/
│   ├── __init__.py
│   ├── budget_schema.py      # BudgetLimits, BUDGET_LIMITS, limits_for_role()
│   ├── budget_tracker.py     # record_cost(), get_spend(), BudgetSpend
│   └── budget_enforcer.py    # BudgetEnforcer.check() -> BudgetDecision
├── cache/
│   └── redis_client.py        # (Day 4, extended) redis_incrbyfloat(), redis_get_float()
├── audit/
│   ├── audit_schema.py        # (Day 17, extended) BUDGET_WARNING_ISSUED, BUDGET_EXCEEDED_BLOCKED
│   └── audit_logger.py        # (Day 17/18, extended) budget_warning(), budget_exceeded()
├── security/
│   └── abuse_schema.py        # (Day 18, extended) ABUSE_THRESHOLDS for BUDGET_EXCEEDED_BLOCKED
├── llm/
│   └── governed_client.py     # (Day 9, extended) budget check + StreamBudgetExceededEvent
└── routers/
    └── billing.py              # NEW - GET /billing/usage, GET /billing/usage/{user_id}

tests/
├── billing/
│   ├── test_budget_tracker.py
│   ├── test_budget_enforcer.py
│   └── test_redis_float_counters.py
└── llm/
    └── test_governed_client_budget.py
```

<a id="step-201"></a>
## Step 20.1 - Per-role budget limits

Same role names as Day 4's `ROLE_LIMITS`, same one-table-says-everything shape as Day 19's `AGENT_SECRET_SCOPES`. Tightening a role's budget after a postmortem is a one-line change here.

```python
# backend/billing/budget_schema.py
"""
Per-role spend limits.

ROLE_LIMITS (Day 4) caps how OFTEN a role can call the model.
BUDGET_LIMITS caps how MUCH it can spend doing so. Two users on the same
rate limit can run up very different bills depending on prompt and
response size - rate limiting alone doesn't catch that, which is the
gap this file exists to close.

Same shape as ROLE_LIMITS and Day 19's AGENT_SECRET_SCOPES: one small,
reviewable table. Changing a role's budget is a one-line diff here, and
that diff is the entire change - no enforcement code needs to move.
"""
from dataclasses import dataclass


@dataclass(frozen=True)
class BudgetLimits:
    daily_usd:   float   # hard cap, resets at UTC midnight
    monthly_usd: float   # hard cap, resets on the 1st of the month (UTC)
    warn_ratio:  float   # fraction of daily_usd that triggers a warning
    description: str


# Design principle, same as Day 4: err on the side of restriction,
# loosen after review. A role that's blocked too often is a 5-minute
# config change; a role that overspent for a month is not.
BUDGET_LIMITS: dict[str, BudgetLimits] = {

    "admin": BudgetLimits(
        daily_usd=200.00, monthly_usd=3_000.00, warn_ratio=0.8,
        description="Full access. Admins run broad diagnostics and incident response.",
    ),

    "analyst": BudgetLimits(
        daily_usd=50.00, monthly_usd=750.00, warn_ratio=0.8,
        description="Power users with broad read/analyze access.",
    ),

    "employee": BudgetLimits(
        daily_usd=10.00, monthly_usd=150.00, warn_ratio=0.8,
        description="Standard employee access. Default role.",
    ),

    "intern": BudgetLimits(
        daily_usd=3.00, monthly_usd=45.00, warn_ratio=0.7,
        description="Limited access. Intern/trainee role.",
    ),

    "guest": BudgetLimits(
        daily_usd=1.00, monthly_usd=15.00, warn_ratio=0.5,
        description="Minimal read-only access. External users.",
    ),

    "service": BudgetLimits(
        daily_usd=500.00, monthly_usd=10_000.00, warn_ratio=0.9,
        description="Machine-to-machine. Internal services calling the API.",
    ),

    # Default: unknown roles treated like guests.
    "unknown": BudgetLimits(
        daily_usd=1.00, monthly_usd=15.00, warn_ratio=0.5,
        description="Unknown/unrecognized role. Minimal access.",
    ),
}


def limits_for_role(role: str) -> BudgetLimits:
    return BUDGET_LIMITS.get(role.lower(), BUDGET_LIMITS["unknown"])
```

<a id="step-202"></a>
## Step 20.2 - Float-precision cost counters in Redis

Day 4's `redis_incr()` wraps Redis's `INCR`, which is integer-only - fine for "count of requests," not fine for "$0.0041 per stream." Redis has had `INCRBYFLOAT` since 2.6, and it's exactly what's needed: an atomic, race-free running total in dollars.

```python
# backend/cache/redis_client.py (additions)

async def redis_incrbyfloat(key: str, amount: float, ttl_seconds: int = 3600) -> float:
    """
    Atomically add `amount` to a float counter.

    Sets a TTL only the first time the key is created - INCRBYFLOAT
    creates a key with no expiry if it doesn't exist yet, and we don't
    want every subsequent charge to push the expiry back out (that
    would make a "resets daily" counter never actually reset).

    Returns the new total, or 0.0 on error - a failed cost write should
    never be the thing that crashes a stream.
    """
    try:
        r = await get_redis()
        if not r:
            return 0.0
        new_total = await r.incrbyfloat(key, amount)
        if float(new_total) == amount:
            # This INCRBYFLOAT created the key - set its expiry now.
            await r.expire(key, ttl_seconds)
        return float(new_total)
    except Exception as e:
        logger.warning(f"Redis INCRBYFLOAT failed for key={key}: {e}")
        return 0.0


async def redis_get_float(key: str) -> float:
    """Get a float counter's current value. Returns 0.0 on miss or error."""
    try:
        r = await get_redis()
        if not r:
            return 0.0
        value = await r.get(key)
        return float(value) if value else 0.0
    except Exception as e:
        logger.warning(f"Redis GET (float) failed for key={key}: {e}")
        return 0.0
```

The "only set TTL on creation" detail is the one easy way to get this subtly wrong: if every `INCRBYFLOAT` also called `EXPIRE`, a user who streams constantly would keep pushing their daily counter's expiry forward and it would never roll over at midnight. Checking `new_total == amount` is a cheap way to detect "this call created the key" without a separate `EXISTS` round-trip.

<a id="step-203"></a>
## Step 20.3 - The budget tracker

Two keys per user: one for today, one for this month. Both float totals in USD, maintained with Step 20.2's helpers.

```python
# backend/billing/budget_tracker.py
"""
Reads and writes the running cost totals that budget enforcement checks
against.

Two counters per user, both float dollar amounts maintained with
INCRBYFLOAT:

  budget:user:{user_id}:{YYYY-MM-DD}   - resets daily  (TTL ~26h)
  budget:user:{user_id}:{YYYY-MM}      - resets monthly (TTL ~32 days)

These are separate from Day 9's cost:user:{user_id}:{date} and
cost:model:{model_id}:{date} counters, which count *stream completions*
(incremented by 1 each) rather than dollars. Both stay - one answers
"how many billed streams today", the other "how many dollars today".
"""
from dataclasses import dataclass
from datetime import datetime, timezone

from backend.cache.redis_client import redis_incrbyfloat, redis_get_float

# A little over a day/month, so a slightly-late write near a boundary
# still lands inside the window it belongs to.
DAILY_TTL_SECONDS   = 26 * 3600
MONTHLY_TTL_SECONDS = 32 * 24 * 3600


def _daily_key(user_id: str, when: datetime | None = None) -> str:
    when = when or datetime.now(timezone.utc)
    return f"budget:user:{user_id}:{when.strftime('%Y-%m-%d')}"


def _monthly_key(user_id: str, when: datetime | None = None) -> str:
    when = when or datetime.now(timezone.utc)
    return f"budget:user:{user_id}:{when.strftime('%Y-%m')}"


@dataclass
class BudgetSpend:
    daily_usd:   float
    monthly_usd: float


async def record_cost(user_id: str, cost_usd: float) -> BudgetSpend:
    """
    Add `cost_usd` to both the daily and monthly running totals for
    `user_id`. Called once per stream, after the actual cost is known -
    including retracted and disconnected streams, since tokens were
    generated (and billed by the model provider) either way.
    """
    if cost_usd <= 0:
        return await get_spend(user_id)

    daily   = await redis_incrbyfloat(_daily_key(user_id),   cost_usd, DAILY_TTL_SECONDS)
    monthly = await redis_incrbyfloat(_monthly_key(user_id), cost_usd, MONTHLY_TTL_SECONDS)
    return BudgetSpend(daily_usd=daily, monthly_usd=monthly)


async def get_spend(user_id: str) -> BudgetSpend:
    """Current running totals, without modifying them."""
    daily   = await redis_get_float(_daily_key(user_id))
    monthly = await redis_get_float(_monthly_key(user_id))
    return BudgetSpend(daily_usd=daily, monthly_usd=monthly)
```

<a id="step-204"></a>
## Step 20.4 - The budget enforcer

The tracker supplies numbers; the enforcer turns them into a decision. Three outcomes: `OK` (proceed silently), `WARN` (proceed, but log it), `BLOCK` (do not call the model).

```python
# backend/billing/budget_enforcer.py
"""
Budget enforcer.

Checked once per stream, before anything else happens (Step 20.6).
Given a user's current spend and their role's limits, decides whether
the stream may proceed and whether a warning should be recorded.

budget_tracker (Step 20.3) supplies the numbers. This module makes the
decision. The audit logger (Step 20.5) records the outcome.
"""
from dataclasses import dataclass
from enum import Enum

from backend.billing.budget_schema import BudgetLimits, limits_for_role
from backend.billing.budget_tracker import BudgetSpend, get_spend


class BudgetAction(str, Enum):
    OK    = "ok"     # under the warn threshold - proceed silently
    WARN  = "warn"   # over the warn threshold, under the hard cap - proceed, log a warning
    BLOCK = "block"  # at or over a hard cap - do not call the model


@dataclass
class BudgetDecision:
    action: BudgetAction
    period: str          # "daily" or "monthly" - which limit drove this decision
    spend:  BudgetSpend
    limits: BudgetLimits
    reason: str


class BudgetEnforcer:

    async def check(self, user_id: str, role: str) -> BudgetDecision:
        """
        Evaluate `user_id`'s current spend against `role`'s limits.

        Checks the monthly cap first. It's the harder constraint to
        recover from - a daily cap resets within 24 hours, but a blown
        monthly cap stays blown for the rest of the month, so if both
        are exceeded the monthly one is the more useful thing to report.
        """
        limits = limits_for_role(role)
        spend  = await get_spend(user_id)

        if spend.monthly_usd >= limits.monthly_usd:
            return BudgetDecision(
                action=BudgetAction.BLOCK, period="monthly", spend=spend, limits=limits,
                reason=(
                    f"Monthly budget of ${limits.monthly_usd:.2f} reached "
                    f"(spent ${spend.monthly_usd:.2f})."
                ),
            )

        if spend.daily_usd >= limits.daily_usd:
            return BudgetDecision(
                action=BudgetAction.BLOCK, period="daily", spend=spend, limits=limits,
                reason=(
                    f"Daily budget of ${limits.daily_usd:.2f} reached "
                    f"(spent ${spend.daily_usd:.2f})."
                ),
            )

        warn_threshold = limits.daily_usd * limits.warn_ratio
        if spend.daily_usd >= warn_threshold:
            return BudgetDecision(
                action=BudgetAction.WARN, period="daily", spend=spend, limits=limits,
                reason=(
                    f"At ${spend.daily_usd:.2f} of ${limits.daily_usd:.2f} daily budget "
                    f"({limits.warn_ratio:.0%} threshold)."
                ),
            )

        return BudgetDecision(
            action=BudgetAction.OK, period="", spend=spend, limits=limits, reason="",
        )
```

<a id="step-205"></a>
## Step 20.5 - New audit events for budget warnings and blocks

Two new event types, following the Day 17 pattern, both flowing through Day 18's `_emit()`.

```python
# backend/audit/audit_schema.py (additions to AuditEventType)

class AuditEventType(str, Enum):
    # ... existing event types from Days 14-19 ...

    # Cost governance (Day 20)
    BUDGET_WARNING_ISSUED   = "budget.warning_issued"
    BUDGET_EXCEEDED_BLOCKED = "budget.exceeded_blocked"
```

```python
# backend/audit/audit_logger.py (additions)

class AuditLogger:
    # ... existing methods from Days 14-19 ...

    def budget_warning(
        self, user_id: str, user_role: str, session_id: str | None,
        period: str, spend_usd: float, limit_usd: float,
    ) -> None:
        """A stream proceeded, but the user has crossed their warn threshold."""
        _emit(AuditEvent(
            event_type=AuditEventType.BUDGET_WARNING_ISSUED,
            actor_id=user_id,
            actor_role=user_role,
            session_id=session_id,
            outcome="warning",
            reason=(
                f"{period} spend ${spend_usd:.2f} has crossed the warn "
                f"threshold of ${limit_usd:.2f}."
            ),
            payload={"period": period, "spend_usd": round(spend_usd, 4), "limit_usd": limit_usd},
        ))

    def budget_exceeded(
        self, user_id: str, user_role: str, session_id: str | None,
        period: str, spend_usd: float, limit_usd: float,
    ) -> None:
        """A stream was blocked before the model was called - budget exhausted."""
        _emit(AuditEvent(
            event_type=AuditEventType.BUDGET_EXCEEDED_BLOCKED,
            actor_id=user_id,
            actor_role=user_role,
            session_id=session_id,
            outcome="blocked",
            reason=(
                f"{period} budget of ${limit_usd:.2f} reached "
                f"(spent ${spend_usd:.2f})."
            ),
            payload={"period": period, "spend_usd": round(spend_usd, 4), "limit_usd": limit_usd},
        ))
```

<a id="step-206"></a>
## Step 20.6 - Enforcing budgets in the streaming flow

This is the step that finally implements the "budget" part of `GovernedStreamManager`'s long-standing docstring. The check runs as **step 0** - before the concurrent stream slot is acquired, before the policy check, before anything that costs a Redis round-trip or an HTTP call to the governance API. A blown monthly budget doesn't need a policy opinion; it needs a fast, cheap "no."

First, a new event type alongside Day 9's `StreamBlockedEvent` and `StreamErrorEvent`:

```python
# backend/llm/governed_client.py (additions)

@dataclass
class StreamBudgetExceededEvent:
    """
    Sent as the ONLY event when the user's daily or monthly budget has
    been reached. The LLM is never called - no tokens generated, no
    cost incurred for this request.

    Deliberately a different event type from StreamBlockedEvent: a
    policy block means "rewrite your prompt," a budget block means
    "wait for your budget to reset (or ask for an increase)." Clients
    should present these differently.
    """
    blocked:   bool  = True
    period:    str   = ""     # "daily" or "monthly"
    spend_usd: float = 0.0
    limit_usd: float = 0.0
    reason:    str   = ""
    ts:        str   = field(default_factory=_now_iso)

    def to_sse(self) -> str:
        return f"data: {json.dumps(asdict(self))}\n\n"
```

Then the integration into `stream()`:

```python
# backend/llm/governed_client.py (additions)

from backend.billing.budget_enforcer import BudgetEnforcer, BudgetAction
from backend.billing.budget_tracker import record_cost
from backend.audit.audit_logger import AuditLogger

_budget_enforcer = BudgetEnforcer()
_audit = AuditLogger()


class GovernedStreamManager:
    # ... existing __init__, _hash, _concurrent_key, etc. ...

    async def stream(self) -> AsyncIterator[str]:
        stream_start = time.time()

        # ── 0. Budget check ────────────────────────────────────────
        # Cheapest check, and the most final - a blown monthly budget
        # doesn't recover until next month, so it goes first, ahead of
        # even the concurrent-stream-slot check.
        decision = await _budget_enforcer.check(self.user_id, self.user_role)

        if decision.action == BudgetAction.BLOCK:
            spend_for_period = getattr(decision.spend,  f"{decision.period}_usd")
            limit_for_period = getattr(decision.limits, f"{decision.period}_usd")

            _audit.budget_exceeded(
                user_id=self.user_id, user_role=self.user_role,
                session_id=self.session_id, period=decision.period,
                spend_usd=spend_for_period, limit_usd=limit_for_period,
            )
            yield StreamBudgetExceededEvent(
                period=decision.period,
                spend_usd=spend_for_period,
                limit_usd=limit_for_period,
                reason=decision.reason,
            ).to_sse()
            await self._write_stream_session(
                outcome="budget_exceeded", tokens=0, cost=0.0,
                duration_ms=int((time.time() - stream_start) * 1000),
                retracted=False,
            )
            return

        if decision.action == BudgetAction.WARN:
            _audit.budget_warning(
                user_id=self.user_id, user_role=self.user_role,
                session_id=self.session_id, period=decision.period,
                spend_usd=decision.spend.daily_usd,
                limit_usd=decision.limits.daily_usd,
            )
            # Not blocking - fall through to step 1.

        # ── 1. Concurrent stream limit ────────────────────────────
        acquired, reason = await self._acquire_stream_slot()
        if not acquired:
            yield StreamBlockedEvent(reason=[reason], risk_score=0).to_sse()
            return

        # ... steps 2+ unchanged from Day 9 (pre-stream governance,
        #     stream start, token streaming, post-stream scan) ...
```

And the actual dollar amount gets recorded alongside Day 9's existing call-count increments, in all three outcome branches that compute `actual_cost`:

```python
# backend/llm/governed_client.py (additions to the existing completion/retraction/
# disconnect branches inside stream())

                    # Day 9: per-stream call counters (unchanged)
                    today = datetime.now(timezone.utc).strftime("%Y-%m-%d")
                    await redis_incr(f"cost:user:{self.user_id}:{today}",   ttl_seconds=86400)
                    await redis_incr(f"cost:model:{self.model_id}:{today}", ttl_seconds=86400)

                    # Day 20: actual dollar totals
                    await record_cost(self.user_id, actual_cost)
```

The same `await record_cost(self.user_id, actual_cost)` line is added to the `retracted` branch and the `CancelledError` (disconnect) branch too - both already compute `actual_cost` for the `StreamSession` record, and Day 9's own comment is explicit that "tokens were consumed even if retracted." A retracted or disconnected stream still cost real money with the model provider, so it counts toward the budget the same way.

<a id="step-207"></a>
## Step 20.7 - Feeding budget events into abuse detection

Like Day 19's secret-access events, `BUDGET_EXCEEDED_BLOCKED` flows through `_emit()`, so wiring it into Day 18's abuse detector is pure configuration.

```python
# backend/security/abuse_schema.py (additions to ABUSE_THRESHOLDS)

ABUSE_THRESHOLDS: list[AbuseThreshold] = [
    # ... existing entries from Days 18-19 ...

    # Cost governance (Day 20)
    #
    # Being out of budget isn't itself suspicious - everyone runs out
    # eventually. Repeatedly retrying right after being blocked is more
    # likely a retry loop or script than a human reading the response,
    # so this throttles rather than suspends - the goal is to slow the
    # loop down, not lock the account out for an hour over a budget
    # that's going to reset on its own.
    AbuseThreshold(
        event_type=AuditEventType.BUDGET_EXCEEDED_BLOCKED,
        window_seconds=3600, count=3, action=AbuseAction.WARN,
        description="3 budget-exceeded stream attempts in 60 minutes",
    ),
    AbuseThreshold(
        event_type=AuditEventType.BUDGET_EXCEEDED_BLOCKED,
        window_seconds=3600, count=10, action=AbuseAction.THROTTLE,
        description="10 budget-exceeded stream attempts in 60 minutes",
    ),
]
```

<a id="step-208"></a>
## Step 20.8 - Spend visibility: the billing router

One endpoint for "how am I doing against my budget," and one for admins to check anyone. Authentication lands on Day 31 (per Day 18's note, these admin-style endpoints stay open like the rest of the audit/security API until then) - until that lands, the caller's identity and role come from the same `X-User-ID` / `X-User-Role` headers the rate limiter (Day 4) already reads.

```python
# backend/routers/billing.py
"""
Spend visibility.

GET /billing/usage              - caller's own current spend vs. their role's limits
GET /billing/usage/{user_id}    - any user's spend, given their role (admin tooling)

Authentication lands on Day 31. Until then this follows the same
convention as Day 4's rate limiter and Day 18's admin endpoints: caller
identity comes from X-User-ID / X-User-Role headers. Do not expose this
service to the public internet without auth in front of it.
"""
from fastapi import APIRouter, Header

from backend.billing.budget_schema import limits_for_role
from backend.billing.budget_tracker import get_spend

router = APIRouter(prefix="/billing", tags=["billing"])


async def _usage_payload(user_id: str, role: str) -> dict:
    spend  = await get_spend(user_id)
    limits = limits_for_role(role)

    return {
        "user_id": user_id,
        "role":    role,
        "daily": {
            "spent_usd":     round(spend.daily_usd, 4),
            "limit_usd":     limits.daily_usd,
            "remaining_usd": round(max(limits.daily_usd - spend.daily_usd, 0.0), 4),
        },
        "monthly": {
            "spent_usd":     round(spend.monthly_usd, 4),
            "limit_usd":     limits.monthly_usd,
            "remaining_usd": round(max(limits.monthly_usd - spend.monthly_usd, 0.0), 4),
        },
    }


@router.get("/usage")
async def my_usage(
    x_user_id:   str = Header(..., alias="X-User-ID"),
    x_user_role: str = Header("employee", alias="X-User-Role"),
):
    """The caller's own spend against their role's daily and monthly budgets."""
    return await _usage_payload(x_user_id, x_user_role)


@router.get("/usage/{user_id}")
async def user_usage(user_id: str, role: str = "employee"):
    """
    Look up any user's spend, given their role.

    `role` is a query parameter rather than looked up from a user
    directory - this service doesn't own user/role data (Day 4's
    rate limiter has the same property), so the caller supplies it.
    Example: GET /billing/usage/alice?role=analyst
    """
    return await _usage_payload(user_id, role)
```

<a id="step-209"></a>
## Step 20.9 - Tests

```python
# tests/billing/test_redis_float_counters.py
import pytest

from backend.cache.redis_client import redis_incrbyfloat, redis_get_float, get_redis


@pytest.fixture(autouse=True)
async def clean_keys():
    r = await get_redis()
    if r:
        for key in await r.keys("test:float:*"):
            await r.delete(key)
    yield
    if r:
        for key in await r.keys("test:float:*"):
            await r.delete(key)


class TestRedisFloatCounters:

    async def test_increment_from_zero(self):
        total = await redis_incrbyfloat("test:float:a", 0.0041)
        assert total == pytest.approx(0.0041)

    async def test_increments_accumulate(self):
        await redis_incrbyfloat("test:float:b", 0.01)
        await redis_incrbyfloat("test:float:b", 0.02)
        total = await redis_incrbyfloat("test:float:b", 0.03)
        assert total == pytest.approx(0.06)

    async def test_get_float_matches_running_total(self):
        await redis_incrbyfloat("test:float:c", 1.2345)
        await redis_incrbyfloat("test:float:c", 0.0001)
        assert await redis_get_float("test:float:c") == pytest.approx(1.2346)

    async def test_get_float_on_missing_key_is_zero(self):
        assert await redis_get_float("test:float:does-not-exist") == 0.0

    async def test_ttl_set_only_on_creation(self):
        r = await get_redis()
        await redis_incrbyfloat("test:float:d", 1.0, ttl_seconds=1000)
        ttl_after_create = await r.ttl("test:float:d")

        await redis_incrbyfloat("test:float:d", 1.0, ttl_seconds=1000)
        ttl_after_second = await r.ttl("test:float:d")

        # TTL shouldn't have been reset to 1000 again on the second call -
        # it should have only decreased (or stayed the same second).
        assert ttl_after_second <= ttl_after_create
```

```python
# tests/billing/test_budget_tracker.py
import pytest

from backend.billing import budget_tracker
from backend.cache.redis_client import get_redis


@pytest.fixture(autouse=True)
async def clean_keys():
    r = await get_redis()
    if r:
        for key in await r.keys("budget:user:tracker-test-*"):
            await r.delete(key)
    yield
    if r:
        for key in await r.keys("budget:user:tracker-test-*"):
            await r.delete(key)


class TestBudgetTracker:

    async def test_record_cost_updates_daily_and_monthly(self):
        spend = await budget_tracker.record_cost("tracker-test-1", 0.05)
        assert spend.daily_usd == pytest.approx(0.05)
        assert spend.monthly_usd == pytest.approx(0.05)

    async def test_record_cost_accumulates(self):
        await budget_tracker.record_cost("tracker-test-2", 0.10)
        await budget_tracker.record_cost("tracker-test-2", 0.25)
        spend = await budget_tracker.get_spend("tracker-test-2")
        assert spend.daily_usd == pytest.approx(0.35)
        assert spend.monthly_usd == pytest.approx(0.35)

    async def test_zero_cost_does_not_change_spend(self):
        await budget_tracker.record_cost("tracker-test-3", 0.10)
        await budget_tracker.record_cost("tracker-test-3", 0.0)
        spend = await budget_tracker.get_spend("tracker-test-3")
        assert spend.daily_usd == pytest.approx(0.10)

    async def test_get_spend_for_unknown_user_is_zero(self):
        spend = await budget_tracker.get_spend("tracker-test-never-used")
        assert spend.daily_usd == 0.0
        assert spend.monthly_usd == 0.0
```

```python
# tests/billing/test_budget_enforcer.py
import pytest

from backend.billing import budget_tracker
from backend.billing.budget_enforcer import BudgetEnforcer, BudgetAction
from backend.cache.redis_client import get_redis


@pytest.fixture(autouse=True)
async def clean_keys():
    r = await get_redis()
    if r:
        for key in await r.keys("budget:user:enforcer-test-*"):
            await r.delete(key)
    yield
    if r:
        for key in await r.keys("budget:user:enforcer-test-*"):
            await r.delete(key)


class TestBudgetEnforcer:

    def setup_method(self):
        self.enforcer = BudgetEnforcer()

    async def test_fresh_user_is_ok(self):
        decision = await self.enforcer.check("enforcer-test-1", "employee")
        assert decision.action == BudgetAction.OK

    async def test_crossing_warn_ratio_warns(self):
        # employee daily budget is $10.00, warn_ratio 0.8 -> $8.00
        await budget_tracker.record_cost("enforcer-test-2", 8.50)
        decision = await self.enforcer.check("enforcer-test-2", "employee")
        assert decision.action == BudgetAction.WARN
        assert decision.period == "daily"

    async def test_reaching_daily_cap_blocks(self):
        await budget_tracker.record_cost("enforcer-test-3", 10.00)
        decision = await self.enforcer.check("enforcer-test-3", "employee")
        assert decision.action == BudgetAction.BLOCK
        assert decision.period == "daily"

    async def test_reaching_monthly_cap_blocks_even_if_daily_ok(self):
        # employee monthly budget is $150.00. Simulate it being hit
        # without the daily counter (a new day) also being at its cap.
        await budget_tracker.record_cost("enforcer-test-4", 0.0)  # no-op, ensures keys exist
        from backend.billing.budget_tracker import _monthly_key
        from backend.cache.redis_client import redis_incrbyfloat
        await redis_incrbyfloat(_monthly_key("enforcer-test-4"), 150.00)

        decision = await self.enforcer.check("enforcer-test-4", "employee")
        assert decision.action == BudgetAction.BLOCK
        assert decision.period == "monthly"

    async def test_unknown_role_uses_unknown_limits(self):
        decision = await self.enforcer.check("enforcer-test-5", "totally-made-up-role")
        assert decision.limits.daily_usd == 1.00  # "unknown" role's daily_usd
```

```python
# tests/llm/test_governed_client_budget.py
import pytest

from backend.audit.audit_models import AuditLogEntry
from backend.billing import budget_tracker
from backend.cache.redis_client import get_redis
from backend.database.db import create_tables, SessionLocal
from backend.llm.governed_client import GovernedStreamManager


@pytest.fixture(autouse=True)
async def fresh_state():
    create_tables()
    db = SessionLocal()
    db.query(AuditLogEntry).delete()
    db.commit()
    db.close()

    r = await get_redis()
    if r:
        for key in await r.keys("budget:user:stream-test-*"):
            await r.delete(key)
        for key in await r.keys("stream:active:stream-test-*"):
            await r.delete(key)


class TestGovernedClientBudget:

    async def test_blocked_user_gets_only_budget_event(self):
        user_id = "stream-test-exhausted"
        # Push the daily counter to the employee cap ($10.00) before
        # the stream starts.
        await budget_tracker.record_cost(user_id, 10.00)

        manager = GovernedStreamManager(
            user_id=user_id, user_role="employee",
            model_id="gpt-4o", prompt="Summarize this.",
        )

        events = [event async for event in manager.stream()]

        assert len(events) == 1
        assert '"blocked": true' in events[0]
        assert '"period": "daily"' in events[0]

    async def test_blocked_stream_logs_budget_exceeded(self):
        user_id = "stream-test-audit"
        await budget_tracker.record_cost(user_id, 150.00)  # employee monthly cap

        manager = GovernedStreamManager(
            user_id=user_id, user_role="employee",
            model_id="gpt-4o", prompt="Summarize this.",
        )
        async for _ in manager.stream():
            pass

        db = SessionLocal()
        try:
            record = (
                db.query(AuditLogEntry)
                .filter(AuditLogEntry.event_type == "budget.exceeded_blocked")
                .first()
            )
        finally:
            db.close()

        assert record is not None
        assert record.actor_id == user_id

    async def test_blocked_stream_does_not_acquire_concurrent_slot(self):
        user_id = "stream-test-no-slot"
        await budget_tracker.record_cost(user_id, 10.00)

        manager = GovernedStreamManager(
            user_id=user_id, user_role="employee",
            model_id="gpt-4o", prompt="Summarize this.",
        )
        async for _ in manager.stream():
            pass

        r = await get_redis()
        active = await r.get(f"stream:active:{user_id}")
        assert active is None
```

Run them:

```bash
pytest tests/billing/ tests/llm/test_governed_client_budget.py -v
```

`test_blocked_stream_does_not_acquire_concurrent_slot` is the one that proves the ordering claim from Step 20.6: the budget check runs *before* `_acquire_stream_slot()`, so a budget-blocked request never touches the concurrent-stream counter at all - it doesn't consume a slot it was never going to use.

<a id="what-you-built"></a>
## What you built today

A `BUDGET_LIMITS` table - same shape as Day 4's `ROLE_LIMITS` - giving every role a daily and monthly USD cap plus a warn threshold. Two new Redis helpers, `redis_incrbyfloat()` and `redis_get_float()`, that finally give this system a way to track *dollars*, not just call counts, with TTL behavior that actually resets daily and monthly windows correctly. A `budget_tracker` module maintaining real running totals per user, fed by `estimate_cost()` - the function Day 9 already calls but whose output, until today, went nowhere. A `BudgetEnforcer` that checks monthly before daily (the harder constraint to recover from) and returns `OK` / `WARN` / `BLOCK`. Two new audit event types, `BUDGET_WARNING_ISSUED` and `BUDGET_EXCEEDED_BLOCKED`, flowing through Day 18's `_emit()`. A budget check wired into `GovernedStreamManager.stream()` as step zero - ahead of the concurrent-stream-slot check, the policy check, and the model-registry check - with its own `StreamBudgetExceededEvent` so clients can distinguish "you're out of budget" from "your prompt was risky." New `ABUSE_THRESHOLDS` entries - pure configuration, no new detector code - that throttle (not suspend) users who keep retrying after a budget block. And a `/billing/usage` router so both users and admins can see exactly where spend stands against limits, in real dollars, right now.

<a id="checklist"></a>
## Day 20 checklist

- [ ] `backend/billing/budget_schema.py` defines `BudgetLimits` and `BUDGET_LIMITS` for every role in Day 4's `ROLE_LIMITS`, plus an `unknown` fallback
- [ ] `backend/cache/redis_client.py` adds `redis_incrbyfloat()` (TTL set only on key creation) and `redis_get_float()`
- [ ] `backend/billing/budget_tracker.py` maintains `budget:user:{user_id}:{date}` and `budget:user:{user_id}:{month}` as running USD totals via `record_cost()` / `get_spend()`
- [ ] `backend/billing/budget_enforcer.py`'s `BudgetEnforcer.check()` checks the monthly cap before the daily cap, and returns `OK` / `WARN` / `BLOCK`
- [ ] `AuditEventType` includes `BUDGET_WARNING_ISSUED` and `BUDGET_EXCEEDED_BLOCKED`, with corresponding `AuditLogger` methods
- [ ] `GovernedStreamManager.stream()` runs the budget check as step 0 - before the concurrent-stream slot, before pre-stream governance - and yields a `StreamBudgetExceededEvent` with zero tokens billed on `BLOCK`
- [ ] `record_cost()` is called with `actual_cost` in the completed, retracted, and disconnected branches of `stream()`, alongside Day 9's existing call-count counters
- [ ] `ABUSE_THRESHOLDS` includes `BUDGET_EXCEEDED_BLOCKED` entries (WARN at 3, THROTTLE at 10 per 60 min)
- [ ] `GET /billing/usage` and `GET /billing/usage/{user_id}` return spend, limit, and remaining for both daily and monthly periods, never raw secret/cost internals beyond USD figures
- [ ] All tests in `tests/billing/` and `tests/llm/test_governed_client_budget.py` pass, including the concurrent-slot ordering test

---

[Back to repo](https://github.com/GuntruTirupathamma/AI-Governance) - Previous: [DAY19.md](./DAY19.md) - Next: DAY21.md - Multi-Tenant Isolation and Data Segregation
