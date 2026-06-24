# MedMind — Memory Extraction Pipeline Design

**Date:** June 24, 2026  
**Status:** Approved — Ready for Implementation  
**Track:** Track 1 — MemoryAgent  

---

## 1. Overview

This document specifies the design for MedMind's memory extraction pipeline — the core system that turns raw chat conversations into structured, retrievable health memories. This is the most technically complex component of the project and the primary focus of hackathon judging.

---

## 2. Local Development Architecture

All infrastructure runs locally during development. Alibaba Cloud deployment happens only after the app works end-to-end on the developer's machine.

| Component | Local | Production (Later) |
|---|---|---|
| **Database** | Docker: PostgreSQL + pgvector | Alibaba Cloud RDS PostgreSQL |
| **Backend** | Python + FastAPI (localhost) | Alibaba Cloud ECS |
| **Frontend** | Next.js via `npm run dev` | Vercel or Alibaba Cloud |
| **LLM** | Qwen Cloud API (remote calls) | Qwen Cloud API (same) |
| **Embeddings** | Qwen `text-embedding-v3` at 1024 dimensions | Same |

> **Rationale:** The only difference between local and production is the database connection string. Everything else is identical, so there are no "works on my machine" surprises.

---

## 3. Pipeline Flow

The pipeline runs **asynchronously** after each chat response, so the user never waits for extraction to complete.

```
User sends message
        │
        ▼
┌──────────────────────────┐
│  Chat LLM (Qwen-Plus)   │
│  Generates response +    │
│  sets trigger flag:      │
│  contains_new_health_fact│
└──────────┬───────────────┘
           │
     ┌─────┴─────┐
     │  flag =   │
     │  true?    │
     └─────┬─────┘
       no  │  yes
       │   │
       ▼   ▼
    DONE  Background Task kicks off
              │
              ▼
    ┌─────────────────────┐
    │ Acquire Session Lock│ (wait if another task is running for this user)
    └─────────┬───────────┘
              │
              ▼
    ┌─────────────────────┐
    │ BEGIN DB Transaction │
    └─────────┬───────────┘
              │
              ▼
    ┌─────────────────────────────────┐
    │ Fact Extraction (Qwen-Plus)     │
    │ Input: last N turns of convo    │
    │        + AI's paraphrased reply │
    │ Output: structured JSON array   │
    └─────────┬───────────────────────┘
              │
              ▼
    ┌─────────────────────────────────┐
    │ JSON Validation                 │
    │ - Parse with try/except         │
    │ - On failure: retry once with   │
    │   stricter prompt               │
    │ - On second failure: log raw    │
    │   output and skip (no data >    │
    │   corrupted data)               │
    └─────────┬───────────────────────┘
              │
              ▼
    ┌─────────────────────────────────┐
    │ Fact Filtering                  │
    │ - Verify source_quote exists    │
    │   in transcript                 │
    │ - Discard subject: "other"      │
    │ - Discard intent: "hypothetical"│
    │   or "question"                 │
    │ - Score confidence              │
    │ - Low confidence → force        │
    │   Ephemeral tier                │
    └─────────┬───────────────────────┘
              │
              ▼
    ┌─────────────────────────────────┐
    │ Dedup & Conflict Check          │
    │ - Semantic search existing DB   │
    │ - If conflict on Critical tier: │
    │   DO NOT update, flag for       │
    │   AI verification next session  │
    │ - If new: insert                │
    │ - If duplicate: skip            │
    │ - If update confirmed: mark old │
    │   record as "superseded",       │
    │   create new version            │
    └─────────┬───────────────────────┘
              │
              ▼
    ┌─────────────────────────────────┐
    │ Embed & Store                   │
    │ - Generate vector via Qwen      │
    │   text-embedding-v3 (1024 dim)  │
    │ - Insert into PostgreSQL +      │
    │   pgvector                      │
    │ - Record last_processed_msg_id  │
    └─────────┬───────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │ COMMIT Transaction  │
    └─────────┬───────────┘
              │
              ▼
    ┌─────────────────────────────────┐
    │ Forgetting Sweep                │
    │ - Delete expired Ephemeral      │
    │   memories (>30 days, 0 access) │
    │ - Decay Contextual relevance    │
    │ - Compress old Important tier   │
    └─────────┬───────────────────────┘
              │
              ▼
    ┌─────────────────────────────────┐
    │ Pattern Detection               │
    │ (runs every 5 sessions)         │
    │ - Group symptoms by type        │
    │ - Check frequency/severity      │
    │   trends over time              │
    │ - If escalation detected:       │
    │   create proactive_alert memory │
    └─────────────────────────────────┘
```

