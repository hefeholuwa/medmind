# MedMind — API Endpoints Design

**Date:** June 24, 2026  
**Status:** Approved — Ready for Implementation  
**Framework:** Python FastAPI  
**Auth:** JWT tokens (7-day expiry, no refresh tokens)  
**Streaming:** Server-Sent Events (SSE) for chat responses  

---

## 1. Overview

This document defines every API endpoint for MedMind's FastAPI backend. All endpoints are prefixed with `/api`. All authenticated endpoints require `Authorization: Bearer <jwt_token>` header.

### Base URL
- **Local:** `http://localhost:8000/api`
- **Production:** `https://<ecs-instance>/api`

### Error Format (All Endpoints)

```json
{
  "error": {
    "code": "MEMORY_NOT_FOUND",
    "message": "No memory found with id abc-123",
    "status": 404
  }
}
```

---

## 2. Endpoint Summary

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/auth/register` | ❌ | Create account |
| `POST` | `/api/auth/login` | ❌ | Log in |
| `POST` | `/api/chat` | ✅ | Send message, get streamed AI response |
| `POST` | `/api/chat/regenerate` | ✅ | Regenerate last AI response |
| `GET` | `/api/sessions` | ✅ | List past sessions |
| `GET` | `/api/sessions/{id}` | ✅ | Get full session history |
| `GET` | `/api/memories` | ✅ | View all stored memories |
| `PATCH` | `/api/memories/{id}` | ✅ | Correct a memory |
| `DELETE` | `/api/memories/{id}` | ✅ | Delete a memory |
| `GET` | `/api/health-summary` | ✅ | Generate doctor-ready summary |
| `GET` | `/api/family` | ✅ | View family health graph |
| `POST` | `/api/family` | ✅ | Add a family member |
| `GET` | `/api/timeline` | ✅ | Get symptom timeline |

---

## 3. Auth Endpoints

### `POST /api/auth/register`

Create a new user account.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "securepassword123",
  "display_name": "Ade"
}
```

**Response (201 Created):**
```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "display_name": "Ade",
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Errors:**
| Status | Code | When |
|---|---|---|
| 409 | `EMAIL_EXISTS` | Email already registered |
| 422 | `INVALID_EMAIL` | Email format invalid |
| 422 | `WEAK_PASSWORD` | Password too short (min 8 chars) |

**Backend logic:**
1. Validate email format and password length
2. Hash password with bcrypt
3. Insert into `users` table
4. Generate JWT token with `user_id` claim
5. Return token

---

### `POST /api/auth/login`

Log in with existing credentials.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "securepassword123"
}
```

**Response (200 OK):**
```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "display_name": "Ade",
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Errors:**
| Status | Code | When |
|---|---|---|
| 401 | `INVALID_CREDENTIALS` | Wrong email or password |

**Backend logic:**
1. Look up user by email
2. Verify password hash with bcrypt
3. Update `last_active_at`
4. Generate JWT token
5. Return token

---

## 4. Chat Endpoints (Core)

### `POST /api/chat`

Send a user message and receive a streamed AI response. This is the most complex endpoint — it orchestrates the entire memory-aware conversation flow.

**Request:**
```json
{
  "session_id": "uuid-or-null",
  "message": "I've been getting headaches lately, maybe 3-4 times a week."
}
```

- `session_id: null` → creates a new session
- `session_id: "uuid"` → continues an existing session

**Response (SSE Stream):**

```
event: session
data: {"session_id": "a1b2c3d4-..."}

event: token
data: {"content": "I"}

event: token
data: {"content": "'m sorry"}

event: token
data: {"content": " to hear about"}

event: token
data: {"content": " your headaches."}

