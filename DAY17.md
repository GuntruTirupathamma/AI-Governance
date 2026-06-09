# DAY 17 - Audit trail and tamper-proof logging

> **Series:** AI Governance Engineering - Zero to Production
> **Author:** GuntruTirupathamma
> **Day:** 17 of 35
> **Previous:** [DAY16.md](./DAY16.md) - Full Injection Test Suite (57 Cases)
> **Next:** DAY18.md - Rate Limiting and Abuse Detection

---

## Table of contents

1. [The problem with logging in AI systems](#the-problem)
2. [What tamper-proof means here](#what-tamper-proof-means)
3. [What you will build today](#what-youll-build)
4. [Step 17.1 - The audit event schema](#step-171)
5. [Step 17.2 - Append-only log writer](#step-172)
6. [Step 17.3 - Hash chaining for tamper detection](#step-173)
7. [Step 17.4 - The audit logger service](#step-174)
8. [Step 17.5 - Integrating with existing agents and orchestrator](#step-175)
9. [Step 17.6 - Log verification CLI](#step-176)
10. [Step 17.7 - FastAPI audit endpoints](#step-177)
11. [Step 17.8 - Tests](#step-178)
12. [What you built today](#what-you-built)
13. [Day 17 checklist](#checklist)

---

<a id="the-problem"></a>
## The problem with logging in AI systems

Regular application logging records what happened. That is useful for debugging. It is not enough for governance.

The difference comes down to one question: could someone delete or modify the logs without anyone noticing?

With a standard log file, the answer is yes. A developer with server access can open the file, change a line, save it, and no record of the edit exists. With a database, an admin can run `DELETE FROM audit_log WHERE ...` and the evidence disappears. This is not hypothetical - it is how compliance violations get covered up.

AI governance needs logs that answer a harder question: did any agent take any action that was not authorized, and can we prove what happened even if someone tries to hide it?

That requires three things regular logging does not give you.

First, every log entry needs a hash of its content, plus the hash of the previous entry. If anyone modifies a record, the chain breaks and the break is detectable. This is the same idea as a blockchain, though you do not need a blockchain to do it.

Second, the log store needs to be append-only. No updates. No deletes. New information gets a new record. Old records stay as they are.

Third, every security-relevant action needs to produce a log entry - not just errors and exceptions, but agent spawns, tool calls, approval decisions, HITL outcomes, and policy violations. If it is not logged, it did not happen as far as a compliance audit is concerned.

---

<a id="what-tamper-proof-means"></a>
## What tamper-proof means here

Completely tamper-proof logging requires hardware security modules, write-once storage, and out-of-band verification systems. That is production infrastructure for a regulated environment.

What you are building today is more practical: a hash-chained append-only log that makes tampering detectable. Someone with database access can still modify records, but the moment they do, the chain breaks. Any verification run will catch it. This is the right trade-off for a development environment being built toward production compliance.

The security model:

```
Each log entry stores:
  id           - sequential integer
  event_type   - what happened
  actor_id     - who did it
  payload      - the full event data (JSON)
  content_hash - SHA-256 of (event_type + actor_id + payload + timestamp)
  chain_hash   - SHA-256 of (content_hash + previous_chain_hash)
  timestamp    - when it was recorded

To verify integrity:
  1. Read all entries in order
  2. Recompute content_hash for each entry
  3. Recompute chain_hash using the previous entry's chain_hash
  4. If any computed hash != stored hash, the record was modified
  5. The first mismatch tells you exactly which record was tampered with
```

If you delete a record, the chain breaks at that position. If you modify a record, the chain breaks there. If you insert a fake record, its chain_hash will not match what the next real record expects.

---

<a id="what-youll-build"></a>
## What you will build today

```
backend/audit/
    __init__.py
    audit_schema.py      - Event types and Pydantic models
    audit_writer.py      - Append-only writer with hash chaining
    audit_logger.py      - Service layer used by agents and orchestrator
    audit_verifier.py    - Detects chain breaks
    audit_models.py      - SQLAlchemy ORM model
    verify_cli.py        - CLI tool to verify chain integrity

backend/routers/
    audit.py             - FastAPI endpoints for querying audit logs

tests/audit/
    __init__.py
    test_audit_writer.py
    test_audit_verifier.py
```

---

<a id="step-171"></a>
## Step 17.1 - The audit event schema

Define what events are worth logging. In a governance system, the list is longer than you might expect.

```python
# backend/audit/audit_schema.py
"""
Audit event types and Pydantic models.

Every security-relevant action in the system produces one of these events.
If an action is not listed here, it probably should be.
"""
from enum import Enum
from datetime import datetime
from typing import Any, Optional
from pydantic import BaseModel, Field
import uuid


class AuditEventType(str, Enum):
    # Session lifecycle
    SESSION_CREATED    = "session.created"
    SESSION_CLOSED     = "session.closed"
    SESSION_EXPIRED    = "session.expired"

    # Agent lifecycle
    AGENT_SPAWN_ALLOWED  = "agent.spawn.allowed"
    AGENT_SPAWN_BLOCKED  = "agent.spawn.blocked"
    AGENT_COMPLETED      = "agent.completed"
    AGENT_FAILED         = "agent.failed"

    # Tool calls
    TOOL_CALL_ALLOWED  = "tool.call.allowed"
    TOOL_CALL_BLOCKED  = "tool.call.blocked"

    # Policy decisions
    GOAL_ALLOWED       = "policy.goal.allowed"
    GOAL_BLOCKED       = "policy.goal.blocked"
    PRIVILEGE_ESC      = "policy.privilege_escalation"
    COMM_BLOCKED       = "policy.communication.blocked"

    # Human-in-the-loop
    HITL_CREATED       = "hitl.created"
    HITL_APPROVED      = "hitl.approved"
    HITL_REJECTED      = "hitl.rejected"
    HITL_EXPIRED       = "hitl.expired"

    # Message bus
    MESSAGE_PASSED     = "message.passed"
    MESSAGE_BLOCKED    = "message.blocked"
    MESSAGE_REDACTED   = "message.redacted"

    # Injection and security
    INJECTION_DETECTED = "security.injection_detected"
    PII_DETECTED       = "security.pii_detected"
    PATH_TRAVERSAL     = "security.path_traversal"

    # Authentication
    AUTH_SUCCESS       = "auth.success"
    AUTH_FAILURE       = "auth.failure"

    # Audit system itself
    AUDIT_VERIFY_OK    = "audit.verify.ok"
    AUDIT_VERIFY_FAIL  = "audit.verify.fail"
    AUDIT_TAMPER       = "audit.tamper_detected"


class AuditEvent(BaseModel):
    """
    A single audit event.

    Produced by any component that takes a security-relevant action.
    The writer assigns id, timestamp, and hashes - callers do not set those.
    """
    event_type:  AuditEventType
    actor_id:    str                      # user_id, agent_id, or "system"
    actor_role:  Optional[str] = None
    session_id:  Optional[str] = None
    resource_id: Optional[str] = None    # agent_id, approval_id, file path
    outcome:     str = "unknown"         # "allowed", "blocked", "completed", "failed"
    reason:      Optional[str] = None
    payload:     dict[str, Any] = Field(default_factory=dict)


class AuditRecord(BaseModel):
    """A stored audit record with integrity fields added by the writer."""
    id:           int
    event_id:     str
    event_type:   str
    actor_id:     str
    actor_role:   Optional[str]
    session_id:   Optional[str]
    resource_id:  Optional[str]
    outcome:      str
    reason:       Optional[str]
    payload:      dict[str, Any]
    timestamp:    datetime
    content_hash: str
    chain_hash:   str
```

---

<a id="step-172"></a>
## Step 17.2 - Append-only log writer

The writer does two things: compute hashes correctly, and never overwrite a record.

```python
# backend/audit/audit_writer.py
"""
Append-only audit log writer with hash chaining.

Rules:
  1. insert() only - no update(), no delete()
  2. Every record stores its own content_hash and the running chain_hash
  3. The chain_hash links each record to the previous one
"""
import hashlib
import json
import uuid
from datetime import datetime, timezone

from sqlalchemy.orm import Session

from backend.audit.audit_schema import AuditEvent
from backend.audit.audit_models import AuditLogEntry
from backend.database.db import SessionLocal


GENESIS_HASH = "0" * 64   # Every chain needs a starting point.


def _compute_content_hash(event: AuditEvent, event_id: str, timestamp: datetime) -> str:
    """SHA-256 of the event's deterministic string representation."""
    content = json.dumps({
        "event_id":    event_id,
        "event_type":  event.event_type.value,
        "actor_id":    event.actor_id,
        "actor_role":  event.actor_role,
        "session_id":  event.session_id,
        "resource_id": event.resource_id,
        "outcome":     event.outcome,
        "reason":      event.reason,
        "payload":     event.payload,
        "timestamp":   timestamp.isoformat(),
    }, sort_keys=True)
    return hashlib.sha256(content.encode()).hexdigest()


def _compute_chain_hash(content_hash: str, prev_chain_hash: str) -> str:
    """SHA-256 of the current content hash chained to the previous chain hash."""
    combined = f"{content_hash}{prev_chain_hash}"
    return hashlib.sha256(combined.encode()).hexdigest()


def _get_last_chain_hash(db: Session) -> str:
    """Get the chain_hash from the most recent record, or the genesis hash."""
    last = (
        db.query(AuditLogEntry)
        .order_by(AuditLogEntry.id.desc())
        .first()
    )
    return last.chain_hash if last else GENESIS_HASH


class AuditWriter:
    """Writes audit events to the database as append-only records."""

    def write(self, event: AuditEvent) -> AuditLogEntry:
        """Write a single audit event. Thread-safe via DB transaction."""
        db = SessionLocal()
        try:
            event_id  = str(uuid.uuid4())
            timestamp = datetime.now(timezone.utc)

            content_hash = _compute_content_hash(event, event_id, timestamp)
            prev_hash    = _get_last_chain_hash(db)
            chain_hash   = _compute_chain_hash(content_hash, prev_hash)

            entry = AuditLogEntry(
                event_id     = event_id,
                event_type   = event.event_type.value,
                actor_id     = event.actor_id,
                actor_role   = event.actor_role,
                session_id   = event.session_id,
                resource_id  = event.resource_id,
                outcome      = event.outcome,
                reason       = event.reason,
                payload      = json.dumps(event.payload),
                timestamp    = timestamp,
                content_hash = content_hash,
                chain_hash   = chain_hash,
            )
            db.add(entry)
            db.commit()
            db.refresh(entry)
            return entry

        finally:
            db.close()

    def write_batch(self, events: list[AuditEvent]) -> list[AuditLogEntry]:
        """Write multiple events atomically. Preserves chain order."""
        db = SessionLocal()
        try:
            entries   = []
            prev_hash = _get_last_chain_hash(db)

            for event in events:
                event_id     = str(uuid.uuid4())
                timestamp    = datetime.now(timezone.utc)
                content_hash = _compute_content_hash(event, event_id, timestamp)
                chain_hash   = _compute_chain_hash(content_hash, prev_hash)

                entry = AuditLogEntry(
                    event_id     = event_id,
                    event_type   = event.event_type.value,
                    actor_id     = event.actor_id,
                    actor_role   = event.actor_role,
                    session_id   = event.session_id,
                    resource_id  = event.resource_id,
                    outcome      = event.outcome,
                    reason       = event.reason,
                    payload      = json.dumps(event.payload),
                    timestamp    = timestamp,
                    content_hash = content_hash,
                    chain_hash   = chain_hash,
                )
                db.add(entry)
                entries.append(entry)
                prev_hash = chain_hash

            db.commit()
            for e in entries:
                db.refresh(e)
            return entries

        finally:
            db.close()
```

---

<a id="step-173"></a>
## Step 17.3 - Hash chaining for tamper detection

The verifier is the most important piece. It reads every record and recomputes the chain from scratch.

```python
# backend/audit/audit_verifier.py
"""
Audit log verifier.

Reads every record from the database and recomputes the hash chain.
Any mismatch means a record was modified, deleted, or a fake was inserted.
All three show up as a chain break at a specific record ID.
"""
import hashlib
import json
from dataclasses import dataclass
from typing import Optional

from backend.audit.audit_models import AuditLogEntry
from backend.audit.audit_writer import GENESIS_HASH
from backend.database.db import SessionLocal


@dataclass
class VerificationResult:
    ok:            bool
    total_records: int
    first_bad_id:  Optional[int]
    error_type:    Optional[str]   # "content_hash", "chain_hash", "sequence_gap"
    message:       str


def verify_chain() -> VerificationResult:
    """
    Walk the entire audit log and verify hash integrity.

    If ok is False, first_bad_id tells you which record is corrupted
    and error_type tells you how.
    """
    db = SessionLocal()
    try:
        records = (
            db.query(AuditLogEntry)
            .order_by(AuditLogEntry.id.asc())
            .all()
        )

        if not records:
            return VerificationResult(
                ok=True, total_records=0,
                first_bad_id=None, error_type=None,
                message="Log is empty. Nothing to verify.",
            )

        prev_chain_hash = GENESIS_HASH
        prev_id         = None

        for record in records:
            # Check for sequence gaps (deleted records)
            if prev_id is not None and record.id != prev_id + 1:
                return VerificationResult(
                    ok=False,
                    total_records=len(records),
                    first_bad_id=record.id,
                    error_type="sequence_gap",
                    message=(
                        f"Sequence gap between records {prev_id} and {record.id}. "
                        f"One or more records were deleted."
                    ),
                )

            # Recompute content hash
            payload_dict = (
                json.loads(record.payload)
                if isinstance(record.payload, str)
                else record.payload
            )
            content_str = json.dumps({
                "event_id":    record.event_id,
                "event_type":  record.event_type,
                "actor_id":    record.actor_id,
                "actor_role":  record.actor_role,
                "session_id":  record.session_id,
                "resource_id": record.resource_id,
                "outcome":     record.outcome,
                "reason":      record.reason,
                "payload":     payload_dict,
                "timestamp":   (
                    record.timestamp.isoformat()
                    if hasattr(record.timestamp, "isoformat")
                    else record.timestamp
                ),
            }, sort_keys=True)
            expected_content_hash = hashlib.sha256(content_str.encode()).hexdigest()

            if record.content_hash != expected_content_hash:
                return VerificationResult(
                    ok=False,
                    total_records=len(records),
                    first_bad_id=record.id,
                    error_type="content_hash",
                    message=(
                        f"Content hash mismatch at record {record.id} "
                        f"(event_type={record.event_type}). "
                        f"The record was modified after it was written."
                    ),
                )

            # Recompute chain hash
            combined = f"{record.content_hash}{prev_chain_hash}"
            expected_chain_hash = hashlib.sha256(combined.encode()).hexdigest()

            if record.chain_hash != expected_chain_hash:
                return VerificationResult(
                    ok=False,
                    total_records=len(records),
                    first_bad_id=record.id,
                    error_type="chain_hash",
                    message=(
                        f"Chain hash mismatch at record {record.id}. "
                        f"Either this record or a prior record was tampered with."
                    ),
                )

            prev_chain_hash = record.chain_hash
            prev_id         = record.id

        return VerificationResult(
            ok=True,
            total_records=len(records),
            first_bad_id=None,
            error_type=None,
            message=f"Chain intact. {len(records)} records verified.",
        )

    finally:
        db.close()


def verify_session(session_id: str) -> VerificationResult:
    """Verify content hashes for all records in a specific session."""
    db = SessionLocal()
    try:
        records = (
            db.query(AuditLogEntry)
            .filter(AuditLogEntry.session_id == session_id)
            .order_by(AuditLogEntry.id.asc())
            .all()
        )

        if not records:
            return VerificationResult(
                ok=True, total_records=0,
                first_bad_id=None, error_type=None,
                message=f"No records found for session {session_id}.",
            )

        for record in records:
            payload_dict = (
                json.loads(record.payload)
                if isinstance(record.payload, str)
                else record.payload
            )
            content_str = json.dumps({
                "event_id":    record.event_id,
                "event_type":  record.event_type,
                "actor_id":    record.actor_id,
                "actor_role":  record.actor_role,
                "session_id":  record.session_id,
                "resource_id": record.resource_id,
                "outcome":     record.outcome,
                "reason":      record.reason,
                "payload":     payload_dict,
                "timestamp":   (
                    record.timestamp.isoformat()
                    if hasattr(record.timestamp, "isoformat")
                    else record.timestamp
                ),
            }, sort_keys=True)
            expected = hashlib.sha256(content_str.encode()).hexdigest()

            if record.content_hash != expected:
                return VerificationResult(
                    ok=False,
                    total_records=len(records),
                    first_bad_id=record.id,
                    error_type="content_hash",
                    message=f"Record {record.id} in session {session_id} was modified.",
                )

        return VerificationResult(
            ok=True,
            total_records=len(records),
            first_bad_id=None,
            error_type=None,
            message=f"Session {session_id}: {len(records)} records verified.",
        )

    finally:
        db.close()
```

---

<a id="step-174"></a>
## Step 17.4 - The audit logger service

The writer handles hashing. The logger handles vocabulary: it translates domain actions into structured audit events.

```python
# backend/audit/audit_logger.py
"""
Audit logger service.

Agents, the orchestrator, and the message bus call this directly.
It translates domain actions into AuditEvents, then hands them to AuditWriter.
"""
from typing import Optional

from backend.audit.audit_schema import AuditEvent, AuditEventType
from backend.audit.audit_writer import AuditWriter

_writer = AuditWriter()


class AuditLogger:

    # Session events

    def session_created(self, session_id: str, user_id: str, role: str, goal: str):
        _writer.write(AuditEvent(
            event_type = AuditEventType.SESSION_CREATED,
            actor_id   = user_id,
            actor_role = role,
            session_id = session_id,
            outcome    = "created",
            payload    = {"goal_length": len(goal)},
        ))

    def session_closed(self, session_id: str, user_id: str, role: str, outcome: str):
        _writer.write(AuditEvent(
            event_type = AuditEventType.SESSION_CLOSED,
            actor_id   = user_id,
            actor_role = role,
            session_id = session_id,
            outcome    = outcome,
        ))

    # Agent spawn events

    def spawn_allowed(
        self, session_id: str, user_id: str, role: str,
        agent_id: str, agent_type: str, risk_level: str,
    ):
        _writer.write(AuditEvent(
            event_type  = AuditEventType.AGENT_SPAWN_ALLOWED,
            actor_id    = user_id,
            actor_role  = role,
            session_id  = session_id,
            resource_id = agent_id,
            outcome     = "allowed",
            payload     = {"agent_type": agent_type, "risk_level": risk_level},
        ))

    def spawn_blocked(
        self, session_id: str, user_id: str, role: str,
        agent_type: str, reason: str,
    ):
        _writer.write(AuditEvent(
            event_type = AuditEventType.AGENT_SPAWN_BLOCKED,
            actor_id   = user_id,
            actor_role = role,
            session_id = session_id,
            outcome    = "blocked",
            reason     = reason,
            payload    = {"agent_type": agent_type},
        ))

    # Policy events

    def goal_blocked(self, user_id: str, role: str, session_id: str, reason: str):
        _writer.write(AuditEvent(
            event_type = AuditEventType.GOAL_BLOCKED,
            actor_id   = user_id,
            actor_role = role,
            session_id = session_id,
            outcome    = "blocked",
            reason     = reason,
        ))

    def privilege_escalation(
        self, user_id: str, current_role: str,
        requested_role: str, session_id: str,
    ):
        _writer.write(AuditEvent(
            event_type = AuditEventType.PRIVILEGE_ESC,
            actor_id   = user_id,
            actor_role = current_role,
            session_id = session_id,
            outcome    = "blocked",
            reason     = f"Escalation from '{current_role}' to '{requested_role}' denied",
            payload    = {"requested_role": requested_role},
        ))

    # HITL events

    def hitl_created(
        self, session_id: str, agent_id: str, approval_id: str,
        severity: str, findings_count: int,
    ):
        _writer.write(AuditEvent(
            event_type  = AuditEventType.HITL_CREATED,
            actor_id    = agent_id,
            session_id  = session_id,
            resource_id = approval_id,
            outcome     = "pending",
            payload     = {"severity": severity, "findings_count": findings_count},
        ))

    def hitl_decided(
        self, approval_id: str, reviewer_id: str,
        decision: str, session_id: str, reason: str = "",
    ):
        event_type = (
            AuditEventType.HITL_APPROVED if decision == "approved"
            else AuditEventType.HITL_REJECTED
        )
        _writer.write(AuditEvent(
            event_type  = event_type,
            actor_id    = reviewer_id,
            session_id  = session_id,
            resource_id = approval_id,
            outcome     = decision,
            reason      = reason,
        ))

    # Message bus events

    def message_blocked(
        self, session_id: str, sender_id: str,
        violations: list[str], was_injection: bool,
    ):
        _writer.write(AuditEvent(
            event_type = AuditEventType.MESSAGE_BLOCKED,
            actor_id   = sender_id,
            session_id = session_id,
            outcome    = "blocked",
            reason     = "; ".join(violations),
            payload    = {"violations": violations, "was_injection": was_injection},
        ))

    def message_redacted(
        self, session_id: str, sender_id: str, pii_types: list[str],
    ):
        _writer.write(AuditEvent(
            event_type = AuditEventType.MESSAGE_REDACTED,
            actor_id   = sender_id,
            session_id = session_id,
            outcome    = "redacted",
            payload    = {"pii_types": pii_types},
        ))

    # Security events

    def injection_detected(
        self, session_id: str, actor_id: str,
        pattern_matched: str, source: str,
    ):
        _writer.write(AuditEvent(
            event_type = AuditEventType.INJECTION_DETECTED,
            actor_id   = actor_id,
            session_id = session_id,
            outcome    = "blocked",
            payload    = {"pattern": pattern_matched, "source": source},
        ))

    def path_traversal(
        self, agent_id: str, session_id: str,
        attempted_path: str, approved_scope: str,
    ):
        _writer.write(AuditEvent(
            event_type = AuditEventType.PATH_TRAVERSAL,
            actor_id   = agent_id,
            session_id = session_id,
            outcome    = "blocked",
            payload    = {"path_outside_scope": True, "approved_scope": approved_scope},
        ))
```

---

<a id="step-175"></a>
## Step 17.5 - Integrating with existing agents and orchestrator

The SQLAlchemy model first:

```python
# backend/audit/audit_models.py
from sqlalchemy import Column, Integer, String, DateTime, Text, Index
from backend.database.db import Base


class AuditLogEntry(Base):
    __tablename__ = "audit_log"

    id           = Column(Integer, primary_key=True, autoincrement=True)
    event_id     = Column(String(36), unique=True, nullable=False)
    event_type   = Column(String(64), nullable=False, index=True)
    actor_id     = Column(String(128), nullable=False, index=True)
    actor_role   = Column(String(64), nullable=True)
    session_id   = Column(String(128), nullable=True, index=True)
    resource_id  = Column(String(256), nullable=True)
    outcome      = Column(String(32), nullable=False)
    reason       = Column(Text, nullable=True)
    payload      = Column(Text, nullable=False, default="{}")
    timestamp    = Column(DateTime, nullable=False, index=True)
    content_hash = Column(String(64), nullable=False)
    chain_hash   = Column(String(64), nullable=False, unique=True)

    __table_args__ = (
        Index("ix_audit_session_type", "session_id", "event_type"),
    )
```

Then plug the logger into three places:

```python
# In backend/orchestrator/orchestrator.py

from backend.audit.audit_logger import AuditLogger
_audit = AuditLogger()

async def orchestrate(self, goal, user_id, user_role, context=None):
    session_id = ...  # as before
    _audit.session_created(session_id, user_id, user_role, goal)
    try:
        # When a spawn is allowed:
        _audit.spawn_allowed(session_id, user_id, user_role,
                             agent_id, agent_type, risk_level)
        # When a spawn is blocked:
        _audit.spawn_blocked(session_id, user_id, user_role, agent_type, reason)

        _audit.session_closed(session_id, user_id, user_role, "completed")
        return result
    except Exception as e:
        _audit.session_closed(session_id, user_id, user_role, "failed")
        raise
```

```python
# In backend/orchestrator/message_bus.py

from backend.audit.audit_logger import AuditLogger
_audit = AuditLogger()

def inspect_message(self, sender_id, receiver_id, content, session_id):
    result = { ... }  # existing logic

    if not result["passed"]:
        _audit.message_blocked(
            session_id=session_id, sender_id=sender_id,
            violations=result["violations"],
            was_injection="injection" in str(result.get("violations", [])).lower(),
        )
    elif result["was_redacted"]:
        _audit.message_redacted(
            session_id=session_id, sender_id=sender_id,
            pii_types=[v["type"] for v in result.get("violations", [])],
        )
    return result
```

```python
# In backend/agents/code_review_agent.py

from backend.audit.audit_logger import AuditLogger
_audit = AuditLogger()

def _safe_path(self, path: str) -> str:
    resolved = os.path.realpath(path)
    if not resolved.startswith(self.approved_scope):
        _audit.path_traversal(
            agent_id=self.agent_id,
            session_id=getattr(self, "session_id", "unknown"),
            attempted_path=path,
            approved_scope=self.approved_scope,
        )
        raise PermissionError(f"Path '{path}' is outside approved scope.")
    return resolved
```

---

<a id="step-176"></a>
## Step 17.6 - Log verification CLI

A verifier you can only call from code is not useful operationally. You need something you can run from a terminal.

```python
# backend/audit/verify_cli.py
"""
CLI for verifying audit log integrity.

  python -m backend.audit.verify_cli
  python -m backend.audit.verify_cli --session SESSION_ID
  python -m backend.audit.verify_cli --json
"""
import argparse
import sys

from backend.audit.audit_verifier import verify_chain, verify_session
from backend.database.db import create_tables


def main():
    parser = argparse.ArgumentParser(description="Verify audit log integrity")
    parser.add_argument("--session", help="Verify a specific session only")
    parser.add_argument("--json",    action="store_true", help="Output JSON")
    args = parser.parse_args()

    create_tables()

    result = verify_session(args.session) if args.session else verify_chain()

    if args.json:
        import json
        print(json.dumps({
            "ok":            result.ok,
            "total_records": result.total_records,
            "first_bad_id":  result.first_bad_id,
            "error_type":    result.error_type,
            "message":       result.message,
        }, indent=2))
    else:
        status = "OK" if result.ok else "FAIL"
        print(f"\n[{status}] {result.message}")
        print(f"      Records checked: {result.total_records}")
        if not result.ok:
            print(f"      First bad record: #{result.first_bad_id}")
            print(f"      Error type:       {result.error_type}")
            sys.exit(1)

    sys.exit(0)


if __name__ == "__main__":
    main()
```

Usage:

```bash
# Verify the full chain
python -m backend.audit.verify_cli

# Verify one session
python -m backend.audit.verify_cli --session abc-123

# CI-friendly output
python -m backend.audit.verify_cli --json

# [OK]   Chain intact. 1,247 records verified.
# [FAIL] Content hash mismatch at record 892 (event_type=hitl.approved).
```

---

<a id="step-177"></a>
## Step 17.7 - FastAPI audit endpoints

```python
# backend/routers/audit.py
"""
Audit log query endpoints. Read-only.
No write or delete operations are exposed here.
"""
from datetime import datetime
from typing import Optional

from fastapi import APIRouter, Query, HTTPException
from pydantic import BaseModel

from backend.audit.audit_models import AuditLogEntry
from backend.audit.audit_verifier import verify_chain, verify_session
from backend.database.db import SessionLocal

router = APIRouter(prefix="/audit", tags=["audit"])


class AuditRecordOut(BaseModel):
    id:           int
    event_id:     str
    event_type:   str
    actor_id:     str
    actor_role:   Optional[str]
    session_id:   Optional[str]
    resource_id:  Optional[str]
    outcome:      str
    reason:       Optional[str]
    timestamp:    datetime
    content_hash: str
    chain_hash:   str


@router.get("/", response_model=list[AuditRecordOut])
def list_audit_records(
    session_id: Optional[str]      = Query(None),
    event_type: Optional[str]      = Query(None),
    actor_id:   Optional[str]      = Query(None),
    since:      Optional[datetime] = Query(None),
    limit:      int                = Query(100, le=1000),
    offset:     int                = Query(0),
):
    db = SessionLocal()
    try:
        q = db.query(AuditLogEntry).order_by(AuditLogEntry.id.desc())
        if session_id: q = q.filter(AuditLogEntry.session_id == session_id)
        if event_type: q = q.filter(AuditLogEntry.event_type == event_type)
        if actor_id:   q = q.filter(AuditLogEntry.actor_id   == actor_id)
        if since:      q = q.filter(AuditLogEntry.timestamp  >= since)
        return q.offset(offset).limit(limit).all()
    finally:
        db.close()


@router.get("/verify")
def verify_log_integrity(session_id: Optional[str] = Query(None)):
    """Run chain verification. Returns 200 if intact, 409 if tampered."""
    result = verify_session(session_id) if session_id else verify_chain()
    if not result.ok:
        raise HTTPException(
            status_code=409,
            detail={
                "ok":           False,
                "error_type":   result.error_type,
                "first_bad_id": result.first_bad_id,
                "message":      result.message,
            }
        )
    return {
        "ok":            True,
        "total_records": result.total_records,
        "message":       result.message,
    }


@router.get("/stats")
def audit_stats():
    """Event counts by type for the last 24 hours."""
    from sqlalchemy import func
    from datetime import timedelta, timezone

    db = SessionLocal()
    try:
        since = datetime.now(timezone.utc) - timedelta(hours=24)
        rows  = (
            db.query(AuditLogEntry.event_type, func.count(AuditLogEntry.id))
            .filter(AuditLogEntry.timestamp >= since)
            .group_by(AuditLogEntry.event_type)
            .all()
        )
        return {"period": "last_24h", "counts": {r[0]: r[1] for r in rows}}
    finally:
        db.close()


@router.get("/security-events")
def security_events(limit: int = Query(50, le=200)):
    """Recent security events only - blocks, injections, escalations."""
    SECURITY_TYPES = [
        "security.injection_detected",
        "security.pii_detected",
        "security.path_traversal",
        "policy.privilege_escalation",
        "policy.goal.blocked",
        "agent.spawn.blocked",
        "message.blocked",
        "hitl.rejected",
    ]
    db = SessionLocal()
    try:
        return (
            db.query(AuditLogEntry)
            .filter(AuditLogEntry.event_type.in_(SECURITY_TYPES))
            .order_by(AuditLogEntry.id.desc())
            .limit(limit)
            .all()
        )
    finally:
        db.close()
```

Register in `backend/main.py` and bump the version to "0.5.0".

---

<a id="step-178"></a>
## Step 17.8 - Tests

```python
# tests/audit/test_audit_writer.py
import pytest
import hashlib
from backend.audit.audit_writer import AuditWriter, GENESIS_HASH
from backend.audit.audit_schema import AuditEvent, AuditEventType
from backend.database.db import create_tables, SessionLocal
from backend.audit.audit_models import AuditLogEntry


@pytest.fixture(autouse=True)
def fresh_db():
    create_tables()
    db = SessionLocal()
    db.query(AuditLogEntry).delete()
    db.commit()
    db.close()


class TestAuditWriter:

    def setup_method(self):
        self.writer = AuditWriter()

    def _event(self):
        return AuditEvent(
            event_type=AuditEventType.SESSION_CREATED,
            actor_id="test-user", actor_role="engineer",
            session_id="test-session", outcome="created",
        )

    def test_write_stores_record(self):
        entry = self.writer.write(self._event())
        assert entry.id is not None
        assert len(entry.content_hash) == 64
        assert len(entry.chain_hash)   == 64

    def test_first_record_chains_to_genesis(self):
        entry    = self.writer.write(self._event())
        expected = hashlib.sha256(
            f"{entry.content_hash}{GENESIS_HASH}".encode()
        ).hexdigest()
        assert entry.chain_hash == expected

    def test_second_record_chains_to_first(self):
        first  = self.writer.write(self._event())
        second = self.writer.write(self._event())
        expected = hashlib.sha256(
            f"{second.content_hash}{first.chain_hash}".encode()
        ).hexdigest()
        assert second.chain_hash == expected

    def test_different_events_produce_different_hashes(self):
        e1 = self.writer.write(AuditEvent(
            event_type=AuditEventType.AGENT_SPAWN_ALLOWED,
            actor_id="user-a", outcome="allowed",
        ))
        e2 = self.writer.write(AuditEvent(
            event_type=AuditEventType.AGENT_SPAWN_BLOCKED,
            actor_id="user-b", outcome="blocked",
        ))
        assert e1.content_hash != e2.content_hash

    def test_write_batch_preserves_order(self):
        events  = [
            AuditEvent(event_type=AuditEventType.SESSION_CREATED,
                       actor_id=f"user-{i}", outcome="created")
            for i in range(5)
        ]
        entries = self.writer.write_batch(events)
        assert len(entries) == 5
        for i, entry in enumerate(entries):
            assert entry.actor_id == f"user-{i}"


# tests/audit/test_audit_verifier.py
import pytest
from backend.audit.audit_writer import AuditWriter
from backend.audit.audit_verifier import verify_chain
from backend.audit.audit_schema import AuditEvent, AuditEventType
from backend.database.db import create_tables, SessionLocal
from backend.audit.audit_models import AuditLogEntry


@pytest.fixture(autouse=True)
def fresh_db():
    create_tables()
    db = SessionLocal()
    db.query(AuditLogEntry).delete()
    db.commit()
    db.close()


class TestAuditVerifier:

    def setup_method(self):
        self.writer = AuditWriter()

    def _write(self, n=3):
        for i in range(n):
            self.writer.write(AuditEvent(
                event_type=AuditEventType.SESSION_CREATED,
                actor_id=f"user-{i}", outcome="created",
            ))

    def test_empty_log_passes(self):
        result = verify_chain()
        assert result.ok is True

    def test_intact_chain_passes(self):
        self._write(5)
        result = verify_chain()
        assert result.ok is True
        assert result.total_records == 5

    def test_modified_record_detected(self):
        self._write(3)
        db = SessionLocal()
        try:
            record = db.query(AuditLogEntry).order_by(AuditLogEntry.id).offset(1).first()
            record.actor_id = "attacker-modified"
            db.commit()
        finally:
            db.close()

        result = verify_chain()
        assert result.ok is False
        assert result.error_type == "content_hash"

    def test_deleted_record_detected(self):
        self._write(5)
        db = SessionLocal()
        try:
            record = db.query(AuditLogEntry).order_by(AuditLogEntry.id).offset(2).first()
            db.delete(record)
            db.commit()
        finally:
            db.close()

        result = verify_chain()
        assert result.ok is False
        assert result.error_type == "sequence_gap"

    def test_first_bad_record_id_reported(self):
        self._write(5)
        db = SessionLocal()
        try:
            records   = db.query(AuditLogEntry).order_by(AuditLogEntry.id).all()
            target    = records[2]
            target_id = target.id
            target.outcome = "tampered"
            db.commit()
        finally:
            db.close()

        result = verify_chain()
        assert result.ok is False
        assert result.first_bad_id == target_id
```

Run the tests:

```bash
pytest tests/audit/ -v --tb=short

# tests/audit/test_audit_writer.py   5 passed
# tests/audit/test_audit_verifier.py 5 passed
```

---

<a id="what-you-built"></a>
## What you built today

```
backend/audit/
  audit_schema.py      31 event types across 8 categories
  audit_models.py      SQLAlchemy model with content_hash and chain_hash
  audit_writer.py      Append-only writer with hash chaining
  audit_verifier.py    Chain verifier (detects modify, delete, insert)
  audit_logger.py      Domain service used by agents and orchestrator
  verify_cli.py        CLI for manual and CI verification

backend/routers/audit.py
  GET /audit/                 Query with filters
  GET /audit/verify           Run chain verification (200 ok, 409 tampered)
  GET /audit/stats            Event counts by type for last 24 hours
  GET /audit/security-events  Recent blocks, injections, escalations

Tests: 10 (5 writer + 5 verifier)
```

What the chain catches:

```
Modified record content    content_hash mismatch
Deleted record             sequence_gap between adjacent IDs
Inserted fake record       chain_hash mismatch on the next real record
Replaced chain_hash        chain_hash mismatch
```

One thing it does not catch: a sophisticated attacker with full DB access who recalculates the entire chain after tampering. Preventing that requires out-of-band anchoring - periodic snapshots stored off-site, or write-once storage. That comes in the production hardening section later in this series.

---

<a id="checklist"></a>
## Day 17 checklist

- [ ] `backend/audit/audit_schema.py` created with 31 event types
- [ ] `backend/audit/audit_models.py` created with content_hash and chain_hash columns
- [ ] `backend/audit/audit_writer.py` created - append-only, hash-chained
- [ ] `backend/audit/audit_verifier.py` created - detects modify/delete/insert
- [ ] `backend/audit/audit_logger.py` created - domain service layer
- [ ] `backend/audit/verify_cli.py` created
- [ ] AuditLogger calls added to orchestrator, message bus, code review agent
- [ ] `backend/routers/audit.py` created with 4 endpoints
- [ ] `backend/main.py` updated to version "0.5.0" with audit router
- [ ] `tests/audit/test_audit_writer.py` - 5 tests pass
- [ ] `tests/audit/test_audit_verifier.py` - 5 tests pass
- [ ] `python -m backend.audit.verify_cli` prints OK on a clean log
- [ ] `GET /audit/verify` returns 200 on a clean log
- [ ] Tamper a record manually, re-run verifier, confirm FAIL output

---

*Day 17 done. You now have a complete audit trail that records what every agent did, who authorized it, and whether the record was modified after the fact. Day 18 covers rate limiting and abuse detection.*

---

**Repository:** [GuntruTirupathamma/AI-Governance](https://github.com/GuntruTirupathamma/AI-Governance)
**Series:** AI Governance Engineering from Scratch
**Next:** `DAY18.md` - Rate Limiting and Abuse Detection
