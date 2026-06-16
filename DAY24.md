---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 24 of 35
Previous: DAY23.md - Live Governance Dashboard and Real-Time Metrics
Next: DAY25.md - Policy-as-Code: Dynamic Governance Rules
---

# Day 24: Compliance Reporting and Evidence Export

## Table of contents

- [The problem with "show me the last 90 days"](#the-problem)
- [What compliance reporting means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 24.1 - The report schema](#step-241)
- [Step 24.2 - A time-range audit query](#step-242)
- [Step 24.3 - The report builder with chain-integrity attestation](#step-243)
- [Step 24.4 - Export formats: JSON, CSV, and text summary](#step-244)
- [Step 24.5 - The compliance router](#step-245)
- [Step 24.6 - A fast incidents endpoint and a CLI for scheduled generation](#step-246)
- [Step 24.7 - Tests](#step-247)
- [What you built today](#what-you-built)
- [Day 24 checklist](#checklist)

<a id="the-problem"></a>
## The problem with "show me the last 90 days"

Day 21's `get_audit_events(org_id, event_type=None, limit=100)` is good for a dashboard feed. It is not good for a SOC 2 auditor who arrives asking for every governance event Acme generated between January 1 and March 31. The function returns at most 100 records, newest first, with no way to filter by time. Running one call per event type and stitching the results together by hand is not something an auditor should have to do, and in practice, they will not.

The gap is real and common: governance systems are built bottom-up, adding event types day by day, and nobody wires up "generate me a 90-day evidence package" until the auditor shows up. By then the question is how many records were lost because the daily limit was hit.

This system has not lost any records. Every governance decision since Day 17 is in `audit_log`, hash-chained, org-scoped, and indexed. What it needs is a query that accepts a date range, a function that groups and summarizes what came back, and a way to hand the result to an auditor as a file.

<a id="what-it-means-here"></a>
## What compliance reporting means here

Three things, each independent enough to be useful on its own:

1. A `ComplianceReport` - a structured snapshot of one org's audit history over a defined period. It includes a summary (totals by event type, incident counts by severity, unique actor count), a flat list of `ComplianceEvent` records safe to hand to a third party, and an integrity attestation confirming whether the audit chain was intact for the period.
2. Export formats - `to_json()`, `to_csv()`, and `to_text_summary()`. JSON for machine consumption, CSV for spreadsheets and auditor tooling, text for email attachments.
3. A fast `GET /compliance/incidents` endpoint that returns only the high and critical events (no full report needed), for the security reviewer who checks this every morning.

What this is not: a live feed (that's Day 23's dashboard), a replacement for `get_audit_events()` in normal application code (the new range query is for batch export, not request-path use), or a PDF generator (any of today's three formats converts trivially with `pandoc` or a browser print).

<a id="what-youll-build"></a>
## What you will build today

- `backend/compliance/report_schema.py` - `IncidentLevel`, `ComplianceEvent`, `ComplianceSummary`, `ComplianceReport`.
- `get_audit_events_range(org_id, start, end, event_type=None, limit=5000)` added to `audit_logger.py` - the time-range variant of Day 21's helper.
- `backend/compliance/report_builder.py` - `build_report(org_id, start, end)`, which queries the audit log, groups records, counts incidents, and checks chain integrity.
- `backend/compliance/exporters.py` - `to_json()`, `to_csv()`, `to_text_summary()`.
- `backend/routers/compliance.py` - `GET /compliance/report`, `GET /compliance/export`, `GET /compliance/incidents`, all admin-only via Day 21's `TenantContext`.
- `backend/compliance/cli.py` - a small `__main__`-runnable script for scheduled or cron-driven report generation.
- Tests under `tests/compliance/`.

<a id="step-241"></a>
## Step 24.1 - The report schema

`ComplianceEvent` is a flattened, export-safe view of Day 21's `AuditRecord`. It drops `content_hash` and `chain_hash` (irrelevant to an auditor reading a CSV) and keeps everything else. `ComplianceSummary` is the aggregate a reviewer reads first. `ComplianceReport` wraps both.

```python
# backend/compliance/report_schema.py

from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum


class IncidentLevel(str, Enum):
    LOW      = "low"
    MEDIUM   = "medium"
    HIGH     = "high"
    CRITICAL = "critical"


@dataclass
class ComplianceEvent:
    event_id:    str
    event_type:  str
    org_id:      str
    actor_id:    str
    actor_role:  str | None
    session_id:  str | None
    resource_id: str | None
    outcome:     str
    reason:      str | None
    payload:     dict
    timestamp:   datetime
    incident_level: IncidentLevel | None = None  # None = informational only

    def to_dict(self) -> dict:
        return {
            "event_id":      self.event_id,
            "event_type":    self.event_type,
            "org_id":        self.org_id,
            "actor_id":      self.actor_id,
            "actor_role":    self.actor_role,
            "session_id":    self.session_id,
            "resource_id":   self.resource_id,
            "outcome":       self.outcome,
            "reason":        self.reason,
            "payload":       self.payload,
            "timestamp":     self.timestamp.isoformat(),
            "incident_level": self.incident_level.value if self.incident_level else None,
        }


@dataclass
class ComplianceSummary:
    total_events:         int
    by_event_type:        dict[str, int]     # {"session.created": 42, ...}
    incidents_by_level:   dict[str, int]     # {"critical": 3, "high": 1, ...}
    unique_actors:        int
    unique_sessions:      int
    chain_intact:         bool               # False if AUDIT_TAMPER was found in the period
    chain_tamper_count:   int               # how many AUDIT_TAMPER events, if any


@dataclass
class ComplianceReport:
    org_id:       str
    period_start: datetime
    period_end:   datetime
    generated_at: datetime
    summary:      ComplianceSummary
    events:       list[ComplianceEvent] = field(default_factory=list)
```

`chain_intact` is the detail that makes this report more than a CSV dump. Day 17 built a hash-chained log so that if anyone modifies a past record, the chain breaks and `AUDIT_TAMPER` gets written. An auditor who sees `chain_intact: false` in this report knows the period's evidence is suspect - which is exactly what they need to know.

<a id="step-242"></a>
## Step 24.2 - A time-range audit query

Day 21's `get_audit_events(org_id, event_type=None, limit=100)` queries by org and event type, newest first, up to a limit. Compliance export needs a time-bounded version that returns everything in a period, ordered oldest-first (so the CSV reads chronologically).

```python
# backend/audit/audit_logger.py (addition alongside get_audit_events)

from datetime import datetime

def get_audit_events_range(
    org_id:     str,
    start:      datetime,
    end:        datetime,
    event_type: str | None = None,
    limit:      int = 5_000,
) -> list[AuditRecord]:
    """
    Returns audit records for org_id in the half-open interval [start, end),
    ordered oldest-first. limit=5000 protects against runaway queries on
    orgs with very high event volumes; callers requesting export of longer
    periods should paginate or pass a higher limit explicitly.
    """
    db = SessionLocal()
    try:
        query = (
            db.query(AuditLogEntry)
            .filter(AuditLogEntry.org_id == org_id)
            .filter(AuditLogEntry.timestamp >= start)
            .filter(AuditLogEntry.timestamp < end)
        )
        if event_type:
            query = query.filter(AuditLogEntry.event_type == event_type)
        rows = query.order_by(AuditLogEntry.timestamp.asc()).limit(limit).all()
        return [AuditRecord.from_orm(r) for r in rows]
    finally:
        db.close()
```

The `timestamp` column on `AuditLogEntry` is already indexed (Day 17) and the `ix_audit_org_type` composite index (Day 21) covers `(org_id, event_type)` - so the full filter `(org_id, timestamp >= start, timestamp < end)` lands on the `org_id` index and scans only that org's rows within the time window. On a table with many orgs and millions of rows this is the difference between a sub-second response and a full table scan.

<a id="step-243"></a>
## Step 24.3 - The report builder with chain-integrity attestation

`INCIDENT_EVENT_TYPES` maps event types to their compliance severity. It is not the same as Day 22's `ALERT_RULES` - alerting answers "should someone be paged now", compliance severity answers "how serious is this for an auditor's risk assessment". The mapping is stricter: a `BUDGET_WARNING_ISSUED` event is a dashboard-level event, not a compliance incident, so it has no entry here.

```python
# backend/compliance/report_builder.py

from datetime import datetime, timezone
from collections import Counter

from backend.audit.audit_logger import get_audit_events_range
from backend.audit.audit_schema import AuditEventType
from backend.compliance.report_schema import (
    ComplianceEvent,
    ComplianceReport,
    ComplianceSummary,
    IncidentLevel,
)

INCIDENT_EVENT_TYPES: dict[str, IncidentLevel] = {
    AuditEventType.AUDIT_TAMPER.value:          IncidentLevel.CRITICAL,
    AuditEventType.SECRET_LEAK_BLOCKED.value:   IncidentLevel.CRITICAL,
    AuditEventType.ACCOUNT_SUSPENDED.value:     IncidentLevel.CRITICAL,
    AuditEventType.TENANT_ACCESS_DENIED.value:  IncidentLevel.HIGH,
    AuditEventType.ABUSE_DETECTED.value:        IncidentLevel.HIGH,
    AuditEventType.BUDGET_EXCEEDED_BLOCKED.value: IncidentLevel.MEDIUM,
    AuditEventType.SECRET_ACCESS_DENIED.value:  IncidentLevel.MEDIUM,
    AuditEventType.AGENT_SPAWN_BLOCKED.value:   IncidentLevel.LOW,
    AuditEventType.GOAL_BLOCKED.value:          IncidentLevel.LOW,
    AuditEventType.ACCOUNT_THROTTLED.value:     IncidentLevel.LOW,
}


def build_report(org_id: str, start: datetime, end: datetime) -> ComplianceReport:
    raw = get_audit_events_range(org_id, start, end, limit=5_000)

    events: list[ComplianceEvent] = []
    by_event_type: Counter = Counter()
    incidents_by_level: Counter = Counter()
    actors: set[str] = set()
    sessions: set[str] = set()
    tamper_count = 0

    for r in raw:
        level = INCIDENT_EVENT_TYPES.get(r.event_type)
        if level:
            incidents_by_level[level.value] += 1
        if r.event_type == AuditEventType.AUDIT_TAMPER.value:
            tamper_count += 1

        by_event_type[r.event_type] += 1
        actors.add(r.actor_id)
        if r.session_id:
            sessions.add(r.session_id)

        events.append(ComplianceEvent(
            event_id=r.event_id,
            event_type=r.event_type,
            org_id=r.org_id,
            actor_id=r.actor_id,
            actor_role=r.actor_role,
            session_id=r.session_id,
            resource_id=r.resource_id,
            outcome=r.outcome,
            reason=r.reason,
            payload=r.payload,
            timestamp=r.timestamp,
            incident_level=level,
        ))

    summary = ComplianceSummary(
        total_events=len(events),
        by_event_type=dict(by_event_type),
        incidents_by_level=dict(incidents_by_level),
        unique_actors=len(actors),
        unique_sessions=len(sessions),
        chain_intact=(tamper_count == 0),
        chain_tamper_count=tamper_count,
    )

    return ComplianceReport(
        org_id=org_id,
        period_start=start,
        period_end=end,
        generated_at=datetime.now(timezone.utc),
        summary=summary,
        events=events,
    )
```

`chain_intact` is `True` whenever `tamper_count == 0`. This is not a full chain verification (that's Day 17's `AuditVerifier.verify_chain()`). It checks whether any tamper detection fired during the period. A later hardening step would call `AuditVerifier` here and set `chain_intact` based on its result. For now, the flag means "no tamper events observed," which is the right starting point.

<a id="step-244"></a>
## Step 24.4 - Export formats: JSON, CSV, and text summary

Three functions, all pure (no I/O, no DB) - they take a `ComplianceReport` and return a string. That makes them easy to test and easy to stream via FastAPI's `StreamingResponse`.

```python
# backend/compliance/exporters.py

import csv
import io
import json
from datetime import datetime

from backend.compliance.report_schema import ComplianceReport


def to_json(report: ComplianceReport, indent: int = 2) -> str:
    return json.dumps(
        {
            "org_id":       report.org_id,
            "period_start": report.period_start.isoformat(),
            "period_end":   report.period_end.isoformat(),
            "generated_at": report.generated_at.isoformat(),
            "summary": {
                "total_events":       report.summary.total_events,
                "by_event_type":      report.summary.by_event_type,
                "incidents_by_level": report.summary.incidents_by_level,
                "unique_actors":      report.summary.unique_actors,
                "unique_sessions":    report.summary.unique_sessions,
                "chain_intact":       report.summary.chain_intact,
                "chain_tamper_count": report.summary.chain_tamper_count,
            },
            "events": [e.to_dict() for e in report.events],
        },
        indent=indent,
        default=str,
    )


def to_csv(report: ComplianceReport) -> str:
    buf = io.StringIO()
    writer = csv.DictWriter(
        buf,
        fieldnames=[
            "timestamp", "event_id", "event_type", "incident_level",
            "org_id", "actor_id", "actor_role", "session_id",
            "resource_id", "outcome", "reason",
        ],
        extrasaction="ignore",
    )
    writer.writeheader()
    for e in report.events:
        row = e.to_dict()
        row["timestamp"] = e.timestamp.isoformat()
        row["incident_level"] = e.incident_level.value if e.incident_level else ""
        writer.writerow(row)
    return buf.getvalue()


def to_text_summary(report: ComplianceReport) -> str:
    s = report.summary
    lines = [
        f"Compliance Report: {report.org_id}",
        f"Period:            {report.period_start.date()} to {report.period_end.date()}",
        f"Generated:         {report.generated_at.isoformat()}",
        "",
        f"Total events:      {s.total_events}",
        f"Unique actors:     {s.unique_actors}",
        f"Unique sessions:   {s.unique_sessions}",
        f"Chain intact:      {'YES' if s.chain_intact else 'NO - ' + str(s.chain_tamper_count) + ' tamper event(s) detected'}",
        "",
        "Incidents by level:",
    ]
    for level in ("critical", "high", "medium", "low"):
        count = s.incidents_by_level.get(level, 0)
        lines.append(f"  {level.upper():<10} {count}")

    lines += ["", "Event type breakdown:"]
    for event_type, count in sorted(s.by_event_type.items()):
        lines.append(f"  {event_type:<45} {count}")

    return "\n".join(lines)
```

The CSV field order is deliberately chronological-first (`timestamp` as the first column), because auditors reading CSV evidence sort by time as their first instinct. The text summary is formatted for plain-text email: no markdown, no HTML, no dependencies, paste-able into a ticket.

<a id="step-245"></a>
## Step 24.5 - The compliance router

Three endpoints. `GET /compliance/report` returns the full JSON report. `GET /compliance/export` returns a file download in the requested format. Both accept `start` and `end` as ISO 8601 query strings and default to the last 30 days if neither is provided.

```python
# backend/routers/compliance.py

from datetime import datetime, timedelta, timezone

from fastapi import APIRouter, Depends, HTTPException, Query
from fastapi.responses import StreamingResponse

from backend.tenancy.context import TenantContext, get_tenant_context
from backend.compliance.report_builder import build_report
from backend.compliance.exporters import to_csv, to_json, to_text_summary

router = APIRouter()

_THIRTY_DAYS = timedelta(days=30)
_MAX_DAYS = 366


def _require_admin(ctx: TenantContext) -> None:
    if ctx.user_role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required")


def _parse_window(
    start: str | None,
    end:   str | None,
) -> tuple[datetime, datetime]:
    now = datetime.now(timezone.utc)
    try:
        end_dt   = datetime.fromisoformat(end)   if end   else now
        start_dt = datetime.fromisoformat(start) if start else end_dt - _THIRTY_DAYS
    except ValueError as exc:
        raise HTTPException(status_code=422, detail=f"Invalid date format: {exc}") from exc

    if (end_dt - start_dt).days > _MAX_DAYS:
        raise HTTPException(status_code=422, detail=f"Date range cannot exceed {_MAX_DAYS} days")
    if start_dt >= end_dt:
        raise HTTPException(status_code=422, detail="start must be before end")

    return start_dt, end_dt


@router.get("/compliance/report")
def get_compliance_report(
    start: str | None = Query(default=None, description="ISO 8601 start datetime (inclusive)"),
    end:   str | None = Query(default=None, description="ISO 8601 end datetime (exclusive)"),
    ctx:   TenantContext = Depends(get_tenant_context),
):
    _require_admin(ctx)
    start_dt, end_dt = _parse_window(start, end)
    report = build_report(ctx.org_id, start_dt, end_dt)
    return {
        "org_id":       report.org_id,
        "period_start": report.period_start.isoformat(),
        "period_end":   report.period_end.isoformat(),
        "generated_at": report.generated_at.isoformat(),
        "summary":      report.summary.__dict__,
        "event_count":  len(report.events),
    }


@router.get("/compliance/export")
def export_compliance_report(
    format: str         = Query(default="json", description="Export format: json, csv, or text"),
    start:  str | None  = Query(default=None),
    end:    str | None  = Query(default=None),
    ctx:    TenantContext = Depends(get_tenant_context),
):
    _require_admin(ctx)
    if format not in ("json", "csv", "text"):
        raise HTTPException(status_code=422, detail="format must be json, csv, or text")

    start_dt, end_dt = _parse_window(start, end)
    report = build_report(ctx.org_id, start_dt, end_dt)

    date_tag = f"{start_dt.date()}_{end_dt.date()}"
    filename = f"compliance_{ctx.org_id}_{date_tag}.{format if format != 'text' else 'txt'}"

    if format == "json":
        content = to_json(report)
        media_type = "application/json"
    elif format == "csv":
        content = to_csv(report)
        media_type = "text/csv"
    else:
        content = to_text_summary(report)
        media_type = "text/plain"

    return StreamingResponse(
        iter([content]),
        media_type=media_type,
        headers={"Content-Disposition": f'attachment; filename="{filename}"'},
    )
```

```python
# backend/main.py (add this line alongside Day 23's dashboard router)
from backend.routers import compliance

app.include_router(compliance.router, tags=["Compliance"])
```

`_parse_window` is a single, reusable date-parsing function shared by both endpoints. The `_MAX_DAYS = 366` cap is a guard against a request that asks for 10 years of events and blocks the request thread for 30 seconds while SQLAlchemy streams results.

<a id="step-246"></a>
## Step 24.6 - A fast incidents endpoint and a CLI for scheduled generation

`GET /compliance/incidents` is a shortcut for the security reviewer's morning check: only events in `INCIDENT_EVENT_TYPES`, last 7 days by default, no full event list, just the counts and the most recent few per level.

```python
# backend/routers/compliance.py (addition to the same router)

from backend.compliance.report_builder import INCIDENT_EVENT_TYPES, build_report
from backend.compliance.report_schema import IncidentLevel

_SEVEN_DAYS = timedelta(days=7)


@router.get("/compliance/incidents")
def get_recent_incidents(
    days: int         = Query(default=7, ge=1, le=90),
    ctx:  TenantContext = Depends(get_tenant_context),
):
    _require_admin(ctx)
    now      = datetime.now(timezone.utc)
    start_dt = now - timedelta(days=days)
    report   = build_report(ctx.org_id, start_dt, now)

    incidents = [e for e in report.events if e.incident_level is not None]
    by_level: dict[str, list[dict]] = {level.value: [] for level in IncidentLevel}

    for event in incidents:
        level_key = event.incident_level.value
        if len(by_level[level_key]) < 5:          # at most 5 examples per level
            by_level[level_key].append(event.to_dict())

    return {
        "org_id":     ctx.org_id,
        "period_days": days,
        "summary":    report.summary.incidents_by_level,
        "chain_intact": report.summary.chain_intact,
        "by_level":   by_level,
    }
```

For the scheduled side - an auditor who wants a fresh report every Monday morning doesn't want to log into the API. The CLI wraps `build_report` and the exporters in a runnable module:

```python
# backend/compliance/cli.py

"""
Generate a compliance report from the command line:

    python -m backend.compliance.cli --org-id acme --days 90 --format csv

Output goes to stdout so callers can redirect to a file:

    python -m backend.compliance.cli --org-id acme --days 30 --format json \
        > /tmp/acme_compliance_2026-Q1.json
"""

import argparse
import sys
from datetime import datetime, timedelta, timezone

from backend.compliance.exporters import to_csv, to_json, to_text_summary
from backend.compliance.report_builder import build_report

EXPORTERS = {"json": to_json, "csv": to_csv, "text": to_text_summary}


def main() -> None:
    parser = argparse.ArgumentParser(description="Generate a compliance evidence report")
    parser.add_argument("--org-id",  required=True)
    parser.add_argument("--days",    type=int, default=30)
    parser.add_argument("--format",  choices=list(EXPORTERS), default="json")
    parser.add_argument("--start",   help="ISO 8601 start date (overrides --days)")
    parser.add_argument("--end",     help="ISO 8601 end date (default: now)")
    args = parser.parse_args()

    now    = datetime.now(timezone.utc)
    end_dt = datetime.fromisoformat(args.end) if args.end else now
    start_dt = (
        datetime.fromisoformat(args.start) if args.start
        else end_dt - timedelta(days=args.days)
    )

    report = build_report(args.org_id, start_dt, end_dt)
    exporter = EXPORTERS[args.format]
    sys.stdout.write(exporter(report))


if __name__ == "__main__":
    main()
```

Wire this into a weekly cron job:

```
# /etc/cron.d/compliance-reports

# Every Monday at 06:00 UTC, generate last-30-day report for every org and
# email it (pipe to sendmail, s3 cp, or slack-file-upload as appropriate).

0 6 * * 1 app python -m backend.compliance.cli --org-id acme --days 30 --format text \
    | mail -s "Acme governance report" compliance@acme.example
```

<a id="step-247"></a>
## Step 24.7 - Tests

```python
# tests/compliance/test_report_builder.py

from datetime import datetime, timezone, timedelta

import pytest

from backend.compliance.report_builder import build_report, INCIDENT_EVENT_TYPES
from backend.compliance.report_schema import IncidentLevel
from backend.audit.audit_schema import AuditEvent, AuditEventType
from backend.audit.audit_logger import _emit


class TestReportBuilder:
    def test_empty_period_returns_zero_summary(self):
        start = datetime(2020, 1, 1, tzinfo=timezone.utc)
        end   = datetime(2020, 1, 2, tzinfo=timezone.utc)
        report = build_report("acme", start, end)

        assert report.summary.total_events == 0
        assert report.summary.chain_intact is True
        assert report.summary.chain_tamper_count == 0
        assert report.events == []

    def test_account_suspended_appears_as_critical_incident(self):
        _emit(AuditEvent(
            event_type=AuditEventType.ACCOUNT_SUSPENDED,
            org_id="acme", actor_id="alice", actor_role="employee",
            outcome="suspended", reason="abuse", payload={},
        ))

        now   = datetime.now(timezone.utc)
        start = now - timedelta(minutes=5)
        report = build_report("acme", start, now)

        incidents = [e for e in report.events if e.incident_level == IncidentLevel.CRITICAL]
        assert any(e.event_type == AuditEventType.ACCOUNT_SUSPENDED.value for e in incidents)
        assert report.summary.incidents_by_level.get("critical", 0) >= 1

    def test_audit_tamper_sets_chain_intact_false(self):
        _emit(AuditEvent(
            event_type=AuditEventType.AUDIT_TAMPER,
            org_id="acme", actor_id="system", actor_role=None,
            outcome="tamper_detected", reason="hash mismatch", payload={},
        ))

        now   = datetime.now(timezone.utc)
        start = now - timedelta(minutes=5)
        report = build_report("acme", start, now)

        assert report.summary.chain_intact is False
        assert report.summary.chain_tamper_count >= 1

    def test_reports_are_org_isolated(self):
        _emit(AuditEvent(
            event_type=AuditEventType.ACCOUNT_SUSPENDED,
            org_id="acme", actor_id="alice", actor_role="employee",
            outcome="suspended", reason="abuse", payload={},
        ))

        now   = datetime.now(timezone.utc)
        start = now - timedelta(minutes=5)

        acme_report   = build_report("acme",   start, now)
        globex_report = build_report("globex", start, now)

        assert acme_report.summary.total_events >= 1
        assert all(e.org_id == "acme" for e in acme_report.events)
        assert all(e.org_id != "acme" for e in globex_report.events)

    def test_informational_events_have_no_incident_level(self):
        _emit(AuditEvent(
            event_type=AuditEventType.SESSION_CREATED,
            org_id="acme", actor_id="bob", actor_role="employee",
            outcome="created", reason=None, payload={},
        ))

        now   = datetime.now(timezone.utc)
        start = now - timedelta(minutes=5)
        report = build_report("acme", start, now)

        informational = [e for e in report.events if e.event_type == AuditEventType.SESSION_CREATED.value]
        assert all(e.incident_level is None for e in informational)
```

```python
# tests/compliance/test_exporters.py

from datetime import datetime, timezone

from backend.compliance.exporters import to_json, to_csv, to_text_summary
from backend.compliance.report_schema import (
    ComplianceEvent, ComplianceReport, ComplianceSummary, IncidentLevel,
)

NOW = datetime.now(timezone.utc)


def _make_report(**overrides) -> ComplianceReport:
    summary = ComplianceSummary(
        total_events=3,
        by_event_type={"session.created": 2, "account.suspended": 1},
        incidents_by_level={"critical": 1},
        unique_actors=2,
        unique_sessions=2,
        chain_intact=True,
        chain_tamper_count=0,
    )
    events = [
        ComplianceEvent(
            event_id="e1", event_type="account.suspended", org_id="acme",
            actor_id="alice", actor_role="employee", session_id="s1",
            resource_id=None, outcome="suspended", reason="abuse",
            payload={}, timestamp=NOW, incident_level=IncidentLevel.CRITICAL,
        ),
    ]
    return ComplianceReport(
        org_id="acme",
        period_start=NOW,
        period_end=NOW,
        generated_at=NOW,
        summary=summary,
        events=events,
        **overrides,
    )


class TestExporters:
    def test_to_json_is_valid_json_with_expected_keys(self):
        import json
        output = to_json(_make_report())
        data = json.loads(output)

        assert data["org_id"] == "acme"
        assert "summary" in data
        assert "events" in data
        assert data["events"][0]["event_type"] == "account.suspended"

    def test_to_csv_has_header_and_one_data_row(self):
        output = to_csv(_make_report())
        lines = output.strip().split("\n")

        assert lines[0].startswith("timestamp,event_id")
        assert len(lines) == 2  # header + 1 event

    def test_to_text_summary_includes_chain_status(self):
        output = to_text_summary(_make_report())
        assert "Chain intact:      YES" in output

    def test_to_text_summary_flags_tamper(self):
        summary = ComplianceSummary(
            total_events=1, by_event_type={}, incidents_by_level={},
            unique_actors=1, unique_sessions=0,
            chain_intact=False, chain_tamper_count=2,
        )
        report = _make_report(summary=summary)
        output = to_text_summary(report)
        assert "NO" in output
        assert "2" in output


class TestRangeQuery:
    def test_range_query_excludes_events_outside_period(self):
        from datetime import timedelta
        from backend.audit.audit_logger import get_audit_events_range
        from backend.audit.audit_schema import AuditEvent, AuditEventType
        from backend.audit.audit_logger import _emit

        _emit(AuditEvent(
            event_type=AuditEventType.SESSION_CREATED,
            org_id="rangetest", actor_id="x", actor_role=None,
            outcome="created", reason=None, payload={},
        ))

        future_start = datetime.now(timezone.utc) + timedelta(hours=1)
        future_end   = future_start + timedelta(hours=1)

        records = get_audit_events_range("rangetest", future_start, future_end)
        assert records == []
```

Run them:

```bash
pytest tests/compliance/ -v
```

<a id="what-you-built"></a>
## What you built today

A time-range variant of Day 21's `get_audit_events()` that accepts `start` and `end` datetimes, orders results chronologically, and uses the existing `org_id` and `timestamp` indexes without table scans. A `build_report()` function that walks those records once, building event-type counts, incident counts by severity (using `INCIDENT_EVENT_TYPES`, which maps 10 governance event types to four severity levels), unique actor and session sets, and a chain-integrity flag derived from whether any `AUDIT_TAMPER` event appeared in the period. Three exporters over that result: JSON for machine consumption, CSV ordered timestamp-first for auditor tooling, and a plain-text summary formatted for email. Two REST endpoints - `GET /compliance/report` for the summary view and `GET /compliance/export?format=csv|json|text` for file download via `StreamingResponse`. A `GET /compliance/incidents` endpoint that runs the same pipeline but returns only the events classified as incidents, bucketed by severity, limited to five examples per level. And a `backend/compliance/cli.py` module runnable via `python -m backend.compliance.cli` for cron-driven scheduled generation without the HTTP layer.

<a id="checklist"></a>
## Day 24 checklist

- [ ] `backend/compliance/report_schema.py` defines `IncidentLevel`, `ComplianceEvent`, `ComplianceSummary`, and `ComplianceReport`; `ComplianceEvent.to_dict()` produces a flat, JSON-serializable dict with `timestamp` as an ISO string
- [ ] `get_audit_events_range(org_id, start, end, event_type=None, limit=5000)` added to `audit_logger.py`; returns records in `timestamp.asc()` order with both start and end filters applied
- [ ] `INCIDENT_EVENT_TYPES` in `report_builder.py` maps 10 `AuditEventType` values to `IncidentLevel`; `BUDGET_WARNING_ISSUED` and purely informational types (e.g., `SESSION_CREATED`) are not in it
- [ ] `build_report(org_id, start, end)` sets `chain_intact=False` and increments `chain_tamper_count` for every `AUDIT_TAMPER` record found in the period
- [ ] `to_csv()` writes column headers matching `ComplianceEvent` fields and outputs `timestamp` as the first column
- [ ] `to_text_summary()` includes `Chain intact: YES` or the tamper count inline; no markdown, no HTML
- [ ] `GET /compliance/report` and `GET /compliance/export` both enforce `start < end`, cap the range at 366 days, and default the range to the last 30 days if no dates are provided
- [ ] `GET /compliance/export` returns a `StreamingResponse` with a `Content-Disposition: attachment` header and a filename of the form `compliance_{org_id}_{start_date}_{end_date}.{ext}`
- [ ] `GET /compliance/incidents` defaults to `days=7`, returns at most 5 example events per incident level, and includes `chain_intact` in the response
- [ ] `backend/compliance/cli.py` is runnable with `python -m backend.compliance.cli --org-id <id> --days <n> --format <fmt>` and writes to stdout
- [ ] All three endpoints are admin-only via Day 21's `TenantContext`; a non-admin request to any compliance endpoint returns 403
- [ ] All tests in `tests/compliance/` pass

---

[Back to repo](https://github.com/GuntruTirupathamma/AI-Governance) - Previous: [DAY23.md](./DAY23.md) - Next: DAY25.md - Policy-as-Code: Dynamic Governance Rules
