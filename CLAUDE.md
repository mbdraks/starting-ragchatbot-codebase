# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Dependencies and runtime are managed with `uv` (Python 3.13+ required). The `.env` file at the repo root must contain `ANTHROPIC_API_KEY`.

**Always use `uv` to run Python files and manage dependencies — never `python`/`pip` directly.**

- Install deps: `uv sync`
- Add a dependency: `uv add <package>` (do not edit `pyproject.toml` by hand or use `pip install`)
- Run a Python file/module: `uv run python <file>` or `uv run <command>`
- Run the server (preferred): `./run.sh`
- Run the server (manual): `cd backend && uv run uvicorn app:app --reload --port 8000`
- App URL: http://localhost:8000 (UI) and http://localhost:8000/docs (FastAPI Swagger)

The server **must** be launched from `backend/` — paths like `../docs`, `../frontend`, and `./chroma_db` are resolved relative to CWD.

There is no test suite, linter, or formatter configured. Do not invent commands for these.

## Architecture

This is a tool-calling RAG system, not a classic "embed-the-query, stuff-the-context" pipeline. The Claude model decides whether and how to retrieve, by invoking tools the backend exposes.

### Backend modules (`backend/`)

- `app.py` — FastAPI app, endpoints, middleware, static mount, startup ingestion.
- `rag_system.py` — orchestrator wiring `DocumentProcessor`, `VectorStore`, `AIGenerator`, `SessionManager`, and `ToolManager`.
- `ai_generator.py` — Claude API client; one tool-use round + follow-up synthesis call.
- `search_tools.py` — `Tool` base class, `CourseSearchTool`, `CourseOutlineTool`, and `ToolManager` registry.
- `vector_store.py` — ChromaDB wrapper, two collections, fuzzy course-name resolution.
- `document_processor.py` — parses course files and does sentence-aware chunking.
- `session_manager.py` — issues sequential session IDs and stores capped conversation history.
- `models.py` — Pydantic models: `Course`, `Lesson`, `CourseChunk`.
- `config.py` — single `Config` dataclass for all tunables.

### Query flow

1. `POST /api/query` → [backend/app.py](backend/app.py) → `RAGSystem.query()` in [backend/rag_system.py](backend/rag_system.py).
2. `RAGSystem` builds a prompt, pulls conversation history from `SessionManager`, and calls `AIGenerator.generate_response()` with the registered tool definitions.
3. [backend/ai_generator.py](backend/ai_generator.py) sends the request to Claude with `tool_choice: auto`. If `stop_reason == "tool_use"`, `_handle_tool_execution` runs the tool(s) via `ToolManager`, appends results, and makes a **second** call (without tools) for the final synthesized answer. There is currently only one round of tool calls — no agentic loop.
4. After the answer comes back, `RAGSystem` pulls `last_sources` from the tool that ran (see "Sources tracking" below), resets them, and stores the exchange in session history.

`GET /api/courses` is a separate, non-LLM endpoint: it calls `RAGSystem.get_course_analytics()` (which reads `course_catalog` metadata) and returns `{total_courses, course_titles}`. Used by the frontend to render the course list.

Anything that needs to change *what the model retrieves* should be done by editing the tool definitions or system prompt in [backend/ai_generator.py](backend/ai_generator.py) — not by pre-fetching context before the LLM call.

### Tools (`backend/search_tools.py`)

Two tools are registered at startup in `RAGSystem.__init__`:

- `search_course_content` — semantic search over chunked lesson text, with optional `course_name` (fuzzy-resolved) and `lesson_number` filters.
- `get_course_outline` — returns the course title, link, and ordered lesson list from catalog metadata.

Adding a tool: subclass `Tool`, implement `get_tool_definition()` (Anthropic tool-use schema) and `execute()`, then register it in `RAGSystem.__init__`. If the tool produces UI-visible sources, set `self.last_sources` inside `execute()`.

### Vector store layout (`backend/vector_store.py`)

