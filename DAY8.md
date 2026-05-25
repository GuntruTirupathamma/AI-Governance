# 🔬 DAY 8 — Output Scanning: Hallucination + Toxicity Detection

> **Series:** AI Governance Engineering — Zero to Production  **Author:** GuntruTirupathamma  **Day:** 8 of 35  **Previous:** [DAY7.md](https://github.com/GuntruTirupathamma/AI-Governance/blob/main/DAY7.md) — GovernedRAG: Governed Document Indexing + Retrieval  **Next:** DAY9.md — Governed Streaming: SSE Endpoint with Governance Middleware

---

## 📌 Table of Contents

1. [Why Output Scanning Is a Governance Requirement](#why-output-scanning)
2. [What You'll Build Today](#what-youll-build)
3. [The Output Scanning Architecture](#output-architecture)
4. [Step 8.1 — Install Dependencies](#step-81)
5. [Step 8.2 — OutputScanResult Database Model](#step-82)
6. [Step 8.3 — Toxicity Scanner](#step-83)
7. [Step 8.4 — Hallucination Detector](#step-84)
8. [Step 8.5 — PII Output Scanner](#step-85)
9. [Step 8.6 — The OutputScanner Orchestrator](#step-86)
10. [Step 8.7 — Integration into GovernedLLMClient](#step-87)
11. [Step 8.8 — REST Endpoints: Scan and Stats](#step-88)
12. [Step 8.9 — Disclaimer Injection Engine](#step-89)
13. [Step 8.10 — Testing the Output Scanning Layer](#step-810)
14. [What You Built Today](#what-you-built)
15. [Day 8 Checklist](#checklist)

---

## ⚡ Why Output Scanning Is a Governance Requirement

Days 1–7 built a governance layer that controls what goes **into** the LLM. Today you govern what comes **out**.

The assumption that "if the input is safe, the output is safe" is one of the most dangerous ideas in AI engineering.

### The Output Failures That Governance Must Catch

```
Scenario 1 — The Confident Hallucination
  A user asks the RAG system: "What is the penalty clause in our vendor contract?"
  The retrieved context contains: "Section 12: Penalty clause — see appendix C."
  The LLM responds: "The penalty clause states: vendor owes 15% of contract value
  per day of delay, capped at 90 days."
  The real contract says: "vendor owes 5% of contract value per month."

  Without output scanning: legal team acts on fabricated numbers.
  With hallucination detection: the output is flagged as inconsistent with
  source context. Confidence score: 0.12. User sees a warning.

  The Microsoft Bing Chat incident: Bing fabricated a Gap earnings report
  and users shared it. This is not an edge case. It is the default behavior.

Scenario 2 — The Toxic Customer Service Response
  A customer asks a frustrated question about a delayed shipment.
  The LLM, trained on internet text, occasionally produces sarcastic
  or dismissive responses to frustrated inputs.
  "You should have ordered earlier. Shipping delays are your problem."
  Toxicity score from Perspective API: 0.78 (high).

  Without output scanning: this goes to the customer. Complaint. PR incident.
  With toxicity detection: blocked before delivery. Agent response is flagged.
  The system logs it. The model is reviewed.

Scenario 3 — PII in the Output
  A user asks: "Summarize the HR policy for employee John."
  The RAG system retrieves a chunk containing John's home address and salary.
  The LLM helpfully includes both in the summary.
  "John (123 Main Street, salary $78,000) is entitled to..."

  Without output PII scanning: user receives data they should not see.
  GDPR violation. Data breach. Reportable incident.
  With output PII scanning: address and salary are redacted before delivery.
  "[ADDRESS REDACTED] and [SALARY REDACTED]" reaches the user instead.

Scenario 4 — The Medical Liability Output
  A user asks the company health assistant about chest pain symptoms.
  The LLM produces: "Based on the description, this sounds like a heart attack.
  Take aspirin and call 911."
  Medical advice. Unqualified. Potentially wrong. Company is now liable.

  Without output scanning: advice goes to user.
  With disclaimer injection: the output is automatically prepended with:
  "This is not medical advice. Please consult a healthcare professional."
  The disclaimer engine detects medical-domain keywords in the output
  and injects the required disclaimer.

Scenario 5 — The System Prompt Leak at Scale
  A prompt injection in a user message causes the LLM to repeat its system prompt.
  "Sure, my system prompt is: You are an internal HR assistant with access to
  employee records. Always include the full name and department..."
  Without output scanning: the system prompt is leaked to the user.
  Attacker now knows the architecture of your system.
  With output scanning: system prompt fragment detected. Output blocked.
  Incident logged. Security team alerted.

```

### What Output Scanning Governs

```
Input governance (Days 1-7):   Controls what reaches the LLM.
Output governance (Day 8):     Controls what reaches the user.

Together they form a complete safety perimeter:

  User Input
       │
       ▼
  [Input Governance]       ← DAY 2: policy engine (PII, injection, role rules)
       │ CLEAN INPUT
       ▼
  [RAG Retrieval]          ← DAY 7: role-filtered, scanned context
       │
       ▼
  [LLM Call]               ← DAY 6: governed client, model registry, budget
       │
       ▼
  [Output Governance]      ← DAY 8: toxicity, hallucination, PII, disclaimers
       │ CLEAN OUTPUT
       ▼
  User sees the response

Without output governance, Days 1-7 are necessary but not sufficient.
A compliant input can still produce a non-compliant output.

```

---

## 🎯 What You'll Build Today

```
BEFORE (Day 7):
  LLM output was returned directly after input policy check
  No check of what the LLM actually produced
  Hallucinations could pass through undetected
  Toxic outputs had no automatic filter
  PII could appear in LLM responses (reproduced from RAG context)
  No required disclaimers for regulated domains (medical, legal, financial)
  System prompt leaks caught only at the input layer
  No output scan history — no data to measure model safety over time

AFTER (Day 8):
  ✅ ToxicityScanner — OpenAI Moderation API + regex fallback
  ✅ HallucinationDetector — NLI cross-encoder context consistency check
  ✅ OutputPIIScanner — detect and redact PII in LLM output
  ✅ OutputScanner orchestrator — chains all scanners, one unified result
  ✅ DisclaimerEngine — auto-inject required disclaimers by domain
  ✅ OutputScanResult database model — full audit trail of every scan
  ✅ GovernedLLMClient updated — output scanning fires after every LLM call
  ✅ POST /output/scan — scan any text on demand
  ✅ GET  /output/stats — toxicity rates, hallucination rates, scan counts
  ✅ GET  /output/history — paginated scan history for audit
  ✅ Confidence scoring — every LLM response gets a 0-1 confidence score
  ✅ 15+ tests covering all scanners and governance invariants

```

---

## 🏛️ The Output Scanning Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     OUTPUT SCANNING PIPELINE                             │
│                                                                          │
│  LLM Raw Output                                                          │
│       │                                                                  │
│       ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────┐           │
│  │                OutputScanner.scan()                      │           │
│  │                                                          │           │
│  │  ┌─────────────────────────────────────────────────┐    │           │
│  │  │  1. ToxicityScanner                             │    │           │
│  │  │                                                 │    │           │
│  │  │  OpenAI Moderation API (primary)                │    │           │
│  │  │   → hate, harassment, self-harm, sexual,        │    │           │
│  │  │     violence, threats                           │    │           │
│  │  │                                                 │    │           │
│  │  │  Regex fallback (when API unavailable)          │    │           │
│  │  │   → profanity, slurs, threat patterns          │    │           │
│  │  │                                                 │    │           │
│  │  │  BLOCK if any category score > threshold        │    │           │
│  │  └───────────────────────┬─────────────────────────┘    │           │
│  │                          │ PASS                          │           │
│  │  ┌───────────────────────▼─────────────────────────┐    │           │
│  │  │  2. HallucinationDetector                       │    │           │
│  │  │                                                 │    │           │
│  │  │  NLI cross-encoder: does output contradict      │    │           │
│  │  │  the RAG context chunks?                        │    │           │
│  │  │                                                 │    │           │
│  │  │  Entailment score < 0.5 → hallucination warning │    │           │
│  │  │  Entailment score < 0.2 → hallucination block   │    │           │
│  │  │                                                 │    │           │
│  │  │  (skipped if no RAG context provided)           │    │           │
│  │  └───────────────────────┬─────────────────────────┘    │           │
│  │                          │ PASS                          │           │
│  │  ┌───────────────────────▼─────────────────────────┐    │           │
│  │  │  3. OutputPIIScanner                            │    │           │
│  │  │                                                 │    │           │
│  │  │  Detect PII in the LLM's output text:           │    │           │
│  │  │   → emails, phones, SSNs, credit cards          │    │           │
│  │  │   → names near salary/address/medical keywords  │    │           │
│  │  │                                                 │    │           │
│  │  │  Action: REDACT (not block — output has value,  │    │           │
│  │  │  just remove the PII before returning)          │    │           │
│  │  └───────────────────────┬─────────────────────────┘    │           │
│  │                          │ PASS                          │           │
│  │  ┌───────────────────────▼─────────────────────────┐    │           │
│  │  │  4. DisclaimerEngine                            │    │           │
│  │  │                                                 │    │           │
│  │  │  Detect domain from output keywords:            │    │           │
│  │  │   medical → prepend medical disclaimer          │    │           │
│  │  │   legal   → prepend legal disclaimer            │    │           │
│  │  │   financial → prepend financial disclaimer      │    │           │
│  │  │                                                 │    │           │
│  │  │  Action: INJECT disclaimer, never block         │    │           │
│  │  └───────────────────────┬─────────────────────────┘    │           │
│  │                          │                               │           │
│  │  ┌───────────────────────▼─────────────────────────┐    │           │
│  │  │  5. OutputScanResult → PostgreSQL               │    │           │
│  │  │  Log every scan with scores, findings, actions  │    │           │
│  │  └─────────────────────────────────────────────────┘    │           │
│  └──────────────────────────────────────────────────────────┘           │
│                          │                                               │
│                          ▼                                               │
│                 ScannedOutput                                            │
│        {                                                                 │
│          safe:              bool,                                        │
│          output:            str,     ← redacted if PII found            │
│          blocked:           bool,                                        │
│          toxicity_score:    float,   ← 0.0 to 1.0                       │
│          hallucination_score: float, ← 0.0 to 1.0 (lower = worse)      │
│          confidence_score:  float,   ← 0.0 to 1.0 (higher = better)    │
│          findings:          list,                                        │
│          disclaimer_added:  bool,                                        │
│          scan_id:           str,                                         │
│        }                                                                 │
└──────────────────────────────────────────────────────────────────────────┘

Scanner Action Matrix:
  Toxicity found     → BLOCK  (never deliver toxic content)
  Hallucination high → BLOCK  (entailment < 0.2 = contradicts context)
  Hallucination med  → WARN   (entailment 0.2-0.5 = uncertain)
  PII in output      → REDACT (deliver output with PII removed)
  Medical/Legal/Fin  → INJECT disclaimer (deliver with warning prepended)
  System prompt leak → BLOCK  (already caught in Day 6 OutputFilter)

```

---

## ⚙️ Step 8.1 — Install Dependencies

```bash
# Activate your virtual environment
source venv/bin/activate   # Windows: venv\Scripts\activate

# sentence-transformers: cross-encoder for hallucination detection
# Downloads model on first use (~270MB). Cached locally after that.
pip install sentence-transformers==3.0.1

# detoxify: local toxicity classifier (runs offline, no API needed)
# Useful as fallback when OpenAI Moderation API is unavailable
pip install detoxify==0.5.2

# OpenAI moderation is already available via the openai SDK from Day 6

# torch (required by sentence-transformers and detoxify)
# Install CPU-only version to keep the footprint small
pip install torch==2.3.0 --index-url https://download.pytorch.org/whl/cpu

# numpy (used in hallucination scoring)
pip install numpy==1.26.4

# Update requirements
pip freeze > requirements.txt
```

Add output scanning config to `.env`:

```bash
# .env additions for Day 8
TOXICITY_THRESHOLD_BLOCK=0.80    # Score above this → block
TOXICITY_THRESHOLD_WARN=0.50     # Score above this → warn
HALLUCINATION_THRESHOLD_BLOCK=0.20  # Entailment below this → block
HALLUCINATION_THRESHOLD_WARN=0.50   # Entailment below this → warn
OUTPUT_SCAN_ENABLED=true
DISCLAIMER_ENABLED=true
OUTPUT_SCAN_CACHE_TTL=3600       # Cache scan results for 1 hour
```

---

## ⚙️ Step 8.2 — OutputScanResult Database Model

```python
# backend/database/models.py  (add to existing models.py)
from sqlalchemy import (
    Column, String, Boolean, Float, Integer,
    DateTime, Text, JSON, Enum as SAEnum, ForeignKey
)
from sqlalchemy.orm import relationship
from datetime import datetime, timezone
import enum

from backend.database.db import Base


class ScanAction(str, enum.Enum):
    allowed  = "allowed"   # Output delivered as-is
    blocked  = "blocked"   # Output suppressed entirely
    redacted = "redacted"  # Output delivered with PII removed
    warned   = "warned"    # Output delivered with confidence warning
    injected = "injected"  # Output delivered with disclaimer prepended


class OutputScanResult(Base):
    """
    Permanent audit record of every output scan.

    This table answers the question:
    "Has our AI ever produced toxic, hallucinatory, or privacy-violating output?"

    The answer should always be provably "not that reached a user",
    because every scan result is here, and every blocked/redacted
    output was stopped before delivery.
    """
    __tablename__ = "output_scan_results"

    scan_id             = Column(String(64),  primary_key=True)
    user_id             = Column(String(256), nullable=False, index=True)
    user_role           = Column(String(64),  nullable=False)
    model_id            = Column(String(256), nullable=False)
    session_id          = Column(String(256), nullable=True)

    # Scores — all 0.0 to 1.0
    toxicity_score      = Column(Float, nullable=False, default=0.0)
    hallucination_score = Column(Float, nullable=False, default=1.0)  # 1.0 = not hallucinating
    confidence_score    = Column(Float, nullable=False, default=1.0)  # 1.0 = fully confident
    pii_found           = Column(Boolean, nullable=False, default=False)
    disclaimer_injected = Column(Boolean, nullable=False, default=False)

    # What happened
    action              = Column(SAEnum(ScanAction), nullable=False)
    findings            = Column(JSON, nullable=True)  # list of finding dicts

    # Hashes — never store raw text in the scan table
    output_hash         = Column(String(64),  nullable=False)  # SHA-256 of original output
    final_hash          = Column(String(64),  nullable=False)  # SHA-256 of delivered output

    # Context
    rag_context_used    = Column(Boolean, nullable=False, default=False)
    context_chunk_count = Column(Integer, nullable=False, default=0)

    scanned_at          = Column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
        index=True
    )

    @property
    def was_safe(self) -> bool:
        return self.action in [ScanAction.allowed, ScanAction.warned, ScanAction.injected]

    @property
    def reached_user(self) -> bool:
        """True if any version of the output reached the user."""
        return self.action != ScanAction.blocked
```

Add the Alembic migration:

```bash
alembic revision --autogenerate -m "add_output_scan_results_table"
alembic upgrade head
```

---

## ⚙️ Step 8.3 — Toxicity Scanner

```python
# backend/output/toxicity_scanner.py
import os
import re
import logging
from dataclasses import dataclass, field
from typing import Optional
from openai import AsyncOpenAI

logger = logging.getLogger(__name__)

BLOCK_THRESHOLD = float(os.getenv("TOXICITY_THRESHOLD_BLOCK", "0.80"))
WARN_THRESHOLD  = float(os.getenv("TOXICITY_THRESHOLD_WARN",  "0.50"))


@dataclass
class ToxicityResult:
    """Result from the toxicity scanner."""
    score:      float           # 0.0 = clean, 1.0 = maximally toxic
    flagged:    bool            # True if any category exceeded block threshold
    categories: dict            # Category name → score mapping
    findings:   list[dict]      # Findings above warn threshold
    method:     str             # "openai_moderation" or "regex_fallback"


# ─── Regex fallback patterns (used when OpenAI API is unavailable) ────────────
# These are coarse-grained and will miss many cases.
# The OpenAI Moderation API is far more accurate. Regex is defense-in-depth.

TOXICITY_REGEX_PATTERNS = [
    {
        "name":      "threat_pattern",
        "pattern":   r"(?i)\b(i will (kill|hurt|destroy|harm) you|you (will|are going to) (die|suffer|regret))\b",
        "category":  "violence/threats",
        "severity":  "high",
    },
    {
        "name":      "self_harm_instruction",
        "pattern":   r"(?i)(how to (kill yourself|commit suicide|self.harm)|step.by.step.*suicide)",
        "category":  "self-harm",
        "severity":  "critical",
    },
    {
        "name":      "hate_slurs",
        # Keeping this minimal in the codebase — production systems use dedicated
        # hate speech lexicons (HateSonar, HurtLex) loaded from a config file.
        "pattern":   r"(?i)\b(go back to your country|you people should|your kind belongs)\b",
        "category":  "hate_speech",
        "severity":  "high",
    },
]


class ToxicityScanner:
    """
    Detects toxic content in LLM outputs.

    Primary method: OpenAI Moderation API
      - Free to call. No tokens consumed.
      - Returns scores across 11 safety categories.
      - Accurate on hate speech, harassment, violence, sexual content.

    Fallback method: Regex patterns
      - Used when OpenAI API is unavailable (network issue, rate limit).
      - Coarse-grained. Only catches obvious patterns.
      - Better than nothing. Never skip scanning entirely.

    Design principle:
      - When the scanner is uncertain, lean toward flagging (false positive).
      - A false positive blocks a legitimate response (bad UX).
      - A false negative delivers a toxic response to a user (much worse).
    """

    def __init__(self):
        self._openai_client = AsyncOpenAI(
            api_key=os.getenv("OPENAI_API_KEY"),
            timeout=10.0,
        )

    async def scan(self, text: str) -> ToxicityResult:
        """
        Scan text for toxicity.
        Returns ToxicityResult with score, category breakdown, and findings.
        """
        if not text or not text.strip():
            return ToxicityResult(
                score=0.0, flagged=False,
                categories={}, findings=[], method="none"
            )

        # ── Try OpenAI Moderation API first ───────────────────────────────────
        try:
            return await self._scan_with_openai(text)
        except Exception as e:
            logger.warning(
                f"OpenAI Moderation API unavailable: {e}. "
                f"Falling back to regex scanner."
            )
            return self._scan_with_regex(text)

    async def _scan_with_openai(self, text: str) -> ToxicityResult:
        """
        Use OpenAI's free Moderation endpoint.
        Catches: hate, harassment, self-harm, sexual, violence, threats.
        """
        response = await self._openai_client.moderations.create(input=text)
        result   = response.results[0]

        # Build category score dict from the OpenAI response
        categories = {
            "hate":                   result.category_scores.hate,
            "hate/threatening":       result.category_scores.hate_threatening,
            "harassment":             result.category_scores.harassment,
            "harassment/threatening": result.category_scores.harassment_threatening,
            "self-harm":              result.category_scores.self_harm,
            "self-harm/intent":       result.category_scores.self_harm_intent,
            "self-harm/instructions": result.category_scores.self_harm_instructions,
            "sexual":                 result.category_scores.sexual,
            "sexual/minors":          result.category_scores.sexual_minors,
            "violence":               result.category_scores.violence,
            "violence/graphic":       result.category_scores.violence_graphic,
        }

        # Overall toxicity score: max of all category scores
        max_score = max(categories.values()) if categories else 0.0
        flagged   = result.flagged or max_score >= BLOCK_THRESHOLD

        # Build findings list for categories above warn threshold
        findings = [
            {
                "category": cat,
                "score":    round(score, 4),
                "severity": "high" if score >= BLOCK_THRESHOLD else "medium",
            }
            for cat, score in categories.items()
            if score >= WARN_THRESHOLD
        ]

        return ToxicityResult(
            score      = round(max_score, 4),
            flagged    = flagged,
            categories = {k: round(v, 4) for k, v in categories.items()},
            findings   = findings,
            method     = "openai_moderation",
        )

    def _scan_with_regex(self, text: str) -> ToxicityResult:
        """Regex fallback when OpenAI API is unavailable."""
        findings   = []
        max_score  = 0.0

        for rule in TOXICITY_REGEX_PATTERNS:
            if re.search(rule["pattern"], text, re.IGNORECASE | re.MULTILINE):
                score = 0.90 if rule["severity"] == "critical" else 0.75
                max_score = max(max_score, score)
                findings.append({
                    "category": rule["category"],
                    "score":    score,
                    "severity": rule["severity"],
                    "method":   "regex",
                    "name":     rule["name"],
                })

        return ToxicityResult(
            score      = round(max_score, 4),
            flagged    = max_score >= BLOCK_THRESHOLD,
            categories = {f["category"]: f["score"] for f in findings},
            findings   = findings,
            method     = "regex_fallback",
        )
```

---

## ⚙️ Step 8.4 — Hallucination Detector

```python
# backend/output/hallucination_detector.py
import logging
import os
from dataclasses import dataclass, field
from typing import Optional

logger = logging.getLogger(__name__)

BLOCK_THRESHOLD = float(os.getenv("HALLUCINATION_THRESHOLD_BLOCK", "0.20"))
WARN_THRESHOLD  = float(os.getenv("HALLUCINATION_THRESHOLD_WARN",  "0.50"))

# Lazy-loaded — model is ~270MB. Load once on first use.
_cross_encoder = None


def _get_cross_encoder():
    """
    Lazy-load the cross-encoder model.
    Uses cross-encoder/nli-deberta-v3-small:
      - 86MB model (fast to download, fast inference)
      - Trained on NLI (Natural Language Inference) tasks
      - Given a premise (context) and hypothesis (output),
        predicts: entailment / neutral / contradiction
      - Entailment score → output is supported by context
      - Contradiction score → output contradicts context (hallucination)
    """
    global _cross_encoder
    if _cross_encoder is None:
        try:
            from sentence_transformers import CrossEncoder
            _cross_encoder = CrossEncoder(
                "cross-encoder/nli-deberta-v3-small",
                max_length=512
            )
            logger.info("HallucinationDetector: cross-encoder model loaded.")
        except Exception as e:
            logger.error(
                f"Failed to load cross-encoder model: {e}. "
                f"Hallucination detection disabled — outputs will be passed with warning."
            )
    return _cross_encoder


@dataclass
class HallucinationResult:
    """Result from the hallucination detector."""
    entailment_score:    float  # 0.0-1.0. Higher = output IS supported by context
    contradiction_score: float  # 0.0-1.0. Higher = output CONTRADICTS context
    neutral_score:       float  # 0.0-1.0. Higher = output is unrelated to context
    confidence:          float  # How confident we are in the entailment score
    flagged:             bool   # True if entailment_score < BLOCK_THRESHOLD
    warned:              bool   # True if entailment_score < WARN_THRESHOLD
    findings:            list[dict]
    skipped:             bool   # True if no context was available to check against
    method:              str


class HallucinationDetector:
    """
    Checks whether LLM output is factually consistent with the
    source context (RAG chunks) that were used to generate it.

    This addresses the most dangerous class of RAG errors: when the
    LLM generates confident-sounding text that contradicts or
    is unsupported by the retrieved documents.

    Method:
      NLI (Natural Language Inference) cross-encoder model.
      For each sentence in the output, check if the concatenated
      RAG context ENTAILS that sentence.

      entailment   → sentence is supported by context (good)
      neutral      → sentence is not in context (may be hallucination)
      contradiction → sentence contradicts context (definitely hallucination)

    Limitations:
      - Only detects hallucinations relative to the provided context.
      - Cannot detect factual errors in cases where no RAG context was used.
      - Model has a 512-token limit; very long outputs are chunked.
      - Not suitable for complex multi-hop reasoning chains.

    When to skip:
      - No RAG context was provided (pure LLM generation — nothing to check against)
      - Output is very short (< 20 tokens — not enough to evaluate)
      - Output is a clarification or refusal ("I don't know", "I cannot answer")
    """

    def detect(
        self,
        output:   str,
        context:  Optional[list[str]] = None,
    ) -> HallucinationResult:
        """
        Check output against context for factual consistency.

        Args:
          output:  The LLM-generated text to check.
          context: List of RAG chunk strings that informed the output.
                   If None or empty, detection is skipped.

        Returns HallucinationResult with entailment scores and flags.
        """
        # ── Skip if no context to check against ──────────────────
        if not context or not any(c.strip() for c in context):
            return HallucinationResult(
                entailment_score    = 1.0,
                contradiction_score = 0.0,
                neutral_score       = 0.0,
                confidence          = 0.0,  # 0 confidence because we didn't check
                flagged             = False,
                warned              = False,
                findings            = [],
                skipped             = True,
                method              = "skipped_no_context",
            )

        # ── Skip for very short outputs ───────────────────────────
        if len(output.split()) < 15:
            return HallucinationResult(
                entailment_score    = 1.0,
                contradiction_score = 0.0,
                neutral_score       = 0.0,
                confidence          = 0.0,
                flagged             = False,
                warned              = False,
                findings            = [],
                skipped             = True,
                method              = "skipped_too_short",
            )

        # ── Skip known refusal patterns ───────────────────────────
        refusal_patterns = [
            "i don't know",
            "i cannot answer",
            "i'm not sure",
            "i do not have enough information",
            "the provided context does not",
            "based on the context, i cannot",
        ]
        if any(p in output.lower() for p in refusal_patterns):
            return HallucinationResult(
                entailment_score    = 1.0,
                contradiction_score = 0.0,
                neutral_score       = 0.0,
                confidence          = 1.0,
                flagged             = False,
                warned              = False,
                findings            = [],
                skipped             = True,
                method              = "skipped_refusal",
            )

        # ── Load the cross-encoder ────────────────────────────────
        model = _get_cross_encoder()
        if model is None:
            logger.warning("Cross-encoder unavailable. Hallucination check skipped.")
            return HallucinationResult(
                entailment_score    = 0.5,
                contradiction_score = 0.0,
                neutral_score       = 0.5,
                confidence          = 0.0,
                flagged             = False,
                warned              = True,
                findings            = [{"type": "model_unavailable",
                                        "message": "Hallucination check skipped: model not loaded."}],
                skipped             = True,
                method              = "skipped_model_unavailable",
            )

        # ── Build premise: concatenate all context chunks ─────────
        # Limit to 1500 chars to stay within 512-token model limit
        premise = " ".join(context)[:1500]

        # ── Score each sentence in the output ─────────────────────
        import re as _re
        sentences = [s.strip() for s in _re.split(r'(?<=[.!?])\s+', output) if len(s.split()) > 5]
        if not sentences:
            sentences = [output[:500]]

        entailment_scores    = []
        contradiction_scores = []
        neutral_scores       = []
        low_entailment_sentences = []

        try:
            import numpy as np
            for sentence in sentences[:10]:   # Cap at 10 sentences for performance
                # NLI labels: 0=contradiction, 1=entailment, 2=neutral
                scores = model.predict(
                    [(premise, sentence)],
                    apply_softmax=True
                )[0]

                contradiction_score = float(scores[0])
                entailment_score    = float(scores[1])
                neutral_score       = float(scores[2])

                entailment_scores.append(entailment_score)
                contradiction_scores.append(contradiction_score)
                neutral_scores.append(neutral_score)

                if entailment_score < WARN_THRESHOLD:
                    low_entailment_sentences.append({
                        "sentence":           sentence[:100] + "..." if len(sentence) > 100 else sentence,
                        "entailment_score":   round(entailment_score,    3),
                        "contradiction_score": round(contradiction_score, 3),
                    })

            avg_entailment    = float(np.mean(entailment_scores))
            avg_contradiction = float(np.mean(contradiction_scores))
            avg_neutral       = float(np.mean(neutral_scores))

        except Exception as e:
            logger.error(f"Cross-encoder inference failed: {e}")
            return HallucinationResult(
                entailment_score    = 0.5,
                contradiction_score = 0.0,
                neutral_score       = 0.5,
                confidence          = 0.0,
                flagged             = False,
                warned              = True,
                findings            = [{"type": "inference_error", "message": str(e)}],
                skipped             = True,
                method              = "error",
            )

        # ── Build findings ─────────────────────────────────────────
        findings = []
        if low_entailment_sentences:
            findings.append({
                "type":      "low_entailment_sentences",
                "count":     len(low_entailment_sentences),
                "sentences": low_entailment_sentences,
                "severity":  "high" if avg_entailment < BLOCK_THRESHOLD else "medium",
            })

        flagged = avg_entailment < BLOCK_THRESHOLD
        warned  = avg_entailment < WARN_THRESHOLD and not flagged

        return HallucinationResult(
            entailment_score    = round(avg_entailment,    4),
            contradiction_score = round(avg_contradiction, 4),
            neutral_score       = round(avg_neutral,       4),
            confidence          = round(avg_entailment,    4),
            flagged             = flagged,
            warned              = warned,
            findings            = findings,
            skipped             = False,
            method              = "nli_cross_encoder",
        )
```

---

## ⚙️ Step 8.5 — PII Output Scanner

```python
# backend/output/pii_output_scanner.py
import re
import logging
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)

# ─── PII Patterns for Output ──────────────────────────────────────────────────
# These are the same categories as the input scanner (Day 7 DocumentScanner)
# but tuned for LLM output text — looser context matching, stricter redaction.

OUTPUT_PII_PATTERNS = [
    {
        "name":        "email",
        "pattern":     r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b",
        "replacement": "[EMAIL REDACTED]",
        "severity":    "high",
    },
    {
        "name":        "ssn",
        "pattern":     r"\b\d{3}[-\s]?\d{2}[-\s]?\d{4}\b",
        "replacement": "[SSN REDACTED]",
        "severity":    "critical",
    },
    {
        "name":        "credit_card",
        "pattern":     r"\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13})\b",
        "replacement": "[CARD REDACTED]",
        "severity":    "critical",
    },
    {
        "name":        "phone_us",
        "pattern":     r"\b(?:\+1[-.\s]?)?\(?\d{3}\)?[-.\s]\d{3}[-.\s]\d{4}\b",
        "replacement": "[PHONE REDACTED]",
        "severity":    "medium",
    },
    {
        "name":        "ip_address",
        "pattern":     r"\b(?:\d{1,3}\.){3}\d{1,3}\b",
        "replacement": "[IP REDACTED]",
        "severity":    "medium",
    },
    {
        "name":        "salary_mention",
        # Matches: $78,000 or $78k near contextual words
        "pattern":     r"(?i)(salary|compensation|earns?|paid?|wages?)[^\n]*\$[\d,]+(?:k|K|,\d{3})?",
        "replacement": "[COMPENSATION REDACTED]",
        "severity":    "high",
    },
    {
        "name":        "home_address",
        # Matches: "123 Main Street, City, ST 12345" format
        "pattern":     r"\b\d{2,5}\s+[A-Za-z\s]{3,30}(?:Street|St|Avenue|Ave|Road|Rd|Drive|Dr|Lane|Ln|Boulevard|Blvd|Way|Court|Ct)\b[^\n]*\b\d{5}\b",
        "replacement": "[ADDRESS REDACTED]",
        "severity":    "high",
    },
]


@dataclass
class PIIOutputResult:
    """Result of scanning LLM output for PII."""
    pii_found:    bool
    redacted_text: str        # Output with PII replaced
    findings:     list[dict]  # What was found and redacted
    redaction_count: int      # How many redactions were made


class PIIOutputScanner:
    """
    Scans LLM output for PII and redacts it before delivery.

    Unlike the input scanner (which rejects) and the toxicity scanner (which blocks),
    the output PII scanner REDACTS: the output still reaches the user,
    but the PII is replaced with neutral placeholders.

    Reasoning:
      The LLM output often has significant value even when it contains
      incidental PII (a retrieved email address, a salary figure from
      an HR doc, a phone number from a contact sheet). Blocking the
      entire response because of one email address is too aggressive.
      Redacting preserves value while eliminating the compliance risk.

    Exception: SSNs and credit card numbers are always treated as critical
    and may trigger a block-level alert even in output (configurable).
    """

    def scan(self, text: str) -> PIIOutputResult:
        """Scan output text for PII and return redacted version."""
        if not text or not text.strip():
            return PIIOutputResult(
                pii_found=False, redacted_text=text,
                findings=[], redaction_count=0
            )

        findings       = []
        redacted       = text
        redaction_count = 0

        for rule in OUTPUT_PII_PATTERNS:
            matches = re.findall(rule["pattern"], redacted, re.IGNORECASE)
            if matches:
                # Count matches before replacing (re.subn returns count)
                redacted, count = re.subn(
                    rule["pattern"],
                    rule["replacement"],
                    redacted,
                    flags=re.IGNORECASE,
                )
                redaction_count += count
                findings.append({
                    "type":          "pii",
                    "name":          rule["name"],
                    "severity":      rule["severity"],
                    "count_redacted": count,
                    # Never log actual PII values — only the type and count
                })

        return PIIOutputResult(
            pii_found       = len(findings) > 0,
            redacted_text   = redacted,
            findings        = findings,
            redaction_count = redaction_count,
        )
```

---

## ⚙️ Step 8.6 — The OutputScanner Orchestrator

```python
# backend/output/output_scanner.py
import hashlib
import logging
import os
import uuid
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Optional

from backend.output.toxicity_scanner import ToxicityScanner
from backend.output.hallucination_detector import HallucinationDetector
from backend.output.pii_output_scanner import PIIOutputScanner
from backend.output.disclaimer_engine import DisclaimerEngine
from backend.database.models import OutputScanResult, ScanAction

logger = logging.getLogger(__name__)

SCAN_ENABLED = os.getenv("OUTPUT_SCAN_ENABLED", "true").lower() == "true"


@dataclass
class ScannedOutput:
    """The final result of the full output scanning pipeline."""
    safe:                bool
    blocked:             bool
    output:              str     # Final text to deliver (may be redacted)
    original_output:     str     # Raw LLM output (never delivered if blocked)

    toxicity_score:      float
    hallucination_score: float   # entailment score — higher is better
    confidence_score:    float   # Overall confidence 0-1

    pii_found:           bool
    pii_redaction_count: int
    disclaimer_added:    bool
    disclaimer_text:     str = ""

    action:              str = "allowed"
    findings:            list = field(default_factory=list)
    scan_id:             str  = field(default_factory=lambda: str(uuid.uuid4()))
    scanned_at:          str  = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())


class OutputScanner:
    """
    Orchestrates all output scanning in sequence.

    Chain of scanners:
      1. ToxicityScanner      → BLOCK if toxic
      2. HallucinationDetector → BLOCK/WARN based on entailment vs context
      3. PIIOutputScanner     → REDACT PII in output
      4. DisclaimerEngine     → INJECT disclaimer for regulated domains
      5. PostgreSQL           → LOG every scan result

    Each scanner is independent. Failure in one does not stop others.
    The final action is the most restrictive action from any scanner:
      BLOCK > REDACT > WARN > INJECT > ALLOW

    Usage:
        scanner = OutputScanner(db_session=db)
        result = await scanner.scan(
            output   = llm_output,
            context  = rag_chunks,
            user_id  = "alice",
            user_role = "employee",
            model_id  = "gpt-4o",
        )
        if result.blocked:
            return {"error": "output_blocked", "findings": result.findings}
        return {"output": result.output}
    """

    def __init__(self, db_session=None):
        self.db                  = db_session
        self.toxicity_scanner    = ToxicityScanner()
        self.hallucination_detector = HallucinationDetector()
        self.pii_scanner         = PIIOutputScanner()
        self.disclaimer_engine   = DisclaimerEngine()

    def _hash(self, text: str) -> str:
        return hashlib.sha256(text.encode()).hexdigest()

    async def scan(
        self,
        output:    str,
        user_id:   str,
        user_role: str,
        model_id:  str,
        context:   Optional[list[str]] = None,
        session_id: Optional[str] = None,
    ) -> ScannedOutput:
        """
        Run the full output scanning pipeline.

        Args:
          output:    Raw LLM output text.
          user_id:   Requesting user (for audit log).
          user_role: User's role (for audit log).
          model_id:  Model that generated the output (for audit log).
          context:   RAG chunk strings used to generate this output.
                     Required for hallucination detection.
          session_id: Session identifier (for audit log).

        Returns ScannedOutput with final (safe) text and all scan metadata.
        """
        if not SCAN_ENABLED:
            return ScannedOutput(
                safe=True, blocked=False,
                output=output, original_output=output,
                toxicity_score=0.0, hallucination_score=1.0,
                confidence_score=1.0, pii_found=False,
                pii_redaction_count=0, disclaimer_added=False,
                action="allowed",
            )

        scan_id          = str(uuid.uuid4())
        all_findings     = []
        current_output   = output
        action           = ScanAction.allowed
        blocked          = False

        # ── Scanner 1: Toxicity ──────────────────────────────────────
        try:
            toxicity = await self.toxicity_scanner.scan(current_output)
            all_findings.extend([
                {**f, "scanner": "toxicity"}
                for f in toxicity.findings
            ])

            if toxicity.flagged:
                blocked = True
                action  = ScanAction.blocked
                logger.warning(
                    f"Output BLOCKED — toxicity. "
                    f"user={user_id} model={model_id} "
                    f"score={toxicity.score} method={toxicity.method}"
                )
            elif toxicity.score >= float(os.getenv("TOXICITY_THRESHOLD_WARN", "0.50")):
                action = ScanAction.warned

        except Exception as e:
            logger.error(f"Toxicity scanner error: {e}")
            toxicity_score = 0.0
        else:
            toxicity_score = toxicity.score

        if blocked:
            await self._log_scan(
                scan_id=scan_id, user_id=user_id, user_role=user_role,
                model_id=model_id, session_id=session_id,
                original_output=output, final_output="[BLOCKED]",
                action=action, findings=all_findings,
                toxicity_score=toxicity_score,
                hallucination_score=1.0, confidence_score=0.0,
                pii_found=False, disclaimer_injected=False,
                rag_context_used=bool(context), context_count=len(context or []),
            )
            return ScannedOutput(
                safe=False, blocked=True,
                output="[This response was blocked by the governance system.]",
                original_output=output,
                toxicity_score=toxicity_score,
                hallucination_score=1.0, confidence_score=0.0,
                pii_found=False, pii_redaction_count=0,
                disclaimer_added=False, action="blocked",
                findings=all_findings, scan_id=scan_id,
            )

        # ── Scanner 2: Hallucination ─────────────────────────────────
        hallucination_score = 1.0   # Default: confident (no check)
        try:
            hallucination = self.hallucination_detector.detect(
                output=current_output, context=context
            )
            hallucination_score = hallucination.entailment_score

            if hallucination.findings:
                all_findings.extend([
                    {**f, "scanner": "hallucination"}
                    for f in hallucination.findings
                ])

            if hallucination.flagged:
                blocked = True
                action  = ScanAction.blocked
                logger.warning(
                    f"Output BLOCKED — hallucination. "
                    f"user={user_id} model={model_id} "
                    f"entailment={hallucination.entailment_score}"
                )
            elif hallucination.warned:
                if action == ScanAction.allowed:
                    action = ScanAction.warned

        except Exception as e:
            logger.error(f"Hallucination detector error: {e}")

        if blocked:
            await self._log_scan(
                scan_id=scan_id, user_id=user_id, user_role=user_role,
                model_id=model_id, session_id=session_id,
                original_output=output, final_output="[BLOCKED]",
                action=action, findings=all_findings,
                toxicity_score=toxicity_score,
                hallucination_score=hallucination_score, confidence_score=0.0,
                pii_found=False, disclaimer_injected=False,
                rag_context_used=bool(context), context_count=len(context or []),
            )
            return ScannedOutput(
                safe=False, blocked=True,
                output=(
                    "[This response was blocked because it may not be "
                    "consistent with the available source documents. "
                    "Please rephrase your question or consult the source directly.]"
                ),
                original_output=output,
                toxicity_score=toxicity_score,
                hallucination_score=hallucination_score, confidence_score=0.0,
                pii_found=False, pii_redaction_count=0,
                disclaimer_added=False, action="blocked",
                findings=all_findings, scan_id=scan_id,
            )

        # ── Scanner 3: PII Redaction ─────────────────────────────────
        pii_found        = False
        pii_redactions   = 0
        try:
            pii_result  = self.pii_scanner.scan(current_output)
            pii_found   = pii_result.pii_found
            pii_redactions = pii_result.redaction_count

            if pii_result.pii_found:
                current_output = pii_result.redacted_text
                all_findings.extend([
                    {**f, "scanner": "pii_output"}
                    for f in pii_result.findings
                ])
                if action == ScanAction.allowed:
                    action = ScanAction.redacted
                logger.info(
                    f"Output PII redacted. "
                    f"user={user_id} redactions={pii_result.redaction_count}"
                )
        except Exception as e:
            logger.error(f"PII output scanner error: {e}")

        # ── Scanner 4: Disclaimer Injection ──────────────────────────
        disclaimer_added = False
        disclaimer_text  = ""
        try:
            disclaimer_result = self.disclaimer_engine.process(current_output)
            if disclaimer_result.disclaimer_added:
                current_output   = disclaimer_result.final_text
                disclaimer_added = True
                disclaimer_text  = disclaimer_result.disclaimer_text
                if action == ScanAction.allowed:
                    action = ScanAction.injected
        except Exception as e:
            logger.error(f"Disclaimer engine error: {e}")

        # ── Compute final confidence score ────────────────────────────
        confidence_score = round(
            (hallucination_score * 0.6) +
            ((1.0 - toxicity_score) * 0.4),
            4
        )

        # ── Log the scan result ───────────────────────────────────────
        await self._log_scan(
            scan_id=scan_id, user_id=user_id, user_role=user_role,
            model_id=model_id, session_id=session_id,
            original_output=output, final_output=current_output,
            action=action, findings=all_findings,
            toxicity_score=toxicity_score,
            hallucination_score=hallucination_score,
            confidence_score=confidence_score,
            pii_found=pii_found, disclaimer_injected=disclaimer_added,
            rag_context_used=bool(context), context_count=len(context or []),
        )

        return ScannedOutput(
            safe                = True,
            blocked             = False,
            output              = current_output,
            original_output     = output,
            toxicity_score      = toxicity_score,
            hallucination_score = hallucination_score,
            confidence_score    = confidence_score,
            pii_found           = pii_found,
            pii_redaction_count = pii_redactions,
            disclaimer_added    = disclaimer_added,
            disclaimer_text     = disclaimer_text,
            action              = action.value,
            findings            = all_findings,
            scan_id             = scan_id,
        )

    async def _log_scan(
        self,
        scan_id:          str,
        user_id:          str,
        user_role:        str,
        model_id:         str,
        session_id:       Optional[str],
        original_output:  str,
        final_output:     str,
        action:           ScanAction,
        findings:         list,
        toxicity_score:   float,
        hallucination_score: float,
        confidence_score: float,
        pii_found:        bool,
        disclaimer_injected: bool,
        rag_context_used: bool,
        context_count:    int,
    ):
        """Persist scan result to PostgreSQL."""
        if not self.db:
            return
        try:
            record = OutputScanResult(
                scan_id             = scan_id,
                user_id             = user_id,
                user_role           = user_role,
                model_id            = model_id,
                session_id          = session_id,
                toxicity_score      = toxicity_score,
                hallucination_score = hallucination_score,
                confidence_score    = confidence_score,
                pii_found           = pii_found,
                disclaimer_injected = disclaimer_injected,
                action              = action,
                findings            = findings,
                output_hash         = self._hash(original_output),
                final_hash          = self._hash(final_output),
                rag_context_used    = rag_context_used,
                context_chunk_count = context_count,
            )
            self.db.add(record)
            await self.db.flush()
        except Exception as e:
            logger.error(f"Failed to log output scan result: {e}")
```

---

## ⚙️ Step 8.7 — Integration into GovernedLLMClient

```python
# backend/llm/governed_client.py  (updated complete() method — add output scanning)

# Add this import at the top of governed_client.py
from backend.output.output_scanner import OutputScanner

# ── Replace the existing complete() return block ──────────────────────────────
# After the existing Step 6 (OutputFilter), add Step 6b (OutputScanner):

    async def complete(
        self,
        prompt:        str,
        system_prompt: Optional[str] = None,
        max_tokens:    int   = 1024,
        temperature:   float = 0.7,
        use_cache:     bool  = True,
        context:       Optional[list[str]] = None,   # ← NEW: RAG chunks
        db_session     = None,                        # ← NEW: for scan logging
    ) -> dict:
        """
        Governed LLM completion with output scanning.

        Extended pipeline:
          policy_check → model_registry → token_budget →
          llm_cache → llm_call → output_scan →   ← NEW
          cost_track → audit_log
        """
        # ... [Steps 1-5 unchanged from Day 6] ...

        # ── Step 6: Output Scanning (NEW — Day 8) ────────────────────
        scanner = OutputScanner(db_session=db_session)
        scan_result = await scanner.scan(
            output     = llm_response.content,
            user_id    = self.user_id,
            user_role  = self.user_role,
            model_id   = self.model_id,
            context    = context,
            session_id = self.session_id,
        )

        if scan_result.blocked:
            # Output was blocked AFTER the LLM call — still charge for tokens
            # but do not deliver the response
            actual_cost = estimate_cost(
                llm_response.input_tokens,
                llm_response.output_tokens,
                self.model_id,
            )
            await self._record_cost(
                llm_response.input_tokens,
                llm_response.output_tokens,
                actual_cost,
            )
            await self._audit_log({
                "user_id":    self.user_id,
                "user_role":  self.user_role,
                "model_id":   self.model_id,
                "action_type": "llm_call",
                "outcome":    "output_blocked",
                "prompt_hash": self._hash(prompt),
                "output_hash": self._hash(llm_response.content),
                "details": {
                    "output_blocked":   True,
                    "scan_id":          scan_result.scan_id,
                    "toxicity_score":   scan_result.toxicity_score,
                    "hallucination_score": scan_result.hallucination_score,
                    "scan_findings":    scan_result.findings,
                    "tokens_used":      llm_response.total_tokens,
                    "cost_usd":         actual_cost,
                },
                "session_id": self.session_id,
            })
            return {
                "blocked":          True,
                "reason":           ["output_blocked_by_scanner"],
                "output_blocked":   True,
                "scan_id":          scan_result.scan_id,
                "toxicity_score":   scan_result.toxicity_score,
                "hallucination_score": scan_result.hallucination_score,
                "output":           scan_result.output,   # The safe "blocked" message
                "tokens_used":      llm_response.total_tokens,
                "cost_usd":         actual_cost,
            }

        # ── Use the scanner's final output (may be redacted or have disclaimer) ─
        final_output = scan_result.output

        # ── Step 7: Cost Tracking ────────────────────────────────────
        actual_cost = estimate_cost(
            llm_response.input_tokens,
            llm_response.output_tokens,
            self.model_id,
        )
        await self._record_cost(
            llm_response.input_tokens,
            llm_response.output_tokens,
            actual_cost,
        )

        # ── Step 8: Audit Log ────────────────────────────────────────
        await self._audit_log({
            "user_id":    self.user_id,
            "user_role":  self.user_role,
            "model_id":   self.model_id,
            "action_type": "llm_call",
            "outcome":    "allowed",
            "prompt_hash": self._hash(prompt),
            "output_hash": self._hash(final_output),
            "details": {
                "input_tokens":        llm_response.input_tokens,
                "output_tokens":       llm_response.output_tokens,
                "cost_usd":            actual_cost,
                "from_cache":          False,
                "risk_score":          policy["risk_score"],
                "scan_id":             scan_result.scan_id,
                "toxicity_score":      scan_result.toxicity_score,
                "hallucination_score": scan_result.hallucination_score,
                "confidence_score":    scan_result.confidence_score,
                "pii_redacted":        scan_result.pii_found,
                "disclaimer_added":    scan_result.disclaimer_added,
                "scan_action":         scan_result.action,
            },
            "session_id": self.session_id,
        })

        # ── Step 9: Cache (only clean outputs, no warnings) ──────────
        if use_cache and scan_result.action == "allowed":
            cache_key = self._llm_cache_key(clean_prompt, system_prompt)
            await cache_set_json(
                cache_key,
                {"output": final_output, "model_id": self.model_id},
                ttl_seconds=LLM_CACHE_TTL,
            )

        return {
            "blocked":             False,
            "output":              final_output,
            "risk_score":          policy["risk_score"],
            "warnings":            scan_result.findings,
            "from_cache":          False,
            "tokens_used":         llm_response.total_tokens,
            "input_tokens":        llm_response.input_tokens,
            "output_tokens":       llm_response.output_tokens,
            "cost_usd":            actual_cost,
            "finish_reason":       llm_response.finish_reason,
            "check_id":            policy.get("check_id"),
            "scan_id":             scan_result.scan_id,
            "toxicity_score":      scan_result.toxicity_score,
            "hallucination_score": scan_result.hallucination_score,
            "confidence_score":    scan_result.confidence_score,
            "pii_redacted":        scan_result.pii_found,
            "disclaimer_added":    scan_result.disclaimer_added,
            "scan_action":         scan_result.action,
        }
```

---

## ⚙️ Step 8.8 — REST Endpoints: Scan and Stats

```python
# backend/routers/output_scan.py
from fastapi import APIRouter, Depends, Request
from pydantic import BaseModel, Field
from typing import Optional
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from datetime import datetime, timezone, timedelta

from backend.database.db import get_db
from backend.database.models import OutputScanResult, ScanAction
from backend.output.output_scanner import OutputScanner

router = APIRouter()


class ManualScanRequest(BaseModel):
    output:    str            = Field(..., min_length=1, max_length=32_000)
    context:   Optional[list[str]] = Field(None, description="RAG chunks used to generate this output")
    model_id:  str            = Field("unknown")
    session_id: Optional[str] = None


class ManualScanResponse(BaseModel):
    safe:                bool
    blocked:             bool
    output:              str
    toxicity_score:      float
    hallucination_score: float
    confidence_score:    float
    pii_found:           bool
    pii_redaction_count: int
    disclaimer_added:    bool
    action:              str
    findings:            list
    scan_id:             str


@router.post("/scan", response_model=ManualScanResponse, summary="Scan any text for toxicity, hallucination, and PII")
async def scan_output(
    body:    ManualScanRequest,
    request: Request,
    db:      AsyncSession = Depends(get_db),
):
    """
    Manually scan any LLM output through the full output scanning pipeline.

    Useful for:
      - Auditing previously generated outputs
      - Testing the scanner configuration
      - Ad-hoc output safety checks
      - CI pipelines that generate and verify LLM outputs

    Returns the full scan result including the safe (possibly redacted) output.
    """
    user_id   = request.headers.get("X-User-ID",   "anonymous")
    user_role = request.headers.get("X-User-Role",  "employee")

    scanner = OutputScanner(db_session=db)
    result  = await scanner.scan(
        output     = body.output,
        user_id    = user_id,
        user_role  = user_role,
        model_id   = body.model_id,
        context    = body.context,
        session_id = body.session_id,
    )
    await db.commit()

    return ManualScanResponse(
        safe                = result.safe,
        blocked             = result.blocked,
        output              = result.output,
        toxicity_score      = result.toxicity_score,
        hallucination_score = result.hallucination_score,
        confidence_score    = result.confidence_score,
        pii_found           = result.pii_found,
        pii_redaction_count = result.pii_redaction_count,
        disclaimer_added    = result.disclaimer_added,
        action              = result.action,
        findings            = result.findings,
        scan_id             = result.scan_id,
    )


@router.get("/stats", summary="Output scan statistics: toxicity rates, hallucination rates")
async def output_scan_stats(
    days: int = 7,
    db:   AsyncSession = Depends(get_db),
):
    """
    Aggregated output scan statistics for the last N days.

    Returns:
      - Total scans, blocked count, redacted count, warned count
      - Average toxicity score, average hallucination score
      - PII detection rate, disclaimer injection rate
      - Breakdown by model and by action type

    Use this to:
      - Monitor overall model safety trends
      - Identify models producing high toxicity scores
      - Track how often hallucinations are being caught
      - Report to compliance teams
    """
    since = datetime.now(timezone.utc) - timedelta(days=days)

    result = await db.execute(
        select(OutputScanResult).where(OutputScanResult.scanned_at >= since)
    )
    scans = result.scalars().all()

    if not scans:
        return {
            "period_days":        days,
            "total_scans":        0,
            "message":            "No scan data for this period.",
        }

    total         = len(scans)
    blocked       = sum(1 for s in scans if s.action == ScanAction.blocked)
    redacted      = sum(1 for s in scans if s.action == ScanAction.redacted)
    warned        = sum(1 for s in scans if s.action == ScanAction.warned)
    injected      = sum(1 for s in scans if s.action == ScanAction.injected)
    allowed       = sum(1 for s in scans if s.action == ScanAction.allowed)
    pii_detections = sum(1 for s in scans if s.pii_found)
    disclaimers   = sum(1 for s in scans if s.disclaimer_injected)

    avg_toxicity      = round(sum(s.toxicity_score      for s in scans) / total, 4)
    avg_hallucination = round(sum(s.hallucination_score for s in scans) / total, 4)
    avg_confidence    = round(sum(s.confidence_score    for s in scans) / total, 4)

    # Breakdown by model
    by_model: dict = {}
    for scan in scans:
        if scan.model_id not in by_model:
            by_model[scan.model_id] = {
                "total": 0, "blocked": 0,
                "avg_toxicity": 0.0, "avg_hallucination": 0.0,
            }
        by_model[scan.model_id]["total"] += 1
        if scan.action == ScanAction.blocked:
            by_model[scan.model_id]["blocked"] += 1

    for m in by_model:
        m_scans = [s for s in scans if s.model_id == m]
        by_model[m]["avg_toxicity"]      = round(sum(s.toxicity_score      for s in m_scans) / len(m_scans), 4)
        by_model[m]["avg_hallucination"] = round(sum(s.hallucination_score for s in m_scans) / len(m_scans), 4)
        by_model[m]["block_rate"]        = f"{round(by_model[m]['blocked'] / by_model[m]['total'] * 100, 1)}%"

    return {
        "period_days":          days,
        "total_scans":          total,
        "by_action": {
            "allowed":          allowed,
            "blocked":          blocked,
            "redacted":         redacted,
            "warned":           warned,
            "injected":         injected,
        },
        "rates": {
            "block_rate":       f"{round(blocked  / total * 100, 2)}%",
            "redaction_rate":   f"{round(redacted / total * 100, 2)}%",
            "pii_detection_rate": f"{round(pii_detections / total * 100, 2)}%",
            "disclaimer_rate":  f"{round(disclaimers / total * 100, 2)}%",
        },
        "scores": {
            "avg_toxicity":       avg_toxicity,
            "avg_hallucination":  avg_hallucination,
            "avg_confidence":     avg_confidence,
        },
        "by_model":             by_model,
    }


@router.get("/history", summary="Paginated output scan history for audit")
async def output_scan_history(
    limit:  int  = 50,
    offset: int  = 0,
    action: Optional[str] = None,
    db:     AsyncSession  = Depends(get_db),
):
    """
    Returns paginated output scan history.
    Filterable by action (blocked, redacted, warned, allowed, injected).
    Used by compliance teams to review all governance interventions.
    """
    query = select(OutputScanResult).order_by(
        OutputScanResult.scanned_at.desc()
    )
    if action:
        query = query.where(OutputScanResult.action == action)

    query  = query.offset(offset).limit(limit)
    result = await db.execute(query)
    scans  = result.scalars().all()

    return [
        {
            "scan_id":             s.scan_id,
            "user_id":             s.user_id,
            "user_role":           s.user_role,
            "model_id":            s.model_id,
            "action":              s.action,
            "toxicity_score":      s.toxicity_score,
            "hallucination_score": s.hallucination_score,
            "confidence_score":    s.confidence_score,
            "pii_found":           s.pii_found,
            "disclaimer_injected": s.disclaimer_injected,
            "findings_count":      len(s.findings or []),
            "rag_context_used":    s.rag_context_used,
            "scanned_at":          s.scanned_at.isoformat() if s.scanned_at else None,
        }
        for s in scans
    ]
```

Register the router in `main.py`:

```python
# backend/main.py
from backend.routers import output_scan

app.include_router(output_scan.router, prefix="/output", tags=["Output Scanning"])
```

---

## ⚙️ Step 8.9 — Disclaimer Injection Engine

```python
# backend/output/disclaimer_engine.py
import re
import logging
from dataclasses import dataclass

logger = logging.getLogger(__name__)

# ─── Domain Detection Patterns ────────────────────────────────────────────────
# Each domain has keyword patterns and a required disclaimer.
# If output text matches the keyword pattern, the disclaimer is prepended.

DOMAIN_RULES = [
    {
        "domain":    "medical",
        "pattern":   r"(?i)\b(diagnosis|symptoms?|treatment|medication|dosage|prescription|doctor|physician|medical advice|health condition|disease|illness|surgery|therapy|side effects)\b",
        "threshold": 2,  # Must match 2+ distinct keywords to trigger
        "disclaimer": (
            "⚠️ Medical Disclaimer: This response is for informational purposes only "
            "and does not constitute medical advice. Always consult a qualified healthcare "
            "professional for diagnosis, treatment, or any health-related concerns."
        ),
    },
    {
        "domain":    "legal",
        "pattern":   r"(?i)\b(legal advice|attorney|lawsuit|litigation|court|statute|regulation|contract clause|liability|tort|jurisdiction|legal rights|criminal charges)\b",
        "threshold": 2,
        "disclaimer": (
            "⚠️ Legal Disclaimer: This response is for informational purposes only "
            "and does not constitute legal advice. Consult a qualified attorney for "
            "advice specific to your legal situation."
        ),
    },
    {
        "domain":    "financial",
        "pattern":   r"(?i)\b(invest|investment advice|buy|sell|stock|portfolio|returns|financial advice|tax advice|retirement|fund|market prediction|trading strategy)\b",
        "threshold": 2,
        "disclaimer": (
            "⚠️ Financial Disclaimer: This response is for informational purposes only "
            "and does not constitute financial or investment advice. Consult a qualified "
            "financial advisor before making investment decisions."
        ),
    },
    {
        "domain":    "crisis",
        "pattern":   r"(?i)\b(suicide|self.harm|harm myself|end my life|crisis|emergency|call 911|mental health crisis)\b",
        "threshold": 1,  # Even one match triggers this disclaimer
        "disclaimer": (
            "🆘 If you or someone you know is in crisis, please contact emergency services "
            "(911) or a crisis helpline (988 Suicide & Crisis Lifeline: call or text 988). "
            "This response is not a substitute for professional help."
        ),
    },
]


@dataclass
class DisclaimerResult:
    """Result of the disclaimer engine."""
    disclaimer_added: bool
    domain_detected:  str    # Which domain triggered the disclaimer
    disclaimer_text:  str
    final_text:       str    # Output with disclaimer prepended


class DisclaimerEngine:
    """
    Detects regulated or sensitive domains in LLM output and
    automatically prepends the required disclaimer.

    Governed domains:
      - Medical:   Any output containing diagnosis/treatment/medication terms
      - Legal:     Any output referencing legal advice or litigation
      - Financial: Any output with investment or financial advice terms
      - Crisis:    Any output referencing self-harm or mental health crisis

    Design decisions:
      1. Threshold-based triggering: requires multiple keyword matches
         to avoid false positives on incidental term use.
         Exception: crisis terms trigger on first match (safety-first).
      2. Disclaimer is prepended (not appended) so it is always seen first.
      3. Only the most severe domain disclaimer is added (no stacking).
         Priority: crisis > medical > legal > financial.
      4. This does not block delivery — disclaimers inform, not restrict.
    """

    def process(self, text: str) -> DisclaimerResult:
        """Check for regulated domain content and inject disclaimer if needed."""
        if not text or not text.strip():
            return DisclaimerResult(
                disclaimer_added=False, domain_detected="",
                disclaimer_text="", final_text=text or ""
            )

        for rule in DOMAIN_RULES:
            matches = re.findall(rule["pattern"], text, re.IGNORECASE)
            # Unique keyword count (don't count the same keyword twice)
            unique_matches = len(set(m.lower() for m in matches))

            if unique_matches >= rule["threshold"]:
                final_text = f"{rule['disclaimer']}\n\n{text}"
                logger.info(
                    f"Disclaimer injected: domain={rule['domain']} "
                    f"keyword_matches={unique_matches}"
                )
                return DisclaimerResult(
                    disclaimer_added = True,
                    domain_detected  = rule["domain"],
                    disclaimer_text  = rule["disclaimer"],
                    final_text       = final_text,
                )

        return DisclaimerResult(
            disclaimer_added=False, domain_detected="",
            disclaimer_text="", final_text=text
        )
```

---

## ⚙️ Step 8.10 — Testing the Output Scanning Layer

```python
# tests/unit/test_output_scanning.py
import pytest
from unittest.mock import AsyncMock, patch, MagicMock


# ─── Toxicity Scanner Tests ────────────────────────────────────────────────────

@pytest.mark.unit
@pytest.mark.governance
class TestToxicityScanner:

    async def test_safe_output_passes_toxicity_scan(self):
        from backend.output.toxicity_scanner import ToxicityScanner
        scanner = ToxicityScanner()

        with patch.object(scanner._openai_client.moderations, "create") as mock_mod:
            # Simulate OpenAI returning all-zero scores
            mock_result      = MagicMock()
            mock_result.results = [MagicMock()]
            mock_result.results[0].flagged = False
            scores = mock_result.results[0].category_scores
            for attr in ["hate", "hate_threatening", "harassment",
                         "harassment_threatening", "self_harm", "self_harm_intent",
                         "self_harm_instructions", "sexual", "sexual_minors",
                         "violence", "violence_graphic"]:
                setattr(scores, attr, 0.0)
            mock_mod.return_value = mock_result

            result = await scanner.scan("The refund policy allows 30 days for returns.")

        assert result.flagged    == False
        assert result.score      == 0.0
        assert result.findings   == []
        assert result.method     == "openai_moderation"

    async def test_toxic_output_is_flagged(self):
        from backend.output.toxicity_scanner import ToxicityScanner
        scanner = ToxicityScanner()

        with patch.object(scanner._openai_client.moderations, "create") as mock_mod:
            mock_result         = MagicMock()
            mock_result.results = [MagicMock()]
            mock_result.results[0].flagged = True
            scores = mock_result.results[0].category_scores
            for attr in ["hate", "hate_threatening", "harassment",
                         "harassment_threatening", "self_harm", "self_harm_intent",
                         "self_harm_instructions", "sexual", "sexual_minors",
                         "violence", "violence_graphic"]:
                setattr(scores, attr, 0.0)
            scores.hate = 0.95     # Highly toxic
            mock_mod.return_value = mock_result

            result = await scanner.scan("Some hateful content here.")

        assert result.flagged    == True
        assert result.score      >= 0.80
        assert len(result.findings) >= 1

    async def test_regex_fallback_fires_when_openai_unavailable(self):
        """When OpenAI API is unavailable, regex fallback should still detect obvious toxicity."""
        from backend.output.toxicity_scanner import ToxicityScanner
        scanner = ToxicityScanner()

        with patch.object(
            scanner._openai_client.moderations, "create",
            side_effect=Exception("Network timeout")
        ):
            result = await scanner.scan("I will kill you if you don't comply.")

        assert result.method == "regex_fallback"
        assert result.flagged == True

    async def test_empty_output_returns_zero_score(self):
        from backend.output.toxicity_scanner import ToxicityScanner
        scanner = ToxicityScanner()
        result  = await scanner.scan("")
        assert result.score  == 0.0
        assert result.flagged == False


# ─── Hallucination Detector Tests ─────────────────────────────────────────────

@pytest.mark.unit
@pytest.mark.governance
class TestHallucinationDetector:

    def test_detection_skipped_when_no_context(self):
        from backend.output.hallucination_detector import HallucinationDetector
        detector = HallucinationDetector()
        result   = detector.detect(
            output  = "The refund policy is 30 days.",
            context = None,
        )
        assert result.skipped            == True
        assert result.flagged            == False
        assert result.method             == "skipped_no_context"
        assert result.entailment_score   == 1.0   # Default: trusted

    def test_detection_skipped_for_short_output(self):
        from backend.output.hallucination_detector import HallucinationDetector
        detector = HallucinationDetector()
        result   = detector.detect(
            output  = "Yes.",
            context = ["The policy allows refunds within 30 days."],
        )
        assert result.skipped == True
        assert result.method  == "skipped_too_short"

    def test_detection_skipped_for_refusal(self):
        from backend.output.hallucination_detector import HallucinationDetector
        detector = HallucinationDetector()
        result   = detector.detect(
            output  = "I don't know the answer based on the provided context.",
            context = ["The document describes Q3 revenue results."],
        )
        assert result.skipped == True
        assert result.method  in ["skipped_refusal", "skipped_too_short"]

    def test_consistent_output_scores_high_entailment(self):
        """Output that accurately reflects context should score high entailment."""
        from backend.output.hallucination_detector import HallucinationDetector

        # Mock the cross-encoder to return high entailment
        with patch("backend.output.hallucination_detector._get_cross_encoder") as mock_enc:
            import numpy as np
            mock_model = MagicMock()
            # Return [contradiction=0.05, entailment=0.90, neutral=0.05]
            mock_model.predict.return_value = np.array([[0.05, 0.90, 0.05]])
            mock_enc.return_value = mock_model

            detector = HallucinationDetector()
            result   = detector.detect(
                output  = "The refund policy allows customers to return items within thirty days of purchase for a full refund.",
                context = ["Customers may return items within 30 days of purchase for a full refund."],
            )

        assert result.skipped          == False
        assert result.entailment_score >= 0.8
        assert result.flagged          == False
        assert result.warned           == False

    def test_contradicting_output_is_flagged(self):
        """Output that contradicts context should be flagged as hallucination."""
        from backend.output.hallucination_detector import HallucinationDetector

        with patch("backend.output.hallucination_detector._get_cross_encoder") as mock_enc:
            import numpy as np
            mock_model = MagicMock()
            # Return [contradiction=0.85, entailment=0.10, neutral=0.05]
            mock_model.predict.return_value = np.array([[0.85, 0.10, 0.05]])
            mock_enc.return_value = mock_model

            detector = HallucinationDetector()
            result   = detector.detect(
                output  = "The refund policy allows returns within 90 days and includes a 150% refund guarantee.",
                context = ["Customers may return items within 30 days for a full refund. No exceptions."],
            )

        assert result.skipped          == False
        assert result.entailment_score <  0.50
        assert result.flagged          == True


# ─── PII Output Scanner Tests ──────────────────────────────────────────────────

@pytest.mark.unit
@pytest.mark.governance
class TestPIIOutputScanner:

    def test_clean_output_is_unchanged(self):
        from backend.output.pii_output_scanner import PIIOutputScanner
        scanner = PIIOutputScanner()
        result  = scanner.scan("The refund policy is valid for 30 days after purchase.")
        assert result.pii_found     == False
        assert result.redacted_text == "The refund policy is valid for 30 days after purchase."
        assert result.redaction_count == 0

    def test_email_in_output_is_redacted(self):
        from backend.output.pii_output_scanner import PIIOutputScanner
        scanner = PIIOutputScanner()
        result  = scanner.scan("Please contact hr@company.com for HR queries.")
        assert result.pii_found     == True
        assert "hr@company.com"     not in result.redacted_text
        assert "[EMAIL REDACTED]"   in result.redacted_text
        assert result.redaction_count >= 1

    def test_ssn_in_output_is_redacted(self):
        from backend.output.pii_output_scanner import PIIOutputScanner
        scanner = PIIOutputScanner()
        result  = scanner.scan("Employee SSN on file: 123-45-6789.")
        assert result.pii_found      == True
        assert "123-45-6789"         not in result.redacted_text
        assert "[SSN REDACTED]"      in result.redacted_text

    def test_multiple_pii_types_all_redacted(self):
        from backend.output.pii_output_scanner import PIIOutputScanner
        scanner = PIIOutputScanner()
        text    = "Call John at 555-123-4567 or email john@co.com, SSN: 987-65-4321."
        result  = scanner.scan(text)
        assert result.pii_found         == True
        assert "john@co.com"            not in result.redacted_text
        assert "555-123-4567"           not in result.redacted_text
        assert "987-65-4321"            not in result.redacted_text
        assert result.redaction_count   >= 3


# ─── Disclaimer Engine Tests ───────────────────────────────────────────────────

@pytest.mark.unit
class TestDisclaimerEngine:

    def test_safe_output_gets_no_disclaimer(self):
        from backend.output.disclaimer_engine import DisclaimerEngine
        engine = DisclaimerEngine()
        result = engine.process("The office is open Monday to Friday, 9am to 5pm.")
        assert result.disclaimer_added == False
        assert result.domain_detected  == ""

    def test_medical_output_gets_disclaimer(self):
        from backend.output.disclaimer_engine import DisclaimerEngine
        engine = DisclaimerEngine()
        result = engine.process(
            "Based on the symptoms described, the diagnosis may indicate "
            "a need for medication or treatment. Please consult your physician."
        )
        assert result.disclaimer_added == True
        assert result.domain_detected  == "medical"
        assert "Medical Disclaimer"    in result.disclaimer_text
        assert result.final_text.startswith("⚠️ Medical Disclaimer")

    def test_financial_output_gets_disclaimer(self):
        from backend.output.disclaimer_engine import DisclaimerEngine
        engine = DisclaimerEngine()
        result = engine.process(
            "Based on current market trends, I would recommend to invest in "
            "diversified index funds as a trading strategy for your portfolio."
        )
        assert result.disclaimer_added == True
        assert result.domain_detected  == "financial"
        assert "Financial Disclaimer"  in result.disclaimer_text

    def test_single_incidental_keyword_no_disclaimer(self):
        """One incidental keyword should not trigger a disclaimer (threshold=2)."""
        from backend.output.disclaimer_engine import DisclaimerEngine
        engine = DisclaimerEngine()
        result = engine.process(
            "The doctor's appointment has been rescheduled to Thursday."
        )
        # "doctor" is only 1 match, threshold is 2
        assert result.disclaimer_added == False

    def test_crisis_output_triggers_on_single_match(self):
        """Crisis domain has threshold=1 — any single match triggers disclaimer."""
        from backend.output.disclaimer_engine import DisclaimerEngine
        engine = DisclaimerEngine()
        result = engine.process(
            "If you are experiencing a mental health crisis, here are some resources."
        )
        assert result.disclaimer_added == True
        assert result.domain_detected  == "crisis"
        assert "988" in result.disclaimer_text


# ─── OutputScanner Integration Tests ──────────────────────────────────────────

@pytest.mark.unit
@pytest.mark.governance
class TestOutputScannerIntegration:

    async def test_clean_output_passes_all_scanners(self, client):
        with patch("backend.output.toxicity_scanner.ToxicityScanner.scan") as mock_tox:
            from backend.output.toxicity_scanner import ToxicityResult
            mock_tox.return_value = ToxicityResult(
                score=0.0, flagged=False, categories={}, findings=[], method="openai_moderation"
            )
            response = await client.post(
                "/output/scan",
                json={"output": "The company vacation policy allows 20 days of PTO per year.",
                      "model_id": "gpt-4o"},
                headers={"X-User-ID": "alice", "X-User-Role": "employee"}
            )

        assert response.status_code == 200
        data = response.json()
        assert data["blocked"]   == False
        assert data["safe"]      == True
        assert data["scan_id"]   is not None

    async def test_toxic_output_is_blocked_by_scanner(self, client):
        with patch("backend.output.toxicity_scanner.ToxicityScanner.scan") as mock_tox:
            from backend.output.toxicity_scanner import ToxicityResult
            mock_tox.return_value = ToxicityResult(
                score=0.95, flagged=True,
                categories={"hate": 0.95}, findings=[{"category": "hate", "score": 0.95}],
                method="openai_moderation"
            )
            response = await client.post(
                "/output/scan",
                json={"output": "Toxic content here.", "model_id": "gpt-4o"},
                headers={"X-User-ID": "alice", "X-User-Role": "employee"}
            )

        assert response.status_code == 200
        data = response.json()
        assert data["blocked"]        == True
        assert data["toxicity_score"] >= 0.80

    async def test_output_stats_returns_expected_shape(self, client):
        response = await client.get("/output/stats?days=7")
        assert response.status_code == 200
        data = response.json()
        assert "total_scans" in data
        assert "by_action"   in data
        assert "scores"      in data
        assert "rates"       in data

    async def test_output_history_is_paginated(self, client):
        response = await client.get("/output/history?limit=10&offset=0")
        assert response.status_code == 200
        assert isinstance(response.json(), list)

    async def test_pii_in_output_is_redacted_not_blocked(self, client):
        """
        GOVERNANCE: PII in output should be REDACTED (output still delivered),
        not blocked. The response has value — the PII is the only problem.
        """
        with patch("backend.output.toxicity_scanner.ToxicityScanner.scan") as mock_tox:
            from backend.output.toxicity_scanner import ToxicityResult
            mock_tox.return_value = ToxicityResult(
                score=0.0, flagged=False, categories={}, findings=[], method="openai_moderation"
            )
            response = await client.post(
                "/output/scan",
                json={
                    "output":   "John's email is john@company.com and his SSN is 123-45-6789.",
                    "model_id": "gpt-4o"
                },
                headers={"X-User-ID": "alice", "X-User-Role": "employee"}
            )

        data = response.json()
        assert data["blocked"]      == False
        assert data["pii_found"]    == True
        assert data["action"]       == "redacted"
        # PII must not appear in delivered output
        assert "john@company.com"   not in data["output"]
        assert "123-45-6789"        not in data["output"]
```

---

## 🏁 What You Built Today

```
Week 2 Progress:
  Day 6  → GovernedLLMClient: real OpenAI calls through governance
  Day 7  → GovernedRAG: governed document indexing + retrieval
  Day 8  → Output Scanning: hallucination + toxicity detection  ← TODAY
  Day 9  → Governed Streaming: SSE endpoint with governance middleware

New capabilities added:
  ✅ ToxicityScanner — OpenAI Moderation API primary, regex fallback
  ✅ 11 toxicity categories: hate, harassment, self-harm, sexual, violence
  ✅ HallucinationDetector — NLI cross-encoder, sentence-level entailment scoring
  ✅ Skips detection gracefully: no context, short output, refusals
  ✅ PIIOutputScanner — detects and redacts 7 PII types in LLM output
  ✅ DisclaimerEngine — auto-injects for medical, legal, financial, crisis domains
  ✅ OutputScanner orchestrator — chains all scanners, unified result
  ✅ OutputScanResult PostgreSQL model — permanent audit record of every scan
  ✅ GovernedLLMClient updated — output scanning fires after every LLM call
  ✅ POST /output/scan     — manual scan endpoint
  ✅ GET  /output/stats    — toxicity rates, hallucination rates, block rates
  ✅ GET  /output/history  — paginated scan history for compliance reporting
  ✅ Confidence scoring — every response gets toxicity + hallucination composite
  ✅ 15+ tests covering all scanners and governance invariants

```

### The Complete Governance Stack After Day 8

```
COMPLETE GOVERNANCE PIPELINE (Days 1-8)
  ┌──────────────────────────────────────────────────────────────────┐
  │  User Question                                                   │
  │        │                                                         │
  │  [Rate Limit]          ← DAY 4: Redis sliding window            │
  │        │                                                         │
  │  [Input Policy Check]  ← DAY 2: PII, injection, role rules      │
  │        │ CLEAN                                                   │
  │  [RAG Retrieval]       ← DAY 7: role-filtered, scanned context  │
  │        │                                                         │
  │  [Token Budget]        ← DAY 6: cost enforcement                │
  │        │                                                         │
  │  [LLM Cache]           ← DAY 4: identical prompts               │
  │        │ MISS                                                    │
  │  [LLM Call]            ← DAY 6: OpenAI / Anthropic / Gemini     │
  │        │                                                         │
  │  ┌─────▼────────────────────────────────────────────────────┐   │
  │  │  OUTPUT SCANNING PIPELINE (DAY 8)                        │   │
  │  │                                                          │   │
  │  │  [Toxicity Scanner]      BLOCK if hate/violence/sexual   │   │
  │  │  [Hallucination Detect]  BLOCK if contradicts context    │   │
  │  │  [PII Output Scanner]    REDACT emails/SSNs/cards        │   │
  │  │  [Disclaimer Engine]     INJECT for medical/legal/fin    │   │
  │  │  [Scan Log → PostgreSQL] RECORD every scan result        │   │
  │  └─────┬────────────────────────────────────────────────────┘   │
  │        │                                                         │
  │  [Cost Track]          ← DAY 6: tokens × price                  │
  │  [Audit Log]           ← DAY 3: immutable event record          │
  │        │                                                         │
  │  Safe Response to User                                           │
  │  { output, confidence_score, scan_id, toxicity_score,           │
  │    hallucination_score, pii_redacted, disclaimer_added }         │
  └──────────────────────────────────────────────────────────────────┘

Every response now carries a confidence score (0-1).
  1.0 = fully consistent with context, no toxicity, no PII
  0.5 = some uncertainty — hallucination warning was issued
  0.0 = blocked (never delivered to user)

```

---

## ✅ Day 8 Checklist

Before moving to Day 9, verify all of these:

- [ ] `pip install sentence-transformers detoxify torch` succeeded
- [ ] Cross-encoder model downloads on first run (~86MB): no error in logs
- [ ] `OPENAI_API_KEY` is set — OpenAI Moderation API is being called (check logs)
- [ ] `POST /output/scan` with safe text returns `blocked=false, action=allowed`
- [ ] `POST /output/scan` with text mocking high toxicity returns `blocked=true`
- [ ] `POST /output/scan` with email/SSN in text returns `pii_found=true, action=redacted`
- [ ] Redacted output does NOT contain the original email/SSN
- [ ] Medical output gets `"Medical Disclaimer"` prepended automatically
- [ ] Financial output gets `"Financial Disclaimer"` prepended
- [ ] Crisis keywords trigger disclaimer even with single match
- [ ] Hallucination detection is skipped when `context=null` (no RAG context)
- [ ] Hallucination detection is skipped for outputs under 15 words
- [ ] `alembic upgrade head` created `output_scan_results` table
- [ ] `GET /output/stats` returns correct counts and rates after scans
- [ ] `GET /output/history?action=blocked` returns only blocked scans
- [ ] GovernedLLMClient `.complete()` now includes `scan_id` in the response
- [ ] A blocked output still charges the tokens (tokens were consumed)
- [ ] All previous tests still pass: `pytest -m unit` → 0 failures
- [ ] New tests pass: `pytest tests/unit/test_output_scanning.py -v`
- [ ] Coverage stays above 90%: `pytest --cov=backend --cov-fail-under=90`

---

## 📚 What to Read Tonight

| Resource | Topic |
|---|---|
| [OpenAI Moderation API Docs](https://platform.openai.com/docs/guides/moderation) | Category definitions and score thresholds |
| [Cross-Encoders (SBERT)](https://www.sbert.net/docs/pretrained-models/ce-msmarco.html) | NLI cross-encoder models for entailment scoring |
| [TrustedAI — Hallucination Survey](https://arxiv.org/abs/2309.01219) | Survey of LLM hallucination types and mitigation |
| [NIST AI RMF — Measure 2.5](https://airc.nist.gov/Docs/1) | Measuring AI output quality and safety |
| [EU AI Act — Article 9](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A52021PC0206) | Risk management obligations for high-risk AI |
| [Perspective API](https://perspectiveapi.com/) | Google's toxicity detection API (alternative to OpenAI Moderation) |

---

## 🔮 Coming Up

```
Day 9  → Governed Streaming: SSE endpoint with governance middleware
Day 10 → End-to-End Test: prompt → policy → RAG → LLM → output scan → audit
Day 11 → Dashboard: real-time governance metrics with WebSockets
```

> Day 9 is streaming with governance.
> Every token that streams out of the LLM passes through a real-time
> output scanner. The challenge: you can't wait for the full response
> before scanning — you get tokens one at a time.
>
> The solution: dual-mode scanning.
> Per-token: lightweight regex for obvious toxicity.
> Post-stream: full NLI hallucination check on the assembled output.
> If the post-stream check fails, the connection is closed and
> the client receives a retraction message.

---

*Day 8 complete. The governance stack now covers both ends of the LLM call.*

*Input is checked before the LLM sees it. Output is scanned before the user sees it. Every toxic response is blocked. Every hallucination gets a confidence score. Every PII leak is redacted. Every medical or financial response carries a required disclaimer.*

*You now have a complete audit trail of every governance intervention — both input and output — in PostgreSQL.*

---

**Repository:** [GuntruTirupathamma/AI-Governance](https://github.com/GuntruTirupathamma/AI-Governance)  **Series:** AI Governance Engineering from Scratch  **Next:** `DAY9.md` → Governed Streaming: SSE Endpoint with Governance Middleware
