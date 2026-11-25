# AI Interview Coach

A multi-agent, adaptive mock interview platform powered by four specialized LLM agents, a hybrid ChromaDB RAG question store, SQLite performance memory, optional voice I/O (Whisper STT + Edge TTS), and a React frontend served by FastAPI.

**Author:** Aditya Gupta ([GitHub](https://github.com/adityagupta1709))

---

## Key Capabilities

- **Dynamic Resume & Job Description Analysis:** Upload a `.pdf`, `.txt`, or `.md` resume and/or paste a job description. The `ResumeParserAgent` infers any target role (not limited to fixed presets), seniority level (`entry` / `mid` / `senior` / `staff`), tailored competencies, key skills, and specific focus areas to probe.
- **Hybrid RAG + Dynamic Question Generation:** Pulls relevant tagged seed questions from `data/question_bank.json` via ChromaDB (local `all-MiniLM-L6-v2` embeddings, $0 API cost) and adapts them—or dynamically generates fresh, context-aware questions grounded in your resume and JD when no bank match fits.
- **Live Follow-Up Probing:** After any answer, trigger an AI follow-up probe targeting specific gaps or claims in what you just said.
- **Calibrated Rubric Evaluation & Coaching:** Every answer is scored 1–10 across `structure`, `specificity`, `technical_accuracy`, and `communication`, paired with rationale, strengths, gaps, two actionable tips, and a rewritten stronger answer.
- **Adaptive Cross-Session Memory:** SQLite persists sessions and answers, computes per-competency averages, and automatically prioritizes your historically weakest competency first in subsequent sessions before rotating across remaining competencies.
- **End-of-Session Debrief & Analytics:** Generates an executive debrief (`overall_verdict`, strengths, priority improvements, next drill plan) and tracks historical competency performance.
- **Optional Voice Mode:** Speak answers via browser microphone (transcribed with Groq Whisper) and hear questions and coaching read aloud via `edge-tts`.

---

## Architecture

```
Resume (PDF/TXT/text) + Job Description
                 |
                 v
       ResumeParserAgent  --->  dynamic role_type, seniority,
                 |              competencies, key_skills, focus_areas
                 v
     InterviewSession Orchestrator  <-------------------+
     (deterministic Python state machine)               |
       /         |          \                           |
      v          v           v                          |
Interviewer  Evaluator    Coach                         |
   Agent       Agent      Agent                         |
      ^          |           |                          |
      |          +-----+-----+                          |
  ChromaDB RAG         |                                |
(question_bank.json)   v                                |
                 SQLite Memory -------------------------+
        (sessions & answers tables;
         weakest-competency targeting)
```

- **Orchestrator (`src/orchestrator.py`):** Deterministic Python state machine (`InterviewSession`) controlling question sequencing, follow-up probing, evaluation, coaching, and debrief generation.
- **Agents (`src/agents/`):** Narrow, single-responsibility agents sharing a resilient Groq client (`src/agents/utils.py`) with JSON mode, fence-tolerant parsing, and model fallback (`openai/gpt-oss-120b` -> `qwen/qwen3.8-27b` -> `openai/gpt-oss-20b`).
- **RAG Store (`src/rag/question_store.py`):** ChromaDB persistent vector store backed by `data/question_bank.json`, dynamically extensible via `POST /api/question-bank`.
- **Persistence (`src/memory/db.py`):** SQLite (`memory.sqlite3`) storing `sessions` and `answers` with safe schema migration and competency analytics.
- **Voice (`src/voice/`):** Groq `whisper-large-v3-turbo` for STT (`stt.py`) and `edge-tts` (`en-US-GuyNeural`) via `ThreadPoolExecutor` for TTS (`tts.py`).
- **API & Frontend (`src/api.py`, `frontend/`):** FastAPI backend serving REST endpoints and the built React + Vite UI (`frontend/dist`).

For a deep technical breakdown of the agents, RAG retrieval pipeline, orchestrator state machine, and backend architecture, see [ARCHITECTURE_REPORT.md](ARCHITECTURE_REPORT.md).

---

## Tech Stack

- **LLM:** Groq (`openai/gpt-oss-120b` default, with fallback to `qwen/qwen3.8-27b` / `openai/gpt-oss-20b`)
- **Retrieval:** ChromaDB with bundled local embeddings (`all-MiniLM-L6-v2`)
- **Memory:** SQLite (`memory.sqlite3`)
- **Voice:** Groq Whisper (`whisper-large-v3-turbo`) + `edge-tts`
- **Backend:** FastAPI, Uvicorn, `pypdf`
- **Frontend:** React 19 + Vite (`lucide-react`)

---

## Setup & Running Locally

### 1. Backend Setup

```powershell
# Create and activate a Python 3.12 virtual environment
python -m venv venv
.\venv\Scripts\activate      # Windows
# source venv/bin/activate   # Mac/Linux

# Install Python dependencies
pip install -r requirements.txt

# Configure environment variables
copy .env.example .env       # Windows (cp .env.example .env on Mac/Linux)
# Add your Groq API key from https://console.groq.com/keys to .env
```

### 2. Frontend Build

```powershell
cd frontend
npm install
npm run build
cd ..
```

### 3. Start the Application

```powershell
python main.py
```

Open **http://127.0.0.1:8000** in your browser.

For frontend development with hot reload, run `npm run dev` inside `frontend/` (Vite runs on `http://localhost:5173` and proxies `/api/*` to `http://127.0.0.1:8000`).

---

## API Overview

| Method | Endpoint | Purpose |
| :--- | :--- | :--- |
| `GET` | `/api/health` | Check server status and configured LLM/voice models |
| `POST` | `/api/parse-profile` | Parse resume text/file (`.pdf`, `.txt`, `.md`) and JD into role & competencies |
| `POST` | `/api/session/start` | Initialize an `InterviewSession` and return the first question |
| `GET` | `/api/session/{session_id}` | Fetch active in-memory session state |
| `POST` | `/api/session/{session_id}/answer` | Submit text or audio answer; returns transcript, rubric scores, and coaching |
| `POST` | `/api/session/{session_id}/next` | Advance to the next adaptive question or finish and return debrief |
| `POST` | `/api/session/{session_id}/followup` | Generate a targeted follow-up probe on the last answer |
| `POST` | `/api/session/{session_id}/finish` | End session early and generate AI debrief |
| `GET` | `/api/history` | Retrieve persisted answers and per-competency statistics |
| `DELETE` | `/api/history/{answer_id}` / `/api/history` | Delete one answer or clear all persisted history |
| `GET` / `POST` | `/api/question-bank` | List indexed seed questions or add & index a new custom question |

---

## Known Limitations & Next Steps

- Active `InterviewSession` instances are held in server memory (`SESSIONS` dict), so restarting the server clears in-progress sessions while completed answers and session records persist in `memory.sqlite3`.
- Generated TTS `.mp3` files accumulate in `static/audio/` over time and are not automatically pruned.
- Adaptive competency targeting uses a deterministic lowest-historical-average heuristic rather than an adaptive difficulty/IRT model.
- Single-user local architecture without authentication or multi-tenant isolation.