ChromaDB persists at `backend/chroma_db/` and uses **two collections** with distinct purposes:

- `course_catalog` — one document per course (the course title is both the document body and the ChromaDB ID). Metadata holds instructor, course link, lesson count, and a JSON-serialized `lessons_json` array (Chroma metadata is flat, so nested data is stringified). This collection is queried only to resolve fuzzy course names → canonical titles, and to look up outlines/links.
- `course_content` — one document per text chunk. Metadata: `course_title`, `lesson_number`, `chunk_index`. Filtered queries combine these via `$and`.

Embeddings: `sentence-transformers/all-MiniLM-L6-v2` for both collections.

A course is identified by its title. Re-ingesting the same course is a no-op (`add_course_folder` checks `existing_course_titles` and skips).

### Document ingestion (`backend/document_processor.py`)

Course files in `docs/` are expected to follow a fixed format — the parser depends on it:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>
<lesson body...>

Lesson 1: <lesson title>
...
```

Chunking is sentence-aware with configurable overlap (`config.CHUNK_SIZE=800`, `config.CHUNK_OVERLAP=100`). The first chunk of each lesson is prefixed with `"Lesson {N} content: "` (and last-lesson chunks with `"Course {title} Lesson {N} content: "`) to give the embedding additional context — keep this in mind if changing the chunking logic, since search relevance depends on it.

On startup, [backend/app.py](backend/app.py) calls `add_course_folder("../docs", clear_existing=False)`, so adding files to `docs/` and restarting is enough to ingest them. To force a rebuild, delete `backend/chroma_db/`.

### Sources tracking

Sources are surfaced to the UI via a side channel, not the LLM response:

1. When a tool runs, it stores human-readable source strings in `self.last_sources`.
2. After `AIGenerator` returns, `RAGSystem.query()` calls `ToolManager.get_last_sources()` (which scans tools for the first non-empty `last_sources`) and then `reset_sources()`.
3. Sources are returned alongside `answer` in `QueryResponse`.

This is why there is no `sources` argument threaded through the LLM call path — the tools mutate state that the orchestrator reads after the fact.

### Sessions (`backend/session_manager.py`)

- IDs are sequential strings: `session_1`, `session_2`, … (counter resets when the process restarts; sessions are in-memory only).
- Per session, `add_exchange()` records a user/assistant pair; the deque is trimmed to the last `MAX_HISTORY` exchanges (i.e., `MAX_HISTORY * 2` messages).
- `get_conversation_history()` returns a flat string formatted as `"User: ...\nAssistant: ..."` per line. `RAGSystem` folds this into the **system prompt** for the next call — it is not threaded into the `messages` array.

### Frontend

[frontend/](frontend/) is plain HTML/CSS/JS, served as static files by FastAPI via `app.mount("/", StaticFiles(...))`. `script.js` calls `/api/query` and `/api/courses`. There is no build step. `DevStaticFiles` (subclass in [backend/app.py](backend/app.py)) sets `Cache-Control: no-cache` (plus `Pragma`/`Expires`) on static responses so frontend edits show up on reload.

The app also installs permissive `CORSMiddleware` (`allow_origins=["*"]`, credentials on) and `TrustedHostMiddleware` (`allowed_hosts=["*"]`) — fine for local dev, tighten before any deploy.

### Configuration

All tunables live in [backend/config.py](backend/config.py): `ANTHROPIC_MODEL` (currently `claude-haiku-4-5-20251001`), embedding model, chunk size/overlap, `MAX_RESULTS` (default 5), and `MAX_HISTORY` (default 2 exchanges = 4 messages). Change values here rather than threading constants through call sites.

### Repo layout notes

- `main.py` at the repo root is unused scaffolding. The real entry point is `backend/app.py`, launched via `./run.sh`.
- `.python-version` pins the interpreter for `uv` and aligns with the Python 3.13+ requirement in `pyproject.toml`.
- There is no `tests/`, `.github/workflows/`, or linter config — adding one is a project change, not a documentation gap.
