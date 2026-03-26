# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Install dependencies:**
```bash
uv sync
```

**Run the application:**
```bash
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

**Access points:**
- Web UI: `http://localhost:8000`
- API docs (Swagger): `http://localhost:8000/docs`

**Environment setup:** Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`.

## Architecture

This is a RAG (Retrieval-Augmented Generation) chatbot for answering questions about course materials. The backend is FastAPI + ChromaDB + Anthropic Claude; the frontend is plain HTML/JS/CSS served statically by FastAPI.

### Data Flow

1. Course `.txt` files in `docs/` are parsed by `backend/document_processor.py` on startup
2. Documents are chunked and stored in ChromaDB (two collections: `course_catalog` for metadata, `course_content` for text)
3. User queries go to `POST /api/query` → `RAGSystem.query()` → `AIGenerator.generate_response()`
4. Claude receives two tools (`search_course_content`, `get_course_outline`) and decides when to use them
5. Tool calls execute semantic search via ChromaDB; results feed back to Claude for final answer generation
6. Sources (lesson links) are extracted from tool execution and returned alongside the answer

### Key Backend Files

- `backend/app.py` — FastAPI app, routes, startup document loading, static file serving
- `backend/rag_system.py` — Central orchestrator tying together all components
- `backend/ai_generator.py` — Anthropic API calls; handles tool-use loop (model: `claude-sonnet-4-20250514`, temp: 0, max_tokens: 800)
- `backend/search_tools.py` — Tool definitions (`CourseSearchTool`, `CourseOutlineTool`) and `ToolManager`
- `backend/vector_store.py` — ChromaDB wrapper; two collections, embedding model `all-MiniLM-L6-v2` (384-dim)
- `backend/document_processor.py` — Parses course `.txt` files with specific format (Course Title/Link/Instructor header + Lesson blocks); sentence-based chunking
- `backend/session_manager.py` — In-memory conversation history; keyed by session ID, trimmed to `MAX_HISTORY * 2` messages
- `backend/config.py` — Central config (chunk size: 800, overlap: 100, max results: 5, max history: 2)
- `backend/models.py` — Pydantic models: `Course`, `Lesson`, `CourseChunk`

### Course Document Format

Files in `docs/` must follow this structure for `document_processor.py` to parse them:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [instructor]
Lesson 0: [Title]
Lesson Link: [url]
[content...]
Lesson 1: [Title]
...
```

### ChromaDB Persistence

The vector database is stored at `backend/chroma_db/`. On startup, `app.py` checks existing course titles and skips re-processing already-loaded courses. To force a reload, clear the `chroma_db/` directory.

### Session Management

Sessions are in-memory only (lost on server restart). Session IDs are returned from `/api/query` and sent back by the frontend on subsequent requests. Clear a session via `DELETE /api/session/{session_id}`.
