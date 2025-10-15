# Copilot Instructions for Course Materials RAG System

## Project Architecture
- **Backend (`backend/`)**: FastAPI app orchestrates a Retrieval-Augmented Generation (RAG) pipeline for course Q&A.
  - `app.py`: FastAPI entrypoint, exposes API endpoints, initializes RAGSystem.
  - `rag_system.py`: Main orchestrator; wires together document processing, vector storage, AI generation, session management, and search tools.
  - `document_processor.py`: Splits course documents into overlapping text chunks for semantic search.
  - `vector_store.py`: Uses ChromaDB and SentenceTransformers for vector search and storage.
  - `ai_generator.py`: Handles Anthropic Claude API calls, enforces strict prompt/response protocol.
  - `search_tools.py`: Defines semantic search tools for Anthropic tool use.
  - `config.py`: Centralizes config, loads `.env` for secrets.
  - `models.py`: Pydantic models for course, lesson, and chunk data.
- **Frontend (`frontend/`)**: Static HTML/JS/CSS for user interaction.
- **Course Documents (`docs/`)**: Plain text scripts for each course, loaded and chunked by backend.

## Key Workflows
- **Install dependencies**: `uv sync` (uses [uv](https://github.com/astral-sh/uv) for Python package management)
- **Run server**: `./run.sh` (preferred) or manually via `uv run uvicorn app:app --reload --port 8000` from `backend/`
- **Environment**: Set `ANTHROPIC_API_KEY` in `.env` at project root.
- **Add course docs**: Place `.txt` files in `docs/`; backend will process and chunk on startup or via API.

## Patterns & Conventions
- **Chunking**: Documents are split into ~800-character chunks with 100-character overlap for semantic search.
- **Vector Search**: ChromaDB stores chunk embeddings; queries use SentenceTransformers (`all-MiniLM-L6-v2`).
- **AI Generation**: All responses use Anthropic Claude, with a strict system prompt (see `ai_generator.py`).
- **Search Tool Protocol**: Only one semantic search per query; results are synthesized, not quoted. No meta-commentary in responses.
- **Config**: All tunable parameters (chunk size, model names, etc.) are in `config.py`.
- **Session Management**: Only last 2 messages are remembered for context.

## Integration Points
- **ChromaDB**: Local vector DB for semantic search.
- **Anthropic Claude**: External API for AI responses (key required).
- **FastAPI**: Serves both API and static frontend.

## Examples
- To add a new course: Drop a `.txt` file in `docs/`, restart backend.
- To change chunking: Edit `CHUNK_SIZE`/`CHUNK_OVERLAP` in `config.py`.
- To update AI model: Change `ANTHROPIC_MODEL` in `config.py`.

## Tips for AI Agents
- Always use the orchestrator (`RAGSystem`) for cross-component logic.
- Respect the single-search-per-query rule in `ai_generator.py`.
- Reference `config.py` for any parameter changes.
- Use Pydantic models for all data passed between components.
- Avoid direct DB/file access; use provided abstractions.

---
If any section is unclear or missing, please request clarification or provide feedback for improvement.