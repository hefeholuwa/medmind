# MedMind — AI Health Navigator with Persistent Memory

**Track:** Track 1 — MemoryAgent  
**Tagline:** "Your AI health companion that remembers your story."  
**Deadline:** July 9, 2026 (2:00 PM PT)  
**Days Remaining:** ~16 days (as of June 23, 2026)

---

## 1. The Problem

Billions of people lack easy access to personalized health guidance. Existing health AI tools (ChatGPT, Ada Health, Buoy, Babylon) share the same critical flaw: **they forget you.** Every conversation starts from zero.

Real healthcare doesn't work this way. A good doctor remembers:
- Your family has a history of diabetes
- You mentioned chest tightness 3 months ago
- You're allergic to penicillin
- You stopped taking your blood pressure medication last month

**No AI health tool today connects the dots across sessions, tracks symptom patterns over time, or proactively follows up.** MedMind does.

---

## 2. What MedMind Does

MedMind is a persistent-memory AI health companion that gets smarter the more you use it. It remembers your medical history, your family's health background, your symptoms over time, and your medications — then uses that context to give progressively better, personalized health guidance.

### Core Capabilities

#### 2.1 Family Health Graph
Users can tell MedMind about their family's health history. The agent builds a family medical tree and uses it when evaluating symptoms.

**Example:**
> Session 1: "My mother has type 2 diabetes and my father had a stroke at 55."  
> Session 5 (3 months later): "I've been really thirsty and urinating frequently."  
> MedMind: "Given your family history of type 2 diabetes on your mother's side, these symptoms — increased thirst and frequent urination — could be significant. I'd strongly recommend getting a fasting blood glucose test. Would you like me to explain what the test involves?"

#### 2.2 Symptom Timeline Detective
MedMind tracks symptoms across sessions and detects patterns that emerge over weeks or months — patterns a user might not connect themselves.

**Example:**
> Session 1 (March): "I've been getting headaches lately, maybe 3-4 times a week."  
> Session 2 (April): "My vision gets blurry sometimes, especially in the evening."  
> Session 3 (June): "I've been feeling dizzy when I stand up quickly."  
> MedMind: "I want to flag something. Over the past 3 months, you've reported a pattern: recurring headaches (March), intermittent blurred vision (April), and now positional dizziness (June). These three symptoms together, especially progressing over months, warrant medical attention. I'd recommend seeing a doctor. Here's a timeline summary you can bring with you."

#### 2.3 Medication & Allergy Tracker
Permanently stores allergies and current medications. Cross-references when users mention new prescriptions or symptoms that could be side effects.

**Example:**
> Session 1: "I'm allergic to sulfa drugs and I take lisinopril for blood pressure."  
> Session 4: "My doctor prescribed Bactrim for an infection."  
> MedMind: "Important — Bactrim (trimethoprim-sulfamethoxazole) is a sulfa drug, and you've told me you're allergic to sulfa drugs. Please confirm this with your doctor before taking it. Would you like me to suggest alternative antibiotics you could ask about?"

#### 2.4 Proactive Follow-ups
MedMind doesn't just answer questions — it follows up on things that matter.

**Example:**
> Session 3: "I think I need to get my cholesterol checked."  
> Session 4 (2 weeks later, user returns for unrelated question):  
> MedMind: "Before we get to your question — you mentioned wanting to get your cholesterol checked about 2 weeks ago. Did you get that done? If you have the results, I can help you understand them."

#### 2.5 Doctor-Ready Health Summary
At any time, the user can request a formatted summary of their health profile to bring to a doctor's appointment. Includes: conditions, medications, allergies, family history, recent symptoms with dates, and pending concerns.

**Example output:**
```
HEALTH SUMMARY — Generated June 23, 2026
Patient-reported information (not a medical record)

ALLERGIES: Penicillin (causes rash), Sulfa drugs
CURRENT MEDICATIONS: Lisinopril 10mg (blood pressure), Metformin 500mg (blood sugar)
FAMILY HISTORY: Mother — Type 2 Diabetes; Father — Stroke (age 55), Hypertension
CHRONIC CONDITIONS: Hypertension (diagnosed 2024), Pre-diabetes (diagnosed 2025)

RECENT SYMPTOM TIMELINE:
- Mar 2026: Recurring headaches (3-4x/week)
- Apr 2026: Intermittent blurred vision (evenings)
- Jun 2026: Positional dizziness

PENDING: Cholesterol test (mentioned Jun 10, not yet completed)
```

