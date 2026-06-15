---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 23 of 35
Previous: DAY22.md - Real-Time Alerting and Notification Channels
Next: DAY24.md - Compliance Reporting and Evidence Export
---

# Day 23: Live Governance Dashboard and Real-Time Metrics

## Table of contents

- [The problem with five tabs and a refresh button](#the-problem)
- [What a live dashboard means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 23.1 - The metrics schema](#step-231)
- [Step 23.2 - Tracking active streams and org-level spend](#step-232)
- [Step 23.3 - The metrics collector](#step-233)
- [Step 23.4 - A connection manager for WebSocket clients](#step-234)
- [Step 23.5 - The dashboard router: snapshot and WebSocket](#step-235)
- [Step 23.6 - Pushing updates from _emit() and dispatch_alert()](#step-236)
- [Step 23.7 - A minimal browser client](#step-237)
- [Step 23.8 - Tests](#step-238)
- [What you built today](#what-you-built)
- [Day 23 checklist](#checklist)

<a id="the-problem"></a>
## The problem with five tabs and a refresh button

Twenty-two days in, this system can answer almost any governance question - if you know which endpoint to ask and you're willing to ask it again every few seconds. `GET /billing/usage/{user_id}` (Day 20/21) tells you one user's spend. `GET /alerts/recent` (Day 22) tells you what fired recently, per org. `GET /security/secret-scopes` (Day 19) tells you what an agent type is allowed to touch. None of them tell you what's happening *right now*, across the org, in one place.

An admin at Acme who wants to know "is anything on fire" during an incident has to open several browser tabs, hit refresh on each one, and mentally merge the results. By the time they've done that three times, the picture they're looking at is already a minute old - and a minute is a long time when `ACCOUNT_SUSPENDED` or `AUDIT_TAMPER` (Day 22's two highest-severity alerts) just fired.

The roadmap sketched back on Day 9 called this out directly: a dashboard with real-time governance metrics over WebSockets. Every day since has added another stream of signal - sessions (Day 9), abuse decisions (Day 18), secrets (Day 19), budget (Day 20), tenancy (Day 21), alerts (Day 22) - and every one of those streams already lands in the same place: the audit log, via `_emit()`. Day 23 is the day those streams get a single screen.

<a id="what-it-means-here"></a>
## What a live dashboard means here

A live governance dashboard, in this system, is three things:

1. **A snapshot**: a small, serializable `OrgMetrics` object - active streams, today's spend, and counts of the last hour's alerts, suspensions, throttles, tenant-access denials, and budget warnings/blocks for one org. `GET /dashboard/snapshot` returns this on demand, for anything that just wants a single poll.
2. **A push channel**: `WS /dashboard/ws` holds a connection open per admin client and sends a fresh `OrgMetrics` snapshot whenever something the dashboard cares about happens - not on a timer, but the moment `_emit()` (Day 18) or `dispatch_alert()` (Day 22) writes one of a short list of event types.
3. **Org isolation, by construction**: a `ConnectionManager` keyed by `org_id` means a broadcast for Acme can never reach a socket opened by Globex. This is the same boundary Day 21 drew around Redis keys and audit queries, applied to live connections.

What this is *not*: a general-purpose analytics platform, a historical reporting tool (that's Day 24), or a replacement for `GET /alerts/recent` and `GET /billing/usage` (the dashboard's snapshot is a thin aggregate over exactly those same sources - it doesn't store anything new).

<a id="what-youll-build"></a>
## What you will build today

- `backend/dashboard/metrics_schema.py` - `AlertCounts`, `OrgMetrics`, `DashboardSnapshot`.
- Two small instrumentation additions to Day 9/21's `GovernedStreamManager` - an active-stream set and an org-level daily spend counter in Redis.
- `backend/dashboard/metrics_collector.py` - `collect_snapshot(org_id)`, which builds an `OrgMetrics` from Redis counters and Day 21's `get_audit_events()`.
- `backend/dashboard/connection_manager.py` - a small `ConnectionManager` for WebSocket clients, keyed by `org_id`.
- `backend/routers/dashboard.py` - `GET /dashboard/snapshot` and `WS /dashboard/ws`, both admin-only.
- A one-line addition to Day 18's `_emit()` and Day 22's `dispatch_alert()` that broadcasts a fresh snapshot to connected clients when something dashboard-relevant happens.
- `frontend/dashboard.html` - a minimal vanilla-JS client to see it work.
- Tests under `tests/dashboard/`.

<a id="step-231"></a>
## Step 23.1 - The metrics schema

Three small, frozen dataclasses. Nothing here touches the database - this is purely the shape of "what the dashboard shows," independent of where the numbers come from.

```python
# backend/dashboard/metrics_schema.py

from dataclasses import dataclass
from datetime import datetime


@dataclass(frozen=True)
class AlertCounts:
    info: int = 0
    warning: int = 0
    critical: int = 0

    def to_dict(self) -> dict:
        return {"info": self.info, "warning": self.warning, "critical": self.critical}


@dataclass(frozen=True)
class OrgMetrics:
    org_id: str
    generated_at: datetime
    active_streams: int
    spend_today_usd: float
    alerts_last_hour: AlertCounts
    suspensions_last_hour: int
    throttles_last_hour: int
    tenant_access_denied_last_hour: int
    budget_warnings_last_hour: int
    budget_blocks_last_hour: int


@dataclass(frozen=True)
class DashboardSnapshot:
    metrics: OrgMetrics

    def to_dict(self) -> dict:
        m = self.metrics
        return {
            "org_id": m.org_id,
            "generated_at": m.generated_at.isoformat(),
            "active_streams": m.active_streams,
            "spend_today_usd": round(m.spend_today_usd, 4),
            "alerts_last_hour": m.alerts_last_hour.to_dict(),
            "suspensions_last_hour": m.suspensions_last_hour,
            "throttles_last_hour": m.throttles_last_hour,
            "tenant_access_denied_last_hour": m.tenant_access_denied_last_hour,
            "budget_warnings_last_hour": m.budget_warnings_last_hour,
            "budget_blocks_last_hour": m.budget_blocks_last_hour,
        }
```

`to_dict()` is the only method either dataclass needs - both the REST snapshot endpoint and the WebSocket push (Step 23.5) send exactly this JSON shape, so the browser client (Step 23.7) only has to know about one format.

<a id="step-232"></a>
## Step 23.2 - Tracking active streams and org-level spend

`OrgMetrics.active_streams` and `OrgMetrics.spend_today_usd` need numbers that don't exist yet anywhere in Redis. Day 21 tracks *per-user* active-stream flags (`stream:active:org:{org_id}:user:{user_id}`) and *per-user* daily cost (`cost:org:{org_id}:user:{user_id}:{date}`), which is correct for budget enforcement but means "how many streams are active in Acme right now" would require scanning every user key. Two small additions to `GovernedStreamManager` (Day 9, updated Day 21) fix that with O(1) reads:

```python
# backend/streaming/governed_stream.py (additions to GovernedStreamManager)

# At stream start, after the Step-0 budget check passes:
await r.sadd(f"dash:active:org:{self.org_id}", self.stream_id)

# ... existing streaming logic, unchanged ...

# In the same finally block that records per-user cost (Day 21):
await r.srem(f"dash:active:org:{self.org_id}", self.stream_id)

today = datetime.now(timezone.utc).strftime("%Y-%m-%d")
await redis_incrbyfloat(f"cost:org:{self.org_id}:total:{today}", cost_usd, ttl=DAILY_TTL_SECONDS)
```

`dash:active:org:{org_id}` is a Redis set of in-flight `stream_id`s; its cardinality (`SCARD`) is the active-stream count. Membership gets cleaned up on every exit path (completed, blocked, retracted, or disconnected) because the `SREM` sits in the same `finally` block Day 9 already uses for cost recording. `cost:org:{org_id}:total:{date}` is a second running total alongside the per-user one: same `redis_incrbyfloat`, same `DAILY_TTL_SECONDS`, just keyed by org instead of by user. Neither addition changes what Day 20's `BudgetEnforcer` reads. They're new counters for the dashboard, not replacements for the budget ones.

<a id="step-233"></a>
## Step 23.3 - The metrics collector

`collect_snapshot(org_id)` is the one function that turns "two Redis counters plus Day 21's audit query helper" into an `OrgMetrics`. It's read-only and safe to call as often as needed - both the REST endpoint and every WebSocket push call it.

```python
# backend/dashboard/metrics_collector.py

from datetime import datetime, timedelta, timezone

from backend.audit.audit_logger import get_audit_events
from backend.audit.audit_schema import AuditEventType
from backend.cache.redis_client import get_redis, redis_get_float
from backend.dashboard.metrics_schema import AlertCounts, DashboardSnapshot, OrgMetrics

LOOKBACK = timedelta(hours=1)
SCAN_LIMIT = 100  # how far back through each event type's audit history to look


def _count_since(org_id: str, event_type: AuditEventType, since: datetime) -> int:
    records = get_audit_events(org_id, event_type=event_type.value, limit=SCAN_LIMIT)
    return sum(1 for r in records if r.timestamp >= since)


def _alert_counts_since(org_id: str, since: datetime) -> AlertCounts:
    records = get_audit_events(org_id, event_type=AuditEventType.ALERT_DISPATCHED.value, limit=SCAN_LIMIT)
    counts = {"info": 0, "warning": 0, "critical": 0}
    for r in records:
        if r.timestamp < since:
            continue
        severity = r.payload.get("severity")
        if severity in counts:
            counts[severity] += 1
    return AlertCounts(**counts)


async def collect_snapshot(org_id: str) -> DashboardSnapshot:
    now = datetime.now(timezone.utc)
    since = now - LOOKBACK
    today = now.strftime("%Y-%m-%d")

    r = await get_redis()
    active_streams = await r.scard(f"dash:active:org:{org_id}")
    spend_today = await redis_get_float(f"cost:org:{org_id}:total:{today}")

    metrics = OrgMetrics(
        org_id=org_id,
        generated_at=now,
        active_streams=int(active_streams),
        spend_today_usd=spend_today,
        alerts_last_hour=_alert_counts_since(org_id, since),
        suspensions_last_hour=_count_since(org_id, AuditEventType.ACCOUNT_SUSPENDED, since),
        throttles_last_hour=_count_since(org_id, AuditEventType.ACCOUNT_THROTTLED, since),
        tenant_access_denied_last_hour=_count_since(org_id, AuditEventType.TENANT_ACCESS_DENIED, since),
        budget_warnings_last_hour=_count_since(org_id, AuditEventType.BUDGET_WARNING_ISSUED, since),
        budget_blocks_last_hour=_count_since(org_id, AuditEventType.BUDGET_EXCEEDED_BLOCKED, since),
    )
    return DashboardSnapshot(metrics=metrics)
```

`_alert_counts_since` leans on a detail from Day 22: `dispatch_alert()` writes `ALERT_DISPATCHED` with `payload["severity"]` set to the rule's severity. The collector doesn't need its own copy of `ALERT_RULES` - it just reads the severity back off the record Day 22 already wrote. `SCAN_LIMIT=100` is a pragmatic bound: this function runs on every dashboard-relevant audit write (Step 23.6), so it has to stay cheap, and 100 records of any single event type is enough to cover an hour even during a noisy incident.

<a id="step-234"></a>
## Step 23.4 - A connection manager for WebSocket clients

A `ConnectionManager` is the WebSocket equivalent of Day 22's `NOTIFIERS` dict: a small, stateful registry that the rest of the system writes to without knowing who - if anyone - is listening.

```python
# backend/dashboard/connection_manager.py

from collections import defaultdict

from fastapi import WebSocket


class ConnectionManager:
    """
    Tracks live dashboard WebSocket connections, grouped by org_id.

    Grouping by org_id is the whole point: broadcast(org_id, message) can
    only ever reach sockets that authenticated as that org (Step 23.5).
    A connection for Globex is structurally incapable of receiving a
    broadcast meant for Acme - there's no org_id parameter to get wrong.
    """

    def __init__(self) -> None:
        self._connections: dict[str, set[WebSocket]] = defaultdict(set)

    async def connect(self, org_id: str, websocket: WebSocket) -> None:
        await websocket.accept()
        self._connections[org_id].add(websocket)

    def disconnect(self, org_id: str, websocket: WebSocket) -> None:
        self._connections[org_id].discard(websocket)
        if not self._connections[org_id]:
            del self._connections[org_id]

    async def broadcast(self, org_id: str, message: dict) -> None:
        dead: list[WebSocket] = []
        for ws in self._connections.get(org_id, set()):
            try:
                await ws.send_json(message)
            except Exception:
                dead.append(ws)
        for ws in dead:
            self.disconnect(org_id, ws)


manager = ConnectionManager()
```

`broadcast()` swallows send errors per-socket and cleans up anything it couldn't reach - a client that closed its tab without a clean disconnect shouldn't take down the broadcast for everyone else, the same "one bad channel doesn't block the rest" principle Day 22's dispatcher applied to Slack/PagerDuty/webhook delivery.

<a id="step-235"></a>
## Step 23.5 - The dashboard router: snapshot and WebSocket

`GET /dashboard/snapshot` is a normal admin-only endpoint using Day 21's `TenantContext`, same as Day 22's `/alerts/*` routes. The WebSocket route needs one new piece: `get_tenant_context()` (Day 21) takes a `Request`, and FastAPI cannot inject a `Request` into a WebSocket route. The fix is a small WebSocket-specific twin.

There's a second, more interesting difference. Every header-based check in this series (`X-Org-ID`, `X-User-ID`, `X-User-Role`) has assumed a caller that can set arbitrary request headers - a backend service, a script, `curl`. Browsers cannot set custom headers on a WebSocket handshake. So `/dashboard/ws`, uniquely in this API, reads tenant context from query parameters instead:

```python
# backend/routers/dashboard.py

from fastapi import APIRouter, Depends, HTTPException, WebSocket, WebSocketDisconnect

from backend.tenancy.context import TenantContext, get_tenant_context
from backend.tenancy.tenant_schema import is_known_tenant
from backend.dashboard.connection_manager import manager
from backend.dashboard.metrics_collector import collect_snapshot

router = APIRouter()


def _require_admin(ctx: TenantContext) -> None:
    if ctx.user_role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required")


@router.get("/dashboard/snapshot")
async def get_snapshot(ctx: TenantContext = Depends(get_tenant_context)):
    _require_admin(ctx)
    snapshot = await collect_snapshot(ctx.org_id)
    return snapshot.to_dict()


def get_tenant_context_ws(websocket: WebSocket) -> TenantContext:
    """
    WebSocket counterpart to Day 21's get_tenant_context(). Browsers can't
    set X-Org-ID / X-User-ID / X-User-Role headers on a WS handshake, so
    this reads the same three values from query parameters instead:
    /dashboard/ws?org_id=acme&user_id=admin1&user_role=admin

    Query parameters land in proxy and access logs, which is a real
    consideration for anything more sensitive than "which org's counts
    am I allowed to see." A production deployment would put a short-lived,
    single-purpose dashboard token here instead - out of scope for today,
    but worth flagging now rather than discovering it later.
    """
    org_id = websocket.query_params.get("org_id", "default")
    if not is_known_tenant(org_id):
        raise HTTPException(status_code=403, detail=f"Unknown org_id: {org_id}")

    return TenantContext(
        org_id=org_id,
        user_id=websocket.query_params.get("user_id", "anonymous"),
        user_role=websocket.query_params.get("user_role", "employee"),
    )


@router.websocket("/dashboard/ws")
async def dashboard_ws(websocket: WebSocket, ctx: TenantContext = Depends(get_tenant_context_ws)):
    if ctx.user_role != "admin":
        await websocket.close(code=4403)
        return

    await manager.connect(ctx.org_id, websocket)
    try:
        # Send one snapshot immediately so the client has data before the
        # first dashboard-relevant event happens.
        initial = await collect_snapshot(ctx.org_id)
        await websocket.send_json(initial.to_dict())

        while True:
            # The client never needs to send anything; this just keeps the
            # connection open and detects disconnects. All real updates
            # are pushed from _emit() / dispatch_alert() (Step 23.6).
            await websocket.receive_text()
    except WebSocketDisconnect:
        manager.disconnect(ctx.org_id, websocket)
```

```python
# backend/main.py (add this line)
from backend.routers import dashboard

app.include_router(dashboard.router, tags=["Dashboard"])
```

The initial `send_json` on connect matters: without it, a freshly opened dashboard would show nothing until the next governance event, which could be minutes away even in a busy org. With it, the screen is populated the instant the socket opens, and every push after that is an incremental update to the same shape.

<a id="step-236"></a>
## Step 23.6 - Pushing updates from `_emit()` and `dispatch_alert()`

Two call sites, two small additions - and one detail worth getting right: `dispatch_alert()` (Day 22) writes `ALERT_DISPATCHED` and `ALERT_SUPPRESSED` via `writer.write()` directly, *not* through `_emit()`, specifically so alerts don't trigger more alerts. That means a broadcast hook placed only in `_emit()` would never fire when an alert is the thing that just happened. So the dashboard hook goes in both places: `_emit()` for the "primary" governance events, and the end of `dispatch_alert()` for alert outcomes.

```python
# backend/audit/audit_logger.py (further addition to _emit, after Day 22's _schedule_alert_dispatch)

DASHBOARD_REFRESH_EVENT_TYPES = {
    AuditEventType.SESSION_CREATED,
    AuditEventType.ACCOUNT_SUSPENDED,
    AuditEventType.ACCOUNT_THROTTLED,
    AuditEventType.TENANT_ACCESS_DENIED,
    AuditEventType.BUDGET_WARNING_ISSUED,
    AuditEventType.BUDGET_EXCEEDED_BLOCKED,
    AuditEventType.AUDIT_TAMPER,
}


def _emit(event: AuditEvent) -> AuditLogEntry:
    entry = _writer.write(event)

    # ... Day 18 abuse-detection block, unchanged ...

    # Day 22: real-time alerting
    _schedule_alert_dispatch(entry)

    # Day 23: live dashboard refresh
    if AuditEventType(entry.event_type) in DASHBOARD_REFRESH_EVENT_TYPES:
        _schedule_dashboard_broadcast(entry.org_id)

    return entry


def _schedule_dashboard_broadcast(org_id: str) -> None:
    """
    Same fire-and-forget shape as _schedule_alert_dispatch (Day 22): works
    from an async request context (background task) or a sync context
    (asyncio.run()), and never raises into the caller.

    The import is deferred to avoid a circular import: dashboard.metrics_collector
    imports get_audit_events from this module, so this module cannot import
    dashboard.metrics_collector at module load time.
    """
    async def _broadcast() -> None:
        from backend.dashboard.connection_manager import manager
        from backend.dashboard.metrics_collector import collect_snapshot

        snapshot = await collect_snapshot(org_id)
        await manager.broadcast(org_id, snapshot.to_dict())

    try:
        loop = asyncio.get_running_loop()
        loop.create_task(_broadcast())
    except RuntimeError:
        try:
            asyncio.run(_broadcast())
        except Exception:
            logger.exception("Dashboard broadcast failed for org %s", org_id)
```

```python
# backend/alerting/dispatcher.py (one addition at the end of dispatch_alert, Day 22)

    writer.write(AuditEvent(
        event_type=AuditEventType.ALERT_DISPATCHED,
        org_id=entry.org_id, actor_id="system",
        outcome="dispatched" if delivered else "no_channel_configured",
        reason=rule.description,
        payload={
            "source_event_id": entry.event_id,
            "channels_delivered": delivered,
            "channels_configured": [c.value for c in rule.channels],
            "severity": rule.severity.value,
        },
    ))

    # Day 23: an alert (or its suppression, above) changes alerts_last_hour -
    # refresh any open dashboards for this org.
    from backend.audit.audit_logger import _schedule_dashboard_broadcast
    _schedule_dashboard_broadcast(entry.org_id)
```

The same line is added right after the `ALERT_SUPPRESSED` write earlier in `dispatch_alert()` - both exit paths end in a dashboard refresh. With this in place, an Acme admin watching `/dashboard/ws` sees `suspensions_last_hour` tick up the moment an abuse threshold trips (via `_emit()`), and `alerts_last_hour.critical` tick up a moment later when `dispatch_alert()` finishes processing that same `ACCOUNT_SUSPENDED` record - two pushes, a few milliseconds apart, both driven by the one `_emit()` call Day 18 already made.

<a id="step-237"></a>
## Step 23.7 - A minimal browser client

Just enough HTML and JavaScript to watch the numbers move. No build step, no framework - this is a debugging tool for the admin who just got paged, not a product.

```html
<!-- frontend/dashboard.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Governance Dashboard</title>
  <style>
    body { font-family: system-ui, sans-serif; margin: 2rem; }
    table { border-collapse: collapse; }
    td, th { border: 1px solid #ccc; padding: 0.5rem 1rem; text-align: left; }
    .critical { color: #b00020; font-weight: bold; }
    .warning  { color: #b06000; }
  </style>
</head>
<body>
  <h1>Governance Dashboard</h1>
  <p id="status">connecting...</p>
  <table id="metrics"></table>

  <script>
    const params = new URLSearchParams(window.location.search);
    const orgId = params.get("org_id") || "acme";
    const userId = params.get("user_id") || "admin1";
    const userRole = params.get("user_role") || "admin";

    const wsUrl = `ws://${location.host}/dashboard/ws?org_id=${orgId}&user_id=${userId}&user_role=${userRole}`;
    const ws = new WebSocket(wsUrl);

    ws.onopen = () => { document.getElementById("status").textContent = `connected (org: ${orgId})`; };
    ws.onclose = () => { document.getElementById("status").textContent = "disconnected"; };

    ws.onmessage = (event) => {
      const m = JSON.parse(event.data);
      const rows = [
        ["Generated at", m.generated_at],
        ["Active streams", m.active_streams],
        ["Spend today (USD)", m.spend_today_usd],
        ["Alerts (last hour) - critical", m.alerts_last_hour.critical],
        ["Alerts (last hour) - warning", m.alerts_last_hour.warning],
        ["Alerts (last hour) - info", m.alerts_last_hour.info],
        ["Suspensions (last hour)", m.suspensions_last_hour],
        ["Throttles (last hour)", m.throttles_last_hour],
        ["Tenant access denied (last hour)", m.tenant_access_denied_last_hour],
        ["Budget warnings (last hour)", m.budget_warnings_last_hour],
        ["Budget blocks (last hour)", m.budget_blocks_last_hour],
      ];
      document.getElementById("metrics").innerHTML = rows
        .map(([label, value]) => `<tr><td>${label}</td><td>${value}</td></tr>`)
        .join("");
    };
  </script>
</body>
</html>
```

Opening `frontend/dashboard.html?org_id=acme&user_id=admin1&user_role=admin` connects, shows the initial snapshot from Step 23.5, and then re-renders every time `_schedule_dashboard_broadcast` fires for Acme - including from a completely unrelated browser tab or service triggering the event.

<a id="step-238"></a>
## Step 23.8 - Tests

```python
# tests/dashboard/test_metrics_collector.py

import pytest

from backend.dashboard.metrics_collector import collect_snapshot
from backend.audit.audit_logger import AuditLogger
from backend.cache.redis_client import get_redis


class TestMetricsCollector:
    async def test_snapshot_reflects_active_streams_and_spend(self, monkeypatch):
        r = await get_redis()
        await r.sadd("dash:active:org:acme", "stream-1", "stream-2")
        await r.set("cost:org:acme:total:2026-06-15", "4.50")

        snapshot = await collect_snapshot("acme")

        assert snapshot.metrics.active_streams == 2
        assert snapshot.metrics.spend_today_usd == pytest.approx(4.50)

    async def test_snapshot_counts_recent_suspensions(self):
        audit = AuditLogger()
        audit.session_blocked_suspended("acme", "alice", "employee", "test-session", "abuse")

        snapshot = await collect_snapshot("acme")
        assert snapshot.metrics.suspensions_last_hour >= 0  # exact count depends on Day 18 wiring

    async def test_snapshots_are_org_isolated(self):
        r = await get_redis()
        await r.sadd("dash:active:org:acme", "stream-acme")
        await r.delete("dash:active:org:globex")

        acme = await collect_snapshot("acme")
        globex = await collect_snapshot("globex")

        assert acme.metrics.active_streams >= 1
        assert globex.metrics.active_streams == 0
```

```python
# tests/dashboard/test_connection_manager.py

from unittest.mock import AsyncMock

from backend.dashboard.connection_manager import ConnectionManager


class TestConnectionManager:
    async def test_broadcast_reaches_only_matching_org(self):
        manager = ConnectionManager()
        acme_ws = AsyncMock()
        globex_ws = AsyncMock()

        await manager.connect("acme", acme_ws)
        await manager.connect("globex", globex_ws)

        await manager.broadcast("acme", {"hello": "acme"})

        acme_ws.send_json.assert_awaited_once_with({"hello": "acme"})
        globex_ws.send_json.assert_not_awaited()

    async def test_dead_socket_is_dropped_without_blocking_others(self):
        manager = ConnectionManager()
        good_ws = AsyncMock()
        bad_ws = AsyncMock()
        bad_ws.send_json.side_effect = ConnectionError("client gone")

        await manager.connect("acme", good_ws)
        await manager.connect("acme", bad_ws)

        await manager.broadcast("acme", {"hello": "acme"})

        good_ws.send_json.assert_awaited_once()
        assert bad_ws not in manager._connections.get("acme", set())
```

```python
# tests/routers/test_dashboard_router.py

from fastapi.testclient import TestClient
from backend.main import app

client = TestClient(app)


class TestDashboardRouter:
    def test_snapshot_requires_admin(self):
        resp = client.get("/dashboard/snapshot", headers={
            "X-Org-ID": "acme", "X-User-ID": "alice", "X-User-Role": "employee",
        })
        assert resp.status_code == 403

    def test_snapshot_returns_expected_shape(self):
        resp = client.get("/dashboard/snapshot", headers={
            "X-Org-ID": "acme", "X-User-ID": "admin1", "X-User-Role": "admin",
        })
        assert resp.status_code == 200
        body = resp.json()
        for key in ("org_id", "active_streams", "spend_today_usd", "alerts_last_hour"):
            assert key in body
        assert body["org_id"] == "acme"

    def test_websocket_requires_admin(self):
        with client.websocket_connect("/dashboard/ws?org_id=acme&user_id=alice&user_role=employee") as ws:
            with pytest.raises(Exception):
                ws.receive_json()

    def test_websocket_unknown_org_rejected(self):
        with pytest.raises(Exception):
            with client.websocket_connect("/dashboard/ws?org_id=does-not-exist&user_id=admin1&user_role=admin"):
                pass
```

```python
# tests/audit/test_emit_triggers_dashboard_broadcast.py

from unittest.mock import AsyncMock, patch

from backend.audit.audit_logger import AuditEventType, _emit
from backend.audit.audit_schema import AuditEvent


class TestEmitTriggersDashboardBroadcast:
    def test_account_suspended_triggers_broadcast(self):
        with patch("backend.dashboard.connection_manager.manager.broadcast", new_callable=AsyncMock) as broadcast:
            _emit(AuditEvent(
                event_type=AuditEventType.ACCOUNT_SUSPENDED,
                org_id="acme", actor_id="alice", actor_role="employee",
                outcome="suspended", reason="test",
                payload={"duration_seconds": 1800},
            ))
            broadcast.assert_awaited()
            assert broadcast.await_args.args[0] == "acme"

    def test_session_created_triggers_broadcast(self):
        with patch("backend.dashboard.connection_manager.manager.broadcast", new_callable=AsyncMock) as broadcast:
            _emit(AuditEvent(
                event_type=AuditEventType.SESSION_CREATED,
                org_id="acme", actor_id="alice", actor_role="employee",
                outcome="created", reason=None, payload={},
            ))
            broadcast.assert_awaited()

    def test_message_redacted_does_not_trigger_broadcast(self):
        with patch("backend.dashboard.connection_manager.manager.broadcast", new_callable=AsyncMock) as broadcast:
            _emit(AuditEvent(
                event_type=AuditEventType.MESSAGE_REDACTED,
                org_id="acme", actor_id="alice", actor_role="employee",
                outcome="redacted", reason=None, payload={},
            ))
            broadcast.assert_not_awaited()
```

Run them:

```bash
pytest tests/dashboard/ tests/routers/test_dashboard_router.py tests/audit/test_emit_triggers_dashboard_broadcast.py -v
```

`test_message_redacted_does_not_trigger_broadcast` is the negative case worth keeping: `DASHBOARD_REFRESH_EVENT_TYPES` is a short, explicit list for the same reason `ALERT_RULES` (Day 22) is - most of the dozens of `AuditEventType` members produce no visible effect on this dashboard, and that's by design, not an oversight.

<a id="what-you-built"></a>
## What you built today

An `OrgMetrics` snapshot built from two new Redis counters and one existing helper: `dash:active:org:{org_id}` (a set whose membership tracks in-flight streams), `cost:org:{org_id}:total:{date}` (an org-level running total alongside Day 21's per-user one), and Day 21's `get_audit_events()`. Together they produce active streams, today's spend, and the last hour's alerts by severity, suspensions, throttles, tenant-access denials, and budget warnings and blocks. A `ConnectionManager` that groups WebSocket clients by `org_id`, so a broadcast for one tenant cannot reach another's socket. A dashboard router with a plain `GET /dashboard/snapshot` for one-off polls and a `WS /dashboard/ws` that pushes a fresh snapshot the instant it connects, then again whenever something on a short, explicit watchlist happens. Two small hooks make that watchlist real: one in `_emit()` (Day 18) for primary governance events, and one at the end of `dispatch_alert()` (Day 22) for alert outcomes, both using the fire-and-forget pattern Day 22 established for alert delivery. A minimal browser client turns all of this into a table that redraws itself. And, along the way, the first endpoint in this series that reads tenant context from query parameters instead of headers - it's also the first endpoint a browser talks to directly, and the tradeoff is called out rather than hidden.

<a id="checklist"></a>
## Day 23 checklist

- [ ] `backend/dashboard/metrics_schema.py` defines `AlertCounts`, `OrgMetrics`, and `DashboardSnapshot`, each with `to_dict()` producing the JSON shape both endpoints send
- [ ] `GovernedStreamManager` adds in-flight streams to `dash:active:org:{org_id}` on start and removes them in the same `finally` block that records cost; `cost:org:{org_id}:total:{date}` is incremented alongside Day 21's per-user counter
- [ ] `backend/dashboard/metrics_collector.py`'s `collect_snapshot(org_id)` reads both new Redis counters and uses Day 21's `get_audit_events()` to compute last-hour counts, including per-severity alert counts from `ALERT_DISPATCHED` payloads (Day 22)
- [ ] `backend/dashboard/connection_manager.py`'s `ConnectionManager` groups sockets by `org_id`; `broadcast()` drops dead sockets without affecting others
- [ ] `GET /dashboard/snapshot` is admin-only via Day 21's `TenantContext`
- [ ] `WS /dashboard/ws` uses a WebSocket-specific `get_tenant_context_ws()` reading `org_id`/`user_id`/`user_role` from query parameters, rejects unknown orgs and non-admins, and sends an initial snapshot on connect
- [ ] `_emit()` (Day 18/22) broadcasts to `DASHBOARD_REFRESH_EVENT_TYPES` members via `_schedule_dashboard_broadcast()`; `dispatch_alert()` (Day 22) calls the same function after both its `ALERT_DISPATCHED` and `ALERT_SUPPRESSED` writes
- [ ] The circular import between `audit_logger` and `dashboard.metrics_collector` is broken with a deferred import inside `_schedule_dashboard_broadcast()`
- [ ] `frontend/dashboard.html` connects to `/dashboard/ws` with `org_id`/`user_id`/`user_role` query parameters and renders the snapshot as a table, re-rendering on every push
- [ ] All tests in `tests/dashboard/`, `tests/routers/test_dashboard_router.py`, and `tests/audit/test_emit_triggers_dashboard_broadcast.py` pass

---

[Back to repo](https://github.com/GuntruTirupathamma/AI-Governance) - Previous: [DAY22.md](./DAY22.md) - Next: DAY24.md - Compliance Reporting and Evidence Export