---

## 4. Extraction Output Schema

Each extracted fact from the LLM must conform to this JSON structure:

```json
{
  "fact": "Allergic to penicillin — causes skin rash",
  "category": "allergy",
  "tier": "critical",
  "subject": "self",
  "intent": "reported",
  "status": "active",
  "confidence": "high",
  "source_quote": "I break out in hives if I take penicillin",
  "action_required": false
}
```

### Field Definitions

| Field | Values | Purpose |
|---|---|---|
| `fact` | Free text | The clean, standardized medical fact |
| `category` | `allergy`, `medication`, `symptom`, `family_history`, `condition`, `lifestyle`, `lab_result`, `surgery`, `follow_up` | What kind of health information this is |
| `tier` | `critical`, `important`, `contextual`, `ephemeral` | Memory priority tier (see PROJECT_SPEC.md §3.1) |
| `subject` | `self`, `family_member`, `other` | Who this fact is about. `other` is discarded |
| `intent` | `reported`, `hypothetical`, `question` | Was the user reporting a fact, asking a hypothetical, or asking a question? Only `reported` is stored |
| `status` | `active`, `resolved`, `historical` | Is this condition current, resolved, or historical? |
| `confidence` | `high`, `medium`, `low` | How confident is the extraction? Emotional/vague language → `low`. Low-confidence facts default to Ephemeral tier |
| `source_quote` | Exact substring from transcript | Proof the fact was actually said. Must match transcript text |
| `action_required` | `true` / `false` | Should MedMind follow up on this? (e.g., user said "I should get that test done") |

### Family Member Normalization

When `subject` is `family_member`, the extraction prompt instructs Qwen to normalize all references to standard labels:

| User says | Normalized to |
|---|---|
| "my mom", "my mum", "my mother" | `mother` |
| "my dad", "my father", "my papa" | `father` |
| "my brother", "my bro" | `sibling_brother` |
| "my sister", "my sis" | `sibling_sister` |
| "my grandma on my mom's side" | `maternal_grandmother` |
| "my grandpa on my dad's side" | `paternal_grandfather` |

> **Decision:** The LLM handles this normalization directly in the extraction prompt. No custom NLP code needed — the model is better at understanding conversational family references than any regex.

---

## 5. Safeguards (13 Total)

### Safeguard 1 — Trigger Flag (Credit Conservation)
**Problem:** Running extraction after every message wastes API credits. Hackathon has a $40 Qwen Cloud limit.  
**Solution:** The Chat LLM outputs a hidden `contains_new_health_fact: true/false` flag alongside its response. Background extraction only runs when `true`.

### Safeguard 2 — Source Quote Verification (Anti-Hallucination)
**Problem:** LLMs can hallucinate facts that were never mentioned.  
**Solution:** Every extracted fact must include a `source_quote` — an exact substring from the conversation transcript. Before saving, the backend verifies this quote actually exists in the chat history. If it doesn't, the fact is discarded.

### Safeguard 3 — Subject Classification (Third-Party Attribution)
**Problem:** "My friend was diagnosed with lupus" could be stored as the user having lupus.  
**Solution:** Each fact has a `subject` field: `self`, `family_member`, or `other`. Only `self` and `family_member` facts are stored. `other` is discarded entirely.

### Safeguard 4 — AI-Driven Confirmation for Critical Tier (Accidental Overwrites)
**Problem:** User might mistype a dosage or medication name. Blindly updating Critical data is dangerous.  
**Solution:** When the Chat LLM detects a discrepancy between its context (existing memories) and the user's message, it asks the user to confirm before the pipeline updates the record. The background extractor will NOT update Critical tier records unless explicit user confirmation is present in the transcript.

### Safeguard 5 — Session-Level Database Lock (Race Conditions)
**Problem:** Fast sequential messages could trigger overlapping background tasks that read/write the same rows simultaneously.  
**Solution:** A PostgreSQL advisory lock per user session. If an extraction task is already running for a user, new tasks wait until the current one finishes.

### Safeguard 6 — Database Transaction + Crash Recovery
**Problem:** Server crash mid-extraction could save some facts but lose others, leaving the database in an inconsistent state.  
**Solution:** The entire extract-and-save flow is wrapped in a single database transaction (all-or-nothing). We also record `last_processed_message_id` per session. On restart, any unprocessed messages are re-extracted.