---

## 3. Memory Architecture (Critical for Track 1 Judging)

This is the technical heart of the project and what judges will scrutinize most. The memory system must demonstrate: **efficient storage, intelligent retrieval, timely forgetting, and recall within limited context windows.**

### 3.1 Memory Tiers

| Tier | What It Stores | TTL (Time to Live) | Retrieval Priority |
|---|---|---|---|
| **Critical** | Allergies, chronic conditions, current medications, family history | **Never expires** | Always injected into context |
| **Important** | Past diagnoses, surgeries, lab results, significant symptoms | **2 years**, compressed after 6 months | Retrieved when semantically relevant |
| **Contextual** | Recent symptoms, lifestyle mentions, doctor visits, mood/sleep | **6 months**, decays if not referenced | Retrieved by similarity search |
| **Ephemeral** | Common cold, minor one-off questions, greetings | **30 days** | Only retrieved if directly asked |

### 3.2 Memory Operations

#### Storage
- After every conversation turn, the agent extracts **health-relevant facts** and classifies them into the appropriate tier
- Facts are stored as structured JSON objects with metadata:
  ```json
  {
    "id": "mem_abc123",
    "tier": "critical",
    "category": "allergy",
    "content": "Allergic to penicillin — causes skin rash",
    "source_session": "session_003",
    "created_at": "2026-03-15T10:30:00Z",
    "last_accessed": "2026-06-20T14:00:00Z",
    "access_count": 7,
    "embedding": [0.012, -0.045, ...]
  }
  ```

#### Retrieval
- **Critical tier:** Always loaded into the system prompt (small, bounded set)
- **Important/Contextual tiers:** Semantic similarity search against the current user message
- **Context window management:** Budget allocation:
  - System prompt + critical memories: ~1,500 tokens
  - Retrieved relevant memories: ~1,000 tokens (top-k by relevance)
  - Conversation history (current session): ~2,000 tokens
  - User message + response generation: remaining tokens
- **Re-ranking:** Retrieved memories are re-ranked by: (relevance score × recency weight × tier priority)

#### Forgetting
- **Ephemeral tier:** Auto-deleted after 30 days with no access
- **Contextual tier:** Relevance score decays over time. After 6 months with zero access, memories are archived (removed from active retrieval)
- **Important tier:** Compressed after 6 months (e.g., "3 headache episodes in March 2026" instead of 3 separate entries). Deleted after 2 years if never accessed
- **Critical tier:** Never forgotten. Can only be manually corrected by user ("Actually, I'm not allergic to penicillin anymore — I was retested")

#### Conflict Resolution
- When new information contradicts stored memory, the agent asks for confirmation:
  > "I have on record that you take Lisinopril 10mg. You just mentioned taking Lisinopril 20mg. Did your dosage change?"
- Upon confirmation, old memory is updated (not deleted) with a change log

### 3.3 Memory Visualization
- Users can ask "What do you remember about me?" and get a categorized view of all stored memories
- Users can correct or delete any memory ("Forget that I mentioned headaches in March")
- Full transparency — no hidden data

---

## 4. Technical Architecture

### 4.1 System Components

```
┌─────────────────────────────────────────────────────────┐
│                    FRONTEND (Web App)                     │
│              Next.js / React — Vercel or ECS              │
│                                                           │
│  ┌─────────┐  ┌──────────┐  ┌───────────┐  ┌──────────┐ │
│  │  Chat    │  │ Health   │  │  Memory   │  │ Family   │ │
│  │  Interface│ │ Timeline │  │  Viewer   │  │ Tree     │ │
│  └─────────┘  └──────────┘  └───────────┘  └──────────┘ │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTPS / WebSocket
                       ▼
┌─────────────────────────────────────────────────────────┐
│              BACKEND (API Server)                         │
│         Python (FastAPI) — Alibaba Cloud ECS              │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐ │
│  │  Conversation │  │   Memory     │  │   Health Fact  │ │
│  │  Manager      │  │   Engine     │  │   Extractor    │ │
│  └──────┬───────┘  └──────┬───────┘  └───────┬────────┘ │
│         │                 │                   │          │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌───────▼────────┐ │
│  │  Context      │  │  Retrieval   │  │  Forgetting    │ │
│  │  Builder      │  │  & Ranking   │  │  Scheduler     │ │
│  └──────┬───────┘  └──────┬───────┘  └───────┬────────┘ │
└─────────┼────────────────┼───────────────────┼──────────┘
          │                │                   │
          ▼                ▼                   ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
│  Qwen Cloud  │  │  Vector DB   │  │  PostgreSQL      │
│  (LLM API)   │  │  (Embeddings │  │  (Structured     │
│              │  │   + Search)  │  │   Health Data)   │
│  - Qwen-Plus │  │              │  │                  │
│  - Qwen-Max  │  │  Alibaba     │  │  Alibaba Cloud   │
│              │  │  Cloud       │  │  RDS              │
└──────────────┘  └──────────────┘  └──────────────────┘
```

