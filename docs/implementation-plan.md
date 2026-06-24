# MedMind — Implementation Plan

**Date:** June 24, 2026  
**Deadline:** July 9, 2026 (2:00 PM PT)  
**Days Remaining:** ~15 days  
**Status:** Ready to Execute  

---

## Build Order (Dependency Chain)

```
Phase 1: Foundation
  Docker + DB Schema + FastAPI Skeleton + Auth
        │
        ▼
Phase 2: Core Chat Loop
  Basic Chat API → Memory Extraction → Memory Storage
        │
        ▼
Phase 3: Memory-Aware Chat
  Retrieval → Context Builder → Conflict Detection → Follow-ups
        │
        ▼
Phase 4: Feature APIs
  Memories CRUD → Family → Timeline → Health Summary → Pattern Detection
        │
        ▼
Phase 5: Frontend
  Auth Page → Chat Page → Memory Viewer → Family Tree → Timeline → Health Summary
        │
        ▼
Phase 6: Deploy & Submit
  Alibaba Cloud → Demo Video → Devpost → Blog Post
```

---

## Phase 1: Foundation (Days 1–2) — June 24–25

**Goal:** Running FastAPI server connected to a local PostgreSQL database with working auth.

### Day 1 — June 24

- [ ] **Initialize project repository**
  - [ ] Create git repo with MIT license
  - [ ] Create `.gitignore` (Python, Node, .env)
  - [ ] Create project README skeleton
  - [ ] Create `backend/` and `frontend/` directories

