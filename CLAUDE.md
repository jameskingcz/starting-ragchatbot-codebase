# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Retrieval-Augmented Generation (RAG) chatbot** that answers questions about course materials using:
- **ChromaDB** for vector storage and semantic search
- **Anthropic Claude API** with tool-use capability (claude-sonnet-4-20250514)
- **FastAPI** backend serving both API and static frontend
- **Sentence Transformers** (all-MiniLM-L6-v2) for embeddings

## Setup & Running

### Initial Setup
```bash
# Install dependencies
uv sync

# Create .env file with API key
echo "ANTHROPIC_API_KEY=your-key-here" > .env
```

### Running the Application
```bash
# Quick start
./run.sh

# Manual start (from root directory)
cd backend && uv run uvicorn app:app --reload --port 8000
```

Access at: `http://localhost:8000` (frontend) or `http://localhost:8000/docs` (API docs)

### Development Commands
```bash
# Install new dependency
uv add package-name

# Sync dependencies after changes
uv sync

# Run from backend directory
cd backend && uv run uvicorn app:app --reload --port 8000
```

## Architecture: Tool-Based RAG Flow

This system uses a **two-step AI process** where Claude decides autonomously when to search:

### Query Processing Flow

1. **Frontend** (frontend/script.js) → POST /api/query with {query, session_id}
2. **API Endpoint** (backend/app.py) → Creates/retrieves session
3. **RAG System** (backend/rag_system.py) → Orchestrates the entire flow
4. **AI Generator** (backend/ai_generator.py) → **First Claude API call** with tool definitions
5. **Claude Decision Point**:
   - If course-specific question → Uses `search_course_content` tool
   - If general question → Answers directly from knowledge
6. **Tool Execution** (backend/search_tools.py) → Searches ChromaDB via VectorStore
7. **Vector Store** (backend/vector_store.py) → Semantic search in ChromaDB
8. **AI Generator** → **Second Claude API call** with search results to synthesize answer
9. **Response** → Returns (answer, sources) back through the chain

### Key Architecture Decisions

**Two ChromaDB Collections:**
- `course_catalog` - Course metadata (title, instructor, lessons)
- `course_content` - Actual lesson content chunks with embeddings

**Tool-Based vs Traditional RAG:**
- Traditional RAG: Always searches first, then generates
- This system: Claude decides if/when to search based on query type
- Tool definition in `search_tools.py:CourseSearchTool.get_tool_definition()`
- Tool execution managed by `ToolManager` class

**Session Management:**
- Sessions created in `SessionManager` (backend/session_manager.py)
- History limited to `MAX_HISTORY` message pairs (default: 2)
- History formatted and included in system prompt for context

**Document Processing Pipeline:**
- Documents must follow format: Line 1 = title, Line 2 = link, Line 3 = instructor
- Lesson markers: "Lesson N: Title" where N is the lesson number
- Chunking: 800 chars with 100 char overlap (configurable in backend/config.py)
- Supports: PDF, DOCX, TXT files

## Configuration (backend/config.py)

All tunable parameters are in `Config` dataclass:
- `ANTHROPIC_MODEL` - Claude model to use
- `EMBEDDING_MODEL` - Sentence transformer model
- `CHUNK_SIZE` / `CHUNK_OVERLAP` - Document chunking parameters
- `MAX_RESULTS` - Number of search results to return
- `MAX_HISTORY` - Conversation pairs to maintain
- `CHROMA_PATH` - Vector database location (./chroma_db)

## Important Implementation Details

### Adding New Courses
- Place documents in `docs/` folder - auto-loaded on startup (app.py:88-98)
- Or use `rag_system.add_course_document(file_path)` or `add_course_folder(folder_path)`
- Duplicate detection: Checks existing course titles before adding

### AI System Prompt
- Located in `ai_generator.py:AIGenerator.SYSTEM_PROMPT`
- Critical instruction: "One search per query maximum"
- Instructs Claude to avoid meta-commentary and provide direct answers
- Temperature set to 0 for deterministic responses

### Source Tracking
- `CourseSearchTool` stores sources in `last_sources` during search
- Retrieved via `tool_manager.get_last_sources()` after query
- Sources formatted as: "Course Title - Lesson N"
- Displayed in collapsible UI section in frontend

### Frontend-Backend Contract
**POST /api/query**
```json
Request: {"query": "string", "session_id": "string|null"}
Response: {"answer": "string", "sources": ["string"], "session_id": "string"}
```

**GET /api/courses**
```json
Response: {"total_courses": int, "course_titles": ["string"]}
```

## File Organization

**Backend Core:**
- `app.py` - FastAPI endpoints, CORS, static file serving
- `rag_system.py` - Main orchestrator connecting all components
- `ai_generator.py` - Claude API integration with tool execution
- `vector_store.py` - ChromaDB wrapper for both collections
- `search_tools.py` - Tool definitions and ToolManager
- `document_processor.py` - Document parsing and chunking
- `session_manager.py` - Conversation history management
- `models.py` - Data classes (Course, Lesson, CourseChunk)

**Frontend:**
- `index.html` - UI structure
- `script.js` - API calls, message handling, Markdown rendering
- `style.css` - Dark theme styling

## When Modifying the System

**To add a new tool:**
1. Create class inheriting from `Tool` in search_tools.py
2. Implement `get_tool_definition()` and `execute()` methods
3. Register with `tool_manager.register_tool(your_tool)` in rag_system.py

**To change search behavior:**
- Modify `vector_store.py:VectorStore.search()` method
- Adjust `MAX_RESULTS` in config.py
- Update tool prompt in `search_tools.py:CourseSearchTool.get_tool_definition()`

**To adjust AI behavior:**
- Edit `AIGenerator.SYSTEM_PROMPT` in ai_generator.py
- Change `temperature` (currently 0) in ai_generator.py base_params
- Modify `max_tokens` (currently 800) for longer/shorter responses

**To change chunking strategy:**
- Update `CHUNK_SIZE` and `CHUNK_OVERLAP` in config.py
- Modify `document_processor.py:DocumentProcessor._chunk_text()` for custom logic
