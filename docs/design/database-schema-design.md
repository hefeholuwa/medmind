# MedMind — Database Schema Design

**Date:** June 24, 2026  
**Status:** Approved — Ready for Implementation  
**Database:** PostgreSQL 16 + pgvector extension  
**Local Setup:** Docker container  

---

## 1. Overview

This document defines every table, relationship, and index in MedMind's PostgreSQL database. The schema supports the memory extraction pipeline, family health graph, proactive follow-ups, and pattern detection — all features described in the project spec.

### Required Extensions

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";   -- UUID generation
CREATE EXTENSION IF NOT EXISTS "vector";       -- pgvector for embeddings
```

---

## 2. Entity Relationship Diagram

```
┌──────────┐       ┌──────────────┐       ┌────────────┐
│  users   │──1:N──│   sessions   │──1:N──│  messages   │
└────┬─────┘       └──────────────┘       └────────────┘
     │                                          │
     │ 1:N                                      │ (source)
     │                                          │
     ├──────────────┬───────────────────────────┤
     │              │                           │
     ▼              ▼                           ▼
┌──────────┐  ┌───────────────┐         ┌──────────────┐
│ memories │  │family_members │         │  (memories    │
│          │  │               │         │   linked via  │
│          │  └───────────────┘         │   source_     │
│          │                            │   message_id) │
└────┬─────┘                            └──────────────┘
     │
     │ 1:N (referenced by)
     │
     ├──────────────┐
     │              │
     ▼              ▼
┌──────────┐  ┌───────────────┐
│follow_ups│  │pattern_alerts │
└──────────┘  └───────────────┘
```

---

## 3. Tables

### 3.1 `users`

Basic user accounts. Minimal auth — not the focus of the project.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY, DEFAULT uuid_generate_v4() | |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | Login identifier |
| `password_hash` | VARCHAR(255) | NOT NULL | bcrypt hashed |
| `display_name` | VARCHAR(100) | | Optional friendly name |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `last_active_at` | TIMESTAMPTZ | | Updated on each login/session |

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    display_name    VARCHAR(100),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    last_active_at  TIMESTAMPTZ
);
```

---

### 3.2 `sessions`

Each conversation session. Tracks extraction state for crash recovery (Pipeline Safeguard 6).

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | |
| `user_id` | UUID | FK → users(id), NOT NULL | |
| `started_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `ended_at` | TIMESTAMPTZ | | NULL if session is still active |
| `last_processed_message_id` | UUID | | For crash recovery — last message the pipeline finished processing |
| `session_number` | INTEGER | NOT NULL | Per-user counter. Pattern detection runs every 5 sessions |

```sql
CREATE TABLE sessions (
    id                          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id                     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    started_at                  TIMESTAMPTZ DEFAULT NOW(),
    ended_at                    TIMESTAMPTZ,
    last_processed_message_id   UUID,
    session_number              INTEGER NOT NULL
);

CREATE INDEX idx_sessions_user ON sessions(user_id);
```

---

### 3.3 `messages`

Every message in every session. The extraction pipeline reads from this table.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | |
| `session_id` | UUID | FK → sessions(id), NOT NULL | |
| `role` | VARCHAR(10) | NOT NULL | `'user'` or `'assistant'` |
| `content` | TEXT | NOT NULL | Raw message text |
| `contains_health_fact` | BOOLEAN | DEFAULT FALSE | Trigger flag (Pipeline Safeguard 1). Set by Chat LLM |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `sequence_num` | INTEGER | NOT NULL | Ordering within session |

```sql
CREATE TABLE messages (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    session_id          UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    role                VARCHAR(10) NOT NULL CHECK (role IN ('user', 'assistant')),
    content             TEXT NOT NULL,
    contains_health_fact BOOLEAN DEFAULT FALSE,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    sequence_num        INTEGER NOT NULL
);