### 4.2 Tech Stack

| Component | Technology | Why |
|---|---|---|
| **LLM** | Qwen-Plus / Qwen-Max via Qwen Cloud API | Required by hackathon. Qwen-Plus for most interactions, Qwen-Max for complex medical reasoning |
| **Backend** | Python + FastAPI | Fast to build, great async support, clean API design |
| **Frontend** | Next.js (React) | Modern, fast, good for the demo video |
| **Structured DB** | PostgreSQL (Alibaba Cloud RDS) | Stores user profiles, health records, session history, family graph |
| **Vector DB** | Alibaba Cloud Elasticsearch or pgvector extension | Semantic search over memory embeddings |
| **Embeddings** | Qwen text-embedding model via Qwen Cloud | Keep everything on the required platform |
| **Hosting** | Alibaba Cloud ECS | Required by hackathon (proof of deployment) |
| **Auth** | Simple email/password or session-based | Keep it minimal — not the focus |

### 4.3 API Design

```
POST   /api/auth/register          — Create account
POST   /api/auth/login             — Login
POST   /api/chat                   — Send message, get response
GET    /api/sessions               — List past sessions
GET    /api/sessions/:id           — Get session history
GET    /api/memory                 — View all stored memories
PATCH  /api/memory/:id             — Correct a memory
DELETE /api/memory/:id             — Delete a memory
GET    /api/health-summary         — Generate doctor-ready summary
GET    /api/family                 — View family health graph
POST   /api/family                 — Add family member health info
GET    /api/timeline               — Get symptom timeline
```

---

## 5. Conversation Flow & Agent Behavior

### 5.1 System Prompt Structure

```
[ROLE]
You are MedMind, a personal health navigator with persistent memory.
You are NOT a doctor. You provide health information and guidance,
not medical diagnoses. Always recommend professional medical care
for serious concerns.

[CRITICAL MEMORIES — Always loaded]
- Allergies: {allergies}
- Medications: {medications}
- Chronic conditions: {conditions}
- Family history: {family_history}

[RELEVANT MEMORIES — Retrieved for this conversation]
{top_k_relevant_memories}

[PENDING FOLLOW-UPS]
{any_proactive_items_due}

[BEHAVIOR RULES]
1. Always check new symptoms against stored history and family history
2. Flag potential drug interactions with stored medications
3. When symptoms form a pattern over time, connect the dots explicitly
4. Never diagnose — suggest what to discuss with a doctor
5. After each conversation, extract and store health-relevant facts
6. If user mentions intent to get a test/see a doctor, create a follow-up reminder
7. Use clear, non-medical language. Explain medical terms when used.
8. If uncertain about severity, err on the side of recommending medical attention
```

### 5.2 Post-Conversation Processing

After every conversation, a background pipeline runs:

1. **Fact Extraction** — Qwen extracts health-relevant facts from the conversation
2. **Classification** — Each fact is classified into a memory tier
3. **Deduplication** — Check if the fact already exists; update if so
4. **Embedding** — Generate vector embedding for semantic search
5. **Storage** — Save to both structured DB (PostgreSQL) and vector DB
6. **Follow-up Detection** — If user expressed intent ("I should get that checked"), create a proactive reminder
7. **Forgetting Sweep** — Check for expired ephemeral/contextual memories

---

## 6. Key Differentiators vs. Existing Tools

