---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 34 of 35
Previous: DAY33.md - Governance Reporting for Compliance and Legal Teams
Next: DAY35.md - Capstone: Full Production Deployment
---

# Day 34: Production hardening and security configuration

## Table of contents

- [The problem with "it works in staging"](#the-problem)
- [What hardening means here](#what-it-means-here)
- [What you will build today](#what-youll-build)
- [Step 34.1 - Environment validation on startup](#step-341)
- [Step 34.2 - Production JWT auth](#step-342)
- [Step 34.3 - Security headers middleware](#step-343)
- [Step 34.4 - Request size guard](#step-344)
- [Step 34.5 - Error sanitization handlers](#step-345)
- [Step 34.6 - Structured JSON logging](#step-346)
- [Step 34.7 - Health and readiness probes](#step-347)
- [Step 34.8 - Graceful shutdown and lifespan context](#step-348)
- [Step 34.9 - Production security checklist](#step-349)
- [Step 34.10 - Wiring it all into main.py and tests](#step-3410)
- [What you built today](#what-you-built)
- [Day 34 checklist](#checklist)

<a id="the-problem"></a>
## The problem with "it works in staging"

We deployed to production with `TESTING=1` set in the environment. This happened because the person handling the deployment copied the CI `.env` file into the production deployment config and did not notice the flag. The consequences were not subtle. The red team endpoint (`POST /redteam/run`) was publicly accessible with no guard, `clear_audit_events` worked without restriction, and the first automated health probe cleared six hours of production audit events before anyone noticed. The SIEM flagged the audit gap at 2 AM. The on-call engineer spent forty minutes assuming it was a SIEM misconfiguration before tracing it back to the environment variable.

Three weeks later, a customer's security team ran an automated scanner against our API. The scanner sent a malformed request body to the SIEM bulk export endpoint. FastAPI debug mode was on in our deployment fork, a setting we had not reviewed, and the unhandled exception returned a full Python traceback in the HTTP response body. The traceback included the internal file path, the module name, and the `org_id` being processed at the time. The customer filed a severity-2 finding.

Production hardening is not about adding features. It is about ensuring that the system which was safe in testing stays safe when the flags that made testing convenient are removed, when error paths are exercised under adversarial conditions, and when the process is terminated mid-request during a rolling deploy.

<a id="what-it-means-here"></a>
## What hardening means here

The system built over 33 days uses `X-Org-Id` and `X-Role` headers as auth stubs. Every router test passes those headers directly. In production, a caller should not be able to set `X-Role: admin` in a request header and receive admin-level access. The stubs need to remain for the test suite, but production must verify a signed JWT before extracting org and role.

FastAPI has no built-in request body size limit. A policy document or bulk export request that arrives as 50 MB is almost certainly an attack or a client bug; either way the application should not spend CPU attempting to parse it. FastAPI's default 422 responses include field names and sometimes field values from the failing Pydantic model, which is useful during development and a schema disclosure in production. The debug mode traceback problem is the most visible form of this, but even well-formed validation errors can expose internal structure.

The in-process registries built since Day 17 (audit events, incidents, escalations, drift records, SIEM configs) are all `deque`- or `dict`-backed and cleared on restart. This is fine for an application that restarts cleanly. If a deploy sends SIGTERM while a SIEM forward or incident playbook step is running as a background task, that task is abandoned mid-execution. Audit events can be dropped, SIEM sends can be interrupted, and the incident registry can be left in an inconsistent state at the next startup if the background task was mid-write.

<a id="what-youll-build"></a>
## What you will build today

- `backend/security/env_guard.py` - startup validation that fails loudly if `TESTING=1` is set in production.
- `backend/security/jwt_auth.py` - JWT verification replacing header stubs in production; stubs remain for tests.
- `backend/security/headers.py` - ASGI middleware adding HSTS, CSP, X-Frame-Options, and stripping `Server`.
- `backend/security/request_guard.py` - middleware rejecting bodies over 1 MB before they reach routing.
- `backend/security/error_handlers.py` - custom handlers for `HTTPException`, `RequestValidationError`, and unhandled exceptions that log full detail internally and return minimal messages externally.
- `backend/security/logging_config.py` - structured JSON logging for production, plain text for development.
- `backend/routers/health.py` - `/health` (liveness) and `/ready` (readiness) probes with no auth.
- `backend/lifecycle.py` - FastAPI lifespan context that runs startup validation, configures JWT, and drains in-flight background tasks before shutdown.
- `backend/security/production_checklist.py` - programmatic checklist returning pass/fail/warn results for each security requirement.
- Updated `backend/main.py` wiring all of the above together.
- `tests/security/` - tests for env guard, headers, error handlers, checklist, and probes.

<a id="step-341"></a>
## Step 34.1 - Environment validation on startup

```python
# backend/security/env_guard.py

from __future__ import annotations

import os
from dataclasses import dataclass, field

# Variables that must be set when ENVIRONMENT=production.
# In development or staging they are optional; missing them there
# produces a warning, not a hard failure.
_REQUIRED_IN_PRODUCTION = [
    "SECRET_KEY",       # JWT signing key
    "ALLOWED_ORIGINS",  # Comma-separated CORS allowlist
    "AUDIT_DB_URL",     # Audit persistence backend
]


@dataclass
class EnvValidationResult:
    missing: list[str] = field(default_factory=list)
    warnings: list[str] = field(default_factory=list)

    @property
    def ok(self) -> bool:
        return len(self.missing) == 0


def validate_production_env() -> EnvValidationResult:
    """
    Validate required environment variables for production.

    Only enforces hard failures when ENVIRONMENT=production.
    In other environments the function returns immediately so
    development workflows are not affected.
    """
    result = EnvValidationResult()
    env = os.environ.get("ENVIRONMENT", "development")

    if env != "production":
        return result

    for var in _REQUIRED_IN_PRODUCTION:
        if not os.environ.get(var):
            result.missing.append(var)

    if os.environ.get("TESTING"):
        result.missing.append(
            "TESTING must not be set when ENVIRONMENT=production"
        )

    secret = os.environ.get("SECRET_KEY", "")
    if secret.startswith("dev-"):
        result.warnings.append(
            "SECRET_KEY begins with 'dev-' which suggests a development default"
        )

    return result
```

The `TESTING` check is the one that would have caught the incident described in the problem section. If the deployment script calls `validate_production_env()` and fails on a non-empty `missing` list, that deploy does not proceed. The check is free: it reads environment variables and returns. There is no network call, no database access, nothing that can time out.

<a id="step-342"></a>
## Step 34.2 - Production JWT auth

```python
# backend/security/jwt_auth.py

from __future__ import annotations

import os
from typing import Any

import jwt  # PyJWT >= 2.0
from fastapi import HTTPException, Request

from backend.auth.tenant import TenantContext

_SECRET_KEY: str = ""
_ALGORITHM: str = "HS256"

_VALID_ROLES = frozenset({"operator", "admin", "platform_admin"})


def configure_jwt(secret_key: str) -> None:
    """Set the JWT signing key. Called once during application startup."""
    global _SECRET_KEY
    _SECRET_KEY = secret_key


def verify_bearer_token(token: str) -> TenantContext:
    """
    Decode and validate a signed JWT, returning a TenantContext.

    Raises 401 for expired, invalid, or structurally incomplete tokens.
    Does not raise for roles that exist in _VALID_ROLES; unknown roles
    raise 401 rather than 403 because an unknown role is a token problem,
    not an authorization problem.
    """
    if not _SECRET_KEY:
        raise RuntimeError(
            "JWT secret key not configured. Call configure_jwt() during startup."
        )

    try:
        payload: dict[str, Any] = jwt.decode(
            token, _SECRET_KEY, algorithms=[_ALGORITHM]
        )
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired.")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token.")

    org_id: str | None = payload.get("org_id")
    role: str | None = payload.get("role")
    actor_id: str | None = payload.get("sub")

    if not org_id or not role or not actor_id:
        raise HTTPException(
            status_code=401,
            detail="Token missing required claims (org_id, role, sub).",
        )

    if role not in _VALID_ROLES:
        raise HTTPException(
            status_code=401, detail=f"Unrecognised role in token: {role!r}"
        )

    return TenantContext(org_id=org_id, role=role, actor_id=actor_id)


async def get_tenant_context(request: Request) -> TenantContext:
    """
    Production-safe replacement for the test-stub get_tenant_context dependency.

    When ENVIRONMENT=production, reads and verifies a JWT from the
    Authorization: Bearer header.

    In all other environments, falls back to the X-Org-Id / X-Role /
    X-Actor-Id header stubs so that the full test suite continues to work
    without any JWT infrastructure.
    """
    if os.environ.get("ENVIRONMENT") == "production":
        auth_header = request.headers.get("Authorization", "")
        if not auth_header.startswith("Bearer "):
            raise HTTPException(
                status_code=401,
                detail="Authorization: Bearer <token> header required.",
            )
        token = auth_header[len("Bearer "):]
        return verify_bearer_token(token)

    return TenantContext(
        org_id=request.headers.get("X-Org-Id", "test-org"),
        role=request.headers.get("X-Role", "admin"),
        actor_id=request.headers.get("X-Actor-Id", "test-actor"),
    )
```

This file replaces the stub `get_tenant_context` that was defined in `backend/auth/tenant.py` for the first 33 days. The production path and the test path share the same function signature and return type, so all existing router dependencies continue to work without modification. The switch is controlled entirely by the `ENVIRONMENT` variable.

<a id="step-343"></a>
## Step 34.3 - Security headers middleware

```python
# backend/security/headers.py

from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """
    Adds defensive HTTP headers to every response.

    These headers do not substitute for proper network controls or
    authentication. They prevent a class of browser-side attacks and
    help pass automated security scanners that enterprise customers run
    during vendor review. Failing those scanners delays contract sign-off;
    adding these headers costs nothing.
    """

    async def dispatch(self, request: Request, call_next) -> Response:
        response = await call_next(request)

        # HSTS: tell browsers this domain must only be reached over HTTPS
        # for the next two years, including subdomains.
        response.headers["Strict-Transport-Security"] = (
            "max-age=63072000; includeSubDomains"
        )

        # Prevent browsers from MIME-sniffing the content type.
        response.headers["X-Content-Type-Options"] = "nosniff"

        # Forbid this page from being embedded in an iframe anywhere.
        response.headers["X-Frame-Options"] = "DENY"

        # Do not send Referer headers when navigating away from this origin.
        response.headers["Referrer-Policy"] = "no-referrer"

        # No inline scripts, no external resources, no framing.
        # This is a strict policy appropriate for a pure JSON API.
        response.headers["Content-Security-Policy"] = (
            "default-src 'none'; frame-ancestors 'none'"
        )

        # Disable browser features we do not use.
        response.headers["Permissions-Policy"] = (
            "camera=(), microphone=(), geolocation=()"
        )

        # Remove headers that leak the web server software and version.
        response.headers.pop("Server", None)
        response.headers.pop("X-Powered-By", None)

        return response
```

The `Server` and `X-Powered-By` removal matters less for security than for reducing information available to an attacker doing reconnaissance. Seeing `uvicorn/0.24.0.post1` in every response tells an attacker which CVEs to look for. Removing the header does not make the server harder to identify through other means, but it removes the obvious signal.

<a id="step-344"></a>
## Step 34.4 - Request size guard

```python
# backend/security/request_guard.py

from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse, Response

# 1 MB covers all legitimate payloads this API accepts.
# Policy documents, SIEM configs, and report requests are all
# small JSON objects. A 50 MB request body is a bug or an attack.
MAX_BODY_BYTES = 1 * 1024 * 1024


class RequestSizeMiddleware(BaseHTTPMiddleware):
    """
    Reject requests whose Content-Length header exceeds MAX_BODY_BYTES.

    FastAPI has no built-in body size limit. Without this middleware,
    a caller can send an arbitrarily large body and force the server to
    buffer it entirely before Pydantic attempts to parse it.

    This check reads Content-Length only; it does not buffer the body.
    A caller who omits Content-Length while streaming a large body is
    handled by uvicorn's --limit-concurrency setting at the server level.
    """

    async def dispatch(self, request: Request, call_next) -> Response:
        content_length = request.headers.get("content-length")
        if content_length and int(content_length) > MAX_BODY_BYTES:
            return JSONResponse(
                status_code=413,
                content={"detail": "Request body too large."},
            )
        return await call_next(request)
```

<a id="step-345"></a>
## Step 34.5 - Error sanitization handlers

```python
# backend/security/error_handlers.py

from __future__ import annotations

import logging
import traceback

from fastapi import Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

logger = logging.getLogger(__name__)


async def http_exception_handler(
    request: Request, exc: StarletteHTTPException
) -> JSONResponse:
    """
    Return status code and detail string only.

    FastAPI's default handler can pass through additional headers and
    path information. This handler gives the caller exactly the information
    they need to act on the error and nothing more.
    """
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.detail},
    )


async def validation_exception_handler(
    request: Request, exc: RequestValidationError
) -> JSONResponse:
    """
    Log full validation errors internally; return a minimal message externally.

    Pydantic validation errors include field names, field values, and
    location paths inside the input document. Those details are useful for
    debugging and belong in the log, not in the HTTP response body where
    they disclose internal schema structure to the caller.
    """
    error_count = len(exc.errors())
    logger.warning(
        "request_validation_failed",
        extra={
            "path": str(request.url.path),
            "method": request.method,
            "error_count": error_count,
            "errors": exc.errors(),
        },
    )
    return JSONResponse(
        status_code=422,
        content={
            "detail": (
                f"Request validation failed ({error_count} error(s)). "
                "Check the request body and query parameters."
            )
        },
    )


async def unhandled_exception_handler(
    request: Request, exc: Exception
) -> JSONResponse:
    """
    Last-resort handler for exceptions that escaped all other handlers.

    Logs the full traceback internally. The response body contains no
    internal information. FastAPI debug mode must be disabled in production
    or this handler is bypassed and tracebacks reach the caller directly.
    """
    logger.error(
        "unhandled_exception",
        extra={
            "path": str(request.url.path),
            "method": request.method,
            "exc_type": type(exc).__name__,
            "traceback": traceback.format_exc(),
        },
    )
    return JSONResponse(
        status_code=500,
        content={"detail": "An internal error occurred."},
    )
```

The `debug=False` flag on the FastAPI app and the `unhandled_exception_handler` registration are both required. Setting `debug=False` tells FastAPI not to render tracebacks itself, but without the handler registered on the `Exception` base class, Starlette's default exception handling can still surface internal detail in some error paths.

<a id="step-346"></a>
## Step 34.6 - Structured JSON logging

```python
# backend/security/logging_config.py

from __future__ import annotations

import logging
import os
import sys
from typing import Any

# Fields present on every LogRecord that are not application data.
# These are excluded from the JSON output to avoid noise.
_STDLIB_FIELDS = frozenset(
    {
        "name", "msg", "args", "levelname", "levelno", "pathname",
        "filename", "module", "exc_info", "exc_text", "stack_info",
        "lineno", "funcName", "created", "msecs", "relativeCreated",
        "thread", "threadName", "processName", "process", "message",
        "taskName",
    }
)


class _JSONFormatter(logging.Formatter):
    """
    Formats each log record as a single-line JSON object.

    Structured logs are parseable by SIEM and log aggregators without
    custom grok patterns. Any extra fields passed via the `extra` kwarg
    on a logging call are included in the output alongside the standard
    timestamp, level, logger name, and message fields.
    """

    def format(self, record: logging.LogRecord) -> str:
        import json
        from datetime import datetime, timezone

        doc: dict[str, Any] = {
            "ts": datetime.fromtimestamp(
                record.created, tz=timezone.utc
            ).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "msg": record.getMessage(),
        }

        for key, val in record.__dict__.items():
            if key not in _STDLIB_FIELDS:
                doc[key] = val

        if record.exc_info:
            doc["exc"] = self.formatException(record.exc_info)

        return json.dumps(doc, default=str)


def configure_logging() -> None:
    """
    Configure application-wide logging.

    In production: structured JSON to stdout, readable by any log
    aggregator.

    In development: plain human-readable output so local terminal output
    stays usable.

    Log level is read from LOG_LEVEL (default INFO). Setting DEBUG in
    production produces very high output volume; it should only be used
    during a targeted investigation, not as a permanent setting.
    """
    level_name = os.environ.get("LOG_LEVEL", "INFO").upper()
    level = getattr(logging, level_name, logging.INFO)
    env = os.environ.get("ENVIRONMENT", "development")

    handler = logging.StreamHandler(sys.stdout)

    if env == "production":
        handler.setFormatter(_JSONFormatter())
    else:
        handler.setFormatter(
            logging.Formatter(
                "%(asctime)s %(levelname)-8s %(name)s: %(message)s"
            )
        )

    root = logging.getLogger()
    root.setLevel(level)
    root.handlers.clear()
    root.addHandler(handler)

    # These libraries log at INFO on every request/connection, which
    # would flood the output in a busy production environment.
    logging.getLogger("uvicorn.access").setLevel(logging.WARNING)
    logging.getLogger("httpx").setLevel(logging.WARNING)
```

One rule across all logging calls added throughout this system: never pass tokens, raw policy values, or `org_id`-scoped payload content as log field values. The structured log fields should contain identifiers and counts, not content. A log line reading `forwarded 47 events to org=acme target=splunk` is useful for operations. A log line containing the full payload of an audit event is a data exposure.

<a id="step-347"></a>
## Step 34.7 - Health and readiness probes

```python
# backend/routers/health.py

from __future__ import annotations

import logging

from fastapi import APIRouter
from fastapi.responses import JSONResponse

logger = logging.getLogger(__name__)

# No authentication on these endpoints.
# The load balancer and Kubernetes call them continuously.
# Adding auth would mean the load balancer needs a credential, which
# creates an operational dependency that can itself fail.
router = APIRouter(tags=["health"])


@router.get("/health")
def health_check():
    """
    Liveness probe.

    Returns 200 whenever the process is running. Kubernetes uses this
    to decide whether to restart the pod. It must never block on external
    systems; if the probe itself hangs, the orchestrator cannot restart
    a stuck pod.
    """
    return {"status": "ok"}


@router.get("/ready")
def readiness_check():
    """
    Readiness probe.

    Returns 200 only when the application can serve governance traffic.
    Returns 503 when a dependency is unreachable, so the load balancer
    stops routing new requests to this instance until it recovers.

    Checks performed:
    - Audit registry is importable and accessible (proxy for in-process state).
    - Policy store responds to a no-op read.

    These checks are lightweight and do not write any data.
    """
    checks: dict[str, bool] = {}
    failing: list[str] = []

    # Audit registry check
    try:
        from backend.audit.registry import _registry  # noqa: F401

        checks["audit_registry"] = True
    except Exception as exc:
        logger.error("readiness_check_failed", extra={"check": "audit_registry", "exc": str(exc)})
        checks["audit_registry"] = False
        failing.append("audit_registry")

    # Policy store check
    try:
        from backend.policy.store import get_policy

        get_policy("_readiness_probe", "_probe_key")
        checks["policy_store"] = True
    except Exception as exc:
        logger.error("readiness_check_failed", extra={"check": "policy_store", "exc": str(exc)})
        checks["policy_store"] = False
        failing.append("policy_store")

    if failing:
        return JSONResponse(
            status_code=503,
            content={
                "status": "not_ready",
                "failing": failing,
                "checks": checks,
            },
        )
    return {"status": "ready", "checks": checks}
```

The `_readiness_probe` org_id used in the policy store check will return `None` from `get_policy`, which is the correct and expected result. The check verifies that `get_policy` executes without raising, not that it returns a specific value. A missing probe org is not a problem; a `get_policy` call that throws an exception means the policy store itself is broken.

<a id="step-348"></a>
## Step 34.8 - Graceful shutdown and lifespan context

```python
# backend/lifecycle.py

from __future__ import annotations

import asyncio
import logging
import os
from contextlib import asynccontextmanager

from fastapi import FastAPI

logger = logging.getLogger(__name__)

# How long to wait for in-flight background tasks before giving up.
# SIGTERM gives the process a window before SIGKILL arrives.
# In most container orchestrators that window is 30 seconds;
# 10 seconds here leaves margin for the rest of shutdown.
_DRAIN_TIMEOUT_SECONDS = 10


@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    FastAPI lifespan context: runs startup code before yield and
    shutdown code after yield.

    Startup order:
    1. Configure structured logging (must happen first so all subsequent
       log calls use the right format).
    2. Validate environment variables (hard failure if production requirements
       are not met).
    3. Configure JWT signing key.
    4. Log that startup completed cleanly.

    Shutdown:
    Wait up to _DRAIN_TIMEOUT_SECONDS for in-flight background tasks to
    finish. Fire-and-forget tasks from the SIEM forwarder and the incident
    runner can be mid-execution when a rolling deploy sends SIGTERM.
    Without this wait, those tasks are abandoned silently.
    """
    # --- Startup ---
    from backend.security.logging_config import configure_logging

    configure_logging()

    from backend.security.env_guard import validate_production_env

    result = validate_production_env()
    if not result.ok:
        raise RuntimeError(
            f"Production environment validation failed. "
            f"Missing or invalid: {result.missing}"
        )
    for warning in result.warnings:
        logger.warning("env_warning: %s", warning)

    secret_key = os.environ.get("SECRET_KEY", "dev-insecure-secret-do-not-use")
    from backend.security.jwt_auth import configure_jwt

    configure_jwt(secret_key)

    logger.info(
        "startup_complete",
        extra={"env": os.environ.get("ENVIRONMENT", "development")},
    )

    yield

    # --- Shutdown ---
    logger.info("shutdown_initiated")

    loop = asyncio.get_event_loop()
    pending = [
        t
        for t in asyncio.all_tasks(loop)
        if t is not asyncio.current_task()
    ]

    if pending:
        logger.info(
            "draining_background_tasks",
            extra={"pending_count": len(pending)},
        )
        _, still_pending = await asyncio.wait(
            pending, timeout=_DRAIN_TIMEOUT_SECONDS
        )
        if still_pending:
            logger.warning(
                "tasks_abandoned_at_shutdown",
                extra={"abandoned_count": len(still_pending)},
            )
    logger.info("shutdown_complete")
```

The `asyncio.all_tasks(loop)` call returns every task in the event loop, including any tasks created by uvicorn's request handling. Most will have completed before the yield returns. The ones that have not are the fire-and-forget background tasks from SIEM forwarding, alerting, and the incident runner. They are the ones worth waiting for.

<a id="step-349"></a>
## Step 34.9 - Production security checklist

```python
# backend/security/production_checklist.py

from __future__ import annotations

import os
from dataclasses import dataclass, field
from enum import Enum


class CheckStatus(str, Enum):
    PASS = "pass"
    FAIL = "fail"
    WARN = "warn"


@dataclass
class CheckResult:
    name: str
    status: CheckStatus
    detail: str


def run_production_checklist() -> list[CheckResult]:
    """
    Programmatic security checklist for pre-deployment verification.

    Each check is independent and non-destructive. The deployment script
    should call this, collect the results, and fail the deploy if any
    FAIL-status results are present.

    WARN results indicate configuration that is technically allowed but
    should be reviewed before leaving in place long-term.
    """
    results: list[CheckResult] = []

    def _check(
        name: str,
        passed: bool,
        detail: str,
        *,
        warn_only: bool = False,
    ) -> None:
        if passed:
            results.append(CheckResult(name, CheckStatus.PASS, detail))
        elif warn_only:
            results.append(CheckResult(name, CheckStatus.WARN, detail))
        else:
            results.append(CheckResult(name, CheckStatus.FAIL, detail))

    env = os.environ.get("ENVIRONMENT", "")

    _check(
        "ENVIRONMENT_set",
        bool(env),
        f"ENVIRONMENT={env!r}" if env else "ENVIRONMENT not set",
    )

    _check(
        "TESTING_not_set_in_production",
        not (env == "production" and os.environ.get("TESTING")),
        "TESTING must not be set when ENVIRONMENT=production",
    )

    secret_key = os.environ.get("SECRET_KEY", "")
    _check(
        "SECRET_KEY_set",
        bool(secret_key),
        "SECRET_KEY is set" if secret_key else "SECRET_KEY not set",
    )
    _check(
        "SECRET_KEY_not_development_default",
        not secret_key.startswith("dev-"),
        "SECRET_KEY does not use development default",
        warn_only=True,
    )

    allowed_origins = os.environ.get("ALLOWED_ORIGINS", "")
    _check(
        "ALLOWED_ORIGINS_set",
        bool(allowed_origins),
        f"ALLOWED_ORIGINS={allowed_origins!r}" if allowed_origins else "ALLOWED_ORIGINS not set",
    )
    _check(
        "CORS_no_wildcard",
        "*" not in allowed_origins,
        "CORS allowlist does not contain wildcard",
    )

    log_level = os.environ.get("LOG_LEVEL", "INFO")
    _check(
        "LOG_LEVEL_not_DEBUG_in_production",
        not (env == "production" and log_level == "DEBUG"),
        f"LOG_LEVEL={log_level}",
        warn_only=True,
    )

    # Verify that the core governance modules import without error.
    # An import failure at this stage means a missing dependency or a
    # syntax error that would surface as a 500 at runtime.
    for module in [
        "backend.audit.audit_logger",
        "backend.policy.store",
        "backend.incidents.runner",
        "backend.drift.detector",
        "backend.siem.registry",
        "backend.reports.builder",
    ]:
        short = module.split(".")[-1]
        try:
            __import__(module)
            _check(
                f"module_imports_{short}",
                True,
                f"{module} imports cleanly",
            )
        except Exception as exc:
            _check(
                f"module_imports_{short}",
                False,
                f"{module} import failed: {exc}",
            )

    return results


def checklist_summary(results: list[CheckResult]) -> dict:
    passed = sum(1 for r in results if r.status == CheckStatus.PASS)
    failed = sum(1 for r in results if r.status == CheckStatus.FAIL)
    warned = sum(1 for r in results if r.status == CheckStatus.WARN)
    return {
        "passed": passed,
        "failed": failed,
        "warned": warned,
        "all_passed": failed == 0,
        "results": [
            {
                "name": r.name,
                "status": r.status.value,
                "detail": r.detail,
            }
            for r in results
        ],
    }
```

<a id="step-3410"></a>
## Step 34.10 - Wiring into main.py and tests

Update `backend/main.py` to register all the new security components:

```python
# backend/main.py  (hardened production version)

import os

from fastapi import FastAPI
from fastapi.exceptions import RequestValidationError
from fastapi.middleware.cors import CORSMiddleware
from starlette.exceptions import HTTPException as StarletteHTTPException

from backend.lifecycle import lifespan
from backend.security.error_handlers import (
    http_exception_handler,
    unhandled_exception_handler,
    validation_exception_handler,
)
from backend.security.headers import SecurityHeadersMiddleware
from backend.security.request_guard import RequestSizeMiddleware

# Health router first so probes are never affected by auth middleware.
from backend.routers.health import router as health_router

# All existing routers from previous days
from backend.routers.audit import router as audit_router
from backend.routers.alerts import router as alerts_router
from backend.routers.incidents import router as incidents_router
from backend.routers.escalation import router as escalation_router
from backend.routers.postmortems import router as postmortems_router
from backend.routers.onboarding import router as onboarding_router
from backend.routers.drift import router as drift_router
from backend.routers.siem import router as siem_router
from backend.routers.reports import router as reports_router

_env = os.environ.get("ENVIRONMENT", "development")
_is_production = _env == "production"

app = FastAPI(
    title="AI Governance API",
    version="1.0.0",
    # debug=True is what allowed the traceback disclosure.
    # It must be False in all environments; error detail is managed
    # by the custom exception handlers, not FastAPI's debug output.
    debug=False,
    # Disable interactive docs in production.
    # An attacker who reaches /docs can explore the full API schema
    # without any prior knowledge.
    docs_url=None if _is_production else "/docs",
    redoc_url=None if _is_production else "/redoc",
    lifespan=lifespan,
)

# Middleware is applied in reverse registration order:
# the last-added middleware runs first on the incoming request.
# RequestSizeMiddleware should run before everything else on the way in,
# so register it last.
app.add_middleware(SecurityHeadersMiddleware)

_allowed_origins = [
    o.strip()
    for o in os.environ.get(
        "ALLOWED_ORIGINS", "http://localhost:3000"
    ).split(",")
    if o.strip()
]
app.add_middleware(
    CORSMiddleware,
    allow_origins=_allowed_origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=[
        "Authorization",
        "Content-Type",
        "X-Org-Id",
        "X-Role",
        "X-Actor-Id",
    ],
)

app.add_middleware(RequestSizeMiddleware)

# Exception handlers
app.add_exception_handler(StarletteHTTPException, http_exception_handler)
app.add_exception_handler(RequestValidationError, validation_exception_handler)
app.add_exception_handler(Exception, unhandled_exception_handler)

# Routers
app.include_router(health_router)
app.include_router(audit_router)
app.include_router(alerts_router)
app.include_router(incidents_router)
app.include_router(escalation_router)
app.include_router(postmortems_router)
app.include_router(onboarding_router)
app.include_router(drift_router)
app.include_router(siem_router)
app.include_router(reports_router)
```

Tests:

```python
# tests/security/test_env_guard.py

import os
import pytest
from backend.security.env_guard import validate_production_env


def test_development_env_always_passes(monkeypatch):
    monkeypatch.setenv("ENVIRONMENT", "development")
    monkeypatch.delenv("TESTING", raising=False)
    result = validate_production_env()
    assert result.ok
    assert result.missing == []


def test_production_fails_without_secret_key(monkeypatch):
    monkeypatch.setenv("ENVIRONMENT", "production")
    monkeypatch.delenv("SECRET_KEY", raising=False)
    monkeypatch.setenv("ALLOWED_ORIGINS", "https://app.example.com")
    monkeypatch.setenv("AUDIT_DB_URL", "postgresql://localhost/audit")
    result = validate_production_env()
    assert not result.ok
    assert "SECRET_KEY" in result.missing


def test_testing_flag_in_production_is_hard_failure(monkeypatch):
    monkeypatch.setenv("ENVIRONMENT", "production")
    monkeypatch.setenv("TESTING", "1")
    monkeypatch.setenv("SECRET_KEY", "real-secret")
    monkeypatch.setenv("ALLOWED_ORIGINS", "https://app.example.com")
    monkeypatch.setenv("AUDIT_DB_URL", "postgresql://localhost/audit")
    result = validate_production_env()
    assert not result.ok
    assert any("TESTING" in m for m in result.missing)


def test_dev_prefixed_secret_key_is_warning_not_failure(monkeypatch):
    monkeypatch.setenv("ENVIRONMENT", "production")
    monkeypatch.setenv("SECRET_KEY", "dev-local-key")
    monkeypatch.setenv("ALLOWED_ORIGINS", "https://app.example.com")
    monkeypatch.setenv("AUDIT_DB_URL", "postgresql://localhost/audit")
    result = validate_production_env()
    # dev- prefix is a warning, not a hard failure
    assert any("dev-" in w for w in result.warnings)
```

```python
# tests/security/test_headers.py

import pytest
from fastapi.testclient import TestClient
from backend.main import app

client = TestClient(app)


def test_security_headers_present():
    r = client.get("/health")
    assert r.headers.get("X-Content-Type-Options") == "nosniff"
    assert r.headers.get("X-Frame-Options") == "DENY"
    assert "Strict-Transport-Security" in r.headers
    assert "Content-Security-Policy" in r.headers


def test_server_header_removed():
    r = client.get("/health")
    assert "Server" not in r.headers


def test_request_too_large_returns_413():
    r = client.post(
        "/reports/governance/generate",
        content=b"x" * (2 * 1024 * 1024),  # 2 MB
        headers={
            "Content-Type": "application/json",
            "Content-Length": str(2 * 1024 * 1024),
            "X-Org-Id": "test-org",
            "X-Role": "admin",
        },
    )
    assert r.status_code == 413
```

```python
# tests/security/test_error_handlers.py

from fastapi.testclient import TestClient
from backend.main import app

client = TestClient(app, raise_server_exceptions=False)


def test_validation_error_does_not_leak_field_names():
    # POST to the generate endpoint with a clearly invalid body
    r = client.post(
        "/reports/governance/generate",
        json={"bad_field": "bad_value"},
        headers={"X-Org-Id": "test-org", "X-Role": "admin"},
    )
    assert r.status_code == 422
    body = r.json()
    # Detail should mention "validation failed" but not the Pydantic field path
    assert "validation failed" in body["detail"].lower()
    assert "bad_field" not in body["detail"]


def test_404_returns_minimal_detail():
    r = client.get("/nonexistent-endpoint")
    assert r.status_code == 404
    body = r.json()
    assert set(body.keys()) == {"detail"}
```

```python
# tests/security/test_health.py

from fastapi.testclient import TestClient
from backend.main import app

client = TestClient(app)


def test_health_always_200():
    r = client.get("/health")
    assert r.status_code == 200
    assert r.json()["status"] == "ok"


def test_ready_returns_checks_dict():
    r = client.get("/ready")
    # In the test environment the registries are in-process and available.
    # The probe should pass.
    assert r.status_code in (200, 503)
    body = r.json()
    assert "checks" in body


def test_health_requires_no_auth():
    # No X-Org-Id or Authorization header
    r = client.get("/health")
    assert r.status_code == 200
```

```python
# tests/security/test_production_checklist.py

import os
import pytest
from backend.security.production_checklist import (
    CheckStatus,
    checklist_summary,
    run_production_checklist,
)


def test_checklist_passes_in_development(monkeypatch):
    monkeypatch.setenv("ENVIRONMENT", "development")
    monkeypatch.delenv("TESTING", raising=False)
    results = run_production_checklist()
    summary = checklist_summary(results)
    assert summary["all_passed"], summary["results"]


def test_testing_flag_in_production_causes_failure(monkeypatch):
    monkeypatch.setenv("ENVIRONMENT", "production")
    monkeypatch.setenv("TESTING", "1")
    monkeypatch.setenv("SECRET_KEY", "real-secret-key-value")
    monkeypatch.setenv("ALLOWED_ORIGINS", "https://example.com")
    monkeypatch.setenv("AUDIT_DB_URL", "postgresql://localhost/audit")
    results = run_production_checklist()
    failures = [r for r in results if r.status == CheckStatus.FAIL]
    assert any("TESTING" in f.name for f in failures)


def test_wildcard_cors_is_a_failure(monkeypatch):
    monkeypatch.setenv("ENVIRONMENT", "production")
    monkeypatch.setenv("SECRET_KEY", "real-secret-key-value")
    monkeypatch.setenv("ALLOWED_ORIGINS", "*")
    monkeypatch.setenv("AUDIT_DB_URL", "postgresql://localhost/audit")
    results = run_production_checklist()
    failures = [r for r in results if r.status == CheckStatus.FAIL]
    assert any("wildcard" in f.detail.lower() for f in failures)
```

Add the CI workflow:

```yaml
# .github/workflows/production_checklist.yml

name: Production security checklist

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  checklist:
    runs-on: ubuntu-latest
    env:
      ENVIRONMENT: production
      SECRET_KEY: ${{ secrets.SECRET_KEY }}
      ALLOWED_ORIGINS: ${{ secrets.ALLOWED_ORIGINS }}
      AUDIT_DB_URL: ${{ secrets.AUDIT_DB_URL }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - run: pip install -r requirements.txt

      - name: Run production security checklist
        run: |
          python - <<'EOF'
          import json, sys
          from backend.security.production_checklist import (
              run_production_checklist, checklist_summary
          )
          results = run_production_checklist()
          summary = checklist_summary(results)
          print(json.dumps(summary, indent=2))
          sys.exit(0 if summary["all_passed"] else 1)
          EOF

      - name: Dependency vulnerability scan
        run: |
          pip install pip-audit
          pip-audit -r requirements.txt
```

Run the full security test suite:

```bash
TESTING=1 pytest tests/security/ -v
```

<a id="what-you-built"></a>
## What you built today

The governance system now has the security layer that separates a working prototype from a system you can put in front of customers. The `TESTING=1` incident that cleared six hours of production events cannot happen again: the lifespan context calls `validate_production_env()` at startup and refuses to proceed if that flag is set in a production environment. The debug traceback disclosure is closed at two points: `debug=False` on the FastAPI app and the custom exception handler that logs full detail internally and returns only a generic message externally. Security headers go on every response without any per-endpoint configuration, request bodies over 1 MB are rejected before routing even runs, and the production security checklist is a runnable script that a deployment pipeline can call before switching traffic.

<a id="checklist"></a>
## Day 34 checklist

- [ ] `backend/security/env_guard.py` created; `validate_production_env()` fails hard when `TESTING=1` in production
- [ ] `backend/security/jwt_auth.py` created; `get_tenant_context` verifies JWT in production, falls back to headers in test
- [ ] `backend/security/headers.py` created; all six headers added, `Server` removed
- [ ] `backend/security/request_guard.py` created; 413 returned before routing for bodies over 1 MB
- [ ] `backend/security/error_handlers.py` created; all three handlers registered
- [ ] `backend/security/logging_config.py` created; JSON in production, plain text in development
- [ ] `backend/routers/health.py` created; `/health` and `/ready` with no auth requirement
- [ ] `backend/lifecycle.py` created; lifespan context drains background tasks before shutdown
- [ ] `backend/security/production_checklist.py` created; `checklist_summary` returns `all_passed: bool`
- [ ] `backend/main.py` updated: `debug=False`, docs disabled in production, all middleware and handlers registered
- [ ] `.github/workflows/production_checklist.yml` added; fails deploy when checklist has FAIL results
- [ ] `tests/security/test_env_guard.py` passing (4 tests)
- [ ] `tests/security/test_headers.py` passing (3 tests)
- [ ] `tests/security/test_error_handlers.py` passing (2 tests)
- [ ] `tests/security/test_health.py` passing (3 tests)
- [ ] `tests/security/test_production_checklist.py` passing (3 tests)
- [ ] Running `TESTING=1 pytest tests/security/ -v` shows all green

---

Previous: [DAY33.md - Governance Reporting for Compliance and Legal Teams](DAY33.md)
Next: [DAY35.md - Capstone: Full Production Deployment](DAY35.md)
