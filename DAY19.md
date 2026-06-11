---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 19 of 35
Previous: DAY18.md - Rate Limiting and Abuse Detection
Next: DAY20.md - Cost Governance and Budget Enforcement
---

# Day 19: Secrets Management and Credential Isolation

## Table of contents

- [The problem with secrets in a multi-agent system](#the-problem)
- [What secrets management means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 19.1 - Secret definitions and per-agent scopes](#step-191)
- [Step 19.2 - The vault](#step-192)
- [Step 19.3 - The credential broker](#step-193)
- [Step 19.4 - The secret-leak scanner](#step-194)
- [Step 19.5 - New audit events for secret access and leaks](#step-195)
- [Step 19.6 - Blocking leaks in the message bus](#step-196)
- [Step 19.7 - Feeding secret events into abuse detection](#step-197)
- [Step 19.8 - Agent integration: GovernedAgent.get_secret()](#step-198)
- [Step 19.9 - Admin visibility: the secret-scopes endpoint](#step-199)
- [Step 19.10 - Tests](#step-1910)
- [What you built today](#what-you-built)
- [Day 19 checklist](#checklist)

<a id="the-problem"></a>
## The problem with secrets in a multi-agent system

Up to Day 18, every agent in this system runs with the same process-level environment. If `OPENAI_API_KEY`, `SEARCH_API_KEY`, `DATABASE_URL`, and `SMTP_CREDENTIALS` are all sitting in `os.environ`, then every agent - the research agent, the code review agent, any agent type you add next month - can read all of them with a single `os.environ.get()` call.

That's not a hypothetical. Day 13 built a static vulnerability scanner that flags hardcoded secrets in *files*. But nothing today stops a *running agent* from doing the equivalent thing at runtime: reading a credential it has no legitimate reason to touch, then either misusing it directly or - more likely, given everything built in Days 15-18 - echoing it back into a message, a tool result, or a final response, where it leaks to another agent, into the audit log, or out to the user.

Two failure modes, concretely:

1. **Over-broad access.** The code review agent has no business holding an OpenAI API key. It reads files and writes findings. If its prompt gets manipulated (Day 15's injection scenarios), the blast radius today includes every secret the process can see, not just the tools that agent is supposed to use.

2. **Accidental leakage.** Even an agent that legitimately holds a credential can leak it. A debugging response that includes "the request failed with header `Authorization: Bearer sk-...`" is a secret leak even though nothing malicious happened - the agent was just trying to be helpful. Day 15/16's `MessageBus.inspect_message()` already redacts PII like SSNs and emails. It does nothing for API keys today.

Day 19 closes both gaps: a **scoped credential broker** so agents can only ever ask for secrets their type is allow-listed for, and a **live leak scanner** wired into the message bus so a credential - whether it matches a known format or is one of *this system's own* configured secrets - never makes it into a message between agents.

<a id="what-it-means-here"></a>
## What secrets management means here

This is not "go set up HashiCorp Vault." That's a real recommendation for production (and the vault module below is written so you can swap in a real backend later without touching any caller), but the governance lesson is independent of which product holds the bytes. The lesson is about the **shape of access**:

- **One reader.** All secret reads go through a single function, `vault.get_secret()`. Nothing else is allowed to call `os.environ.get("OPENAI_API_KEY")` directly - if you can grep your codebase for direct environment reads of credential names and find none outside `backend/secrets/vault.py`, the vault is doing its job.

- **An allow-list, not a block-list.** `AGENT_SECRET_SCOPES` says what each agent type *can* access. Anything not listed is denied. Adding a new agent type means it starts with zero secret access until someone deliberately grants it - the safe default is "no credentials," not "all credentials."

- **Every access is audited - granted or denied.** Day 17 built a hash-chained audit log specifically so that "who accessed what, and when" is tamper-evident. Secret access is exactly the kind of event that belongs there. A pattern of `SECRET_ACCESS_DENIED` events from one agent is itself a signal worth watching - which is why Step 19.7 plugs it straight into Day 18's abuse detector.

- **Leaks are blocked, not redacted.** Day 15/16's PII handling redacts SSNs and emails and lets the (now-clean) message continue. Secrets get different treatment: if a message contains something that looks like - or *is* - a live credential, the message is blocked outright and the sending agent is treated as a security event. The reasoning is in Step 19.6, but the short version is that redact-and-continue is fine for "this text shouldn't be stored," and not fine for "a credential that grants real access has just been exposed."

By the end of today, an agent that tries to read a secret outside its scope gets a clean exception and an audit trail entry. An agent that *has* a secret and accidentally echoes it gets the message blocked before it reaches anyone else, and after a couple of repeats, gets suspended by the same machinery Day 18 built for everything else.

<a id="what-youll-build"></a>
## What you will build today

```
backend/
├── secrets/
│   ├── __init__.py
│   ├── secret_schema.py      # SecretName enum, SECRET_REGISTRY, AGENT_SECRET_SCOPES
│   ├── vault.py               # get_secret(), is_configured(), all_known_values()
│   └── credential_broker.py   # CredentialBroker - the only way agents read secrets
├── security/
│   ├── secret_scanner.py      # find_secrets(), redact_secrets()
│   ├── abuse_schema.py         # (Day 18, extended) new ABUSE_THRESHOLDS entries
│   └── routers/security.py    # (Day 18, extended) GET /security/secret-scopes
├── audit/
│   ├── audit_schema.py         # (Day 17, extended) SECRET_ACCESS_GRANTED/DENIED, SECRET_LEAK_BLOCKED
│   └── audit_logger.py         # (Day 17/18, extended) secret_access_granted/denied, secret_leak_blocked
├── orchestrator/
│   └── message_bus.py          # (Day 15/16, extended) secret-leak check in inspect_message()
└── agents/
    ├── base_agent.py           # (extended) GovernedAgent.get_secret()
    ├── research_agent.py       # (extended) uses self.get_secret(...) instead of os.environ
    └── code_review_agent.py    # (unchanged code, new test proving it has zero secret scope)

tests/
├── secrets/
│   ├── test_vault.py
│   └── test_credential_broker.py
└── security/
    ├── test_secret_scanner.py
    └── test_message_bus_secret_leak.py
```

<a id="step-191"></a>
## Step 19.1 - Secret definitions and per-agent scopes

Everything starts with a registry: what secrets exist, what environment variable backs each one, and which agent types are allowed to ask for them. This file is deliberately the smallest, most-reviewed file in the feature - widening an agent's access is a one-line diff here, which is exactly the point.

```python
# backend/secrets/secret_schema.py
"""
Secret definitions and per-agent-type access scopes.

This is the single place that says which secrets exist, where they live
(which environment variable backs them), and which agent types are
allowed to request them. Adding a new secret or widening an agent's
access means editing this file - and that edit shows up as an
isolated, easy-to-review diff.
"""
from dataclasses import dataclass
from enum import Enum

from backend.orchestrator.agent_registry import AgentType


class SecretName(str, Enum):
    OPENAI_API_KEY    = "openai_api_key"
    ANTHROPIC_API_KEY = "anthropic_api_key"
    SEARCH_API_KEY    = "search_api_key"
    DATABASE_URL      = "database_url"
    SMTP_CREDENTIALS  = "smtp_credentials"


@dataclass(frozen=True)
class SecretDefinition:
    name:        SecretName
    env_var:     str
    description: str


SECRET_REGISTRY: dict[SecretName, SecretDefinition] = {
    SecretName.OPENAI_API_KEY: SecretDefinition(
        SecretName.OPENAI_API_KEY, "OPENAI_API_KEY",
        "OpenAI API key used for governed LLM calls"),
    SecretName.ANTHROPIC_API_KEY: SecretDefinition(
        SecretName.ANTHROPIC_API_KEY, "ANTHROPIC_API_KEY",
        "Anthropic API key used for governed LLM calls"),
    SecretName.SEARCH_API_KEY: SecretDefinition(
        SecretName.SEARCH_API_KEY, "SEARCH_API_KEY",
        "Web search provider key used by the research agent"),
    SecretName.DATABASE_URL: SecretDefinition(
        SecretName.DATABASE_URL, "DATABASE_URL",
        "Primary application database connection string"),
    SecretName.SMTP_CREDENTIALS: SecretDefinition(
        SecretName.SMTP_CREDENTIALS, "SMTP_CREDENTIALS",
        "SMTP username:password used by the notification service"),
}


# Which secrets each actor type is allowed to request. This is an
# allow-list: anything not listed for a given actor type is denied,
# full stop. New agent types start here with an empty list and earn
# scopes deliberately.
#
# "system" is not an AgentType - it's the identifier internal services
# (the orchestrator itself, the notification service) use when they ask
# the broker for credentials. Agents never get to claim to be "system".
AGENT_SECRET_SCOPES: dict[str, list[SecretName]] = {
    AgentType.RESEARCH.value: [
        SecretName.OPENAI_API_KEY,
        SecretName.ANTHROPIC_API_KEY,
        SecretName.SEARCH_API_KEY,
    ],
    AgentType.CODE_REVIEW.value: [
        # Reads files from the workspace and writes findings.
        # No external calls, no secrets needed.
    ],
    "system": [
        SecretName.DATABASE_URL,
        SecretName.SMTP_CREDENTIALS,
    ],
}
```

A few things worth calling out:

- `AgentType.CODE_REVIEW.value` maps to an explicit empty list, not an absent key. `AGENT_SECRET_SCOPES.get(agent_type, [])` would treat a missing key the same way, but writing it out makes the "this agent type has zero secrets, on purpose" decision visible in code review instead of being an accident of a dict that just doesn't mention it.

- `SecretName` values are strings (`"openai_api_key"`, not the env var name `"OPENAI_API_KEY"`). Code never refers to environment variable names directly - `SECRET_REGISTRY` is the only place that mapping exists, which is what makes swapping the storage backend later a one-file change.

<a id="step-192"></a>
## Step 19.2 - The vault

The vault is the only thing in the codebase allowed to call `os.environ.get()` for a credential. Everything else - the broker, agents, the leak scanner - goes through `get_secret()`.

```python
# backend/secrets/vault.py
"""
Secrets vault.

A single, narrow interface for reading secret values: get_secret(name).
Backed by environment variables today. Swapping to AWS Secrets Manager,
HashiCorp Vault, or a similar service later means changing the body of
_read_from_backend() in this file - nothing that calls get_secret()
needs to change.

Hard rules enforced here:
  - Raw secret values are never logged.
  - get_secret() is the only function that returns a value. There is
    deliberately no "list everything" function that dumps the registry
    with values attached.
  - all_known_values() exists for exactly one caller (the secret-leak
    scanner in Step 19.4) and returns values with no identifying labels,
    specifically so it can't be used to enumerate "which secret is
    which" from its output alone.
"""
import os
import logging

from backend.secrets.secret_schema import SecretName, SECRET_REGISTRY

logger = logging.getLogger(__name__)

# In-process cache so repeated lookups don't keep re-touching the
# backend. Cleared only by restarting the process (or explicitly in
# tests).
_cache: dict[SecretName, str] = {}


class SecretNotConfigured(Exception):
    """Raised when a registered secret has no value in the backend."""


def _read_from_backend(env_var: str) -> str | None:
    """
    The only line in this file that touches an external secret store.

    Today: environment variables. A production deployment can replace
    this with a call to AWS Secrets Manager / Vault / etc. Everything
    above and below this function is unaffected by that change.
    """
    return os.environ.get(env_var)


def get_secret(name: SecretName) -> str:
    """
    Return the value of a registered secret.

    Raises SecretNotConfigured if the secret is registered but has no
    value - never returns None or "" silently. A silently-missing
    credential fails in confusing ways three layers up; an exception
    here fails loudly, where the cause is obvious.
    """
    if name in _cache:
        return _cache[name]

    if name not in SECRET_REGISTRY:
        raise ValueError(f"'{name}' is not a registered secret. Add it to SECRET_REGISTRY first.")

    definition = SECRET_REGISTRY[name]
    value = _read_from_backend(definition.env_var)

    if not value:
        raise SecretNotConfigured(
            f"Secret '{name.value}' (env var '{definition.env_var}') is not configured."
        )

    _cache[name] = value
    return value


def is_configured(name: SecretName) -> bool:
    """Check whether a secret has a value, without raising or returning it."""
    try:
        get_secret(name)
        return True
    except SecretNotConfigured:
        return False


def all_known_values() -> set[str]:
    """
    Every configured secret's value, as an unlabeled set.

    Used only by the secret-leak scanner (Step 19.4) to check whether a
    piece of text contains one of this system's own credentials
    verbatim. This function exists for exactly one caller - think hard
    before adding a second.
    """
    values = set()
    for name in SECRET_REGISTRY:
        if is_configured(name):
            values.add(get_secret(name))
    return values
```

<a id="step-193"></a>
## Step 19.3 - The credential broker

The vault knows how to fetch values. The broker knows *who is allowed to ask*. It is the only object agents talk to, and every call - whether it succeeds or fails - leaves an audit trail entry.

```python
# backend/secrets/credential_broker.py
"""
Credential broker.

The only sanctioned way for an agent (or internal service) to obtain a
secret value. Checks the requester's type against AGENT_SECRET_SCOPES
- an allow-list - before ever calling the vault, and logs every grant
and denial through the Day 17 audit pipeline.

backend.secrets.vault knows how to fetch values.
backend.secrets.credential_broker knows who is allowed to ask for them.
"""
from backend.audit.audit_logger import AuditLogger
from backend.secrets.secret_schema import SecretName, AGENT_SECRET_SCOPES
from backend.secrets.vault import get_secret, SecretNotConfigured

_audit = AuditLogger()


class SecretAccessDenied(Exception):
    """Raised when an actor type requests a secret outside its scope."""


class CredentialBroker:

    def get_secret(
        self,
        secret_name: SecretName,
        agent_id: str,
        agent_type: str,
        session_id: str | None = None,
    ) -> str:
        """
        Return a secret value if `agent_type` is scoped for it.

        Raises SecretAccessDenied (and records SECRET_ACCESS_DENIED) if
        `agent_type`'s scope doesn't include `secret_name`.

        Raises SecretNotConfigured if the scope check passes but the
        underlying credential isn't configured in the vault. This is a
        deployment problem, not an authorization problem, so it isn't
        treated as a security event.
        """
        allowed = AGENT_SECRET_SCOPES.get(agent_type, [])

        if secret_name not in allowed:
            _audit.secret_access_denied(
                agent_id=agent_id,
                agent_type=agent_type,
                session_id=session_id,
                secret_name=secret_name.value,
            )
            raise SecretAccessDenied(
                f"Agent type '{agent_type}' is not scoped for secret "
                f"'{secret_name.value}'. Allowed secrets for this type: "
                f"{[s.value for s in allowed]}"
            )

        value = get_secret(secret_name)  # may raise SecretNotConfigured

        _audit.secret_access_granted(
            agent_id=agent_id,
            agent_type=agent_type,
            session_id=session_id,
            secret_name=secret_name.value,
        )
        return value
```

Note the ordering: the scope check happens *before* the vault is touched. A code review agent asking for `OPENAI_API_KEY` gets `SecretAccessDenied` regardless of whether `OPENAI_API_KEY` is even configured - the authorization decision doesn't depend on, and doesn't leak, whether the secret exists.

<a id="step-194"></a>
## Step 19.4 - The secret-leak scanner

This is the piece that catches credentials in *output* - text an agent is about to send to another agent, write to a log, or return to the user. Two layers:

1. **Pattern matching** - well-known credential formats (OpenAI/Anthropic keys, AWS access keys, GitHub tokens, Slack tokens, and a generic `name = value`-style assignment, reusing the regex family Day 13's static scanner already established).
2. **Exact-value matching** - does the text contain one of *this deployment's own* configured secret values, verbatim? This catches things layer 1 can't, like a `DATABASE_URL` with a custom hostname that doesn't match any generic pattern, because the scanner knows the literal value to look for.

```python
# backend/security/secret_scanner.py
"""
Secret-leak scanner.

Two detection layers:

  1. Pattern matching - text that LOOKS like a credential, based on
     well-known formats. Catches secrets even if they aren't this
     system's own (e.g. a key copy-pasted from somewhere else that
     ends up in a debugging message).

  2. Exact-value matching - does the text contain one of THIS
     deployment's configured secret values, verbatim? This is the more
     important layer operationally: it catches a real, live credential
     leaking out even when its format is unremarkable (a DATABASE_URL
     with a custom hostname matches no regex, but we know its exact
     value).

Both layers feed into find_secrets() / redact_secrets(). Step 19.6 uses
find_secrets() to decide whether to BLOCK a message outright; redact_secrets()
is provided for callers (logging, transcripts) that need a scrubbed copy
of text rather than an all-or-nothing decision.
"""
import re
from dataclasses import dataclass

from backend.secrets.vault import all_known_values

REDACTION_PLACEHOLDER = "[REDACTED-SECRET]"

# (label, compiled pattern) - format-based detection. The generic
# assignment pattern reuses the same shape as Day 13's static
# vulnerability scanner, applied here at runtime to live text instead
# of source files.
_PATTERNS: list[tuple[str, re.Pattern]] = [
    ("openai_api_key",    re.compile(r"sk-[A-Za-z0-9]{20,}")),
    ("anthropic_api_key", re.compile(r"sk-ant-[A-Za-z0-9\-_]{20,}")),
    ("aws_access_key",    re.compile(r"AKIA[0-9A-Z]{16}")),
    ("github_token",      re.compile(r"gh[pousr]_[A-Za-z0-9]{30,}")),
    ("slack_token",       re.compile(r"xox[baprs]-[A-Za-z0-9-]{10,}")),
    ("generic_assignment", re.compile(
        r"(?i)(password|passwd|pwd|secret|api[_-]?key|access[_-]?token|"
        r"auth[_-]?token|private[_-]?key)\s*[:=]\s*['\"]?[^\s'\"]{6,}['\"]?"
    )),
]


@dataclass
class SecretMatch:
    label: str          # which pattern matched, or "configured_secret"
    span:  tuple[int, int]  # (start, end) offsets in the original text


def find_secrets(text: str) -> list[SecretMatch]:
    """Return every secret-like match in `text`. Empty list if none found."""
    matches: list[SecretMatch] = []

    for label, pattern in _PATTERNS:
        for m in pattern.finditer(text):
            matches.append(SecretMatch(label=label, span=m.span()))

    for value in all_known_values():
        start = 0
        while True:
            idx = text.find(value, start)
            if idx == -1:
                break
            matches.append(SecretMatch(label="configured_secret", span=(idx, idx + len(value))))
            start = idx + len(value)

    return matches


def redact_secrets(text: str) -> tuple[str, list[SecretMatch]]:
    """
    Replace every detected secret with a placeholder.

    Returns (redacted_text, matches). Overlapping or adjacent spans are
    merged before replacement so a format match and a configured-value
    match on the same substring don't produce a doubled-up placeholder,
    and replacement happens back-to-front so earlier edits don't shift
    the offsets of later ones.
    """
    matches = find_secrets(text)
    if not matches:
        return text, []

    spans = sorted({m.span for m in matches})
    merged: list[list[int]] = []
    for start, end in spans:
        if merged and start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])

    redacted = text
    for start, end in reversed(merged):
        redacted = redacted[:start] + REDACTION_PLACEHOLDER + redacted[end:]

    return redacted, matches
```

One rule that applies everywhere this module's output is used: **never put the matched substring itself into a log line, an audit payload, or an exception message.** Only `label` (a pattern name like `"aws_access_key"`) is safe to record. Step 19.5's audit methods take a list of labels for exactly this reason - if the audit log started storing the leaked values themselves, the audit log would become a second copy of the secret store, with weaker access controls.

<a id="step-195"></a>
## Step 19.5 - New audit events for secret access and leaks

Three new event types, following the same pattern Day 17 established for everything else.

```python
# backend/audit/audit_schema.py (additions to AuditEventType)

class AuditEventType(str, Enum):
    # ... existing session lifecycle, agent lifecycle, tool call,
    #     policy, HITL, message bus, injection/security, auth, and
    #     audit-system events from Days 14-18 ...

    # Secrets management (Day 19)
    SECRET_ACCESS_GRANTED = "secret.access_granted"
    SECRET_ACCESS_DENIED  = "secret.access_denied"
    SECRET_LEAK_BLOCKED   = "secret.leak_blocked"
```

And the corresponding `AuditLogger` methods, using the `_emit()` wrapper Day 18 introduced (the one that writes the audit event *and* feeds the abuse detector):

```python
# backend/audit/audit_logger.py (additions)

class AuditLogger:
    # ... existing methods from Days 14-18 ...

    def secret_access_granted(
        self, agent_id: str, agent_type: str, session_id: str | None, secret_name: str,
    ) -> None:
        """An agent successfully retrieved a secret it's scoped for."""
        _emit(AuditEvent(
            event_type=AuditEventType.SECRET_ACCESS_GRANTED,
            actor_id=agent_id,
            actor_role=agent_type,
            session_id=session_id,
            resource_id=secret_name,
            outcome="granted",
        ))

    def secret_access_denied(
        self, agent_id: str, agent_type: str, session_id: str | None, secret_name: str,
    ) -> None:
        """An agent requested a secret outside its allow-listed scope."""
        _emit(AuditEvent(
            event_type=AuditEventType.SECRET_ACCESS_DENIED,
            actor_id=agent_id,
            actor_role=agent_type,
            session_id=session_id,
            resource_id=secret_name,
            outcome="denied",
            reason=f"Agent type '{agent_type}' is not scoped for '{secret_name}'.",
        ))

    def secret_leak_blocked(
        self, session_id: str | None, sender_id: str, labels: list[str],
    ) -> None:
        """
        A message containing what looks like a live credential was
        blocked before delivery.

        `labels` must be pattern/category names only (e.g.
        ["aws_access_key", "configured_secret"]) - never the matched
        text itself. See the warning at the end of Step 19.4.
        """
        _emit(AuditEvent(
            event_type=AuditEventType.SECRET_LEAK_BLOCKED,
            actor_id=sender_id,
            session_id=session_id,
            outcome="blocked",
            payload={"matched_patterns": sorted(set(labels))},
        ))
```

Because these go through `_emit()`, every one of them is automatically a candidate signal for Day 18's abuse detector - which is exactly what Step 19.7 configures.

<a id="step-196"></a>
## Step 19.6 - Blocking leaks in the message bus

`MessageBus.inspect_message()` (Day 15/16) already runs two checks in order: injection detection (which blocks) and PII detection (which redacts and continues). Secret-leak detection slots in as a third check, and it's deliberately positioned **after injection but before PII**, and it **blocks** rather than redacts.

```python
# backend/orchestrator/message_bus.py (additions)

from backend.security.secret_scanner import find_secrets
from backend.audit.audit_logger import AuditLogger

_audit = AuditLogger()


class MessageBus:
    # ... existing __init__, etc. ...

    def inspect_message(self, sender_id, receiver_id, content, session_id):
        """
        Run content through governance checks before it can be
        delivered. Order matters - each check is more "fixable" than
        the one before it:

          1. Injection detection (Day 15/16) - BLOCKS. A message
             carrying a prompt injection isn't safe to deliver in any
             form.

          2. Secret-leak detection (Day 19) - BLOCKS. A message
             containing a live credential isn't safe to deliver even
             with the credential redacted - see the rationale below.

          3. PII detection (Day 15/16) - REDACTS and continues. SSNs,
             emails, etc. are scrubbed, and the cleaned message is
             delivered.
        """
        # --- 1. Injection check (existing, Day 15/16) ---
        injection_result = self._check_injection(content)
        if not injection_result["passed"]:
            _audit.injection_detected(
                session_id=session_id, sender_id=sender_id,
                pattern=injection_result["pattern"],
            )
            return {
                "passed": False,
                "content": None,
                "violations": injection_result["violations"],
                "was_redacted": False,
            }

        # --- 2. Secret-leak check (new, Day 19) ---
        secret_matches = find_secrets(content)
        if secret_matches:
            labels = sorted({m.label for m in secret_matches})
            _audit.secret_leak_blocked(
                session_id=session_id, sender_id=sender_id, labels=labels,
            )
            return {
                "passed": False,
                "content": None,
                "violations": [f"secret_leak:{label}" for label in labels],
                "was_redacted": False,
            }

        # --- 3. PII check (existing, Day 15/16) ---
        cleaned, was_redacted, pii_violations = self._redact_pii(content)
        if was_redacted:
            _audit.message_redacted(
                session_id=session_id, sender_id=sender_id,
                violations=pii_violations,
            )

        return {
            "passed": True,
            "content": cleaned,
            "violations": pii_violations,
            "was_redacted": was_redacted,
        }
```

**Why block instead of redact-and-continue, like PII?** Two reasons:

- A redacted *copy* doesn't undo exposure that already happened. By the time `inspect_message()` runs, the content exists - if it's a tool result or an LLM completion that the *sending* agent already has in its own context, redacting the copy that goes to the *receiving* agent doesn't un-leak it from the sender's side. Treating it as "scrub and move on" understates what happened.

- A live credential appearing in agent-to-agent traffic is itself the incident, independent of where it ends up. The right response is to stop the message, record the event, and - via Step 19.7 - start counting it toward suspension if it keeps happening. PII redaction assumes the conversation should continue with cleaner content; a secret leak means something upstream (a prompt, a tool, a misconfigured agent) is treating a credential as ordinary data, and that needs attention before the conversation continues at all.

<a id="step-197"></a>
## Step 19.7 - Feeding secret events into abuse detection

Day 18 built `ABUSE_THRESHOLDS`: a list of `(event_type, window, count) -> action` rules, evaluated by `_emit()` every time an audit event is written. Because Step 19.5's new events go through `_emit()`, wiring them into abuse detection is pure configuration - no new detector code.

```python
# backend/security/abuse_schema.py (additions to ABUSE_THRESHOLDS)

ABUSE_THRESHOLDS: list[AbuseThreshold] = [
    # ... existing Day 18 entries for INJECTION_DETECTED, PRIVILEGE_ESC,
    #     PATH_TRAVERSAL, GOAL_BLOCKED, AGENT_SPAWN_BLOCKED,
    #     MESSAGE_BLOCKED, HITL_REJECTED ...

    # Secrets (Day 19)
    #
    # A single denied access can be a harmless coding mistake (an agent
    # built before its scope was widened, say). A handful in a short
    # window looks like probing.
    AbuseThreshold(
        event_type=AuditEventType.SECRET_ACCESS_DENIED,
        window_seconds=1800, count=3, action=AbuseAction.WARN,
        description="3 denied secret access attempts in 30 minutes",
    ),
    AbuseThreshold(
        event_type=AuditEventType.SECRET_ACCESS_DENIED,
        window_seconds=1800, count=5, action=AbuseAction.SUSPEND,
        description="5 denied secret access attempts in 30 minutes",
    ),

    # A blocked secret leak is already a security event by itself - the
    # threshold to WARN is 1. A second one in the same hour suspends the
    # actor; this isn't "three strikes" territory.
    AbuseThreshold(
        event_type=AuditEventType.SECRET_LEAK_BLOCKED,
        window_seconds=3600, count=1, action=AbuseAction.WARN,
        description="1 secret leak blocked in 60 minutes",
    ),
    AbuseThreshold(
        event_type=AuditEventType.SECRET_LEAK_BLOCKED,
        window_seconds=3600, count=2, action=AbuseAction.SUSPEND,
        description="2 secret leaks blocked in 60 minutes",
    ),
]
```

This is the same pattern as Day 18's `INJECTION_DETECTED` and `PATH_TRAVERSAL` thresholds: low counts, short fuses, because these signals are rarely false positives and the cost of a false suspension (an agent gets paused, a human reviews it) is much lower than the cost of letting either pattern continue unchecked.

<a id="step-198"></a>
## Step 19.8 - Agent integration: GovernedAgent.get_secret()

`GovernedAgent` (the base class every agent type extends) gets one new method. It's a thin wrapper over the broker, but it's the *only* way an agent is supposed to obtain a credential - the convention from here on is that `os.environ` never appears in agent code.

```python
# backend/agents/base_agent.py (additions)

from backend.secrets.credential_broker import CredentialBroker, SecretAccessDenied
from backend.secrets.secret_schema import SecretName


class GovernedAgent:
    def __init__(self, agent_id: str, agent_type: str, user_id: str, session_id: str):
        self.agent_id = agent_id
        self.agent_type = agent_type
        self.user_id = user_id
        self.session_id = session_id
        # ... existing init from earlier days ...
        self._broker = CredentialBroker()

    def get_secret(self, secret_name: SecretName) -> str:
        """
        Retrieve a credential this agent is scoped for.

        Raises SecretAccessDenied if `self.agent_type` isn't allow-listed
        for `secret_name` in AGENT_SECRET_SCOPES - this is the runtime
        enforcement of the scopes defined in Step 19.1, and it cannot be
        bypassed by an agent subclass; there is no other path to a
        credential value.
        """
        return self._broker.get_secret(
            secret_name,
            agent_id=self.agent_id,
            agent_type=self.agent_type,
            session_id=self.session_id,
        )
```

The research agent, which Step 19.1 scoped for `OPENAI_API_KEY`, switches its client construction over:

```python
# backend/agents/research_agent.py (excerpt)

from backend.secrets.secret_schema import SecretName


class ResearchAgent(GovernedAgent):

    def _llm_client(self):
        # Before Day 19: api_key = os.environ["OPENAI_API_KEY"]
        api_key = self.get_secret(SecretName.OPENAI_API_KEY)
        return OpenAI(api_key=api_key)

    def _search_client(self):
        api_key = self.get_secret(SecretName.SEARCH_API_KEY)
        return SearchClient(api_key=api_key)
```

The code review agent needs no changes - it never read a credential before, and `AGENT_SECRET_SCOPES[AgentType.CODE_REVIEW.value]` is an empty list, so if anything ever calls `self.get_secret(...)` from `CodeReviewAgent`, it raises `SecretAccessDenied` immediately. Step 19.10's tests prove this directly.

<a id="step-199"></a>
## Step 19.9 - Admin visibility: the secret-scopes endpoint

Operators need to be able to answer "which agent types can touch which secrets, and is everything configured?" without grepping code or, worse, printing environment variables. This endpoint answers exactly that, and nothing else - it returns scope mappings and configuration *status* (a boolean), never values.

```python
# backend/routers/security.py (additions, alongside Day 18's admin endpoints)

from backend.secrets.secret_schema import SECRET_REGISTRY, AGENT_SECRET_SCOPES
from backend.secrets.vault import is_configured


@router.get("/secret-scopes")
def list_secret_scopes(admin: User = Depends(require_role("admin"))):
    """
    Which agent types can access which secrets, and whether each secret
    is currently configured in this deployment.

    Never returns secret values - `configured` is a boolean, derived
    from vault.is_configured(), which itself never returns a value.
    """
    return {
        "secrets": [
            {
                "name":        name.value,
                "env_var":     definition.env_var,
                "description": definition.description,
                "configured":  is_configured(name),
            }
            for name, definition in SECRET_REGISTRY.items()
        ],
        "scopes": {
            agent_type: [s.value for s in secrets]
            for agent_type, secrets in AGENT_SECRET_SCOPES.items()
        },
    }
```

Like Day 18's `/security/*` endpoints, this sits behind the existing admin role check - scope information ("the research agent can read SEARCH_API_KEY") is not itself a secret, but it's exactly the kind of map an attacker would want before trying to find a way to impersonate that agent type.

<a id="step-1910"></a>
## Step 19.10 - Tests

Four files. The vault and broker tests are pure unit tests; the scanner tests cover both detection layers; the message bus test is the integration test that proves a leaked credential actually gets blocked end-to-end and that repeated leaks trigger Day 18's suspension.

```python
# tests/secrets/test_vault.py
import pytest

from backend.secrets import vault
from backend.secrets.secret_schema import SecretName


@pytest.fixture(autouse=True)
def clear_cache_and_env(monkeypatch):
    vault._cache.clear()
    monkeypatch.delenv("OPENAI_API_KEY", raising=False)
    yield
    vault._cache.clear()


class TestVault:

    def test_missing_secret_raises(self):
        with pytest.raises(vault.SecretNotConfigured):
            vault.get_secret(SecretName.OPENAI_API_KEY)

    def test_configured_secret_returned(self, monkeypatch):
        monkeypatch.setenv("OPENAI_API_KEY", "sk-test1234567890abcdef")
        assert vault.get_secret(SecretName.OPENAI_API_KEY) == "sk-test1234567890abcdef"

    def test_is_configured(self, monkeypatch):
        assert vault.is_configured(SecretName.OPENAI_API_KEY) is False
        monkeypatch.setenv("OPENAI_API_KEY", "sk-test1234567890abcdef")
        assert vault.is_configured(SecretName.OPENAI_API_KEY) is True

    def test_unregistered_secret_raises_value_error(self):
        with pytest.raises(ValueError):
            vault.get_secret("not_a_real_secret")

    def test_all_known_values_includes_configured_only(self, monkeypatch):
        monkeypatch.setenv("OPENAI_API_KEY", "sk-test1234567890abcdef")
        values = vault.all_known_values()
        assert "sk-test1234567890abcdef" in values
        assert len(values) == 1
```

```python
# tests/secrets/test_credential_broker.py
import pytest

from backend.audit.audit_models import AuditLogEntry
from backend.database.db import create_tables, SessionLocal
from backend.secrets import vault
from backend.secrets.credential_broker import CredentialBroker, SecretAccessDenied
from backend.secrets.secret_schema import SecretName


@pytest.fixture(autouse=True)
def fresh_state(monkeypatch):
    create_tables()
    db = SessionLocal()
    db.query(AuditLogEntry).delete()
    db.commit()
    db.close()

    vault._cache.clear()
    monkeypatch.setenv("OPENAI_API_KEY", "sk-test1234567890abcdef")
    monkeypatch.setenv("SEARCH_API_KEY", "search-test-key-abcdef")
    yield
    vault._cache.clear()


class TestCredentialBroker:

    def setup_method(self):
        self.broker = CredentialBroker()

    def test_research_agent_can_get_openai_key(self):
        value = self.broker.get_secret(
            SecretName.OPENAI_API_KEY, agent_id="agent-1",
            agent_type="research", session_id="sess-1",
        )
        assert value == "sk-test1234567890abcdef"

    def test_code_review_agent_denied_openai_key(self):
        with pytest.raises(SecretAccessDenied):
            self.broker.get_secret(
                SecretName.OPENAI_API_KEY, agent_id="agent-2",
                agent_type="code_review", session_id="sess-1",
            )

    def test_denied_access_logged(self):
        try:
            self.broker.get_secret(
                SecretName.OPENAI_API_KEY, agent_id="agent-2",
                agent_type="code_review", session_id="sess-1",
            )
        except SecretAccessDenied:
            pass

        db = SessionLocal()
        try:
            record = (
                db.query(AuditLogEntry)
                .filter(AuditLogEntry.event_type == "secret.access_denied")
                .first()
            )
        finally:
            db.close()

        assert record is not None
        assert record.resource_id == "openai_api_key"

    def test_granted_access_logged(self):
        self.broker.get_secret(
            SecretName.OPENAI_API_KEY, agent_id="agent-1",
            agent_type="research", session_id="sess-1",
        )

        db = SessionLocal()
        try:
            record = (
                db.query(AuditLogEntry)
                .filter(AuditLogEntry.event_type == "secret.access_granted")
                .first()
            )
        finally:
            db.close()

        assert record is not None
        assert record.resource_id == "openai_api_key"

    def test_in_scope_but_unconfigured_raises_not_configured(self, monkeypatch):
        monkeypatch.delenv("SEARCH_API_KEY", raising=False)
        vault._cache.pop(SecretName.SEARCH_API_KEY, None)

        with pytest.raises(vault.SecretNotConfigured):
            self.broker.get_secret(
                SecretName.SEARCH_API_KEY, agent_id="agent-1",
                agent_type="research", session_id="sess-1",
            )
```

```python
# tests/security/test_secret_scanner.py
import pytest

from backend.secrets import vault
from backend.security.secret_scanner import find_secrets, redact_secrets, REDACTION_PLACEHOLDER


@pytest.fixture(autouse=True)
def clear_cache():
    vault._cache.clear()
    yield
    vault._cache.clear()


class TestSecretScanner:

    def test_openai_key_pattern_detected(self):
        text = "Here's my key: sk-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa for testing"
        matches = find_secrets(text)
        assert any(m.label == "openai_api_key" for m in matches)

    def test_aws_key_pattern_detected(self):
        text = "AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE"
        matches = find_secrets(text)
        assert any(m.label == "aws_access_key" for m in matches)

    def test_generic_assignment_detected(self):
        text = 'db_password = "Sup3rSecretValue!"'
        matches = find_secrets(text)
        assert any(m.label == "generic_assignment" for m in matches)

    def test_clean_text_no_matches(self):
        assert find_secrets("The function returns a list of users.") == []

    def test_configured_secret_value_detected_even_without_pattern(self, monkeypatch):
        monkeypatch.setenv("DATABASE_URL", "postgresql://app:hunter2@db.internal:5432/governance")
        text = "Connection failed: postgresql://app:hunter2@db.internal:5432/governance"
        matches = find_secrets(text)
        assert any(m.label == "configured_secret" for m in matches)

    def test_redact_replaces_with_placeholder(self):
        text = "key=sk-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa end"
        redacted, matches = redact_secrets(text)
        assert "sk-aaaa" not in redacted
        assert REDACTION_PLACEHOLDER in redacted
        assert "end" in redacted
        assert len(matches) >= 1

    def test_redact_no_secrets_returns_unchanged(self):
        text = "Nothing sensitive here."
        redacted, matches = redact_secrets(text)
        assert redacted == text
        assert matches == []
```

```python
# tests/security/test_message_bus_secret_leak.py
import pytest

from backend.audit.audit_models import AuditLogEntry
from backend.database.db import create_tables, SessionLocal
from backend.orchestrator.message_bus import MessageBus
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
        for pattern in ("abuse:signal:leak-test*", "abuse:suspended:leak-test*"):
            for key in client.scan_iter(match=pattern):
                client.delete(key)


class TestMessageBusSecretLeak:

    def setup_method(self):
        self.bus = MessageBus()

    def test_message_with_api_key_is_blocked(self):
        content = "Here is the key you asked for: sk-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
        result = self.bus.inspect_message(
            sender_id="leak-test-1", receiver_id="agent-b",
            content=content, session_id="sess-1",
        )
        assert result["passed"] is False
        assert result["content"] is None
        assert any(v.startswith("secret_leak:") for v in result["violations"])

    def test_secret_leak_logged(self):
        self.bus.inspect_message(
            sender_id="leak-test-2", receiver_id="agent-b",
            content="token=ghp_aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
            session_id="sess-1",
        )

        db = SessionLocal()
        try:
            record = (
                db.query(AuditLogEntry)
                .filter(AuditLogEntry.event_type == "secret.leak_blocked")
                .first()
            )
        finally:
            db.close()

        assert record is not None

    def test_two_leaks_suspend_actor(self):
        actor = "leak-test-3"
        for _ in range(2):
            self.bus.inspect_message(
                sender_id=actor, receiver_id="agent-b",
                content="token=ghp_aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
                session_id="sess-1",
            )

        assert suspensions.is_suspended(actor) is not None

    def test_clean_message_passes(self):
        result = self.bus.inspect_message(
            sender_id="leak-test-4", receiver_id="agent-b",
            content="The scan found 3 medium severity issues.",
            session_id="sess-1",
        )
        assert result["passed"] is True
        assert result["content"] is not None
```

Run them:

```bash
pytest tests/secrets/ tests/security/test_secret_scanner.py tests/security/test_message_bus_secret_leak.py -v
```

`test_two_leaks_suspend_actor` is the one worth pausing on: it doesn't call any Day 19 suspension code directly. It calls `inspect_message()` twice, and Day 18's existing abuse-detection pipeline - unmodified since yesterday - reads the `ABUSE_THRESHOLDS` entry added in Step 19.7 and suspends the actor on its own. That's the payoff of building these systems to compose: today's feature needed zero new lines of detection or enforcement logic, only configuration.

<a id="what-you-built"></a>
## What you built today

A secrets vault with exactly one read path (`vault.get_secret()`), backed by environment variables today and swappable later without touching any caller. A `SECRET_REGISTRY` and `AGENT_SECRET_SCOPES` allow-list that says, in one small file, which secrets exist and which agent types may request them - with new agent types starting at zero access. A `CredentialBroker` that enforces that allow-list at runtime and audits every grant and denial. A two-layer secret-leak scanner - known credential formats, plus exact matches against this deployment's own configured secrets - wired into `MessageBus.inspect_message()` as a third check that **blocks** (unlike PII's redact-and-continue) because a live credential leaking between agents is itself the incident. Three new audit event types (`SECRET_ACCESS_GRANTED`, `SECRET_ACCESS_DENIED`, `SECRET_LEAK_BLOCKED`) that flow through Day 18's `_emit()` into the hash-chained audit log. New `ABUSE_THRESHOLDS` entries - pure configuration - that turn repeated denied access attempts or blocked leaks into automatic suspensions via Day 18's existing detector. A `GovernedAgent.get_secret()` method that's now the only way agent code reads a credential, with the research agent updated to use it and the code review agent left unchanged precisely because its scope is empty. An admin endpoint, `GET /security/secret-scopes`, that shows the full scope map and configuration status without ever exposing a value. And a full test suite covering the vault, the broker's grant/deny/audit behavior, both scanner layers, and the end-to-end "leaked credential gets blocked, repeated leaks get suspended" path.

<a id="checklist"></a>
## Day 19 checklist

- [ ] `backend/secrets/secret_schema.py` defines `SecretName`, `SECRET_REGISTRY`, and `AGENT_SECRET_SCOPES`, with `AgentType.CODE_REVIEW` mapped to an explicit empty list
- [ ] `backend/secrets/vault.py` provides `get_secret()`, `is_configured()`, and `all_known_values()`, and is the only module that reads credential environment variables
- [ ] `backend/secrets/credential_broker.py`'s `CredentialBroker.get_secret()` checks `AGENT_SECRET_SCOPES` before calling the vault, and audits both grants and denials
- [ ] `backend/security/secret_scanner.py` detects known credential formats and exact matches against configured secret values, via `find_secrets()` and `redact_secrets()`
- [ ] `AuditEventType` includes `SECRET_ACCESS_GRANTED`, `SECRET_ACCESS_DENIED`, `SECRET_LEAK_BLOCKED`, with corresponding `AuditLogger` methods that never log matched secret values - only pattern labels
- [ ] `MessageBus.inspect_message()` runs the secret-leak check after injection detection and before PII redaction, and **blocks** (does not redact-and-continue) on any match
- [ ] `ABUSE_THRESHOLDS` includes entries for `SECRET_ACCESS_DENIED` (WARN at 3, SUSPEND at 5 per 30 min) and `SECRET_LEAK_BLOCKED` (WARN at 1, SUSPEND at 2 per 60 min)
- [ ] `GovernedAgent.get_secret()` is the only path agent code uses to read a credential; `ResearchAgent` uses it for `OPENAI_API_KEY` and `SEARCH_API_KEY`
- [ ] `GET /security/secret-scopes` (admin-only) returns scope mappings and configuration status, never values
- [ ] All tests in `tests/secrets/` and the new `tests/security/` files pass, including the end-to-end suspension test

---

[Back to repo](https://github.com/GuntruTirupathamma/AI-Governance) - Previous: [DAY18.md](./DAY18.md) - Next: DAY20.md - Cost Governance and Budget Enforcement
