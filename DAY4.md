# 🛡️ DAY 4 — Redis Rate Limiting + Response Caching

> **Series:** AI Governance Engineering — Zero to Production
> **Author:** GuntruTirupathamma
> **Day:** 4 of 35
> **Previous:** [DAY3.md](./DAY3.md) — Audit Logger + Stats Endpoints
> **Next:** DAY5.md — Unit Tests for Week 1

---

## 📌 Table of Contents

1. [Why Rate Limiting and Caching Are Governance Controls](#why-redis)
2. [What You'll Build Today](#what-youll-build)
3. [The Redis Architecture](#redis-architecture)
4. [Step 4.1 — Install Dependencies](#step-41)
5. [Step 4.2 — Redis Connection Manager](#step-42)
6. [Step 4.3 — Rate Limiting Middleware](#step-43)
7. [Step 4.4 — Role-Based Rate Limits](#step-44)
8. [Step 4.5 — Policy Check Response Caching](#step-45)
9. [Step 4.6 — LLM Response Caching (Cost Governance)](#step-46)
10. [Step 4.7 — Model Registry Cache](#step-47)
11. [Step 4.8 — Cache and Rate Limit Stats Endpoints](#step-48)
12. [Step 4.9 — Update Docker Compose](#step-49)
13. [Step 4.10 — Testing Everything](#step-410)
14. [What You Built Today](#what-you-built)
15. [Day 4 Checklist](#checklist)

---

<a id="why-redis"></a>
## ⚡ Why Rate Limiting and Caching Are Governance Controls

Most engineers think of rate limiting as a performance concern and caching as an optimization. In AI governance, they are **compliance requirements**.

### Rate Limiting as Governance

```
Scenario 1 — The Runaway Script
  A developer writes a buggy agent loop.
  The agent calls the LLM 10,000 times in 3 minutes.
  Cost: $800 in OpenAI bills. Overnight.
  With rate limiting: capped at 100 calls/hour. Caught at call 101.

Scenario 2 — The Disgruntled Insider
  A terminated employee still has API access.
  They run a script to exfiltrate data via LLM prompts.
  400 requests in 5 minutes, all containing customer records.
  Without rate limiting: all 400 go through.
  With rate limiting: blocked at request 11. Alert fires.

Scenario 3 — The Injection Storm
  An attacker finds your API endpoint.
  Runs 50,000 prompt injection variants hoping one leaks data.
  Without rate limiting: bruteforce succeeds eventually.
  With rate limiting: 10 attempts per minute max. Attack fails.

Scenario 4 — The High-Risk Role
  An intern role should have lower LLM access than an admin.
  Without role-based limits: intern can call GPT-4 1000x/day.
  With role-based limits: intern gets 20 calls/day. Enforced.
```

**Rate limiting is not throttling. It is access control with a time dimension.**

### Caching as Governance

```
Scenario 1 — Cost Governance
  100 users ask "What is the company refund policy?"
  Without cache: 100 identical LLM calls. $0.10 × 100 = $10.
  With cache: 1 LLM call, 99 cache hits.  $0.10 × 1  = $0.10.
  Governance lens: you are accountable for AI spend. Cache = audit trail of savings.

Scenario 2 — Consistent Outputs
  Same question, asked twice, gets two different answers.
  Regulator asks: "Why did user A get a different policy explanation than user B?"
  Without cache: non-determinism is baked in.
  With cache: identical inputs → identical outputs. Deterministic. Auditable.

Scenario 3 — Policy Check Caching
  The same prompt goes through the policy engine 500 times a day.
  Without cache: 500 database queries, 500 regex evaluations.
  With cache: 1 evaluation, 499 cache hits. DB load drops 99%.
```

**Every cache hit is a governance decision: we decided this answer is safe for reuse.**

---

<a id="what-youll-build"></a>
## 🎯 What You'll Build Today

```
BEFORE (Day 3):
  No rate limiting — any user can call any endpoint unlimited times
  No caching — every policy check hits PostgreSQL and runs all regex
  No cost controls on LLM calls
  No per-role access differentiation on request volume

AFTER (Day 4):
  ✅ Redis connection manager (async, connection pooling)
  ✅ Sliding window rate limiter middleware for all API endpoints
  ✅ Role-based rate limits (admin > analyst > employee > intern)
  ✅ Per-model rate limits (expensive models get tighter caps)
  ✅ Per-IP rate limits (brute-force protection)
  ✅ Policy check result cache (TTL: 5 minutes)
  ✅ LLM response cache (TTL: 1 hour, keyed by prompt hash)
  ✅ Model registry cache (TTL: 10 minutes)
  ✅ Cache stats endpoint (hit rate, memory usage)
  ✅ Rate limit headers in every response (X-RateLimit-*)
  ✅ Docker Compose updated with Redis service
  ✅ Graceful degradation (if Redis is down, system still works)
```

---

<a id="redis-architecture"></a>
## 🏛️ The Redis Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         REQUEST FLOW WITH REDIS                      │
│                                                                      │
│  Incoming Request                                                    │
│        │                                                             │
│        ▼                                                             │
│  ┌─────────────────────────────────────────┐                        │
│  │     RATE LIMIT MIDDLEWARE               │                        │
│  │                                         │                        │
│  │  Key: rate:{user_id}:{endpoint}         │                        │
│  │  Key: rate:{ip_address}:global          │                        │
│  │  Key: rate:{model_id}:calls             │                        │
│  │                                         │                        │
│  │  Algorithm: Sliding Window Counter      │                        │
│  │  Storage: Redis INCR + EXPIRE           │                        │
│  │                                         │                        │
│  │  429 Too Many Requests ←── OVER LIMIT   │                        │
│  └───────────────┬─────────────────────────┘                        │
│                  │ UNDER LIMIT                                       │
│                  ▼                                                   │
│  ┌─────────────────────────────────────────┐                        │
│  │     CACHE LOOKUP                        │                        │
│  │                                         │                        │
│  │  Key: cache:policy:{prompt_hash}        │ ──→ HIT: return cached │
│  │  Key: cache:llm:{prompt_hash}           │ ──→ HIT: return cached │
│  │  Key: cache:model:{model_id}            │ ──→ HIT: return cached │
│  │                                         │                        │
│  └───────────────┬─────────────────────────┘                        │
│                  │ MISS                                              │
│                  ▼                                                   │
│  ┌─────────────────────────────────────────┐                        │
│  │     ACTUAL PROCESSING                   │                        │
│  │  PostgreSQL query / regex eval / LLM    │                        │
│  └───────────────┬─────────────────────────┘                        │
│                  │                                                   │
│                  ▼                                                   │
│  ┌─────────────────────────────────────────┐                        │
│  │     CACHE WRITE                         │                        │
│  │  Store result in Redis with TTL         │                        │
│  └───────────────┬─────────────────────────┘                        │
│                  │                                                   │
│                  ▼                                                   │
│           Return Response                                            │
│     (with X-RateLimit-* headers)                                     │
└──────────────────────────────────────────────────────────────────────┘

Redis Key Namespace Design:
  rate:user:{user_id}:{window}         → sliding window counter
  rate:ip:{ip_address}:{window}        → IP-level counter
  rate:model:{model_id}:{window}       → per-model counter
  cache:policy:{sha256_hash}           → policy check result
  cache:llm:{sha256_hash}              → LLM response
  cache:model_registry:{model_id}      → model registry entry
  cache:stats:hit_count                → cache hit counter
  cache:stats:miss_count               → cache miss counter
```

### Redis Data Structures Used

```
Rate Limiting → STRING with INCR + EXPIRE
  INCR rate:user:alice:2026-05-20T14     → 1, 2, 3 ... 100
  EXPIRE rate:user:alice:2026-05-20T14 3600
  When count > limit → 429

Response Cache → STRING with JSON + EXPIRE
  SET cache:policy:{hash} '{"allowed":true,...}' EX 300
  GET cache:policy:{hash}  → hit or nil

Blacklist (future) → SET
  SADD blacklist:users blocked-user-123
  SISMEMBER blacklist:users some-user-id

Stats Counters → STRING with INCR
  INCR cache:stats:hits
  INCR cache:stats:misses
```

---

<a id="step-41"></a>
## ⚙️ Step 4.1 — Install Dependencies

```bash
# Activate your virtual environment
source venv/bin/activate  # Windows: venv\Scripts\activate

# Redis async client (the modern choice for FastAPI)
pip install redis==5.0.1

# For type-safe Redis operations
pip install hiredis==2.3.2    # C-level parser, 10x faster than pure Python

# Update requirements
pip freeze > requirements.txt
```

Start Redis with Docker (fastest way, no install needed):

```bash
# Start just Redis from your docker-compose
docker compose up redis -d

# Verify it's running
docker exec -it governance-redis redis-cli ping
# → PONG

# Check Redis version
docker exec -it governance-redis redis-cli INFO server | grep redis_version
# → redis_version:7.x.x
```

---

<a id="step-42"></a>
## ⚙️ Step 4.2 — Redis Connection Manager

One file. One connection pool. Every part of the app uses the same pool.

```python
# backend/cache/redis_client.py
import redis.asyncio as aioredis
from redis.asyncio import Redis
from redis.asyncio.connection import ConnectionPool
import os
import json
import logging
from typing import Optional, Any

logger = logging.getLogger(__name__)

# ─── Configuration ─────────────────────────────────────────────────────────────
REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379")
REDIS_MAX_CONNECTIONS = int(os.getenv("REDIS_MAX_CONNECTIONS", "20"))

# ─── Global Pool (created once on startup) ─────────────────────────────────────
_pool: Optional[ConnectionPool] = None
_redis: Optional[Redis] = None


async def get_redis() -> Optional[Redis]:
    """
    Returns the shared Redis connection.
    Returns None if Redis is unavailable — callers must handle this gracefully.
    Governance systems must never fail because the cache is down.
    """
    global _redis
    if _redis is None:
        try:
            _redis = await init_redis()
        except Exception as e:
            logger.warning(f"Redis unavailable: {e}. Running without cache.")
            return None
    return _redis


async def init_redis() -> Redis:
    """Initialize Redis connection pool on app startup."""
    global _pool, _redis
    _pool = ConnectionPool.from_url(
        REDIS_URL,
        max_connections=REDIS_MAX_CONNECTIONS,
        decode_responses=True,      # Return strings, not bytes
        socket_connect_timeout=5,   # Fail fast if Redis is down
        socket_timeout=3,
        retry_on_timeout=True,
    )
    _redis = Redis(connection_pool=_pool)
    await _redis.ping()             # Verify connection immediately
    logger.info(f"✅ Redis connected: {REDIS_URL}")
    return _redis


async def close_redis():
    """Close Redis connection pool on app shutdown."""
    global _redis, _pool
    if _redis:
        await _redis.aclose()
        _redis = None
    if _pool:
        await _pool.aclose()
        _pool = None
    logger.info("Redis connection closed.")


# ─── Safe Redis Operations ──────────────────────────────────────────────────────
# These wrappers always catch exceptions so Redis failures don't crash the API.

async def redis_get(key: str) -> Optional[str]:
    """Get a value. Returns None on miss or error."""
    try:
        r = await get_redis()
        if not r:
            return None
        return await r.get(key)
    except Exception as e:
        logger.warning(f"Redis GET failed for key={key}: {e}")
        return None


async def redis_set(key: str, value: str, ttl_seconds: int = 300) -> bool:
    """Set a value with TTL. Returns False on error."""
    try:
        r = await get_redis()
        if not r:
            return False
        await r.setex(key, ttl_seconds, value)
        return True
    except Exception as e:
        logger.warning(f"Redis SET failed for key={key}: {e}")
        return False


async def redis_delete(key: str) -> bool:
    """Delete a key. Returns False on error."""
    try:
        r = await get_redis()
        if not r:
            return False
        await r.delete(key)
        return True
    except Exception as e:
        logger.warning(f"Redis DELETE failed for key={key}: {e}")
        return False


async def redis_incr(key: str, ttl_seconds: int = 3600) -> int:
    """
    Atomically increment a counter. Sets TTL on first increment.
    Used for rate limiting sliding windows.
    Returns 0 on error (treat as no rate limit applied).
    """
    try:
        r = await get_redis()
        if not r:
            return 0
        # Pipeline: INCR + EXPIRE in one round-trip
        pipe = r.pipeline()
        pipe.incr(key)
        pipe.expire(key, ttl_seconds)
        results = await pipe.execute()
        return results[0]  # The incremented value
    except Exception as e:
        logger.warning(f"Redis INCR failed for key={key}: {e}")
        return 0


async def redis_ttl(key: str) -> int:
    """Returns seconds until key expires. -1 = no expiry. -2 = key missing."""
    try:
        r = await get_redis()
        if not r:
            return -2
        return await r.ttl(key)
    except Exception:
        return -2


# ─── JSON Helpers ───────────────────────────────────────────────────────────────

async def cache_get_json(key: str) -> Optional[Any]:
    """Get and deserialize a JSON-encoded cached value."""
    raw = await redis_get(key)
    if raw is None:
        return None
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        return None


async def cache_set_json(key: str, value: Any, ttl_seconds: int = 300) -> bool:
    """Serialize and cache a JSON-encodable value."""
    try:
        return await redis_set(key, json.dumps(value, default=str), ttl_seconds)
    except (TypeError, ValueError) as e:
        logger.warning(f"Cache JSON serialization failed: {e}")
        return False
```

---

<a id="step-43"></a>
## ⚙️ Step 4.3 — Rate Limiting Middleware

FastAPI middleware runs on every request before any route handler. The rate limiter lives here — it intercepts all requests and checks Redis before they reach your endpoints.

```python
# backend/middleware/rate_limiter.py
from fastapi import Request, Response
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
from datetime import datetime, timezone
import logging

from backend.cache.redis_client import redis_incr, redis_ttl
from backend.middleware.rate_limit_config import (
    get_rate_limit_for_request,
    BYPASS_PATHS,
    RateLimitConfig,
)

logger = logging.getLogger(__name__)


class RateLimitMiddleware(BaseHTTPMiddleware):
    """
    Sliding window rate limiter.

    Applied to every request. Checks three dimensions:
      1. Per-user limit    (identified by X-User-ID header or 'anonymous')
      2. Per-IP limit      (from request.client.host)
      3. Per-model limit   (from request body if it's a policy check)

    Uses Redis INCR with hourly TTL keys.
    If Redis is down: requests pass through (fail open).
    """

    async def dispatch(self, request: Request, call_next) -> Response:
        # Skip rate limiting for health checks, metrics, docs
        if request.url.path in BYPASS_PATHS:
            return await call_next(request)

        user_id    = request.headers.get("X-User-ID", "anonymous")
        user_role  = request.headers.get("X-User-Role", "employee")
        ip_address = request.client.host if request.client else "unknown"

        # ── Build time window key (hourly sliding window) ─────────
        now     = datetime.now(timezone.utc)
        window  = now.strftime("%Y-%m-%dT%H")   # e.g. "2026-05-20T14"

        # ── Get configured limits for this user's role ─────────────
        config = get_rate_limit_for_request(user_role, request.url.path)

        # ── Check 1: Per-user limit ────────────────────────────────
        user_key   = f"rate:user:{user_id}:{window}"
        user_count = await redis_incr(user_key, ttl_seconds=3600)

        if user_count > config.requests_per_hour:
            retry_after = await redis_ttl(user_key)
            logger.warning(
                f"Rate limit exceeded: user={user_id} "
                f"count={user_count} limit={config.requests_per_hour}"
            )
            return JSONResponse(
                status_code=429,
                content={
                    "error":       "rate_limit_exceeded",
                    "message":     f"Rate limit of {config.requests_per_hour} requests/hour exceeded.",
                    "user_id":     user_id,
                    "limit":       config.requests_per_hour,
                    "window":      "1 hour",
                    "retry_after": retry_after,
                },
                headers={
                    "X-RateLimit-Limit":     str(config.requests_per_hour),
                    "X-RateLimit-Remaining": "0",
                    "X-RateLimit-Reset":     str(retry_after),
                    "Retry-After":           str(retry_after),
                }
            )

        # ── Check 2: Per-IP limit (brute-force protection) ─────────
        ip_key   = f"rate:ip:{ip_address}:{window}"
        ip_count = await redis_incr(ip_key, ttl_seconds=3600)

        if ip_count > config.ip_requests_per_hour:
            retry_after = await redis_ttl(ip_key)
            logger.warning(f"IP rate limit exceeded: ip={ip_address} count={ip_count}")
            return JSONResponse(
                status_code=429,
                content={
                    "error":   "ip_rate_limit_exceeded",
                    "message": "Too many requests from this IP address.",
                    "retry_after": retry_after,
                },
                headers={"Retry-After": str(retry_after)}
            )

        # ── Process request ────────────────────────────────────────
        response = await call_next(request)

        # ── Add rate limit headers to every response ───────────────
        remaining = max(0, config.requests_per_hour - user_count)
        response.headers["X-RateLimit-Limit"]     = str(config.requests_per_hour)
        response.headers["X-RateLimit-Remaining"] = str(remaining)
        response.headers["X-RateLimit-Used"]      = str(user_count)
        response.headers["X-RateLimit-Window"]    = "3600"

        return response
```

---

<a id="step-44"></a>
## ⚙️ Step 4.4 — Role-Based Rate Limits

Different roles get different ceilings. An intern should not have the same LLM access volume as an admin.

```python
# backend/middleware/rate_limit_config.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class RateLimitConfig:
    requests_per_hour:    int     # Total API calls per hour
    ip_requests_per_hour: int     # Max from a single IP per hour
    policy_checks_per_hour: int   # Max policy checks specifically
    llm_calls_per_hour:   int     # Max LLM calls specifically
    description:          str


# ─── Role Definitions ──────────────────────────────────────────────────────────
# Each role has carefully considered limits.
# Design principle: err on the side of restriction, loosen after review.
ROLE_LIMITS: dict[str, RateLimitConfig] = {

    "admin": RateLimitConfig(
        requests_per_hour     = 10_000,
        ip_requests_per_hour  = 50_000,
        policy_checks_per_hour = 5_000,
        llm_calls_per_hour    = 1_000,
        description           = "Full access. Admins manage the system."
    ),

    "analyst": RateLimitConfig(
        requests_per_hour     = 2_000,
        ip_requests_per_hour  = 10_000,
        policy_checks_per_hour = 1_000,
        llm_calls_per_hour    = 500,
        description           = "Power users with broad read/analyze access."
    ),

    "employee": RateLimitConfig(
        requests_per_hour     = 500,
        ip_requests_per_hour  = 2_000,
        policy_checks_per_hour = 200,
        llm_calls_per_hour    = 100,
        description           = "Standard employee access. Default role."
    ),

    "intern": RateLimitConfig(
        requests_per_hour     = 100,
        ip_requests_per_hour  = 500,
        policy_checks_per_hour = 50,
        llm_calls_per_hour    = 20,
        description           = "Limited access. Intern/trainee role."
    ),

    "guest": RateLimitConfig(
        requests_per_hour     = 30,
        ip_requests_per_hour  = 100,
        policy_checks_per_hour = 10,
        llm_calls_per_hour    = 5,
        description           = "Minimal read-only access. External users."
    ),

    "service": RateLimitConfig(
        requests_per_hour     = 50_000,
        ip_requests_per_hour  = 100_000,
        policy_checks_per_hour = 20_000,
        llm_calls_per_hour    = 10_000,
        description           = "Machine-to-machine. Internal services calling the API."
    ),

    # Default: unknown roles treated like guests
    "unknown": RateLimitConfig(
        requests_per_hour     = 20,
        ip_requests_per_hour  = 50,
        policy_checks_per_hour = 5,
        llm_calls_per_hour    = 2,
        description           = "Unknown/unrecognized role. Minimal access."
    ),
}

# ─── Per-Model Rate Limits ─────────────────────────────────────────────────────
# Some models are expensive. Some are high-risk.
# These limits apply ON TOP OF role limits (the tighter limit wins).
MODEL_LIMITS: dict[str, int] = {
    # Expensive flagship models — tight limits
    "gpt-4-turbo":       200,   # per hour, per model globally
    "gpt-4o":            300,
    "claude-opus":       150,
    "claude-sonnet":     400,
    "gemini-ultra":      200,

    # Mid-tier — moderate limits
    "gpt-3.5-turbo":    2000,
    "claude-haiku":     2000,
    "gemini-pro":       1500,

    # Local / cheap models — generous limits
    "llama-3-local":   10000,
    "mistral-local":   10000,
}

# ─── Paths that skip rate limiting entirely ────────────────────────────────────
BYPASS_PATHS = {
    "/",
    "/health",
    "/health/",
    "/docs",
    "/openapi.json",
    "/redoc",
    "/metrics",
}


def get_rate_limit_for_request(
    user_role: str,
    path:      str,
) -> RateLimitConfig:
    """
    Returns the applicable rate limit config for a given role.
    Falls back to 'unknown' if role is unrecognized.
    """
    role = user_role.lower() if user_role else "unknown"
    return ROLE_LIMITS.get(role, ROLE_LIMITS["unknown"])


async def check_model_rate_limit(model_id: str, window: str) -> tuple[bool, int]:
    """
    Check if a specific model has exceeded its global hourly call limit.
    Returns (is_within_limit: bool, current_count: int).
    """
    from backend.cache.redis_client import redis_incr, get_redis

    limit = MODEL_LIMITS.get(model_id)
    if not limit:
        return True, 0   # Unknown models: no model-level limit

    key   = f"rate:model:{model_id}:{window}"
    count = await redis_incr(key, ttl_seconds=3600)

    return count <= limit, count
```

---

<a id="step-45"></a>
## ⚙️ Step 4.5 — Policy Check Response Caching

The policy engine runs regex against every prompt. For high-traffic systems, the same prompts appear repeatedly. Cache the results.

```python
# backend/routers/policies.py  (updated check_policy with caching)
import hashlib
import json
from backend.cache.redis_client import cache_get_json, cache_set_json, redis_incr

# ─── Cache TTL Constants ───────────────────────────────────────────────────────
POLICY_CACHE_TTL  = 300    # 5 minutes: policy rules don't change often
LLM_CACHE_TTL     = 3600   # 1 hour: LLM responses for identical safe prompts
MODEL_CACHE_TTL   = 600    # 10 minutes: model registry entries


@router.post("/check", response_model=PolicyCheckResponse)
async def check_policy(
    request: PolicyCheckRequest,
    db: AsyncSession = Depends(get_db)
):
    """
    Policy check with Redis caching.

    Cache key: SHA256(prompt + user_role + model_id)
    TTL: 5 minutes

    Why include user_role in the cache key?
    Because role-restriction rules produce different outcomes
    for the same prompt depending on who's asking.
    "delete all records" → blocked for intern, allowed for admin.
    """
    # ── Build cache key ────────────────────────────────────────────
    cache_input = f"{request.prompt}:{request.user_role}:{request.model_id}"
    cache_key   = f"cache:policy:{hashlib.sha256(cache_input.encode()).hexdigest()}"

    # ── Try cache first ────────────────────────────────────────────
    cached = await cache_get_json(cache_key)
    if cached:
        # Track cache hit in Redis counter
        await redis_incr("cache:stats:policy:hits", ttl_seconds=86400)
        # Return cached result with cache indicator
        cached["cache_hit"] = True
        return PolicyCheckResponse(**cached)

    await redis_incr("cache:stats:policy:misses", ttl_seconds=86400)

    # ── Cache miss: run full policy check ──────────────────────────
    rules_result = await db.execute(
        select(PolicyRule).where(PolicyRule.enabled == True)
    )
    active_rules = rules_result.scalars().all()

    violations   = []
    rules_fired  = []
    risk_score   = 0
    modified     = request.prompt
    should_block = False

    for rule in active_rules:
        if not rule.pattern:
            continue
        if rule.applies_to_roles and request.user_role not in rule.applies_to_roles:
            continue

        try:
            if re.search(rule.pattern, request.prompt, re.IGNORECASE):
                violations.append(f"[{rule.rule_id}] {rule.name}")
                rules_fired.append(rule.rule_id)
                risk_score = min(100, risk_score + rule.risk_score_increment)

                if rule.action == RuleAction.block:
                    should_block = True
                elif rule.action == RuleAction.redact:
                    modified = re.sub(
                        rule.pattern,
                        f"[{rule.rule_id.upper()} REDACTED]",
                        modified,
                        flags=re.IGNORECASE
                    )
        except re.error:
            continue

    allowed  = not should_block
    check_id = str(uuid.uuid4())

    if should_block:
        audit_outcome = "blocked"
    elif violations:
        audit_outcome = "redacted"
    else:
        audit_outcome = "allowed"

    prompt_hash = hashlib.sha256(request.prompt.encode()).hexdigest()

    # ── Write to DB ────────────────────────────────────────────────
    db.add(PolicyCheck(
        check_id    = check_id,
        model_id    = request.model_id,
        user_id     = request.user_id,
        user_role   = request.user_role,
        prompt_hash = prompt_hash,
        violations  = violations,
        rules_fired = rules_fired,
        risk_score  = risk_score,
        allowed     = allowed,
        session_id  = request.session_id,
    ))

    db.add(AuditEvent(
        event_id    = f"evt-{str(uuid.uuid4())}",
        user_id     = request.user_id,
        user_role   = request.user_role,
        model_id    = request.model_id,
        action_type = "policy_check",
        resource    = f"policy_check:{check_id}",
        outcome     = audit_outcome,
        prompt_hash = prompt_hash,
        details     = {
            "violations":  violations,
            "rules_fired": rules_fired,
            "risk_score":  risk_score,
            "check_id":    check_id,
            "cache_hit":   False,
        },
        session_id  = request.session_id,
        timestamp   = datetime.now(timezone.utc),
    ))

    result = PolicyCheckResponse(
        allowed         = allowed,
        violations      = violations,
        rules_fired     = rules_fired,
        modified_prompt = modified,
        risk_score      = risk_score,
        check_id        = check_id,
    )

    # ── Cache the result ───────────────────────────────────────────
    # IMPORTANT: Only cache SAFE (allowed) results.
    # Blocked results may be security incidents — always re-evaluate them.
    # Never serve a cached "blocked" from a different user's session.
    if allowed:
        await cache_set_json(
            cache_key,
            result.model_dump(),
            ttl_seconds=POLICY_CACHE_TTL
        )

    return result
```

---

<a id="step-46"></a>
## ⚙️ Step 4.6 — LLM Response Caching (Cost Governance)

Every identical LLM call that hits the cache is money saved. On a system processing 10,000 calls/day with 30% cache hit rate, that is hundreds of dollars per month.

```python
# backend/llm/governed_client.py  (updated with caching)
import hashlib
import httpx
from openai import AsyncOpenAI
from typing import Optional
from datetime import datetime, timezone

from backend.cache.redis_client import cache_get_json, cache_set_json, redis_incr

GOVERNANCE_API  = "http://localhost:8000"
LLM_CACHE_TTL   = 3600    # 1 hour


class GovernedLLMClient:
    """
    LLM client with full governance stack:
      1. Policy check (with Redis cache)
      2. Model rate limit check
      3. LLM response cache (identical safe prompts reuse cached output)
      4. Actual LLM call (only if cache miss)
      5. Audit log
    """

    def __init__(self, user_id: str, user_role: str, model_id: str):
        self.user_id   = user_id
        self.user_role = user_role
        self.model_id  = model_id
        self.openai    = AsyncOpenAI()

    def _prompt_hash(self, prompt: str) -> str:
        return hashlib.sha256(prompt.encode()).hexdigest()

    async def complete(
        self,
        prompt:       str,
        session_id:   Optional[str] = None,
        use_cache:    bool = True,
        system_prompt: Optional[str] = None,
    ) -> dict:
        """
        Governed LLM completion with caching.

        Flow:
          policy_check → model_rate_limit → llm_cache → llm_call → audit_log
        """
        async with httpx.AsyncClient(timeout=30.0) as client:

            # ── Step 1: Policy check ─────────────────────────────────
            policy_resp = await client.post(
                f"{GOVERNANCE_API}/policies/check",
                json={
                    "prompt":     prompt,
                    "user_role":  self.user_role,
                    "model_id":   self.model_id,
                    "user_id":    self.user_id,
                    "session_id": session_id,
                },
                headers={
                    "X-User-ID":   self.user_id,
                    "X-User-Role": self.user_role,
                }
            )
            policy = policy_resp.json()

            if not policy["allowed"]:
                return {
                    "blocked":      True,
                    "reason":       policy["violations"],
                    "risk_score":   policy["risk_score"],
                    "output":       None,
                    "cache_hit":    False,
                    "from_cache":   False,
                }

            clean_prompt = policy["modified_prompt"]

            # ── Step 2: LLM response cache lookup ────────────────────
            if use_cache:
                cache_input = f"{clean_prompt}:{self.model_id}:{system_prompt or ''}"
                llm_cache_key = f"cache:llm:{self._prompt_hash(cache_input)}"
                cached_response = await cache_get_json(llm_cache_key)

                if cached_response:
                    await redis_incr("cache:stats:llm:hits", ttl_seconds=86400)
                    return {
                        "blocked":    False,
                        "output":     cached_response["output"],
                        "risk_score": policy["risk_score"],
                        "warnings":   policy["violations"],
                        "from_cache": True,
                        "cache_key":  llm_cache_key,
                        "tokens_used": 0,    # Cached: zero cost
                        "cost_usd":    0.0,
                    }

                await redis_incr("cache:stats:llm:misses", ttl_seconds=86400)

            # ── Step 3: Actual LLM call ───────────────────────────────
            messages = []
            if system_prompt:
                messages.append({"role": "system", "content": system_prompt})
            messages.append({"role": "user", "content": clean_prompt})

            response = await self.openai.chat.completions.create(
                model=self.model_id,
                messages=messages,
            )
            output     = response.choices[0].message.content
            tokens_used = response.usage.total_tokens

            # ── Step 4: Cache the LLM response ───────────────────────
            # Only cache if the prompt was fully clean (no violations at all).
            # We don't cache responses to redacted prompts, because the original
            # prompt intent was different and caching would hide that.
            if use_cache and not policy["violations"]:
                await cache_set_json(
                    llm_cache_key,
                    {"output": output, "model_id": self.model_id},
                    ttl_seconds=LLM_CACHE_TTL,
                )

            # ── Step 5: Audit log the LLM call ───────────────────────
            await client.post(
                f"{GOVERNANCE_API}/audit/log",
                json={
                    "user_id":     self.user_id,
                    "user_role":   self.user_role,
                    "model_id":    self.model_id,
                    "action_type": "llm_call",
                    "outcome":     "allowed",
                    "prompt_hash": self._prompt_hash(prompt),
                    "output_hash": self._prompt_hash(output),
                    "details":     {
                        "tokens_used": tokens_used,
                        "from_cache":  False,
                        "risk_score":  policy["risk_score"],
                    },
                    "session_id":  session_id,
                },
                headers={
                    "X-User-ID":   self.user_id,
                    "X-User-Role": self.user_role,
                }
            )

            return {
                "blocked":     False,
                "output":      output,
                "risk_score":  policy["risk_score"],
                "warnings":    policy["violations"],
                "from_cache":  False,
                "tokens_used": tokens_used,
            }
```

---

<a id="step-47"></a>
## ⚙️ Step 4.7 — Model Registry Cache

The model registry is queried on every single LLM call to verify the model is registered and approved. That is a database hit every time. Cache it.

```python
# backend/routers/models.py  (add caching to get_model and register_model)
from backend.cache.redis_client import cache_get_json, cache_set_json, redis_delete

MODEL_CACHE_TTL = 600   # 10 minutes


@router.get("/{model_id}", response_model=AIModelResponse)
async def get_model(model_id: str, db: AsyncSession = Depends(get_db)):
    """
    Get a model by ID. Cached in Redis for 10 minutes.
    Cache is invalidated on model updates or approval changes.
    """
    cache_key = f"cache:model_registry:{model_id}"

    # ── Try cache first ────────────────────────────────────────────
    cached = await cache_get_json(cache_key)
    if cached:
        return AIModelResponse(**cached)

    # ── Cache miss: hit the database ───────────────────────────────
    result = await db.execute(
        select(AIModel).where(AIModel.model_id == model_id)
    )
    model = result.scalar_one_or_none()
    if not model:
        raise HTTPException(
            status_code=404,
            detail=f"Model '{model_id}' not found in registry."
        )

    # ── Cache the result ───────────────────────────────────────────
    model_dict = {
        "model_id":    model.model_id,
        "name":        model.name,
        "provider":    model.provider,
        "version":     model.version,
        "owner":       model.owner,
        "risk_level":  model.risk_level,
        "pii_access":  model.pii_access,
        "approved":    model.approved,
        "approved_by": model.approved_by,
        "approved_at": model.approved_at.isoformat() if model.approved_at else None,
        "description": model.description,
        "created_at":  model.created_at.isoformat(),
        "updated_at":  model.updated_at.isoformat(),
    }
    await cache_set_json(cache_key, model_dict, ttl_seconds=MODEL_CACHE_TTL)

    return model


@router.patch("/{model_id}/approve", response_model=AIModelResponse)
async def approve_model(
    model_id: str,
    approval: AIModelApprove,
    db: AsyncSession = Depends(get_db)
):
    """Approve model. Invalidates the model cache immediately."""
    result = await db.execute(
        select(AIModel).where(AIModel.model_id == model_id)
    )
    model = result.scalar_one_or_none()
    if not model:
        raise HTTPException(status_code=404, detail=f"Model '{model_id}' not found")

    model.approved    = True
    model.approved_by = approval.approved_by
    model.approved_at = datetime.utcnow()
    model.updated_at  = datetime.utcnow()

    # ── Invalidate cache so next read gets fresh data ──────────────
    await redis_delete(f"cache:model_registry:{model_id}")

    await db.flush()
    await db.refresh(model)
    return model
```

---

<a id="step-48"></a>
## ⚙️ Step 4.8 — Cache and Rate Limit Stats Endpoints

You need to see what Redis is doing. These endpoints expose cache hit rates and current rate limit status.

```python
# backend/routers/cache_stats.py
from fastapi import APIRouter, Depends
from backend.cache.redis_client import get_redis, redis_get

router = APIRouter()


@router.get("/stats", summary="Cache hit rates and memory usage")
async def cache_stats():
    """
    Shows cache performance metrics.
    Use this to tune TTLs and understand cache effectiveness.

    A healthy system should show:
    - Policy cache hit rate: 40-70% (many repeated prompts)
    - LLM cache hit rate: 20-50% (common questions reuse responses)
    - Model cache hit rate: 90%+ (model data rarely changes)
    """
    policy_hits   = int(await redis_get("cache:stats:policy:hits")  or 0)
    policy_misses = int(await redis_get("cache:stats:policy:misses") or 0)
    llm_hits      = int(await redis_get("cache:stats:llm:hits")      or 0)
    llm_misses    = int(await redis_get("cache:stats:llm:misses")    or 0)

    def hit_rate(hits, misses):
        total = hits + misses
        return round(hits / total * 100, 1) if total > 0 else 0.0

    # Get Redis memory info
    r = await get_redis()
    memory_info = {}
    if r:
        try:
            info = await r.info("memory")
            memory_info = {
                "used_memory_human":       info.get("used_memory_human"),
                "used_memory_peak_human":  info.get("used_memory_peak_human"),
                "maxmemory_human":         info.get("maxmemory_human", "no limit"),
                "mem_fragmentation_ratio": info.get("mem_fragmentation_ratio"),
            }
        except Exception:
            memory_info = {"error": "Could not fetch Redis memory info"}

    return {
        "policy_cache": {
            "hits":     policy_hits,
            "misses":   policy_misses,
            "hit_rate": f"{hit_rate(policy_hits, policy_misses)}%",
        },
        "llm_cache": {
            "hits":     llm_hits,
            "misses":   llm_misses,
            "hit_rate": f"{hit_rate(llm_hits, llm_misses)}%",
        },
        "redis_memory": memory_info,
    }


@router.get("/rate-limit/status/{user_id}", summary="Check current rate limit usage for a user")
async def rate_limit_status(user_id: str):
    """
    See how many requests a user has made in the current hour.
    Used by admins to investigate rate limit complaints.

    Returns remaining capacity for each role-based limit.
    """
    from datetime import datetime, timezone
    from backend.middleware.rate_limit_config import ROLE_LIMITS
    from backend.cache.redis_client import redis_ttl

    now    = datetime.now(timezone.utc)
    window = now.strftime("%Y-%m-%dT%H")
    key    = f"rate:user:{user_id}:{window}"

    current_count = int(await redis_get(key) or 0)
    ttl           = await redis_ttl(key)

    # Show status against all role limits for reference
    role_comparison = {}
    for role, config in ROLE_LIMITS.items():
        remaining = max(0, config.requests_per_hour - current_count)
        role_comparison[role] = {
            "limit":     config.requests_per_hour,
            "used":      current_count,
            "remaining": remaining,
            "exceeded":  current_count > config.requests_per_hour,
        }

    return {
        "user_id":         user_id,
        "current_window":  window,
        "requests_made":   current_count,
        "window_resets_in_seconds": ttl if ttl > 0 else 3600,
        "by_role":         role_comparison,
    }


@router.delete("/flush", summary="Flush all caches (admin only — use carefully)")
async def flush_cache(confirm: str = "no"):
    """
    Flush all cached data. Use only when policy rules have been updated
    and you need cache to reflect the new rules immediately.

    Requires confirm=yes query parameter to prevent accidental flush.
    """
    if confirm.lower() != "yes":
        return {
            "flushed": False,
            "message": "Add ?confirm=yes to actually flush the cache."
        }

    r = await get_redis()
    if not r:
        return {"flushed": False, "message": "Redis unavailable"}

    # Only flush cache keys, not rate limit keys
    # We don't want to reset rate limits (that would defeat the purpose)
    cache_keys = await r.keys("cache:*")
    if cache_keys:
        await r.delete(*cache_keys)

    return {
        "flushed":      True,
        "keys_deleted": len(cache_keys),
        "message":      f"Flushed {len(cache_keys)} cache keys. Rate limit counters preserved."
    }
```

---

<a id="step-49"></a>
## ⚙️ Step 4.9 — Update Docker Compose and main.py

### Update docker-compose.yml

Redis was already declared in the Day 1 docker-compose. Verify it is configured correctly with persistence:

```yaml
# docker-compose.yml  (redis section — update to this)
  redis:
    image: redis:7-alpine
    container_name: governance-redis
    ports:
      - "6379:6379"
    command: >
      redis-server
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
    volumes:
      - redis-data:/data
    networks:
      - governance-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

# Add to volumes section:
volumes:
  postgres-data:
  prometheus-data:
  grafana-data:
  redis-data:          # ← Add this
```

**Redis config explained:**
- `maxmemory 256mb` — prevents Redis from eating all RAM
- `allkeys-lru` — when memory is full, evict least-recently-used keys (safe for cache)
- `appendonly yes` — write-ahead log for durability (rate limit counters survive restarts)
- `appendfsync everysec` — sync to disk every second (balance between safety and speed)

### Update main.py with Redis Lifecycle

```python
# backend/main.py  (updated with Redis)
from fastapi import FastAPI
from contextlib import asynccontextmanager

from backend.database.db import init_db, close_db, AsyncSessionLocal
from backend.cache.redis_client import init_redis, close_redis
from backend.routers import models, policies, audit, health, cache_stats
from backend.routers.policies import seed_default_rules
from backend.middleware.rate_limiter import RateLimitMiddleware


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ── Startup ──────────────────────────────────────────────────
    print("🚀 Governance API starting up...")

    # Initialize PostgreSQL
    await init_db()
    async with AsyncSessionLocal() as db:
        await seed_default_rules(db)
    print("✅ PostgreSQL ready. Default rules seeded.")

    # Initialize Redis (non-fatal if unavailable)
    try:
        await init_redis()
        print("✅ Redis connected. Caching and rate limiting active.")
    except Exception as e:
        print(f"⚠️  Redis unavailable ({e}). Running without cache/rate limiting.")
        print("    → Start Redis: docker compose up redis -d")

    print("✅ Ready to accept traffic.\n")
    yield

    # ── Shutdown ─────────────────────────────────────────────────
    print("⏬ Governance API shutting down...")
    await close_db()
    await close_redis()
    print("✅ Shutdown complete.")


app = FastAPI(
    title="AI Governance API",
    description="Central governance layer for all AI systems.",
    version="0.4.0",
    lifespan=lifespan,
)

# ─── Middleware (order matters: runs outermost first) ─────────────────────────
app.add_middleware(RateLimitMiddleware)

# ─── Routers ──────────────────────────────────────────────────────────────────
app.include_router(health.router,       prefix="/health",      tags=["Health"])
app.include_router(models.router,       prefix="/models",      tags=["Model Registry"])
app.include_router(policies.router,     prefix="/policies",    tags=["Policy Engine"])
app.include_router(audit.router,        prefix="/audit",       tags=["Audit Logs"])
app.include_router(cache_stats.router,  prefix="/cache",       tags=["Cache & Rate Limits"])


@app.get("/", tags=["Root"])
async def root():
    return {
        "service": "AI Governance API",
        "version": "0.4.0",
        "docs":    "/docs",
        "status":  "operational"
    }
```

### Updated Project Structure

```
ai-governance/
├── backend/
│   ├── main.py
│   ├── routers/
│   │   ├── models.py
│   │   ├── policies.py
│   │   ├── audit.py
│   │   ├── health.py
│   │   └── cache_stats.py       ← NEW today
│   ├── cache/
│   │   └── redis_client.py      ← NEW today
│   ├── middleware/
│   │   ├── rate_limiter.py      ← NEW today
│   │   └── rate_limit_config.py ← NEW today
│   ├── llm/
│   │   └── governed_client.py   ← UPDATED today
│   └── database/
│       ├── db.py
│       └── models.py
```

---

<a id="step-410"></a>
## ⚙️ Step 4.10 — Testing Everything

### Start the Full Stack

```bash
# Start PostgreSQL + Redis
docker compose up db redis -d

# Verify both are running
docker compose ps

# Run migrations
alembic upgrade head

# Start the API
uvicorn backend.main:app --reload --port 8000

# You should see:
# ✅ PostgreSQL ready. Default rules seeded.
# ✅ Redis connected. Caching and rate limiting active.
# ✅ Ready to accept traffic.
```

### Test Rate Limiting

```bash
# ── Test 1: Check rate limit headers in every response ────────────────────────
curl -v -X POST http://localhost:8000/policies/check \
  -H "Content-Type: application/json" \
  -H "X-User-ID: alice" \
  -H "X-User-Role: employee" \
  -d '{"prompt": "hello", "user_role": "employee", "model_id": "test-model", "user_id": "alice"}'

# Check response headers:
# X-RateLimit-Limit: 500
# X-RateLimit-Remaining: 499
# X-RateLimit-Used: 1
# X-RateLimit-Window: 3600

# ── Test 2: Check rate limit status for a user ────────────────────────────────
curl http://localhost:8000/cache/rate-limit/status/alice

# ── Test 3: Hit the intern rate limit (20 requests/hour) ─────────────────────
# Run this loop to trigger the limit:
for i in {1..25}; do
  curl -s -o /dev/null -w "Request $i: HTTP %{http_code}\n" \
    -X POST http://localhost:8000/policies/check \
    -H "Content-Type: application/json" \
    -H "X-User-ID: test-intern-$(date +%s)" \
    -H "X-User-Role: intern" \
    -d '{"prompt": "test prompt '$i'", "user_role": "intern", "model_id": "test", "user_id": "intern-test"}'
done

# First 20: HTTP 200
# Request 21+: HTTP 429

# ── Test 4: Verify 429 response body ─────────────────────────────────────────
# After hitting the limit, the response looks like:
# {
#   "error": "rate_limit_exceeded",
#   "message": "Rate limit of 100 requests/hour exceeded.",
#   "limit": 100,
#   "window": "1 hour",
#   "retry_after": 3487
# }
```

### Test Response Caching

```bash
# ── Test 1: First request — cache MISS, slower ────────────────────────────────
time curl -s -X POST http://localhost:8000/policies/check \
  -H "Content-Type: application/json" \
  -H "X-User-ID: alice" \
  -H "X-User-Role: employee" \
  -d '{"prompt": "What is the weather today?", "user_role": "employee", "model_id": "test", "user_id": "alice"}'

# Note the response time (includes DB query)

# ── Test 2: Second identical request — cache HIT, faster ─────────────────────
time curl -s -X POST http://localhost:8000/policies/check \
  -H "Content-Type: application/json" \
  -H "X-User-ID: alice" \
  -H "X-User-Role: employee" \
  -d '{"prompt": "What is the weather today?", "user_role": "employee", "model_id": "test", "user_id": "alice"}'

# Response includes: "cache_hit": true
# Response time should be significantly faster

# ── Test 3: Check cache stats ─────────────────────────────────────────────────
curl http://localhost:8000/cache/stats

# Expected:
# {
#   "policy_cache": { "hits": 1, "misses": 1, "hit_rate": "50.0%" },
#   "llm_cache":    { "hits": 0, "misses": 0, "hit_rate": "0.0%" },
#   "redis_memory": { "used_memory_human": "2.50M", ... }
# }

# ── Test 4: Verify cache key exists in Redis ──────────────────────────────────
docker exec -it governance-redis redis-cli
> KEYS cache:policy:*
> TTL cache:policy:abc123...
> GET cache:policy:abc123...
> QUIT
```

### Test Model Registry Cache

```bash
# Register a model
curl -X POST http://localhost:8000/models/register \
  -H "Content-Type: application/json" \
  -d '{
    "model_id": "gpt-4-cache-test",
    "name": "Cache Test Model",
    "provider": "openai",
    "version": "gpt-4-turbo",
    "owner": "test-team",
    "risk_level": "medium",
    "pii_access": false,
    "description": "Testing model registry caching"
  }'

# First GET — hits database, populates cache
curl http://localhost:8000/models/gpt-4-cache-test

# Second GET — served from Redis cache
curl http://localhost:8000/models/gpt-4-cache-test

# Approve the model — should invalidate cache
curl -X PATCH http://localhost:8000/models/gpt-4-cache-test/approve \
  -H "Content-Type: application/json" \
  -d '{"approved_by": "lead-engineer"}'

# GET after approval — cache was invalidated, fresh data from DB
curl http://localhost:8000/models/gpt-4-cache-test
# → approved: true (not stale cached false)

# Verify in Redis:
docker exec -it governance-redis redis-cli
> KEYS cache:model_registry:*
> GET cache:model_registry:gpt-4-cache-test
```

### Test Graceful Degradation

```bash
# Stop Redis — system should still work, just without caching
docker compose stop redis

# Policy check still works (falls back to database)
curl -X POST http://localhost:8000/policies/check \
  -H "Content-Type: application/json" \
  -H "X-User-ID: alice" \
  -H "X-User-Role: employee" \
  -d '{"prompt": "test graceful degradation", "user_role": "employee", "model_id": "test", "user_id": "alice"}'

# Should return 200 OK with normal result (not a 500 error)
# You should see a warning in the server logs:
# WARNING: Redis GET failed for key=...: ...

# Restart Redis — caching resumes automatically on next request
docker compose start redis
```

---

<a id="what-you-built"></a>
## 🏁 What You Built Today

```
Week 1 Progress:
  Day 1 → FastAPI skeleton + in-memory everything
  Day 2 → PostgreSQL persistence for all data
  Day 3 → Full audit system with queries, stats, CSV export
  Day 4 → Redis: rate limiting + caching  ← TODAY

New capabilities added:
  ✅ Redis connection manager with graceful degradation
  ✅ Sliding window rate limiter (per user, per IP)
  ✅ Role-based rate limits (6 roles, carefully tuned)
  ✅ Per-model rate limits for expensive models
  ✅ Rate limit headers on every response (X-RateLimit-*)
  ✅ 429 responses with retry-after guidance
  ✅ Policy check caching (5 min TTL, safe-results only)
  ✅ LLM response caching (1 hour TTL, identical prompts)
  ✅ Model registry caching (10 min TTL, invalidated on update)
  ✅ Cache stats endpoint (hit rates, memory usage)
  ✅ Rate limit status endpoint (per user)
  ✅ Cache flush endpoint (admin, with confirmation)
  ✅ Docker Compose Redis config with persistence and memory limits
  ✅ Updated main.py with Redis lifecycle management
```

### The Complete Governance Stack After Day 4

```
INCOMING REQUEST
      │
      ▼
[Rate Limit Middleware]   ← Redis: sliding window per user/IP
      │ PASS
      ▼
[Route Handler]
      │
      ├──→ [Cache Lookup]        ← Redis: hit = instant return
      │         │ MISS
      │         ▼
      ├──→ [PostgreSQL Query]    ← DB: only on cache miss
      │         │
      │         ▼
      ├──→ [Cache Write]         ← Redis: store result
      │         │
      ▼         ▼
[Audit Logger]                  ← PostgreSQL: immutable event
      │
      ▼
RESPONSE
  + X-RateLimit-* headers
  + cache_hit indicator
```

---

<a id="checklist"></a>
## ✅ Day 4 Checklist

Before moving to Day 5, verify all of these:

- [ ] Redis starts: `docker compose up redis -d` and `redis-cli ping` returns PONG
- [ ] API starts with "Redis connected" message in logs
- [ ] Every response has `X-RateLimit-Limit`, `X-RateLimit-Remaining` headers
- [ ] Intern role (`X-User-Role: intern`) hits 429 after 100 requests/hour
- [ ] 429 response includes `retry_after` and `limit` in body
- [ ] Second identical policy check returns `"cache_hit": true`
- [ ] Cache hit is noticeably faster than first call
- [ ] `GET /cache/stats` shows non-zero hit/miss counts
- [ ] `GET /cache/rate-limit/status/{user_id}` shows correct usage
- [ ] Model approval invalidates the registry cache (re-fetch returns updated data)
- [ ] Stopping Redis: API still works, logs show Redis warnings, no 500 errors
- [ ] Restarting Redis: caching resumes automatically
- [ ] Redis persistence: `docker compose restart redis` and rate limit counters still exist
- [ ] `DELETE /cache/flush?confirm=yes` clears cache keys, preserves rate limit keys

---

## 📚 What to Read Tonight

| Resource | Topic |
|---|---|
| [Redis Commands Reference](https://redis.io/commands/) | INCR, SETEX, EXPIRE, KEYS, TTL |
| [redis-py Async Docs](https://redis-py.readthedocs.io/en/stable/examples/asyncio_examples.html) | Async Redis in Python |
| [FastAPI Middleware Guide](https://fastapi.tiangolo.com/tutorial/middleware/) | How middleware intercepts requests |
| [Sliding Window Rate Limiting](https://www.figma.com/blog/an-alternative-approach-to-rate-limiting/) | The algorithm explained |
| [Redis Eviction Policies](https://redis.io/docs/latest/develop/reference/eviction/) | allkeys-lru and why we use it |

---

## 🔮 Coming Up

```
Day 5  → Unit Tests: pytest + httpx covering all Week 1 components
Day 6  → GovernedLLMClient: real OpenAI calls flowing through governance
Day 7  → GovernedRAG: governed document indexing + retrieval
```

> Day 5 is test day. You'll write pytest tests for every endpoint built
> across Days 1-4. Rate limiter tests, cache hit tests, policy engine tests,
> audit query tests. This is what separates a learning project from
> production-grade software.

---

*Day 4 complete. The governance layer now has memory (PostgreSQL), a voice (audit logs), and boundaries (rate limits + caching).*

*One user can no longer take down the system. Costs are controlled. Identical requests are answered from memory.*

---

**Repository:** [GuntruTirupathamma/AI-Governance](https://github.com/GuntruTirupathamma/AI-Governance)
**Series:** AI Governance Engineering from Scratch
**Next:** `DAY5.md` → Unit Tests for Week 1 (pytest + httpx)