| Feature | ChatGPT / Ada / Buoy | MedMind |
|---|---|---|
| Remembers across sessions | ❌ | ✅ Full persistent memory |
| Family health history | ❌ | ✅ Family health graph |
| Symptom pattern detection over time | ❌ | ✅ Timeline detective |
| Drug interaction checks vs. stored meds | ❌ | ✅ Automatic |
| Proactive follow-ups | ❌ | ✅ Checks back on pending items |
| Doctor-ready health summary export | ❌ | ✅ Formatted, chronological |
| Memory transparency (view/edit/delete) | ❌ | ✅ Full user control |
| Intelligent forgetting | ❌ | ✅ Tiered TTL with decay |

---

## 7. Demo Scenario (3-Minute Video)

The demo video must tell a compelling story. Here's the script:

### Scene 1: First Visit (0:00–0:45)
- User signs up and starts chatting
- Tells MedMind about family history (mom: diabetes, dad: hypertension, stroke at 55)
- Reports current medications (lisinopril 10mg) and allergy (penicillin)
- MedMind acknowledges and stores everything
- Quick flash of the Memory Viewer showing stored critical memories

### Scene 2: Second Visit — Weeks Later (0:45–1:30)
- User returns with new symptoms: increased thirst, frequent urination
- MedMind immediately connects to family diabetes history WITHOUT being reminded
- Recommends fasting blood glucose test, explains why family history matters
- User mentions they'll get the test done
- Show the Symptom Timeline updating in real-time

### Scene 3: Third Visit — Follow-Up (1:30–2:15)
- User returns for an unrelated question (back pain)
- MedMind first asks about the blood glucose test from last session
- User shares results (slightly elevated)
- MedMind provides context, recommends lifestyle changes, suggests follow-up with doctor
- Shows how the Health Summary has been building over time

### Scene 4: The Power Moment (2:15–2:50)
- User says "I have a doctor's appointment tomorrow"
- MedMind generates a complete health summary with timeline
- Show the formatted output — allergies, meds, family history, symptom timeline, pending concerns
- User can copy/print it

### Scene 5: Close (2:50–3:00)
- Quick montage: Memory Viewer, Family Tree visualization, Symptom Timeline
- Tagline: "MedMind — your AI health companion that remembers your story."

---

## 8. Submission Checklist

### Required Deliverables

- [ ] **Public GitHub Repository** with open source license (MIT or Apache 2.0)
  - [ ] All source code (frontend + backend)
  - [ ] README with setup instructions
  - [ ] Architecture diagram in repo
  - [ ] Requirements/dependencies documented

- [ ] **Proof of Alibaba Cloud Deployment**
  - [ ] Code file in repo showing Alibaba Cloud service usage (ECS, RDS, etc.)
  - [ ] Short recording of backend running on Alibaba Cloud (separate from demo)

- [ ] **Architecture Diagram**
  - [ ] Visual showing: Frontend → Backend (Alibaba Cloud ECS) → Qwen Cloud API + PostgreSQL (RDS) + Vector DB

- [ ] **Demo Video** (≤ 3 minutes)
  - [ ] Uploaded to YouTube (public)
  - [ ] Shows project functioning
  - [ ] No copyrighted music
  - [ ] Follows the demo scenario above

- [ ] **Text Description** on Devpost
  - [ ] Features and functionality
  - [ ] How memory architecture works
  - [ ] Track selection: Track 1 — MemoryAgent

- [ ] **Track Selection**: Track 1 — MemoryAgent

### Optional (But Do It)

- [ ] **Blog Post** about the building journey → eligible for $500 bonus prize
  - [ ] Published publicly (Medium, Dev.to, personal blog, or social post)
  - [ ] Link included in Devpost submission

---

## 9. Development Timeline (16 Days)

### Phase 1: Foundation (Days 1–3) — June 23–25
- [ ] Set up Alibaba Cloud account (ECS instance, RDS PostgreSQL)
- [ ] Set up Qwen Cloud account and API keys
- [ ] Initialize project repo with license and README
- [ ] Set up FastAPI backend skeleton
- [ ] Set up Next.js frontend skeleton
- [ ] Design and create database schema (users, memories, sessions, family)
- [ ] Implement basic auth (register/login)

### Phase 2: Core Memory Engine (Days 4–7) — June 26–29
- [ ] Build memory extraction pipeline (Qwen extracts facts from conversations)
- [ ] Implement memory tier classification
- [ ] Set up vector embeddings (Qwen embedding model)
- [ ] Implement semantic memory retrieval (top-k by relevance)
- [ ] Build context window manager (budget allocation for system prompt)
- [ ] Implement memory deduplication and conflict resolution
- [ ] Build forgetting scheduler (TTL-based expiry + decay)

