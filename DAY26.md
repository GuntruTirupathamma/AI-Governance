---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 26 of 35
Previous: DAY25.md - Policy-as-Code: Dynamic Governance Rules
Next: DAY27.md - Governance Red Team: Simulating Attacks and Testing Defenses
---

# Day 26: Incident Response Playbooks and Automated Remediation

## Table of contents

- [The problem with alerts that stop at alerting](#the-problem)
- [What incident response means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 26.1 - Incident and playbook schema](#step-261)
- [Step 26.2 - Hardcoded playbooks for known incident types](#step-262)
- [Step 26.3 - Remediation actions](#step-263)
- [Step 26.4 - The incident runner](#step-264)
- [Step 26.5 - New audit event types for incident lifecycle](#step-265)
- [Step 26.6 - Wiring the runner into dispatch_alert()](#step-266)
- [Step 26.7 - The incidents admin router](#step-267)
- [Step 26.8 - Tests](#step-268)
- [What you built today](#what-you-built)
- [Day 26 checklist](#checklist)

<a id="the-problem"></a>
## The problem with alerts that stop at alerting

Day 22 closed the gap between "event recorded" and "person notified." That was necessary. But look at what happens after the Slack message lands.

`AUDIT_TAMPER` fires at 03:14. PagerDuty pages the on-call. The engineer wakes up and opens a laptop. They need to: pull a compliance export covering the last 24 hours as evidence before anyone can argue the tamper window, set the affected org's budget to zero so no further spending can happen while they investigate, and page the security lead on a separate channel because this one is above the tier where on-call handles it alone. Three manual steps, each requiring a different tool, executed by a half-awake engineer on a Friday night.

The same pattern repeats for `SECRET_LEAK_BLOCKED`: page fires, engineer logs in, checks which actor caused it, suspends that actor's account, pulls evidence, posts to the security channel. `ABUSE_DETECTED` with action=SUSPEND means the suspension already happened (Day 18), but no one has gathered the evidence or briefed the security team yet.

The common thread: Day 22's alert got a human's attention. But the human still has to remember, under pressure and possibly at 3 AM, which steps belong to which incident type. A playbook is just that checklist, made executable so the system runs it automatically, logs every step as an auditable event, and presents the incident record to the engineer when they open their laptop instead of a blank terminal.

A playbook does not replace human judgment. The engineer still decides whether to escalate, whether the remediation was proportional, and whether to reverse any automated action. It removes the dependency on a stressed person remembering which steps belong to which incident and in what order.

<a id="what-it-means-here"></a>
## What incident response means here

An incident in this system starts with a CRITICAL alert dispatched by Day 22's `dispatch_alert()`. Not every alert becomes an incident: INFO and WARNING alerts get delivered to their channels and stop there. CRITICAL alerts additionally trigger a playbook lookup: is there a sequence of remediation steps registered for this event type? If yes, run it.

A `Playbook` is a list of `PlaybookStep` objects. Each step names an action (an enum member) and carries a small `params` dict for context: a suspension reason, a notification message, or a budget value to set. The actions are:

- `SUSPEND_ACTOR`: suspend the `actor_id` from the triggering audit event by calling the abuse detector's suspension path (Day 18)
- `PULL_EVIDENCE`: export a compliance report for the org for the preceding 24 hours using Day 24's report builder, store the result to a local path, and record that path in the incident
- `FREEZE_ORG_BUDGET`: call Day 25's `set_policy()` to set the org's `daily_usd` to 0 and `monthly_usd` to 0, cutting off further spending until an admin reverses it
- `NOTIFY_CHANNEL`: POST a structured incident notification to a named alert channel (Slack or webhook), separate from the original alert, with incident context included
- `LOG_ONLY`: emit an audit event with the incident context and stop; used for lower-severity playbooks where the right response is observability, not action

An `IncidentRecord` tracks: incident ID (UUID), org, trigger event ID and type, playbook name, status (RUNNING/CLOSED/FAILED), a list of `StepResult` objects (one per step: action, success/failure, output string, timestamp), the evidence file path if PULL_EVIDENCE ran, and opened/closed timestamps.

The in-memory `IncidentRegistry` holds recent incidents per org. The audit log is the authoritative record: `INCIDENT_OPENED`, `INCIDENT_STEP_EXECUTED`, and `INCIDENT_CLOSED` events carry the full context, so the compliance report for a bad week will show every incident that ran alongside the events that triggered them.

<a id="what-youll-build"></a>
## What you will build today

- `backend/incidents/incident_schema.py` - `PlaybookAction(str, Enum)`, `PlaybookStep`, `Playbook`, `IncidentStatus(str, Enum)`, `StepResult`, `IncidentRecord`.
- `backend/incidents/playbooks.py` - `PLAYBOOKS: dict[str, Playbook]` with entries for `AUDIT_TAMPER`, `SECRET_LEAK_BLOCKED`, `ACCOUNT_SUSPENDED`, `ABUSE_DETECTED`, and `TENANT_ACCESS_DENIED`.
- `backend/incidents/actions.py` - one sync function per `PlaybookAction` value, called by the runner.
- `backend/incidents/runner.py` - `IncidentRunner` with an in-memory `IncidentRegistry` and a `run(trigger_event)` method.
- Three new `AuditEventType` members: `INCIDENT_OPENED`, `INCIDENT_STEP_EXECUTED`, `INCIDENT_CLOSED`.
- A one-line change to `dispatch_alert()` (Day 22) that triggers the runner for CRITICAL alerts.
- `backend/routers/incidents.py` - `GET /incidents` and `GET /incidents/{incident_id}`, admin-only.
- Tests under `tests/incidents/`.

<a id="step-261"></a>
## Step 26.1 - Incident and playbook schema

```python
# backend/incidents/incident_schema.py

from __future__ import annotations

import uuid
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Any


class PlaybookAction(str, Enum):
    SUSPEND_ACTOR     = "suspend_actor"
    PULL_EVIDENCE     = "pull_evidence"
    FREEZE_ORG_BUDGET = "freeze_org_budget"
    NOTIFY_CHANNEL    = "notify_channel"
    LOG_ONLY          = "log_only"


@dataclass(frozen=True)
class PlaybookStep:
    action: PlaybookAction
    params: dict[str, Any] = field(default_factory=dict)


@dataclass(frozen=True)
class Playbook:
    name:         str
    steps:        tuple[PlaybookStep, ...]
    description:  str = ""


class IncidentStatus(str, Enum):
    RUNNING = "running"
    CLOSED  = "closed"
    FAILED  = "failed"


@dataclass
class StepResult:
    action:    PlaybookAction
    success:   bool
    output:    str
    ran_at:    datetime = field(default_factory=lambda: datetime.now(timezone.utc))

    def to_dict(self) -> dict:
        return {
            "action":  self.action.value,
            "success": self.success,
            "output":  self.output,
            "ran_at":  self.ran_at.isoformat(),
        }


@dataclass
class IncidentRecord:
    incident_id:       str
    org_id:            str
    trigger_event_id:  str
    trigger_event_type: str
    playbook_name:     str
    status:            IncidentStatus
    steps:             list[StepResult] = field(default_factory=list)
    evidence_path:     str | None = None
    opened_at:         datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    closed_at:         datetime | None = None

    def to_dict(self) -> dict:
        return {
            "incident_id":        self.incident_id,
            "org_id":             self.org_id,
            "trigger_event_id":   self.trigger_event_id,
            "trigger_event_type": self.trigger_event_type,
            "playbook_name":      self.playbook_name,
            "status":             self.status.value,
            "steps":              [s.to_dict() for s in self.steps],
            "evidence_path":      self.evidence_path,
            "opened_at":          self.opened_at.isoformat(),
            "closed_at":          self.closed_at.isoformat() if self.closed_at else None,
        }


def new_incident(
    org_id:             str,
    trigger_event_id:   str,
    trigger_event_type: str,
    playbook_name:      str,
) -> IncidentRecord:
    return IncidentRecord(
        incident_id=str(uuid.uuid4()),
        org_id=org_id,
        trigger_event_id=trigger_event_id,
        trigger_event_type=trigger_event_type,
        playbook_name=playbook_name,
        status=IncidentStatus.RUNNING,
    )
```

`PlaybookStep` is frozen so playbooks are truly immutable once defined. `IncidentRecord` is mutable because the runner appends `StepResult` objects as steps execute. The `closed_at` field stays `None` until the runner finishes, which makes an in-progress incident distinguishable from a completed one at a glance.

<a id="step-262"></a>
## Step 26.2 - Hardcoded playbooks for known incident types

```python
# backend/incidents/playbooks.py

from backend.incidents.incident_schema import Playbook, PlaybookAction, PlaybookStep

PLAYBOOKS: dict[str, Playbook] = {

    "audit.tamper_detected": Playbook(
        name="audit.tamper_detected",
        description=(
            "The hash chain does not verify. Preserve evidence before anything else moves, "
            "cut spending to zero so no new tokens are billed against a compromised log, "
            "and escalate immediately."
        ),
        steps=(
            PlaybookStep(PlaybookAction.PULL_EVIDENCE, {"lookback_hours": 24}),
            PlaybookStep(PlaybookAction.FREEZE_ORG_BUDGET, {"reason": "audit chain tamper detected"}),
            PlaybookStep(PlaybookAction.NOTIFY_CHANNEL, {
                "message": "CRITICAL: audit chain tamper detected. Budget frozen. Evidence pulled. Human review required immediately.",
                "channel":  "slack",
            }),
        ),
    ),

    "secret.leak_blocked": Playbook(
        name="secret.leak_blocked",
        description=(
            "A secret was caught in outbound content and blocked. "
            "Suspend the actor to prevent further leaks while the scope of exposure is assessed."
        ),
        steps=(
            PlaybookStep(PlaybookAction.SUSPEND_ACTOR, {"reason": "secret leak in outbound content"}),
            PlaybookStep(PlaybookAction.PULL_EVIDENCE, {"lookback_hours": 4}),
            PlaybookStep(PlaybookAction.NOTIFY_CHANNEL, {
                "message": "Secret leak blocked and actor suspended. Evidence export complete.",
                "channel":  "slack",
            }),
        ),
    ),

    "account.suspended": Playbook(
        name="account.suspended",
        description=(
            "Account was auto-suspended by the abuse detector (Day 18). "
            "Pull evidence so the on-call can review the decision and decide whether to reverse it."
        ),
        steps=(
            PlaybookStep(PlaybookAction.PULL_EVIDENCE, {"lookback_hours": 2}),
            PlaybookStep(PlaybookAction.NOTIFY_CHANNEL, {
                "message": "Account auto-suspended. Evidence export attached to incident.",
                "channel":  "slack",
            }),
        ),
    ),

    "abuse.detected": Playbook(
        name="abuse.detected",
        description=(
            "Abuse pattern detected. The abuse detector may have already throttled or suspended "
            "the actor. Pull evidence regardless and log for compliance."
        ),
        steps=(
            PlaybookStep(PlaybookAction.PULL_EVIDENCE, {"lookback_hours": 2}),
            PlaybookStep(PlaybookAction.LOG_ONLY, {
                "note": "Abuse detected and auto-remediated by abuse detector. Evidence preserved.",
            }),
        ),
    ),

    "tenant.access_denied": Playbook(
        name="tenant.access_denied",
        description=(
            "A cross-tenant access attempt was blocked. If this is the first occurrence it may be "
            "a misconfiguration. Repeated occurrences within a short window are already handled by "
            "Day 21's threshold - this playbook covers CRITICAL-severity triggers."
        ),
        steps=(
            PlaybookStep(PlaybookAction.PULL_EVIDENCE, {"lookback_hours": 1}),
            PlaybookStep(PlaybookAction.NOTIFY_CHANNEL, {
                "message": "Cross-tenant access denied at CRITICAL severity. Evidence pulled.",
                "channel":  "slack",
            }),
        ),
    ),
}


def get_playbook(event_type_value: str) -> Playbook | None:
    return PLAYBOOKS.get(event_type_value)
```

`PLAYBOOKS` is keyed by `AuditEventType.value` strings (not the enum members) for the same reason `ALERT_RULES` moved to string keys in Day 25: the runner receives a deserialized audit record from the database, where event types are already strings. A conversion that might raise `ValueError` is an unnecessary failure point at 03:00.

The `audit.tamper_detected` playbook runs `PULL_EVIDENCE` before `FREEZE_ORG_BUDGET`. Order matters: if the DB is frozen or inaccessible because of the tamper, at least the export was attempted first. `NOTIFY_CHANNEL` is always last: notify after you have something to say, not before.

<a id="step-263"></a>
## Step 26.3 - Remediation actions

Each action function returns a `tuple[bool, str]`: a success flag and an output message. The runner treats a `False` as a failed step, records it, and continues to the next step rather than aborting the whole playbook. A failed `PULL_EVIDENCE` should not prevent `FREEZE_ORG_BUDGET` from running.

```python
# backend/incidents/actions.py

from __future__ import annotations

import logging
from datetime import datetime, timedelta, timezone

logger = logging.getLogger(__name__)


def suspend_actor(org_id: str, actor_id: str, reason: str) -> tuple[bool, str]:
    """
    Suspend actor_id under org_id by writing a CRITICAL AuditEvent
    through the abuse detector's suspension path (Day 18).
    """
    if not actor_id:
        return False, "No actor_id on trigger event; suspension skipped."
    try:
        from backend.security.abuse_detector import force_suspend_actor
        force_suspend_actor(org_id=org_id, actor_id=actor_id, reason=reason)
        return True, f"Actor {actor_id} suspended: {reason}"
    except Exception as exc:
        logger.exception("suspend_actor failed for %s/%s", org_id, actor_id)
        return False, f"Suspension failed: {exc}"


def pull_evidence(org_id: str, lookback_hours: int) -> tuple[bool, str, str | None]:
    """
    Export a compliance report for the preceding lookback_hours and write it to
    /tmp/incidents/{org_id}_{timestamp}.json. Returns (success, message, file_path).
    """
    import json
    import os

    try:
        from backend.compliance.report_builder import build_report
        from backend.compliance.exporters import to_json

        end   = datetime.now(timezone.utc)
        start = end - timedelta(hours=lookback_hours)
        report = build_report(org_id=org_id, start=start, end=end)
        content = to_json(report)

        out_dir = "/tmp/incidents"
        os.makedirs(out_dir, exist_ok=True)
        ts   = end.strftime("%Y%m%dT%H%M%SZ")
        path = f"{out_dir}/{org_id}_{ts}.json"
        with open(path, "w") as fh:
            fh.write(content)

        return True, f"Evidence exported to {path} ({report.summary.total_events} events)", path
    except Exception as exc:
        logger.exception("pull_evidence failed for %s", org_id)
        return False, f"Evidence export failed: {exc}", None


def freeze_org_budget(org_id: str, reason: str) -> tuple[bool, str]:
    """
    Set both daily_usd and monthly_usd to 0 for every role under org_id
    using Day 25's policy store. This blocks all further LLM spend until
    an admin restores limits via PUT /policies/budget_limits/{role}.
    """
    try:
        from backend.policies.policy_store import get_policy, set_policy
        from backend.policies.policy_schema import PolicyTable

        ROLES = ["admin", "operator", "employee", "agent", "unknown"]
        frozen = []
        for role in ROLES:
            existing = get_policy(PolicyTable.BUDGET_LIMITS.value, role) or {}
            updated  = {**existing, "daily_usd": 0.0, "monthly_usd": 0.0, "warn_ratio": 0.0, "description": f"FROZEN: {reason}"}
            set_policy(PolicyTable.BUDGET_LIMITS.value, role, updated, notes=f"incident freeze: {reason}")
            frozen.append(role)

        return True, f"Budget frozen for roles: {', '.join(frozen)}. Reason: {reason}"
    except Exception as exc:
        logger.exception("freeze_org_budget failed for %s", org_id)
        return False, f"Budget freeze failed: {exc}"


def notify_channel(org_id: str, message: str, channel: str, incident_id: str) -> tuple[bool, str]:
    """
    Post an incident notification to a named channel using Day 22's notifiers.
    The message is wrapped with incident context before sending.
    """
    try:
        full_message = f"[INCIDENT {incident_id}] org={org_id} | {message}"

        if channel == "slack":
            from backend.alerting.notifiers import SlackNotifier
            notifier = SlackNotifier(org_id=org_id)
            notifier.send(full_message)
        elif channel == "webhook":
            from backend.alerting.notifiers import WebhookNotifier
            notifier = WebhookNotifier(org_id=org_id)
            notifier.send(full_message)
        else:
            return False, f"Unknown channel: {channel}"

        return True, f"Notification sent to {channel}"
    except Exception as exc:
        logger.exception("notify_channel failed for %s", org_id)
        return False, f"Notification failed: {exc}"


def log_only(org_id: str, note: str, incident_id: str) -> tuple[bool, str]:
    logger.info("Incident %s (org=%s): %s", incident_id, org_id, note)
    return True, f"Logged: {note}"
```

`pull_evidence` writes to `/tmp/incidents/` with a path structured as `{org_id}_{timestamp}.json`. In production this would be an object store path (S3, GCS), but for the tutorial `/tmp` is concrete and testable. The path lands in `IncidentRecord.evidence_path` so the admin endpoint can surface it.

`freeze_org_budget` overwrites all five role tiers for the org. An admin who wants to selectively restore one tier can `PUT /policies/budget_limits/employee` with the previous values. The `FROZEN:` prefix in `description` makes it obvious to an admin reading the policy list that these values were set by an incident, not by normal configuration.

<a id="step-264"></a>
## Step 26.4 - The incident runner

```python
# backend/incidents/runner.py

from __future__ import annotations

import asyncio
import logging
from collections import defaultdict, deque
from datetime import datetime, timezone
from typing import Any

from backend.incidents.actions import (
    freeze_org_budget, log_only, notify_channel, pull_evidence, suspend_actor,
)
from backend.incidents.incident_schema import (
    IncidentRecord, IncidentStatus, PlaybookAction, StepResult, new_incident,
)
from backend.incidents.playbooks import get_playbook

logger = logging.getLogger(__name__)

# In-memory store: org_id -> deque of recent IncidentRecord objects (capped at 50 per org)
_registry: dict[str, deque[IncidentRecord]] = defaultdict(lambda: deque(maxlen=50))


def get_incidents(org_id: str) -> list[IncidentRecord]:
    return list(_registry[org_id])


def get_incident(org_id: str, incident_id: str) -> IncidentRecord | None:
    for record in _registry[org_id]:
        if record.incident_id == incident_id:
            return record
    return None


def run_incident(trigger_event: Any) -> IncidentRecord | None:
    """
    Look up the playbook for trigger_event.event_type. If no playbook exists,
    return None. Otherwise run every step synchronously, recording results, and
    emit INCIDENT_OPENED / INCIDENT_STEP_EXECUTED / INCIDENT_CLOSED audit events.
    """
    event_type = trigger_event.event_type.value if hasattr(trigger_event.event_type, "value") else trigger_event.event_type
    playbook = get_playbook(event_type)
    if playbook is None:
        return None

    org_id    = trigger_event.org_id
    actor_id  = getattr(trigger_event, "actor_id", "") or ""
    event_id  = str(trigger_event.event_id) if hasattr(trigger_event, "event_id") else ""

    incident = new_incident(
        org_id=org_id,
        trigger_event_id=event_id,
        trigger_event_type=event_type,
        playbook_name=playbook.name,
    )
    _registry[org_id].appendleft(incident)

    _emit_incident_event("incident.opened", org_id, actor_id, incident)

    all_ok = True
    for step in playbook.steps:
        result = _execute_step(step, incident, actor_id)
        incident.steps.append(result)
        _emit_incident_event(
            "incident.step_executed", org_id, actor_id, incident,
            payload={"action": step.action.value, "success": result.success, "output": result.output},
        )
        if not result.success:
            all_ok = False
            logger.warning(
                "Incident %s step %s failed: %s",
                incident.incident_id, step.action.value, result.output,
            )

    incident.status   = IncidentStatus.CLOSED if all_ok else IncidentStatus.FAILED
    incident.closed_at = datetime.now(timezone.utc)
    _emit_incident_event("incident.closed", org_id, actor_id, incident)

    return incident


def _execute_step(step, incident: IncidentRecord, actor_id: str) -> StepResult:
    org_id      = incident.org_id
    incident_id = incident.incident_id
    params      = dict(step.params)

    try:
        if step.action == PlaybookAction.SUSPEND_ACTOR:
            ok, msg = suspend_actor(
                org_id=org_id,
                actor_id=actor_id,
                reason=params.get("reason", "incident playbook"),
            )
            return StepResult(action=step.action, success=ok, output=msg)

        elif step.action == PlaybookAction.PULL_EVIDENCE:
            ok, msg, path = pull_evidence(
                org_id=org_id,
                lookback_hours=params.get("lookback_hours", 24),
            )
            if ok and path:
                incident.evidence_path = path
            return StepResult(action=step.action, success=ok, output=msg)

        elif step.action == PlaybookAction.FREEZE_ORG_BUDGET:
            ok, msg = freeze_org_budget(
                org_id=org_id,
                reason=params.get("reason", "incident playbook"),
            )
            return StepResult(action=step.action, success=ok, output=msg)

        elif step.action == PlaybookAction.NOTIFY_CHANNEL:
            ok, msg = notify_channel(
                org_id=org_id,
                message=params.get("message", "Incident response triggered."),
                channel=params.get("channel", "slack"),
                incident_id=incident_id,
            )
            return StepResult(action=step.action, success=ok, output=msg)

        elif step.action == PlaybookAction.LOG_ONLY:
            ok, msg = log_only(
                org_id=org_id,
                note=params.get("note", ""),
                incident_id=incident_id,
            )
            return StepResult(action=step.action, success=ok, output=msg)

        else:
            return StepResult(action=step.action, success=False, output=f"Unknown action: {step.action}")

    except Exception as exc:
        logger.exception("Unhandled error in step %s for incident %s", step.action, incident_id)
        return StepResult(action=step.action, success=False, output=f"Exception: {exc}")


def _emit_incident_event(event_type_str: str, org_id: str, actor_id: str, incident: IncidentRecord, payload: dict | None = None) -> None:
    try:
        from backend.audit.audit_logger import _emit
        from backend.audit.audit_schema import AuditEvent, AuditEventType
        _emit(AuditEvent(
            event_type=AuditEventType(event_type_str),
            org_id=org_id,
            actor_id=actor_id,
            outcome="success" if incident.status != IncidentStatus.FAILED else "failure",
            payload={
                "incident_id":        incident.incident_id,
                "playbook":           incident.playbook_name,
                "trigger_event_type": incident.trigger_event_type,
                **(payload or {}),
            },
        ))
    except Exception:
        logger.exception("Failed to emit incident audit event %s", event_type_str)
```

The registry is a `defaultdict` of `deque(maxlen=50)`. New incidents go to `appendleft`, so `get_incidents()` returns them newest-first. At 50 incidents per org the memory footprint is trivial; in a production system you'd back this with the database. The audit log already has the durable copy; the registry is purely for the admin endpoint to query without hitting the DB.

`_execute_step` catches all exceptions and returns a `StepResult(success=False)` rather than letting an unhandled error in one action stop the rest of the playbook. A failed evidence gather should not prevent the budget freeze. Each step failure is logged as a warning; the incident as a whole is marked FAILED only if any step failed.

`_emit_incident_event` uses a deferred import to avoid the same circular import pattern that `_schedule_dashboard_broadcast` (Day 23) solved. The runner module does not import `audit_logger` at the top level; it defers to inside the helper function.

<a id="step-265"></a>
## Step 26.5 - New audit event types for incident lifecycle

```python
# backend/audit/audit_schema.py (additions to AuditEventType)

    # Incident response (new - Day 26)
    INCIDENT_OPENED       = "incident.opened"
    INCIDENT_STEP_EXECUTED = "incident.step_executed"
    INCIDENT_CLOSED       = "incident.closed"
```

Three event types, not one. This lets Day 24's compliance report correctly count incidents by phase (`by_event_type`), and lets an admin query the audit log for every `incident.step_executed` event belonging to a given incident ID, useful for answering "exactly what did the system do during incident abc-123?"

`INCIDENT_STEP_EXECUTED`'s payload carries `action`, `success`, and `output`. A compliance audit can therefore reconstruct the full sequence of automated actions that ran during an incident from the audit log alone, without touching the in-memory registry.

<a id="step-266"></a>
## Step 26.6 - Wiring the runner into dispatch_alert()

```python
# backend/alerting/dispatcher.py (update dispatch_alert)

from backend.alerting.alert_schema import AlertSeverity

# Inside dispatch_alert(), after the alert is confirmed sent and ALERT_DISPATCHED is emitted:

    if rule.severity == AlertSeverity.CRITICAL:
        _schedule_incident_run(entry)


def _schedule_incident_run(entry) -> None:
    """
    Fire-and-forget: run the incident playbook for CRITICAL alerts.
    Uses the same asyncio.get_running_loop() pattern as _schedule_dashboard_broadcast (Day 23)
    to avoid blocking the alert dispatch path.
    """
    import asyncio

    async def _run() -> None:
        from backend.incidents.runner import run_incident
        run_incident(entry)

    try:
        loop = asyncio.get_running_loop()
        loop.create_task(_run())
    except RuntimeError:
        try:
            from backend.incidents.runner import run_incident
            run_incident(entry)
        except Exception:
            logger.exception("Incident runner failed for event %s", entry.event_type)
```

The pattern is identical to `_schedule_dashboard_broadcast` (Day 23). CRITICAL alerts go through the normal alert dispatch path first (Slack and PagerDuty get their messages), then the incident runner fires as a background task. If the runner raises, the exception is caught and logged; it does not surface to the caller. The alert already succeeded.

`run_incident(entry)` is synchronous. The `async def _run()` wrapper exists only because `loop.create_task()` requires a coroutine. The synchronous work happens inside `run_incident` directly; there's no `await` in `_run()`.

<a id="step-267"></a>
## Step 26.7 - The incidents admin router

```python
# backend/routers/incidents.py

from fastapi import APIRouter, Depends, HTTPException

from backend.incidents.runner import get_incident, get_incidents
from backend.tenancy.context import TenantContext, get_tenant_context

router = APIRouter(prefix="/incidents", tags=["Incidents"])


def _require_admin(ctx: TenantContext) -> None:
    if ctx.user_role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required")


@router.get("")
def list_incidents(ctx: TenantContext = Depends(get_tenant_context)):
    _require_admin(ctx)
    records = get_incidents(ctx.org_id)
    return {
        "org_id":    ctx.org_id,
        "count":     len(records),
        "incidents": [r.to_dict() for r in records],
    }


@router.get("/{incident_id}")
def get_incident_detail(
    incident_id: str,
    ctx: TenantContext = Depends(get_tenant_context),
):
    _require_admin(ctx)
    record = get_incident(ctx.org_id, incident_id)
    if record is None:
        raise HTTPException(status_code=404, detail=f"Incident {incident_id} not found")
    return record.to_dict()
```

```python
# backend/main.py (add alongside other routers)
from backend.routers import incidents

app.include_router(incidents.router)
```

The router is intentionally read-only. Incidents are created by the system, not by the admin. An admin who wants to rerun a playbook can do so by calling `run_incident(entry)` directly in the Python REPL or by triggering the original event again in a test environment; adding a `POST /incidents/{id}/rerun` endpoint would require passing arbitrary audit events through an HTTP body, which adds complexity without much benefit at this stage of the series.

The `GET /incidents` response always reflects the in-memory registry's current state. If the service restarts, the registry is empty; use `GET /compliance/incidents` (Day 24) for durable incident history from the audit log.

<a id="step-268"></a>
## Step 26.8 - Tests

```python
# tests/incidents/test_runner.py

from unittest.mock import MagicMock, patch

from backend.incidents.incident_schema import IncidentStatus, PlaybookAction
from backend.incidents.playbooks import PLAYBOOKS, get_playbook
from backend.incidents.runner import get_incident, get_incidents, run_incident, _registry


def _make_event(event_type_value: str, org_id: str = "acme", actor_id: str = "user1"):
    event = MagicMock()
    event.event_type.value = event_type_value
    event.org_id  = org_id
    event.actor_id = actor_id
    event.event_id = "evt-test-001"
    return event


class TestPlaybooks:
    def test_all_critical_event_types_have_playbooks(self):
        """The five known CRITICAL event types must all have playbooks."""
        expected = {
            "audit.tamper_detected",
            "secret.leak_blocked",
            "account.suspended",
            "abuse.detected",
            "tenant.access_denied",
        }
        for key in expected:
            assert get_playbook(key) is not None, f"No playbook for {key}"

    def test_unknown_event_type_returns_none(self):
        assert get_playbook("session.created") is None

    def test_audit_tamper_playbook_has_pull_evidence_first(self):
        pb = PLAYBOOKS["audit.tamper_detected"]
        assert pb.steps[0].action == PlaybookAction.PULL_EVIDENCE, (
            "PULL_EVIDENCE must be the first step for audit tamper "
            "so evidence is preserved before any freeze or notification."
        )

    def test_audit_tamper_playbook_has_freeze_budget(self):
        pb = PLAYBOOKS["audit.tamper_detected"]
        actions = [s.action for s in pb.steps]
        assert PlaybookAction.FREEZE_ORG_BUDGET in actions


class TestRunner:
    def setup_method(self):
        _registry.clear()

    @patch("backend.incidents.actions.pull_evidence", return_value=(True, "ok", "/tmp/test.json"))
    @patch("backend.incidents.actions.suspend_actor", return_value=(True, "suspended"))
    @patch("backend.incidents.actions.notify_channel", return_value=(True, "notified"))
    @patch("backend.incidents.runner._emit_incident_event")
    def test_run_incident_creates_record(self, mock_emit, mock_notify, mock_suspend, mock_pull):
        event = _make_event("secret.leak_blocked")
        record = run_incident(event)

        assert record is not None
        assert record.org_id == "acme"
        assert record.playbook_name == "secret.leak_blocked"
        assert record.status == IncidentStatus.CLOSED

    @patch("backend.incidents.actions.pull_evidence", return_value=(True, "ok", "/tmp/test.json"))
    @patch("backend.incidents.actions.notify_channel", return_value=(True, "notified"))
    @patch("backend.incidents.runner._emit_incident_event")
    def test_evidence_path_set_after_pull_evidence(self, mock_emit, mock_notify, mock_pull):
        event = _make_event("account.suspended")
        record = run_incident(event)

        assert record.evidence_path == "/tmp/test.json"

    @patch("backend.incidents.actions.pull_evidence", return_value=(False, "db unavailable", None))
    @patch("backend.incidents.actions.freeze_org_budget", return_value=(True, "frozen"))
    @patch("backend.incidents.actions.notify_channel", return_value=(True, "notified"))
    @patch("backend.incidents.runner._emit_incident_event")
    def test_failed_step_does_not_abort_remaining_steps(self, mock_emit, mock_notify, mock_freeze, mock_pull):
        """If PULL_EVIDENCE fails, FREEZE_ORG_BUDGET and NOTIFY_CHANNEL must still run."""
        event = _make_event("audit.tamper_detected")
        record = run_incident(event)

        assert record.status == IncidentStatus.FAILED  # because PULL_EVIDENCE failed
        # freeze and notify still ran
        mock_freeze.assert_called_once()
        mock_notify.assert_called_once()
        # all three steps appear in record
        assert len(record.steps) == 3

    @patch("backend.incidents.actions.pull_evidence", return_value=(True, "ok", "/tmp/test.json"))
    @patch("backend.incidents.actions.notify_channel", return_value=(True, "notified"))
    @patch("backend.incidents.runner._emit_incident_event")
    def test_incident_retrievable_from_registry(self, mock_emit, mock_notify, mock_pull):
        event = _make_event("account.suspended", org_id="globex")
        record = run_incident(event)

        found = get_incident("globex", record.incident_id)
        assert found is not None
        assert found.incident_id == record.incident_id

    def test_no_playbook_returns_none(self):
        event = _make_event("session.created")
        result = run_incident(event)
        assert result is None

    @patch("backend.incidents.runner._emit_incident_event")
    @patch("backend.incidents.actions.pull_evidence", return_value=(True, "ok", "/tmp/t.json"))
    @patch("backend.incidents.actions.log_only", return_value=(True, "logged"))
    def test_emit_called_for_open_and_close(self, mock_log, mock_pull, mock_emit):
        event = _make_event("abuse.detected")
        run_incident(event)

        call_types = [c.args[0] for c in mock_emit.call_args_list]
        assert "incident.opened" in call_types
        assert "incident.closed" in call_types
```

Run them:

```bash
pytest tests/incidents/ -v
```

The test for the `audit.tamper_detected` playbook explicitly asserts `PULL_EVIDENCE` is first. That assertion documents an architectural decision: in the tamper case, evidence preservation takes priority over everything else. If someone reorders the playbook steps without understanding why, this test fails and explains the reason in the assertion message.

`test_failed_step_does_not_abort_remaining_steps` is the most important test here. It verifies that when `PULL_EVIDENCE` fails (which it might if the database is involved in the tamper), the playbook still freezes the budget and sends the notification. Incident response code that stops on the first failure is less reliable than incident response code that keeps going.

<a id="what-you-built"></a>
## What you built today

A `Playbook` type mapping named action steps to a specific incident trigger, with five concrete playbooks for the event types that have historically needed human intervention (`AUDIT_TAMPER`, `SECRET_LEAK_BLOCKED`, `ACCOUNT_SUSPENDED`, `ABUSE_DETECTED`, `TENANT_ACCESS_DENIED`). A set of sync action functions (`suspend_actor`, `pull_evidence`, `freeze_org_budget`, `notify_channel`, `log_only`) that wrap existing system capabilities with a uniform `(success, message)` return contract and full exception isolation. An `IncidentRunner` that chains those actions, appends per-step results to an `IncidentRecord`, continues through failures rather than stopping, and emits `INCIDENT_OPENED`, `INCIDENT_STEP_EXECUTED`, and `INCIDENT_CLOSED` audit events so the full remediation sequence appears in the compliance log. A fire-and-forget hook in `dispatch_alert()` that triggers the runner for every CRITICAL alert, using the same async loop pattern as Day 23's dashboard broadcast. And an admin-only `GET /incidents` router backed by an in-memory per-org registry, capped at 50 records, for the on-call engineer who opens their laptop and wants to see what the system already did before they got there.

<a id="checklist"></a>
## Day 26 checklist

- [ ] `PlaybookAction` has five members: `SUSPEND_ACTOR`, `PULL_EVIDENCE`, `FREEZE_ORG_BUDGET`, `NOTIFY_CHANNEL`, `LOG_ONLY`
- [ ] `PLAYBOOKS` dict has entries for `audit.tamper_detected`, `secret.leak_blocked`, `account.suspended`, `abuse.detected`, `tenant.access_denied`; `audit.tamper_detected` runs `PULL_EVIDENCE` before `FREEZE_ORG_BUDGET`
- [ ] `run_incident(entry)` returns `None` if no playbook exists for the event type; returns a closed `IncidentRecord` if all steps succeed; returns a failed `IncidentRecord` if any step fails, but still runs all remaining steps
- [ ] `PULL_EVIDENCE` stores the file path in `incident.evidence_path` when the export succeeds
- [ ] `FREEZE_ORG_BUDGET` sets `daily_usd=0` and `monthly_usd=0` for all five role tiers under the org via `policy_store.set_policy()`
- [ ] `_emit_incident_event` uses a deferred import of `audit_logger` to avoid circular imports
- [ ] `dispatch_alert()` calls `_schedule_incident_run(entry)` after confirming `rule.severity == CRITICAL`
- [ ] `_schedule_incident_run` uses `asyncio.get_running_loop().create_task()` with a `RuntimeError` fallback, matching the Day 23 dashboard broadcast pattern
- [ ] `AuditEventType.INCIDENT_OPENED`, `INCIDENT_STEP_EXECUTED`, and `INCIDENT_CLOSED` are defined in `audit_schema.py`
- [ ] `GET /incidents` returns the in-memory registry for the requesting org, newest incidents first, admin-only
- [ ] `GET /incidents/{incident_id}` returns 404 if the ID is not in the registry; the audit log remains the durable source for post-restart queries
- [ ] `test_failed_step_does_not_abort_remaining_steps` passes: a failed `PULL_EVIDENCE` in the `audit.tamper_detected` playbook still triggers `FREEZE_ORG_BUDGET` and `NOTIFY_CHANNEL`
- [ ] All tests in `tests/incidents/` pass

---

[Back to repo](https://github.com/GuntruTirupathamma/AI-Governance) - Previous: [DAY25.md](./DAY25.md) - Next: DAY27.md - Governance Red Team: Simulating Attacks and Testing Defenses
