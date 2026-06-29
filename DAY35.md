---
Series: AI Governance Engineering - Zero to Production
Author: GuntruTirupathamma
Day: 35 of 35
Previous: DAY34.md - Production Hardening and Security Configuration
---

# Day 35: Capstone - Full production deployment

## Table of contents

- [The problem with 35 modules and no deployment](#the-problem)
- [What this capstone builds](#what-youll-build)
- [Step 35.1 - Dockerfile (multi-stage, non-root)](#step-351)
- [Step 35.2 - docker-compose for local development](#step-352)
- [Step 35.3 - Environment variable reference (.env.example)](#step-353)
- [Step 35.4 - Kubernetes manifests](#step-354)
- [Step 35.5 - Full end-to-end smoke test](#step-355)
- [Step 35.6 - System architecture and request flow](#step-356)
- [Step 35.7 - Operational runbook](#step-357)
- [Step 35.8 - What the complete system looks like](#step-358)
- [What comes next](#what-comes-next)
- [Day 35 checklist](#checklist)

<a id="the-problem"></a>
## The problem with 35 modules and no deployment

A DevOps engineer joined the team on a Monday and was asked to have the governance stack running in the staging cluster by Friday. The repository had 35 days of Python modules, a test suite, and a README that said "run `uvicorn main:app`." No Dockerfile. No environment variable documentation. No compose file for the three services that the application depends on at runtime.

By Wednesday afternoon, after tracking down that `ALLOWED_ORIGINS` needed to be set or Starlette threw during startup, and that `SECRET_KEY` had to be at least 32 characters or the JWT library rejected it silently, the engineer had the app running locally. On Thursday they discovered the SIEM forwarder was configured against a staging endpoint that had been decommissioned two sprints ago. The Friday smoke test found that the drift scanner was calling a module that had been refactored in Day 31 without updating the import path in the runner. None of these were hard problems. All of them cost time that a proper deployment artifact and a smoke test would have saved.

This is the gap that Day 35 closes. The code is correct. The tests pass. What is missing is the artifact that packages it, the configuration that documents every variable, the Kubernetes manifests that describe the desired runtime state, and the smoke test script that can tell you in 90 seconds whether a fresh deployment is working.

<a id="what-youll-build"></a>
## What this capstone builds

- `Dockerfile` - multi-stage build, non-root user, pinned base image.
- `docker-compose.yml` - local development stack with the governance app and supporting services.
- `.env.example` - every environment variable from all 35 days, with type, default, and description.
- `k8s/deployment.yaml` - Kubernetes Deployment with resource limits, liveness and readiness probes, and a rolling update strategy.
- `k8s/service.yaml` - ClusterIP Service for internal traffic.
- `k8s/configmap.yaml` - non-secret configuration mounted as environment variables.
- `k8s/hpa.yaml` - HorizontalPodAutoscaler targeting 70% CPU.
- `k8s/pdb.yaml` - PodDisruptionBudget ensuring at least one pod stays up during cluster maintenance.
- `scripts/smoke_test.py` - ten-section end-to-end verification script that exits 0 on success.
- `scripts/deploy_check.py` - pre-deploy wrapper that runs the production checklist and the smoke test in sequence.

<a id="step-351"></a>
## Step 35.1 - Dockerfile (multi-stage, non-root)

```dockerfile
# Dockerfile

# --- Build stage ---
# Install dependencies in an isolated layer so the final image does not
# include pip, wheel, or any build tooling.
FROM python:3.12-slim AS builder

WORKDIR /build

COPY requirements.txt .
RUN pip install --upgrade pip \
    && pip install --no-cache-dir --prefix=/install -r requirements.txt


# --- Runtime stage ---
# Copy only the installed packages and application code.
# The final image has no pip, no compiler, and no shell utilities
# beyond what the base image provides.
FROM python:3.12-slim AS runtime

# Create a non-root user.
# Running as root inside a container is unnecessary and expands the blast
# radius of a container escape. This governance API handles audit chains
# and org secrets; it should not run with root privileges.
RUN groupadd --gid 1001 appgroup \
    && useradd --uid 1001 --gid appgroup --no-create-home appuser

WORKDIR /app

# Copy installed packages from the build stage.
COPY --from=builder /install /usr/local

# Copy application code.
COPY backend/ ./backend/
COPY scripts/ ./scripts/

# Switch to non-root user before the final CMD.
USER appuser

# Expose the port uvicorn will bind to.
EXPOSE 8000

# --workers 1 keeps the in-process registries consistent.
# For horizontal scaling, set workers=1 and run multiple replicas
# behind a load balancer rather than multiple workers in one process.
# The in-memory deques are per-process; multiple workers in one container
# would give each worker its own separate registry.
CMD ["uvicorn", "backend.main:app", \
     "--host", "0.0.0.0", \
     "--port", "8000", \
     "--workers", "1", \
     "--no-access-log"]
```

Build and verify:

```bash
docker build -t ai-governance:latest .

# Confirm the process runs as uid 1001, not root
docker run --rm ai-governance:latest id
# Expected: uid=1001(appuser) gid=1001(appgroup)
```

<a id="step-352"></a>
## Step 35.2 - docker-compose for local development

```yaml
# docker-compose.yml

services:
  governance:
    build: .
    image: ai-governance:latest
    ports:
      - "8000:8000"
    env_file:
      - .env
    environment:
      # Override .env for local development defaults
      ENVIRONMENT: development
      LOG_LEVEL: INFO
    healthcheck:
      test: ["CMD", "python", "-c",
             "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 15s
    depends_on:
      redis:
        condition: service_healthy
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  # Optional: run the smoke test against the local stack after startup.
  # docker-compose run --rm smoke-test
  smoke-test:
    build: .
    image: ai-governance:latest
    env_file:
      - .env
    environment:
      BASE_URL: http://governance:8000
      ORG_ID: smoke-test-org
    command: ["python", "scripts/smoke_test.py"]
    depends_on:
      governance:
        condition: service_healthy
    profiles:
      - test
```

Start the local stack:

```bash
# Start governance API and Redis
docker-compose up -d

# Run the smoke test against the local stack
docker-compose run --rm smoke-test

# View structured logs
docker-compose logs -f governance

# Tear down
docker-compose down
```

<a id="step-353"></a>
## Step 35.3 - Environment variable reference

```bash
# .env.example
#
# Copy to .env and fill in values before starting the application.
# Variables marked REQUIRED must be set when ENVIRONMENT=production.
# Variables marked OPTIONAL have working defaults for development.
#
# Generated from the full 35-day series. Each variable is annotated
# with the day it was introduced and whether it is required in production.

# ─── Core ────────────────────────────────────────────────────────────────────

# Runtime environment. Controls auth mode, log format, and doc exposure.
# Values: production | staging | development
# Default: development
# REQUIRED in production
ENVIRONMENT=development

# JWT signing key. Must be at least 32 random bytes in production.
# Generate with: python -c "import secrets; print(secrets.token_hex(32))"
# Introduced: Day 34
# REQUIRED in production
SECRET_KEY=dev-insecure-secret-do-not-use-in-production

# Comma-separated list of allowed CORS origins. No wildcards in production.
# Example: https://app.example.com,https://admin.example.com
# Introduced: Day 34
# REQUIRED in production
ALLOWED_ORIGINS=http://localhost:3000

# Log level. DEBUG produces high output volume; use INFO or WARNING in production.
# Values: DEBUG | INFO | WARNING | ERROR
# Default: INFO
LOG_LEVEL=INFO

# ─── Audit (Day 17) ──────────────────────────────────────────────────────────

# Persistence backend for audit events. Currently informational;
# the in-process registry is used. Set this when migrating to a
# database-backed audit store.
# OPTIONAL
AUDIT_DB_URL=

# ─── Secrets management (Day 19) ─────────────────────────────────────────────

# Encryption key for secrets stored in the secrets vault.
# Generate with: python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
# REQUIRED in production if using the secrets vault
VAULT_ENCRYPTION_KEY=

# ─── Alerting (Day 22) ───────────────────────────────────────────────────────

# Slack webhook URL for governance alert notifications.
# OPTIONAL - alerts are suppressed if not set
SLACK_WEBHOOK_URL=

# Generic webhook for alert forwarding (non-Slack targets)
# OPTIONAL
ALERT_WEBHOOK_URL=

# ─── Testing (Days 17, 27) ───────────────────────────────────────────────────

# Enables test-only endpoints (red team clear, audit reset).
# MUST NOT be set in production. The Day 34 startup check fails the
# application if ENVIRONMENT=production and TESTING=1 are both set.
# TESTING=1

# ─── Onboarding (Day 30) ─────────────────────────────────────────────────────

# Org ID that has platform-admin privileges (cross-org operations).
PLATFORM_ADMIN_ORG_ID=platform-admin

# ─── SIEM (Day 32) ───────────────────────────────────────────────────────────

# SIEM targets are configured per-org via the /siem/config API.
# No static environment variables needed; configs are stored in the
# per-org registry and survive in-process across requests.

# ─── Redis (optional, for distributed rate limiting) ─────────────────────────

# Redis URL for distributed rate limit state. If not set, rate limiting
# uses the in-process deque registry from Day 18 (not shared across replicas).
# OPTIONAL
REDIS_URL=redis://localhost:6379/0
```

<a id="step-354"></a>
## Step 35.4 - Kubernetes manifests

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: governance-config
  namespace: governance
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"
  PLATFORM_ADMIN_ORG_ID: "platform-admin"
```

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: governance-api
  namespace: governance
  labels:
    app: governance-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: governance-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      # At most one pod unavailable during an update.
      # With 2 replicas this means one pod is always serving traffic.
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: governance-api
    spec:
      # Ensure pods are spread across nodes so a single node failure
      # does not take down the entire governance stack.
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: governance-api
      containers:
        - name: governance-api
          image: your-registry/ai-governance:latest
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: governance-config
            - secretRef:
                name: governance-secrets
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 2
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
      securityContext:
        seccompProfile:
          type: RuntimeDefault
```

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: governance-api
  namespace: governance
spec:
  selector:
    app: governance-api
  ports:
    - port: 80
      targetPort: 8000
      protocol: TCP
  type: ClusterIP
```

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: governance-api-hpa
  namespace: governance
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: governance-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: governance-api-pdb
  namespace: governance
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: governance-api
```

Create the Secret separately (never commit secrets to the repository):

```bash
kubectl create secret generic governance-secrets \
  --namespace governance \
  --from-literal=SECRET_KEY="$(python -c 'import secrets; print(secrets.token_hex(32))')" \
  --from-literal=ALLOWED_ORIGINS="https://app.example.com" \
  --from-literal=AUDIT_DB_URL="postgresql://user:pass@db:5432/audit"
```

Apply all manifests:

```bash
kubectl apply -f k8s/
kubectl rollout status deployment/governance-api -n governance
```

<a id="step-355"></a>
## Step 35.5 - Full end-to-end smoke test

```python
# scripts/smoke_test.py
"""
End-to-end smoke test for the AI Governance API.

Calls every major subsystem built over 35 days and reports pass/fail
per section. Exits 0 when all checks pass, 1 when any fail.

Usage:
    # Against local docker-compose stack:
    BASE_URL=http://localhost:8000 python scripts/smoke_test.py

    # Against a production deployment with JWT auth:
    BASE_URL=https://api.example.com \\
    ADMIN_TOKEN=$(python scripts/get_token.py) \\
    python scripts/smoke_test.py
"""

from __future__ import annotations

import os
import sys
import time

import httpx

BASE_URL = os.environ.get("BASE_URL", "http://localhost:8000")
ORG_ID = os.environ.get("ORG_ID", "smoke-test-org")
ADMIN_TOKEN = os.environ.get("ADMIN_TOKEN", "")
ACTOR_ID = "smoke-test-actor"

# JWT mode (production) vs header stub mode (development/test)
if ADMIN_TOKEN:
    AUTH_HEADERS: dict[str, str] = {
        "Authorization": f"Bearer {ADMIN_TOKEN}",
        "Content-Type": "application/json",
    }
else:
    AUTH_HEADERS = {
        "X-Org-Id": ORG_ID,
        "X-Role": "admin",
        "X-Actor-Id": ACTOR_ID,
        "Content-Type": "application/json",
    }

PLATFORM_HEADERS = {
    **AUTH_HEADERS,
    "X-Org-Id": os.environ.get("PLATFORM_ADMIN_ORG_ID", "platform-admin"),
    "X-Role": "platform_admin",
}

_passed: list[str] = []
_failed: list[str] = []


def ok(name: str) -> None:
    _passed.append(name)
    print(f"  [PASS] {name}")


def fail(name: str, detail: str = "") -> None:
    _failed.append(name)
    msg = f"  [FAIL] {name}"
    if detail:
        msg += f"  ({detail})"
    print(msg)


def section(title: str) -> None:
    print(f"\n{'=' * 56}")
    print(f"  {title}")
    print("=" * 56)


def check(name: str, condition: bool, detail: str = "") -> None:
    if condition:
        ok(name)
    else:
        fail(name, detail)


client = httpx.Client(base_url=BASE_URL, timeout=15.0)

# ── Section 1: Health ────────────────────────────────────────────────────────

section("1 / 10  Health and readiness")
r = client.get("/health")
check("GET /health returns 200", r.status_code == 200)
check("health body: status ok", r.json().get("status") == "ok")

r = client.get("/ready")
check("GET /ready returns 200 or 503", r.status_code in (200, 503))
check("ready body: checks dict present", "checks" in r.json())

# ── Section 2: Security headers ──────────────────────────────────────────────

section("2 / 10  Security headers (Day 34)")
r = client.get("/health")
check("HSTS header present", "Strict-Transport-Security" in r.headers)
check("X-Frame-Options is DENY", r.headers.get("X-Frame-Options") == "DENY")
check("X-Content-Type-Options is nosniff",
      r.headers.get("X-Content-Type-Options") == "nosniff")
check("Server header absent", "Server" not in r.headers)

# ── Section 3: Onboarding ────────────────────────────────────────────────────

section("3 / 10  Onboarding validation (Day 30)")
r = client.post(
    "/onboarding/validate",
    headers=PLATFORM_HEADERS,
    json={"org_id": ORG_ID},
)
check("POST /onboarding/validate returns 200", r.status_code == 200)
body = r.json()
check("onboarding report has 'ready' field", "ready" in body)
check("onboarding report has 'checks' list", "checks" in body)

# ── Section 4: Audit trail ───────────────────────────────────────────────────

section("4 / 10  Audit trail (Day 17)")
r = client.get("/audit/events", headers=AUTH_HEADERS)
check("GET /audit/events returns 200", r.status_code == 200)
check("audit response is a list", isinstance(r.json(), list))

# ── Section 5: Policy store ──────────────────────────────────────────────────

section("5 / 10  Policy store (Day 25)")
r = client.put(
    "/policies",
    headers=AUTH_HEADERS,
    json={"table": "smoke_test", "key": "probe_key", "value": "probe_value"},
)
check("PUT /policies upserts successfully", r.status_code in (200, 201))

r = client.get("/policies/smoke_test/probe_key", headers=AUTH_HEADERS)
check("GET /policies/table/key returns 200", r.status_code == 200)

# ── Section 6: Incidents ─────────────────────────────────────────────────────

section("6 / 10  Incident registry (Day 26)")
r = client.get("/incidents", headers=AUTH_HEADERS)
check("GET /incidents returns 200", r.status_code == 200)
check("incidents response is a list", isinstance(r.json(), list))

# ── Section 7: SIEM configuration ───────────────────────────────────────────

section("7 / 10  SIEM configuration (Day 32)")
r = client.get("/siem/config", headers=AUTH_HEADERS)
check("GET /siem/config returns 200", r.status_code == 200)

# ── Section 8: Drift detection ───────────────────────────────────────────────

section("8 / 10  Drift detection (Day 31)")
r = client.post("/drift/scan", headers=AUTH_HEADERS)
check("POST /drift/scan returns 200", r.status_code == 200)
body = r.json()
check("drift scan returns list of records", isinstance(body, list))

# ── Section 9: Governance report ─────────────────────────────────────────────

section("9 / 10  Governance report (Day 33)")
r = client.get("/reports/governance?days=7", headers=AUTH_HEADERS)
check("GET /reports/governance returns 200", r.status_code == 200)
body = r.json()
check("report has health_score field", "health_score" in body)
check("health_score is 0-100 integer", isinstance(body.get("health_score"), int))

r = client.get("/reports/governance?days=7&format=markdown", headers=AUTH_HEADERS)
check("markdown format returns 200", r.status_code == 200)
check(
    "markdown content-type is text/plain",
    "text/plain" in r.headers.get("content-type", ""),
)
check("markdown body contains expected sections",
      "## Security events" in r.text and "## Incidents" in r.text)

# ── Section 10: Error handling ───────────────────────────────────────────────

section("10 / 10  Error sanitization (Day 34)")
r = client.get("/reports/governance/nonexistent-endpoint", headers=AUTH_HEADERS)
check("unknown route returns 404", r.status_code == 404)
body = r.json()
check("404 body has only 'detail' key", set(body.keys()) == {"detail"})

r = client.post(
    "/reports/governance/generate",
    headers=AUTH_HEADERS,
    json={"bad_field": True},
)
check("invalid body returns 422", r.status_code == 422)
body = r.json()
check("422 detail does not expose field path", "bad_field" not in body.get("detail", ""))

# ── Summary ──────────────────────────────────────────────────────────────────

print(f"\n{'=' * 56}")
print(f"  RESULTS: {len(_passed)} passed / {len(_failed)} failed")
print("=" * 56)

if _failed:
    print("\nFailing checks:")
    for name in _failed:
        print(f"  - {name}")
    sys.exit(1)

print("\nAll checks passed. The system is ready for traffic.")
sys.exit(0)
```

```python
# scripts/deploy_check.py
"""
Pre-deploy verification: runs the production security checklist and then
the smoke test against the target environment.

Fails the deploy if either step finds problems.

Usage:
    BASE_URL=https://staging.example.com python scripts/deploy_check.py
"""

from __future__ import annotations

import json
import os
import subprocess
import sys


def run_production_checklist() -> bool:
    print("\n=== Production security checklist ===\n")
    from backend.security.production_checklist import (
        checklist_summary,
        run_production_checklist as _run,
    )

    results = _run()
    summary = checklist_summary(results)
    print(json.dumps(summary, indent=2))
    return summary["all_passed"]


def run_smoke_test() -> bool:
    print("\n=== Smoke test ===\n")
    result = subprocess.run(
        [sys.executable, "scripts/smoke_test.py"],
        env=os.environ.copy(),
    )
    return result.returncode == 0


def main() -> None:
    checklist_ok = run_production_checklist()
    smoke_ok = run_smoke_test()

    print("\n=== Deploy check summary ===")
    print(f"  Security checklist : {'PASS' if checklist_ok else 'FAIL'}")
    print(f"  Smoke test         : {'PASS' if smoke_ok else 'FAIL'}")

    if checklist_ok and smoke_ok:
        print("\nDeploy check passed. Safe to switch traffic.\n")
        sys.exit(0)
    else:
        print("\nDeploy check failed. Do not switch traffic.\n")
        sys.exit(1)


if __name__ == "__main__":
    main()
```

<a id="step-356"></a>
## Step 35.6 - System architecture and request flow

The 35 modules built in this series group into four layers. Here is how they connect, and what happens when a caller hits the `POST /stream` endpoint.

```
                    Incoming request
                          |
                          v
            +-----------------------------+
            |    Day 34: Security layer   |
            |  RequestSizeMiddleware      |
            |  SecurityHeadersMiddleware  |
            |  CORSMiddleware             |
            |  JWT / get_tenant_context   |
            +-----------------------------+
                          |
                          v
            +-----------------------------+
            |    Day 30: Onboarding gate  |
            |  require_onboarded()        |
            |  (403 if not onboarded)     |
            +-----------------------------+
                          |
                          v
            +-----------------------------+
            |    Day 21: Tenant context   |
            |  org_id scoping             |
            |  role enforcement           |
            +-----------------------------+
                          |
               +----------+----------+
               |          |          |
               v          v          v
          Day 18      Day 19      Day 20
       Rate limit    Secrets     Budget
       Abuse det.    manager     tracking
               |          |          |
               +----------+----------+
                          |
                          v
            +-----------------------------+
            |    Day 25: Policy engine    |
            |  get_policy_dynamic()       |
            |  evaluate injection rules   |
            +-----------------------------+
                          |
                          v
            +-----------------------------+
            |    Day 17: Audit logger     |
            |  emit_audit_event()         |
            |  chain hash computation     |
            +-----------------------------+
                          |
               +----------+----------+
               |                     |
               v                     v
         Day 22: Alerts        Day 32: SIEM
         Slack / webhook       Splunk / Datadog /
         dedup, TTL            webhook forward
               |
               v
         Day 26: Incident runner
         Playbook evaluation
               |
         Day 28: Escalation
         Human-in-the-loop
               |
         Day 29: SLA / Postmortem
         Day 31: Drift detection
               |
               v
         Day 33: Governance report
         Day 27: Red team (scheduled)
         Day 30: Onboarding check (scheduled)
```

Every audit event flows through this path. The SIEM forward and the alert notification happen as fire-and-forget background tasks so they never block the response to the caller. The incident runner, SLA tracker, and drift scanner run either on alert trigger or on a scheduled GitHub Actions cron.

<a id="step-357"></a>
## Step 35.7 - Operational runbook

### Rotating the JWT secret

JWT tokens signed with the old key become invalid immediately when the key rotates. Coordinate with callers before rotating in production.

```bash
# Generate a new key
NEW_KEY=$(python -c "import secrets; print(secrets.token_hex(32))")

# Update the Kubernetes secret (triggers a pod restart via the deployment)
kubectl patch secret governance-secrets \
  --namespace governance \
  --type=merge \
  --patch="{\"stringData\":{\"SECRET_KEY\":\"$NEW_KEY\"}}"

# Restart pods to pick up the new key
kubectl rollout restart deployment/governance-api -n governance
kubectl rollout status deployment/governance-api -n governance
```

### Recovering from a critical drift event

When `POST /drift/scan` returns a CRITICAL record, onboarding has been revoked for the affected org. The org cannot use the `/stream` endpoint until re-onboarded.

```bash
# Identify the affected org from the drift record
curl -s -H "X-Role: admin" -H "X-Org-Id: YOUR_ORG" \
  http://localhost:8000/drift | python -m json.tool

# Re-run onboarding once the underlying issue is resolved
curl -s -X POST \
  -H "X-Role: platform_admin" -H "X-Org-Id: platform-admin" \
  -H "Content-Type: application/json" \
  -d '{"org_id": "YOUR_ORG"}' \
  http://localhost:8000/onboarding/validate | python -m json.tool
```

### Viewing a governance report after an incident

```bash
# JSON report for the last 30 days
curl -s -H "X-Role: admin" -H "X-Org-Id: YOUR_ORG" \
  "http://localhost:8000/reports/governance?days=30" | python -m json.tool

# Markdown report for the compliance team
curl -s -H "X-Role: admin" -H "X-Org-Id: YOUR_ORG" \
  "http://localhost:8000/reports/governance?days=30&format=markdown"
```

### Running the red team suite on demand

```bash
# Requires TESTING=1 in a non-production environment
curl -s -X POST \
  -H "X-Role: admin" -H "X-Org-Id: YOUR_ORG" \
  http://localhost:8000/redteam/run | python -m json.tool
```

### Clearing the report cache after a drift scan

```bash
curl -s -X DELETE \
  -H "X-Role: admin" -H "X-Org-Id: YOUR_ORG" \
  http://localhost:8000/reports/governance/cache
```

### Checking SLA performance for a postmortem

```bash
curl -s -H "X-Role: admin" -H "X-Org-Id: YOUR_ORG" \
  http://localhost:8000/postmortems/sla/summary | python -m json.tool
```

<a id="step-358"></a>
## Step 35.8 - What the complete system looks like

Run the full test suite one final time to confirm everything passes:

```bash
TESTING=1 pytest tests/ -v --tb=short
```

Start the local stack and run the smoke test:

```bash
docker-compose up -d
sleep 15   # wait for health checks to pass
docker-compose run --rm smoke-test
```

Run the deploy check against staging:

```bash
BASE_URL=https://staging.example.com \
ENVIRONMENT=production \
SECRET_KEY=your-staging-secret \
ALLOWED_ORIGINS=https://staging.example.com \
AUDIT_DB_URL=postgresql://user:pass@staging-db/audit \
python scripts/deploy_check.py
```

If `deploy_check.py` exits 0, switch traffic.

The GitHub Actions schedule that runs automatically after this:

```
drift_scan.yml     - every 15 minutes, scans all onboarded orgs
red_team.yml       - daily at 03:00 UTC, runs the red team suite
production_checklist.yml - on every push to main
```

<a id="what-comes-next"></a>
## What comes next

The system built in this series is production-ready for a single-region deployment with in-process state. These are the natural next extensions once it is running under real load.

**Persistent audit storage.** The in-process `deque` registries reset on deploy. For a system that needs to answer "show me all events from 90 days ago," the audit log needs a database backend. PostgreSQL with a `governance_audit_events` table and a chain hash column is the minimal change. The `emit_audit_event` function signature stays the same; only the storage layer changes.

**Distributed rate limiting.** Day 18's abuse detection uses per-process counters. With multiple replicas, each pod has its own view of request rates. Redis with an atomic increment and TTL gives you a shared counter across all replicas without introducing a synchronous database call on every request.

**Multi-region audit replication.** A tamper-proof chain that lives in one region is a single point of failure for the audit record. Streaming audit events to a secondary region via Kafka or a managed change data capture tool gives you a verified copy that can be used for recovery if the primary is compromised.

**OpenTelemetry tracing.** Adding `opentelemetry-instrumentation-fastapi` to the app gives you distributed traces from the request entry point through policy evaluation, audit emission, and SIEM forwarding. This makes it possible to profile exactly where latency is being introduced and to correlate a governance event in the SIEM with the original HTTP request.

**Formal policy language.** Day 25's policy store uses key-value pairs with Python-evaluated conditions. For a larger organization, a formal policy language (OPA/Rego or Cedar) gives policy authors a way to express complex conditions without touching Python code. The `evaluate_policy` function becomes a call to the OPA REST API instead of a local eval.

<a id="checklist"></a>
## Day 35 checklist

- [ ] `Dockerfile` builds without error; container runs as uid 1001
- [ ] `docker-compose up -d` starts the stack cleanly
- [ ] `docker-compose run --rm smoke-test` exits 0
- [ ] `.env.example` documents every environment variable from all 35 days
- [ ] `k8s/` directory contains all five manifests
- [ ] `kubectl apply -f k8s/` succeeds against a test cluster
- [ ] `scripts/smoke_test.py` passes all 10 sections locally
- [ ] `scripts/deploy_check.py` exits 0 when all checks pass
- [ ] `TESTING=1 pytest tests/ -v` is green across all 35 days of tests
- [ ] GitHub Actions: `drift_scan.yml`, `red_team.yml`, `production_checklist.yml` all present

---

## Series complete

You built a production AI governance system from scratch in 35 days.

Day 1 started with the problem of an AI API with no audit trail, no tenant isolation, and no way to answer "what did this system do last Tuesday." Day 35 ends with a deployable, testable, documented system that can answer a GDPR data subject access request, detect governance drift in 15 minutes, escalate a budget freeze to a human reviewer before it becomes a customer incident, and forward tamper-proof audit events to a SIEM in real time.

The architecture decisions made along the way were chosen for clarity over cleverness: in-memory registries with `deque` and `defaultdict` because they are simple to understand and easy to replace; fire-and-forget async tasks because blocking the audit write on a SIEM network call is never acceptable; frozen dataclasses for immutable records because mutating an audit entry after the fact is the first step toward a tampered chain.

The next engineer who reads this code should be able to understand every decision from the comments and the test names. That is what production-ready actually means.

---

Previous: [DAY34.md - Production Hardening and Security Configuration](DAY34.md)