### Phase 3: Health Features (Days 8–10) — June 30–July 2
- [ ] Implement chat interface with streaming responses
- [ ] Build family health graph (store + retrieve + display)
- [ ] Build symptom timeline tracker
- [ ] Implement medication/allergy tracking with interaction checks
- [ ] Build proactive follow-up system
- [ ] Implement doctor-ready health summary export

### Phase 4: Frontend & Polish (Days 11–13) — July 3–5
- [ ] Design and build chat UI (clean, medical-themed, premium feel)
- [ ] Build Memory Viewer page (view/edit/delete memories)
- [ ] Build Family Tree visualization
- [ ] Build Symptom Timeline visualization
- [ ] Build Health Summary export page
- [ ] Responsive design (mobile-friendly)
- [ ] Add smooth animations and transitions

### Phase 5: Deploy & Submit (Days 14–16) — July 6–9
- [ ] Deploy backend to Alibaba Cloud ECS
- [ ] Deploy frontend (Vercel or Alibaba Cloud)
- [ ] Record proof of Alibaba Cloud deployment
- [ ] Create architecture diagram (clean, professional)
- [ ] Record 3-minute demo video
- [ ] Upload video to YouTube
- [ ] Write Devpost text description
- [ ] Write blog post about the journey
- [ ] Submit on Devpost
- [ ] Test everything one final time

---

## 10. Risk Mitigation

| Risk | Mitigation |
|---|---|
| **$40 credit limit on Qwen Cloud** | Use Qwen-Plus (cheaper) for most calls; Qwen-Max only for complex reasoning. Cache embeddings. Minimize unnecessary API calls during development by using mock responses. |
| **Vector search performance** | Start with pgvector (PostgreSQL extension) — simpler than a separate vector DB. Only move to Elasticsearch if needed. |
| **Scope creep** | The MVP is: chat + memory storage/retrieval + forgetting + health summary. Family graph and timeline are stretch goals that make it competitive. |
| **Medical accuracy concerns** | Heavy disclaimers. MedMind is NOT a diagnostic tool. It provides health information and recommends seeing doctors. This is clearly stated in the system prompt and UI. |
| **Time pressure** | If behind schedule, cut the Family Tree visualization (keep the data, skip the visual). The Memory Viewer and chat are non-negotiable. |
| **Demo video quality** | Script it in advance (Scene 1–5 above). Seed test data so the demo flows perfectly. Record multiple takes. |

---

## 11. What Judges Will Look For (Mapped to Our Project)

### Innovation & AI Creativity (30%)
- ✅ Tiered memory architecture with intelligent forgetting
- ✅ Semantic retrieval with re-ranking
- ✅ Health fact extraction pipeline
- ✅ Cross-session symptom pattern detection

### Technical Depth & Engineering (30%)
- ✅ Clean modular architecture (memory engine, context builder, fact extractor as separate components)
- ✅ Proper database design (relational + vector)
- ✅ Context window management with token budgeting
- ✅ Conflict resolution for contradictory memories

### Problem Value & Impact (25%)
- ✅ Universal healthcare access problem
- ✅ Clear productization path
- ✅ Serves underserved populations (emerging markets, rural areas)
- ✅ Bridges gap between AI health tools and real medical care

### Presentation & Documentation (15%)
- ✅ Scripted 3-minute demo with clear narrative arc
- ✅ Architecture diagram
- ✅ Clean README with setup instructions
- ✅ Blog post about the building journey

---

## 12. Project Name Options

| Name | Why |
|---|---|
| **MedMind** | Clean, memorable, implies medical + intelligence |
| **HealthMemo** | Directly references health + memory |
| **CarePal** | Friendly, approachable, companion feel |
| **VitalRecall** | Medical (vital signs) + memory (recall) |

**Current pick: MedMind** — short, professional, clear.

---

## 13. Important Disclaimers (Must Include in App & Submission)

- MedMind is an AI health information tool, NOT a medical professional
- It does not diagnose, treat, or prescribe
- Information provided is for educational purposes only
- Users should always consult qualified healthcare professionals
- MedMind is not a substitute for professional medical advice
- In emergencies, call local emergency services immediately
- Health data is stored securely but users should review the privacy policy
