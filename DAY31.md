---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 31 of 35
Previous: DAY30.md - Governance Onboarding: Validating a New Org Before Go-Live
Next: DAY32.md - SIEM Integration and Audit Log Export
---

# Day 31: Governance Drift Detection and Config Auditing

## Table of contents

- [The problem with point-in-time verification](#the-problem)
- [What drift detection means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 31.1 - Drift record schema](#step-311)
- [Step 31.2 - Per-org drift detection](#step-312)
- [Step 31.3 - Drift registry](#step-313)
- [Step 31.4 - Scanning all onboarded orgs](#step-314)
- [Step 31.5 - GitHub Actions cron scan](#step-315)
- [Step 31.6 - Drift admin endpoints](#step-316)
- [Step 31.7 - Wiring into the audit event types](#step-317)
- [Step 31.8 - Tests](#step-318)
- [What you built today](#what-you-built)
- [Day 31 checklist](#checklist)

<a id="the-problem"></a>
## The problem with point-in-time verification

Day 30 built an onboarding validator that confirms, at the moment an org goes live, that every required governance control is in place. The `is_onboarded` registry remembers the result. The `require_onboarded` gate blocks requests from orgs that never passed. This is a real improvement over what existed before, but it still answers only one question: was this org correctly configured when it was first provisioned?

It says nothing about now.

A policy default that was present at provisioning time can be gone by the following Tuesday. A developer with database access runs a cleanup query that wipes the wrong rows. A new deployment resets an in-process cache to empty and nobody notices because the TTL hasn't expired yet, so reads still return stale data. An admin uses the `DELETE /policies/{table}/{key}` endpoint from Day 25 on what they thought was a test org but was actually production. In every one of these cases, `is_onboarded` still returns `True`. The registry remembers the past. It has no awareness that the config it verified no longer exists.

The specific failure mode I ran into was a Django-style migration that dropped and re-created the `governance_policies` table as part of a schema rename. The migration worked correctly. The policy store's in-process cache served correct values for another thirty seconds, until TTL expired, and then every call to `get_policy` returned `None`. Every dynamic policy loader fell back to hardcoded defaults. `AUDIT_TAMPER` threshold went from our enterprise customer's custom value back to the system default, which was stricter. Their next legitimate migration tripped the verifier and opened an incident. The customer's status page showed red for eighteen minutes before anyone understood what had happened.

The onboarding registry said they were verified. They were. Six days earlier.

<a id="what-it-means-here"></a>
## What drift detection means here

A drift detector re-runs the same onboarding check functions from Day 30 against every currently-onboarded org, on a schedule. When a check that should be passing now returns a failure, that is drift. The response depends on how critical the check is.

Checks for `audit_chain` and `policy_defaults` get `CRITICAL` severity. If either fails on an onboarded org, the org is immediately removed from the `_onboarded` registry via `revoke_onboarding`. Requests to `/stream` will start returning 403 until the config is restored and the org is re-validated manually. The disruption is intentional: an org whose audit chain anchor is gone cannot be monitored correctly, and an org whose policy defaults are missing is running on a different configuration than the one that was verified.

Checks for `alert_rules`, `budget_limits`, and `abuse_thresholds` get `WARNING` severity. They degrade the governance posture but do not completely break it. The drift is recorded and an audit event fires, but `revoke_onboarding` is not called.

In every case, a `DriftRecord` is written to the per-org drift registry, and `GOVERNANCE_DRIFT_DETECTED` is emitted into the audit log. That event can trigger the Day 22 alerting system the same way any other `CRITICAL` audit event does, so the on-call gets a Slack message without any new wiring.

<a id="what-youll-build"></a>
## What you will build today

- `backend/drift/drift_schema.py` - `DriftSeverity`, `DriftRecord`.
- `backend/drift/drift_detector.py` - `detect_drift_for_org`, `scan_all_orgs`, the per-org drift registry.
- `.github/workflows/drift_scan.yml` - cron workflow that runs the scan every 15 minutes and fails the job if drift is found.
- `backend/routers/drift.py` - `GET /drift`, `POST /drift/scan`, `POST /drift/scan-all`.
- `tests/drift/test_drift_detector.py`, `test_drift_router.py` - tests.

<a id="step-311"></a>
## Step 31.1 - Drift record schema

```python
# backend/drift/drift_schema.py

from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime
from enum import Enum


class DriftSeverity(str, Enum):
    CRITICAL = "critical"   # revoke onboarding automatically
    WARNING  = "warning"    # alert but keep onboarding active
    INFO     = "info"       # log only


@dataclass
class DriftRecord:
    drift_id:     str
    org_id:       str
    check_name:   str
    detected_at:  datetime
    severity:     DriftSeverity
    detail:       str
    auto_revoked: bool = False

    def to_dict(self) -> dict:
        return {
            "drift_id":     self.drift_id,
            "org_id":       self.org_id,
            "check_name":   self.check_name,
            "detected_at":  self.detected_at.isoformat(),
            "severity":     self.severity.value,
            "detail":       self.detail,
            "auto_revoked": self.auto_revoked,
        }
```

`auto_revoked` records whether the drift event caused an immediate onboarding revocation. This matters for the audit trail: an admin reading a drift record two hours later needs to know whether the org's traffic was already blocked at the time the drift was detected, or whether it was still passing requests through the gate while awaiting manual review.

<a id="step-312"></a>
## Step 31.2 - Per-org drift detection

```python
# backend/drift/drift_detector.py

from __future__ import annotations

import uuid
from collections import defaultdict, deque
from datetime import datetime, timezone

from backend.drift.drift_schema import DriftRecord, DriftSeverity
from backend.onboarding.onboarding_schema import CheckStatus

# Registry: newest records first, capped at 100 per org.
_registry: dict[str, deque[DriftRecord]] = defaultdict(lambda: deque(maxlen=100))

# Maps each check name to the severity that applies when it fails on an
# already-onboarded org. The smoke test is not included; it needs a
# TestClient and is too expensive to run on every org every 15 minutes.
_CHECK_SEVERITY: dict[str, DriftSeverity] = {
    "policy_defaults":  DriftSeverity.CRITICAL,
    "audit_chain":      DriftSeverity.CRITICAL,
    "alert_rules":      DriftSeverity.WARNING,
    "budget_limits":    DriftSeverity.WARNING,
    "abuse_thresholds": DriftSeverity.WARNING,
}


def detect_drift_for_org(org_id: str) -> list[DriftRecord]:
    """
    Runs the five scheduled-safe onboarding checks against an already-onboarded
    org and returns a DriftRecord for every check that now fails.
    Checks with CRITICAL severity revoke onboarding immediately.
    All failures emit a GOVERNANCE_DRIFT_DETECTED audit event.
    """
    from backend.onboarding.checks import (
        check_abuse_thresholds,
        check_alert_rules,
        check_audit_chain,
        check_budget_limits,
        check_policy_defaults,
    )

    check_fns = [
        check_policy_defaults,
        check_audit_chain,
        check_alert_rules,
        check_budget_limits,
        check_abuse_thresholds,
    ]

    new_records: list[DriftRecord] = []
    for check_fn in check_fns:
        result = check_fn(org_id)
        if result.status != CheckStatus.FAILED:
            continue

        severity = _CHECK_SEVERITY.get(result.name, DriftSeverity.WARNING)
        auto_revoked = False

        if severity == DriftSeverity.CRITICAL:
            from backend.onboarding.registry import revoke_onboarding
            revoke_onboarding(org_id)
            auto_revoked = True

        _emit_drift_event(org_id, result.name, severity, result.detail)

        record = DriftRecord(
            drift_id=str(uuid.uuid4()),
            org_id=org_id,
            check_name=result.name,
            detected_at=datetime.now(timezone.utc),
            severity=severity,
            detail=result.detail,
            auto_revoked=auto_revoked,
        )
        _registry[org_id].appendleft(record)
        new_records.append(record)

    return new_records


def _emit_drift_event(org_id: str, check_name: str, severity: DriftSeverity, detail: str) -> None:
    try:
        from backend.audit.audit_logger import emit
        from backend.audit.event_types import AuditEventType
        emit(
            org_id=org_id,
            actor_id="drift-scanner",
            event_type=AuditEventType.GOVERNANCE_DRIFT_DETECTED.value,
            payload={
                "check":    check_name,
                "severity": severity.value,
                "detail":   detail,
            },
        )
    except Exception:
        pass  # drift detection must not raise; the record is already written


def list_drift_records(org_id: str, limit: int = 50) -> list[DriftRecord]:
    return list(_registry.get(org_id, []))[:limit]
```

`detect_drift_for_org` runs every check regardless of whether an earlier one returned CRITICAL. An org that has lost both its policy defaults and its audit chain anchor needs both failures visible in the report; if the function short-circuited on the first critical failure, the admin fixing the issue would see one problem, fix it, re-run, and discover a second one. Running all checks to completion is slower but always gives the full picture.

`_emit_drift_event` catches all exceptions silently. The drift record has already been written to the registry before the emit call; if the audit logger fails (for example, because the database is down, which might also be why the `audit_chain` check failed), the failure should not prevent the drift record from being stored or `revoke_onboarding` from being called.

<a id="step-313"></a>
## Step 31.3 - Drift registry

The drift registry is the `_registry` defaultdict defined in `drift_detector.py` above, the same in-process deque pattern used by the incident registry in Day 26 and the escalation queue in Day 28. It does not need its own file; keeping it in `drift_detector.py` next to the functions that write to it avoids the import cycle that would appear if it lived in a separate module that both `drift_detector.py` and the router imported.

One function beyond `list_drift_records` is worth adding explicitly:

```python
# backend/drift/drift_detector.py (continued)

def has_open_critical_drift(org_id: str) -> bool:
    """
    Returns True if the org has any CRITICAL drift record in the last 24 hours.
    Used by the onboarding status endpoint to explain why an org is not currently
    marked as onboarded even if it passed validation before.
    """
    from datetime import timedelta
    cutoff = datetime.now(timezone.utc) - timedelta(hours=24)
    for record in _registry.get(org_id, []):
        if record.severity == DriftSeverity.CRITICAL and record.detected_at >= cutoff:
            return True
    return False
```

This feeds into `GET /onboarding/status` (Day 30). An admin seeing `is_onboarded: false` for an org that definitely passed onboarding last week deserves an explanation. Without `has_open_critical_drift`, the status endpoint can only say "not onboarded"; with it, the response can say "not onboarded because drift was detected 4 hours ago."

<a id="step-314"></a>
## Step 31.4 - Scanning all onboarded orgs

```python
# backend/drift/drift_detector.py (continued)

def scan_all_orgs() -> dict[str, list[DriftRecord]]:
    """
    Runs detect_drift_for_org across every org currently in the onboarded
    registry. Returns only orgs where drift was found; clean orgs are not
    included in the result.
    """
    from backend.onboarding.registry import onboarded_orgs

    results: dict[str, list[DriftRecord]] = {}
    for org_id in onboarded_orgs():
        drift = detect_drift_for_org(org_id)
        if drift:
            results[org_id] = drift
    return results
```

`onboarded_orgs()` was added to the Day 30 registry module (`backend/onboarding/registry.py`) and returns a `frozenset[str]`. Iterating a frozen copy means that even if `detect_drift_for_org` calls `revoke_onboarding` mid-scan (which modifies the live set), the scan does not skip or repeat any org.

<a id="step-315"></a>
## Step 31.5 - GitHub Actions cron scan

```yaml
# .github/workflows/drift_scan.yml

name: Governance Drift Scan

on:
  schedule:
    # Run every 15 minutes. This is aggressive; adjust to every hour for
    # orgs with low traffic volume where a 60-minute exposure window is
    # acceptable. 15 minutes matches the Day 28 escalation sweep interval
    # so both run on the same clock.
    - cron: "*/15 * * * *"
  workflow_dispatch:
    # Also allow manual triggers for post-deployment verification.

jobs:
  drift-scan:
    runs-on: ubuntu-latest
    env:
      TESTING: "1"
      DATABASE_URL: "sqlite:///./drift_scan.db"
      PLATFORM_ADMIN_ORG_ID: ${{ secrets.PLATFORM_ADMIN_ORG_ID }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run alembic migrations
        run: alembic upgrade head

      - name: Run governance drift scan
        run: |
          python - <<'EOF'
          import json, sys
          from backend.drift.drift_detector import scan_all_orgs

          results = scan_all_orgs()
          if not results:
              print("No drift detected across all onboarded orgs.")
              sys.exit(0)

          print("GOVERNANCE DRIFT DETECTED:")
          for org_id, records in results.items():
              for r in records:
                  print(f"  [{r.severity.upper()}] org={org_id} check={r.check_name}: {r.detail}")
                  if r.auto_revoked:
                      print(f"    -> Onboarding revoked automatically for {org_id}")

          critical = [r for records in results.values() for r in records if r.severity == "critical"]
          if critical:
              print(f"\n{len(critical)} critical drift(s) found. Failing the job.")
              sys.exit(1)

          # Warnings do not fail the job but are visible in the CI log.
          sys.exit(0)
          EOF

      - name: Upload drift report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: drift-scan-report
          path: drift_scan.db
```

The script only exits with code 1 for `CRITICAL` drift. Warnings pass the job but print to the CI log where they are visible on the next review. This is a deliberate threshold: revoking an org's traffic for a warning-level config issue (say, one alert rule is misconfigured but the others are fine) is disproportionate, but letting a critical failure (audit chain gone) pass silently is not acceptable.

`workflow_dispatch` is included alongside the cron so that a post-deployment verification can trigger the scan immediately rather than waiting up to 15 minutes for the next scheduled run.

<a id="step-316"></a>
## Step 31.6 - Drift admin endpoints

```python
# backend/routers/drift.py

from __future__ import annotations

import os

from fastapi import APIRouter, Depends, HTTPException

from backend.drift.drift_detector import (
    detect_drift_for_org,
    has_open_critical_drift,
    list_drift_records,
    scan_all_orgs,
)
from backend.tenancy.context import TenantContext, get_tenant_context

router = APIRouter(prefix="/drift", tags=["Drift"])

_PLATFORM_ADMIN_ORG = os.getenv("PLATFORM_ADMIN_ORG_ID", "")


def _require_admin(ctx: TenantContext) -> None:
    if ctx.user_role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required.")


def _require_platform_admin(ctx: TenantContext) -> None:
    if not _PLATFORM_ADMIN_ORG or ctx.org_id != _PLATFORM_ADMIN_ORG or ctx.user_role != "admin":
        raise HTTPException(status_code=403, detail="Platform admin access required.")


@router.get("")
def get_drift_for_own_org(ctx: TenantContext = Depends(get_tenant_context)):
    """
    Returns drift records for the caller's own org, newest first.
    Any org admin can see their own drift history.
    """
    _require_admin(ctx)
    records = list_drift_records(ctx.org_id)
    return {
        "org_id":              ctx.org_id,
        "has_critical_drift":  has_open_critical_drift(ctx.org_id),
        "records":             [r.to_dict() for r in records],
    }


@router.post("/scan")
def scan_own_org(ctx: TenantContext = Depends(get_tenant_context)):
    """
    Triggers an immediate drift scan for the caller's own org.
    Returns the drift records found, if any.
    """
    _require_admin(ctx)
    drift = detect_drift_for_org(ctx.org_id)
    return {
        "org_id":   ctx.org_id,
        "drifted":  len(drift) > 0,
        "records":  [r.to_dict() for r in drift],
    }


@router.post("/scan-all")
def scan_all(ctx: TenantContext = Depends(get_tenant_context)):
    """
    Triggers a drift scan across all onboarded orgs.
    Platform admin only.
    """
    _require_platform_admin(ctx)
    results = scan_all_orgs()
    return {
        "total_orgs_with_drift": len(results),
        "results": {
            org_id: [r.to_dict() for r in records]
            for org_id, records in results.items()
        },
    }


@router.get("/{org_id}")
def get_drift_for_org(org_id: str, ctx: TenantContext = Depends(get_tenant_context)):
    """
    Returns drift records for an arbitrary org. Platform admin only.
    """
    _require_platform_admin(ctx)
    records = list_drift_records(org_id)
    return {
        "org_id":             org_id,
        "has_critical_drift": has_open_critical_drift(org_id),
        "records":            [r.to_dict() for r in records],
    }
```

`GET /drift` and `POST /drift/scan` are available to any org admin for their own org. `GET /drift/{org_id}` and `POST /drift/scan-all` require platform admin, same split as Day 30's onboarding endpoints. This way a customer's security team can monitor and trigger drift scans for their own org without being able to read or affect any other org's data.

<a id="step-317"></a>
## Step 31.7 - Wiring into the audit event types

```python
# backend/audit/event_types.py (add to AuditEventType)

class AuditEventType(str, Enum):
    # ... existing entries from Days 17-30 ...
    GOVERNANCE_DRIFT_DETECTED = "governance.drift_detected"
```

```python
# backend/alerts/alert_rules.py or playbooks config (Day 22 alert rules)
# Add an alert rule for GOVERNANCE_DRIFT_DETECTED so CRITICAL drift
# fires through the Day 22 notifier and lands in Slack automatically.

ALERT_RULES: dict[str, AlertRule] = {
    # ... existing rules ...
    "governance.drift_detected": AlertRule(
        event_type=AuditEventType.GOVERNANCE_DRIFT_DETECTED,
        severity=AlertSeverity.CRITICAL,
        channels=["slack"],
        dedup_window_seconds=300,   # 5-minute dedup; same org drifting twice fast should not spam
        description="A governance configuration has drifted from its verified state.",
    ),
}
```

Because `GOVERNANCE_DRIFT_DETECTED` is now in `ALERT_RULES`, `dispatch_alert()` from Day 22 picks it up automatically. There is no new alerting code to write. The Day 27 red team suite does not need to add a scenario for it either, since the alert rule is now a policy entry that `check_alert_rules` in Day 30 will verify exists during onboarding. If someone removes the drift alert rule, `detect_drift_for_org` catches it as a `WARNING`-level `alert_rules` failure in the next scheduled scan.

<a id="step-318"></a>
## Step 31.8 - Tests

```python
# tests/drift/test_drift_detector.py

import os
os.environ["TESTING"] = "1"

from unittest.mock import patch

from backend.drift.drift_detector import (
    detect_drift_for_org,
    has_open_critical_drift,
    list_drift_records,
)
from backend.drift.drift_schema import DriftSeverity
from backend.onboarding.onboarding_schema import CheckResult, CheckStatus
from backend.onboarding.registry import is_onboarded, mark_onboarded


def _pass(name):
    return CheckResult(name=name, status=CheckStatus.PASSED, detail="ok")


def _fail(name, required=True):
    return CheckResult(name=name, status=CheckStatus.FAILED, detail=f"{name} is broken", required=required)


@patch("backend.drift.drift_detector.check_policy_defaults", side_effect=lambda org_id: _fail("policy_defaults"))
@patch("backend.drift.drift_detector.check_audit_chain",     side_effect=lambda org_id: _pass("audit_chain"))
@patch("backend.drift.drift_detector.check_alert_rules",     side_effect=lambda org_id: _pass("alert_rules"))
@patch("backend.drift.drift_detector.check_budget_limits",   side_effect=lambda org_id: _pass("budget_limits"))
@patch("backend.drift.drift_detector.check_abuse_thresholds",side_effect=lambda org_id: _pass("abuse_thresholds"))
@patch("backend.drift.drift_detector._emit_drift_event")
def test_critical_drift_revokes_onboarding(*mocks):
    mark_onboarded("org-crit")
    assert is_onboarded("org-crit")

    records = detect_drift_for_org("org-crit")

    assert len(records) == 1
    assert records[0].check_name == "policy_defaults"
    assert records[0].severity == DriftSeverity.CRITICAL
    assert records[0].auto_revoked is True
    assert not is_onboarded("org-crit")


@patch("backend.drift.drift_detector.check_policy_defaults", side_effect=lambda org_id: _pass("policy_defaults"))
@patch("backend.drift.drift_detector.check_audit_chain",     side_effect=lambda org_id: _pass("audit_chain"))
@patch("backend.drift.drift_detector.check_alert_rules",     side_effect=lambda org_id: _fail("alert_rules"))
@patch("backend.drift.drift_detector.check_budget_limits",   side_effect=lambda org_id: _pass("budget_limits"))
@patch("backend.drift.drift_detector.check_abuse_thresholds",side_effect=lambda org_id: _pass("abuse_thresholds"))
@patch("backend.drift.drift_detector._emit_drift_event")
def test_warning_drift_does_not_revoke_onboarding(*mocks):
    mark_onboarded("org-warn")
    records = detect_drift_for_org("org-warn")

    assert len(records) == 1
    assert records[0].severity == DriftSeverity.WARNING
    assert records[0].auto_revoked is False
    assert is_onboarded("org-warn")


@patch("backend.drift.drift_detector.check_policy_defaults", side_effect=lambda org_id: _fail("policy_defaults"))
@patch("backend.drift.drift_detector.check_audit_chain",     side_effect=lambda org_id: _fail("audit_chain"))
@patch("backend.drift.drift_detector.check_alert_rules",     side_effect=lambda org_id: _fail("alert_rules"))
@patch("backend.drift.drift_detector.check_budget_limits",   side_effect=lambda org_id: _pass("budget_limits"))
@patch("backend.drift.drift_detector.check_abuse_thresholds",side_effect=lambda org_id: _pass("abuse_thresholds"))
@patch("backend.drift.drift_detector._emit_drift_event")
def test_all_failing_checks_produce_records_not_just_first(*mocks):
    mark_onboarded("org-all-fail")
    records = detect_drift_for_org("org-all-fail")

    # Should see records for policy_defaults, audit_chain, AND alert_rules.
    names = [r.check_name for r in records]
    assert "policy_defaults" in names
    assert "audit_chain" in names
    assert "alert_rules" in names
    assert len(records) == 3


@patch("backend.drift.drift_detector.check_policy_defaults", side_effect=lambda org_id: _pass("policy_defaults"))
@patch("backend.drift.drift_detector.check_audit_chain",     side_effect=lambda org_id: _pass("audit_chain"))
@patch("backend.drift.drift_detector.check_alert_rules",     side_effect=lambda org_id: _pass("alert_rules"))
@patch("backend.drift.drift_detector.check_budget_limits",   side_effect=lambda org_id: _pass("budget_limits"))
@patch("backend.drift.drift_detector.check_abuse_thresholds",side_effect=lambda org_id: _pass("abuse_thresholds"))
def test_no_drift_returns_empty_list(*mocks):
    mark_onboarded("org-clean")
    records = detect_drift_for_org("org-clean")
    assert records == []


@patch("backend.drift.drift_detector.check_policy_defaults", side_effect=lambda org_id: _fail("policy_defaults"))
@patch("backend.drift.drift_detector.check_audit_chain",     side_effect=lambda org_id: _pass("audit_chain"))
@patch("backend.drift.drift_detector.check_alert_rules",     side_effect=lambda org_id: _pass("alert_rules"))
@patch("backend.drift.drift_detector.check_budget_limits",   side_effect=lambda org_id: _pass("budget_limits"))
@patch("backend.drift.drift_detector.check_abuse_thresholds",side_effect=lambda org_id: _pass("abuse_thresholds"))
@patch("backend.drift.drift_detector._emit_drift_event")
def test_has_open_critical_drift_true_after_critical_record(*mocks):
    mark_onboarded("org-crit-flag")
    detect_drift_for_org("org-crit-flag")
    assert has_open_critical_drift("org-crit-flag") is True


def test_has_open_critical_drift_false_for_clean_org():
    assert has_open_critical_drift("org-never-drifted") is False
```

```python
# tests/drift/test_drift_router.py

import os
os.environ["TESTING"] = "1"

import pytest
from fastapi.testclient import TestClient

from backend.main import app


@pytest.fixture
def client():
    with TestClient(app) as c:
        yield c


def _headers(org_id, role):
    return {"X-Org-ID": org_id, "X-User-ID": "u1", "X-User-Role": role}


def test_get_drift_requires_admin(client):
    resp = client.get("/drift", headers=_headers("org-a", "employee"))
    assert resp.status_code == 403


def test_scan_own_org_requires_admin(client):
    resp = client.post("/drift/scan", headers=_headers("org-a", "employee"))
    assert resp.status_code == 403


def test_scan_all_requires_platform_admin(client):
    resp = client.post("/drift/scan-all", headers=_headers("org-a", "admin"))
    assert resp.status_code == 403


def test_get_drift_for_other_org_requires_platform_admin(client):
    resp = client.get("/drift/other-org", headers=_headers("org-a", "admin"))
    assert resp.status_code == 403
```

Run them:

```bash
TESTING=1 pytest tests/drift/ -v
```

`test_all_failing_checks_produce_records_not_just_first` is the test that enforces the "run all checks regardless" behavior. Without it, a future refactor that adds an `if auto_revoked: return` early exit in `detect_drift_for_org` would pass all other tests and silently hide the second and third drift records. The test names that exact failure mode so anyone reading it knows what it is guarding.

<a id="what-you-built"></a>
## What you built today

A `DriftRecord` with a three-tier severity (CRITICAL, WARNING, INFO), a flag recording whether onboarding was automatically revoked, and full audit trail fields. A `detect_drift_for_org` function that re-runs the five scheduled-safe onboarding checks from Day 30 against a single org, maps each failing check to a severity, writes the record, calls `revoke_onboarding` on CRITICAL failures, emits `GOVERNANCE_DRIFT_DETECTED` into the audit log for all failures, and always runs every check rather than stopping at the first failure. A `scan_all_orgs` function that iterates a frozen copy of the `onboarded_orgs` set so mid-scan revocations do not affect the iteration. A GitHub Actions workflow running the scan every 15 minutes via cron and on `workflow_dispatch`, failing the job only for CRITICAL drift while leaving warnings visible in the CI log. Four admin endpoints with a clear permission split: any org admin can read and trigger a scan for their own org, and only the platform admin can scan all orgs or read another org's drift history. A `has_open_critical_drift` function for the Day 30 status endpoint to explain why a previously-onboarded org is now marked unverified. A `GOVERNANCE_DRIFT_DETECTED` audit event type that wires drift alerts automatically through the existing Day 22 notifier, so no new alerting code is needed.

<a id="checklist"></a>
## Day 31 checklist

- [ ] `DriftRecord.auto_revoked` is `True` only when `revoke_onboarding` was actually called, not for all drift records
- [ ] `_CHECK_SEVERITY` assigns `CRITICAL` to `policy_defaults` and `audit_chain`; all others are `WARNING`
- [ ] `detect_drift_for_org` runs every check to completion regardless of whether an earlier one returned CRITICAL
- [ ] `_emit_drift_event` catches all exceptions so a failing audit logger does not prevent drift records from being written or onboarding from being revoked
- [ ] `scan_all_orgs` iterates a `frozenset` snapshot of `onboarded_orgs()` so mid-scan revocations do not affect the loop
- [ ] `GOVERNANCE_DRIFT_DETECTED` is added to `AuditEventType` and has a corresponding entry in `ALERT_RULES` so the Day 22 notifier picks it up automatically
- [ ] `GET /drift` and `POST /drift/scan` return 403 for non-admin roles; `POST /drift/scan-all` and `GET /drift/{org_id}` return 403 unless the caller is the platform admin
- [ ] `has_open_critical_drift` checks the last 24 hours only, not the full drift history
- [ ] `test_all_failing_checks_produce_records_not_just_first` passes, confirming the detector never short-circuits after the first failure
- [ ] `workflow_dispatch` is included alongside `schedule` in the CI workflow so drift scans can be triggered manually after a deployment
- [ ] `TESTING=1 pytest tests/drift/ -v` passes

---

[Back to repo](https://github.com/GuntruTirupathamma/AI-Governance) - Previous: [DAY30.md](./DAY30.md) - Next: DAY32.md - SIEM Integration and Audit Log Export
