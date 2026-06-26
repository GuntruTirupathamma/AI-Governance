---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 32 of 35
Previous: DAY31.md - Governance Drift Detection and Config Auditing
Next: DAY33.md - Governance Reporting for Compliance and Legal Teams
---

# Day 32: SIEM Integration and Audit Log Export

## Table of contents

- [The problem with audit data that only the app can read](#the-problem)
- [What SIEM integration means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 32.1 - SIEM config schema](#step-321)
- [Step 32.2 - Per-org SIEM config registry](#step-322)
- [Step 32.3 - Event formatters](#step-323)
- [Step 32.4 - The forwarder](#step-324)
- [Step 32.5 - Wiring into the audit logger](#step-325)
- [Step 32.6 - Bulk export for historical data](#step-326)
- [Step 32.7 - SIEM admin endpoints](#step-327)
- [Step 32.8 - Tests](#step-328)
- [What you built today](#what-you-built)
- [Day 32 checklist](#checklist)

<a id="the-problem"></a>
## The problem with audit data that only the app can read

The quarterly SOC 2 audit landed on a Friday afternoon. The compliance team needed all CRITICAL-severity audit events from the last 90 days, across every tenant, in a format that could be loaded into their SIEM. What that meant in practice was that someone on the engineering team spent the following Monday morning logged into the app, querying org by org through the REST API, exporting JSON, converting it to CSV, and uploading it to a shared drive. The CISO's audit report described this as "manual evidence gathering," which is a polite way of saying we had no automated export pipeline.

The SIEM already had logs from the load balancer, the WAF, the database slow query log, the container orchestrator, and the VPN. It had everything except the governance layer, which is the part that matters most for AI security. An `AUDIT_TAMPER_DETECTED` event that fired six weeks ago was invisible to the security team unless someone told them to go look for it. A policy freeze that locked a customer's budget for twenty minutes existed only inside a Slack thread and the in-process incident registry, which resets on every deploy.

This is the gap between an operational system and an auditable one. The governance machinery built over the last 31 days produces a lot of signals. None of them flow anywhere outside the application unless something is actively consuming the REST API. A SIEM integration fixes that permanently, in two ways: a real-time forwarder pushes new events to the SIEM as they happen, and a bulk export endpoint lets a new SIEM connection catch up on history without requiring an engineer to do it manually.

<a id="what-it-means-here"></a>
## What SIEM integration means here

Three target types cover most enterprise security stacks. Splunk's HTTP Event Collector (HEC) accepts JSON events over HTTPS with a bearer token in the `Authorization` header. Datadog's Logs API takes a slightly different JSON shape, using `ddsource`, `ddtags`, and a flat field structure that its indexer expects. A generic webhook target covers everything else: any system that accepts JSON over HTTP with a configurable auth header, from Elastic to Sumo Logic to a custom log aggregator.

Each target gets its own event formatter that maps an `AuditLogEntry` to the shape the vendor expects. The forwarder sends events to all configured targets for an org after each audit entry is written, using the same fire-and-forget async scheduling pattern from Day 22's alerting and Day 26's incident runner. A failure at the SIEM endpoint never blocks an audit write.

The bulk export function iterates through a date range of audit events and sends them synchronously, one by one, so the caller can get an accurate count of what was forwarded and what failed. This is different from the real-time path, where fire-and-forget is acceptable because individual events stream continuously. A catch-up export is a deliberate operation where knowing the success count matters.

<a id="what-youll-build"></a>
## What you will build today

- `backend/siem/siem_schema.py` - `SIEMTargetType`, `SIEMConfig`, `SIEMEvent`.
- `backend/siem/registry.py` - per-org SIEM config store.
- `backend/siem/formatters.py` - `format_for_splunk`, `format_for_datadog`, `format_for_webhook`.
- `backend/siem/forwarder.py` - `forward_event` (fire-and-forget), `forward_event_sync` (bulk export path), retry logic, auth headers.
- `backend/siem/exporter.py` - `bulk_export` for historical catch-up.
- `backend/routers/siem.py` - config CRUD, test send, bulk export trigger.
- `tests/siem/` - formatter, forwarder, registry, router tests.

<a id="step-321"></a>
## Step 32.1 - SIEM config schema

```python
# backend/siem/siem_schema.py

from __future__ import annotations

from enum import Enum

from pydantic import BaseModel, field_validator


class SIEMTargetType(str, Enum):
    SPLUNK_HEC = "splunk_hec"
    DATADOG    = "datadog"
    WEBHOOK    = "webhook"


class SIEMConfig(BaseModel):
    target_type:     SIEMTargetType
    endpoint_url:    str
    auth_token:      str
    index:           str | None  = None   # Splunk index name; ignored for other targets
    source:          str         = "ai_governance"
    enabled:         bool        = True
    include_payload: bool        = True   # set False to strip raw payload from forwarded events

    @field_validator("endpoint_url")
    @classmethod
    def must_be_https(cls, v: str) -> str:
        if not v.startswith("https://"):
            raise ValueError("SIEM endpoint_url must use HTTPS.")
        return v

    def masked(self) -> dict:
        """Returns a dict safe to send to the client, with auth_token replaced."""
        d = self.model_dump()
        d["auth_token"] = "***"
        return d
```

`include_payload` is a compliance toggle some customers need. A `SECRET_LEAK_BLOCKED` event's payload may contain a redacted version of the secret that leaked; whether that should leave the application and land in a SIEM depends on the customer's data handling policy. The default is `True` (forward everything), but an org that has stricter data residency requirements can set it to `False` so the forwarded event carries the event type, org, actor, and severity but not the raw payload content.

<a id="step-322"></a>
## Step 32.2 - Per-org SIEM config registry

```python
# backend/siem/registry.py

from __future__ import annotations

from collections import defaultdict

from backend.siem.siem_schema import SIEMConfig, SIEMTargetType

# Keyed by org_id -> target_type value -> config.
# In production this should be persisted (e.g., encrypted rows in the
# governance_policies table or a dedicated secrets store). The in-process
# dict here matches the tutorial's infrastructure model and is sufficient
# for demonstrating the integration.
_configs: dict[str, dict[str, SIEMConfig]] = defaultdict(dict)


def set_siem_config(org_id: str, config: SIEMConfig) -> None:
    _configs[org_id][config.target_type.value] = config


def get_siem_configs(org_id: str) -> list[SIEMConfig]:
    return list(_configs.get(org_id, {}).values())


def get_siem_config(org_id: str, target_type: SIEMTargetType) -> SIEMConfig | None:
    return _configs.get(org_id, {}).get(target_type.value)


def remove_siem_config(org_id: str, target_type: SIEMTargetType) -> None:
    _configs.get(org_id, {}).pop(target_type.value, None)


def has_siem_config(org_id: str) -> bool:
    return bool(_configs.get(org_id))
```

The comment about production persistence is worth reading carefully. `auth_token` for a Splunk HEC endpoint is a real secret. Storing it in the in-process dict is fine for the tutorial because the process doesn't persist to disk and the token is not queryable without API access. In production, the token should go through the Day 19 secrets manager (or its equivalent vault integration), with only a reference ID stored in the registry and the actual token fetched at send time.

<a id="step-323"></a>
## Step 32.3 - Event formatters

```python
# backend/siem/formatters.py

from __future__ import annotations

import socket
import time
from typing import Any

from backend.siem.siem_schema import SIEMConfig, SIEMTargetType

_HOSTNAME = socket.gethostname()

# Severity mapping from AuditEventType prefixes to SIEM severity strings.
# This is a best-effort mapping; individual orgs can customize through policy.
_SEVERITY_MAP: dict[str, str] = {
    "audit.tamper_detected":     "critical",
    "secret.leak_blocked":       "high",
    "security.injection_detected": "high",
    "security.abuse_detected":   "high",
    "account.suspended":         "high",
    "budget.exceeded_blocked":   "medium",
    "tenant.access_denied":      "medium",
    "governance.drift_detected": "critical",
    "incident.opened":           "medium",
    "incident.closed":           "info",
    "policy.updated":            "info",
    "session.created":           "info",
}


def _severity(event_type: str) -> str:
    return _SEVERITY_MAP.get(event_type, "info")


def _base_fields(entry, config: SIEMConfig) -> dict[str, Any]:
    payload = entry.payload if config.include_payload else {}
    return {
        "event_id":   entry.event_id,
        "event_type": entry.event_type,
        "org_id":     entry.org_id,
        "actor_id":   entry.actor_id,
        "severity":   _severity(entry.event_type),
        "timestamp":  entry.created_at.isoformat(),
        "source":     config.source,
        "payload":    payload,
    }


def format_for_splunk(entry, config: SIEMConfig) -> dict:
    """
    Splunk HEC batch format. The 'time' field must be a Unix epoch float;
    Splunk rejects ISO 8601 strings in the top-level time field.
    """
    return {
        "time":       entry.created_at.timestamp(),
        "host":       _HOSTNAME,
        "source":     config.source,
        "sourcetype": "_json",
        "index":      config.index or "ai_governance",
        "event":      _base_fields(entry, config),
    }


def format_for_datadog(entry, config: SIEMConfig) -> dict:
    """
    Datadog Logs API format. Tags are comma-separated key:value pairs that
    Datadog uses for faceted search. The 'message' field is what appears in
    the log stream view; everything else is a structured attribute.
    """
    fields = _base_fields(entry, config)
    return {
        "ddsource":  config.source,
        "ddtags":    f"org_id:{entry.org_id},event_type:{entry.event_type},severity:{fields['severity']}",
        "hostname":  _HOSTNAME,
        "service":   config.source,
        "message":   f"{entry.event_type} for org {entry.org_id} actor {entry.actor_id}",
        "@timestamp": int(entry.created_at.timestamp() * 1000),
        **fields,
    }


def format_for_webhook(entry, config: SIEMConfig) -> dict:
    """
    Generic webhook format. Flat JSON, no vendor-specific envelope.
    Compatible with Elastic, Sumo Logic, and most custom log pipelines.
    """
    return {
        "schema_version": "1.0",
        **_base_fields(entry, config),
    }


def format_event(entry, config: SIEMConfig) -> dict:
    if config.target_type == SIEMTargetType.SPLUNK_HEC:
        return format_for_splunk(entry, config)
    elif config.target_type == SIEMTargetType.DATADOG:
        return format_for_datadog(entry, config)
    return format_for_webhook(entry, config)
```

The `_SEVERITY_MAP` is a policy decision embedded in code. This will need to become a policy store entry (Day 25) in a real deployment, because different customers have different severity classifications for the same event types. One enterprise customer may treat `TENANT_ACCESS_DENIED` as a `high` because their threat model considers cross-tenant probing a significant risk; another may leave it at `medium` because their use case sees occasional access denial from normal configuration errors. For now the hardcoded map is a sensible starting point that gets events into the SIEM at approximately the right priority.

<a id="step-324"></a>
## Step 32.4 - The forwarder

```python
# backend/siem/forwarder.py

from __future__ import annotations

import logging
import time

import httpx

from backend.siem.formatters import format_event
from backend.siem.registry import get_siem_configs
from backend.siem.siem_schema import SIEMConfig, SIEMTargetType

logger = logging.getLogger(__name__)

MAX_RETRIES     = 3
RETRY_DELAY_SEC = 0.5
TIMEOUT_SEC     = 5


def _auth_headers(config: SIEMConfig) -> dict:
    if config.target_type == SIEMTargetType.SPLUNK_HEC:
        return {"Authorization": f"Splunk {config.auth_token}"}
    elif config.target_type == SIEMTargetType.DATADOG:
        return {"DD-API-KEY": config.auth_token}
    return {"Authorization": f"Bearer {config.auth_token}"}


def _send_once(config: SIEMConfig, payload: dict) -> None:
    """Sends one event to one target. Raises on failure."""
    with httpx.Client(timeout=TIMEOUT_SEC) as client:
        resp = client.post(
            config.endpoint_url,
            json=payload,
            headers={
                "Content-Type": "application/json",
                **_auth_headers(config),
            },
        )
        resp.raise_for_status()


def _send_with_retry(config: SIEMConfig, payload: dict) -> bool:
    """
    Tries to send up to MAX_RETRIES times. Returns True on success,
    False if all attempts fail. Never raises.
    """
    for attempt in range(MAX_RETRIES):
        try:
            _send_once(config, payload)
            return True
        except Exception as exc:
            if attempt < MAX_RETRIES - 1:
                time.sleep(RETRY_DELAY_SEC * (attempt + 1))
            else:
                logger.warning(
                    "SIEM forward failed after %d attempts to %s: %s",
                    MAX_RETRIES, config.endpoint_url, exc,
                )
    return False


def forward_event(org_id: str, audit_entry) -> None:
    """
    Fire-and-forget path. Called from the audit logger after each emit().
    Errors are logged but never raised; a SIEM being down must not affect
    the audit write.
    """
    for config in get_siem_configs(org_id):
        if not config.enabled:
            continue
        payload = format_event(audit_entry, config)
        _send_with_retry(config, payload)


def forward_event_sync(org_id: str, audit_entry) -> dict[str, bool]:
    """
    Synchronous path for bulk export. Returns per-target success flags
    so the exporter can count successes and failures accurately.
    """
    results: dict[str, bool] = {}
    for config in get_siem_configs(org_id):
        if not config.enabled:
            results[config.target_type.value] = False
            continue
        payload = format_event(audit_entry, config)
        results[config.target_type.value] = _send_with_retry(config, payload)
    return results
```

`forward_event` and `forward_event_sync` differ in return type. The real-time path returns nothing and logs failures; the bulk path returns a per-target success map so the exporter can report to the caller how many events made it to each target. Sharing `_send_with_retry` between both paths means the retry logic lives in exactly one place.

`_send_with_retry` uses a linear backoff (0.5s, 1.0s, 1.5s) rather than exponential. For the real-time path this is fine: the total wait on three failures is 3 seconds, which is short enough that the async task won't hold resources for long. For a bulk export of 10,000 events hitting a rate limit, it would be wrong. If the bulk export path encounters sustained rate-limiting from the SIEM endpoint, callers should reduce the batch size or add per-batch delays at the exporter level rather than relying on the per-event retry loop.

<a id="step-325"></a>
## Step 32.5 - Wiring into the audit logger

```python
# backend/audit/audit_logger.py (changes)

import asyncio

def emit(org_id: str, actor_id: str, event_type: str, payload: dict | None = None) -> AuditLogEntry:
    # ... existing DB write, chain hash, audit log append ...
    entry = _write_to_db(org_id, actor_id, event_type, payload)

    # Fire alerts (Day 22) and incidents (Day 26) - existing wiring.
    _schedule_alert(entry)

    # Forward to any configured SIEM targets (Day 32).
    _schedule_siem_forward(entry, org_id)

    return entry


def _schedule_siem_forward(entry: AuditLogEntry, org_id: str) -> None:
    from backend.siem.registry import has_siem_config
    if not has_siem_config(org_id):
        return  # avoid creating an async task for orgs with no SIEM configured

    async def _run() -> None:
        from backend.siem.forwarder import forward_event
        forward_event(org_id, entry)

    try:
        loop = asyncio.get_running_loop()
        loop.create_task(_run())
    except RuntimeError:
        from backend.siem.forwarder import forward_event
        forward_event(org_id, entry)
```

`has_siem_config` is a fast, non-blocking check against the in-process dict. Checking it before creating the async task means that orgs with no SIEM configured (the majority during development) pay zero overhead per audit event. The task creation and coroutine overhead only exists for orgs that have actually registered a target.

<a id="step-326"></a>
## Step 32.6 - Bulk export for historical data

```python
# backend/siem/exporter.py

from __future__ import annotations

from datetime import datetime, timezone

from backend.siem.forwarder import forward_event_sync


def bulk_export(
    org_id:     str,
    since:      datetime,
    until:      datetime | None = None,
) -> dict:
    """
    Exports all audit events in [since, until] to all SIEM targets configured
    for org_id. Runs synchronously so the caller gets an accurate result count.
    Returns a summary with per-target success/failure counts.
    """
    from backend.audit.audit_logger import get_audit_events_range

    until = until or datetime.now(timezone.utc)
    events = get_audit_events_range(org_id, since=since, until=until)

    per_target: dict[str, dict[str, int]] = {}

    for event in events:
        results = forward_event_sync(org_id, event)
        for target_type, success in results.items():
            bucket = per_target.setdefault(target_type, {"sent": 0, "failed": 0})
            if success:
                bucket["sent"] += 1
            else:
                bucket["failed"] += 1

    return {
        "org_id":      org_id,
        "since":       since.isoformat(),
        "until":       until.isoformat(),
        "total_events": len(events),
        "per_target":  per_target,
    }
```

`get_audit_events_range` was built in Day 24 for the compliance export. Reusing it here means the bulk export respects the same pagination and date range logic, including the chain verification the compliance export uses. If the audit chain has a break in the export window, the caller will see it in the compliance export; the SIEM export does not try to detect breaks but forwards whatever records are in the database.

The per-target failure count matters for the SOC 2 use case. A compliance auditor asking "were all events forwarded to Splunk?" deserves an answer of `{"sent": 8742, "failed": 0}`, not just "we tried." If the answer is `{"sent": 8740, "failed": 2}`, the auditor needs to know which two events failed and why, which means they should re-run the export for a narrower window after fixing the SIEM endpoint.

<a id="step-327"></a>
## Step 32.7 - SIEM admin endpoints

```python
# backend/routers/siem.py

from __future__ import annotations

from datetime import datetime, timezone

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

from backend.siem.exporter import bulk_export
from backend.siem.forwarder import forward_event_sync
from backend.siem.registry import (
    get_siem_config, get_siem_configs, remove_siem_config, set_siem_config,
)
from backend.siem.siem_schema import SIEMConfig, SIEMTargetType
from backend.tenancy.context import TenantContext, get_tenant_context

router = APIRouter(prefix="/siem", tags=["SIEM"])


def _require_admin(ctx: TenantContext) -> None:
    if ctx.user_role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required.")


@router.get("/config")
def list_configs(ctx: TenantContext = Depends(get_tenant_context)):
    _require_admin(ctx)
    return [c.masked() for c in get_siem_configs(ctx.org_id)]


@router.put("/config")
def upsert_config(config: SIEMConfig, ctx: TenantContext = Depends(get_tenant_context)):
    _require_admin(ctx)
    set_siem_config(ctx.org_id, config)
    return config.masked()


@router.delete("/config/{target_type}")
def delete_config(target_type: SIEMTargetType, ctx: TenantContext = Depends(get_tenant_context)):
    _require_admin(ctx)
    if get_siem_config(ctx.org_id, target_type) is None:
        raise HTTPException(status_code=404, detail=f"No {target_type.value} config found for this org.")
    remove_siem_config(ctx.org_id, target_type)
    return {"deleted": target_type.value}


class TestSendRequest(BaseModel):
    target_type: SIEMTargetType | None = None   # None means test all configured targets


@router.post("/test")
def send_test_event(body: TestSendRequest, ctx: TenantContext = Depends(get_tenant_context)):
    """
    Sends a synthetic test event to all configured SIEM targets (or one
    specific target if target_type is provided). The test event has
    event_type='siem.test' and no real payload. It does not write to the
    audit log.
    """
    _require_admin(ctx)

    class _FakeEntry:
        event_id  = "siem-test-event"
        event_type = "siem.test"
        org_id    = ctx.org_id
        actor_id  = ctx.user_id
        payload   = {"test": True}
        created_at = datetime.now(timezone.utc)

    entry = _FakeEntry()
    results = forward_event_sync(ctx.org_id, entry)

    if body.target_type:
        filtered = {k: v for k, v in results.items() if k == body.target_type.value}
        return {"results": filtered}
    return {"results": results}


class ExportRequest(BaseModel):
    since: datetime
    until: datetime | None = None


@router.post("/export")
def trigger_export(body: ExportRequest, ctx: TenantContext = Depends(get_tenant_context)):
    _require_admin(ctx)
    result = bulk_export(ctx.org_id, since=body.since, until=body.until)
    return result
```

`POST /siem/test` does not write an audit entry. A test event that appeared in the real audit log would corrupt the chain hash and add noise to the compliance export. The synthetic entry exists only for the duration of the HTTP call; it formats and sends to the SIEM and then disappears.

`masked()` on `SIEMConfig` is enforced in every response path. The `auth_token` never leaves the server in plaintext. An admin who sets up a Splunk HEC token via `PUT /siem/config` will see `"auth_token": "***"` in every subsequent `GET /siem/config` response.

<a id="step-328"></a>
## Step 32.8 - Tests

```python
# tests/siem/test_formatters.py

from datetime import datetime, timezone
from unittest.mock import MagicMock

from backend.siem.formatters import format_for_datadog, format_for_splunk, format_for_webhook
from backend.siem.siem_schema import SIEMConfig, SIEMTargetType


def _entry(event_type="audit.tamper_detected"):
    e = MagicMock()
    e.event_id  = "evt-1"
    e.event_type = event_type
    e.org_id    = "org-test"
    e.actor_id  = "actor-1"
    e.payload   = {"detail": "hash mismatch"}
    e.created_at = datetime(2024, 1, 15, 12, 0, 0, tzinfo=timezone.utc)
    return e


def _config(target_type, include_payload=True):
    return SIEMConfig(
        target_type=target_type,
        endpoint_url="https://splunk.example.com/services/collector",
        auth_token="tok",
        include_payload=include_payload,
    )


class TestSplunkFormatter:
    def test_time_is_float_not_string(self):
        result = format_for_splunk(_entry(), _config(SIEMTargetType.SPLUNK_HEC))
        assert isinstance(result["time"], float)

    def test_event_contains_event_type(self):
        result = format_for_splunk(_entry(), _config(SIEMTargetType.SPLUNK_HEC))
        assert result["event"]["event_type"] == "audit.tamper_detected"

    def test_severity_critical_for_tamper(self):
        result = format_for_splunk(_entry(), _config(SIEMTargetType.SPLUNK_HEC))
        assert result["event"]["severity"] == "critical"

    def test_payload_excluded_when_include_payload_false(self):
        result = format_for_splunk(_entry(), _config(SIEMTargetType.SPLUNK_HEC, include_payload=False))
        assert result["event"]["payload"] == {}


class TestDatadogFormatter:
    def test_timestamp_is_milliseconds(self):
        result = format_for_datadog(_entry(), _config(SIEMTargetType.DATADOG))
        expected_ms = int(datetime(2024, 1, 15, 12, 0, 0, tzinfo=timezone.utc).timestamp() * 1000)
        assert result["@timestamp"] == expected_ms

    def test_ddtags_contains_org_and_event_type(self):
        result = format_for_datadog(_entry(), _config(SIEMTargetType.DATADOG))
        assert "org_id:org-test" in result["ddtags"]
        assert "event_type:audit.tamper_detected" in result["ddtags"]

    def test_message_field_is_human_readable(self):
        result = format_for_datadog(_entry(), _config(SIEMTargetType.DATADOG))
        assert "org-test" in result["message"]
        assert "actor-1" in result["message"]


class TestWebhookFormatter:
    def test_schema_version_present(self):
        result = format_for_webhook(_entry(), _config(SIEMTargetType.WEBHOOK))
        assert result["schema_version"] == "1.0"

    def test_flat_structure_no_envelope(self):
        result = format_for_webhook(_entry(), _config(SIEMTargetType.WEBHOOK))
        assert "event_type" in result
        assert "event" not in result   # no Splunk-style nested envelope
```

```python
# tests/siem/test_forwarder.py

from unittest.mock import MagicMock, patch

from backend.siem.forwarder import _send_with_retry, forward_event_sync
from backend.siem.siem_schema import SIEMConfig, SIEMTargetType


def _config():
    return SIEMConfig(
        target_type=SIEMTargetType.WEBHOOK,
        endpoint_url="https://siem.example.com/ingest",
        auth_token="tok",
    )


def _entry():
    e = MagicMock()
    e.event_id   = "evt-1"
    e.event_type  = "siem.test"
    e.org_id     = "org-1"
    e.actor_id   = "actor-1"
    e.payload    = {}
    from datetime import datetime, timezone
    e.created_at = datetime.now(timezone.utc)
    return e


@patch("backend.siem.forwarder._send_once")
def test_send_with_retry_returns_true_on_first_success(mock_send):
    mock_send.return_value = None
    result = _send_with_retry(_config(), {"event": "data"})
    assert result is True
    assert mock_send.call_count == 1


@patch("backend.siem.forwarder._send_once", side_effect=Exception("timeout"))
@patch("backend.siem.forwarder.time")
def test_send_with_retry_retries_max_times(mock_time, mock_send):
    result = _send_with_retry(_config(), {"event": "data"})
    assert result is False
    assert mock_send.call_count == 3   # MAX_RETRIES


@patch("backend.siem.forwarder._send_once")
@patch("backend.siem.registry.get_siem_configs")
def test_forward_event_sync_disabled_config_returns_false(mock_get, mock_send):
    cfg = _config()
    cfg.enabled = False
    mock_get.return_value = [cfg]
    results = forward_event_sync("org-1", _entry())
    assert results[SIEMTargetType.WEBHOOK.value] is False
    mock_send.assert_not_called()
```

```python
# tests/siem/test_siem_router.py

import os
os.environ["TESTING"] = "1"

import pytest
from fastapi.testclient import TestClient

from backend.main import app


@pytest.fixture
def client():
    with TestClient(app) as c:
        yield c


def _headers(role="admin"):
    return {"X-Org-ID": "org-siem", "X-User-ID": "u1", "X-User-Role": role}


def test_list_configs_requires_admin(client):
    resp = client.get("/siem/config", headers=_headers("employee"))
    assert resp.status_code == 403


def test_upsert_config_masks_token(client):
    resp = client.put(
        "/siem/config",
        json={
            "target_type": "webhook",
            "endpoint_url": "https://siem.example.com/ingest",
            "auth_token": "real-secret-token",
        },
        headers=_headers("admin"),
    )
    assert resp.status_code == 200
    assert resp.json()["auth_token"] == "***"


def test_delete_config_returns_404_when_not_found(client):
    resp = client.delete("/siem/config/splunk_hec", headers=_headers("admin"))
    assert resp.status_code == 404


def test_export_requires_admin(client):
    resp = client.post(
        "/siem/export",
        json={"since": "2024-01-01T00:00:00Z"},
        headers=_headers("employee"),
    )
    assert resp.status_code == 403
```

Run them:

```bash
TESTING=1 pytest tests/siem/ -v
```

`test_upsert_config_masks_token` is the most important router test in this file. The `PUT /siem/config` endpoint receives a real token in the request body (that's unavoidable; the token has to arrive somehow). The test confirms that the same token does not appear in the response. Without it, a future change to the endpoint that returns the full config object instead of calling `.masked()` would silently start leaking credentials to every admin who calls `GET /siem/config`, and the only way to discover that in production would be a log review.

<a id="what-you-built"></a>
## What you built today

A `SIEMConfig` with HTTPS validation and a `masked()` method that hides `auth_token` from API responses. A per-org config registry that holds one config per target type, up to the three supported targets (Splunk HEC, Datadog, generic webhook). Three event formatters that map any `AuditLogEntry` to the correct JSON shape for each vendor: Unix float timestamp for Splunk, millisecond integer and `ddtags` string for Datadog, flat schema-versioned JSON for the webhook target. A forwarder with linear-backoff retry that returns a success flag per target, with a fire-and-forget variant for the real-time path and a synchronous variant for the bulk export path. Wiring in the audit logger that checks `has_siem_config` before creating an async task, so orgs with no SIEM configured pay zero overhead per audit event. A bulk exporter that sends a full date range of historical events synchronously and returns per-target success and failure counts. Four admin endpoints: config list (tokens masked), config upsert, config delete, test send (synthetic event, no audit log write), and bulk export trigger.

<a id="checklist"></a>
## Day 32 checklist

- [ ] `SIEMConfig.endpoint_url` rejects non-HTTPS URLs at validation time
- [ ] `SIEMConfig.masked()` replaces `auth_token` with `"***"` in all API responses; the raw token never appears in `GET /siem/config` output
- [ ] `format_for_splunk` produces a float `time` field, not an ISO 8601 string
- [ ] `format_for_datadog` produces `@timestamp` in milliseconds and `ddtags` with `org_id` and `event_type`
- [ ] `include_payload=False` produces an empty dict for the `payload` field across all formatters
- [ ] `_send_with_retry` retries exactly `MAX_RETRIES` times on failure and logs a warning on the final attempt without raising
- [ ] `forward_event` is fire-and-forget; `forward_event_sync` returns a per-target success dict
- [ ] `_schedule_siem_forward` in `audit_logger.py` calls `has_siem_config` before creating any async task
- [ ] `bulk_export` returns `per_target` counts with `sent` and `failed` for each target type
- [ ] `POST /siem/test` sends a synthetic event and does not write to the audit log
- [ ] `DELETE /siem/config/{target_type}` returns 404 when the config does not exist
- [ ] `test_upsert_config_masks_token` passes, confirming the API never returns the raw auth token
- [ ] `TESTING=1 pytest tests/siem/ -v` passes

---

[Back to repo](https://github.com/GuntruTirupathamma/AI-Governance) - Previous: [DAY31.md](./DAY31.md) - Next: DAY33.md - Governance Reporting for Compliance and Legal Teams
