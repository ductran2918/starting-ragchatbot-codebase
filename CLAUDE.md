# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000

# Install dependencies
uv sync
```

### Environment Setup
Create `.env` file in root with:
```
ANTHROPIC_API_KEY=your_anthropic_api_key_here
```

## Architecture Overview

This is a **Retrieval-Augmented Generation (RAG) system** for querying course materials with AI-powered responses.

### Core Architecture Pattern
The system uses a **tool-calling RAG architecture** where Claude autonomously decides when to search vs. use existing knowledge:

1. **Query Processing Flow**: User → FastAPI → RAG System → Claude API (with tools) → Vector Search → Response
2. **Tool-Based Search**: Claude uses `CourseSearchTool` to perform semantic searches when needed
3. **Session Management**: Conversation history maintained across queries
4. **Vector Storage**: ChromaDB with sentence-transformer embeddings

### Key Components Structure

**Backend (`/backend/`)**:
- `app.py` - FastAPI application with `/api/query` and `/api/courses` endpoints
- `rag_system.py` - Main orchestrator coordinating all components
- `ai_generator.py` - Claude API integration with tool calling support
- `search_tools.py` - Tool definitions and execution for Claude function calling
- `vector_store.py` - ChromaDB vector storage with semantic search
- `document_processor.py` - Course document parsing and intelligent chunking
- `session_manager.py` - Conversation history management
- `models.py` - Pydantic data models (Course, Lesson, CourseChunk)
- `config.py` - Centralized configuration with environment variables

**Frontend (`/frontend/`)**:
- Static HTML/CSS/JS interface served by FastAPI
- Communicates via REST API calls to backend

### Document Processing Pipeline
Documents follow structured format:
```
Course Title: [title]
Course Link: [url] 
Course Instructor: [instructor]

Lesson 0: Introduction
Lesson Link: [url]
[lesson content]
```

Processing stages:
1. **Metadata Extraction** - Parse course info from header
2. **Lesson Parsing** - Detect "Lesson X:" markers and collect content
3. **Smart Chunking** - Sentence-based chunking with configurable overlap
4. **Context Enhancement** - Add course/lesson prefixes to chunks for better retrieval

### RAG System Flow
1. `RAGSystem.query()` receives user question
2. Builds prompt and retrieves conversation history
3. `AIGenerator` calls Claude with available tools
4. Claude decides whether to use `CourseSearchTool` 
5. If tool used: `VectorStore.search()` performs semantic search
6. Claude synthesizes search results into natural response
7. Sources tracked and returned to frontend

### Configuration
Key settings in `config.py`:
- `CHUNK_SIZE`: 800 chars (text chunk size)
- `CHUNK_OVERLAP`: 100 chars (context preservation)
- `MAX_RESULTS`: 5 (vector search results)
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2" (sentence-transformers)
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"

### Data Flow
Course documents → Document Processing → ChromaDB storage → Semantic search → Claude tool calling → Response generation → Frontend display

The system automatically loads course documents from `/docs/` on startup and maintains conversation context across queries while providing source attribution for responses.
- always use uv to run the serverr do not use pip directly
- make sure to use uv to manage all dependencies