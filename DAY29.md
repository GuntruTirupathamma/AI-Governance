---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 29 of 35
Previous: DAY28.md - Human-in-the-Loop Escalation and Review Workflows
Next: DAY30.md - Governance Onboarding: Validating a New Org Before Go-Live
---

# Day 29: Governance SLAs and Postmortem Automation

## Table of contents

- [The problem with postmortems nobody writes](#the-problem)
- [What SLA tracking and postmortems mean here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 29.1 - A gap from Day 28: losing the approval reference](#step-291)
- [Step 29.2 - SLA targets and results](#step-292)
- [Step 29.3 - Computing incident metrics](#step-293)
- [Step 29.4 - The postmortem document](#step-294)
- [Step 29.5 - Recurrence tracking](#step-295)
- [Step 29.6 - The postmortem router](#step-296)
- [Step 29.7 - Aggregate SLA reporting](#step-297)
- [Step 29.8 - Tests](#step-298)
- [What you built today](#what-you-built)
- [Day 29 checklist](#checklist)

<a id="the-problem"></a>
## The problem with postmortems nobody writes

Three weeks after the false-positive `audit.tamper_detected` incident from Day 28, it happened again. Same root cause: another database migration that touched `chain_hash` during a backfill and tripped the verifier. Same org. Same forty-minute window before anyone realized what was going on.

Nobody connected the two incidents until I went digging through Slack history by hand, because the first one never became anything more than a Slack thread that scrolled out of view. There was no document, no root cause field, no flag anywhere saying "this trigger type has happened before and the fix from last time didn't stick." The audit log had every timestamp needed to know exactly how long detection, approval, and resolution took both times. Nobody had ever pulled those numbers out and looked at them.

This is the gap between having data and having information. Day 17 built an audit trail. Day 26 built incident playbooks. Day 28 added an approval gate with its own timestamps. All three systems know precisely when things happened. None of them turn that into an answer to the two questions that actually get asked after an incident: how long did this take, and has it happened before.

<a id="what-it-means-here"></a>
## What SLA tracking and postmortems mean here

Today's work has two parts that feed each other. The first is a small set of service-level targets for incident handling: how fast the first playbook step should fire after an incident opens, how fast a human should respond to an approval request, and how fast the whole incident should close. Each target gets computed from timestamps that already exist on `IncidentRecord` and `PendingApproval`; nothing new needs to be measured, it just needs to be read out.

The second part is a postmortem document generated automatically the moment an incident closes or fails. It carries the trigger, the playbook that ran, every step with its outcome, the SLA results for that specific incident, and a recurrence count: how many times this same trigger type has fired for this org in the last 30 days. An incident with a recurrence count above zero gets flagged, because a second occurrence of the same trigger usually means the first fix didn't actually fix anything.

None of this is meant to replace a written postmortem for serious incidents. It's meant to make sure a baseline document exists for every incident, written or not, so a pattern across incidents is visible without someone manually cross-referencing Slack threads.

<a id="step-291"></a>
## Step 29.1 - A gap from Day 28: losing the approval reference

Before building the SLA for approval response time, there's a bug to fix first. Day 28's `resume_incident` clears `pending_approval_id` back to `None` once a decision is processed:

```python
# backend/incidents/runner.py (Day 28, the problem)

record.status              = IncidentStatus.RUNNING
record.pending_approval_id = None   # the reference is gone after this line
record.pending_step_index  = None
```

That's fine for Day 28's purposes; the runner doesn't need the reference again once it moves past the gate. But it means that by the time an incident reaches `CLOSED`, there is no way to find out which approval gated it, which means no way to measure how long that approval took to decide. The data existed for exactly as long as the incident was paused, then got thrown away.

The fix is to keep a running list instead of overwriting a single field:

```python
# backend/incidents/incident_schema.py (change)

@dataclass
class IncidentRecord:
    incident_id:          str
    org_id:               str
    trigger_event_id:     str
    trigger_event_type:   str
    playbook_name:        str
    status:               IncidentStatus
    steps:                list[StepResult]
    evidence_path:        str | None = None
    opened_at:            datetime | None = None
    closed_at:            datetime | None = None
    pending_approval_id:  str | None = None
    pending_step_index:   int | None = None
    approval_ids:         list[str] = field(default_factory=list)   # new in Day 29
```

```python
# backend/incidents/runner.py (resume_incident, change)

record.approval_ids.append(record.pending_approval_id)
record.status              = IncidentStatus.RUNNING
record.pending_approval_id = None
record.pending_step_index  = None
```

This is the kind of thing that's easy to miss while building the feature that needs it first. Day 28 had no reason to keep the approval history around; nothing consumed it yet. Today does.

<a id="step-292"></a>
## Step 29.2 - SLA targets and results

```python
# backend/postmortem/sla_schema.py

from __future__ import annotations

from dataclasses import dataclass
from datetime import timedelta


@dataclass(frozen=True)
class SLATarget:
    name:        str
    target:      timedelta
    description: str


SLA_TARGETS: dict[str, SLATarget] = {
    "time_to_first_action": SLATarget(
        name="time_to_first_action",
        target=timedelta(seconds=60),
        description="Time from incident open to the first playbook step executing.",
    ),
    "time_to_approval_response": SLATarget(
        name="time_to_approval_response",
        target=timedelta(minutes=30),
        description="Time from an approval being requested to a human deciding it.",
    ),
    "time_to_close": SLATarget(
        name="time_to_close",
        target=timedelta(hours=4),
        description="Time from incident open to incident closed or failed.",
    ),
}


@dataclass
class SLAResult:
    name:   str
    actual: timedelta | None
    target: timedelta
    met:    bool | None   # None means not measurable yet, not "failed"

    def to_dict(self) -> dict:
        return {
            "name":           self.name,
            "actual_seconds": self.actual.total_seconds() if self.actual is not None else None,
            "target_seconds": self.target.total_seconds(),
            "met":            self.met,
        }


@dataclass
class IncidentMetrics:
    incident_id: str
    org_id:      str
    sla_results: list[SLAResult]

    @property
    def all_met(self) -> bool:
        return all(r.met for r in self.sla_results if r.met is not None)

    def to_dict(self) -> dict:
        return {
            "incident_id": self.incident_id,
            "org_id":      self.org_id,
            "all_met":     self.all_met,
            "slas":        [r.to_dict() for r in self.sla_results],
        }
```

`met` is a three-state field, not a boolean. An incident that's still `AWAITING_APPROVAL` has no `time_to_close` yet; that's a different condition from having missed the target, and the two should never get collapsed into the same `False`. `all_met` accounts for this by only checking results where `met` is not `None`, so an in-progress incident doesn't get reported as having failed every SLA it hasn't reached yet.

<a id="step-293"></a>
## Step 29.3 - Computing incident metrics

```python
# backend/postmortem/metrics.py

from __future__ import annotations

from backend.escalation.escalation_queue import get_approval
from backend.incidents.incident_schema import IncidentRecord
from backend.postmortem.sla_schema import SLA_TARGETS, IncidentMetrics, SLAResult


def compute_incident_metrics(record: IncidentRecord) -> IncidentMetrics:
    results = [
        _time_to_first_action(record),
        _time_to_approval_response(record),
        _time_to_close(record),
    ]
    return IncidentMetrics(incident_id=record.incident_id, org_id=record.org_id, sla_results=results)


def _time_to_first_action(record: IncidentRecord) -> SLAResult:
    target = SLA_TARGETS["time_to_first_action"]
    if not record.steps or record.opened_at is None:
        return SLAResult(target.name, None, target.target, None)
    actual = record.steps[0].ran_at - record.opened_at
    return SLAResult(target.name, actual, target.target, actual <= target.target)


def _time_to_approval_response(record: IncidentRecord) -> SLAResult:
    target = SLA_TARGETS["time_to_approval_response"]
    if not record.approval_ids:
        # No gate was hit, or the only gate hasn't been decided yet.
        return SLAResult(target.name, None, target.target, None)
    approval = get_approval(record.org_id, record.approval_ids[-1])
    if approval is None or approval.decided_at is None:
        return SLAResult(target.name, None, target.target, None)
    actual = approval.decided_at - approval.requested_at
    return SLAResult(target.name, actual, target.target, actual <= target.target)


def _time_to_close(record: IncidentRecord) -> SLAResult:
    target = SLA_TARGETS["time_to_close"]
    if record.closed_at is None or record.opened_at is None:
        return SLAResult(target.name, None, target.target, None)
    actual = record.closed_at - record.opened_at
    return SLAResult(target.name, actual, target.target, actual <= target.target)


def summarize_sla_breaches(metrics_list: list[IncidentMetrics]) -> dict:
    """
    Aggregates met/missed/not_measurable counts per SLA name across a set
    of incidents. Used for the org-wide summary endpoint in Step 29.7.
    """
    summary: dict[str, dict[str, int]] = {}
    for metrics in metrics_list:
        for sla in metrics.sla_results:
            bucket = summary.setdefault(sla.name, {"met": 0, "missed": 0, "not_measurable": 0})
            if sla.met is True:
                bucket["met"] += 1
            elif sla.met is False:
                bucket["missed"] += 1
            else:
                bucket["not_measurable"] += 1
    return summary
```

`_time_to_approval_response` reads `record.approval_ids[-1]` rather than the first entry. Today's playbooks only have one gate each, so this distinction does not matter yet, but a future playbook with two gated steps would otherwise report the wrong approval's timing. Taking the most recent entry keeps the function correct now and later without a rewrite.

<a id="step-294"></a>
## Step 29.4 - The postmortem document

```python
# backend/postmortem/postmortem_schema.py

from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime


@dataclass
class PostmortemDocument:
    incident_id:         str
    org_id:              str
    trigger_event_type:  str
    playbook_name:       str
    status:              str
    opened_at:           datetime | None
    closed_at:           datetime | None
    steps_summary:       list[dict]
    sla_summary:         dict
    is_recurring:        bool
    recurrence_count:    int
    generated_at:        datetime

    def to_dict(self) -> dict:
        return {
            "incident_id":        self.incident_id,
            "org_id":             self.org_id,
            "trigger_event_type": self.trigger_event_type,
            "playbook_name":      self.playbook_name,
            "status":             self.status,
            "opened_at":          self.opened_at.isoformat() if self.opened_at else None,
            "closed_at":          self.closed_at.isoformat() if self.closed_at else None,
            "steps_summary":      self.steps_summary,
            "sla_summary":        self.sla_summary,
            "is_recurring":       self.is_recurring,
            "recurrence_count":   self.recurrence_count,
            "generated_at":       self.generated_at.isoformat(),
        }
```

```python
# backend/postmortem/generator.py

from __future__ import annotations

from datetime import datetime, timezone

from backend.incidents.incident_schema import IncidentRecord
from backend.postmortem.metrics import compute_incident_metrics
from backend.postmortem.postmortem_schema import PostmortemDocument
from backend.postmortem.recurrence import count_recent_occurrences


def generate_postmortem(record: IncidentRecord) -> PostmortemDocument:
    metrics = compute_incident_metrics(record)
    recurrence_count = count_recent_occurrences(
        record.org_id, record.trigger_event_type, exclude_incident_id=record.incident_id,
    )

    return PostmortemDocument(
        incident_id=record.incident_id,
        org_id=record.org_id,
        trigger_event_type=record.trigger_event_type,
        playbook_name=record.playbook_name,
        status=record.status.value,
        opened_at=record.opened_at,
        closed_at=record.closed_at,
        steps_summary=[
            {
                "action":  s.action.value,
                "success": s.success,
                "output":  s.output,
                "ran_at":  s.ran_at.isoformat(),
            }
            for s in record.steps
        ],
        sla_summary=metrics.to_dict(),
        is_recurring=recurrence_count > 0,
        recurrence_count=recurrence_count,
        generated_at=datetime.now(timezone.utc),
    )


def to_markdown(doc: PostmortemDocument) -> str:
    lines = [
        f"# Postmortem: {doc.incident_id}",
        "",
        f"- Trigger: `{doc.trigger_event_type}`",
        f"- Playbook: `{doc.playbook_name}`",
        f"- Status: `{doc.status}`",
        f"- Opened: {doc.opened_at.isoformat() if doc.opened_at else 'unknown'}",
        f"- Closed: {doc.closed_at.isoformat() if doc.closed_at else 'still open'}",
        f"- Recurrence: {doc.recurrence_count} prior occurrence(s) in the last 30 days"
        + (" (flagged as recurring)" if doc.is_recurring else ""),
        "",
        "## SLA results",
        "",
    ]
    for sla in doc.sla_summary["slas"]:
        if sla["met"] is True:
            status = "met"
        elif sla["met"] is False:
            status = "missed"
        else:
            status = "not measurable"
        actual = f"{sla['actual_seconds']:.0f}s" if sla["actual_seconds"] is not None else "n/a"
        lines.append(f"- {sla['name']}: {actual} (target {sla['target_seconds']:.0f}s, {status})")

    lines += ["", "## Steps", ""]
    for step in doc.steps_summary:
        mark = "ok" if step["success"] else "FAILED"
        lines.append(f"- [{mark}] `{step['action']}` at {step['ran_at']}: {step['output']}")

    return "\n".join(lines)
```

The markdown output is plain text by design, not a templated HTML report. It needs to paste cleanly into a Slack message, a GitHub issue, or an email without anyone fighting with formatting. The JSON form (`to_dict`) is for the API and for the SLA summary aggregation in Step 29.7; the markdown form is for a human reading it directly.

<a id="step-295"></a>
## Step 29.5 - Recurrence tracking

```python
# backend/incidents/runner.py (add a small public accessor)

def list_incidents(org_id: str) -> list[IncidentRecord]:
    """
    Returns the org's incident history, newest first, same order as the
    underlying deque. Exists so other modules (recurrence tracking, the
    postmortem router) don't reach into the private _registry directly.
    """
    return list(_registry.get(org_id, []))
```

```python
# backend/postmortem/recurrence.py

from __future__ import annotations

from datetime import datetime, timedelta, timezone

from backend.incidents.runner import list_incidents

RECURRENCE_WINDOW_DAYS = 30


def count_recent_occurrences(
    org_id:              str,
    trigger_event_type:  str,
    exclude_incident_id: str | None = None,
    window_days:         int = RECURRENCE_WINDOW_DAYS,
) -> int:
    cutoff = datetime.now(timezone.utc) - timedelta(days=window_days)
    count = 0
    for record in list_incidents(org_id):
        if record.incident_id == exclude_incident_id:
            continue
        if record.trigger_event_type != trigger_event_type:
            continue
        if record.opened_at and record.opened_at >= cutoff:
            count += 1
    return count
```

`list_incidents` is a one-line addition, but it matters: without it, `recurrence.py` would need to import the underscore-prefixed `_registry` straight out of `runner.py`, which is private for a reason. The incident registry's internal structure (a `defaultdict` of capped deques) is an implementation detail that other modules should not depend on directly. A public accessor keeps that detail free to change later without breaking anything that reads incident history.

The 30-day window is a starting point, not a law. An org running this for six months might want a 90-day window to catch slow-burning recurring issues; the parameter is there so that's a config change, not a code change.

<a id="step-296"></a>
## Step 29.6 - The postmortem router

```python
# backend/routers/postmortem.py

from fastapi import APIRouter, Depends, HTTPException
from fastapi.responses import PlainTextResponse

from backend.incidents.runner import list_incidents
from backend.postmortem.generator import generate_postmortem, to_markdown
from backend.tenancy.context import TenantContext, get_tenant_context

router = APIRouter(prefix="/postmortems", tags=["Postmortems"])

_CLOSED_STATUSES = {"closed", "failed"}


def _require_admin(ctx: TenantContext) -> None:
    if ctx.user_role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required")


@router.get("")
def list_postmortems(ctx: TenantContext = Depends(get_tenant_context)):
    _require_admin(ctx)
    closed = [r for r in list_incidents(ctx.org_id) if r.status.value in _CLOSED_STATUSES]
    return [generate_postmortem(r).to_dict() for r in closed]


@router.get("/{incident_id}")
def get_postmortem(
    incident_id: str,
    format:      str = "json",
    ctx:         TenantContext = Depends(get_tenant_context),
):
    _require_admin(ctx)
    record = next((r for r in list_incidents(ctx.org_id) if r.incident_id == incident_id), None)
    if record is None:
        raise HTTPException(status_code=404, detail="Incident not found")

    doc = generate_postmortem(record)
    if format == "markdown":
        return PlainTextResponse(to_markdown(doc))
    return doc.to_dict()
```

A postmortem is generated on demand from the incident's current state rather than stored once and cached. The incident registry already holds everything needed to rebuild it (steps, timestamps, approval references), so there's no separate postmortem store to keep in sync, and no risk of an old cached postmortem disagreeing with the incident record it describes.

<a id="step-297"></a>
## Step 29.7 - Aggregate SLA reporting

```python
# backend/routers/postmortem.py (continued)

from backend.postmortem.metrics import compute_incident_metrics, summarize_sla_breaches


@router.get("/sla/summary")
def sla_summary(ctx: TenantContext = Depends(get_tenant_context)):
    _require_admin(ctx)
    closed = [r for r in list_incidents(ctx.org_id) if r.status.value in _CLOSED_STATUSES]
    metrics = [compute_incident_metrics(r) for r in closed]
    return {
        "org_id":            ctx.org_id,
        "incident_count":    len(closed),
        "sla_breakdown":     summarize_sla_breaches(metrics),
    }
```

This is the endpoint that answers the question leadership actually asks: not "what happened in this one incident" but "how is the governance system performing overall, this month, across everything." A `sla_breakdown` showing `time_to_approval_response` missed in six out of ten incidents points at a real operational problem (on-call response time, alert routing, or a TTL set too aggressively for how the team actually works) in a way that no single incident's postmortem could show on its own.

<a id="step-298"></a>
## Step 29.8 - Tests

```python
# tests/postmortem/test_metrics.py

from datetime import datetime, timedelta, timezone

from backend.escalation.escalation_queue import create_approval, decide_approval
from backend.incidents.incident_schema import (
    IncidentRecord, IncidentStatus, PlaybookAction, PlaybookStep, StepResult,
)
from backend.postmortem.metrics import compute_incident_metrics


def _now():
    return datetime.now(timezone.utc)


def test_time_to_first_action_met_when_fast():
    opened = _now()
    record = IncidentRecord(
        incident_id="i1", org_id="org1", trigger_event_id="e1",
        trigger_event_type="abuse.detected", playbook_name="abuse.detected",
        status=IncidentStatus.CLOSED, steps=[], opened_at=opened,
    )
    record.steps.append(StepResult(
        action=PlaybookAction.PULL_EVIDENCE, success=True, output="ok",
        ran_at=opened + timedelta(seconds=2),
    ))
    record.closed_at = opened + timedelta(seconds=5)
    metrics = compute_incident_metrics(record)
    first_action = next(r for r in metrics.sla_results if r.name == "time_to_first_action")
    assert first_action.met is True


def test_time_to_close_not_measurable_while_open():
    record = IncidentRecord(
        incident_id="i2", org_id="org1", trigger_event_id="e2",
        trigger_event_type="abuse.detected", playbook_name="abuse.detected",
        status=IncidentStatus.RUNNING, steps=[], opened_at=_now(), closed_at=None,
    )
    metrics = compute_incident_metrics(record)
    close_sla = next(r for r in metrics.sla_results if r.name == "time_to_close")
    assert close_sla.met is None


def test_all_met_ignores_unmeasurable_slas():
    record = IncidentRecord(
        incident_id="i3", org_id="org1", trigger_event_id="e3",
        trigger_event_type="abuse.detected", playbook_name="abuse.detected",
        status=IncidentStatus.RUNNING, steps=[], opened_at=_now(), closed_at=None,
    )
    metrics = compute_incident_metrics(record)
    # No SLA has been missed; the unmeasurable ones should not count against it.
    assert metrics.all_met is True


def test_approval_response_uses_most_recent_approval_id():
    opened = _now()
    record = IncidentRecord(
        incident_id="i4", org_id="org2", trigger_event_id="e4",
        trigger_event_type="audit.tamper_detected", playbook_name="audit.tamper_detected",
        status=IncidentStatus.CLOSED, steps=[], opened_at=opened, closed_at=opened + timedelta(minutes=10),
    )
    step = PlaybookStep(PlaybookAction.FREEZE_ORG_BUDGET, requires_approval=True)
    approval = create_approval("org2", "i4", step, "test reason")
    decide_approval("org2", approval.approval_id, approve=True, decided_by="admin1")
    record.approval_ids.append(approval.approval_id)

    metrics = compute_incident_metrics(record)
    approval_sla = next(r for r in metrics.sla_results if r.name == "time_to_approval_response")
    assert approval_sla.met is not None
```

```python
# tests/postmortem/test_recurrence.py

from datetime import datetime, timedelta, timezone

from backend.incidents.incident_schema import IncidentRecord, IncidentStatus
from backend.incidents.runner import _registry
from backend.postmortem.recurrence import count_recent_occurrences


def _make_record(org_id, incident_id, trigger_type, opened_at):
    return IncidentRecord(
        incident_id=incident_id, org_id=org_id, trigger_event_id=f"evt-{incident_id}",
        trigger_event_type=trigger_type, playbook_name=trigger_type,
        status=IncidentStatus.CLOSED, steps=[], opened_at=opened_at,
        closed_at=opened_at + timedelta(minutes=5),
    )


def test_count_recent_occurrences_within_window():
    now = datetime.now(timezone.utc)
    _registry["org-recur"].appendleft(_make_record("org-recur", "old", "audit.tamper_detected", now - timedelta(days=10)))
    _registry["org-recur"].appendleft(_make_record("org-recur", "current", "audit.tamper_detected", now))

    count = count_recent_occurrences("org-recur", "audit.tamper_detected", exclude_incident_id="current")
    assert count == 1


def test_count_recent_occurrences_excludes_outside_window():
    now = datetime.now(timezone.utc)
    _registry["org-old"].appendleft(_make_record("org-old", "ancient", "audit.tamper_detected", now - timedelta(days=45)))

    count = count_recent_occurrences("org-old", "audit.tamper_detected", window_days=30)
    assert count == 0


def test_count_recent_occurrences_ignores_other_trigger_types():
    now = datetime.now(timezone.utc)
    _registry["org-mixed"].appendleft(_make_record("org-mixed", "a", "audit.tamper_detected", now))
    _registry["org-mixed"].appendleft(_make_record("org-mixed", "b", "secret.leak_blocked", now))

    count = count_recent_occurrences("org-mixed", "audit.tamper_detected", exclude_incident_id="a")
    assert count == 0
```

```python
# tests/postmortem/test_postmortem_router.py

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


def test_list_postmortems_requires_admin(client):
    resp = client.get("/postmortems", headers=_headers("org-x", "employee"))
    assert resp.status_code == 403


def test_get_postmortem_unknown_incident_returns_404(client):
    resp = client.get("/postmortems/not-real", headers=_headers("org-x", "admin"))
    assert resp.status_code == 404


def test_sla_summary_requires_admin(client):
    resp = client.get("/postmortems/sla/summary", headers=_headers("org-x", "employee"))
    assert resp.status_code == 403
```

Run them:

```bash
TESTING=1 pytest tests/postmortem/ -v
```

`test_all_met_ignores_unmeasurable_slas` is worth calling out on its own. Without that distinction, every incident still in progress would show up as failing its SLAs simply because they hadn't had time to succeed yet, which would make the SLA summary useless during any period with active incidents, which is exactly when someone is most likely to be looking at it.

<a id="what-you-built"></a>
## What you built today

A fix to Day 28's incident runner that keeps a running `approval_ids` list instead of discarding the reference once a decision is processed, so approval response time stays measurable after an incident closes. Three SLA targets (`time_to_first_action`, `time_to_approval_response`, `time_to_close`) computed directly from timestamps already on `IncidentRecord` and `PendingApproval`, with a three-state result (met, missed, not yet measurable) so in-progress incidents don't get reported as failures. A `PostmortemDocument` generated on demand for any closed or failed incident, with both a JSON form for the API and a plain markdown form meant to paste straight into Slack or a GitHub issue. A recurrence tracker that counts how many times the same trigger type has fired for an org in the last 30 days and flags an incident as recurring if the count is above zero. A public `list_incidents` accessor on the incident runner so other modules stop reaching into its private registry. An admin-only router exposing individual postmortems, a list of all closed postmortems, and an aggregate SLA summary across an org's incident history.

<a id="checklist"></a>
## Day 29 checklist

- [ ] `IncidentRecord.approval_ids` accumulates every approval ID a playbook gate produced, instead of being overwritten and lost
- [ ] `SLAResult.met` is `None` when a metric is not yet measurable, distinct from `False` for an actual miss
- [ ] `IncidentMetrics.all_met` only evaluates SLAs where `met is not None`
- [ ] `compute_incident_metrics` reads `record.approval_ids[-1]` for the approval response SLA, not the first entry
- [ ] `generate_postmortem` builds a `PostmortemDocument` with steps, SLA results, and a recurrence count, without needing a separate postmortem store
- [ ] `to_markdown` produces plain text with no special characters that would break when pasted into Slack or a GitHub issue
- [ ] `count_recent_occurrences` filters by org, by trigger type, and by the lookback window, and excludes the incident being evaluated from its own count
- [ ] `list_incidents` is the only way other modules read the incident registry; nothing outside `runner.py` imports `_registry` directly
- [ ] `GET /postmortems`, `GET /postmortems/{id}`, and `GET /postmortems/sla/summary` all return 403 for non-admin roles
- [ ] `GET /postmortems/{id}` returns 404 for an incident ID that does not exist in the org's history
- [ ] `test_all_met_ignores_unmeasurable_slas` passes, confirming in-progress incidents are not reported as SLA failures
- [ ] `TESTING=1 pytest tests/postmortem/ -v` passes

---

[Back to repo](https://github.com/GuntruTirupathamma/AI-Governance) - Previous: [DAY28.md](./DAY28.md) - Next: DAY30.md - Governance Onboarding: Validating a New Org Before Go-Live
