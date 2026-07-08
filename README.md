# Study Buddy Agent

A multi-agent adaptive learning system built with [Google ADK](https://google.github.io/adk-docs/). Give it a topic (and optionally a PDF), and it designs a study plan, tutors you on the material, quizzes you with LLM-as-judge grading, and tracks your mastery over time using spaced repetition.

---

## What it does

1. **Ingest** — pulls source material from web search and/or a PDF you provide, chunks it into a knowledge base
2. **Plan** — a Curriculum Agent produces a sequenced study plan grounded in the source material
3. **Tutor** — a Tutor Agent answers your questions with context from the knowledge base
4. **Quiz** — a Quiz Generation Agent creates MCQ and open-ended questions; a Quiz Grading Agent scores open-ended answers using LLM-as-judge
5. **Remember** — a Memory Store tracks per-topic mastery with an exponential moving average and flags topics for spaced-repetition review

---

## Architecture

```
StudySession (orchestrator)
├── Ingestion          web search + PDF → SourceChunks
├── Curriculum Agent   structured StudyPlan JSON
├── Tutor Agent        RAG-lite explanation and Q&A
├── Quiz Gen Agent     MCQ + open-ended QuizSet JSON
├── Quiz Grading Agent LLM-as-judge → GradedAnswer JSON
└── Memory Store       JSON persistence + spaced repetition
```

All agents are built with Google ADK (`google-adk`). Auth auto-detects between Vertex AI (Application Default Credentials) and Gemini API key. The orchestrator (`StudySession`) is plain Python — not an ADK workflow — because the session loop depends on real user input.

---

## Requirements

- Python 3.10 or later
- A Google Cloud project with the [Vertex AI API enabled](https://console.cloud.google.com/apis/library/aiplatform.googleapis.com) **or** a [Gemini API key](https://aistudio.google.com/apikey)

---

## Installation

```bash
git clone https://github.com/<your-org>/study-buddy-agent.git
cd study-buddy-agent

python3 -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

---

## Auth setup

The app supports two auth paths, tried in this order:

### Option A — Vertex AI (gcloud ADC)

Suitable for local development with a GCP account that has Vertex AI + Gemini API access.

```bash
gcloud auth application-default login
gcloud config set project <your-gcp-project-id>
gcloud auth application-default set-quota-project <your-gcp-project-id>
```

The app sets `GOOGLE_GENAI_USE_VERTEXAI=True` automatically and defaults to `us-central1`. To override the region:

```bash
export GOOGLE_CLOUD_LOCATION=us-east4
```

> **Note:** Gemini publisher models (`gemini-1.5-flash`, `gemini-2.0-flash-001`, etc.) must be enabled for your project. If you see a `404 NOT_FOUND` error for a publisher model, verify the Vertex AI API is enabled and your project has been granted Gemini model access by your GCP admin.

### Option B — Gemini API key

Suitable for Kaggle notebooks or anywhere you have a Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey).

```bash
export GOOGLE_API_KEY="your-api-key-here"
```

Or copy `.env.example` to `.env` and fill in the key:

```bash
cp .env.example .env
# edit .env and set GOOGLE_API_KEY
```

---

## Running the notebook

The notebook (`study_buddy_agent.ipynb`) is self-contained and walks through a complete study session on Transformer attention mechanisms.

```bash
source .venv/bin/activate
jupyter notebook study_buddy_agent.ipynb
```

Run cells top to bottom. The **Model Selection** section (after Auth) lists available Gemini models for your project and lets you set `MODEL_NAME` before any agents are constructed. `gemini-1.5-flash` is the recommended default — it is available in all Vertex AI regions.

**Kernel restart:** if you change auth or model settings mid-session, use **Kernel → Restart & Run All** to ensure the new values take effect.

---

## Running the CLI

The `study_buddy/cli.py` entry point runs an interactive terminal session.

```bash
source .venv/bin/activate
python -m study_buddy.cli
```

You will be prompted for a topic and an optional PDF path. The CLI runs all five pipeline stages and drops into an interactive Q&A loop after the study plan is generated.

---

## Project structure

```
study_buddy/
├── agents/
│   ├── curriculum_agent.py   # builds structured StudyPlan from source material
│   ├── tutor_agent.py        # answers questions grounded in knowledge base
│   ├── quiz_agent.py         # generates MCQ and open-ended questions
│   ├── orchestrator.py       # StudySession: ties all agents together
│   ├── runner_utils.py       # synchronous wrapper around ADK InMemoryRunner
│   └── schemas.py            # Pydantic output schemas (StudyPlan, QuizSet, etc.)
├── memory/
│   └── store.py              # MemoryStore: JSON persistence + spaced repetition
├── tools/
│   └── ingestion.py          # PDF extraction + web search → SourceChunks
├── config/
│   └── models.py             # auth detection and model configuration
└── cli.py                    # interactive terminal entry point

tests/
├── test_ingestion.py
├── test_memory_store.py
└── test_model_config.py

study_buddy_agent.ipynb       # self-contained Kaggle/Jupyter demo notebook
requirements.txt
.env.example
```

---

## Key design decisions

**Why multi-agent?** Each agent has a single, testable responsibility. The Curriculum Agent only designs plans; the Quiz Grading Agent only scores answers. This makes it straightforward to swap prompts or models per agent without affecting others, and to unit-test each agent independently by mocking the LLM.

**LLM-as-judge grading:** MCQ questions are graded deterministically (zero cost, zero latency). Open-ended answers are sent to a dedicated grading agent with a rubric-style instruction. This is the [LLM-as-judge eval pattern](https://arxiv.org/abs/2306.05685): using a second model call to score a first model's output against a reference answer.

**Spaced repetition:** `MemoryStore` tracks `topic → TopicRecord` in a JSON file. Each record stores quiz score history (exponential moving average → `mastery`) and a `last_reviewed` date. Mastery decays by 0.02 per day; any topic below 0.7 effective mastery is flagged for review at session start. This is a deliberate simplification of the Leitner / spaced-repetition principle — enough to demonstrate adaptive behavior without a heavy SRS library dependency.

**Async/sync bridge:** The ADK runner is async internally. `runner_utils.py` runs each agent turn in a dedicated thread with its own event loop (`asyncio.run()` via `ThreadPoolExecutor`), ensuring session creation and execution share a single event loop and avoiding cross-loop session lookup failures in Jupyter's already-running event loop.

---

## What to improve next

- **Vector search** — swap the flat chunk list for an embedding index (FAISS or ChromaDB) for semantic retrieval instead of truncated concatenation
- **Gemini Files API** — for large PDFs, use `google.generativeai.upload_file()` so the document lives server-side and doesn't consume context window
- **Persistent sessions** — replace `InMemoryRunner` with a `DatabaseSessionService` so tutor conversations survive across notebook runs
- **Confidence calibration** — detect when the grading agent systematically over- or under-scores and adjust the rubric accordingly

---

## License

MIT
