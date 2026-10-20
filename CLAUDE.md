# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Dependencies are managed with **uv** (Python 3.13). Use `uv add <pkg>` / `uv run …`, not pip.

```bash
uv sync                      # install dependencies into .venv
./run.sh                     # start the app (cd backend && uvicorn app:app --reload --port 8000)
cd backend && uv run uvicorn app:app --reload --port 8000   # manual start
```

- Web UI: http://localhost:8000 · API docs: http://localhost:8000/docs
- Requires `ANTHROPIC_API_KEY` in `.env` at the repo root (template: `.env.example`).
- On Windows, run commands in Git Bash.
- There are no tests, linter or formatter configured. `main.py` is an unused placeholder.

## Architecture

A FastAPI app that answers questions about course transcripts using Claude with a single search tool over ChromaDB. The same server serves the static frontend (`frontend/`, mounted at `/` in `backend/app.py`) and the API (`POST /api/query`, `GET /api/courses`).

**The server must run from `backend/`**: paths are relative to it (`../docs`, `../frontend`, `./chroma_db`).

### Query flow (tool-based RAG)

`app.py` → `RAGSystem.query()` → `AIGenerator.generate_response()`:

1. First Claude call with the `search_course_content` tool and `tool_choice: auto`. Claude decides whether to search.
2. If `stop_reason == "tool_use"`, `_handle_tool_execution()` runs the tool via `ToolManager`, then makes a **second call without tools**. This caps each query at one search round.
3. `RAGSystem` reads sources from `ToolManager.get_last_sources()` (the `last_sources` attribute on the tool), resets them, and stores the exchange in `SessionManager`.

Details that span files:

- Conversation history is injected as text into the **system prompt** (not the `messages` array). Sessions are in-memory only and keep the last `MAX_HISTORY` exchanges (`config.py`).
- `AIGenerator` returns `response.content[0].text`, assuming the first content block is text. It also sends `temperature: 0`. Both matter if you change `ANTHROPIC_MODEL`: newer models that think by default or reject non-default sampling parameters will break this code.
- New tools subclass `Tool` in `search_tools.py` and are registered on `ToolManager` in `RAGSystem.__init__`. Expose a `last_sources` list if the tool's results should appear as UI sources. Each source is a `{"text", "link"}` dict (`Source` model in `app.py`); the frontend renders it as a new-tab link when `link` is set.

### Vector store (`vector_store.py`)

Two ChromaDB collections, both using the local `all-MiniLM-L6-v2` SentenceTransformer embeddings:

- `course_catalog`: one document per course (the title), with lessons serialized as JSON in `lessons_json` metadata. It is used to **resolve fuzzy course names** (e.g. "MCP") to the exact title via semantic search.
- `course_content`: text chunks with `course_title`, `lesson_number` and `chunk_index` metadata, filtered with a `where` clause built by `_build_filter()`.

The course title is the unique ID throughout (catalog ID, chunk metadata, chunk IDs `<title_with_underscores>_<index>`).

### Ingestion (`document_processor.py`)

At startup, `app.py` loads every `.txt`/`.pdf`/`.docx` in `docs/` and **skips courses whose title already exists** in ChromaDB. Editing a transcript does not re-index it; delete `backend/chroma_db/` (or call `add_course_folder(..., clear_existing=True)`) to rebuild.

Expected file format:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>
<transcript text…>
Lesson 1: …
```

Chunking is sentence-based (`CHUNK_SIZE` 800 chars, `CHUNK_OVERLAP` 100). Known inconsistency: for most lessons, only the first chunk gets a `Lesson N content:` prefix, but every chunk of the **last** lesson gets `Course <title> Lesson N content:`.