CREATE INDEX idx_messages_session ON messages(session_id, sequence_num);
```

> **Decision:** We keep full chat history permanently for the hackathon. In production, we would add a retention policy.

---

### 3.4 `memories` (Core Table)

The heart of MedMind. Every field maps directly to the extraction pipeline's output schema defined in `memory-extraction-pipeline-design.md §4`.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | |
| `user_id` | UUID | FK → users(id), NOT NULL | |
| `fact` | TEXT | NOT NULL | Clean, standardized medical fact |
| `category` | VARCHAR(20) | NOT NULL | See allowed values below |
| `tier` | VARCHAR(15) | NOT NULL | `critical`, `important`, `contextual`, `ephemeral` |
| `subject` | VARCHAR(15) | NOT NULL | `self` or `family_member` (pipeline discards `other`) |
| `status` | VARCHAR(15) | NOT NULL, DEFAULT 'active' | `active`, `resolved`, `historical` |
| `confidence` | VARCHAR(10) | NOT NULL | `high`, `medium`, `low` |
| `source_quote` | TEXT | NOT NULL | Exact transcript quote (Safeguard 2) |
| `source_message_id` | UUID | FK → messages(id) | Which message this fact came from |
| `action_required` | BOOLEAN | DEFAULT FALSE | Creates a follow_up if true |
| `family_member_label` | VARCHAR(30) | | NULL if subject = 'self'. Normalized label (e.g., `mother`) |
| `embedding` | VECTOR(1024) | | pgvector. Qwen text-embedding-v3 |
| `version` | INTEGER | DEFAULT 1 | Increments on updates (Safeguard 8) |
| `superseded_by` | UUID | FK → memories(id) | NULL if this is the current version |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `last_accessed_at` | TIMESTAMPTZ | | Updated each time retrieval loads this memory |
| `access_count` | INTEGER | DEFAULT 0 | Used for forgetting decay calculations |
| `expires_at` | TIMESTAMPTZ | | Calculated from tier TTL rules. NULL for critical tier |

#### Allowed `category` values:
`allergy`, `medication`, `symptom`, `family_history`, `condition`, `lifestyle`, `lab_result`, `surgery`, `follow_up`

```sql
CREATE TABLE memories (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    fact                TEXT NOT NULL,
    category            VARCHAR(20) NOT NULL CHECK (category IN (
                            'allergy', 'medication', 'symptom', 'family_history',
                            'condition', 'lifestyle', 'lab_result', 'surgery', 'follow_up'
                        )),
    tier                VARCHAR(15) NOT NULL CHECK (tier IN (
                            'critical', 'important', 'contextual', 'ephemeral'
                        )),
    subject             VARCHAR(15) NOT NULL CHECK (subject IN ('self', 'family_member')),
    status              VARCHAR(15) NOT NULL DEFAULT 'active' CHECK (status IN (
                            'active', 'resolved', 'historical'
                        )),
    confidence          VARCHAR(10) NOT NULL CHECK (confidence IN ('high', 'medium', 'low')),
    source_quote        TEXT NOT NULL,
    source_message_id   UUID REFERENCES messages(id),
    action_required     BOOLEAN DEFAULT FALSE,
    family_member_label VARCHAR(30),
    embedding           VECTOR(1024),
    version             INTEGER DEFAULT 1,
    superseded_by       UUID REFERENCES memories(id),
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    last_accessed_at    TIMESTAMPTZ,
    access_count        INTEGER DEFAULT 0,
    expires_at          TIMESTAMPTZ
);
```

---

### 3.5 `family_members`

The family health graph. Each row is a unique family member for a user.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | |
| `user_id` | UUID | FK → users(id), NOT NULL | |
| `label` | VARCHAR(30) | NOT NULL | Normalized: `mother`, `father`, `sibling_brother`, `paternal_grandfather`, etc. |
| `display_name` | VARCHAR(100) | | Optional friendly name: "Mom", "Uncle James" |
| `alive` | BOOLEAN | | NULL if unknown |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | |

> **How it connects:** Family members' health conditions are stored as regular `memories` rows with `subject = 'family_member'` and `family_member_label` matching the `label` here. No separate health table needed — the memories table handles it.

```sql
CREATE TABLE family_members (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    label           VARCHAR(30) NOT NULL,
    display_name    VARCHAR(100),
    alive           BOOLEAN,
    created_at      TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(user_id, label)
);
```

---

### 3.6 `follow_ups`

Proactive reminders. Created when the pipeline detects `action_required: true`.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | |
| `user_id` | UUID | FK → users(id), NOT NULL | |
| `source_memory_id` | UUID | FK → memories(id) | The memory that triggered this follow-up |
| `reminder_text` | TEXT | NOT NULL | "Ask if user completed cholesterol test" |
| `due_after` | TIMESTAMPTZ | NOT NULL | When to start reminding |
| `status` | VARCHAR(15) | DEFAULT 'pending' | `pending`, `delivered`, `resolved` |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `resolved_at` | TIMESTAMPTZ | | When user confirmed they did it |

```sql
CREATE TABLE follow_ups (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    source_memory_id    UUID REFERENCES memories(id),
    reminder_text       TEXT NOT NULL,
    due_after           TIMESTAMPTZ NOT NULL,
    status              VARCHAR(15) DEFAULT 'pending' CHECK (status IN (
                            'pending', 'delivered', 'resolved'
                        )),
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    resolved_at         TIMESTAMPTZ
);

CREATE INDEX idx_follow_ups_pending ON follow_ups(user_id, status)
    WHERE status = 'pending';
