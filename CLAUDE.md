# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Always use `uv` for Python — never invoke `pip`, `pip3`, `python`, or `python3` directly.** All package management, virtualenv handling, and script execution must go through `uv`. This is non-negotiable: `pyproject.toml` + `uv.lock` are the source of truth, and `pip` would bypass the lockfile and the managed venv.

Dependency management (all via `uv`):
```bash
uv sync                    # install/sync deps from pyproject.toml + uv.lock
uv add <pkg>               # add a new dependency (updates pyproject.toml + uv.lock)
uv add --dev <pkg>         # add a dev-only dependency
uv remove <pkg>            # remove a dependency
uv lock                    # refresh the lockfile without installing
uv tree                    # inspect the resolved dep graph
```
Never edit `pyproject.toml` deps by hand and never run `pip install` — let `uv add`/`uv remove` mutate both files together so the lockfile stays consistent.

Environment setup:
```bash
cp .env.example .env       # then set ANTHROPIC_API_KEY
```

Run the app (Python 3.13):
```bash
./run.sh                                                      # preferred
cd backend && uv run uvicorn app:app --reload --port 8000     # manual
```
Run any other script with `uv run <cmd>` so it executes inside the managed venv.
Web UI at http://localhost:8000, OpenAPI docs at http://localhost:8000/docs.

There is **no test suite, linter, or formatter configured** — don't invent commands for them. `main.py` at the repo root is an unused stub; the real entrypoint is `backend/app.py`.

## Architecture

This is a tool-augmented RAG system, **not** a classic always-retrieve RAG. Claude decides per-query whether to call the search tool; general-knowledge questions are answered without retrieval. Understanding this is essential before changing retrieval behavior.

### Request flow (one query)

```
frontend/script.js  POST /api/query {query, session_id}
  → backend/app.py:query_documents
    → RAGSystem.query (rag_system.py)
      → SessionManager.get_conversation_history     (prior turns as plain text)
      → AIGenerator.generate_response               ── Claude call #1 (with tools)
          if stop_reason == "tool_use":
            → ToolManager.execute_tool
              → CourseSearchTool.execute
                → VectorStore.search
                    → course_catalog.query   (semantic course-name resolution)
                    → course_content.query   (filtered semantic search, top 5)
                ← formatted hits + last_sources stashed on the tool
          ── Claude call #2 (no tools, synthesizes final answer)
      → ToolManager.get_last_sources / reset_sources
      → SessionManager.add_exchange
    ← (answer, sources)
```

Two Claude calls per tool-using query — one to choose+invoke the tool, a second (without tools) to synthesize the answer from the tool result. The system prompt in `ai_generator.py` enforces **"one search per query maximum"**; the second call deliberately omits `tools` so Claude cannot chain searches. Don't loosen this without revisiting the prompt.

### Two ChromaDB collections (vector_store.py)

- `course_catalog` — one row per course. Document text = course title. Metadata holds instructor, course link, and `lessons_json` (a JSON-serialized list of `{lesson_number, lesson_title, lesson_link}`). Used to resolve fuzzy course names like `"MCP"` → the exact stored title before content search.
- `course_content` — one row per chunk. Metadata = `{course_title, lesson_number, chunk_index}`. Content search filters by the resolved `course_title` and/or `lesson_number` via a ChromaDB `where` clause.

Both embed with `all-MiniLM-L6-v2` via `SentenceTransformerEmbeddingFunction`. The split exists so a query like *"what does lesson 2 of the MCP course say about X"* can do name resolution and content retrieval as separate semantic operations.

**Course title is the primary key** in both the catalog (row id) and content chunk ids (`<title_with_underscores>_<chunk_index>`). `add_course_folder` skips files whose title is already present, so re-ingestion is idempotent on title — but renaming a course will create a duplicate, not update.

### Document ingestion (document_processor.py)

Course files in `docs/` follow a strict header format:
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>          ← optional, consumed if present
<body...>

Lesson 1: ...
```
Parser starts lesson scanning at line 4 (or 5 if line 4 is blank). Lessons are detected by the regex `^Lesson\s+(\d+):\s*(.+)$`. If no lesson markers are found the entire body is chunked as one lesson-less course.

Chunking is sentence-based with overlap (`CHUNK_SIZE=800`, `CHUNK_OVERLAP=100` chars in `config.py`). The sentence splitter respects abbreviations (`U.S.`, `Dr.`). There is a known **inconsistency**: mid-document lessons prepend `"Lesson N content: "` only to the *first* chunk, but the *last* lesson prepends `"Course <title> Lesson N content: "` to *every* chunk (`document_processor.py:184` vs `:234`). This affects embeddings; preserve or fix deliberately.

Despite `rag_system.py:81` filtering `.pdf`/`.docx`/`.txt`, `read_file` only does plain text — only `.txt` actually works.

### Tool system (search_tools.py)

Tools implement an `ABC` (`Tool.get_tool_definition` + `execute`). `ToolManager` registers them, exposes Anthropic tool definitions, and dispatches by name. Sources flow back to the API via a side channel: `CourseSearchTool` stashes `self.last_sources` during `_format_results`, and `RAGSystem.query` reads then resets it after each request. If you add another search-like tool, give it a `last_sources` attribute or `get_last_sources` will silently miss it.

To add a tool: subclass `Tool`, register it in `RAGSystem.__init__` next to `CourseSearchTool`. The `tool_choice={"type":"auto"}` setting in `ai_generator.py` lets Claude pick.

### Session state

`SessionManager` is **in-memory only** (`Dict[str, List[Message]]`); restarting the server drops all conversations. `MAX_HISTORY=2` exchanges, kept as `max_history * 2 = 4` messages. History is injected into the system prompt as plain `Role: content` lines, not as proper message turns.

### Frontend

Single-page vanilla JS in `frontend/`, served as static files by FastAPI itself (`app.py:119`). `script.js` renders assistant messages through `marked.parse()` **without sanitization** — the app trusts Claude's markdown output. Sources are rendered as a comma-joined string inside a collapsible `<details>`; they are plain `"Course - Lesson N"` labels, not links, even though `vector_store.py` has unused `get_course_link` / `get_lesson_link` helpers.

`DevStaticFiles` (`app.py:107`) forces `Cache-Control: no-cache` so frontend edits show up on reload without bumping a version.

### Startup ingestion

`@app.on_event("startup")` calls `rag_system.add_course_folder("../docs", clear_existing=False)` (`app.py:88`). The relative path means **the server must be started from `backend/`** — `run.sh` does this; running uvicorn from the repo root would silently load nothing.

### Config

All knobs are in `backend/config.py` (`Config` dataclass). Notable: `ANTHROPIC_MODEL = "claude-sonnet-4-20250514"`, `MAX_RESULTS=5`, ChromaDB persists at `backend/chroma_db/` (gitignored). To force a re-ingest after changing chunking logic, delete that directory or call `add_course_folder(..., clear_existing=True)`.
