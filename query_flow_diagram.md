# RAG System Query Processing Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                 FRONTEND                                        │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────────┐  │
│  │   User Input    │───▶│  script.js:45   │───▶│ POST /api/query             │  │
│  │ "What is MCP?"  │    │ sendMessage()   │    │ {query, session_id}         │  │
│  └─────────────────┘    └─────────────────┘    └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              FASTAPI BACKEND                                   │
│  ┌─────────────────────────────┐                                              │
│  │     app.py:56-74           │                                              │
│  │ @app.post("/api/query")    │                                              │
│  │ ┌─────────────────────────┐ │                                              │
│  │ │ 1. Create/get session   │ │                                              │
│  │ │ 2. Call rag_system      │ │───────────────────────┐                     │
│  │ │ 3. Return response      │ │                       │                     │
│  │ └─────────────────────────┘ │                       ▼                     │
│  └─────────────────────────────┘           ┌─────────────────────────────────┐ │
└─────────────────────────────────────────────│      RAG SYSTEM CORE          │─┘
                                              │   rag_system.py:102-140       │
                                              │ ┌───────────────────────────┐ │
                                              │ │ 1. Build prompt           │ │
                                              │ │ 2. Get conversation hist  │ │
                                              │ │ 3. Call AI with tools     │ │
                                              │ │ 4. Extract sources        │ │
                                              │ │ 5. Update session         │ │
                                              │ └───────────────────────────┘ │
                                              └─────────────────────────────────┘
                                                             │
                                                             ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            AI GENERATION                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                    ai_generator.py:43-135                                  │ │
│  │                                                                             │ │
│  │ ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐ │ │
│  │ │ Claude API Call │───▶│ Tool Detection  │───▶│ Need to search?         │ │ │
│  │ │ + System Prompt │    │ stop_reason:    │    │ "tool_use"              │ │ │
│  │ │ + Conversation  │    │ "tool_use"      │    └─────────────────────────┘ │ │
│  │ │ + Available     │    └─────────────────┘              │                │ │
│  │ │   Tools         │                                     ▼                │ │
│  │ └─────────────────┘    ┌─────────────────────────────────────────────────┐ │ │
│  └────────────────────────│         TOOL EXECUTION                         │─┘ │
└──────────────────────────┤ _handle_tool_execution()                        │───┘
                           │ ┌─────────────────────────────────────────────┐ │
                           │ │ Execute: search_course_content              │ │
                           │ │ Params: {query, course_name?, lesson_num?}  │ │
                           │ │ Get results, send back to Claude           │ │
                           │ └─────────────────────────────────────────────┘ │
                           └─────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           SEARCH TOOLS                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                    search_tools.py:52-114                                  │ │
│  │                                                                             │ │
│  │ ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐ │ │
│  │ │ CourseSearchTool│───▶│ vector_store    │───▶│ Format & Track Sources  │ │ │
│  │ │ .execute()      │    │ .search()       │    │ self.last_sources = [   │ │ │
│  │ │ - query         │    │ - embeddings    │    │   "Course - Lesson X"   │ │ │
│  │ │ - course_name?  │    │ - semantic sim  │    │ ]                       │ │ │
│  │ │ - lesson_num?   │    │ - filters       │    └─────────────────────────┘ │ │
│  │ └─────────────────┘    └─────────────────┘                                │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          VECTOR STORE                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                     vector_store.py                                        │ │
│  │                                                                             │ │
│  │ ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐ │ │
│  │ │ ChromaDB Query  │───▶│ SentenceTransf. │───▶│ Return SearchResults    │ │ │
│  │ │ - Course chunks │    │ Embeddings      │    │ - documents[]           │ │ │
│  │ │ - Metadata      │    │ - Query vector  │    │ - metadata[]            │ │ │
│  │ │ - Semantic sim  │    │ - Cosine sim    │    │ - distances[]           │ │ │
│  │ └─────────────────┘    └─────────────────┘    └─────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         RESPONSE PATH                                           │
│                                                                                 │
│  Claude API ◄── Tool Results ◄── Formatted Chunks ◄── Vector Search            │
│      │                                                                          │
│      ▼                                                                          │
│  Final Response ──────────────────────────────────────┐                        │
│  "MCP (Model Context Protocol)                        │                        │
│   enables AI assistants to..."                        │                        │
│                                                        │                        │
│  Sources: ["Building Towards Computer Use - Lesson 4"] │                        │
└────────────────────────────────────────────────────────┼────────────────────────┘
                                                         │
                                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            FRONTEND DISPLAY                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                       script.js:76-85                                      │ │
│  │                                                                             │ │
│  │ ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐ │ │
│  │ │ Receive JSON    │───▶│ Update Session  │───▶│ Display Message         │ │ │
│  │ │ {answer,        │    │ currentSessionId│    │ + Sources Collapsible   │ │ │
│  │ │  sources,       │    │ = data.session  │    │ + Markdown Rendering    │ │ │
│  │ │  session_id}    │    │                 │    │                         │ │ │
│  │ └─────────────────┘    └─────────────────┘    └─────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                              KEY FEATURES                                      │
│                                                                                 │
│ • Session Management: Conversation history tracking                             │
│ • Tool Calling: Claude decides when to search vs. use knowledge                │
│ • Semantic Search: Vector embeddings for context-aware retrieval               │
│ • Source Tracking: UI shows which courses/lessons provided information         │
│ • Smart Chunking: Sentence-based with overlap for context preservation         │
│ • Contextual Formatting: "Course X Lesson Y content: [chunk]"                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Flow Summary

1. **User Input** → Frontend captures query
2. **API Call** → POST to `/api/query` with session context  
3. **RAG Processing** → System orchestrates AI + tools
4. **AI Decision** → Claude determines if search is needed
5. **Tool Execution** → Search course content if required
6. **Vector Search** → ChromaDB semantic similarity matching
7. **Context Assembly** → Format chunks with course/lesson context
8. **AI Synthesis** → Claude generates response from search results
9. **Response Return** → Answer + sources sent to frontend
10. **UI Display** → Markdown rendering with collapsible sources