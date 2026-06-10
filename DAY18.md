# DAY 18 - Rate Limiting and Abuse Detection

> **Series:** AI Governance Engineering - Zero to Production
> **Author:** GuntruTirupathamma
> **Day:** 18 of 35
> **Previous:** [DAY17.md](./DAY17.md) - Audit Trail and Tamper-Proof Logging
> **Next:** DAY19.md - Secrets Management and Credential Isolation

---

## Table of contents

1. [The problem with rate limiting alone](#the-problem)
2. [What abuse detection means here](#what-abuse-detection-means)
3. [What you will build today](#what-youll-build)
4. [Step 18.1 - Abuse signal thresholds](#step-181)
5. [Step 18.2 - Sliding-window signal tracker](#step-182)
6. [Step 18.3 - The abuse detector](#step-183)
7. [Step 18.4 - Suspension and throttle store](#step-184)
8. [Step 18.5 - Wiring abuse detection into the audit logger](#step-185)
9. [Step 18.6 - Enforcing suspensions in the orchestrator](#step-186)
10. [Step 18.7 - Applying throttles in the rate limiter](#step-187)
11. [Step 18.8 - Security admin endpoints](#step-188)
12. [Step 18.9 - Tests](#step-189)
13. [What you built today](#what-you-built)
14. [Day 18 checklist](#checklist)

---

<a id="the-problem"></a>
## The problem with rate limiting alone

Day 4 added Redis-backed rate limiting: every request gets counted against a per-user, per-role limit, and once that limit is hit the API returns 429. That stops the obvious abuse - someone hammering the API in a tight loop.

It does not stop the abuse that matters most in a governed multi-agent system, because that abuse does not look like "too many requests."

Picture three users, all comfortably under their rate limit:

```
User A: 1 request every 4 minutes. Each one is a different prompt
        injection technique - role-play framing, encoded instructions,
        fake system messages, nested quoting tricks.

User B: 1 request every 10 minutes. Each one asks the orchestrator
        to spawn an agent with a slightly higher privilege level than
        their role allows. Every attempt gets blocked. They keep trying
        with small variations.

User C: 1 request every 2 minutes. Each one asks a research agent to
        read a file path that walks up and out of its approved
        directory scope - ../../../etc/passwd, then a URL-encoded
        variant, then a symlink trick.
```

None of these trip a rate limit. Each individual request is well-formed and arrives at a normal pace. But the pattern across requests - the same actor, repeatedly triggering the same category of security block - is the textbook signature of someone probing the system for a gap in the policy engine.

Day 17 already records every one of these as a structured audit event: `INJECTION_DETECTED`, `PRIVILEGE_ESC`, `PATH_TRAVERSAL`, `GOAL_BLOCKED`, `AGENT_SPAWN_BLOCKED`, `MESSAGE_BLOCKED`, `HITL_REJECTED`. The data already exists. What is missing is something watching that stream in real time and acting on it.

That is today's work: turn the audit log from a record of what happened into a live signal that can throttle or suspend an actor before they find what they are looking for.

---

<a id="what-abuse-detection-means"></a>
## What abuse detection means here

The design has three pieces, and they map directly onto the three levels of response a real system needs.

**Signals.** Certain audit event types are "abuse signals" when they repeat for the same actor within a time window. A single `INJECTION_DETECTED` is normal - automated scanners and curious users trigger it constantly, and the message bus already blocked it. Five `INJECTION_DETECTED` events from the same actor in ten minutes is a different story.

**Thresholds and actions.** Each signal type has one or more thresholds. Crossing a threshold produces one of three actions:

```
WARN     - Record an abuse event. Visible to reviewers. No functional change.
THROTTLE - Cut the actor's request rate limit by a fixed factor for a
           cooldown period. They can still work, just slower.
SUSPEND  - Block the actor from starting new orchestrator sessions until
           the suspension expires or an admin lifts it manually.
```

**Sliding windows, not fixed buckets.** Day 9's cost tracking used `redis_incr` with a TTL - a fixed bucket that resets abruptly at a clock boundary. That is fine for "tokens spent today." It is the wrong shape for abuse detection: an actor could send 4 injection attempts at 11:59 and 4 more at 12:01 and never trip a "5 per hour" fixed-bucket limit, despite sending 8 in two minutes. Today's tracker uses a Redis sorted set keyed by timestamp, so "how many in the last N seconds" is always accurate regardless of when you ask.

One design note before the code: the audit pipeline (`AuditWriter`, built on Day 17) is synchronous - plain SQLAlchemy calls, no event loop. Abuse detection runs inline with every audit write, so it uses a small synchronous Redis client, separate from the async one Day 4 built for caching and streaming. Same Redis instance, two connection pools, each suited to the code path that uses it.

And the same caveat from Day 17 applies here: this fails open. If Redis is unreachable, abuse detection silently does nothing rather than blocking the audit pipeline or the orchestrator. That is the right trade-off for a learning environment - a missing cache should never be the reason a governance system goes down - but in production you would alert loudly on "abuse detector has been unable to reach Redis for N minutes."

What this does **not** do: catch distributed abuse, where one person spreads the same probing pattern across many different accounts, each staying just under every threshold. That requires correlating signals across actors - shared IP, shared device fingerprint, shared timing pattern - which is a different kind of analysis for a later hardening day.

---

<a id="what-youll-build"></a>
## What you will build today

```
backend/security/
    __init__.py
    abuse_schema.py        - Thresholds: which audit events count as abuse
                              signals, how many in what window, and what
                              action each threshold triggers
    redis_sync.py          - Synchronous Redis client shared by the
                              tracker and the suspension store
    abuse_tracker.py        - Sliding-window counters per actor per signal
    abuse_detector.py        - Evaluates one audit event against the
                              thresholds and returns an action, if any
    suspensions.py           - Suspend / throttle state, Redis-backed
                              with TTL, plus admin lift/list helpers

backend/audit/audit_schema.py   (modified) - 5 new event types
backend/audit/audit_logger.py   (modified) - _emit() hook that runs
                                              every audit write through
                                              the abuse detector

backend/orchestrator/orchestrator.py  (modified) - reject new sessions
                                                    from suspended actors
backend/middleware/rate_limiter.py     (modified) - apply throttle
                                                      factor to role limits

backend/routers/security.py     - Admin endpoints: list suspensions,
                                   lift a suspension, view recent abuse
                                   events

tests/security/
    __init__.py
    test_abuse_tracker.py
    test_abuse_detector.py
    test_suspensions.py
    test_audit_abuse_integration.py
```

---

<a id="step-181"></a>
## Step 18.1 - Abuse signal thresholds

Start with configuration, not code. Deciding what counts as abuse - and how aggressively to respond - is a policy decision, and like the other policy decisions in this series it belongs in one place where it is easy to read and easy to tune.

```python
# backend/security/abuse_schema.py
"""
Abuse detection configuration.

Maps audit event types to thresholds. When the same actor triggers a
watched event type enough times within a window, AbuseDetector returns
the action to take: warn, throttle, or suspend.

Thresholds for the same event type are ordered least to most severe.
AbuseDetector checks all of them and returns the highest-severity action
the actor has actually crossed.
"""
from dataclasses import dataclass
from enum import Enum

from backend.audit.audit_schema import AuditEventType


class AbuseAction(str, Enum):
    NONE     = "none"
    WARN     = "warn"
    THROTTLE = "throttle"
    SUSPEND  = "suspend"


@dataclass(frozen=True)
class AbuseThreshold:
    event_type:     AuditEventType
    window_seconds: int
    count:          int
    action:         AbuseAction
    description:    str


# ─── Thresholds ──────────────────────────────────────────────────────────
#
# Rule of thumb used below:
#   - Security bypass attempts (injection, privilege escalation, path
#     traversal) get short windows and low counts. One or two is
#     suspicious; these are not things that happen by accident.
#   - Policy-boundary probing (blocked goals, blocked spawns, blocked
#     messages, HITL rejections) gets longer windows and higher counts.
#     Legitimate users occasionally hit a policy they didn't know about;
#     repeatedly hitting the *same kind* of block is the signal.
#
ABUSE_THRESHOLDS: list[AbuseThreshold] = [

    # Prompt injection attempts.
    AbuseThreshold(AuditEventType.INJECTION_DETECTED, window_seconds=600, count=3,
                    action=AbuseAction.WARN,
                    description="3 injection attempts in 10 minutes"),
    AbuseThreshold(AuditEventType.INJECTION_DETECTED, window_seconds=600, count=5,
                    action=AbuseAction.SUSPEND,
                    description="5 injection attempts in 10 minutes"),

    # Privilege escalation - even a couple of attempts is a strong signal.
    AbuseThreshold(AuditEventType.PRIVILEGE_ESC, window_seconds=1800, count=2,
                    action=AbuseAction.WARN,
                    description="2 privilege escalation attempts in 30 minutes"),
    AbuseThreshold(AuditEventType.PRIVILEGE_ESC, window_seconds=1800, count=3,
                    action=AbuseAction.SUSPEND,
                    description="3 privilege escalation attempts in 30 minutes"),

    # Path traversal - almost never accidental, even once.
    AbuseThreshold(AuditEventType.PATH_TRAVERSAL, window_seconds=3600, count=1,
                    action=AbuseAction.WARN,
                    description="1 path traversal attempt in 60 minutes"),
    AbuseThreshold(AuditEventType.PATH_TRAVERSAL, window_seconds=3600, count=2,
                    action=AbuseAction.SUSPEND,
                    description="2 path traversal attempts in 60 minutes"),

    # Goals blocked by policy - probing for what's allowed.
    AbuseThreshold(AuditEventType.GOAL_BLOCKED, window_seconds=900, count=5,
                    action=AbuseAction.WARN,
                    description="5 blocked goals in 15 minutes"),
    AbuseThreshold(AuditEventType.GOAL_BLOCKED, window_seconds=900, count=10,
                    action=AbuseAction.THROTTLE,
                    description="10 blocked goals in 15 minutes"),

    # Agent spawns blocked by policy.
    AbuseThreshold(AuditEventType.AGENT_SPAWN_BLOCKED, window_seconds=900, count=5,
                    action=AbuseAction.THROTTLE,
                    description="5 blocked agent spawns in 15 minutes"),

    # Inter-agent messages blocked by the message bus.
    AbuseThreshold(AuditEventType.MESSAGE_BLOCKED, window_seconds=900, count=8,
                    action=AbuseAction.THROTTLE,
                    description="8 blocked inter-agent messages in 15 minutes"),

    # Repeated HITL rejections - the agent keeps proposing things humans
    # keep turning down.
    AbuseThreshold(AuditEventType.HITL_REJECTED, window_seconds=3600, count=3,
                    action=AbuseAction.WARN,
                    description="3 HITL rejections in 60 minutes"),
    AbuseThreshold(AuditEventType.HITL_REJECTED, window_seconds=3600, count=5,
                    action=AbuseAction.THROTTLE,
                    description="5 HITL rejections in 60 minutes"),
]


# ─── Action durations ───────────────────────────────────────────────────
# Kept simple on purpose: one throttle config, one suspend duration,
# regardless of which threshold fired. A production system might scale
# these by severity; here, a human reviews every suspension anyway.
THROTTLE_DURATION_SECONDS = 1800   # 30 minutes at a reduced rate limit
THROTTLE_FACTOR           = 4      # role's normal limit divided by this
SUSPEND_DURATION_SECONDS  = 3600   # 1 hour, or until an admin lifts it


@dataclass
class AbuseDecision:
    """What AbuseDetector decided to do about one actor's behavior."""
    action:         AbuseAction
    signal_type:    AuditEventType
    count:          int
    threshold:      int
    window_seconds: int
    reason:         str
```

---

<a id="step-182"></a>
## Step 18.2 - Sliding-window signal tracker

First, the shared synchronous Redis client. This mirrors the structure of `backend/cache/redis_client.py` from Day 4, but uses the plain `redis` client instead of `redis.asyncio`, and lives in its own module so the security package has no dependency on the streaming/caching code.

```python
# backend/security/redis_sync.py
"""
Synchronous Redis client for the security module.

Audit writes (backend/audit/audit_writer.py) run on a synchronous
SQLAlchemy session - no event loop involved. Abuse detection runs inline
with every audit write, so it needs a synchronous Redis client too. This
is a second, small connection alongside the async one in
backend/cache/redis_client.py, pointed at the same Redis instance.
"""
import os
import logging
from typing import Optional

import redis

logger = logging.getLogger(__name__)

REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379")

_client: Optional["redis.Redis"] = None
_warned  = False


def get_client() -> Optional["redis.Redis"]:
    """
    Returns the shared synchronous Redis client, or None if Redis is
    unreachable.

    Callers must handle None. Abuse detection fails open rather than
    breaking the audit pipeline - see the design note in DAY18.md.
    """
    global _client, _warned

    if _client is not None:
        return _client

    try:
        client = redis.Redis.from_url(REDIS_URL, decode_responses=True)
        client.ping()
        _client = client
        return _client
    except Exception as e:
        if not _warned:
            logger.warning(
                f"Security module: Redis unavailable ({e}). "
                f"Abuse detection and suspensions are disabled."
            )
            _warned = True
        return None
```

Now the tracker itself. Each signal is one entry in a Redis sorted set, scored by timestamp. `record()` adds the new entry, drops everything older than the window, and returns the count - all in one pipeline.

```python
# backend/security/abuse_tracker.py
"""
Sliding-window event counters for abuse detection.

Each call to record() adds a timestamped entry to a Redis sorted set for
(actor_id, signal_type), trims anything older than the window, and returns
how many entries remain. That count is "how many times has this actor
triggered this signal in the last `window_seconds` seconds, including
right now."

Sorted sets give a true sliding window. A fixed-bucket counter (the
redis_incr + TTL pattern from Day 9) resets abruptly at a clock boundary;
a sorted set does not care when you ask.
"""
import time
import uuid

from backend.security.redis_sync import get_client

KEY_PREFIX = "abuse:signal"


def _key(actor_id: str, signal_type: str) -> str:
    return f"{KEY_PREFIX}:{actor_id}:{signal_type}"


def record(actor_id: str, signal_type: str, window_seconds: int) -> int:
    """
    Record one occurrence of `signal_type` for `actor_id` and return the
    number of occurrences within the trailing `window_seconds`.

    Returns 0 if Redis is unreachable - see redis_sync.get_client().
    """
    client = get_client()
    if client is None:
        return 0

    key    = _key(actor_id, signal_type)
    now    = time.time()
    cutoff = now - window_seconds
    member = f"{now}:{uuid.uuid4().hex}"   # unique even for same-millisecond hits

    pipe = client.pipeline()
    pipe.zadd(key, {member: now})
    pipe.zremrangebyscore(key, 0, cutoff)
    pipe.zcard(key)
    pipe.expire(key, window_seconds)
    _, _, count, _ = pipe.execute()

    return int(count)


def reset(actor_id: str, signal_type: str) -> None:
    """Clear an actor's counter for one signal type. Used by tests and admins."""
    client = get_client()
    if client is None:
        return
    client.delete(_key(actor_id, signal_type))
```

---

<a id="step-183"></a>
## Step 18.3 - The abuse detector

The detector's job is narrow: given one audit event, decide whether it pushes the actor over a threshold, and if so, which action applies. It does not write audit records or change any state outside Redis - that happens one layer up, in the audit logger, where the response is decided.

```python
# backend/security/abuse_detector.py
"""
Abuse detector.

Watches the stream of audit events for patterns that indicate a single
actor is probing the system - repeated injection attempts, repeated
privilege escalation, repeated policy violations - even when each
individual request is well within normal rate limits.

evaluate() is called from AuditLogger after every write (Step 18.5). Only
event types listed in ABUSE_THRESHOLDS do anything; everything else is a
fast no-op.
"""
from typing import Optional

from backend.audit.audit_schema import AuditEvent, AuditEventType
from backend.security.abuse_schema import ABUSE_THRESHOLDS, AbuseAction, AbuseDecision
from backend.security import abuse_tracker


# Index thresholds by event type for O(1) lookup on the hot path.
_THRESHOLDS_BY_EVENT: dict[AuditEventType, list[object]] = {}
for _t in ABUSE_THRESHOLDS:
    _THRESHOLDS_BY_EVENT.setdefault(_t.event_type, []).append(_t)

# Used to pick the worst action when an event crosses more than one
# threshold at once (e.g. the 5th injection attempt crosses both the
# WARN-at-3 and SUSPEND-at-5 thresholds).
_ACTION_SEVERITY = {
    AbuseAction.NONE:     0,
    AbuseAction.WARN:     1,
    AbuseAction.THROTTLE: 2,
    AbuseAction.SUSPEND:  3,
}


class AbuseDetector:

    def evaluate(self, event: AuditEvent) -> Optional[AbuseDecision]:
        """
        Record this event as a signal (if it's a watched type) and check
        whether the actor has crossed an abuse threshold.

        Returns None if the event type isn't watched, the actor is
        "system", or no threshold was crossed. Otherwise returns the
        highest-severity AbuseDecision triggered by this event.
        """
        thresholds = _THRESHOLDS_BY_EVENT.get(event.event_type)
        if not thresholds:
            return None

        if not event.actor_id or event.actor_id == "system":
            return None

        # Multiple thresholds for the same event type may share a window.
        # Record once per distinct window so the same occurrence isn't
        # double-counted.
        windows = sorted({t.window_seconds for t in thresholds})
        counts  = {
            w: abuse_tracker.record(event.actor_id, event.event_type.value, w)
            for w in windows
        }

        best: Optional[AbuseDecision] = None
        for threshold in thresholds:
            count = counts[threshold.window_seconds]
            if count < threshold.count:
                continue

            if best is None or _ACTION_SEVERITY[threshold.action] > _ACTION_SEVERITY[best.action]:
                best = AbuseDecision(
                    action         = threshold.action,
                    signal_type    = event.event_type,
                    count          = count,
                    threshold      = threshold.count,
                    window_seconds = threshold.window_seconds,
                    reason = (
                        f"actor='{event.actor_id}' event='{event.event_type.value}' "
                        f"count={count} in {threshold.window_seconds}s "
                        f"(threshold: {threshold.description})"
                    ),
                )

        return best
```

---

<a id="step-184"></a>
## Step 18.4 - Suspension and throttle store

State for two things, both Redis-backed with a TTL so they expire automatically: a suspension blocks new orchestrator sessions outright, a throttle reduces the rate-limit middleware's allowance for an actor.

```python
# backend/security/suspensions.py
"""
Suspension and throttle state.

A suspended actor cannot start new orchestrator sessions until the
suspension expires or an admin lifts it via backend/routers/security.py.

A throttled actor keeps full access but with a reduced request rate limit,
applied in backend/middleware/rate_limiter.py.

Both are stored in Redis with a TTL, so an unreviewed suspension or
throttle expires on its own rather than persisting forever.
"""
import json
import time
from typing import Optional

from backend.security.redis_sync import get_client

SUSPEND_PREFIX  = "abuse:suspended"
THROTTLE_PREFIX = "abuse:throttled"


def suspend(actor_id: str, reason: str, duration_seconds: int) -> None:
    """Suspend an actor for `duration_seconds`. Idempotent - overwrites any existing suspension."""
    client = get_client()
    if client is None:
        return

    record = json.dumps({
        "reason":     reason,
        "since":      time.time(),
        "expires_at": time.time() + duration_seconds,
    })
    client.set(f"{SUSPEND_PREFIX}:{actor_id}", record, ex=duration_seconds)


def is_suspended(actor_id: str) -> Optional[dict]:
    """Returns the suspension record if the actor is currently suspended, else None."""
    client = get_client()
    if client is None:
        return None

    raw = client.get(f"{SUSPEND_PREFIX}:{actor_id}")
    return json.loads(raw) if raw else None


def lift_suspension(actor_id: str) -> bool:
    """Manually lift a suspension before it expires. Returns True if one existed."""
    client = get_client()
    if client is None:
        return False

    existing = client.get(f"{SUSPEND_PREFIX}:{actor_id}")
    client.delete(f"{SUSPEND_PREFIX}:{actor_id}")
    return existing is not None


def throttle(actor_id: str, duration_seconds: int, factor: int) -> None:
    """Reduce an actor's effective rate limit by `factor` for `duration_seconds`."""
    client = get_client()
    if client is None:
        return

    record = json.dumps({
        "factor":     factor,
        "since":      time.time(),
        "expires_at": time.time() + duration_seconds,
    })
    client.set(f"{THROTTLE_PREFIX}:{actor_id}", record, ex=duration_seconds)


def get_throttle_factor(actor_id: str) -> int:
    """Returns the rate-limit divisor currently applied to this actor, or 1 if none."""
    client = get_client()
    if client is None:
        return 1

    raw = client.get(f"{THROTTLE_PREFIX}:{actor_id}")
    if not raw:
        return 1
    return json.loads(raw).get("factor", 1)


def list_suspended() -> list[str]:
    """All currently suspended actor IDs. Used by the admin endpoint."""
    client = get_client()
    if client is None:
        return []

    return [
        key.split(":")[-1]
        for key in client.scan_iter(match=f"{SUSPEND_PREFIX}:*")
    ]
```

---

<a id="step-185"></a>
## Step 18.5 - Wiring abuse detection into the audit logger

Two small additions to the audit module from Day 17: five new event types, and a single `_emit()` function that every `AuditLogger` method routes through instead of calling `_writer.write()` directly.

First, add these to the `AuditEventType` enum in `backend/audit/audit_schema.py`, after `AUDIT_TAMPER`:

```python
# backend/audit/audit_schema.py  (addition to AuditEventType)

    # Abuse detection and account actions
    ABUSE_DETECTED            = "security.abuse_detected"
    ACCOUNT_THROTTLED         = "account.throttled"
    ACCOUNT_SUSPENDED         = "account.suspended"
    ACCOUNT_SUSPENSION_LIFTED = "account.suspension_lifted"
    SESSION_BLOCKED_SUSPENDED = "session.blocked_suspended"
```

None of these five are in `ABUSE_THRESHOLDS`, so logging them never re-triggers the detector. That matters - without it, suspending someone for too many `INJECTION_DETECTED` events would log `ACCOUNT_SUSPENDED`, which (if it were itself a watched type) could spiral.

Now the logger. Add the imports and `_emit()` near the top of `backend/audit/audit_logger.py`:

```python
# backend/audit/audit_logger.py  (additions)

from backend.audit.audit_models import AuditLogEntry
from backend.security.abuse_detector import AbuseDetector
from backend.security.abuse_schema import (
    AbuseAction,
    THROTTLE_DURATION_SECONDS, THROTTLE_FACTOR, SUSPEND_DURATION_SECONDS,
)
from backend.security import suspensions

_writer   = AuditWriter()
_detector = AbuseDetector()


def _emit(event: AuditEvent) -> AuditLogEntry:
    """
    Write an audit event, then run it through the abuse detector.

    Every method on AuditLogger calls this instead of _writer.write()
    directly. For event types AbuseDetector doesn't watch, evaluate()
    returns None immediately and this is exactly equivalent to the old
    _writer.write(event) call.
    """
    entry = _writer.write(event)

    decision = _detector.evaluate(event)
    if decision is None:
        return entry

    _writer.write(AuditEvent(
        event_type = AuditEventType.ABUSE_DETECTED,
        actor_id   = event.actor_id,
        actor_role = event.actor_role,
        session_id = event.session_id,
        outcome    = decision.action.value,
        reason     = decision.reason,
        payload    = {
            "signal_type":    decision.signal_type.value,
            "count":          decision.count,
            "threshold":      decision.threshold,
            "window_seconds": decision.window_seconds,
        },
    ))

    if decision.action == AbuseAction.SUSPEND:
        suspensions.suspend(event.actor_id, decision.reason, SUSPEND_DURATION_SECONDS)
        _writer.write(AuditEvent(
            event_type = AuditEventType.ACCOUNT_SUSPENDED,
            actor_id   = event.actor_id,
            actor_role = event.actor_role,
            session_id = event.session_id,
            outcome    = "suspended",
            reason     = decision.reason,
            payload    = {"duration_seconds": SUSPEND_DURATION_SECONDS},
        ))

    elif decision.action == AbuseAction.THROTTLE:
        suspensions.throttle(event.actor_id, THROTTLE_DURATION_SECONDS, THROTTLE_FACTOR)
        _writer.write(AuditEvent(
            event_type = AuditEventType.ACCOUNT_THROTTLED,
            actor_id   = event.actor_id,
            actor_role = event.actor_role,
            session_id = event.session_id,
            outcome    = "throttled",
            reason     = decision.reason,
            payload    = {"duration_seconds": THROTTLE_DURATION_SECONDS, "factor": THROTTLE_FACTOR},
        ))

    return entry
```

Then replace `_writer.write(` with `_emit(` in every method below it - `session_created`, `session_closed`, `spawn_allowed`, `spawn_blocked`, `goal_blocked`, `privilege_escalation`, `hitl_created`, `hitl_decided`, `message_blocked`, `message_redacted`, `injection_detected`, `path_traversal`. The method bodies do not otherwise change. For example:

```python
    # backend/audit/audit_logger.py  (example of the mechanical change)

    def injection_detected(
        self, session_id: str, actor_id: str,
        pattern_matched: str, source: str,
    ):
        _emit(AuditEvent(                       # was: _writer.write(AuditEvent(
            event_type = AuditEventType.INJECTION_DETECTED,
            actor_id   = actor_id,
            session_id = session_id,
            outcome    = "blocked",
            payload    = {"pattern": pattern_matched, "source": source},
        ))
```

Finally, two new methods for the event types added above:

```python
    # backend/audit/audit_logger.py  (new methods)

    def session_blocked_suspended(self, user_id: str, role: str, reason: str):
        """The orchestrator refused to start a session because the actor is suspended."""
        _writer.write(AuditEvent(
            event_type = AuditEventType.SESSION_BLOCKED_SUSPENDED,
            actor_id   = user_id,
            actor_role = role,
            outcome    = "blocked",
            reason     = reason,
        ))

    def suspension_lifted(self, actor_id: str, reviewer_id: str):
        """An admin manually lifted a suspension before it expired."""
        _writer.write(AuditEvent(
            event_type  = AuditEventType.ACCOUNT_SUSPENSION_LIFTED,
            actor_id    = reviewer_id,
            resource_id = actor_id,
            outcome     = "lifted",
            payload     = {"suspended_actor": actor_id},
        ))
```

These two call `_writer.write()` directly rather than `_emit()`. Neither event type is in `ABUSE_THRESHOLDS`, so the behavior would be identical either way - using `_writer.write()` here is just a small signal to the reader that these are administrative actions, not signals the detector needs to look at.

---

<a id="step-186"></a>
## Step 18.6 - Enforcing suspensions in the orchestrator

Detection without enforcement is just a dashboard. The orchestrator's entry point - the one place every governed action starts - is where a suspension actually stops something.

```python
# backend/orchestrator/orchestrator.py  (additions)

from backend.security import suspensions


class AgentOrchestrator:

    # ... __init__ unchanged ...

    async def orchestrate(
        self,
        goal:      str,
        user_id:   str,
        user_role: str,
        context:   dict = None,
    ) -> dict:

        suspension = suspensions.is_suspended(user_id)
        if suspension is not None:
            _audit.session_blocked_suspended(user_id, user_role, suspension["reason"])
            raise PermissionError(
                f"Account suspended pending review. Reason: {suspension['reason']}. "
                f"Suspension expires at {suspension['expires_at']}."
            )

        start_time = datetime.utcnow()
        session_id = self.session_mgr.create_session(
            user_id=user_id, user_role=user_role, goal=goal
        )
        # ... rest of orchestrate() unchanged ...
```

The check happens before `create_session()` - a suspended actor never gets a session ID, never shows up in `_active_sessions`, and never reaches the policy engine or agent registry. The `PermissionError` propagates up to the FastAPI route, which is already set up (Day 13/14) to turn exceptions like this into a 403.

Two things worth calling out:

The suspension check itself - `suspensions.is_suspended()` - is not in `ABUSE_THRESHOLDS` and does not call `_emit()`. It is a read, not a write. Only `session_blocked_suspended()` writes a record, and that record is informational (`SESSION_BLOCKED_SUSPENDED` is not a watched type), so a suspended user repeatedly trying to start sessions does not escalate into anything worse. They are already suspended; there is nothing further to do until a human reviews it.

A suspension blocks *new* sessions. It does not reach into a session that is already running. If you need a kill switch for in-flight sessions too, that is a session-manager change, not an orchestrator-entry change - worth keeping in mind, but out of scope for today.

---

<a id="step-187"></a>
## Step 18.7 - Applying throttles in the rate limiter

The rate limiter from Day 4/5 looks up a per-role request limit from `ROLE_LIMITS` in `backend/middleware/rate_limit_config.py` and counts requests against it in Redis. The only change today is: before comparing the request count to the limit, divide the limit by the actor's current throttle factor.

```python
# backend/middleware/rate_limiter.py  (additions)

from backend.security import suspensions
from backend.middleware.rate_limit_config import ROLE_LIMITS


def _effective_limit(role: str, actor_id: str) -> int:
    """
    The role's normal per-window request limit, reduced by the actor's
    current throttle factor (1 if the actor isn't throttled).

    A factor of 4 on a 100-requests-per-minute role means the actor gets
    25 requests per minute until the throttle expires.
    """
    base_limit = ROLE_LIMITS.get(role, ROLE_LIMITS["default"])
    factor     = suspensions.get_throttle_factor(actor_id)
    return max(1, base_limit // factor)
```

In the middleware's request-handling path, replace the existing lookup:

```python
        # Before:
        # limit = ROLE_LIMITS.get(user_role, ROLE_LIMITS["default"])

        # After:
        limit = _effective_limit(user_role, user_id)

        # ... existing logic: increment the request counter, compare
        #     to `limit`, return 429 if exceeded ...
```

`get_throttle_factor()` is a single Redis `GET` - the same cost as the rate check that already runs on every request, so this adds negligible latency. If Redis is unreachable, `get_throttle_factor()` returns `1` (no throttle applied), which is consistent with the rest of this design failing open.

One consequence worth being explicit about: a throttled actor's reduced limit is *global* across all their requests, not scoped to the kind of action that got them throttled. Someone throttled for repeatedly triggering blocked agent spawns also gets a slower API in general. That is intentional - the throttle is a "things are getting suspicious, slow down across the board" signal, not a surgical restriction on one endpoint.

---

<a id="step-188"></a>
## Step 18.8 - Security admin endpoints

Read access to suspensions and abuse events, plus one write endpoint - lifting a suspension early. This follows the same shape as `backend/routers/audit.py` from Day 17: query the audit log directly, no separate reporting table to keep in sync.

```python
# backend/routers/security.py
"""
Admin endpoints for abuse detection, suspensions, and throttles.

Read-only except for /security/suspensions/{actor_id}/lift, which clears
a suspension before it expires on its own.

Note: this series adds authentication on Day 31. Until then, these
endpoints are open like the rest of the audit API - do not expose this
service to the public internet without auth in front of it.
"""
from fastapi import APIRouter, HTTPException, Query
from pydantic import BaseModel
from typing import Optional

from backend.audit.audit_logger import AuditLogger
from backend.audit.audit_models import AuditLogEntry
from backend.audit.audit_schema import AuditEventType
from backend.database.db import SessionLocal
from backend.security import suspensions

router = APIRouter(prefix="/security", tags=["security"])
_audit = AuditLogger()


class SuspensionOut(BaseModel):
    actor_id:   str
    reason:     str
    since:      float
    expires_at: float


@router.get("/suspensions", response_model=list[SuspensionOut])
def list_suspensions():
    """Every actor currently suspended, with the reason and expiry time."""
    out = []
    for actor_id in suspensions.list_suspended():
        record = suspensions.is_suspended(actor_id)
        if record:
            out.append({"actor_id": actor_id, **record})
    return out


@router.post("/suspensions/{actor_id}/lift")
def lift_suspension(actor_id: str, reviewer_id: str = Query(...)):
    """Manually lift a suspension before it expires. Logs who did it."""
    lifted = suspensions.lift_suspension(actor_id)
    if not lifted:
        raise HTTPException(status_code=404, detail=f"No active suspension for '{actor_id}'.")

    _audit.suspension_lifted(actor_id, reviewer_id=reviewer_id)
    return {"actor_id": actor_id, "status": "lifted", "reviewer_id": reviewer_id}


@router.get("/throttle/{actor_id}")
def get_throttle_status(actor_id: str):
    """Current throttle factor for an actor. 1 means not throttled."""
    return {"actor_id": actor_id, "throttle_factor": suspensions.get_throttle_factor(actor_id)}


@router.get("/abuse-events")
def recent_abuse_events(limit: int = Query(50, le=200)):
    """Recent abuse detections and the account actions they triggered."""
    types = [
        AuditEventType.ABUSE_DETECTED.value,
        AuditEventType.ACCOUNT_THROTTLED.value,
        AuditEventType.ACCOUNT_SUSPENDED.value,
        AuditEventType.ACCOUNT_SUSPENSION_LIFTED.value,
        AuditEventType.SESSION_BLOCKED_SUSPENDED.value,
    ]
    db = SessionLocal()
    try:
        return (
            db.query(AuditLogEntry)
            .filter(AuditLogEntry.event_type.in_(types))
            .order_by(AuditLogEntry.id.desc())
            .limit(limit)
            .all()
        )
    finally:
        db.close()


@router.get("/abuse-summary/{actor_id}")
def actor_abuse_summary(actor_id: str):
    """One actor's abuse history: current suspension/throttle plus recent abuse events."""
    db = SessionLocal()
    try:
        events = (
            db.query(AuditLogEntry)
            .filter(
                AuditLogEntry.actor_id == actor_id,
                AuditLogEntry.event_type == AuditEventType.ABUSE_DETECTED.value,
            )
            .order_by(AuditLogEntry.id.desc())
            .limit(20)
            .all()
        )
    finally:
        db.close()

    return {
        "actor_id":         actor_id,
        "suspended":        suspensions.is_suspended(actor_id),
        "throttle_factor":  suspensions.get_throttle_factor(actor_id),
        "recent_abuse_events": events,
    }
```

Register the router in `backend/main.py` and bump the version to `"0.6.0"`:

```python
# backend/main.py  (additions)

from backend.routers import security as security_router

app.include_router(security_router.router)

app = FastAPI(
    title="AI Governance Platform",
    version="0.6.0",
)
```

---

<a id="step-189"></a>
## Step 18.9 - Tests

Four files: the tracker in isolation, the detector against the threshold table, the suspension/throttle store, and an end-to-end test that drives real abuse through `AuditLogger` and checks that a suspension comes out the other end.

```python
# tests/security/test_abuse_tracker.py
import time
import pytest

from backend.security import abuse_tracker
from backend.security.redis_sync import get_client


@pytest.fixture(autouse=True)
def clean_keys():
    client = get_client()
    if client:
        for key in client.scan_iter(match="abuse:signal:test-*"):
            client.delete(key)
    yield
    if client:
        for key in client.scan_iter(match="abuse:signal:test-*"):
            client.delete(key)


class TestAbuseTracker:

    def test_record_increments_count(self):
        actor = "test-user-1"
        first  = abuse_tracker.record(actor, "security.injection_detected", 60)
        second = abuse_tracker.record(actor, "security.injection_detected", 60)
        assert second == first + 1

    def test_window_expires_old_entries(self):
        actor = "test-user-2"
        abuse_tracker.record(actor, "security.injection_detected", window_seconds=1)
        time.sleep(1.2)
        count = abuse_tracker.record(actor, "security.injection_detected", window_seconds=1)
        # The first entry aged out of the 1-second window.
        assert count == 1

    def test_different_actors_are_independent(self):
        count_a = abuse_tracker.record("test-user-a", "security.injection_detected", 60)
        count_b = abuse_tracker.record("test-user-b", "security.injection_detected", 60)
        assert count_a == 1
        assert count_b == 1

    def test_different_signal_types_are_independent(self):
        actor = "test-user-3"
        abuse_tracker.record(actor, "security.injection_detected", 60)
        count = abuse_tracker.record(actor, "policy.privilege_escalation", 60)
        assert count == 1

    def test_reset_clears_counter(self):
        actor = "test-user-4"
        abuse_tracker.record(actor, "security.injection_detected", 60)
        abuse_tracker.reset(actor, "security.injection_detected")
        count = abuse_tracker.record(actor, "security.injection_detected", 60)
        assert count == 1


# tests/security/test_abuse_detector.py
import pytest

from backend.audit.audit_schema import AuditEvent, AuditEventType
from backend.security.abuse_detector import AbuseDetector
from backend.security.abuse_schema import AbuseAction
from backend.security.redis_sync import get_client


@pytest.fixture(autouse=True)
def clean_keys():
    client = get_client()
    if client:
        for key in client.scan_iter(match="abuse:signal:detector-test-*"):
            client.delete(key)


def _injection_event(actor: str) -> AuditEvent:
    return AuditEvent(
        event_type = AuditEventType.INJECTION_DETECTED,
        actor_id   = actor,
        outcome    = "blocked",
        payload    = {"pattern": "ignore previous instructions", "source": "user_input"},
    )


class TestAbuseDetector:

    def setup_method(self):
        self.detector = AbuseDetector()

    def test_below_threshold_returns_none(self):
        actor    = "detector-test-1"
        decision = None
        for _ in range(2):
            decision = self.detector.evaluate(_injection_event(actor))
        assert decision is None

    def test_third_injection_triggers_warn(self):
        actor    = "detector-test-2"
        decision = None
        for _ in range(3):
            decision = self.detector.evaluate(_injection_event(actor))
        assert decision is not None
        assert decision.action == AbuseAction.WARN
        assert decision.count  == 3

    def test_fifth_injection_triggers_suspend(self):
        actor    = "detector-test-3"
        decision = None
        for _ in range(5):
            decision = self.detector.evaluate(_injection_event(actor))
        assert decision.action == AbuseAction.SUSPEND
        assert decision.count  == 5

    def test_unwatched_event_type_returns_none(self):
        event = AuditEvent(
            event_type = AuditEventType.SESSION_CREATED,
            actor_id   = "detector-test-4",
            outcome    = "created",
        )
        assert self.detector.evaluate(event) is None

    def test_system_actor_is_ignored(self):
        event    = _injection_event("system")
        decision = None
        for _ in range(5):
            decision = self.detector.evaluate(event)
        assert decision is None


# tests/security/test_suspensions.py
import pytest

from backend.security import suspensions
from backend.security.redis_sync import get_client


@pytest.fixture(autouse=True)
def clean_keys():
    client = get_client()
    if client:
        for pattern in ("abuse:suspended:susp-test-*", "abuse:throttled:susp-test-*"):
            for key in client.scan_iter(match=pattern):
                client.delete(key)


class TestSuspensions:

    def test_suspend_and_check(self):
        actor = "susp-test-1"
        suspensions.suspend(actor, "test reason", duration_seconds=60)
        record = suspensions.is_suspended(actor)
        assert record is not None
        assert record["reason"] == "test reason"

    def test_unsuspended_actor_returns_none(self):
        assert suspensions.is_suspended("susp-test-2") is None

    def test_lift_suspension(self):
        actor = "susp-test-3"
        suspensions.suspend(actor, "test reason", duration_seconds=60)
        assert suspensions.lift_suspension(actor) is True
        assert suspensions.is_suspended(actor) is None

    def test_lift_nonexistent_returns_false(self):
        assert suspensions.lift_suspension("susp-test-4") is False

    def test_throttle_factor_default_and_set(self):
        actor = "susp-test-5"
        assert suspensions.get_throttle_factor(actor) == 1
        suspensions.throttle(actor, duration_seconds=60, factor=4)
        assert suspensions.get_throttle_factor(actor) == 4

    def test_list_suspended_includes_actor(self):
        actor = "susp-test-6"
        suspensions.suspend(actor, "reason", duration_seconds=60)
        assert actor in suspensions.list_suspended()


# tests/security/test_audit_abuse_integration.py
import pytest

from backend.audit.audit_logger import AuditLogger
from backend.audit.audit_models import AuditLogEntry
from backend.database.db import create_tables, SessionLocal
from backend.security import suspensions
from backend.security.redis_sync import get_client


@pytest.fixture(autouse=True)
def fresh_state():
    create_tables()
    db = SessionLocal()
    db.query(AuditLogEntry).delete()
    db.commit()
    db.close()

    client = get_client()
    if client:
        for pattern in (
            "abuse:signal:integration-user*",
            "abuse:suspended:integration-user*",
            "abuse:throttled:integration-user*",
        ):
            for key in client.scan_iter(match=pattern):
                client.delete(key)


class TestAuditAbuseIntegration:

    def setup_method(self):
        self.audit = AuditLogger()

    def test_repeated_injections_suspend_the_actor(self):
        actor = "integration-user-1"

        for _ in range(5):
            self.audit.injection_detected(
                session_id="sess-1", actor_id=actor,
                pattern_matched="ignore previous instructions", source="user_input",
            )

        assert suspensions.is_suspended(actor) is not None

        db = SessionLocal()
        try:
            event_types = [
                r.event_type for r in
                db.query(AuditLogEntry).filter(AuditLogEntry.actor_id == actor).all()
            ]
        finally:
            db.close()

        assert "security.injection_detected" in event_types
        assert "security.abuse_detected"      in event_types
        assert "account.suspended"            in event_types

    def test_two_injections_do_not_suspend(self):
        actor = "integration-user-2"

        for _ in range(2):
            self.audit.injection_detected(
                session_id="sess-1", actor_id=actor,
                pattern_matched="ignore previous instructions", source="user_input",
            )

        assert suspensions.is_suspended(actor) is None

    def test_lifting_suspension_logs_audit_event(self):
        actor = "integration-user-3"
        suspensions.suspend(actor, "manual test suspension", duration_seconds=60)

        self.audit.suspension_lifted(actor, reviewer_id="admin-1")

        db = SessionLocal()
        try:
            record = (
                db.query(AuditLogEntry)
                .filter(AuditLogEntry.event_type == "account.suspension_lifted")
                .first()
            )
        finally:
            db.close()

        assert record is not None
        assert record.actor_id    == "admin-1"
        assert record.resource_id == actor
```

Run the tests:

```bash
pytest tests/security/ -v --tb=short

# tests/security/test_abuse_tracker.py             5 passed
# tests/security/test_abuse_detector.py            5 passed
# tests/security/test_suspensions.py               6 passed
# tests/security/test_audit_abuse_integration.py   3 passed
```

---

<a id="what-you-built"></a>
## What you built today

```
backend/security/
  abuse_schema.py       11 thresholds across 6 audit event types,
                         3 actions: warn, throttle, suspend
  redis_sync.py         Synchronous Redis client for the audit-side path
  abuse_tracker.py       Sliding-window counters (Redis sorted sets)
  abuse_detector.py       Threshold evaluation, returns the worst action
                         crossed by a single event
  suspensions.py          Suspend / throttle state with TTL + admin helpers

backend/audit/audit_schema.py   +5 event types (abuse, throttle,
                                 suspend, suspension lifted, session
                                 blocked)
backend/audit/audit_logger.py   _emit() hook - every audit write now
                                 also runs through the abuse detector

backend/orchestrator/orchestrator.py  Suspended actors can't start
                                       new sessions (403 via PermissionError)
backend/middleware/rate_limiter.py     Throttled actors get role limit
                                        divided by 4 for 30 minutes

backend/routers/security.py
  GET  /security/suspensions               List active suspensions
  POST /security/suspensions/{id}/lift     Lift one early
  GET  /security/throttle/{id}             Current throttle factor
  GET  /security/abuse-events              Recent abuse/account events
  GET  /security/abuse-summary/{id}        One actor's abuse history

Tests: 19 (5 tracker + 5 detector + 6 suspensions + 3 integration)
```

What gets caught now that wasn't before:

```
5 injection attempts in 10 minutes      -> WARN at 3, SUSPEND at 5
3 privilege escalation attempts in 30m  -> WARN at 2, SUSPEND at 3
2 path traversal attempts in 60m        -> WARN at 1, SUSPEND at 2
10 blocked goals in 15 minutes          -> WARN at 5, THROTTLE at 10
5 blocked agent spawns in 15 minutes    -> THROTTLE
8 blocked messages in 15 minutes        -> THROTTLE
5 HITL rejections in 60 minutes         -> WARN at 3, THROTTLE at 5
```

What it does not catch: the same person spreading these attempts across many different accounts, each staying under every threshold. Per-actor sliding windows can't see that - it requires correlating signals across actors by IP, device, or timing, which is a cross-account analysis problem for a later hardening day.

---

<a id="checklist"></a>
## Day 18 checklist

- [ ] `backend/security/abuse_schema.py` created with 11 thresholds across 6 event types
- [ ] `backend/security/redis_sync.py` created - synchronous client, fails open if Redis is down
- [ ] `backend/security/abuse_tracker.py` created - sliding-window counters via Redis sorted sets
- [ ] `backend/security/abuse_detector.py` created - returns the worst action crossed
- [ ] `backend/security/suspensions.py` created - suspend/throttle state with TTL
- [ ] `backend/audit/audit_schema.py` updated with 5 new event types
- [ ] `backend/audit/audit_logger.py` updated - `_emit()` added, all `_writer.write(` calls replaced
- [ ] `backend/audit/audit_logger.py` - `session_blocked_suspended()` and `suspension_lifted()` added
- [ ] `backend/orchestrator/orchestrator.py` - suspension check added before `create_session()`
- [ ] `backend/middleware/rate_limiter.py` - `_effective_limit()` applies the throttle factor
- [ ] `backend/routers/security.py` created with 5 endpoints
- [ ] `backend/main.py` updated to version `"0.6.0"` with security router
- [ ] `tests/security/test_abuse_tracker.py` - 5 tests pass
- [ ] `tests/security/test_abuse_detector.py` - 5 tests pass
- [ ] `tests/security/test_suspensions.py` - 6 tests pass
- [ ] `tests/security/test_audit_abuse_integration.py` - 3 tests pass
- [ ] Manually trigger 5 `injection_detected` calls for one actor, confirm `GET /security/suspensions` shows them suspended
- [ ] `POST /security/suspensions/{id}/lift?reviewer_id=...` clears the suspension and logs `account.suspension_lifted`
- [ ] Trigger 10 `goal_blocked` calls for one actor, confirm `GET /security/throttle/{id}` returns `4`

---

*Day 18 done. Audit events are no longer just a record - they're a live feed that throttles or suspends an actor before a probing pattern turns into a real breach. Day 19 covers secrets management and credential isolation: making sure agents that need API keys and database credentials can't leak them, hardcode them, or pass them to each other across the message bus.*

---

**Repository:** [GuntruTirupathamma/AI-Governance](https://github.com/GuntruTirupathamma/AI-Governance)
**Series:** AI Governance Engineering from Scratch
**Next:** `DAY19.md` - Secrets Management and Credential Isolation
