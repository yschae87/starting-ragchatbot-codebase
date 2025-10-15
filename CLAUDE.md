# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A full-stack RAG (Retrieval-Augmented Generation) system for answering questions about course materials using semantic search and Claude AI. The system processes course transcripts, stores them as vector embeddings, and provides intelligent Q&A with source citations.

## Development Commands

### Setup
```bash
# Install uv package manager (first time only)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync

# Set up environment
cp .env.example .env
# Edit .env and add your ANTHROPIC_API_KEY
```

### Running the Application
```bash
# Recommended: Use the provided script
./run.sh

# Alternative: Manual start
cd backend && uv run uvicorn app:app --reload --port 8000

# Access points:
# - Web UI: http://localhost:8000
# - API Docs: http://localhost:8000/docs
```

### Working with Course Documents
```bash
# Add new course documents
# 1. Place .txt files in docs/ folder
# 2. Server will auto-process on startup (see backend/app.py:88-98)
# 3. Or restart server to reload

# Clear and rebuild vector database
# Restart server with clear_existing=True in app.py startup_event
```

## Architecture Overview

### Request Flow Architecture
```
Frontend (JS) → FastAPI → RAG System → AI Generator ↔ Tool Manager
                                           ↓              ↓
                                      Claude API    Vector Store
                                                         ↓
                                                     ChromaDB
```

**Critical Pattern: Tool-Based Retrieval**
- Claude decides when to search (not every query searches)
- Maximum one search per query (enforced in ai_generator.py system prompt)
- General knowledge questions skip search entirely

### Component Responsibilities

**RAG System (`rag_system.py`)** - Central orchestrator
- Coordinates all components
- Manages query lifecycle
- Handles session tracking
- **Key**: All cross-component logic goes here

**AI Generator (`ai_generator.py`)** - Claude API integration
- Two-phase generation: initial call with tools, then final response with results
- Strict system prompt enforces response protocol
- **Important**: `_handle_tool_execution()` manages tool call flow (lines 89-135)

**Vector Store (`vector_store.py`)** - ChromaDB wrapper
- Two collections: `course_catalog` (metadata) and `course_content` (chunks)
- `search()` method (lines 61-100) handles course name resolution + vector search
- Course name resolution uses semantic matching (fuzzy match "MCP" → full title)

**Search Tools (`search_tools.py`)** - Tool abstraction
- `CourseSearchTool` provides Anthropic tool definition
- Formats results with headers: `[Course Title - Lesson N]`
- Tracks sources in `last_sources` for UI display
- **Pattern**: Tool execution returns formatted string, not structured data

**Document Processor (`document_processor.py`)** - Parsing & chunking
- Expected format: Line 1=Title, Line 2=Link, Line 3=Instructor, then lessons
- Lesson markers: `Lesson N: Title`
- Chunks add context prefix: `"Course {title} Lesson {N} content: {chunk}"`
- Overlap ensures context continuity (configurable in config.py)

### Data Flow Pattern

**Query Processing (9 steps total):**
1. Frontend: User input → POST /api/query
2. FastAPI: Route to RAG system
3. RAG System: Retrieve history + build prompt
4. AI Generator: First Claude call with tools
5. Tool Manager: Execute search if Claude requests it
6. Vector Store: Semantic search → top 5 chunks
7. AI Generator: Second Claude call with results
8. RAG System: Extract sources + update session
9. Frontend: Display response + sources

**Key Decision Point**: Step 4 - Claude's `stop_reason`
- `"tool_use"`: Proceed to search (steps 5-7)
- `"end_turn"`: Return direct answer (skip to step 8)

## Configuration System

All tunable parameters are in `backend/config.py`:
- `CHUNK_SIZE`: 1000 characters (affects search granularity)
- `CHUNK_OVERLAP`: 200 characters (context continuity)
- `MAX_RESULTS`: 5 (vector search limit)
- `MAX_HISTORY`: 10 exchanges (session memory)
- `ANTHROPIC_MODEL`: claude-3-5-sonnet-20241022