```

---

### 3.7 `pattern_alerts`

Created by the pattern detection sweep that runs every 5 sessions (Pipeline Safeguard 13).

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | |
| `user_id` | UUID | FK → users(id), NOT NULL | |
| `alert_type` | VARCHAR(20) | NOT NULL | `escalation`, `frequency_increase`, `new_pattern` |
| `description` | TEXT | NOT NULL | "Headache frequency increased over 3 months" |
| `related_memory_ids` | UUID[] | | Array of memory IDs that form the pattern |
| `status` | VARCHAR(15) | DEFAULT 'pending' | `pending`, `delivered`, `dismissed` |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `delivered_at` | TIMESTAMPTZ | | When the alert was shown to the user |

```sql
CREATE TABLE pattern_alerts (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    alert_type          VARCHAR(20) NOT NULL CHECK (alert_type IN (
                            'escalation', 'frequency_increase', 'new_pattern'
                        )),
    description         TEXT NOT NULL,
    related_memory_ids  UUID[],
    status              VARCHAR(15) DEFAULT 'pending' CHECK (status IN (
                            'pending', 'delivered', 'dismissed'
                        )),
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    delivered_at        TIMESTAMPTZ
);

CREATE INDEX idx_pattern_alerts_pending ON pattern_alerts(user_id, status)
    WHERE status = 'pending';
```

---

## 4. Key Indexes

These indexes are critical for performance of the pipeline and retrieval system.

```sql
-- Fast memory retrieval: current (non-superseded) memories for a user, filtered by tier
CREATE INDEX idx_memories_user_tier
    ON memories(user_id, tier)
    WHERE superseded_by IS NULL;

-- Vector similarity search (pgvector ivfflat index)
-- Note: ivfflat requires training data. Create AFTER inserting initial memories.
-- For early development, use exact search (no index). Add ivfflat when data grows.
CREATE INDEX idx_memories_embedding
    ON memories USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

-- Forgetting sweep: find expired memories efficiently
CREATE INDEX idx_memories_expiry
    ON memories(expires_at)
    WHERE superseded_by IS NULL AND expires_at IS NOT NULL;

-- Messages ordered within a session (for retrieval and extraction)
CREATE INDEX idx_messages_session_order
    ON messages(session_id, sequence_num);

-- Pending follow-ups for a user (loaded into system prompt)
CREATE INDEX idx_follow_ups_pending
    ON follow_ups(user_id, status)
    WHERE status = 'pending';

-- Pending pattern alerts (loaded into system prompt)
CREATE INDEX idx_pattern_alerts_pending
    ON pattern_alerts(user_id, status)
    WHERE status = 'pending';
```

---

## 5. TTL Rules (Forgetting)

The `expires_at` column is calculated when a memory is created, based on its tier:

| Tier | TTL | `expires_at` calculation | Forgetting behavior |
|---|---|---|---|
| **Critical** | Never | `NULL` | Never deleted. Only user can manually correct. |
| **Important** | 2 years | `created_at + INTERVAL '2 years'` | Compressed after 6 months (merge similar entries). Deleted after 2 years if `access_count = 0`. |
| **Contextual** | 6 months | `created_at + INTERVAL '6 months'` | Relevance decays if not accessed. Archived after 6 months with zero access. |
| **Ephemeral** | 30 days | `created_at + INTERVAL '30 days'` | Auto-deleted after 30 days with no access. |

> **Note:** Accessing a memory (loading it into the prompt) resets `last_accessed_at` and increments `access_count`. Frequently accessed memories survive longer.

---

## 6. Docker Setup (Local Development)

```yaml
# docker-compose.yml (for local dev)
version: '3.8'
services:
  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: medmind
      POSTGRES_USER: medmind
      POSTGRES_PASSWORD: medmind_dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

**Connection string (local):**
```
postgresql://medmind:medmind_dev@localhost:5432/medmind
```

**Connection string (production — Alibaba Cloud RDS):**
```
postgresql://<user>:<password>@<rds-endpoint>:5432/medmind
```

> The only thing that changes between local and production is this connection string.

---

## 7. Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Chat history retention** | Keep everything permanently | Simpler for hackathon. Add retention policy for production later. |
| **Family health storage** | Reuse `memories` table with `subject = 'family_member'` | Avoids a separate table. Family conditions get the same embedding, retrieval, and versioning as personal memories. |
| **pgvector index type** | `ivfflat` (add later, not at init) | Exact search is fine for small datasets. Add the index once we have enough data to train it. |
| **UUID generation** | Database-side via `uuid_generate_v4()` | Consistent across all insertion paths (API, pipeline, scripts). |
| **Cascade deletes** | `ON DELETE CASCADE` from users | If a user deletes their account, everything goes with them. Clean data hygiene. |

---

## 8. Next Steps

1. Design remaining components (Frontend, API Endpoints, System Prompt, Auth)
2. Write the full implementation plan
3. Set up local development environment using the Docker config above
4. Run the SQL migration to create all tables
