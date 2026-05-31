# 📊 DAY 11 — Dashboard: Real-Time Governance Metrics with WebSockets

> **Series:** AI Governance Engineering — Zero to Production  **Author:** GuntruTirupathamma  **Day:** 11 of 35  **Previous:** [DAY10.md](https://github.com/GuntruTirupathamma/AI-Governance/blob/main/DAY10.md) — End-to-End Test: Full Pipeline Verified  **Next:** DAY12.md — Multi-Tenant Isolation: Namespace All Data by org_id

---

## 📌 Table of Contents

1. [Why Real-Time Governance Metrics Are Not Optional](#why-metrics)
2. [What You'll Build Today](#what-youll-build)
3. [The Metrics Dashboard Architecture](#metrics-architecture)
4. [Step 11.1 — Install Dependencies](#step-111)
5. [Step 11.2 — GovernanceMetricsSnapshot Data Model](#step-112)
6. [Step 11.3 — MetricsCollector: Aggregating All Sources](#step-113)
7. [Step 11.4 — Policy Metrics](#step-114)
8. [Step 11.5 — LLM and Cost Metrics](#step-115)
9. [Step 11.6 — Output Scan and Streaming Metrics](#step-116)
10. [Step 11.7 — WebSocket Connection Manager](#step-117)
11. [Step 11.8 — WebSocket Endpoint: Live Push Every 5 Seconds](#step-118)
12. [Step 11.9 — REST Snapshot Endpoints](#step-119)
13. [Step 11.10 — Testing Metrics and WebSocket Layer](#step-1110)
14. [What You Built Today](#what-you-built)
15. [Day 11 Checklist](#checklist)

---

## ⚡ Why Real-Time Governance Metrics Are Not Optional

Days 1–10 built governance controls that generate data with every single request. Policy block rates, toxicity scores, hallucination rates, cost per model, retraction rates — all of it flows into PostgreSQL and Redis every second. Without a metrics layer, this data sits invisible and useless.

### The Four Scenarios That Demand Real-Time Visibility

```
Scenario 1 — The Silent Spike
  Monday morning. The team arrives.
  Overnight, a misconfigured agent ran 4,000 LLM calls.
  Policy block rate jumped from 2% to 47%.
  Cost: $380 in OpenAI charges while everyone slept.

  Without real-time metrics: discovered at 9 AM when someone checks bills.
  With real-time metrics: dashboard shows the spike at 2:17 AM.
                          Alerting fires at 2:18 AM.
                          On-call engineer cancels the agent at 2:19 AM.
                          Total extra cost: $3.40 instead of $380.

Scenario 2 — The Drifting Model
  A model that was performing well last week now shows:
    Average hallucination score:    0.83 → 0.51  (dropping)
    Average confidence score:       0.91 → 0.62  (dropping)
    Output scan block rate:         1.2% → 8.7%  (rising)

  Without real-time metrics: users start complaining about bad answers.
                              Investigation takes 2 days.
  With real-time metrics: the drift is visible on the dashboard in real time.
                           The model is pulled and reviewed before users notice.

Scenario 3 — The Compliance Audit Question
  An auditor asks:
  "Between 9 AM and 5 PM on March 15, 2026, what was your policy
   block rate and what were the top three rules that fired?"

  Without metrics history: hours spent writing ad-hoc SQL queries.
  With metrics dashboard: select the date range. Export. Done in 90 seconds.

Scenario 4 — The Cost Runaway by Department
  Finance asks: "Which team is spending the most on AI this month?"
  Without cost metrics by user/model: unknown.
  With real-time cost dashboard:
    by_model:   { "gpt-4o": $847, "gpt-3.5-turbo": $42 }
    by_user:    { "analytics-team": $612, "hr-bot": $277 }
    today:      $189 (currently updating every 5 seconds)

  Finance can now hold teams accountable for AI spend.
  That accountability only exists if the data is visible.

```

**Every governance control you built in Days 1–10 is only as strong as the visibility into whether it is working. Metrics are not a dashboard feature — they are the proof of governance.**

---

## 🎯 What You'll Build Today

```
BEFORE (Day 10):
  Metrics data exists scattered across PostgreSQL tables and Redis keys
  No single view of overall governance health
  No real-time updates — data is stale until someone queries manually
  No WebSocket push — clients must poll REST endpoints
  No historical metric snapshots (point-in-time governance state)
  Compliance team has no self-service visibility
  Cost monitoring requires direct Redis inspection

AFTER (Day 11):
  ✅ GovernanceMetricsSnapshot: typed dataclass for a full system snapshot
  ✅ MetricsCollector: aggregates policy + LLM + RAG + output + streaming data
  ✅ Policy metrics: block rate, top fired rules, risk score distribution
  ✅ LLM metrics: calls/hour, cache hit rate, cost by model, token totals
  ✅ RAG metrics: documents indexed, retrieval counts, clearance breakdown
  ✅ Output scan metrics: toxicity rate, hallucination rate, retraction rate
  ✅ Streaming metrics: total streams, retraction rate, avg duration
  ✅ System health metrics: Redis memory, DB connection count, uptime
  ✅ WebSocketConnectionManager: broadcast to multiple dashboard clients
  ✅ WS /ws/metrics: live push every 5 seconds to all connected clients
  ✅ GET /metrics/snapshot: full current metrics in one REST call
  ✅ GET /metrics/policy: policy-specific metrics with time range filter
  ✅ GET /metrics/cost: cost breakdown by model, user, and date range
  ✅ GET /metrics/health: system health at a glance
  ✅ MetricsSnapshot PostgreSQL model: hourly snapshots for history
  ✅ Background task: snapshot writer runs every hour automatically
  ✅ 12+ tests covering metrics collection and WebSocket behavior

```

---

## 🏛️ The Metrics Dashboard Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    GOVERNANCE METRICS ARCHITECTURE                       │
│                                                                          │
│  DATA SOURCES (all built in Days 1-10):                                  │
│                                                                          │
│  PostgreSQL tables:               Redis keys:                            │
│  ├─ policy_checks                 ├─ cache:stats:policy:hits             │
│  ├─ audit_events                  ├─ cache:stats:llm:hits                │
│  ├─ output_scan_results           ├─ cost:user:{id}:{date}               │
│  ├─ indexed_documents             ├─ cost:model:{id}:{date}              │
│  ├─ retrieval_events              ├─ stream:stats:total                  │
│  └─ stream_sessions               └─ stream:stats:retractions            │
│                          │                    │                          │
│                          └────────┬───────────┘                          │
│                                   │                                      │
│                                   ▼                                      │
│                    ┌──────────────────────────────┐                      │
│                    │      MetricsCollector        │                      │
│                    │                              │                      │
│                    │  collect_policy_metrics()    │                      │
│                    │  collect_llm_metrics()       │                      │
│                    │  collect_rag_metrics()       │                      │
│                    │  collect_scan_metrics()      │                      │
│                    │  collect_stream_metrics()    │                      │
│                    │  collect_system_health()     │                      │
│                    │                              │                      │
│                    │  Returns:                    │                      │
│                    │  GovernanceMetricsSnapshot   │                      │
│                    └──────────────┬───────────────┘                      │
│                                   │                                      │
│              ┌────────────────────┼──────────────────────┐               │
│              │                    │                      │               │
│              ▼                    ▼                      ▼               │
│  ┌────────────────────┐  ┌────────────────┐  ┌───────────────────────┐  │
│  │  REST Endpoints    │  │  Background    │  │   WebSocket Server    │  │
│  │                    │  │  Task (hourly) │  │                       │  │
│  │  GET /metrics/     │  │                │  │  WS /ws/metrics       │  │
│  │    snapshot        │  │  Writes        │  │                       │  │
│  │    policy          │  │  MetricsSnapshot│  │  Every 5 seconds:     │  │
│  │    cost            │  │  to PostgreSQL │  │  collect snapshot     │  │
│  │    health          │  │  for history   │  │  → broadcast to all   │  │
│  └────────────────────┘  └────────────────┘  │    connected clients  │  │
│                                               │                       │  │
│                                               │  Client 1 ───────────┤  │
│                                               │  Client 2 ───────────┤  │
│                                               │  Client 3 ───────────┘  │
└──────────────────────────────────────────────────────────────────────────┘

GovernanceMetricsSnapshot structure:
  {
    "timestamp":    "2026-05-29T14:30:00Z",
    "period_hours": 24,
    "policy": {
      "total_checks":    1847,
      "blocked":          234,
      "allowed":         1613,
      "block_rate":    "12.7%",
      "top_rules_fired": [...],
      "risk_score_avg":   18.3,
    },
    "llm": {
      "total_calls":      892,
      "cache_hit_rate": "34.2%",
      "tokens_total":  1_840_000,
      "cost_total_usd":   74.82,
      "cost_by_model":   {...},
      "avg_latency_ms":   1840,
    },
    "rag": {
      "documents_indexed":  147,
      "total_retrievals":   634,
      "cache_hit_rate":  "41.8%",
      "by_clearance":      {...},
    },
    "output_scan": {
      "total_scans":       892,
      "blocked":            19,
      "redacted":           43,
      "toxicity_block_rate": "2.1%",
      "avg_confidence":    0.84,
    },
    "streaming": {
      "total_streams":     201,
      "retractions":         8,
      "retraction_rate": "3.98%",
    },
    "system": {
      "redis_memory_mb":   12.4,
      "uptime_hours":      47.2,
      "active_streams":       3,
    }
  }

WebSocket broadcast cadence:
  Every 5s  → full GovernanceMetricsSnapshot (for live dashboard)
  Every 60s → MetricsSnapshot written to PostgreSQL (for history)

```

---

## ⚙️ Step 11.1 — Install Dependencies

```bash
# Activate your virtual environment
source venv/bin/activate   # Windows: venv\Scripts\activate

# WebSocket support — already included in FastAPI/Starlette
# No additional package needed for basic WebSocket.
# Verify fastapi is already installed:
pip show fastapi
# Should show: Name: fastapi, Version: 0.111.x

# websockets — the underlying WebSocket library FastAPI uses
# Usually installed as a fastapi transitive dependency.
# Install explicitly to ensure latest version:
pip install websockets==12.0

# APScheduler — for the background hourly snapshot task
pip install apscheduler==3.10.4

# Update requirements
pip freeze > requirements.txt
```

Add metrics config to `.env`:

```bash
# .env additions for Day 11
METRICS_PUSH_INTERVAL_SECONDS=5       # WebSocket push cadence
METRICS_SNAPSHOT_INTERVAL_SECONDS=3600 # PostgreSQL snapshot cadence
METRICS_DEFAULT_PERIOD_HOURS=24       # Default lookback for aggregations
METRICS_ENABLED=true
WS_MAX_CONNECTIONS=50                 # Max simultaneous dashboard clients
```

---

## ⚙️ Step 11.2 — GovernanceMetricsSnapshot Data Model

```python
# backend/metrics/snapshot.py
"""
GovernanceMetricsSnapshot: a fully typed dataclass that captures
the complete governance health of the system at a point in time.

This is the contract between the MetricsCollector and every consumer
(WebSocket clients, REST endpoints, PostgreSQL history writer).
"""
from dataclasses import dataclass, field, asdict
from datetime import datetime, timezone
from typing import Optional
import json


@dataclass
class PolicyMetrics:
    """Aggregated policy engine metrics."""
    total_checks:     int   = 0
    blocked:          int   = 0
    allowed:          int   = 0
    redacted:         int   = 0
    block_rate:       str   = "0.0%"
    avg_risk_score:   float = 0.0
    top_rules_fired:  list  = field(default_factory=list)
    # List of {"rule_id": str, "name": str, "fire_count": int}
    risk_distribution: dict = field(default_factory=dict)
    # {"low": N, "medium": N, "high": N, "critical": N}


@dataclass
class LLMMetrics:
    """Aggregated LLM call and cost metrics."""
    total_calls:       int   = 0
    cache_hits:        int   = 0
    cache_misses:      int   = 0
    cache_hit_rate:    str   = "0.0%"
    tokens_input:      int   = 0
    tokens_output:     int   = 0
    tokens_total:      int   = 0
    cost_total_usd:    float = 0.0
    cost_by_model:     dict  = field(default_factory=dict)
    # {"gpt-4o": 47.20, "gpt-3.5-turbo": 3.10}
    cost_by_user:      dict  = field(default_factory=dict)
    avg_latency_ms:    float = 0.0
    output_blocked:    int   = 0


@dataclass
class RAGMetrics:
    """Aggregated RAG indexing and retrieval metrics."""
    documents_indexed:  int   = 0
    documents_rejected: int   = 0
    rejection_rate:     str   = "0.0%"
    total_retrievals:   int   = 0
    cache_hits:         int   = 0
    cache_hit_rate:     str   = "0.0%"
    by_clearance:       dict  = field(default_factory=dict)
    # {"public": 40, "internal": 82, "confidential": 15, "restricted": 10}
    avg_chunks_returned: float = 0.0


@dataclass
class OutputScanMetrics:
    """Aggregated output scanning metrics."""
    total_scans:         int   = 0
    blocked:             int   = 0
    redacted:            int   = 0
    warned:              int   = 0
    injected:            int   = 0   # Disclaimer injected
    allowed:             int   = 0
    block_rate:          str   = "0.0%"
    avg_toxicity_score:  float = 0.0
    avg_hallucination:   float = 0.0
    avg_confidence:      float = 0.0
    pii_detections:      int   = 0
    disclaimer_injections: int = 0
    by_model:            dict  = field(default_factory=dict)


@dataclass
class StreamingMetrics:
    """Aggregated streaming session metrics."""
    total_streams:    int   = 0
    completed:        int   = 0
    retracted:        int   = 0
    blocked:          int   = 0
    disconnected:     int   = 0
    retraction_rate:  str   = "0.0%"
    avg_duration_ms:  float = 0.0
    avg_tokens:       float = 0.0


@dataclass
class SystemHealthMetrics:
    """System-level health and resource metrics."""
    redis_connected:    bool  = False
    redis_memory_mb:    float = 0.0
    redis_hit_rate:     str   = "0.0%"
    db_connected:       bool  = False
    active_ws_clients:  int   = 0
    active_streams:     int   = 0
    uptime_seconds:     float = 0.0
    api_version:        str   = "0.11.0"


@dataclass
class GovernanceMetricsSnapshot:
    """
    Complete governance system health snapshot.
    Produced by MetricsCollector every METRICS_PUSH_INTERVAL_SECONDS.
    Broadcast to all connected WebSocket clients.
    Persisted to PostgreSQL every METRICS_SNAPSHOT_INTERVAL_SECONDS.
    """
    timestamp:     str           = field(
        default_factory=lambda: datetime.now(timezone.utc).isoformat()
    )
    period_hours:  int           = 24
    policy:        PolicyMetrics         = field(default_factory=PolicyMetrics)
    llm:           LLMMetrics            = field(default_factory=LLMMetrics)
    rag:           RAGMetrics            = field(default_factory=RAGMetrics)
    output_scan:   OutputScanMetrics     = field(default_factory=OutputScanMetrics)
    streaming:     StreamingMetrics      = field(default_factory=StreamingMetrics)
    system:        SystemHealthMetrics   = field(default_factory=SystemHealthMetrics)

    def to_json(self) -> str:
        """Serialize to JSON for WebSocket transmission."""
        return json.dumps(asdict(self), default=str)

    def to_dict(self) -> dict:
        return asdict(self)
```

Add `MetricsSnapshot` PostgreSQL model:

```python
# backend/database/models.py  (add MetricsSnapshot)
from sqlalchemy import Column, String, Integer, Float, Boolean, DateTime, JSON

class MetricsSnapshot(Base):
    """
    Hourly point-in-time snapshot of the governance metrics.
    Used for historical analysis and compliance reporting.
    One row per hour. Kept for 90 days by default.
    """
    __tablename__ = "metrics_snapshots"

    snapshot_id    = Column(String(64),  primary_key=True)
    captured_at    = Column(DateTime(timezone=True), nullable=False, index=True)
    period_hours   = Column(Integer,     nullable=False, default=24)

    # Policy
    policy_total         = Column(Integer, default=0)
    policy_blocked       = Column(Integer, default=0)
    policy_block_rate    = Column(Float,   default=0.0)
    policy_avg_risk      = Column(Float,   default=0.0)

    # LLM + Cost
    llm_total_calls      = Column(Integer, default=0)
    llm_cache_hit_rate   = Column(Float,   default=0.0)
    llm_cost_usd         = Column(Float,   default=0.0)
    llm_tokens_total     = Column(Integer, default=0)

    # Output Scan
    scan_total           = Column(Integer, default=0)
    scan_blocked         = Column(Integer, default=0)
    scan_avg_confidence  = Column(Float,   default=0.0)

    # Streaming
    stream_total         = Column(Integer, default=0)
    stream_retraction_rate = Column(Float, default=0.0)

    # Full JSON blob for any field not in columns
    full_snapshot        = Column(JSON, nullable=True)
```

```bash
# Create and run migration
alembic revision --autogenerate -m "add_metrics_snapshots_table"
alembic upgrade head
```

---

## ⚙️ Step 11.3 — MetricsCollector: Aggregating All Sources

```python
# backend/metrics/collector.py
"""
MetricsCollector aggregates data from all sources built in Days 1-10:
  - PostgreSQL: policy_checks, audit_events, output_scan_results,
                indexed_documents, retrieval_events, stream_sessions
  - Redis: cache stats, cost counters, stream stats

Produces a GovernanceMetricsSnapshot in one call.
"""
import logging
import os
import time
from datetime import datetime, timezone, timedelta
from typing import Optional

from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession

from backend.metrics.snapshot import (
    GovernanceMetricsSnapshot, PolicyMetrics, LLMMetrics,
    RAGMetrics, OutputScanMetrics, StreamingMetrics, SystemHealthMetrics,
)
from backend.database.models import (
    PolicyCheck, AuditEvent, OutputScanResult, ScanAction,
    IndexedDocument, DocumentScanStatus, RetrievalEvent, StreamSession,
)
from backend.cache.redis_client import redis_get, get_redis

logger = logging.getLogger(__name__)

_startup_time = time.time()


class MetricsCollector:
    """
    Collects and aggregates governance metrics from all data sources.

    Usage:
        collector = MetricsCollector(db_session=db)
        snapshot  = await collector.collect(period_hours=24)
        # snapshot is a fully populated GovernanceMetricsSnapshot
    """

    def __init__(self, db_session: AsyncSession):
        self.db = db_session

    async def collect(self, period_hours: int = 24) -> GovernanceMetricsSnapshot:
        """
        Collect a full governance metrics snapshot.
        Queries all sources in parallel conceptually; executes sequentially.
        Returns GovernanceMetricsSnapshot.
        """
        since = datetime.now(timezone.utc) - timedelta(hours=period_hours)

        policy      = await self._collect_policy(since)
        llm         = await self._collect_llm(since)
        rag         = await self._collect_rag(since)
        output_scan = await self._collect_output_scan(since)
        streaming   = await self._collect_streaming(since)
        system      = await self._collect_system()

        return GovernanceMetricsSnapshot(
            period_hours = period_hours,
            policy       = policy,
            llm          = llm,
            rag          = rag,
            output_scan  = output_scan,
            streaming    = streaming,
            system       = system,
        )

    # ─── Policy Metrics ────────────────────────────────────────────────────────

    async def _collect_policy(self, since: datetime) -> PolicyMetrics:
        """Aggregate policy check metrics for the period."""
        try:
            result = await self.db.execute(
                select(PolicyCheck).where(PolicyCheck.checked_at >= since)
            )
            checks = result.scalars().all()

            if not checks:
                return PolicyMetrics()

            total    = len(checks)
            blocked  = sum(1 for c in checks if not c.allowed)
            allowed  = total - blocked
            redacted = sum(
                1 for c in checks
                if c.allowed and c.violations and len(c.violations) > 0
            )
            block_rate   = round(blocked / total * 100, 1) if total > 0 else 0.0
            avg_risk     = round(
                sum(c.risk_score for c in checks) / total, 1
            ) if total > 0 else 0.0

            # Top rules fired — count occurrences across all checks
            rule_counts: dict[str, int] = {}
            for check in checks:
                for rule_id in (check.rules_fired or []):
                    rule_counts[rule_id] = rule_counts.get(rule_id, 0) + 1

            top_rules = sorted(
                [{"rule_id": k, "fire_count": v} for k, v in rule_counts.items()],
                key=lambda x: x["fire_count"],
                reverse=True,
            )[:5]

            # Risk score distribution
            risk_dist = {"low": 0, "medium": 0, "high": 0, "critical": 0}
            for c in checks:
                if c.risk_score == 0:
                    risk_dist["low"] += 1
                elif c.risk_score < 40:
                    risk_dist["medium"] += 1
                elif c.risk_score < 70:
                    risk_dist["high"] += 1
                else:
                    risk_dist["critical"] += 1

            return PolicyMetrics(
                total_checks      = total,
                blocked           = blocked,
                allowed           = allowed,
                redacted          = redacted,
                block_rate        = f"{block_rate}%",
                avg_risk_score    = avg_risk,
                top_rules_fired   = top_rules,
                risk_distribution = risk_dist,
            )
        except Exception as e:
            logger.error(f"Policy metrics collection failed: {e}")
            return PolicyMetrics()

    # ─── LLM Metrics ───────────────────────────────────────────────────────────

    async def _collect_llm(self, since: datetime) -> LLMMetrics:
        """Aggregate LLM call, cost, and cache metrics."""
        try:
            result = await self.db.execute(
                select(AuditEvent).where(
                    AuditEvent.action_type == "llm_call",
                    AuditEvent.timestamp   >= since,
                )
            )
            events = result.scalars().all()

            total_calls    = len(events)
            output_blocked = sum(
                1 for e in events if e.outcome == "output_blocked"
            )

            # Token and latency aggregation from details JSON
            tokens_input  = 0
            tokens_output = 0
            latencies     = []
            for e in events:
                d = e.details or {}
                tokens_input  += d.get("input_tokens",  0)
                tokens_output += d.get("output_tokens", 0)
                if d.get("latency_ms"):
                    latencies.append(d["latency_ms"])

            avg_latency = round(sum(latencies) / len(latencies), 0) if latencies else 0.0

            # Cost from Redis (accumulated counters)
            today     = datetime.now(timezone.utc).strftime("%Y-%m-%d")
            r         = await get_redis()
            cost_total = 0.0
            cost_by_model: dict = {}

            if r:
                try:
                    model_keys = await r.keys(f"cost:model:*:{today}")
                    for key in model_keys:
                        model_id   = key.split(":")[2]
                        cost_val   = float(await redis_get(key) or 0.0)
                        cost_total += cost_val
                        cost_by_model[model_id] = round(cost_val, 4)

                    # Cost by user (top 10)
                    user_keys  = await r.keys(f"cost:user:*:{today}")
                    cost_by_user: dict = {}
                    for key in user_keys[:20]:
                        user_id = key.split(":")[2]
                        val     = float(await redis_get(key) or 0.0)
                        cost_by_user[user_id] = round(val, 4)
                except Exception as e:
                    logger.warning(f"Redis cost fetch failed: {e}")

            # Cache stats from Redis
            cache_hits   = int(await redis_get("cache:stats:llm:hits")   or 0)
            cache_misses = int(await redis_get("cache:stats:llm:misses") or 0)
            cache_total  = cache_hits + cache_misses
            hit_rate     = round(cache_hits / cache_total * 100, 1) if cache_total > 0 else 0.0

            return LLMMetrics(
                total_calls     = total_calls,
                cache_hits      = cache_hits,
                cache_misses    = cache_misses,
                cache_hit_rate  = f"{hit_rate}%",
                tokens_input    = tokens_input,
                tokens_output   = tokens_output,
                tokens_total    = tokens_input + tokens_output,
                cost_total_usd  = round(cost_total, 4),
                cost_by_model   = dict(
                    sorted(cost_by_model.items(), key=lambda x: x[1], reverse=True)
                ),
                avg_latency_ms  = avg_latency,
                output_blocked  = output_blocked,
            )
        except Exception as e:
            logger.error(f"LLM metrics collection failed: {e}")
            return LLMMetrics()

    # ─── RAG Metrics ───────────────────────────────────────────────────────────

    async def _collect_rag(self, since: datetime) -> RAGMetrics:
        """Aggregate RAG indexing and retrieval metrics."""
        try:
            # Documents indexed in the period
            doc_result = await self.db.execute(
                select(IndexedDocument).where(
                    IndexedDocument.indexed_at >= since,
                    IndexedDocument.deleted_at == None,
                )
            )
            docs = doc_result.scalars().all()

            total_indexed = len(docs)
            rejected      = sum(
                1 for d in docs
                if d.scan_status == DocumentScanStatus.rejected
            )
            rejection_rate = (
                round(rejected / total_indexed * 100, 1)
                if total_indexed > 0 else 0.0
            )

            by_clearance: dict = {}
            for doc in docs:
                if doc.scan_status == DocumentScanStatus.clean:
                    cl = doc.clearance.value if doc.clearance else "unknown"
                    by_clearance[cl] = by_clearance.get(cl, 0) + 1

            # Retrieval events
            ret_result = await self.db.execute(
                select(RetrievalEvent).where(RetrievalEvent.timestamp >= since)
            )
            retrievals = ret_result.scalars().all()
            total_retrievals = len(retrievals)
            avg_chunks = (
                round(sum(r.chunks_returned for r in retrievals) / total_retrievals, 1)
                if total_retrievals > 0 else 0.0
            )
            cache_from = sum(1 for r in retrievals if r.from_cache)
            rag_hit_rate = (
                round(cache_from / total_retrievals * 100, 1)
                if total_retrievals > 0 else 0.0
            )

            return RAGMetrics(
                documents_indexed  = total_indexed - rejected,
                documents_rejected = rejected,
                rejection_rate     = f"{rejection_rate}%",
                total_retrievals   = total_retrievals,
                cache_hits         = cache_from,
                cache_hit_rate     = f"{rag_hit_rate}%",
                by_clearance       = by_clearance,
                avg_chunks_returned = avg_chunks,
            )
        except Exception as e:
            logger.error(f"RAG metrics collection failed: {e}")
            return RAGMetrics()

    # ─── Output Scan Metrics ───────────────────────────────────────────────────

    async def _collect_output_scan(self, since: datetime) -> OutputScanMetrics:
        """Aggregate output scanning metrics."""
        try:
            result = await self.db.execute(
                select(OutputScanResult).where(OutputScanResult.scanned_at >= since)
            )
            scans = result.scalars().all()

            if not scans:
                return OutputScanMetrics()

            total    = len(scans)
            blocked  = sum(1 for s in scans if s.action == ScanAction.blocked)
            redacted = sum(1 for s in scans if s.action == ScanAction.redacted)
            warned   = sum(1 for s in scans if s.action == ScanAction.warned)
            injected = sum(1 for s in scans if s.action == ScanAction.injected)
            allowed  = sum(1 for s in scans if s.action == ScanAction.allowed)
            pii_det  = sum(1 for s in scans if s.pii_found)
            discl    = sum(1 for s in scans if s.disclaimer_injected)

            block_rate     = round(blocked / total * 100, 1) if total > 0 else 0.0
            avg_toxicity   = round(sum(s.toxicity_score      for s in scans) / total, 4)
            avg_hallucin   = round(sum(s.hallucination_score for s in scans) / total, 4)
            avg_confidence = round(sum(s.confidence_score    for s in scans) / total, 4)

            by_model: dict = {}
            for scan in scans:
                m  = scan.model_id
                bm = by_model.setdefault(m, {"total": 0, "blocked": 0})
                bm["total"]  += 1
                if scan.action == ScanAction.blocked:
                    bm["blocked"] += 1
            for m in by_model:
                t = by_model[m]["total"]
                b = by_model[m]["blocked"]
                by_model[m]["block_rate"] = f"{round(b/t*100,1)}%" if t > 0 else "0.0%"

            return OutputScanMetrics(
                total_scans          = total,
                blocked              = blocked,
                redacted             = redacted,
                warned               = warned,
                injected             = injected,
                allowed              = allowed,
                block_rate           = f"{block_rate}%",
                avg_toxicity_score   = avg_toxicity,
                avg_hallucination    = avg_hallucin,
                avg_confidence       = avg_confidence,
                pii_detections       = pii_det,
                disclaimer_injections = discl,
                by_model             = by_model,
            )
        except Exception as e:
            logger.error(f"Output scan metrics collection failed: {e}")
            return OutputScanMetrics()

    # ─── Streaming Metrics ─────────────────────────────────────────────────────

    async def _collect_streaming(self, since: datetime) -> StreamingMetrics:
        """Aggregate streaming session metrics."""
        try:
            result = await self.db.execute(
                select(StreamSession).where(StreamSession.started_at >= since)
            )
            sessions = result.scalars().all()

            if not sessions:
                return StreamingMetrics()

            total       = len(sessions)
            completed   = sum(1 for s in sessions if s.outcome == "completed")
            retracted   = sum(1 for s in sessions if s.retracted)
            blocked     = sum(1 for s in sessions if s.outcome == "blocked")
            disconnected = sum(1 for s in sessions if s.outcome == "disconnected")

            retraction_rate = round(retracted / total * 100, 2) if total > 0 else 0.0
            avg_duration    = round(
                sum(s.duration_ms for s in sessions) / total, 0
            ) if total > 0 else 0.0
            avg_tokens      = round(
                sum(s.tokens_used for s in sessions) / total, 0
            ) if total > 0 else 0.0

            return StreamingMetrics(
                total_streams   = total,
                completed       = completed,
                retracted       = retracted,
                blocked         = blocked,
                disconnected    = disconnected,
                retraction_rate = f"{retraction_rate}%",
                avg_duration_ms = avg_duration,
                avg_tokens      = avg_tokens,
            )
        except Exception as e:
            logger.error(f"Streaming metrics collection failed: {e}")
            return StreamingMetrics()

    # ─── System Health ─────────────────────────────────────────────────────────

    async def _collect_system(self) -> SystemHealthMetrics:
        """Collect system-level health metrics."""
        redis_connected  = False
        redis_memory_mb  = 0.0
        redis_hit_rate   = "0.0%"
        active_streams   = 0

        try:
            r = await get_redis()
            if r:
                redis_connected = True
                info = await r.info("memory")
                redis_memory_mb = round(
                    int(info.get("used_memory", 0)) / 1024 / 1024, 2
                )
                hits   = int(await redis_get("cache:stats:policy:hits")  or 0)
                misses = int(await redis_get("cache:stats:policy:misses") or 0)
                total  = hits + misses
                if total > 0:
                    redis_hit_rate = f"{round(hits / total * 100, 1)}%"

                # Active streams from Redis slot counters
                stream_keys = await r.keys("stream:active:*")
                active_streams = len(stream_keys)
        except Exception as e:
            logger.warning(f"System health Redis check failed: {e}")

        db_connected = True   # If we got here, DB is working
        uptime = round(time.time() - _startup_time, 1)

        from backend.metrics.ws_manager import ws_manager
        active_ws = ws_manager.connection_count()

        return SystemHealthMetrics(
            redis_connected   = redis_connected,
            redis_memory_mb   = redis_memory_mb,
            redis_hit_rate    = redis_hit_rate,
            db_connected      = db_connected,
            active_ws_clients = active_ws,
            active_streams    = active_streams,
            uptime_seconds    = uptime,
            api_version       = "0.11.0",
        )
```

---

## ⚙️ Step 11.4 — Policy Metrics

The policy metrics endpoint returns a richer view of the policy engine than the snapshot — including time-series breakdowns and per-rule performance.

```python
# backend/metrics/policy_metrics.py
from datetime import datetime, timezone, timedelta
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from backend.database.models import PolicyCheck, PolicyRule


async def get_policy_metrics_detail(
    db:           AsyncSession,
    period_hours: int = 24,
) -> dict:
    """
    Detailed policy metrics with per-rule breakdown and hourly time series.
    Used by the REST endpoint GET /metrics/policy.
    """
    since = datetime.now(timezone.utc) - timedelta(hours=period_hours)

    result  = await db.execute(
        select(PolicyCheck).where(PolicyCheck.checked_at >= since)
    )
    checks  = result.scalars().all()
    total   = len(checks)

    if total == 0:
        return {"total": 0, "message": "No policy checks in this period"}

    blocked  = sum(1 for c in checks if not c.allowed)
    allowed  = total - blocked

    # Per-rule stats
    rule_stats: dict = {}
    for check in checks:
        for rule_id in (check.rules_fired or []):
            if rule_id not in rule_stats:
                rule_stats[rule_id] = {"rule_id": rule_id, "fires": 0,
                                        "led_to_block": 0}
            rule_stats[rule_id]["fires"] += 1
            if not check.allowed:
                rule_stats[rule_id]["led_to_block"] += 1

    # Enrich with rule names from DB
    rules_result = await db.execute(select(PolicyRule))
    rules_by_id  = {r.rule_id: r.name for r in rules_result.scalars().all()}
    for rs in rule_stats.values():
        rs["name"] = rules_by_id.get(rs["rule_id"], rs["rule_id"])

    top_rules = sorted(
        rule_stats.values(),
        key=lambda x: x["fires"],
        reverse=True
    )[:10]

    # Hourly block rate time series (last 24h)
    hourly: dict = {}
    for check in checks:
        if check.checked_at:
            hour = check.checked_at.strftime("%Y-%m-%dT%H:00")
            if hour not in hourly:
                hourly[hour] = {"hour": hour, "total": 0, "blocked": 0}
            hourly[hour]["total"] += 1
            if not check.allowed:
                hourly[hour]["blocked"] += 1

    for h in hourly.values():
        h["block_rate"] = (
            f"{round(h['blocked'] / h['total'] * 100, 1)}%"
            if h["total"] > 0 else "0.0%"
        )

    time_series = sorted(hourly.values(), key=lambda x: x["hour"])

    return {
        "period_hours":  period_hours,
        "total_checks":  total,
        "blocked":       blocked,
        "allowed":       allowed,
        "block_rate":    f"{round(blocked / total * 100, 1)}%",
        "top_rules":     top_rules,
        "hourly_series": time_series,
    }
```

---

## ⚙️ Step 11.5 — LLM and Cost Metrics

```python
# backend/metrics/cost_metrics.py
from datetime import datetime, timezone, timedelta
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from backend.database.models import AuditEvent
from backend.cache.redis_client import get_redis, redis_get


async def get_cost_metrics_detail(
    db:           AsyncSession,
    period_days:  int = 30,
) -> dict:
    """
    Detailed cost metrics with daily breakdown and per-user/model spend.
    Used by the REST endpoint GET /metrics/cost.
    """
    today  = datetime.now(timezone.utc)
    r      = await get_redis()

    # Daily cost series from Redis
    daily_costs = []
    total_usd   = 0.0
    for i in range(period_days):
        day   = (today - timedelta(days=i)).strftime("%Y-%m-%d")
        keys  = await r.keys(f"cost:model:*:{day}") if r else []
        day_cost = 0.0
        for key in keys:
            day_cost += float(await redis_get(key) or 0.0)
        daily_costs.append({"date": day, "cost_usd": round(day_cost, 4)})
        total_usd += day_cost

    daily_costs = sorted(daily_costs, key=lambda x: x["date"])

    # By model (today)
    today_str   = today.strftime("%Y-%m-%d")
    cost_by_model: dict = {}
    if r:
        model_keys = await r.keys(f"cost:model:*:{today_str}")
        for key in model_keys:
            model_id = key.split(":")[2]
            cost_by_model[model_id] = round(float(await redis_get(key) or 0.0), 4)

    # By user (today, top 20)
    cost_by_user: dict = {}
    if r:
        user_keys = await r.keys(f"cost:user:*:{today_str}")
        for key in user_keys[:20]:
            user_id = key.split(":")[2]
            cost_by_user[user_id] = round(float(await redis_get(key) or 0.0), 4)

    # LLM call count from audit events (last 24h)
    since  = today - timedelta(hours=24)
    result = await db.execute(
        select(AuditEvent).where(
            AuditEvent.action_type == "llm_call",
            AuditEvent.timestamp   >= since,
        )
    )
    events      = result.scalars().all()
    calls_24h   = len(events)
    tokens_24h  = sum((e.details or {}).get("total_tokens", 0) for e in events)
    cost_24h    = sum((e.details or {}).get("cost_usd", 0.0) for e in events)

    return {
        "period_days":     period_days,
        "total_usd":       round(total_usd, 4),
        "today_usd":       round(
            sum(d["cost_usd"] for d in daily_costs[-1:]), 4
        ),
        "last_24h": {
            "calls":    calls_24h,
            "tokens":   tokens_24h,
            "cost_usd": round(cost_24h, 4),
        },
        "by_model":        dict(
            sorted(cost_by_model.items(), key=lambda x: x[1], reverse=True)
        ),
        "by_user":         dict(
            sorted(cost_by_user.items(), key=lambda x: x[1], reverse=True)
        ),
        "daily_series":    daily_costs,
    }
```

---

## ⚙️ Step 11.6 — Output Scan and Streaming Metrics

```python
# backend/metrics/scan_metrics.py
from datetime import datetime, timezone, timedelta
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from backend.database.models import OutputScanResult, ScanAction, StreamSession


async def get_scan_metrics_detail(db: AsyncSession, period_hours: int = 24) -> dict:
    """
    Detailed output scan metrics including per-model toxicity trends.
    Used by GET /metrics/output.
    """
    since  = datetime.now(timezone.utc) - timedelta(hours=period_hours)
    result = await db.execute(
        select(OutputScanResult).where(OutputScanResult.scanned_at >= since)
    )
    scans  = result.scalars().all()
    total  = len(scans)

    if total == 0:
        return {"total": 0, "message": "No output scans in this period"}

    # Hourly toxicity trend
    hourly: dict = {}
    for scan in scans:
        if scan.scanned_at:
            hour = scan.scanned_at.strftime("%Y-%m-%dT%H:00")
            if hour not in hourly:
                hourly[hour] = {
                    "hour": hour, "scans": 0, "blocked": 0,
                    "toxicity_sum": 0.0, "confidence_sum": 0.0,
                }
            h  = hourly[hour]
            h["scans"]          += 1
            h["toxicity_sum"]   += scan.toxicity_score
            h["confidence_sum"] += scan.confidence_score
            if scan.action == ScanAction.blocked:
                h["blocked"] += 1

    for h in hourly.values():
        n = h["scans"]
        h["avg_toxicity"]   = round(h.pop("toxicity_sum")   / n, 4) if n > 0 else 0.0
        h["avg_confidence"] = round(h.pop("confidence_sum") / n, 4) if n > 0 else 0.0
        h["block_rate"]     = f"{round(h['blocked']/n*100,1)}%" if n > 0 else "0.0%"

    return {
        "period_hours":  period_hours,
        "total_scans":   total,
        "blocked":       sum(1 for s in scans if s.action == ScanAction.blocked),
        "redacted":      sum(1 for s in scans if s.action == ScanAction.redacted),
        "pii_found":     sum(1 for s in scans if s.pii_found),
        "disclaimers":   sum(1 for s in scans if s.disclaimer_injected),
        "avg_toxicity":  round(sum(s.toxicity_score for s in scans) / total, 4),
        "avg_confidence": round(sum(s.confidence_score for s in scans) / total, 4),
        "hourly_series": sorted(hourly.values(), key=lambda x: x["hour"]),
    }
```

---

## ⚙️ Step 11.7 — WebSocket Connection Manager

```python
# backend/metrics/ws_manager.py
"""
WebSocketConnectionManager: manages all connected dashboard clients.
Thread-safe broadcast to all active connections.
Handles connect, disconnect, and dead connection cleanup.
"""
import asyncio
import logging
from fastapi import WebSocket

logger = logging.getLogger(__name__)


class WebSocketConnectionManager:
    """
    Manages active WebSocket connections for the metrics dashboard.

    All connected clients receive the same broadcast every 5 seconds.
    Dead connections are silently removed on the next broadcast attempt.

    Usage:
        manager = WebSocketConnectionManager()
        await manager.connect(websocket)
        await manager.broadcast(json_string)
        manager.disconnect(websocket)
    """

    def __init__(self):
        self._active: list[WebSocket] = []
        self._lock = asyncio.Lock()

    async def connect(self, websocket: WebSocket) -> bool:
        """
        Accept a new WebSocket connection.
        Returns False if max connections reached.
        """
        import os
        max_conn = int(os.getenv("WS_MAX_CONNECTIONS", "50"))

        async with self._lock:
            if len(self._active) >= max_conn:
                await websocket.close(
                    code=1008,
                    reason=f"Max connections ({max_conn}) reached"
                )
                return False
            await websocket.accept()
            self._active.append(websocket)
            logger.info(
                f"WebSocket client connected. "
                f"Total active: {len(self._active)}"
            )
            return True

    def disconnect(self, websocket: WebSocket):
        """Remove a disconnected client."""
        if websocket in self._active:
            self._active.remove(websocket)
            logger.info(
                f"WebSocket client disconnected. "
                f"Total active: {len(self._active)}"
            )

    async def broadcast(self, message: str):
        """
        Broadcast a JSON string to all connected clients.
        Dead connections are removed silently.
        """
        if not self._active:
            return

        dead: list[WebSocket] = []

        async with self._lock:
            for ws in list(self._active):
                try:
                    await ws.send_text(message)
                except Exception:
                    # Connection is dead — mark for removal
                    dead.append(ws)

            for ws in dead:
                if ws in self._active:
                    self._active.remove(ws)

        if dead:
            logger.info(
                f"Removed {len(dead)} dead WebSocket connection(s). "
                f"Active: {len(self._active)}"
            )

    def connection_count(self) -> int:
        return len(self._active)

    async def send_personal(self, websocket: WebSocket, message: str):
        """Send a message to a single client only."""
        try:
            await websocket.send_text(message)
        except Exception:
            self.disconnect(websocket)


# ─── Singleton instance ────────────────────────────────────────────────────────
ws_manager = WebSocketConnectionManager()
```

---

## ⚙️ Step 11.8 — WebSocket Endpoint: Live Push Every 5 Seconds

```python
# backend/routers/metrics.py
import asyncio
import json
import logging
import os
import uuid
from datetime import datetime, timezone, timedelta

from fastapi import APIRouter, WebSocket, WebSocketDisconnect, Depends, Request
from sqlalchemy.ext.asyncio import AsyncSession

from backend.database.db import get_db, AsyncSessionLocal
from backend.metrics.collector import MetricsCollector
from backend.metrics.ws_manager import ws_manager
from backend.metrics.policy_metrics import get_policy_metrics_detail
from backend.metrics.cost_metrics import get_cost_metrics_detail
from backend.metrics.scan_metrics import get_scan_metrics_detail
from backend.database.models import MetricsSnapshot

router  = APIRouter()
logger  = logging.getLogger(__name__)

PUSH_INTERVAL = int(os.getenv("METRICS_PUSH_INTERVAL_SECONDS", "5"))
PERIOD_HOURS  = int(os.getenv("METRICS_DEFAULT_PERIOD_HOURS",  "24"))


# ─── WebSocket Endpoint ────────────────────────────────────────────────────────

@router.websocket("/ws/metrics")
async def metrics_websocket(websocket: WebSocket):
    """
    Live governance metrics WebSocket endpoint.

    Clients connect here to receive the full GovernanceMetricsSnapshot
    pushed every METRICS_PUSH_INTERVAL_SECONDS (default: 5).

    Protocol:
      Client → Server: any message (treated as a ping/keep-alive)
      Server → Client: GovernanceMetricsSnapshot JSON every 5 seconds

    Client disconnects are handled gracefully — the server does not crash.

    Usage (JavaScript):
        const ws = new WebSocket('ws://localhost:8000/metrics/ws/metrics');
        ws.onmessage = (event) => {
            const snapshot = JSON.parse(event.data);
            updateDashboard(snapshot);
        };
    """
    connected = await ws_manager.connect(websocket)
    if not connected:
        return

    try:
        # Send an immediate snapshot on connect (don't make the client wait 5s)
        async with AsyncSessionLocal() as db:
            collector = MetricsCollector(db_session=db)
            snapshot  = await collector.collect(period_hours=PERIOD_HOURS)
            await ws_manager.send_personal(websocket, snapshot.to_json())

        # Stream loop — push every PUSH_INTERVAL seconds
        while True:
            # Sleep first, then push (already sent one on connect)
            await asyncio.sleep(PUSH_INTERVAL)

            async with AsyncSessionLocal() as db:
                collector = MetricsCollector(db_session=db)
                snapshot  = await collector.collect(period_hours=PERIOD_HOURS)

            await ws_manager.send_personal(websocket, snapshot.to_json())

    except WebSocketDisconnect:
        ws_manager.disconnect(websocket)
        logger.info("WebSocket client disconnected normally.")
    except Exception as e:
        logger.error(f"WebSocket error: {e}")
        ws_manager.disconnect(websocket)


# ─── Background Snapshot Writer ───────────────────────────────────────────────

async def write_hourly_snapshot():
    """
    Background task: collect a full snapshot and write it to PostgreSQL.
    Called by the APScheduler job configured in main.py.
    Creates a historical record for compliance reporting.
    """
    try:
        async with AsyncSessionLocal() as db:
            collector = MetricsCollector(db_session=db)
            snapshot  = await collector.collect(period_hours=24)
            data      = snapshot.to_dict()

            record = MetricsSnapshot(
                snapshot_id         = str(uuid.uuid4()),
                captured_at         = datetime.now(timezone.utc),
                period_hours        = 24,
                policy_total        = data["policy"]["total_checks"],
                policy_blocked      = data["policy"]["blocked"],
                policy_block_rate   = float(data["policy"]["block_rate"].rstrip("%")),
                policy_avg_risk     = data["policy"]["avg_risk_score"],
                llm_total_calls     = data["llm"]["total_calls"],
                llm_cache_hit_rate  = float(data["llm"]["cache_hit_rate"].rstrip("%")),
                llm_cost_usd        = data["llm"]["cost_total_usd"],
                llm_tokens_total    = data["llm"]["tokens_total"],
                scan_total          = data["output_scan"]["total_scans"],
                scan_blocked        = data["output_scan"]["blocked"],
                scan_avg_confidence = data["output_scan"]["avg_confidence"],
                stream_total        = data["streaming"]["total_streams"],
                stream_retraction_rate = float(
                    data["streaming"]["retraction_rate"].rstrip("%")
                ),
                full_snapshot       = data,
            )
            db.add(record)
            await db.commit()
            logger.info(f"Hourly metrics snapshot written: {record.snapshot_id}")
    except Exception as e:
        logger.error(f"Hourly snapshot write failed: {e}")


# ─── REST Snapshot Endpoints ───────────────────────────────────────────────────

@router.get("/snapshot", summary="Full governance metrics snapshot right now")
async def get_snapshot(
    period_hours: int = 24,
    db: AsyncSession = Depends(get_db),
):
    """
    Returns the current GovernanceMetricsSnapshot as JSON.
    Equivalent to what connected WebSocket clients receive every 5 seconds.

    Use this for:
      - REST-based dashboards that don't use WebSocket
      - Scripted compliance reports
      - Alerting integrations that poll on a schedule
    """
    collector = MetricsCollector(db_session=db)
    snapshot  = await collector.collect(period_hours=period_hours)
    return snapshot.to_dict()


@router.get("/policy", summary="Detailed policy metrics with hourly time series")
async def get_policy_metrics(
    period_hours: int = 24,
    db: AsyncSession = Depends(get_db),
):
    """
    Policy engine metrics: block rate, top rules, hourly block rate series.
    Useful for understanding what prompts are being blocked and why.
    """
    return await get_policy_metrics_detail(db, period_hours)


@router.get("/cost", summary="LLM cost breakdown by model, user, and daily series")
async def get_cost_metrics(
    period_days: int = 30,
    db: AsyncSession = Depends(get_db),
):
    """
    AI spend metrics: total cost, by model, by user, daily series.
    Finance and engineering leadership use this for cost governance.
    """
    return await get_cost_metrics_detail(db, period_days)


@router.get("/output", summary="Output scan metrics: toxicity, hallucination, PII")
async def get_output_metrics(
    period_hours: int = 24,
    db: AsyncSession = Depends(get_db),
):
    """
    Output scanning metrics: toxicity block rate, hallucination trends,
    PII detections, disclaimer injections.
    """
    return await get_scan_metrics_detail(db, period_hours)


@router.get("/health", summary="System health: Redis, DB, active connections")
async def get_system_health(db: AsyncSession = Depends(get_db)):
    """
    System health at a glance: Redis status, DB connection, active WebSocket
    clients, active streams, uptime.
    """
    collector = MetricsCollector(db_session=db)
    health    = await collector._collect_system()
    from dataclasses import asdict
    return {
        **asdict(health),
        "ws_clients":    ws_manager.connection_count(),
        "ws_endpoint":   "ws://localhost:8000/metrics/ws/metrics",
    }


@router.get("/history", summary="Historical snapshots for compliance reporting")
async def get_metrics_history(
    days:  int = 7,
    db:    AsyncSession = Depends(get_db),
):
    """
    Returns hourly governance snapshots for the last N days.
    Used for compliance reporting: "How did the system behave last week?"
    """
    since  = datetime.now(timezone.utc) - timedelta(days=days)
    result = await db.execute(
        select(MetricsSnapshot)
        .where(MetricsSnapshot.captured_at >= since)
        .order_by(MetricsSnapshot.captured_at.desc())
        .limit(days * 24)
    )
    snapshots = result.scalars().all()
    return [
        {
            "snapshot_id":         s.snapshot_id,
            "captured_at":         s.captured_at.isoformat(),
            "policy_block_rate":   f"{s.policy_block_rate}%",
            "llm_cost_usd":        s.llm_cost_usd,
            "scan_avg_confidence": s.scan_avg_confidence,
            "stream_retraction_rate": f"{s.stream_retraction_rate}%",
        }
        for s in snapshots
    ]
```

---

## ⚙️ Step 11.9 — REST Snapshot Endpoints

Update `main.py` to register the router and the hourly background task:

```python
# backend/main.py  (add metrics router and background scheduler)
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from backend.routers import metrics as metrics_router
from backend.routers.metrics import write_hourly_snapshot

SNAPSHOT_INTERVAL = int(os.getenv("METRICS_SNAPSHOT_INTERVAL_SECONDS", "3600"))

_scheduler = AsyncIOScheduler()

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ... existing startup code ...

    # Start the hourly metrics snapshot writer
    _scheduler.add_job(
        write_hourly_snapshot,
        trigger   = "interval",
        seconds   = SNAPSHOT_INTERVAL,
        id        = "metrics_snapshot_writer",
        max_instances = 1,
    )
    _scheduler.start()
    print("✅ Metrics snapshot scheduler started (hourly).")

    yield

    # ... existing shutdown code ...
    _scheduler.shutdown(wait=False)
    print("✅ Metrics scheduler stopped.")


# Add router
app.include_router(
    metrics_router.router,
    prefix = "/metrics",
    tags   = ["Governance Dashboard"],
)
```

Updated project structure:

```
ai-governance/
├── backend/
│   ├── metrics/                    ← NEW today
│   │   ├── __init__.py
│   │   ├── snapshot.py             ← GovernanceMetricsSnapshot dataclass
│   │   ├── collector.py            ← MetricsCollector (all sources)
│   │   ├── ws_manager.py           ← WebSocketConnectionManager
│   │   ├── policy_metrics.py       ← Policy detail + time series
│   │   ├── cost_metrics.py         ← Cost breakdown + daily series
│   │   └── scan_metrics.py         ← Output scan detail + trends
│   └── routers/
│       └── metrics.py              ← WebSocket + REST endpoints
```

### Manual Testing with cURL and wscat

```bash
# ── Install wscat (WebSocket command-line client) ────────────────────────────
npm install -g wscat

# ── Test 1: Connect to live metrics WebSocket ────────────────────────────────
wscat -c ws://localhost:8000/metrics/ws/metrics
# You'll receive a full GovernanceMetricsSnapshot JSON every 5 seconds.
# Output example:
# < {"timestamp":"2026-05-29T14:30:02Z","period_hours":24,"policy":{"total_checks":1847,...}}
# < {"timestamp":"2026-05-29T14:30:07Z","period_hours":24,"policy":{"total_checks":1848,...}}

# ── Test 2: REST snapshot ─────────────────────────────────────────────────────
curl http://localhost:8000/metrics/snapshot | python -m json.tool

# ── Test 3: Policy metrics with time range ───────────────────────────────────
curl "http://localhost:8000/metrics/policy?period_hours=48" | python -m json.tool

# ── Test 4: Cost metrics ──────────────────────────────────────────────────────
curl "http://localhost:8000/metrics/cost?period_days=7" | python -m json.tool

# ── Test 5: System health ─────────────────────────────────────────────────────
curl http://localhost:8000/metrics/health | python -m json.tool

# ── Test 6: Historical snapshots ─────────────────────────────────────────────
curl "http://localhost:8000/metrics/history?days=3" | python -m json.tool

# ── Test 7: Count active WebSocket connections ───────────────────────────────
# Open 3 wscat connections in separate terminals, then:
curl http://localhost:8000/metrics/health
# "active_ws_clients": 3

# ── Test 8: Verify hourly snapshot was written to DB ─────────────────────────
# After waiting 1 hour (or triggering manually):
curl "http://localhost:8000/metrics/history?days=1"
# Should show at least 1 snapshot with captured_at timestamp
```

---

## ⚙️ Step 11.10 — Testing Metrics and WebSocket Layer

```python
# tests/unit/test_metrics.py
import pytest
import json
from unittest.mock import AsyncMock, patch, MagicMock
from dataclasses import asdict

from backend.metrics.snapshot import (
    GovernanceMetricsSnapshot, PolicyMetrics, LLMMetrics,
    RAGMetrics, OutputScanMetrics, StreamingMetrics, SystemHealthMetrics,
)


# ─── Snapshot Serialization Tests ─────────────────────────────────────────────

@pytest.mark.unit
class TestGovernanceMetricsSnapshot:

    def test_snapshot_serializes_to_valid_json(self):
        """GovernanceMetricsSnapshot.to_json() produces valid JSON."""
        snapshot = GovernanceMetricsSnapshot(
            policy     = PolicyMetrics(total_checks=100, blocked=12, block_rate="12.0%"),
            llm        = LLMMetrics(total_calls=88, cost_total_usd=4.20),
            rag        = RAGMetrics(documents_indexed=15),
            output_scan = OutputScanMetrics(total_scans=88, avg_confidence=0.87),
            streaming  = StreamingMetrics(total_streams=20, retraction_rate="5.0%"),
            system     = SystemHealthMetrics(redis_connected=True),
        )
        json_str = snapshot.to_json()
        parsed   = json.loads(json_str)

        assert parsed["policy"]["total_checks"] == 100
        assert parsed["policy"]["blocked"]      == 12
        assert parsed["llm"]["cost_total_usd"]  == 4.20
        assert parsed["system"]["redis_connected"] == True

    def test_snapshot_to_dict_has_all_sections(self):
        """Snapshot dict has all 6 sections."""
        snapshot = GovernanceMetricsSnapshot()
        data     = snapshot.to_dict()
        for section in ["policy", "llm", "rag", "output_scan", "streaming", "system"]:
            assert section in data, f"Missing section: {section}"

    def test_default_snapshot_has_zero_values(self):
        """A freshly created snapshot has zeroes everywhere."""
        snapshot = GovernanceMetricsSnapshot()
        assert snapshot.policy.total_checks  == 0
        assert snapshot.llm.total_calls      == 0
        assert snapshot.output_scan.blocked  == 0
        assert snapshot.streaming.retracted  == 0

    def test_snapshot_timestamp_is_iso_format(self):
        """Snapshot timestamp is a valid ISO 8601 string."""
        from datetime import datetime
        snapshot = GovernanceMetricsSnapshot()
        dt = datetime.fromisoformat(snapshot.timestamp.replace("Z", "+00:00"))
        assert dt is not None


# ─── MetricsCollector Tests ────────────────────────────────────────────────────

@pytest.mark.unit
class TestMetricsCollector:

    async def test_collect_returns_snapshot(self, client, db_session):
        """MetricsCollector.collect() returns a GovernanceMetricsSnapshot."""
        from backend.metrics.collector import MetricsCollector
        collector = MetricsCollector(db_session=db_session)

        with patch.object(collector, "_collect_system") as mock_sys:
            mock_sys.return_value = SystemHealthMetrics(redis_connected=True)
            snapshot = await collector.collect(period_hours=24)

        assert isinstance(snapshot, GovernanceMetricsSnapshot)
        assert snapshot.period_hours == 24

    async def test_collector_handles_empty_db_gracefully(self, db_session):
        """With no data in DB, collector returns zeroes without crashing."""
        from backend.metrics.collector import MetricsCollector
        collector = MetricsCollector(db_session=db_session)
        snapshot  = await collector.collect(period_hours=24)

        assert snapshot.policy.total_checks  == 0
        assert snapshot.llm.total_calls      == 0
        assert snapshot.rag.total_retrievals == 0

    async def test_policy_metrics_counts_blocked_correctly(
        self, client, db_session, e2e_approved_model
    ):
        """Policy metrics correctly counts blocked vs allowed checks."""
        from backend.metrics.collector import MetricsCollector

        # Create some policy checks
        for i in range(5):
            await client.post(
                "/policies/check",
                json={
                    "prompt":     "Safe question about vacation policy.",
                    "user_role":  "employee",
                    "model_id":   "gpt-4o-e2e-test",
                    "user_id":    f"user-{i}",
                    "session_id": f"s-{i}",
                },
                headers={"X-User-ID": f"user-{i}", "X-User-Role": "employee"},
            )

        # Create a blocked check
        await client.post(
            "/policies/check",
            json={
                "prompt":     "SSN: 123-45-6789 email: x@y.com",
                "user_role":  "employee",
                "model_id":   "gpt-4o-e2e-test",
                "user_id":    "blocked-user",
                "session_id": "s-block",
            },
            headers={"X-User-ID": "blocked-user", "X-User-Role": "employee"},
        )

        collector = MetricsCollector(db_session=db_session)
        snapshot  = await collector.collect(period_hours=1)

        assert snapshot.policy.total_checks >= 6
        assert snapshot.policy.blocked      >= 1
        assert snapshot.policy.allowed      >= 5


# ─── WebSocket Connection Manager Tests ───────────────────────────────────────

@pytest.mark.unit
class TestWebSocketConnectionManager:

    async def test_connect_and_disconnect(self):
        from backend.metrics.ws_manager import WebSocketConnectionManager
        manager = WebSocketConnectionManager()

        mock_ws = AsyncMock()
        connected = await manager.connect(mock_ws)
        assert connected             == True
        assert manager.connection_count() == 1

        manager.disconnect(mock_ws)
        assert manager.connection_count() == 0

    async def test_broadcast_sends_to_all_clients(self):
        from backend.metrics.ws_manager import WebSocketConnectionManager
        manager = WebSocketConnectionManager()

        ws1 = AsyncMock()
        ws2 = AsyncMock()
        ws3 = AsyncMock()

        for ws in [ws1, ws2, ws3]:
            await manager.connect(ws)

        await manager.broadcast('{"test": true}')

        ws1.send_text.assert_called_once_with('{"test": true}')
        ws2.send_text.assert_called_once_with('{"test": true}')
        ws3.send_text.assert_called_once_with('{"test": true}')

    async def test_dead_connection_removed_on_broadcast(self):
        from backend.metrics.ws_manager import WebSocketConnectionManager
        manager = WebSocketConnectionManager()

        live_ws = AsyncMock()
        dead_ws = AsyncMock()
        dead_ws.send_text.side_effect = Exception("Connection closed")

        await manager.connect(live_ws)
        await manager.connect(dead_ws)
        assert manager.connection_count() == 2

        await manager.broadcast('{"data": 1}')

        # Dead connection should be removed
        assert manager.connection_count() == 1
        live_ws.send_text.assert_called_once()

    async def test_max_connections_respected(self):
        import os
        from backend.metrics.ws_manager import WebSocketConnectionManager

        os.environ["WS_MAX_CONNECTIONS"] = "2"
        manager = WebSocketConnectionManager()

        ws1 = AsyncMock()
        ws2 = AsyncMock()
        ws3 = AsyncMock()  # This one should be rejected

        await manager.connect(ws1)
        await manager.connect(ws2)
        result = await manager.connect(ws3)

        assert result == False
        ws3.close.assert_called_once()
        assert manager.connection_count() == 2

        del os.environ["WS_MAX_CONNECTIONS"]


# ─── REST Metrics Endpoint Tests ───────────────────────────────────────────────

@pytest.mark.unit
class TestMetricsEndpoints:

    async def test_snapshot_endpoint_returns_200(self, client):
        response = await client.get("/metrics/snapshot")
        assert response.status_code == 200
        data = response.json()
        for section in ["policy", "llm", "rag", "output_scan", "streaming", "system"]:
            assert section in data

    async def test_health_endpoint_returns_connection_info(self, client):
        response = await client.get("/metrics/health")
        assert response.status_code == 200
        data = response.json()
        assert "redis_connected"    in data
        assert "db_connected"       in data
        assert "active_ws_clients"  in data
        assert "ws_endpoint"        in data

    async def test_cost_endpoint_returns_by_model(self, client):
        response = await client.get("/metrics/cost?period_days=1")
        assert response.status_code == 200
        data = response.json()
        assert "total_usd"    in data
        assert "by_model"     in data
        assert "daily_series" in data

    async def test_policy_endpoint_returns_top_rules(self, client):
        response = await client.get("/metrics/policy?period_hours=24")
        assert response.status_code == 200
        data = response.json()
        assert "total_checks"  in data
        assert "block_rate"    in data

    async def test_history_endpoint_returns_list(self, client):
        response = await client.get("/metrics/history?days=1")
        assert response.status_code == 200
        assert isinstance(response.json(), list)

    async def test_output_metrics_returns_scan_data(self, client):
        response = await client.get("/metrics/output?period_hours=24")
        assert response.status_code == 200
```

---

## 🏁 What You Built Today

```
Week 3 Progress:
  Day 11 → Dashboard: real-time governance metrics  ← TODAY
  Day 12 → Multi-tenant isolation: namespace by org_id
  Day 13 → Alerting: PagerDuty/Slack on threshold breaches

New capabilities added:
  ✅ GovernanceMetricsSnapshot — fully typed 6-section dataclass
  ✅ MetricsCollector — aggregates data from all 6 DB tables + Redis
  ✅ Policy metrics with top rules, risk distribution, hourly time series
  ✅ LLM metrics with cost by model/user and cache hit tracking
  ✅ RAG metrics with indexing rates and clearance breakdown
  ✅ Output scan metrics with toxicity/hallucination/PII trends
  ✅ Streaming metrics with retraction rate tracking
  ✅ System health metrics with Redis memory and uptime
  ✅ WebSocketConnectionManager — broadcast to 50 simultaneous clients
  ✅ WS /metrics/ws/metrics — live push every 5 seconds
  ✅ Background APScheduler task — hourly PostgreSQL snapshot
  ✅ MetricsSnapshot PostgreSQL model — 90-day governance history
  ✅ GET /metrics/snapshot  — full snapshot REST endpoint
  ✅ GET /metrics/policy    — policy detail + hourly time series
  ✅ GET /metrics/cost      — cost breakdown + 30-day daily series
  ✅ GET /metrics/output    — output scan trends + hourly series
  ✅ GET /metrics/health    — system health at a glance
  ✅ GET /metrics/history   — historical snapshots for compliance
  ✅ 12+ tests: snapshot, collector, WebSocket manager, REST endpoints

```

### The Complete Governance Stack After Day 11

```
ALL GOVERNANCE DATA NOW VISIBLE IN REAL TIME

  Day 1-10: Build data                Day 11: See data
  ────────────────────────────────────────────────────────
  PolicyCheck → DB          →   policy block rate (live)
  AuditEvent  → DB          →   LLM call rate (live)
  OutputScanResult → DB     →   toxicity/hallucination (live)
  RetrievalEvent → DB       →   RAG cache hit rate (live)
  StreamSession → DB        →   retraction rate (live)
  cost:* → Redis            →   spend by model/user (live)
  cache:stats:* → Redis     →   cache performance (live)
                     │
                     ▼
           MetricsCollector.collect()
                     │
                     ▼
        GovernanceMetricsSnapshot (JSON)
                     │
            ┌────────┴────────┐
            ▼                 ▼
      WebSocket push      REST snapshot
      every 5 seconds     on demand
      to all clients
            │
     Dashboard client
     (compliance officer
      opens Monday morning)

  MetricsSnapshot → PostgreSQL (every hour)
  90 days of hourly governance history.
  Auditor asks about March 15?
  SELECT * FROM metrics_snapshots WHERE DATE(captured_at) = '2026-03-15';
  Done in 1 second.

```

---

## ✅ Day 11 Checklist

Before moving to Day 12, verify all of these:

- [ ] `pip install websockets apscheduler` succeeded
- [ ] `alembic upgrade head` created `metrics_snapshots` table
- [ ] `GET /metrics/snapshot` returns valid JSON with all 6 sections
- [ ] `GET /metrics/health` shows `redis_connected: true`
- [ ] `wscat -c ws://localhost:8000/metrics/ws/metrics` connects successfully
- [ ] After connecting with wscat, a full snapshot JSON arrives within 1 second
- [ ] Subsequent pushes arrive every ~5 seconds (check with `date` in wscat)
- [ ] Open 3 wscat connections simultaneously — all 3 receive broadcasts
- [ ] `GET /metrics/health` shows `active_ws_clients: 3` while 3 are connected
- [ ] `GET /metrics/policy?period_hours=24` returns `top_rules` list and `hourly_series`
- [ ] `GET /metrics/cost?period_days=7` returns `by_model` and `daily_series`
- [ ] `GET /metrics/output?period_hours=24` returns toxicity and confidence averages
- [ ] `GET /metrics/history?days=1` returns a list (may be empty if < 1 hour uptime)
- [ ] APScheduler log shows "Metrics snapshot written" after 1 hour OR trigger manually
- [ ] Dead WebSocket connection is silently removed (test: kill wscat mid-stream, check logs)
- [ ] All existing tests still pass: `pytest -m unit` → 0 failures
- [ ] New metrics tests pass: `pytest tests/unit/test_metrics.py -v`
- [ ] Coverage stays above 90%: `pytest --cov=backend --cov-fail-under=90`

---

## 📚 What to Read Tonight

| Resource | Topic |
|---|---|
| [FastAPI WebSockets Guide](https://fastapi.tiangolo.com/advanced/websockets/) | WebSocket lifecycle in FastAPI |
| [APScheduler Docs](https://apscheduler.readthedocs.io/en/stable/) | Background jobs with AsyncIOScheduler |
| [wscat GitHub](https://github.com/websockets/wscat) | WebSocket command-line testing |
| [NIST AI RMF — Measure 2.9](https://airc.nist.gov/Docs/1) | Monitoring and evaluating AI performance |
| [Google SRE — Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/) | Latency, traffic, errors, saturation — the 4 metrics that matter |
| [EU AI Act — Article 72](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A52021PC0206) | Post-market monitoring obligations for high-risk AI |

---

## 🔮 Coming Up

```
Day 12 → Multi-tenant isolation: namespace all governance data by org_id
Day 13 → Alerting: fire PagerDuty/Slack on governance threshold breaches
Day 14 → Model versioning and canary deployments with governance
```

> Day 12 is the multi-tenancy day.
> Right now the governance system treats all users as belonging to one
> organization. Day 12 adds org_id to every model: policy checks,
> audit events, RAG documents, output scans — all namespaced by org.
>
> Tenant A's block rate does not appear in Tenant B's dashboard.
> Tenant A's documents are not retrievable by Tenant B's users.
> Tenant A's cost records are invisible to Tenant B's admins.
>
> This is the architecture change that makes the system sellable
> as a B2B product rather than a single-tenant internal tool.

---

*Day 11 complete. The governance system can now see itself.*

*Every policy block, every toxicity score, every LLM cost, every retracted stream — all of it flows into a single real-time snapshot, broadcast via WebSocket to every connected dashboard client every 5 seconds.*

*A compliance officer opens the dashboard on Monday morning. The block rate from the weekend is right there. The top rule that fired 847 times is right there. The cost breakdown by model is right there. The question "how did the AI behave?" now has a real-time answer.*

---

**Repository:** [GuntruTirupathamma/AI-Governance](https://github.com/GuntruTirupathamma/AI-Governance)  **Series:** AI Governance Engineering from Scratch  **Next:** `DAY12.md` → Multi-Tenant Isolation: Namespace All Data by org_id