### Safeguard 7 — Token-Counted Retrieval (Context Overflow)
**Problem:** A fixed top-k retrieval could either overflow the context window or leave space unused.  
**Solution:** Instead of retrieving a fixed number of memories, we iterate through ranked results, accumulating token count, and stop when we hit the ~1,000 token budget. This guarantees maximum context packed into the limit without ever exceeding it.

### Safeguard 8 — Versioned Memories (Change History)
**Problem:** If we overwrite a medication dosage, we lose the historical record.  
**Solution:** Updates mark the old fact as `superseded` (with a timestamp) and create a new record. The retrieval system loads only the latest version, but the full history is queryable. This also supports the Doctor-Ready Summary feature showing medication change timelines.

### Safeguard 9 — Intent Classification (Hypotheticals)
**Problem:** "What would happen if I stopped taking my medication?" could be stored as "User stopped taking medication."  
**Solution:** Each extracted fact has an `intent` field: `reported`, `hypothetical`, or `question`. Only `reported` facts are stored. The other two are discarded.

### Safeguard 10 — Temporal Status (Past vs. Present)
**Problem:** "I used to have asthma as a kid" stored as a current condition leads to incorrect drug interaction warnings.  
**Solution:** Each fact has a `status` field: `active`, `resolved`, or `historical`. Resolved conditions are stored (doctors may want to know) but excluded from active drug interaction checks.

### Safeguard 11 — Confidence Scoring (Emotional Language)
**Problem:** "This headache is killing me, I feel like I'm dying" gets misinterpreted as high clinical severity.  
**Solution:** Each fact gets a `confidence` score: `high`, `medium`, or `low`. Casual, emotional, or vague language is scored `low`. Low-confidence facts are automatically assigned to the Ephemeral tier regardless of category, preventing them from polluting Critical or Important tiers.

### Safeguard 12 — Dual-Source Extraction (Multi-Language)
**Problem:** Users may code-switch between languages (e.g., English and Pidgin: "My body dey hot since yesterday"). The extractor might fail to parse non-English medical terms.  
**Solution:** The extraction prompt is given both the user's raw message AND the AI's paraphrased response. The AI response will contain the standardized medical interpretation ("You mentioned you've had a fever since yesterday"), which is much more reliably extractable.

### Safeguard 13 — Periodic Pattern Detection (Gradual Escalation)
**Problem:** Individual sessions seem minor, but symptoms are gradually escalating over months. No single message triggers an alarm.  
**Solution:** A pattern detection sweep runs every **5 sessions**. It queries all symptoms for the user, groups them by type, and checks for frequency/severity trends. If escalation is detected, it creates a `proactive_alert` memory injected into the next session context.

---

## 6. Memory Retrieval Strategy

When building the prompt for a new chat session:

```
System Prompt (~500 tokens)
        +
Critical Memories — always loaded (~1,000 tokens max)
  - All allergies
  - All current medications (status: active only)
  - All chronic conditions (status: active only)
  - Family history summary
        +
Retrieved Memories — semantic search (~1,000 tokens, token-counted)
  - Ranked by: (relevance_score × recency_weight × tier_priority)
  - Loaded until token budget is exhausted
        +
Proactive Alerts — if any pending (~200 tokens)
  - Follow-up reminders
  - Escalation alerts from pattern detection
        +
Current Session History (~2,000 tokens)
  - Last N turns of the current conversation
        +
User Message + Response Generation (remaining tokens)
```

---

## 7. Resolved Design Decisions

| Question | Decision | Rationale |
|---|---|---|
| **Qwen JSON reliability** | `try/except` with one retry, then skip | No data is better than corrupted data. Log failures for debugging. |
| **Embedding model** | Qwen `text-embedding-v3`, 1024 dimensions | Latest model, good quality for medical text, supports variable dimensions. Pick once, never change mid-project. |
| **Pattern detection frequency** | Every 5 sessions | Judges won't wait a real week. Session-based works whether user visits daily or monthly. |
| **Family member normalization** | LLM handles it in the extraction prompt | Model understands "my mum" = `mother` better than any regex. No custom NLP code needed. |

---

## 8. Next Steps

1. Write the full implementation plan (all components, not just the pipeline)
2. Set up local development environment (Docker + FastAPI + Next.js)
3. Begin Phase 1: Foundation
