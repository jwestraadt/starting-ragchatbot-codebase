# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Run the application** (from project root, requires Git Bash on Windows):
```bash
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

**Only use `uv` to manage dependencies — never use `pip` or any other package manager:**
```bash
uv add <package>       # add a package
uv remove <package>    # remove a package
uv sync                # install/sync all dependencies
```

**Run a Python script in the project environment:**
```bash
uv run python <script.py>
```

The app is available at `http://localhost:8000`. There are no tests or linting configs defined in this project.

## Environment

Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. The server must be started from the `backend/` directory (or via `run.sh`) because config paths like `../docs` and `./chroma_db` are relative to that directory.

## Architecture

This is a RAG chatbot where Claude uses a **tool-calling loop** to search course materials before answering. The backend serves the frontend as static files, so there is only one server.

**Query flow:**
1. Browser POSTs to `/api/query`
2. `RAGSystem` (`rag_system.py`) orchestrates the pipeline
3. `AIGenerator` (`ai_generator.py`) makes a first Claude API call, providing the `search_course_content` tool definition
4. Claude calls the tool → `CourseSearchTool` (`search_tools.py`) executes a semantic search via `VectorStore` (`vector_store.py`) against ChromaDB
5. Search results are returned to Claude in a second API call, which produces the final answer
6. `SessionManager` (`session_manager.py`) stores the last 2 exchanges in memory for conversation context

**Document ingestion** (runs once at startup, skips already-indexed courses):
- `DocumentProcessor` (`document_processor.py`) parses `.txt`/`.pdf`/`.docx` files from `docs/`
- Expected format: 3-line header (`Course Title:`, `Course Link:`, `Course Instructor:`), then `Lesson N: <title>` markers with optional `Lesson Link:` lines
- Text is split into sentence-based chunks (800 chars, 100 char overlap)
- `VectorStore` stores two ChromaDB collections: `course_catalog` (one doc per course, used for fuzzy course name resolution) and `course_content` (one doc per chunk, used for semantic search)
- Embeddings are generated locally using `all-MiniLM-L6-v2` (no API call)
- ChromaDB persists to `backend/chroma_db/` (gitignored)

**Key configuration** (`backend/config.py`):
- Model: `claude-sonnet-4-20250514`
- Chunk size: 800 chars, overlap: 100 chars
- Max search results: 5, max conversation history: 2 exchanges
