# 🧠 MedMind — AI Health Navigator with Persistent Memory

> *Your AI health companion that remembers your story.*

**Track:** Track 1 — MemoryAgent | **Hackathon:** Global AI Hackathon Series with Qwen Cloud

---

## What is MedMind?

MedMind is a persistent-memory AI health companion that gets smarter the more you use it. Unlike existing AI health tools that forget you after every session, MedMind remembers your medical history, family health background, symptoms over time, and medications — then uses that context to give progressively better, personalized health guidance.

### Key Features

- **🧬 Family Health Graph** — Tracks family medical history and uses it when evaluating your symptoms
- **🔍 Symptom Timeline Detective** — Connects symptom patterns across sessions over weeks and months
- **💊 Medication & Allergy Tracker** — Cross-references new prescriptions against your known allergies and medications
- **🔔 Proactive Follow-ups** — Remembers when you mention wanting to get a test done and asks about it later
- **📋 Doctor-Ready Health Summary** — Generates a formatted summary you can bring to your doctor's appointment
- **🧠 Memory Transparency** — View, edit, or delete anything MedMind remembers about you

### Memory Architecture

MedMind uses a **tiered memory system** with intelligent forgetting:

| Tier | What It Stores | Lifespan |
|---|---|---|
| **Critical** | Allergies, medications, chronic conditions, family history | Never expires |
| **Important** | Past diagnoses, surgeries, lab results | 2 years |
| **Contextual** | Recent symptoms, lifestyle, doctor visits | 6 months |
| **Ephemeral** | Minor complaints, general questions | 30 days |

---

## Tech Stack

| Component | Technology |
|---|---|
| **LLM** | Qwen-Plus / Qwen-Max via Qwen Cloud API |
| **Embeddings** | Qwen text-embedding-v3 (1024 dimensions) |
| **Backend** | Python + FastAPI |
| **Frontend** | Next.js 14 (React) |
| **Database** | PostgreSQL 16 + pgvector |
| **Hosting** | Alibaba Cloud ECS + RDS |

---

## Architecture

```
┌───────────────────┐
│    Frontend        │
│    (Next.js)       │
└────────┬──────────┘
         │ HTTPS / SSE
         ▼
┌───────────────────┐     ┌──────────────┐
│    Backend         │────▶│  Qwen Cloud  │
│    (FastAPI)       │     │  (LLM API)   │
└────────┬──────────┘     └──────────────┘
         │
         ▼
┌───────────────────┐
│   PostgreSQL       │
│   + pgvector       │
│   (Alibaba RDS)    │
└───────────────────┘
```

---

## Local Development Setup

### Prerequisites

- Python 3.11+
- Node.js 18+
- Docker & Docker Compose
- Qwen Cloud API key ([sign up here](https://www.qwencloud.com/))

### 1. Clone the repo

```bash
git clone https://github.com/<your-username>/medmind.git
cd medmind
```

### 2. Start the database

```bash
docker compose up -d
```

### 3. Set up the backend

```bash
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your Qwen Cloud API key
uvicorn main:app --reload
```

### 4. Set up the frontend

```bash
cd frontend
npm install
npm run dev
```

### 5. Open the app

Visit `http://localhost:3000`

---

## Project Structure

```
medmind/
├── backend/                 # Python FastAPI server
│   ├── main.py              # App entry point
│   ├── config.py            # Environment configuration
│   ├── database.py          # Database connection
│   ├── auth.py              # JWT + bcrypt auth
│   ├── llm.py               # Qwen Cloud API client
│   ├── extraction.py        # Memory extraction pipeline
│   ├── retrieval.py         # Memory retrieval + ranking
│   ├── context.py           # System prompt builder
│   ├── forgetting.py        # Memory TTL + decay
│   ├── patterns.py          # Symptom pattern detection
│   ├── routes/              # API route handlers
│   │   ├── auth.py
│   │   ├── chat.py
│   │   ├── memories.py
│   │   ├── family.py
│   │   ├── timeline.py
│   │   └── health_summary.py
│   └── migrations/          # SQL migration files
│       └── 001_initial_schema.sql
├── frontend/                # Next.js 14 app
│   ├── app/                 # App Router pages
│   ├── components/          # React components
│   └── lib/                 # API client, utilities
├── docker-compose.yml       # Local PostgreSQL + pgvector
├── LICENSE                  # MIT
└── README.md                # This file
```

---

## Disclaimer

MedMind is an AI health information tool, **NOT** a medical professional. It does not diagnose, treat, or prescribe. Information provided is for educational purposes only. Always consult qualified healthcare professionals for medical decisions. In emergencies, call your local emergency services immediately.

---

## License

MIT — see [LICENSE](LICENSE) for details.