**To modify behavior:**
1. Change config values in `config.py`
2. Restart server (no rebuild needed)
3. For chunking changes: clear DB via `clear_existing=True` in app.py startup

## Important Patterns & Constraints

### Search Protocol (Critical)
- **One search per query maximum** (enforced in ai_generator.py:12)
- AI decides whether to search based on query type
- Search results are synthesized, never quoted verbatim
- No meta-commentary about search process in responses

### Session Management
- In-memory storage (resets on server restart)
- Session IDs auto-created on first query
- History format: "User: ...\nAssistant: ...\nUser: ..."
- Used for conversation context in system prompt

### Vector Search Strategy
1. Query text → embedding (sentence-transformers)
2. Cosine similarity against stored chunks
3. Optional filters: course_title, lesson_number
4. Returns top N with metadata (course, lesson, distance)

### Tool Execution Flow
```python
# AI Generator pattern (ai_generator.py:83-135)
if response.stop_reason == "tool_use":
    # Extract tool calls from response.content
    # Execute via tool_manager
    # Build new messages array with tool results
    # Second API call WITHOUT tools parameter
    # Return final synthesized response
```

### Document Format Requirements
Course documents in `docs/` must follow this structure:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [title]
Lesson Link: [url]
[lesson content...]

Lesson 1: [title]
...
```
Regex: `r'^Lesson\s+(\d+):\s*(.+)$'` (document_processor.py:166)

## Testing & Debugging

### Check Vector Store
```python
# In Python REPL from backend/
from vector_store import VectorStore
from config import config

store = VectorStore(config.CHROMA_PATH, config.EMBEDDING_MODEL, config.MAX_RESULTS)
print(store.get_course_count())
print(store.get_existing_course_titles())
```

### Query API Directly
```bash
curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What is RAG?", "session_id": null}'
```

### Debug Tool Calls
- Check `ai_generator.py` system prompt (lines 8-30)
- Tool definition in `search_tools.py:27-50`
- Tool execution logs in terminal when `stop_reason == "tool_use"`

## Common Modification Scenarios

### Adding New Tool
1. Create tool class in `search_tools.py` implementing `Tool` interface
2. Register in `rag_system.py:24`: `self.tool_manager.register_tool(new_tool)`
3. Update AI Generator system prompt if needed

### Changing AI Behavior
- Modify `ai_generator.py` SYSTEM_PROMPT (lines 8-30)
- Adjust temperature (line 39, currently 0 for deterministic)
- Change max_tokens (line 40, currently 800)

### Custom Chunking Strategy
- Override `document_processor.py:chunk_text()` method
- Maintain `CourseChunk` output format
- Update `CHUNK_SIZE`/`CHUNK_OVERLAP` in config.py

### Frontend Modifications
- `frontend/script.js`: API calls and message display
- `frontend/index.html`: UI structure
- Server serves static files from `frontend/` (app.py:119)

## Integration Points

**ChromaDB**: Persistent storage in `./chroma_db/`
- Automatically created on first run
- Delete folder to clear all data
- Uses sentence-transformers for embeddings

**Anthropic Claude API**: External dependency
- Requires `ANTHROPIC_API_KEY` in `.env`
- Rate limits apply (handled by SDK)
- Two API calls per search query (tool use pattern)

**FastAPI**: Serves both API and frontend
- CORS enabled for all origins (app.py:25-32)
- Pydantic models for validation (models.py)
- Auto-generated docs at `/docs`

## File References for Key Functionality

- **Query entry point**: `backend/app.py:56-74`
- **RAG orchestration**: `backend/rag_system.py:102-140`
- **Tool execution logic**: `backend/ai_generator.py:89-135`
- **Vector search**: `backend/vector_store.py:61-100`
- **Document parsing**: `backend/document_processor.py:97-259`
- **Frontend API calls**: `frontend/script.js:45-96`
- **System prompt**: `backend/ai_generator.py:8-30`

## Prerequisites

- Python 3.13+
- uv package manager
- Anthropic API key
- Windows users: Use Git Bash for scripts
