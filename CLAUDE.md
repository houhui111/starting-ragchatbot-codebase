# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Install dependencies (always use `uv`, never `pip`):**
```bash
uv sync          # install from uv.lock
uv add <pkg>     # add a new dependency
uv remove <pkg>  # remove a dependency
```

**Run the application (always use `uv`, never `pip` or plain `python`):**
```bash
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

**Access points:**
- Web UI: http://localhost:8000
- API docs: http://localhost:8000/docs

**Environment setup:** Copy `.env.example` to `.env` and add your `ANTHROPIC_API_KEY`.

## Architecture

This is a RAG (Retrieval-Augmented Generation) chatbot for course materials. The backend is FastAPI + ChromaDB + Anthropic Claude; the frontend is vanilla HTML/CSS/JS.

### Data Flow

```
User Query → POST /api/query → RAGSystem.query()
  → Claude API (function calling) → CourseSearchTool.search()
  → ChromaDB vector search → Claude synthesizes response
  → answer + sources → Frontend
```

### Key Backend Modules (`backend/`)

- **`rag_system.py`** — Central orchestrator; coordinates document ingestion, search, and AI generation. Entry point for all queries.
- **`ai_generator.py`** — Wraps Anthropic Claude with tool-use (function calling). Claude calls `CourseSearchTool` to retrieve relevant chunks rather than receiving them directly in the prompt. Model: `claude-sonnet-4-20250514`, temp=0, max_tokens=800.
- **`vector_store.py`** — ChromaDB with two collections: `course_catalog` (titles/metadata) and `course_content` (chunked text). Handles fuzzy course name resolution and lesson filtering.
- **`document_processor.py`** — Parses course documents expecting this format:
  ```
  Course Title: ...
  Course Link: ...
  Course Instructor: ...
  Lesson N: Title
  [content]
  ```
  Chunks at 800 chars with 100-char overlap using sentence-aware splitting.
- **`search_tools.py`** — Tool interface for Claude function calling; wraps vector search with optional course/lesson filters. Tracks sources for UI display.
- **`session_manager.py`** — In-memory per-session conversation history (default: 2 exchanges max).
- **`config.py`** — All tuneable constants (chunk size, model name, max results, etc.).

### API Endpoints

- `POST /api/query` — `{query, session_id?}` → `{answer, sources, session_id}`
- `GET /api/courses` — returns course count and titles
- `GET /` — serves the frontend static files

### Frontend (`frontend/`)

Vanilla JS with no build step. Uses `marked.js` for markdown rendering. Communicates with backend via relative URLs so it works behind proxies.

### Storage

ChromaDB persists to `backend/chroma_db/`. Delete this directory to reset the vector database. Sample course documents live in `docs/` and are auto-loaded on startup.