- [ ] **Set up Docker database**
  - [ ] Create `docker-compose.yml` with `pgvector/pgvector:pg16`
  - [ ] Start the container, verify connection
  - [ ] Reference: [database-schema-design.md](file:///Users/user/Desktop/medmind/docs/design/database-schema-design.md) §6

- [ ] **Set up FastAPI backend**
  - [ ] Create `requirements.txt` (fastapi, uvicorn, asyncpg, sqlalchemy, bcrypt, python-jose, httpx)
  - [ ] Create `backend/main.py` (FastAPI app with CORS)
  - [ ] Create `backend/database.py` (async database connection)
  - [ ] Create `backend/config.py` (environment variables via pydantic-settings)
  - [ ] Create `.env.example` with all required variables
  - [ ] Verify `uvicorn` starts and `/docs` loads

- [ ] **Run database migration**
  - [ ] Create `backend/migrations/001_initial_schema.sql`
  - [ ] Create all 7 tables (users, sessions, messages, memories, family_members, follow_ups, pattern_alerts)
  - [ ] Create all indexes
  - [ ] Run migration against Docker Postgres, verify tables exist
  - [ ] Reference: [database-schema-design.md](file:///Users/user/Desktop/medmind/docs/design/database-schema-design.md) §3–4

### Day 2 — June 25

- [ ] **Implement auth**
  - [ ] Create `backend/auth.py` (JWT generation, password hashing, middleware)
  - [ ] Create `backend/routes/auth.py` (register + login endpoints)
  - [ ] Test: register a user via `/docs`, verify DB row exists
  - [ ] Test: login with correct password → get token
  - [ ] Test: login with wrong password → 401
  - [ ] Test: access protected endpoint without token → 401
  - [ ] Reference: [auth-design.md](file:///Users/user/Desktop/medmind/docs/design/auth-design.md)

- [ ] **Set up Next.js frontend skeleton**
  - [ ] Initialize Next.js 14 in `frontend/` with App Router
  - [ ] Create `frontend/app/globals.css` with full design system (CSS variables)
  - [ ] Create placeholder pages for all 6 routes
  - [ ] Verify `npm run dev` loads
  - [ ] Reference: [frontend-ui-design.md](file:///Users/user/Desktop/medmind/docs/design/frontend-ui-design.md) §2

**Phase 1 Checkpoint:** ✅ FastAPI running. ✅ Postgres running with all tables. ✅ Can register/login. ✅ Next.js dev server running.

---

## Phase 2: Core Chat Loop (Days 3–5) — June 26–28

**Goal:** User can chat with MedMind via the API. The AI responds with streaming. Health facts are extracted and stored in the background.

### Day 3 — June 26

- [ ] **Set up Qwen Cloud integration**
  - [ ] Create `backend/llm.py` (Qwen API client wrapper)
  - [ ] Implement streaming chat call (SSE-compatible)
  - [ ] Implement non-streaming call (for extraction)
  - [ ] Implement embedding call (`text-embedding-v3`, 1024 dimensions)
  - [ ] Test: send a basic prompt, get a response
  - [ ] Test: generate an embedding, verify 1024-dim vector returned

- [ ] **Build basic chat endpoint (no memory yet)**
  - [ ] Create `backend/routes/chat.py`
  - [ ] Implement `POST /api/chat` with SSE streaming
  - [ ] Save user message to `messages` table
  - [ ] Save assistant response to `messages` table
  - [ ] Handle new session creation (`session_id: null`)
  - [ ] Handle existing session continuation
  - [ ] Test: send message via `/docs`, see streamed response
  - [ ] Reference: [api-endpoints-design.md](file:///Users/user/Desktop/medmind/docs/design/api-endpoints-design.md) §4

### Day 4 — June 27

- [ ] **Build memory extraction pipeline**
  - [ ] Create `backend/extraction.py`
  - [ ] Implement the extraction prompt call to Qwen
  - [ ] Implement JSON parsing with retry on failure (Safeguard: JSON validation)
  - [ ] Implement source_quote verification (Safeguard 2)
  - [ ] Implement subject filtering — discard `other` (Safeguard 3)
  - [ ] Implement intent filtering — discard `hypothetical` and `question` (Safeguard 9)
  - [ ] Implement confidence scoring — low confidence → force Ephemeral (Safeguard 11)
  - [ ] Implement temporal status classification (Safeguard 10)
  - [ ] Reference: [memory-extraction-pipeline-design.md](file:///Users/user/Desktop/medmind/docs/design/memory-extraction-pipeline-design.md) §4–5

- [ ] **Wire extraction into chat endpoint**
  - [ ] Parse `contains_health_fact` flag from AI response metadata (Safeguard 1)
  - [ ] Strip metadata tag before saving assistant message
  - [ ] If flag is true → enqueue background extraction task via FastAPI `BackgroundTasks`
  - [ ] Test: send health-related message → check that memories appear in DB
  - [ ] Test: send greeting → verify no extraction runs

### Day 5 — June 28

- [ ] **Implement memory storage safeguards**
  - [ ] Implement embedding generation for extracted facts
  - [ ] Implement session-level advisory lock (Safeguard 5)
  - [ ] Implement DB transaction wrapping (Safeguard 6)
  - [ ] Implement `last_processed_message_id` tracking (Safeguard 6)
  - [ ] Implement deduplication via semantic similarity check
  - [ ] Implement conflict detection (flag for next session, don't auto-update Critical) (Safeguard 4)
  - [ ] Implement dual-source extraction — feed AI response too (Safeguard 12)
  - [ ] Test: send same fact twice → verify no duplicate
  - [ ] Test: send conflicting medication dosage → verify NOT auto-updated

- [ ] **Implement forgetting sweep**
  - [ ] Create `backend/forgetting.py`
  - [ ] Delete Ephemeral memories past 30-day TTL with zero access
  - [ ] Archive Contextual memories past 6-month TTL with zero access
  - [ ] Wire sweep to run after each extraction task
  - [ ] Reference: [database-schema-design.md](file:///Users/user/Desktop/medmind/docs/design/database-schema-design.md) §5

**Phase 2 Checkpoint:** ✅ Can chat with MedMind via API. ✅ AI streams responses. ✅ Health facts extracted and stored in DB. ✅ All safeguards active. ✅ No duplicate/conflicting memories.

---

## Phase 3: Memory-Aware Chat (Days 6–7) — June 29–30

**Goal:** MedMind remembers the user. Chat responses reference past conversations and stored health data.

### Day 6 — June 29

- [ ] **Build memory retrieval system**
  - [ ] Create `backend/retrieval.py`
  - [ ] Implement Critical tier loading (always injected)
  - [ ] Implement semantic search via pgvector (`<=>` cosine distance)
  - [ ] Implement token-counted retrieval loop (Safeguard 7)
  - [ ] Implement re-ranking: `relevance_score × recency_weight × tier_priority`
  - [ ] Update `last_accessed_at` and `access_count` on retrieval
  - [ ] Test: store 3 memories, query semantically, verify ranked results

- [ ] **Build context builder**
  - [ ] Create `backend/context.py`
  - [ ] Assemble full system prompt from template + injected memories
  - [ ] Load pending follow-ups from `follow_ups` table
  - [ ] Load pending pattern alerts from `pattern_alerts` table
  - [ ] Handle first-time user vs. returning user greeting
  - [ ] Wire context builder into chat endpoint
  - [ ] Test: chat after storing memories → verify AI references them
  - [ ] Reference: [system-prompt-design.md](file:///Users/user/Desktop/medmind/docs/design/system-prompt-design.md) §2–3

### Day 7 — June 30

- [ ] **Implement versioned memory updates**
  - [ ] When conflict is confirmed by user → create new version, mark old as `superseded` (Safeguard 8)
  - [ ] Implement family member auto-creation (when pipeline extracts `subject: family_member`)
  - [ ] Test: update medication dosage → verify old version preserved

- [ ] **Implement follow-up system**
  - [ ] When extraction detects `action_required: true` → create `follow_ups` row
  - [ ] Context builder loads pending follow-ups into system prompt
  - [ ] When user confirms action done → mark follow-up as resolved
  - [ ] Test: say "I should get my cholesterol checked" → verify follow-up created → start new session → verify AI asks about it

- [ ] **Implement chat regenerate endpoint**
  - [ ] Create `POST /api/chat/regenerate`
  - [ ] Delete last assistant message and any memories extracted from it
  - [ ] Re-run chat with same context
  - [ ] Test: regenerate → verify different response, old memories cleaned up
  - [ ] Reference: [api-endpoints-design.md](file:///Users/user/Desktop/medmind/docs/design/api-endpoints-design.md) §4

**Phase 3 Checkpoint:** ✅ MedMind remembers across sessions. ✅ References family history, medications, past symptoms. ✅ Asks about pending follow-ups. ✅ Handles conflicts by asking for confirmation. ✅ Regenerate works.

---

## Phase 4: Feature APIs (Days 8–9) — July 1–2

**Goal:** All remaining API endpoints working and tested.

### Day 8 — July 1

- [ ] **Session endpoints**
  - [ ] `GET /api/sessions` — list sessions with previews
  - [ ] `GET /api/sessions/{id}` — get full message history
  - [ ] Test both via `/docs`

- [ ] **Memory CRUD endpoints**
  - [ ] `GET /api/memories` — list with filters (tier, category, subject)
  - [ ] `PATCH /api/memories/{id}` — correct a memory (creates new version)
  - [ ] `DELETE /api/memories/{id}` — hard delete with cascade
  - [ ] Test: create memories via chat → list → edit → delete → verify
  - [ ] Reference: [api-endpoints-design.md](file:///Users/user/Desktop/medmind/docs/design/api-endpoints-design.md) §6

- [ ] **Family endpoints**
  - [ ] `GET /api/family` — view family members with conditions
  - [ ] `POST /api/family` — manually add family member
  - [ ] Test: chat about family → verify auto-created → list via API
  - [ ] Reference: [api-endpoints-design.md](file:///Users/user/Desktop/medmind/docs/design/api-endpoints-design.md) §8

### Day 9 — July 2

- [ ] **Timeline endpoint**
  - [ ] `GET /api/timeline` — symptoms grouped by month with alerts
  - [ ] Test: add symptoms across months → verify timeline output
  - [ ] Reference: [api-endpoints-design.md](file:///Users/user/Desktop/medmind/docs/design/api-endpoints-design.md) §9

- [ ] **Health summary endpoint**
  - [ ] `GET /api/health-summary` — structured JSON + formatted text
  - [ ] Include medication change history from version chain
  - [ ] Include pending follow-ups
  - [ ] Test: populate data via chat → generate summary → verify completeness
  - [ ] Reference: [api-endpoints-design.md](file:///Users/user/Desktop/medmind/docs/design/api-endpoints-design.md) §7

- [ ] **Pattern detection sweep**
  - [ ] Create `backend/patterns.py`
  - [ ] Implement pattern detection prompt call to Qwen
  - [ ] Wire to run after every 5th session (`session_number % 5 == 0`)
  - [ ] Store alerts in `pattern_alerts` table
  - [ ] Test: seed escalating symptoms → trigger detection → verify alert created
  - [ ] Reference: [system-prompt-design.md](file:///Users/user/Desktop/medmind/docs/design/system-prompt-design.md) §5

**Phase 4 Checkpoint:** ✅ All 13 API endpoints working. ✅ Full CRUD on memories. ✅ Health summary generates correctly. ✅ Pattern detection creates alerts. ✅ Backend is 100% complete.

---

## Phase 5: Frontend (Days 10–13) — July 3–6

**Goal:** Beautiful, responsive frontend connected to all APIs.

### Day 10 — July 3

- [ ] **Auth page**
  - [ ] Login/register form with toggle
  - [ ] Split-screen layout (branding left, form right)
  - [ ] JWT storage in localStorage
  - [ ] Redirect to `/chat` on success
  - [ ] Error handling (wrong password, email taken)

- [ ] **App layout + sidebar**
  - [ ] Collapsible sidebar with session list + navigation
  - [ ] Responsive: full → icon-only → overlay
  - [ ] New Chat button
  - [ ] Logout button

- [ ] **Chat page (core)**
  - [ ] Message bubbles (user right/blue, AI left/gray)
  - [ ] SSE streaming integration (tokens appear in real-time)
  - [ ] Streaming indicator (animated dots)
  - [ ] Auto-scroll to latest message
  - [ ] Chat input with submit button
  - [ ] Load session history from sidebar click
  - [ ] Reference: [frontend-ui-design.md](file:///Users/user/Desktop/medmind/docs/design/frontend-ui-design.md) §4.2

### Day 11 — July 4

- [ ] **Chat page (polish)**
  - [ ] Regenerate button on last AI message
  - [ ] Session sidebar grouped by date (Today, Yesterday, Last Week)
  - [ ] First-time user welcome experience
  - [ ] Empty state when no sessions exist
  - [ ] fadeInUp animation on new messages

- [ ] **Memory Viewer page**
  - [ ] Grouped by tier with colored left borders
  - [ ] Collapsible tier sections (Critical always expanded)
  - [ ] Filter dropdown (by tier, category)
  - [ ] Edit memory modal (inline editing)
  - [ ] Delete memory with confirmation
  - [ ] Version history modal (for medications etc.)
  - [ ] Reference: [frontend-ui-design.md](file:///Users/user/Desktop/medmind/docs/design/frontend-ui-design.md) §4.3

### Day 12 — July 5

- [ ] **Family Tree page**
  - [ ] CSS tree layout with connecting lines
  - [ ] Family member cards (glassmorphism)
  - [ ] Conditions listed under each member
  - [ ] "YOU" card at the bottom
  - [ ] Risk factors summary section
  - [ ] Add member modal
  - [ ] Reference: [frontend-ui-design.md](file:///Users/user/Desktop/medmind/docs/design/frontend-ui-design.md) §4.4

- [ ] **Symptom Timeline page**
  - [ ] Vertical timeline with month markers
  - [ ] Symptom entries connected to the line
  - [ ] Alert banner at top for pattern alerts
  - [ ] Time range selector dropdown
  - [ ] Reference: [frontend-ui-design.md](file:///Users/user/Desktop/medmind/docs/design/frontend-ui-design.md) §4.5

### Day 13 — July 6

- [ ] **Health Summary page**
  - [ ] Formatted summary card (print-friendly)
  - [ ] Copy to clipboard button
  - [ ] Print button with `@media print` styles
  - [ ] Disclaimer footer
  - [ ] Reference: [frontend-ui-design.md](file:///Users/user/Desktop/medmind/docs/design/frontend-ui-design.md) §4.6

- [ ] **Full UI polish**
  - [ ] All micro-animations working (hover effects, transitions)
  - [ ] Responsive testing (desktop, tablet, mobile)
  - [ ] Loading states for all API calls
  - [ ] Error states (network failure, 500 errors)
  - [ ] Empty states for all pages
  - [ ] Medical disclaimer in sidebar footer

**Phase 5 Checkpoint:** ✅ All 6 pages working. ✅ Chat streams in real-time. ✅ All CRUD operations work from UI. ✅ Responsive on all screen sizes. ✅ Animations and polish complete.

---

## Phase 6: Deploy & Submit (Days 14–16) — July 7–9

**Goal:** Deployed on Alibaba Cloud. Demo video recorded. Submission complete.

### Day 14 — July 7

- [ ] **Alibaba Cloud deployment**
  - [ ] Set up ECS instance
  - [ ] Set up RDS PostgreSQL with pgvector
  - [ ] Deploy FastAPI backend to ECS
  - [ ] Deploy Next.js frontend (Vercel or ECS)
  - [ ] Update `.env` with production credentials
  - [ ] Test full flow on deployed version
  - [ ] Take screenshots of Alibaba Cloud console showing running services

- [ ] **Proof of Alibaba Cloud deployment**
  - [ ] Create code file showing Alibaba Cloud service usage
  - [ ] Record short screen recording of backend running on Alibaba Cloud

### Day 15 — July 8

- [ ] **Seed demo data**
  - [ ] Create a demo user account
  - [ ] Seed realistic session history (March → April → June scenario from spec)
  - [ ] Verify all memories, family tree, timeline populated correctly
  - [ ] Run through the demo scenario end-to-end to catch bugs

- [ ] **Record demo video (≤ 3 minutes)**
  - [ ] Scene 1 (0:00–0:45): First visit — family history, medications, allergy
  - [ ] Scene 2 (0:45–1:30): Second visit — symptoms + family history connection
  - [ ] Scene 3 (1:30–2:15): Third visit — follow-up + health summary
  - [ ] Scene 4 (2:15–2:50): Doctor appointment — generate summary
  - [ ] Scene 5 (2:50–3:00): Montage — Memory Viewer, Family Tree, Timeline
  - [ ] Upload to YouTube (public)
  - [ ] Reference: [PROJECT_SPEC.md](file:///Users/user/Desktop/medmind/PROJECT_SPEC.md) §7

### Day 16 — July 9 (DEADLINE DAY)

- [ ] **Create architecture diagram**
  - [ ] Clean, professional visual showing all components
  - [ ] Frontend → Backend (ECS) → Qwen Cloud + PostgreSQL (RDS)

- [ ] **Write Devpost submission**
  - [ ] Project description (features, functionality)
  - [ ] How memory architecture works
  - [ ] Track selection: Track 1 — MemoryAgent
  - [ ] Link to GitHub repo
  - [ ] Link to demo video
  - [ ] Link to architecture diagram
  - [ ] Link to Alibaba Cloud deployment proof

- [ ] **Write blog post** (for $500 bonus prize)
  - [ ] Published on Medium or Dev.to
  - [ ] Cover the building journey with QwenCloud
  - [ ] Include link in Devpost submission

- [ ] **Final checks**
  - [ ] GitHub repo is public with MIT license
  - [ ] README has setup instructions
  - [ ] All links in Devpost work
  - [ ] Demo video is public on YouTube
  - [ ] Submit before 2:00 PM PT

---

## Risk Checkpoints

| After Phase | Ask Yourself | If Behind |
|---|---|---|
| Phase 2 (Day 5) | Can I chat with memory extraction working? | Cut safeguards to essential 5 (1-4, 6). Add others later. |
| Phase 3 (Day 7) | Does the AI remember across sessions? | This MUST work. Don't move on until it does. |
| Phase 4 (Day 9) | Are all APIs done? | Cut pattern detection. It's nice-to-have. |
| Phase 5 (Day 13) | Is the UI polished? | Cut Family Tree page (keep the API). Cut Timeline (keep the API). Focus on Chat + Memory Viewer + Health Summary. |
| Phase 6 (Day 16) | Is everything submitted? | Blog post is optional. Submit without it if needed. |

---

## Design Docs Reference

| Document | Path |
|---|---|
| Memory Extraction Pipeline | [memory-extraction-pipeline-design.md](file:///Users/user/Desktop/medmind/docs/design/memory-extraction-pipeline-design.md) |
| Database Schema | [database-schema-design.md](file:///Users/user/Desktop/medmind/docs/design/database-schema-design.md) |
| API Endpoints | [api-endpoints-design.md](file:///Users/user/Desktop/medmind/docs/design/api-endpoints-design.md) |
| System Prompt & Behavior | [system-prompt-design.md](file:///Users/user/Desktop/medmind/docs/design/system-prompt-design.md) |
| Frontend & UI | [frontend-ui-design.md](file:///Users/user/Desktop/medmind/docs/design/frontend-ui-design.md) |
| Auth | [auth-design.md](file:///Users/user/Desktop/medmind/docs/design/auth-design.md) |
| Project Spec | [PROJECT_SPEC.md](file:///Users/user/Desktop/medmind/PROJECT_SPEC.md) |
| Hackathon Rules | [HACKATHON_RULES.md](file:///Users/user/Desktop/medmind/HACKATHON_RULES.md) |
