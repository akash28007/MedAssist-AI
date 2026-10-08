# MedAssist AI

A healthcare information chatbot that answers health questions from a local library of medical PDFs. It uses Retrieval-Augmented Generation (RAG) with Google Gemini, shows the source document and page for each answer, detects emergencies, and supports voice input and output in the browser.

> **Disclaimer:** MedAssist AI is for educational purposes only. It does not diagnose, prescribe, or replace professional medical advice.

---

## Features

- **Answers grounded in your documents.** Each question is embedded and matched against PDF chunks stored in ChromaDB. The top matches (5 by default) are passed to Gemini, and the response lists the source document and page for every chunk used.
- **Emergency detection.** Before any LLM call, the question is checked against 28 emergency keywords (e.g. *chest pain*, *choking*, *stroke*, *overdose*). On a match, the app returns emergency-services guidance (911 / 999 / 112) instead of an answer.
- **Clarifying questions.** Very short questions (under 3 words) or vague ones (e.g. *"Tell me about diabetes"*) get a follow-up question. Topics such as diabetes, pain, medication, vaccines and cancer have tailored follow-ups.
- **Conversation context.** The last 6 messages of the chat are included in the prompt, so follow-up questions work.
- **Voice input and output.** Uses the browser's built-in Web Speech API for speech-to-text and text-to-speech. No external voice service is involved; Chrome or Edge is recommended.
- **Safety instructions.** The system prompt tells the model to answer only from the provided context, never diagnose or prescribe, cite its sources, and end with an educational-use reminder.
- **Retry with backoff.** Failed LLM calls are retried with exponential backoff.

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS |
| AI service | Python, FastAPI, LangChain, LangGraph |
| LLM | Google Gemini (`gemini-2.5-flash` by default) |
| Embeddings | Sentence-Transformers `all-MiniLM-L6-v2` (runs locally on CPU) |
| Vector store | ChromaDB (persistent, on disk) |
| Gateway (optional) | NestJS |

## How It Works

```
Question (typed or spoken)
        │
        ▼
 Emergency keyword?  ──yes──►  Emergency-services guidance
        │ no
        ▼
 Too short / vague?  ──yes──►  Clarifying question
        │ no
        ▼
 Embed question → search ChromaDB (top 5 chunks)
        │
        ▼
 Gemini writes an answer from those chunks + recent chat
        │
        ▼
 Answer + sources (document, page) → shown in chat, can be read aloud
```

The routing step is a LangGraph state graph in [`agent.py`](backend/ai-service/agent.py). The RAG pipeline is in [`chain.py`](backend/ai-service/chain.py) and [`retriever.py`](backend/ai-service/retriever.py).

## Project Structure

```
MedAssist-AI/
├── frontend/                 # React + TypeScript chat UI
├── backend/
│   ├── ai-service/           # FastAPI service: routing, RAG, LLM
│   │   ├── main.py           # API endpoints
│   │   ├── agent.py          # Intent routing (escalate / clarify / answer)
│   │   ├── chain.py          # Prompt building + LLM calls
│   │   ├── retriever.py      # ChromaDB similarity search
│   │   ├── ingest.py         # PDF loading, chunking, embedding
│   │   ├── prompts/          # System prompt
│   │   └── tests/            # pytest suite
│   └── gateway/              # NestJS proxy (optional, not used by default)
├── knowledge_base/           # Medical PDFs (gitignored)
├── scripts/
│   └── create_sample_pdfs.py # Generates 5 sample medical PDFs
└── product-documents/        # PRD, architecture and design docs
```

---

## Getting Started

### Prerequisites

- **Python 3.12**
- **Node.js 18+**
- A **Gemini API key**, free from [Google AI Studio](https://aistudio.google.com/apikey)

### 1. Set up the AI service

```bash
cd backend/ai-service
python -m venv venv
venv\Scripts\activate             # Windows
# source venv/bin/activate        # macOS / Linux

pip install -r requirements.txt fpdf2
copy .env.example .env            # Windows  (macOS / Linux: cp .env.example .env)
```

Open `.env` and add your key:

```env
GEMINI_API_KEY=your_key_here
GEMINI_MODEL=gemini-2.5-flash
```

### 2. Build the knowledge base

```bash
# From the project root: create 5 sample PDFs
# (Cold & Flu, Diabetes, First Aid, Hypertension, Mental Health)
python scripts/create_sample_pdfs.py

# From backend/ai-service: chunk, embed and store them in ChromaDB
python ingest.py
```

To use your own documents, put PDFs in `knowledge_base/` and run `python ingest.py` again.

### 3. Start the AI service

```bash
cd backend/ai-service
python main.py
```

This runs at `http://localhost:8000`. Open `http://localhost:8000/health` to check it, or `http://localhost:8000/docs` for the interactive API docs.

### 4. Start the frontend

```bash
cd frontend
npm install
npm run dev
```

Open `http://localhost:5173`.

### 5. Gateway (optional)

The frontend talks to the AI service directly. The NestJS gateway is a separate proxy exposing `/api/health`, `/api/query` and `/api/ingest`:

```bash
cd backend/gateway
npm install
npm run start:dev                 # http://localhost:3001
```

---

## Configuration

All settings live in `backend/ai-service/.env`:

| Variable | Default | Description |
|----------|---------|-------------|
| `GEMINI_API_KEY` | — | **Required.** Google Gemini API key |
| `GEMINI_MODEL` | `gemini-2.5-flash` | Gemini model name |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Local embedding model |
| `CHROMA_PERSIST_DIR` | `./chroma_db` | Where ChromaDB stores data |
| `CHROMA_COLLECTION_NAME` | `medassist_docs` | ChromaDB collection |
| `CHUNK_SIZE` / `CHUNK_OVERLAP` | `600` / `100` | Text chunking (characters) |
| `TOP_K_RESULTS` | `5` | Chunks retrieved per question |
| `HOST` / `PORT` | `0.0.0.0` / `8000` | Server address |
| `ADMIN_SECRET` | *(empty)* | When set, `/ingest` and `/upload` require it in the `X-Admin-Key` header |

The frontend reads `VITE_API_URL` (default `http://127.0.0.1:8000`).

## API

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | Service status and number of indexed chunks |
| `POST` | `/query` | `{ "question": "...", "chat_history": [...] }` → answer, intent, sources |
| `POST` | `/ingest` | Re-ingest all PDFs in `knowledge_base/` |
| `POST` | `/upload` | Upload a single PDF (multipart form) and ingest it |

## Running Tests

```bash
cd backend/ai-service
pytest
```
