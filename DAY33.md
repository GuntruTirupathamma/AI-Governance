---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 33 of 35
Previous: DAY32.md - SIEM Integration and Audit Log Export
Next: DAY34.md - Production Hardening and Security Configuration
---

# Day 33: Governance reporting for compliance and legal teams

## Table of contents

- [The problem with raw data and real stakeholders](#the-problem)
- [What governance reporting means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 33.1 - Report period and schema](#step-331)
- [Step 33.2 - Security event collector](#step-332)
- [Step 33.3 - Incident and SLA collector](#step-333)
- [Step 33.4 - Escalation and drift collector](#step-334)
- [Step 33.5 - Policy change collector and builder](#step-335)
- [Step 33.6 - Report builder with caching](#step-336)
- [Step 33.7 - Markdown and JSON renderers](#step-337)
- [Step 33.8 - Report router](#step-338)
- [Step 33.9 - Tests](#step-339)
- [What you built today](#what-you-built)
- [Day 33 checklist](#checklist)

<a id="the-problem"></a>
## The problem with raw data and real stakeholders

The data subject access request came in on a Tuesday. GDPR gives 30 days to respond, and this request asked what AI operations had processed the user's data over the prior six months. The compliance team already had Day 24's raw audit export: 80,000 JSON records covering three major incidents, two postmortems, a governance drift event, and a dozen policy changes. When the legal team saw that file, they said "we can't submit this to a regulator." They needed a narrative document, not a data dump.

Separately, the CISO wanted a monthly board brief. The board's question was simple: "how is AI risk looking this month?" Answering it from the existing endpoints meant hitting six different APIs, extracting numbers from each, and writing a human summary by hand. Someone on the team did this three months in a row before writing a Python script on their laptop. That script was never committed anywhere. When the person left for a week, the brief didn't go out.

Day 24's compliance export addressed the "give us all the records" case. It is what a technical auditor imports into a GRC platform to verify chain integrity. What it does not do is answer "how is governance performing across incidents, escalations, drift, and SLAs" in a way that a CISO or legal team can read without an engineering translator. Day 33 builds that: a cross-system governance summary report that aggregates signals from every module built since Day 17 into a single document in two formats: a structured JSON output for machines and a plain Markdown document for the humans who have to sign off on it.

<a id="what-it-means-here"></a>
## What governance reporting means here

A governance report is not a log file. It covers a time period and answers four questions about that period: what security events happened and were blocked, how incidents were handled (time to close, SLA compliance, whether anything recurred), whether escalations were resolved in time, and whether governance drift was detected. Those four dimensions, combined with a policy change count, give a complete picture of how the system behaved under load.

The report also produces a health score. This is not a black-box number. Each deduction is named, bounded, and derived directly from the underlying counts, so a legal team or compliance officer can reproduce it by hand from the same data. A score that cannot be explained from first principles is not useful in an audit context.

Caching matters because report generation reads from multiple modules. A `GET /reports/governance?days=30` request that hits incidents, audit events, escalations, drift records, and SLA metrics simultaneously is not something you want to recalculate on every page refresh. The builder caches on `(org_id, since date, until date)` and exposes a `bust_cache` flag for when a fresh read is needed.

<a id="what-youll-build"></a>
## What you will build today

- `backend/reports/report_schema.py` - `ReportPeriod`, `IncidentSummary`, `SecurityEventSummary`, `EscalationSummary`, `DriftSummary`, `SLASummary`, `GovernanceReport`.
- `backend/reports/collectors.py` - one function per domain: security events, incidents, escalations, drift, SLA summary, policy changes.
- `backend/reports/builder.py` - `build_governance_report(org_id, period, *, bust_cache)` with in-process cache.
- `backend/reports/renderers.py` - `to_markdown(report)` and `to_dict(report)`.
- `backend/routers/reports.py` - `GET /reports/governance`, `POST /reports/governance/generate`.
- `tests/reports/` - empty period, health score deductions, recurring trigger detection, format parameter.

<a id="step-331"></a>
## Step 33.1 - Report period and schema

```python
# backend/reports/report_schema.py

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone


@dataclass(frozen=True)
class ReportPeriod:
    since: datetime
    until: datetime

    @classmethod
    def last_n_days(cls, n: int) -> "ReportPeriod":
        until = datetime.now(timezone.utc)
        return cls(since=until - timedelta(days=n), until=until)


@dataclass
class IncidentSummary:
    total: int
    closed: int
    failed: int
    by_trigger: dict[str, int]
    avg_mttr_seconds: float | None
    sla_met: int
    sla_missed: int
    recurring_triggers: list[str]


@dataclass
class SecurityEventSummary:
    injection_attempts: int
    secret_leaks_blocked: int
    abuse_detections: int
    tamper_detections: int
    tenant_access_denials: int
    budget_blocks: int
    total_critical: int


@dataclass
class EscalationSummary:
    total: int
    approved: int
    denied: int
    expired: int
    avg_response_seconds: float | None


@dataclass
class DriftSummary:
    total_events: int
    critical_count: int
    warning_count: int
    auto_revocations: int
    affected_org_count: int


@dataclass
class SLASummary:
    # name -> {"met": int, "missed": int, "not_measurable": int}
    per_sla: dict[str, dict[str, int]]


@dataclass
class GovernanceReport:
    org_id: str
    period: ReportPeriod
    generated_at: datetime
    policy_change_count: int
    incidents: IncidentSummary
    security_events: SecurityEventSummary
    escalations: EscalationSummary
    drift: DriftSummary
    sla_summary: SLASummary

    def health_score(self) -> int:
        """
        Returns a 0-100 score.

        Each deduction is named and capped so any compliance officer can
        reproduce this number from the underlying counts. There is no
        hidden weighting.

        Deductions:
          critical drift events  : -10 each, capped at 30
          SLA misses             : -5 each, capped at 20
          recurring triggers     : -5 each, capped at 20
          expired escalations    : -5 each, capped at 15
          warning drift events   : -2 each, capped at 10
        """
        score = 100
        score -= min(self.drift.critical_count * 10, 30)
        score -= min(self.incidents.sla_missed * 5, 20)
        score -= min(len(self.incidents.recurring_triggers) * 5, 20)
        score -= min(self.escalations.expired * 5, 15)
        score -= min(self.drift.warning_count * 2, 10)
        return max(score, 0)

    def to_dict(self) -> dict:
        return {
            "org_id": self.org_id,
            "period": {
                "since": self.period.since.isoformat(),
                "until": self.period.until.isoformat(),
            },
            "generated_at": self.generated_at.isoformat(),
            "health_score": self.health_score(),
            "policy_change_count": self.policy_change_count,
            "incidents": {
                "total": self.incidents.total,
                "closed": self.incidents.closed,
                "failed": self.incidents.failed,
                "by_trigger": self.incidents.by_trigger,
                "avg_mttr_seconds": self.incidents.avg_mttr_seconds,
                "sla_met": self.incidents.sla_met,
                "sla_missed": self.incidents.sla_missed,
                "recurring_triggers": self.incidents.recurring_triggers,
            },
            "security_events": {
                "injection_attempts": self.security_events.injection_attempts,
                "secret_leaks_blocked": self.security_events.secret_leaks_blocked,
                "abuse_detections": self.security_events.abuse_detections,
                "tamper_detections": self.security_events.tamper_detections,
                "tenant_access_denials": self.security_events.tenant_access_denials,
                "budget_blocks": self.security_events.budget_blocks,
                "total_critical": self.security_events.total_critical,
            },
            "escalations": {
                "total": self.escalations.total,
                "approved": self.escalations.approved,
                "denied": self.escalations.denied,
                "expired": self.escalations.expired,
                "avg_response_seconds": self.escalations.avg_response_seconds,
            },
            "drift": {
                "total_events": self.drift.total_events,
                "critical_count": self.drift.critical_count,
                "warning_count": self.drift.warning_count,
                "auto_revocations": self.drift.auto_revocations,
                "affected_org_count": self.drift.affected_org_count,
            },
            "sla_summary": self.sla_summary.per_sla,
        }
```

The `health_score` docstring is the formula. Not a comment that might drift from the code: the docstring is the source of truth, and the code below it is a direct translation. If the formula needs to change for a new SLA tier or a compliance requirement, both are updated together.

<a id="step-332"></a>
## Step 33.2 - Security event collector

```python
# backend/reports/collectors.py  (part 1)

from __future__ import annotations

from backend.reports.report_schema import (
    DriftSummary,
    EscalationSummary,
    IncidentSummary,
    ReportPeriod,
    SLASummary,
    SecurityEventSummary,
)

_CRITICAL_EVENT_TYPES = frozenset(
    {
        "audit.tamper_detected",
        "security.injection_detected",
        "secret.leak_blocked",
        "security.abuse_detected",
        "account.suspended",
    }
)


def collect_security_events(
    org_id: str, period: ReportPeriod
) -> SecurityEventSummary:
    """
    Reads the audit log for the period and counts notable event types.

    Imports deferred to avoid circular dependency between the reports
    package and the audit package.
    """
    from backend.audit.audit_logger import get_audit_events_range

    events = get_audit_events_range(
        org_id, since=period.since, until=period.until
    )
    counts: dict[str, int] = {}
    for e in events:
        counts[e.event_type] = counts.get(e.event_type, 0) + 1

    return SecurityEventSummary(
        injection_attempts=counts.get("security.injection_detected", 0),
        secret_leaks_blocked=counts.get("secret.leak_blocked", 0),
        abuse_detections=counts.get("security.abuse_detected", 0),
        tamper_detections=counts.get("audit.tamper_detected", 0),
        tenant_access_denials=counts.get("tenant.access_denied", 0),
        budget_blocks=counts.get("budget.exceeded_blocked", 0),
        total_critical=sum(
            counts.get(t, 0) for t in _CRITICAL_EVENT_TYPES
        ),
    )
```

The deferred import pattern shows up again here. The reports package should not import from the audit package at module load time because the audit package imports from the policy package, which imports from the tenant package. Any of those import chains can introduce a cycle. Deferring to inside the function body breaks the cycle at the cost of a trivial per-call dict lookup.

<a id="step-333"></a>
## Step 33.3 - Incident and SLA collector

```python
# backend/reports/collectors.py  (part 2, append after part 1)


def collect_incidents(org_id: str, period: ReportPeriod) -> IncidentSummary:
    from backend.incidents.runner import list_incidents
    from backend.postmortem.metrics import compute_incident_metrics

    all_records = list_incidents(org_id)
    in_period = [
        r
        for r in all_records
        if r.opened_at and period.since <= r.opened_at <= period.until
    ]

    closed = [r for r in in_period if r.status.value == "closed"]
    failed = [r for r in in_period if r.status.value == "failed"]

    by_trigger: dict[str, int] = {}
    for r in in_period:
        by_trigger[r.trigger_event_type] = (
            by_trigger.get(r.trigger_event_type, 0) + 1
        )

    mttr_values = [
        (r.closed_at - r.opened_at).total_seconds()
        for r in in_period
        if r.closed_at and r.opened_at
    ]

    sla_met = sla_missed = 0
    for r in in_period:
        metrics = compute_incident_metrics(r)
        close_sla = next(
            (s for s in metrics.sla_results if s.name == "time_to_close"),
            None,
        )
        if close_sla and close_sla.met is True:
            sla_met += 1
        elif close_sla and close_sla.met is False:
            sla_missed += 1

    return IncidentSummary(
        total=len(in_period),
        closed=len(closed),
        failed=len(failed),
        by_trigger=by_trigger,
        avg_mttr_seconds=(
            sum(mttr_values) / len(mttr_values) if mttr_values else None
        ),
        sla_met=sla_met,
        sla_missed=sla_missed,
        recurring_triggers=[
            t for t, count in by_trigger.items() if count > 1
        ],
    )


def collect_sla_summary(org_id: str, period: ReportPeriod) -> SLASummary:
    from backend.incidents.runner import list_incidents
    from backend.postmortem.metrics import compute_incident_metrics
    from backend.postmortem.sla import summarize_sla_breaches

    all_records = list_incidents(org_id)
    in_period = [
        r
        for r in all_records
        if r.opened_at and period.since <= r.opened_at <= period.until
    ]
    metrics_list = [compute_incident_metrics(r) for r in in_period]
    return SLASummary(per_sla=summarize_sla_breaches(metrics_list))
```

`recurring_triggers` is built from the same `by_trigger` dict used for the breakdown. Any trigger type that appears more than once in the period is flagged. This is the same signal Day 29's recurrence checker tracks, but scoped to the report period rather than a rolling 30-day window. A legal team reading a DSAR response needs to know if the same event type triggered multiple incidents during the data subject's activity window.

<a id="step-334"></a>
## Step 33.4 - Escalation and drift collector

```python
# backend/reports/collectors.py  (part 3, append after part 2)


def collect_escalations(
    org_id: str, period: ReportPeriod
) -> EscalationSummary:
    from backend.escalation.queue import list_approvals
    from backend.escalation.approval_schema import ApprovalStatus

    approvals = [
        a
        for a in list_approvals(org_id)
        if period.since <= a.requested_at <= period.until
    ]

    response_times = [
        (a.decided_at - a.requested_at).total_seconds()
        for a in approvals
        if a.decided_at and a.status != ApprovalStatus.EXPIRED
    ]

    return EscalationSummary(
        total=len(approvals),
        approved=sum(
            1 for a in approvals if a.status == ApprovalStatus.APPROVED
        ),
        denied=sum(
            1 for a in approvals if a.status == ApprovalStatus.DENIED
        ),
        expired=sum(
            1 for a in approvals if a.status == ApprovalStatus.EXPIRED
        ),
        avg_response_seconds=(
            sum(response_times) / len(response_times)
            if response_times
            else None
        ),
    )


def collect_drift(org_id: str, period: ReportPeriod) -> DriftSummary:
    from backend.drift.detector import list_drift_records
    from backend.drift.drift_schema import DriftSeverity

    records = [
        r
        for r in list_drift_records(org_id)
        if period.since <= r.detected_at <= period.until
    ]

    affected = {r.org_id for r in records}

    return DriftSummary(
        total_events=len(records),
        critical_count=sum(
            1 for r in records if r.severity == DriftSeverity.CRITICAL
        ),
        warning_count=sum(
            1 for r in records if r.severity == DriftSeverity.WARNING
        ),
        auto_revocations=sum(1 for r in records if r.auto_revoked),
        affected_org_count=len(affected),
    )
```

Expired escalations are counted separately from denied ones because they represent a different failure mode. A denial means someone reviewed the step and said no. An expiry means nobody answered. For the board report those two look the same (approval was not granted), but for the CISO they signal different things: one is a deliberate decision, the other is a process gap.

The drift collector counts `affected_org_count` by collecting the `org_id` from each drift record into a set. This is future-proofing: when the platform admin calls `GET /reports/governance` for their own org_id, the drift collector will have run `detect_drift_for_org` against all tenants and stored records with the per-tenant `org_id`. A single platform-level report can show how many customer orgs had drift events in the period.

<a id="step-335"></a>
## Step 33.5 - Policy change collector

```python
# backend/reports/collectors.py  (part 4, append after part 3)


def collect_policy_changes(org_id: str, period: ReportPeriod) -> int:
    """
    Counts policy.updated events in the audit log for the period.

    Policy deletions and creations emit separate event types and are
    excluded here. The compliance team cares about mutations to existing
    rules - additions and deletions show up in the incident and drift
    signals instead.
    """
    from backend.audit.audit_logger import get_audit_events_range

    events = get_audit_events_range(
        org_id, since=period.since, until=period.until
    )
    return sum(1 for e in events if e.event_type == "policy.updated")
```

<a id="step-336"></a>
## Step 33.6 - Report builder with caching

```python
# backend/reports/builder.py

from __future__ import annotations

from datetime import datetime, timezone

from backend.reports.collectors import (
    collect_drift,
    collect_escalations,
    collect_incidents,
    collect_policy_changes,
    collect_security_events,
    collect_sla_summary,
)
from backend.reports.report_schema import GovernanceReport, ReportPeriod

# In-process cache keyed (org_id, since-date, until-date).
# Reports are expensive to build; they aggregate across six modules.
# Cache invalidates on deploy (in-process dict, not Redis) which is
# acceptable for a compliance report that covers days or weeks.
_cache: dict[str, GovernanceReport] = {}


def build_governance_report(
    org_id: str,
    period: ReportPeriod,
    *,
    bust_cache: bool = False,
) -> GovernanceReport:
    cache_key = (
        f"{org_id}:"
        f"{period.since.date().isoformat()}:"
        f"{period.until.date().isoformat()}"
    )

    if not bust_cache and cache_key in _cache:
        return _cache[cache_key]

    report = GovernanceReport(
        org_id=org_id,
        period=period,
        generated_at=datetime.now(timezone.utc),
        policy_change_count=collect_policy_changes(org_id, period),
        incidents=collect_incidents(org_id, period),
        security_events=collect_security_events(org_id, period),
        escalations=collect_escalations(org_id, period),
        drift=collect_drift(org_id, period),
        sla_summary=collect_sla_summary(org_id, period),
    )
    _cache[cache_key] = report
    return report


def clear_report_cache(org_id: str | None = None) -> None:
    """
    Remove cached reports. Pass org_id to clear only that org's reports,
    or None to clear everything. Intended for tests and forced refreshes.
    """
    if org_id is None:
        _cache.clear()
        return
    to_remove = [k for k in _cache if k.startswith(f"{org_id}:")]
    for k in to_remove:
        del _cache[k]
```

The cache key uses date strings rather than full timestamps. This means two calls for "last 30 days" made two hours apart on the same day return the same cached report. That is intentional for compliance purposes: the legal team wants a stable snapshot they can reference by date, not a number that shifts with every page refresh. When a fresh read is needed after a drift scan or a new incident, the caller passes `bust_cache=True`.

<a id="step-337"></a>
## Step 33.7 - Markdown and JSON renderers

```python
# backend/reports/renderers.py

from __future__ import annotations

from backend.reports.report_schema import GovernanceReport


def _fmt_seconds(s: float | None) -> str:
    if s is None:
        return "n/a"
    if s < 60:
        return f"{s:.0f}s"
    if s < 3600:
        return f"{s / 60:.1f}m"
    return f"{s / 3600:.1f}h"


def _score_label(score: int) -> str:
    if score >= 90:
        return "good"
    if score >= 70:
        return "fair"
    if score >= 50:
        return "poor"
    return "critical"


def to_markdown(report: GovernanceReport) -> str:
    r = report
    score = r.health_score()
    label = _score_label(score)

    lines: list[str] = [
        f"# Governance report: {r.org_id}",
        "",
        f"Period: {r.period.since.strftime('%Y-%m-%d')} to "
        f"{r.period.until.strftime('%Y-%m-%d')}",
        f"Generated: {r.generated_at.strftime('%Y-%m-%d %H:%M UTC')}",
        f"Health score: {score}/100 ({label})",
        "",
        "---",
        "",
        "## Security events",
        "",
        f"Injection attempts blocked: {r.security_events.injection_attempts}",
        f"Secret leaks blocked: {r.security_events.secret_leaks_blocked}",
        f"Abuse detections: {r.security_events.abuse_detections}",
        f"Audit chain tamper detections: {r.security_events.tamper_detections}",
        f"Cross-tenant access denials: {r.security_events.tenant_access_denials}",
        f"Budget overrun blocks: {r.security_events.budget_blocks}",
        f"Total critical events: {r.security_events.total_critical}",
        "",
        "## Incidents",
        "",
        f"Total: {r.incidents.total} "
        f"(closed: {r.incidents.closed}, failed: {r.incidents.failed})",
        f"Average time to resolution: {_fmt_seconds(r.incidents.avg_mttr_seconds)}",
        f"SLA met: {r.incidents.sla_met}",
        f"SLA missed: {r.incidents.sla_missed}",
        "",
    ]

    if r.incidents.by_trigger:
        lines.append("Incidents by trigger event type:")
        lines.append("")
        for trigger, count in sorted(r.incidents.by_trigger.items()):
            lines.append(f"- {trigger}: {count}")
        lines.append("")

    if r.incidents.recurring_triggers:
        lines.append(
            "Recurring triggers (more than one incident in period): "
            + ", ".join(sorted(r.incidents.recurring_triggers))
        )
        lines.append("")

    lines += [
        "## Human-in-the-loop escalations",
        "",
        f"Total approval requests: {r.escalations.total}",
        f"Approved: {r.escalations.approved}",
        f"Denied: {r.escalations.denied}",
        f"Expired without response: {r.escalations.expired}",
        f"Average response time: {_fmt_seconds(r.escalations.avg_response_seconds)}",
        "",
        "## Governance drift",
        "",
        f"Total drift events: {r.drift.total_events}",
        f"Critical (auto-revoked onboarding): {r.drift.critical_count}",
        f"Warning (alerted, no revocation): {r.drift.warning_count}",
        f"Auto-revocations: {r.drift.auto_revocations}",
        f"Orgs affected: {r.drift.affected_org_count}",
        "",
        "## SLA performance",
        "",
    ]

    for sla_name, counts in sorted(r.sla_summary.per_sla.items()):
        lines.append(
            f"{sla_name}: "
            f"met {counts['met']}, "
            f"missed {counts['missed']}, "
            f"not measurable {counts['not_measurable']}"
        )

    lines += [
        "",
        "## Policy activity",
        "",
        f"Policy updates recorded: {r.policy_change_count}",
        "",
        "## Health score breakdown",
        "",
        f"Score: {score}/100 ({label})",
        "",
        "Deductions (see health_score() in report_schema.py for formula):",
        f"  critical drift events    : -{min(r.drift.critical_count * 10, 30)}"
        f" of 30 max",
        f"  SLA misses               : -{min(r.incidents.sla_missed * 5, 20)}"
        f" of 20 max",
        f"  recurring triggers       : "
        f"-{min(len(r.incidents.recurring_triggers) * 5, 20)} of 20 max",
        f"  expired escalations      : -{min(r.escalations.expired * 5, 15)}"
        f" of 15 max",
        f"  warning drift events     : -{min(r.drift.warning_count * 2, 10)}"
        f" of 10 max",
    ]

    return "\n".join(lines)
```

The Markdown output lists the actual deduction values for the current period. A legal team can look at the health score breakdown section, add the numbers, and verify the score without reading any code. That auditability is more important than making the output look polished.

<a id="step-338"></a>
## Step 33.8 - Report router

```python
# backend/routers/reports.py

from __future__ import annotations

from datetime import datetime, timezone

from fastapi import APIRouter, Depends, HTTPException, Query
from fastapi.responses import PlainTextResponse
from pydantic import BaseModel

from backend.auth.tenant import TenantContext, get_tenant_context
from backend.reports.builder import build_governance_report, clear_report_cache
from backend.reports.renderers import to_markdown
from backend.reports.report_schema import ReportPeriod

router = APIRouter(prefix="/reports", tags=["reports"])


def _require_admin(ctx: TenantContext) -> None:
    if ctx.role not in ("admin", "platform_admin"):
        raise HTTPException(status_code=403, detail="Admin role required.")


class GenerateReportRequest(BaseModel):
    since: datetime
    until: datetime
    format: str = "json"


@router.get("/governance")
def get_governance_report(
    days: int = Query(default=30, ge=1, le=365),
    format: str = Query(default="json", pattern="^(json|markdown)$"),
    bust_cache: bool = Query(default=False),
    ctx: TenantContext = Depends(get_tenant_context),
):
    """
    Return a governance summary for the authenticated org over the last N days.
    Requires admin or platform_admin role.

    ?format=markdown returns a plain-text Markdown document suitable for
    attaching to a DSAR response or board brief.
    """
    _require_admin(ctx)
    period = ReportPeriod.last_n_days(days)
    report = build_governance_report(ctx.org_id, period, bust_cache=bust_cache)

    if format == "markdown":
        return PlainTextResponse(to_markdown(report))
    return report.to_dict()


@router.post("/governance/generate")
def generate_governance_report(
    body: GenerateReportRequest,
    ctx: TenantContext = Depends(get_tenant_context),
):
    """
    Generate a report for a specific date range.
    Always busts the cache (this is an explicit regeneration request).

    Accepts since/until as ISO 8601 datetimes so a compliance engineer
    can request an exact window rather than relying on "last N days".
    """
    _require_admin(ctx)

    if body.since >= body.until:
        raise HTTPException(
            status_code=422, detail="since must be before until."
        )

    period = ReportPeriod(since=body.since, until=body.until)
    report = build_governance_report(ctx.org_id, period, bust_cache=True)

    if body.format == "markdown":
        return PlainTextResponse(to_markdown(report))
    return report.to_dict()


@router.delete("/governance/cache")
def clear_governance_cache(
    ctx: TenantContext = Depends(get_tenant_context),
):
    """
    Clear cached reports for this org.
    Useful after a bulk drift scan or incident close that should appear
    in a fresh report immediately.
    """
    _require_admin(ctx)
    clear_report_cache(ctx.org_id)
    return {"cleared": True, "org_id": ctx.org_id}
```

Wire the router into `main.py`:

```python
# backend/main.py  (add alongside existing router imports)

from backend.routers.reports import router as reports_router

app.include_router(reports_router)
```

The `DELETE /reports/governance/cache` endpoint is small but worth including. After the platform drift scan runs at 15-minute intervals, a cached report that was generated two minutes before the scan will not reflect the drift events. Rather than auto-invalidating on every drift write (which would create an indirect dependency from the drift module to the reports module), the explicit cache clear lets an admin or an automated script trigger a refresh at a known point.

<a id="step-339"></a>
## Step 33.9 - Tests

```python
# tests/reports/test_report_schema.py

from datetime import datetime, timedelta, timezone

import pytest

from backend.reports.report_schema import (
    DriftSummary,
    EscalationSummary,
    GovernanceReport,
    IncidentSummary,
    ReportPeriod,
    SLASummary,
    SecurityEventSummary,
)


def _make_report(**overrides) -> GovernanceReport:
    defaults = dict(
        org_id="test-org",
        period=ReportPeriod.last_n_days(30),
        generated_at=datetime.now(timezone.utc),
        policy_change_count=0,
        incidents=IncidentSummary(
            total=0,
            closed=0,
            failed=0,
            by_trigger={},
            avg_mttr_seconds=None,
            sla_met=0,
            sla_missed=0,
            recurring_triggers=[],
        ),
        security_events=SecurityEventSummary(
            injection_attempts=0,
            secret_leaks_blocked=0,
            abuse_detections=0,
            tamper_detections=0,
            tenant_access_denials=0,
            budget_blocks=0,
            total_critical=0,
        ),
        escalations=EscalationSummary(
            total=0,
            approved=0,
            denied=0,
            expired=0,
            avg_response_seconds=None,
        ),
        drift=DriftSummary(
            total_events=0,
            critical_count=0,
            warning_count=0,
            auto_revocations=0,
            affected_org_count=0,
        ),
        sla_summary=SLASummary(per_sla={}),
    )
    defaults.update(overrides)
    return GovernanceReport(**defaults)


def test_empty_period_health_score_is_100():
    report = _make_report()
    assert report.health_score() == 100


def test_critical_drift_deducts_correctly():
    report = _make_report(
        drift=DriftSummary(
            total_events=2,
            critical_count=2,
            warning_count=0,
            auto_revocations=2,
            affected_org_count=2,
        )
    )
    # 2 critical * 10 = 20, well under the 30 cap
    assert report.health_score() == 80


def test_critical_drift_capped_at_30():
    report = _make_report(
        drift=DriftSummary(
            total_events=10,
            critical_count=10,
            warning_count=0,
            auto_revocations=10,
            affected_org_count=10,
        )
    )
    # 10 * 10 = 100 but capped at 30; all other deductions zero
    assert report.health_score() == 70


def test_all_deductions_floor_at_zero():
    report = _make_report(
        incidents=IncidentSummary(
            total=10,
            closed=5,
            failed=5,
            by_trigger={"audit.tamper_detected": 5, "secret.leak_blocked": 5},
            avg_mttr_seconds=3600.0,
            sla_met=0,
            sla_missed=10,
            recurring_triggers=["audit.tamper_detected", "secret.leak_blocked"],
        ),
        escalations=EscalationSummary(
            total=5,
            approved=0,
            denied=0,
            expired=5,
            avg_response_seconds=None,
        ),
        drift=DriftSummary(
            total_events=20,
            critical_count=10,
            warning_count=10,
            auto_revocations=10,
            affected_org_count=5,
        ),
    )
    assert report.health_score() == 0


def test_recurring_triggers_detected_in_summary():
    report = _make_report(
        incidents=IncidentSummary(
            total=3,
            closed=3,
            failed=0,
            by_trigger={"audit.tamper_detected": 2, "secret.leak_blocked": 1},
            avg_mttr_seconds=600.0,
            sla_met=3,
            sla_missed=0,
            recurring_triggers=["audit.tamper_detected"],
        )
    )
    assert "audit.tamper_detected" in report.incidents.recurring_triggers
    assert "secret.leak_blocked" not in report.incidents.recurring_triggers
    # Recurring trigger deducts 5 from health score
    assert report.health_score() == 95


def test_expired_escalations_deduct_health_score():
    report = _make_report(
        escalations=EscalationSummary(
            total=2,
            approved=0,
            denied=0,
            expired=2,
            avg_response_seconds=None,
        )
    )
    # 2 * 5 = 10 deducted
    assert report.health_score() == 90


def test_to_dict_includes_health_score():
    report = _make_report()
    d = report.to_dict()
    assert "health_score" in d
    assert d["health_score"] == 100
    assert d["org_id"] == "test-org"
```

```python
# tests/reports/test_renderers.py

from backend.reports.renderers import _score_label, to_markdown
from tests.reports.test_report_schema import _make_report


def test_markdown_contains_all_section_headers():
    report = _make_report()
    md = to_markdown(report)
    assert "## Security events" in md
    assert "## Incidents" in md
    assert "## Human-in-the-loop escalations" in md
    assert "## Governance drift" in md
    assert "## SLA performance" in md
    assert "## Policy activity" in md
    assert "## Health score breakdown" in md


def test_score_label_boundaries():
    assert _score_label(100) == "good"
    assert _score_label(90) == "good"
    assert _score_label(89) == "fair"
    assert _score_label(70) == "fair"
    assert _score_label(69) == "poor"
    assert _score_label(50) == "poor"
    assert _score_label(49) == "critical"
    assert _score_label(0) == "critical"


def test_markdown_includes_recurring_trigger_line():
    from backend.reports.report_schema import IncidentSummary

    report = _make_report(
        incidents=IncidentSummary(
            total=2,
            closed=2,
            failed=0,
            by_trigger={"audit.tamper_detected": 2},
            avg_mttr_seconds=None,
            sla_met=2,
            sla_missed=0,
            recurring_triggers=["audit.tamper_detected"],
        )
    )
    md = to_markdown(report)
    assert "Recurring triggers" in md
    assert "audit.tamper_detected" in md


def test_markdown_omits_recurring_section_when_none():
    report = _make_report()
    md = to_markdown(report)
    assert "Recurring triggers" not in md
```

```python
# tests/reports/test_router.py

import pytest
from fastapi.testclient import TestClient

from backend.main import app
from backend.reports.builder import clear_report_cache

client = TestClient(app)


@pytest.fixture(autouse=True)
def reset_cache():
    clear_report_cache()
    yield
    clear_report_cache()


def _admin_headers(org_id: str = "test-org") -> dict:
    return {"X-Org-Id": org_id, "X-Role": "admin"}


def test_get_report_returns_health_score():
    r = client.get("/reports/governance?days=7", headers=_admin_headers())
    assert r.status_code == 200
    body = r.json()
    assert "health_score" in body
    assert isinstance(body["health_score"], int)


def test_format_markdown_returns_plain_text():
    r = client.get(
        "/reports/governance?days=7&format=markdown",
        headers=_admin_headers(),
    )
    assert r.status_code == 200
    assert "text/plain" in r.headers["content-type"]
    assert "## Security events" in r.text


def test_non_admin_gets_403():
    r = client.get(
        "/reports/governance",
        headers={"X-Org-Id": "test-org", "X-Role": "operator"},
    )
    assert r.status_code == 403


def test_generate_with_invalid_range_returns_422():
    from datetime import datetime, timezone

    now = datetime.now(timezone.utc)
    r = client.post(
        "/reports/governance/generate",
        json={
            "since": now.isoformat(),
            "until": now.isoformat(),
            "format": "json",
        },
        headers=_admin_headers(),
    )
    assert r.status_code == 422


def test_cache_clear_returns_org_id():
    r = client.delete("/reports/governance/cache", headers=_admin_headers())
    assert r.status_code == 200
    assert r.json()["org_id"] == "test-org"
```

Run the tests:

```bash
TESTING=1 pytest tests/reports/ -v
```

<a id="what-you-built"></a>
## What you built today

The governance system now produces a human-readable summary that pulls from every module built over the last 16 days. A single `GET /reports/governance?days=30&format=markdown` request returns a plain-text document that the legal team can attach to a DSAR response, the CISO can paste into a board brief, and a compliance officer can verify by hand from the health score breakdown section. The health score is transparent: each deduction is named, bounded, and reproducible from the same numbers that appear on the page above it. The JSON format gives the same data to automated systems that process governance signals downstream. Both formats share the same `GovernanceReport` dataclass and are built from the same six collectors, so there is one source of truth for the numbers regardless of which format a consumer requests.

<a id="checklist"></a>
## Day 33 checklist

- [ ] `backend/reports/report_schema.py` created with all dataclasses and `health_score()`
- [ ] `backend/reports/collectors.py` created with all six collector functions
- [ ] `backend/reports/builder.py` created with `build_governance_report` and `clear_report_cache`
- [ ] `backend/reports/renderers.py` created with `to_markdown` and `_fmt_seconds`
- [ ] `backend/routers/reports.py` created with three endpoints
- [ ] Reports router registered in `main.py`
- [ ] `tests/reports/test_report_schema.py` passing (7 tests)
- [ ] `tests/reports/test_renderers.py` passing (4 tests)
- [ ] `tests/reports/test_router.py` passing (5 tests)
- [ ] `GET /reports/governance?format=markdown` returns `text/plain` with all section headers
- [ ] Health score is 100 for an org with no incidents, drift, or escalations
- [ ] Cache clear endpoint returns 200 and the requesting org's id

---

Previous: [DAY32.md - SIEM Integration and Audit Log Export](DAY32.md)
Next: [DAY34.md - Production Hardening and Security Configuration](DAY34.md)
   