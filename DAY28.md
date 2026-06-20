---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 28 of 35
Previous: DAY27.md - Governance Red Team: Simulating Attacks and Testing Defenses
Next: DAY29.md - Governance SLAs and Postmortem Automation
---

# Day 28: Human-in-the-Loop Escalation and Review Workflows

## Table of contents

- [The problem with fully automated remediation](#the-problem)
- [What human-in-the-loop means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 28.1 - Escalation schema: pending approvals](#step-281)
- [Step 28.2 - Marking playbook steps as approval-required](#step-282)
- [Step 28.3 - The approval queue](#step-283)
- [Step 28.4 - Pausing and resuming the incident runner](#step-284)
- [Step 28.5 - Escalation router](#step-285)
- [Step 28.6 - Notifying a human](#step-286)
- [Step 28.7 - Expiry and the sweep job](#step-287)
- [Step 28.8 - Tests](#step-288)
- [What you built today](#what-you-built)
- [Day 28 checklist](#checklist)

<a id="the-problem"></a>
## The problem with fully automated remediation

Three weeks after Day 26 shipped, the playbook for `audit.tamper_detected` fired on a false positive. A migration had touched the `chain_hash` column on a handful of rows during a backfill, and the verifier read that as tampering. The playbook did exactly what it was built to do: it pulled evidence, then froze the budget for every role in that org, then posted to Slack.

The org happened to be our largest enterprise customer. Their production agents went quiet for forty minutes before anyone on our side noticed the Slack message, and another twenty before we figured out it was a false positive and not an actual breach. Their on-call had already opened a ticket asking why their AI stopped responding to customers.

Nothing about that playbook run was wrong, technically. `FREEZE_ORG_BUDGET` did what it says. The audit log captured every step correctly. The problem is that the system had no way to tell the difference between "this org's spending should stop right now no matter what" and "this org's spending should stop, pending a human glancing at the evidence first." Both cases looked identical to the runner: a `CRITICAL` alert, a matching playbook, a sequence of steps.

Some governance actions are cheap to get wrong. Pulling evidence and posting a Slack message cost nothing if the trigger turns out to be noise. Other actions are expensive to get wrong: suspending an account locks a real person out, and freezing a budget can take down a paying customer's production traffic. The fix is not to remove automation from the expensive actions; it's to add a checkpoint that a human has to clear before they execute.

<a id="what-it-means-here"></a>
## What human-in-the-loop means here

Today's design adds an approval gate to specific playbook steps. A step marked `requires_approval=True` does not run immediately. Instead, the runner creates a `PendingApproval` record, posts a notification with enough context for a human to make a call, and pauses the incident at that point in the playbook. Everything before the gate (pulling evidence, in every playbook built so far) still runs automatically, because evidence collection has no downside. Everything after the gate waits.

A human then approves or denies the pending action through an admin endpoint. Approval resumes the playbook from where it stopped: the gated step executes, and any steps after it run as normal. Denial skips the gated step, records why, and continues with whatever comes next. If nobody responds within a time window, the approval expires and is treated the same as a denial, so an incident can never hang open indefinitely waiting on someone who is asleep or on vacation.

Two existing actions get the gate in this design: `FREEZE_ORG_BUDGET` and `SUSPEND_ACTOR`. Both have real, immediate impact on a real account. `PULL_EVIDENCE`, `NOTIFY_CHANNEL`, and `LOG_ONLY` stay automatic, since none of them change a customer's experience.

<a id="step-281"></a>
## Step 28.1 - Escalation schema: pending approvals

```python
# backend/escalation/escalation_schema.py

from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum

from pydantic import BaseModel

from backend.incidents.incident_schema import PlaybookStep


class ApprovalStatus(str, Enum):
    PENDING  = "pending"
    APPROVED = "approved"
    DENIED   = "denied"
    EXPIRED  = "expired"


@dataclass
class PendingApproval:
    approval_id:   str
    org_id:        str
    incident_id:   str
    step:          PlaybookStep
    reason:        str
    requested_at:  datetime
    expires_at:    datetime
    status:        ApprovalStatus = ApprovalStatus.PENDING
    decided_by:    str | None = None
    decided_at:    datetime | None = None
    decision_note: str | None = None

    def is_overdue(self, now: datetime | None = None) -> bool:
        now = now or datetime.now(timezone.utc)
        return self.status == ApprovalStatus.PENDING and now >= self.expires_at

    def to_dict(self) -> dict:
        return {
            "approval_id":   self.approval_id,
            "org_id":        self.org_id,
            "incident_id":   self.incident_id,
            "action":        self.step.action.value,
            "params":        self.step.params,
            "reason":        self.reason,
            "requested_at":  self.requested_at.isoformat(),
            "expires_at":    self.expires_at.isoformat(),
            "status":        self.status.value,
            "decided_by":    self.decided_by,
            "decided_at":    self.decided_at.isoformat() if self.decided_at else None,
            "decision_note": self.decision_note,
        }


class ApprovalDecision(BaseModel):
    approve: bool
    note: str | None = None
```

`is_overdue` checks `status == PENDING` itself, rather than relying on the caller to filter first. An approval that's already `APPROVED` and past its `expires_at` should never flip to `EXPIRED`; the deadline only matters while a decision is still outstanding.

<a id="step-282"></a>
## Step 28.2 - Marking playbook steps as approval-required

```python
# backend/incidents/incident_schema.py (changes from Day 26)

@dataclass(frozen=True)
class PlaybookStep:
    action:            PlaybookAction
    params:             dict[str, Any] = field(default_factory=dict)
    requires_approval:  bool = False          # new in Day 28


class IncidentStatus(str, Enum):
    RUNNING            = "running"
    AWAITING_APPROVAL  = "awaiting_approval"  # new in Day 28
    CLOSED             = "closed"
    FAILED             = "failed"


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
    pending_approval_id:  str | None = None   # new in Day 28
    pending_step_index:   int | None = None   # new in Day 28
```

```python
# backend/incidents/playbooks.py (diff from Day 26)

PLAYBOOKS: dict[str, Playbook] = {
    "audit.tamper_detected": Playbook(
        name="audit.tamper_detected",
        description="Evidence first, then cut spending, then alert.",
        steps=(
            PlaybookStep(PlaybookAction.PULL_EVIDENCE, {"lookback_hours": 24}),
            PlaybookStep(PlaybookAction.FREEZE_ORG_BUDGET, requires_approval=True),
            PlaybookStep(PlaybookAction.NOTIFY_CHANNEL, {"channel": "slack"}),
        ),
    ),
    "secret.leak_blocked": Playbook(
        name="secret.leak_blocked",
        description="Suspend the actor, then gather evidence, then alert.",
        steps=(
            PlaybookStep(PlaybookAction.SUSPEND_ACTOR, requires_approval=True),
            PlaybookStep(PlaybookAction.PULL_EVIDENCE, {"lookback_hours": 4}),
            PlaybookStep(PlaybookAction.NOTIFY_CHANNEL, {"channel": "slack"}),
        ),
    ),
    # account.suspended, abuse.detected, and tenant.access_denied are unchanged:
    # none of their steps touch a customer's spending or access directly.
}
```

The gate is a property of the step, not the action. `SUSPEND_ACTOR` requires approval inside `secret.leak_blocked` because the actor might be a legitimate admin who tripped the secret scanner on a false positive. If a future playbook ever needs to suspend an actor automatically with no real downside (a bot account created purely for abuse, say), that step can be added with `requires_approval=False` without touching this one.

<a id="step-283"></a>
## Step 28.3 - The approval queue

```python
# backend/escalation/escalation_queue.py

from __future__ import annotations

import uuid
from collections import defaultdict, deque
from datetime import datetime, timedelta, timezone

from backend.escalation.escalation_schema import ApprovalStatus, PendingApproval
from backend.incidents.incident_schema import PlaybookStep

DEFAULT_TTL_HOURS = 4

# Capped per-org history, same pattern as the Day 26 incident registry.
_registry: dict[str, deque[PendingApproval]] = defaultdict(lambda: deque(maxlen=200))


def create_approval(
    org_id:      str,
    incident_id: str,
    step:        PlaybookStep,
    reason:      str,
    ttl_hours:   int = DEFAULT_TTL_HOURS,
) -> PendingApproval:
    now = datetime.now(timezone.utc)
    approval = PendingApproval(
        approval_id=str(uuid.uuid4()),
        org_id=org_id,
        incident_id=incident_id,
        step=step,
        reason=reason,
        requested_at=now,
        expires_at=now + timedelta(hours=ttl_hours),
    )
    _registry[org_id].appendleft(approval)
    return approval


def get_approval(org_id: str, approval_id: str) -> PendingApproval | None:
    for approval in _registry.get(org_id, []):
        if approval.approval_id == approval_id:
            return approval
    return None


def list_pending(org_id: str) -> list[PendingApproval]:
    """
    Returns approvals still waiting on a decision. Has the side effect of
    flipping any overdue PENDING approval to EXPIRED before filtering, so
    callers never see a stale entry that's actually past its deadline.
    """
    now = datetime.now(timezone.utc)
    for approval in _registry.get(org_id, []):
        if approval.is_overdue(now):
            approval.status = ApprovalStatus.EXPIRED
    return [a for a in _registry.get(org_id, []) if a.status == ApprovalStatus.PENDING]


def decide_approval(
    org_id:      str,
    approval_id: str,
    approve:     bool,
    decided_by:  str,
    note:        str | None = None,
) -> PendingApproval | None:
    approval = get_approval(org_id, approval_id)
    if approval is None or approval.status != ApprovalStatus.PENDING:
        return None
    approval.status        = ApprovalStatus.APPROVED if approve else ApprovalStatus.DENIED
    approval.decided_by    = decided_by
    approval.decided_at    = datetime.now(timezone.utc)
    approval.decision_note = note
    return approval
```

`decide_approval` returns `None` if the approval was already decided or has expired, instead of silently overwriting a prior decision. An admin double-clicking "approve" twice, or two admins racing to respond to the same Slack alert, should not be able to flip a decision after the fact.

<a id="step-284"></a>
## Step 28.4 - Pausing and resuming the incident runner

```python
# backend/incidents/runner.py (changes from Day 26)

from __future__ import annotations

from datetime import datetime, timezone

from backend.escalation.escalation_queue import create_approval, get_approval
from backend.escalation.escalation_schema import ApprovalStatus
from backend.incidents.incident_schema import (
    IncidentRecord, IncidentStatus, PlaybookStep, StepResult,
)
from backend.incidents.playbooks import PLAYBOOKS


def run_incident(trigger_event) -> IncidentRecord | None:
    event_type = trigger_event.event_type
    playbook = PLAYBOOKS.get(event_type)
    if playbook is None:
        return None

    record = IncidentRecord(
        incident_id=str(uuid.uuid4()),
        org_id=trigger_event.org_id,
        trigger_event_id=trigger_event.event_id,
        trigger_event_type=event_type,
        playbook_name=playbook.name,
        status=IncidentStatus.RUNNING,
        steps=[],
        opened_at=datetime.now(timezone.utc),
    )
    _registry[record.org_id].appendleft(record)
    _emit_incident_event(record, "incident.opened")

    _run_steps(record, playbook.steps, start_index=0)
    return record


def _run_steps(record: IncidentRecord, steps: tuple[PlaybookStep, ...], start_index: int) -> None:
    all_ok = True
    for idx in range(start_index, len(steps)):
        step = steps[idx]

        if step.requires_approval:
            reason = f"{step.action.value} has direct customer impact and needs a human sign-off."
            approval = create_approval(record.org_id, record.incident_id, step, reason)
            record.status              = IncidentStatus.AWAITING_APPROVAL
            record.pending_approval_id = approval.approval_id
            record.pending_step_index  = idx
            _emit_incident_event(record, "incident.awaiting_approval", approval_id=approval.approval_id)
            _notify_approval_requested(record, approval)
            return  # stop here; resume_incident continues once a decision lands

        result = _execute_step(record.org_id, step)
        record.steps.append(result)
        _emit_incident_event(record, "incident.step_executed", action=step.action.value)
        if not result.success:
            all_ok = False

    record.status    = IncidentStatus.CLOSED if all_ok else IncidentStatus.FAILED
    record.closed_at = datetime.now(timezone.utc)
    _emit_incident_event(record, "incident.closed")


def resume_incident(org_id: str, incident_id: str) -> IncidentRecord | None:
    """
    Called after an approval has been decided (approved, denied, or expired).
    Executes or skips the gated step, then continues running the rest of
    the playbook from the next index.
    """
    record = _find_incident(org_id, incident_id)
    if record is None or record.status != IncidentStatus.AWAITING_APPROVAL:
        return None

    approval = get_approval(org_id, record.pending_approval_id)
    if approval is None or approval.status == ApprovalStatus.PENDING:
        return None  # still waiting; nothing to resume yet

    playbook = PLAYBOOKS[record.trigger_event_type]
    step_index = record.pending_step_index
    step = playbook.steps[step_index]

    if approval.status == ApprovalStatus.APPROVED:
        result = _execute_step(record.org_id, step)
    else:
        result = StepResult(
            action=step.action,
            success=False,
            output=f"Skipped: approval {approval.status.value} ({approval.decision_note or 'no note'})",
            ran_at=datetime.now(timezone.utc),
        )

    record.steps.append(result)
    record.status              = IncidentStatus.RUNNING
    record.pending_approval_id = None
    record.pending_step_index  = None
    _emit_incident_event(record, "incident.approval_decided", approval_status=approval.status.value)

    _run_steps(record, playbook.steps, start_index=step_index + 1)
    return record


def _find_incident(org_id: str, incident_id: str) -> IncidentRecord | None:
    for record in _registry.get(org_id, []):
        if record.incident_id == incident_id:
            return record
    return None
```

The runner never blocks a thread waiting on a human. `_run_steps` returns as soon as it hits a gated step, and the `IncidentRecord` sits in the registry with `status=AWAITING_APPROVAL` until something calls `resume_incident`. This keeps the fire-and-forget async pattern from Day 26 intact: `dispatch_alert()` still schedules `run_incident()` and moves on without waiting for it to finish.

Denial and expiry hit the same branch in `resume_incident`. From the playbook's point of view, "a human said no" and "no human answered in time" are the same outcome: the gated action does not run, and whatever comes next in the playbook still does.

<a id="step-285"></a>
## Step 28.5 - Escalation router

```python
# backend/routers/escalation.py

from fastapi import APIRouter, Depends, HTTPException

from backend.escalation.escalation_queue import decide_approval, list_pending
from backend.escalation.escalation_schema import ApprovalDecision
from backend.incidents.runner import resume_incident, sweep_expired_and_resume
from backend.tenancy.context import TenantContext, get_tenant_context

router = APIRouter(prefix="/escalations", tags=["Escalations"])


def _require_admin(ctx: TenantContext) -> None:
    if ctx.user_role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required")


@router.get("")
def get_pending_approvals(ctx: TenantContext = Depends(get_tenant_context)):
    _require_admin(ctx)
    approvals = list_pending(ctx.org_id)
    return [a.to_dict() for a in approvals]


@router.post("/{approval_id}/decide")
def decide(
    approval_id: str,
    body:        ApprovalDecision,
    ctx:         TenantContext = Depends(get_tenant_context),
):
    _require_admin(ctx)
    approval = decide_approval(
        ctx.org_id, approval_id, body.approve, decided_by=ctx.user_id, note=body.note,
    )
    if approval is None:
        raise HTTPException(status_code=404, detail="Approval not found or already decided")

    record = resume_incident(ctx.org_id, approval.incident_id)
    return {
        "approval": approval.to_dict(),
        "incident": record.to_dict() if record else None,
    }


@router.post("/sweep")
def sweep(ctx: TenantContext = Depends(get_tenant_context)):
    """
    Resumes any incident whose approval went past its deadline unanswered.
    Meant to be called on a schedule (every 15 minutes is plenty), not on
    every page load.
    """
    _require_admin(ctx)
    resumed = sweep_expired_and_resume(ctx.org_id)
    return {"resumed_incident_count": len(resumed)}
```

`POST /escalations/{approval_id}/decide` calls `resume_incident` immediately after recording the decision, so an admin who approves a frozen budget release sees the actual unfreeze result in the same response, not a status they have to poll for separately.

<a id="step-286"></a>
## Step 28.6 - Notifying a human

```python
# backend/incidents/runner.py (continued)

from backend.incidents.actions import notify_channel


def _notify_approval_requested(record: IncidentRecord, approval) -> None:
    message = (
        f"[INCIDENT {record.incident_id}] Action '{approval.step.action.value}' "
        f"is waiting on your approval. Reason: {approval.reason} "
        f"Decide at POST /escalations/{approval.approval_id}/decide "
        f"before {approval.expires_at.isoformat()} (UTC), or it will be treated as denied."
    )
    notify_channel(record.org_id, message, channel="slack", incident_id=record.incident_id)
```

This reuses `notify_channel` from Day 26 rather than writing a second Slack integration. The message includes the deadline in plain text, since the on-call engineer reading it on a phone at 2 a.m. is not going to do timezone arithmetic on `expires_at` themselves.

<a id="step-287"></a>
## Step 28.7 - Expiry and the sweep job

```python
# backend/incidents/runner.py (continued)

def sweep_expired_and_resume(org_id: str) -> list[IncidentRecord]:
    """
    Finds incidents stuck in AWAITING_APPROVAL whose approval has expired,
    and resumes each one (which records the action as skipped and runs
    whatever steps come after it). Safe to call repeatedly; an incident
    that isn't actually stuck is left alone.
    """
    list_pending(org_id)  # side effect: flips overdue PENDING approvals to EXPIRED

    resumed = []
    for record in _registry.get(org_id, []):
        if record.status != IncidentStatus.AWAITING_APPROVAL:
            continue
        approval = get_approval(org_id, record.pending_approval_id)
        if approval and approval.status == ApprovalStatus.EXPIRED:
            result = resume_incident(org_id, record.incident_id)
            if result:
                resumed.append(result)
    return resumed
```

A four-hour default TTL is long enough that a single missed Slack notification doesn't strand an incident, and short enough that a genuinely urgent freeze doesn't sit unresolved over a weekend. The TTL is a parameter on `create_approval`, so a future playbook step that needs a tighter or looser window can pass its own value without changing the default for everyone else.

The sweep itself is not a background thread inside the app process; it is a plain function meant to be triggered externally, the same way Day 27's red team suite runs on a GitHub Actions cron rather than an in-process scheduler. A simple option is a cron job hitting `POST /escalations/sweep` for each org every fifteen minutes. A more involved option is wiring it into APScheduler or Celery beat if the deployment already runs one of those. Either way, the sweep logic itself stays a single, testable function, independent of whatever triggers it.

<a id="step-288"></a>
## Step 28.8 - Tests

```python
# tests/escalation/test_escalation_queue.py

import os
os.environ["TESTING"] = "1"

from datetime import datetime, timedelta, timezone

from backend.escalation.escalation_queue import (
    create_approval, decide_approval, get_approval, list_pending,
)
from backend.escalation.escalation_schema import ApprovalStatus
from backend.incidents.incident_schema import PlaybookAction, PlaybookStep


def _step():
    return PlaybookStep(PlaybookAction.FREEZE_ORG_BUDGET, requires_approval=True)


def test_create_approval_is_pending():
    approval = create_approval("org1", "inc1", _step(), "test reason")
    assert approval.status == ApprovalStatus.PENDING
    assert approval in list_pending("org1")


def test_list_pending_excludes_decided():
    approval = create_approval("org2", "inc2", _step(), "test reason")
    decide_approval("org2", approval.approval_id, approve=True, decided_by="admin1")
    assert approval not in list_pending("org2")


def test_list_pending_flips_overdue_to_expired():
    approval = create_approval("org3", "inc3", _step(), "test reason", ttl_hours=0)
    approval.expires_at = datetime.now(timezone.utc) - timedelta(seconds=1)
    assert approval not in list_pending("org3")
    assert get_approval("org3", approval.approval_id).status == ApprovalStatus.EXPIRED


def test_decide_approval_twice_returns_none_second_time():
    approval = create_approval("org4", "inc4", _step(), "test reason")
    first  = decide_approval("org4", approval.approval_id, approve=True, decided_by="admin1")
    second = decide_approval("org4", approval.approval_id, approve=False, decided_by="admin2")
    assert first is not None
    assert second is None
    assert get_approval("org4", approval.approval_id).status == ApprovalStatus.APPROVED


def test_decide_approval_unknown_id_returns_none():
    assert decide_approval("org5", "not-a-real-id", approve=True, decided_by="admin1") is None
```

```python
# tests/escalation/test_runner_pause_resume.py

import os
os.environ["TESTING"] = "1"

from unittest.mock import patch

from backend.escalation.escalation_queue import decide_approval, list_pending
from backend.incidents.incident_schema import IncidentStatus
from backend.incidents.runner import resume_incident, run_incident, sweep_expired_and_resume


class _FakeTriggerEvent:
    def __init__(self, org_id, event_type):
        self.org_id = org_id
        self.event_id = "evt-1"
        self.event_type = event_type


@patch("backend.incidents.runner._notify_approval_requested")
@patch("backend.incidents.runner._execute_step")
def test_run_incident_pauses_at_approval_gate(mock_execute, mock_notify):
    mock_execute.return_value.success = True
    event = _FakeTriggerEvent("org-pause", "audit.tamper_detected")

    record = run_incident(event)

    assert record.status == IncidentStatus.AWAITING_APPROVAL
    assert record.pending_approval_id is not None
    # Only PULL_EVIDENCE ran before the gate; FREEZE_ORG_BUDGET did not.
    assert mock_execute.call_count == 1


@patch("backend.incidents.runner._notify_approval_requested")
@patch("backend.incidents.runner._execute_step")
def test_approve_resumes_and_closes_incident(mock_execute, mock_notify):
    mock_execute.return_value.success = True
    event = _FakeTriggerEvent("org-approve", "audit.tamper_detected")
    record = run_incident(event)

    decide_approval("org-approve", record.pending_approval_id, approve=True, decided_by="admin1")
    resumed = resume_incident("org-approve", record.incident_id)

    assert resumed.status == IncidentStatus.CLOSED
    # PULL_EVIDENCE, FREEZE_ORG_BUDGET, NOTIFY_CHANNEL: all three ran.
    assert mock_execute.call_count == 3


@patch("backend.incidents.runner._notify_approval_requested")
@patch("backend.incidents.runner._execute_step")
def test_deny_skips_gated_step_but_continues(mock_execute, mock_notify):
    mock_execute.return_value.success = True
    event = _FakeTriggerEvent("org-deny", "audit.tamper_detected")
    record = run_incident(event)

    decide_approval("org-deny", record.pending_approval_id, approve=False, decided_by="admin1")
    resumed = resume_incident("org-deny", record.incident_id)

    assert resumed.status == IncidentStatus.CLOSED
    # FREEZE_ORG_BUDGET was skipped, so only PULL_EVIDENCE and NOTIFY_CHANNEL
    # went through _execute_step; the skip is recorded as a StepResult directly.
    assert mock_execute.call_count == 2
    skipped_step = resumed.steps[1]
    assert skipped_step.success is False
    assert "denied" in skipped_step.output


@patch("backend.incidents.runner._notify_approval_requested")
@patch("backend.incidents.runner._execute_step")
def test_sweep_resumes_expired_approval(mock_execute, mock_notify):
    mock_execute.return_value.success = True
    event = _FakeTriggerEvent("org-expire", "audit.tamper_detected")
    record = run_incident(event)

    pending = list_pending("org-expire")[0]
    pending.expires_at = pending.requested_at  # force it into the past

    resumed = sweep_expired_and_resume("org-expire")

    assert len(resumed) == 1
    assert resumed[0].status == IncidentStatus.CLOSED


@patch("backend.incidents.runner._notify_approval_requested")
@patch("backend.incidents.runner._execute_step")
def test_resume_before_decision_is_a_noop(mock_execute, mock_notify):
    mock_execute.return_value.success = True
    event = _FakeTriggerEvent("org-noop", "audit.tamper_detected")
    record = run_incident(event)

    result = resume_incident("org-noop", record.incident_id)

    assert result is None
    assert record.status == IncidentStatus.AWAITING_APPROVAL
```

```python
# tests/escalation/test_escalation_router.py

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


def test_list_approvals_requires_admin(client):
    resp = client.get("/escalations", headers=_headers("org-x", "employee"))
    assert resp.status_code == 403


def test_decide_requires_admin(client):
    resp = client.post(
        "/escalations/some-id/decide",
        json={"approve": True},
        headers=_headers("org-x", "employee"),
    )
    assert resp.status_code == 403


def test_decide_unknown_id_returns_404(client):
    resp = client.post(
        "/escalations/not-real/decide",
        json={"approve": True},
        headers=_headers("org-x", "admin"),
    )
    assert resp.status_code == 404
```

`test_resume_before_decision_is_a_noop` matters as much as the approve and deny tests. Without it, a bug where `resume_incident` runs the gated step regardless of whether anyone has actually decided would slip through, defeating the entire point of the gate.

Run them:

```bash
TESTING=1 pytest tests/escalation/ -v
```

<a id="what-you-built"></a>
## What you built today

A `PendingApproval` record that tracks a single gated playbook step from request through decision or expiry, with a TTL that defaults to four hours. A `requires_approval` flag added to `PlaybookStep`, set on `FREEZE_ORG_BUDGET` and `SUSPEND_ACTOR` wherever they appear in a playbook, since both carry direct customer impact. A queue (`escalation_queue.py`) that creates, lists, and decides approvals per org, with double-decision protection and lazy expiry. A new `IncidentStatus.AWAITING_APPROVAL` state and a rewritten incident runner that pauses `_run_steps` at a gated step instead of executing it, then resumes from that exact point once `resume_incident` is called with a decision. A Slack notification on every approval request that includes the deadline and the exact endpoint needed to respond. A `sweep_expired_and_resume` function that finds incidents stuck waiting on an approval that went unanswered and treats the timeout as a denial, so nothing can hang open forever. An admin-only router with `GET /escalations`, `POST /escalations/{id}/decide`, and `POST /escalations/sweep`.

<a id="checklist"></a>
## Day 28 checklist

- [ ] `PlaybookStep` has a `requires_approval` field defaulting to `False`; `FREEZE_ORG_BUDGET` and `SUSPEND_ACTOR` steps in existing playbooks set it to `True`
- [ ] `PendingApproval.is_overdue()` only flips logic for `PENDING` approvals; decided approvals never become `EXPIRED`
- [ ] `create_approval` stores a `PendingApproval` keyed by org with a configurable TTL (default 4 hours)
- [ ] `decide_approval` returns `None` on a second decision attempt for the same approval, rather than overwriting the first
- [ ] `run_incident` stops at the first gated step it reaches and sets `IncidentRecord.status = AWAITING_APPROVAL`; steps before the gate still execute automatically
- [ ] `resume_incident` returns `None` if the approval is still `PENDING`, executes the gated step on `APPROVED`, and records a skip on `DENIED` or `EXPIRED`
- [ ] `resume_incident` continues running the rest of the playbook after handling the gated step, including hitting a second gate if one exists
- [ ] `sweep_expired_and_resume` resumes every incident in an org whose approval has gone past its deadline unanswered
- [ ] `GET /escalations` and `POST /escalations/{id}/decide` both return 403 for non-admin roles
- [ ] `POST /escalations/{id}/decide` with an unknown or already-decided approval ID returns 404
- [ ] `test_resume_before_decision_is_a_noop` passes, confirming the gate cannot be bypassed before a decision exists
- [ ] `TESTING=1 pytest tests/escalation/ -v` passes

---

[Back to repo](https://github.com/GuntruTirupathamma/AI-Governance) - Previous: [DAY27.md](./DAY27.md) - Next: DAY29.md - Governance SLAs and Postmortem Automation