event: done
data: {"message_id": "uuid", "contains_health_fact": true}
```

**Errors:**
| Status | Code | When |
|---|---|---|
| 404 | `SESSION_NOT_FOUND` | session_id doesn't exist or belongs to another user |
| 422 | `EMPTY_MESSAGE` | Message is blank |
| 503 | `LLM_UNAVAILABLE` | Qwen Cloud API is down |

**Backend logic (in order):**
1. If `session_id` is null, create a new session (increment `session_number` for user)
2. Save user message to `messages` table with next `sequence_num`
3. Load Critical memories (always injected into prompt)
4. Semantic search for relevant Important/Contextual memories (token-counted retrieval)
5. Load pending follow-ups and pattern alerts
6. Build the full system prompt (see system-prompt-design.md)
7. Call Qwen-Plus API with streaming enabled
8. Stream tokens to client via SSE
9. When stream completes, save assistant message to `messages` table
10. Qwen's response includes hidden `contains_health_fact` flag
11. If `contains_health_fact: true`, enqueue background extraction task (see memory-extraction-pipeline-design.md)
12. Send `done` event with metadata

---

### `POST /api/chat/regenerate`

Regenerate the last AI response in a session. Implements the "Try Again" button.

**Request:**
```json
{
  "session_id": "uuid"
}
```

**Response:** Same SSE stream format as `/api/chat`.

**Errors:**
| Status | Code | When |
|---|---|---|
| 404 | `SESSION_NOT_FOUND` | Session doesn't exist |
| 400 | `NO_MESSAGES` | Session has no messages to regenerate from |

**Backend logic:**
1. Find the last assistant message in the session
2. Delete it from `messages` table
3. Delete any memories extracted from that specific message (`source_message_id`)
4. Re-run the same flow as `/api/chat` using the last user message
5. Stream the new response

---

## 5. Session Endpoints

### `GET /api/sessions`

List all past sessions for the logged-in user.

**Query params:** None (returns all sessions, newest first)

**Response (200 OK):**
```json
{
  "sessions": [
    {
      "id": "uuid",
      "session_number": 5,
      "started_at": "2026-06-20T10:00:00Z",
      "ended_at": "2026-06-20T10:30:00Z",
      "preview": "I've been getting headaches...",
      "message_count": 8
    },
    {
      "id": "uuid",
      "session_number": 4,
      "started_at": "2026-06-15T14:00:00Z",
      "ended_at": "2026-06-15T14:20:00Z",
      "preview": "My doctor prescribed Bactrim...",
      "message_count": 6
    }
  ]
}
```

**Backend logic:**
1. Query `sessions` for user, ordered by `started_at DESC`
2. For each session, join with `messages` to get the first user message (preview) and count

> **Note:** `preview` is the first 100 characters of the first user message in that session.

---

### `GET /api/sessions/{id}`

Get full message history for a specific session.

**Response (200 OK):**
```json
{
  "session_id": "uuid",
  "session_number": 5,
  "started_at": "2026-06-20T10:00:00Z",
  "ended_at": "2026-06-20T10:30:00Z",
  "messages": [
    {
      "id": "uuid",
      "role": "user",
      "content": "I've been getting headaches lately, maybe 3-4 times a week.",
      "created_at": "2026-06-20T10:00:15Z"
    },
    {
      "id": "uuid",
      "role": "assistant",
      "content": "I'm sorry to hear about your headaches...",
      "created_at": "2026-06-20T10:00:18Z"
    }
  ]
}
```

**Errors:**
| Status | Code | When |
|---|---|---|
| 404 | `SESSION_NOT_FOUND` | Session doesn't exist or belongs to another user |

---

## 6. Memory Endpoints

### `GET /api/memories`

View all stored memories. Powers the Memory Viewer page.

**Query params:**
- `?tier=critical` — filter by tier (optional)
- `?category=allergy` — filter by category (optional)
- `?subject=self` — filter by subject (optional)

**Response (200 OK):**
```json
{
  "memories": [
    {
      "id": "uuid",
      "fact": "Allergic to penicillin — causes skin rash",
      "category": "allergy",
      "tier": "critical",
      "status": "active",
      "confidence": "high",
      "subject": "self",
      "family_member_label": null,
      "version": 1,
      "created_at": "2026-03-15T10:30:00Z",
      "last_accessed_at": "2026-06-20T14:00:00Z",
      "access_count": 7
    },
    {
      "id": "uuid",
      "fact": "Type 2 Diabetes",
      "category": "family_history",
      "tier": "critical",
      "status": "active",
      "confidence": "high",
      "subject": "family_member",
      "family_member_label": "mother",
      "version": 1,
      "created_at": "2026-03-10T09:00:00Z",
      "last_accessed_at": "2026-06-18T11:00:00Z",
      "access_count": 12
    }
  ],
  "counts": {
    "critical": 5,
    "important": 12,
    "contextual": 8,
    "ephemeral": 3
  }
}
```

**Backend logic:**
1. Query `memories` where `user_id` matches and `superseded_by IS NULL` (current versions only)
2. Apply optional filters
3. Count memories per tier for the summary

---

### `PATCH /api/memories/{id}`

Correct a memory. Creates a new version — does NOT edit in place (Safeguard 8: versioned memories).

**Request:**
```json
{
  "fact": "Allergic to penicillin — causes hives (not rash)"
}
```

**Response (200 OK):**
```json
{
  "id": "new-uuid",
  "fact": "Allergic to penicillin — causes hives (not rash)",
  "version": 2,
  "previous_version_id": "old-uuid",
  "created_at": "2026-06-24T07:50:00Z"
}
```

**Errors:**
| Status | Code | When |
|---|---|---|
| 404 | `MEMORY_NOT_FOUND` | Memory doesn't exist or belongs to another user |
| 400 | `ALREADY_SUPERSEDED` | Memory was already superseded by a newer version |

**Backend logic:**
1. Find the memory, verify ownership
2. Create a new memory record with incremented `version`, copy all other fields
3. Update the `fact` field with new value
4. Generate new embedding for the updated fact
5. Set `superseded_by` on the old record to point to the new one

---

### `DELETE /api/memories/{id}`

Delete a memory permanently. Full transparency — "forget that I mentioned headaches in March."

**Response:** `204 No Content`

**Errors:**
| Status | Code | When |
|---|---|---|
| 404 | `MEMORY_NOT_FOUND` | Memory doesn't exist or belongs to another user |

**Backend logic:**
1. Find the memory, verify ownership
2. Hard delete the memory and all its previous versions (cascade through `superseded_by` chain)
3. Delete any `follow_ups` linked to this memory

---

## 7. Health Summary Endpoint

### `GET /api/health-summary`

Generate the doctor-ready health summary. Returns both structured JSON and a pre-formatted text version.

**Query params:** None

**Response (200 OK):**
```json
{
  "generated_at": "2026-06-23T12:00:00Z",
  "allergies": [
    "Penicillin (causes hives)",
    "Sulfa drugs"
  ],
  "medications": [
    {
      "name": "Lisinopril 20mg",
      "purpose": "blood pressure",
      "status": "active",
      "change_history": [
        { "date": "2024-01", "detail": "Started at 10mg" },
        { "date": "2026-05", "detail": "Increased to 20mg" }
      ]
    }
  ],
  "conditions": [
    {
      "name": "Hypertension",
      "status": "active",
      "diagnosed": "2024"
    },
    {
      "name": "Childhood asthma",
      "status": "resolved"
    }
  ],
  "family_history": [
    {
      "member": "Mother",
      "conditions": ["Type 2 Diabetes"]
    },
    {
      "member": "Father",
      "conditions": ["Stroke (age 55)", "Hypertension"]
    }
  ],
  "symptom_timeline": [
    { "date": "2026-03", "symptom": "Recurring headaches (3-4x/week)" },
    { "date": "2026-04", "symptom": "Intermittent blurred vision (evenings)" },
    { "date": "2026-06", "symptom": "Positional dizziness" }
  ],
  "pending_actions": [
    { "item": "Cholesterol test", "mentioned": "2026-06-10" }
  ],
  "formatted_text": "HEALTH SUMMARY — Generated June 23, 2026\nPatient-reported information (not a medical record)\n\nALLERGIES: Penicillin (causes hives), Sulfa drugs\nCURRENT MEDICATIONS: Lisinopril 20mg (blood pressure)\n..."
}
```

**Backend logic:**
1. Query all Critical tier memories (active, non-superseded)
2. Group by category (allergies, medications, conditions, family_history)
3. For medications, include version history (via `superseded_by` chain) → this shows dosage changes
4. Query symptom memories ordered by `created_at`
5. Query pending follow-ups
6. Build the `formatted_text` string for copy/paste
7. Return both structured and formatted versions

---

## 8. Family Endpoints

### `GET /api/family`

View the family health graph. Returns family members with their associated health conditions.

**Response (200 OK):**
```json
{
  "members": [
    {
      "id": "uuid",
      "label": "mother",
      "display_name": "Mom",
      "alive": true,
      "conditions": [
        {
          "memory_id": "uuid",
          "fact": "Type 2 Diabetes",
          "status": "active",
          "created_at": "2026-03-10T09:00:00Z"
        }
      ]
    },
    {
      "id": "uuid",
      "label": "father",
      "display_name": "Dad",
      "alive": true,
      "conditions": [
        {
          "memory_id": "uuid",
          "fact": "Stroke at age 55",
          "status": "historical",
          "created_at": "2026-03-10T09:05:00Z"
        },
        {
          "memory_id": "uuid",
          "fact": "Hypertension",
          "status": "active",
          "created_at": "2026-03-10T09:05:00Z"
        }
      ]
    }
  ]
}
```

**Backend logic:**
1. Query `family_members` for the user
2. For each member, query `memories` where `subject = 'family_member'` and `family_member_label` matches
3. Return the combined view

---

### `POST /api/family`

Manually add a family member. (Usually the pipeline creates family members automatically when it extracts family health facts, but this allows manual entry too.)

**Request:**
```json
{
  "label": "mother",
  "display_name": "Mom",
  "alive": true
}
```

**Response (201 Created):**
```json
{
  "id": "uuid",
  "label": "mother",
  "display_name": "Mom",
  "alive": true,
  "conditions": []
}
```

**Errors:**
| Status | Code | When |
|---|---|---|
| 409 | `MEMBER_EXISTS` | A family member with this label already exists for this user |
| 422 | `INVALID_LABEL` | Label not in the allowed normalized set |

---

## 9. Timeline Endpoint

### `GET /api/timeline`

Get symptom timeline for the visualization page. Groups symptoms by month.

**Query params:**
- `?months=6` — How far back to look (default: 6, max: 24)

**Response (200 OK):**
```json
{
  "timeline": [
    {
      "month": "2026-06",
      "entries": [
        {
          "memory_id": "uuid",
          "fact": "Positional dizziness",
          "category": "symptom",
          "confidence": "high",
          "created_at": "2026-06-18T09:00:00Z"
        }
      ]
    },
    {
      "month": "2026-04",
      "entries": [
        {
          "memory_id": "uuid",
          "fact": "Intermittent blurred vision (evenings)",
          "category": "symptom",
          "confidence": "high",
          "created_at": "2026-04-12T11:00:00Z"
        }
      ]
    },
    {
      "month": "2026-03",
      "entries": [
        {
          "memory_id": "uuid",
          "fact": "Recurring headaches (3-4x/week)",
          "category": "symptom",
          "confidence": "high",
          "created_at": "2026-03-08T14:00:00Z"
        }
      ]
    }
  ],
  "alerts": [
    {
      "alert_id": "uuid",
      "type": "escalation",
      "description": "Headache frequency increased over 3 months — recommend medical evaluation",
      "created_at": "2026-06-18T09:05:00Z"
    }
  ]
}
```

**Backend logic:**
1. Query `memories` where `category = 'symptom'` and `created_at` is within the requested timeframe
2. Group by month (formatted as `YYYY-MM`)
3. Query `pattern_alerts` for any pending or recently delivered alerts
4. Return grouped timeline with alerts

---

## 10. Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Streaming method** | SSE (Server-Sent Events) | Simpler than WebSocket. Works over plain HTTP. We only need server→client streaming. FastAPI supports it natively with `StreamingResponse`. |
| **Auth tokens** | JWT, 7-day expiry, no refresh | Hackathon scope. No need for refresh token complexity. |
| **Regenerate approach** | Delete last message + re-run | Clean and simple. Also cleans up any memories extracted from the deleted response. |
| **Memory deletion** | Hard delete | Full transparency promise. When user says "forget this", we actually forget it. |
| **Memory correction** | New version (not edit in place) | Preserves audit trail. Supports medication change history in health summary. |
| **Session previews** | First 100 chars of first user message | Avoids storing a separate "title" field. Good enough for a session list. |
| **Health summary format** | JSON + formatted text | JSON for the frontend to render a nice UI. Text for copy/paste into an email or print for the doctor. |

---

## 11. Next Steps

1. Design remaining components (System Prompt & Agent Behavior, Frontend & UI, Auth)
2. Write the full implementation plan
3. Begin coding
