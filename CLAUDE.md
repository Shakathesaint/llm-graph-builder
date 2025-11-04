# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Knowledge Graph Builder application that transforms unstructured data (PDFs, DOCs, TXT, YouTube videos, web pages) into structured Knowledge Graphs stored in Neo4j using Large Language Models (LLMs) and the LangChain framework.

**Technology Stack:**
- Backend: Python 3.x, FastAPI, LangChain, Neo4j Python Driver
- Frontend: React 18, TypeScript, Vite, Neo4j Design Language (NDL)
- Database: Neo4j 5.23+ with APOC
- LLMs: Supports 11+ providers (OpenAI, Gemini, Diffbot, Azure, Anthropic, etc.)

---

## Development Commands

### Frontend (React + Vite)
```bash
cd frontend
yarn                    # Install dependencies
yarn run dev            # Start dev server (http://localhost:5173)
yarn run build          # Build for production (TypeScript compile + Vite build)
yarn run lint           # Run ESLint
yarn run format         # Format code with Prettier
yarn run lint-staged    # Run linting on staged files (pre-commit hook)
```

### Backend (FastAPI + Python)
```bash
cd backend
python -m venv envName              # Create virtual environment
source envName/bin/activate         # Activate venv (Linux/Mac)
envName\Scripts\activate            # Activate venv (Windows)
pip install -r requirements.txt     # Install dependencies
uvicorn score:app --reload          # Start dev server (http://localhost:8000)
```

### Docker Compose (Full Stack)
```bash
docker-compose up --build           # Build and run both services
# Frontend: http://localhost:8080
# Backend: http://localhost:8000
```

### Testing
```bash
# Backend tests (from backend/)
pytest test_integrationqa.py       # Integration tests for Q&A
pytest test_commutiesqa.py         # Community detection tests
python Performance_test.py         # Performance benchmarks
locust -f locustperf.py            # Load testing with Locust
```

---

## Architecture

### Backend Architecture (`backend/`)

**Entry Point:** `score.py` - Main FastAPI application with 30+ REST endpoints

**Core Processing Pipeline:**
1. **Document Upload** → `score.py:/url/scan` creates Document node in Neo4j
2. **Extraction** → `main.py:extract_graph_from_file_local_file()` orchestrates:
   - Document loading via source-specific loaders (`src/sources/`)
   - Text splitting with `TokenTextSplitter` (configurable chunk size/overlap)
   - Batch processing with progress tracking
   - LLM-based entity extraction via `LLMGraphTransformer`
   - Graph document creation in Neo4j
3. **Post-Processing** → Optional indexing, community detection, deduplication

**Key Modules:**
- `src/main.py` - Extraction orchestration, async processing, progress tracking
- `src/llm.py` - Unified LLM interface for 11+ providers (OpenAI, Gemini, Anthropic, etc.)
- `src/graphDB_dataAccess.py` - Neo4j CRUD operations, Cypher query execution
- `src/QA_integration.py` - Chat/Q&A with 7 retrieval modes (vector, graph, hybrid, etc.)
- `src/sources/` - Modular loaders: `local_file.py`, `s3_loader.py`, `gcs_loader.py`, `youtube_loader.py`, `web_loader.py`, `wiki_loader.py`
- `src/post_processing.py` - Vector indexes, full-text indexes, entity embeddings
- `src/communities.py` - Community detection algorithm
- `src/graph_query.py` - Graph visualization queries
- `src/shared/common_fn.py` - Utility functions, environment config

**Configuration:** All settings via environment variables (see README ENV section)

### Frontend Architecture (`frontend/`)

**Entry Point:** `src/index.tsx` → `App.tsx` → `QuickStarter.tsx`

**State Management:** React Context API (no Redux)
- `UserCredentialsContext` - Neo4j connection credentials
- `FileContext` - File upload state, processing status
- `MessageContext` - Chat messages and history
- `AlertContext` - Toast notifications

**Component Hierarchy:**
```
QuickStarter (main layout)
├── Header (branding, connection status)
├── PageLayout
│   ├── LeftSideNav (file upload, source selection)
│   ├── Content (main work area)
│   │   ├── DrawerDropzone (file upload with chunking)
│   │   ├── FilesTable (file grid with actions)
│   │   ├── GraphViewModal (Neo4j Bloom integration)
│   │   └── ChatBot (Q&A interface)
│   └── RightSideNav (chat sidebar, settings)
└── ConnectionModal (Neo4j credentials)
```

**File Upload Flow:**
1. User drops files → `DrawerDropzone` validates and chunks files
2. Chunks uploaded via `axios` with retry logic → `/upload` endpoint
3. Backend saves to storage (local, GCS, or S3) → `/url/scan` creates Document nodes
4. UI polls for status updates → FileContext state synchronized

**Build Output:** `dist/` folder with static assets served by backend or cloud hosting

### Neo4j Graph Model

**Node Labels:**
- `Document` - Represents uploaded files (properties: fileName, fileSize, fileType, url, status)
- `Chunk` - Text chunks from documents (properties: text, index, position, embedding)
- `__Entity__` - Dynamic label for extracted entities (merged into specific types like Person, Organization, etc.)
- `__Community__` - Detected communities from graph structure

**Relationships:**
- `PART_OF` - Chunk → Document
- `NEXT_CHUNK` - Chunk → Chunk (sequential ordering)
- `SIMILAR` - Chunk → Chunk (semantic similarity via embeddings)
- `HAS_ENTITY` - Chunk → Entity (entity mentions)
- `IN_COMMUNITY` - Entity → Community

**Indexes:**
- Vector index on Chunk.embedding (for semantic search)
- Full-text index on Chunk.text (for keyword search)
- Hybrid retrieval combines both

---

## Data Flow: Upload → Graph

1. **Upload Phase**
   - Frontend chunks file → POST `/upload` (multipart/form-data)
   - Backend saves to storage, returns metadata
   - POST `/url/scan` creates Document node with status="New"

2. **Extraction Phase** (triggered by user via "Generate Graph" button)
   - POST `/extract` with list of file names
   - For each file:
     - Load document via source loader (`src/sources/`)
     - Split into chunks using `TokenTextSplitter` (default 5242880 bytes)
     - Create embeddings for each chunk (model: `all-MiniLM-L6-v2` or OpenAI)
     - Process chunks in batches (default: 20 chunks per update)
     - For each batch:
       - Extract entities and relationships via LLM (`LLMGraphTransformer`)
       - Apply schema constraints if configured
       - Merge nodes/relationships into Neo4j
       - Create HAS_ENTITY relationships from chunks to entities
     - Update Document status: "New" → "Processing" → "Completed" or "Failed"

3. **Post-Processing** (optional, triggered separately)
   - Create vector indexes on embeddings
   - Create full-text indexes on chunk text
   - Run community detection algorithm
   - Deduplicate similar entities (configurable distance threshold)

4. **Visualization**
   - User clicks "View" or "Preview Graph" → GET `/graph_query`
   - Backend runs Cypher query to fetch graph for selected documents
   - Returns nodes and relationships as JSON
   - Frontend renders in Neo4j Bloom or custom renderer

5. **Chat/Q&A**
   - User asks question → POST `/chat_bot` with mode (vector, graph, hybrid, etc.)
   - Backend retrieves context using selected mode:
     - `vector`: Semantic search via embeddings
     - `graph`: Traversal-based retrieval
     - `graph+vector`: Combines both
     - `fulltext`: Keyword search
     - `entity_vector`: Entity-focused retrieval
   - LLM generates answer with source citations
   - Response includes chunk metadata and document references

---

## Configuration and Environment

All configuration via environment variables (`.env` files). Key settings:

**Backend Critical Variables:**
- `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD` - Neo4j connection (optional, can use UI login)
- `OPENAI_API_KEY`, `DIFFBOT_API_KEY` - Required for respective LLMs
- `GEMINI_ENABLED=True` - Enable Gemini (requires GCP setup)
- `GCS_FILE_CACHE=True` - Store files in GCS instead of local filesystem
- `EMBEDDING_MODEL=all-MiniLM-L6-v2` - Default embedding model
- `MAX_TOKEN_CHUNK_SIZE=10000` - Max tokens per chunk

**Frontend Critical Variables:**
- `VITE_BACKEND_API_URL=http://localhost:8000` - Backend endpoint
- `VITE_LLM_MODELS` - Comma-separated list of enabled models (dev)
- `VITE_LLM_MODELS_PROD` - Models for production environment
- `VITE_REACT_APP_SOURCES=local,youtube,wiki,s3,gcs,web` - Enabled sources
- `VITE_CHAT_MODES=vector,graph+vector,graph,fulltext` - Enabled chat modes
- `VITE_ENV=DEV` or `PROD` - Environment toggle

**Docker Compose:** Combines both `.env` files, uses `host.docker.internal` for cross-container communication

---

## Important Patterns and Conventions

### LLM Provider Integration
All LLM providers are abstracted in `src/llm.py` with unified interface:
```python
llm = get_llm(model_name)  # Returns LangChain LLM instance
```
Configuration via `LLM_MODEL_CONFIG_<provider>_<model>` env variables (format: `model_id,api_base_url`)

### Async Processing with Cancellation
Long-running extraction tasks support cancellation:
- Backend stores task state in `processing_filenames` dict
- Frontend sends DELETE `/cancel/processing_file` to stop extraction
- Backend checks cancellation flag in batch processing loop

### Progress Tracking
Extraction progress reported via:
1. Periodic database updates (every `UPDATE_GRAPH_CHUNKS_PROCESSED` chunks)
2. SSE (Server-Sent Events) for real-time updates via `/sse_chat` endpoint

### Error Handling
- Custom exception: `LLMGraphBuilderException` for domain errors
- All API endpoints return structured responses via `create_api_response()`
- Frontend uses AlertContext for user-facing error messages

### Schema-Driven Extraction
Users can define custom schemas in UI (Settings → Entity Graph Extraction):
- Specify allowed node labels (e.g., Person, Organization, Product)
- Specify allowed relationship types (e.g., WORKS_FOR, BOUGHT)
- Backend passes schema to `LLMGraphTransformer` to constrain extraction

---

## Testing and Quality

### Backend Tests
- **Integration Tests:** `test_integrationqa.py` validates Q&A retrieval modes
- **Community Tests:** `test_commutiesqa.py` validates graph algorithms
- **Performance Tests:** `Performance_test.py` measures extraction throughput
- **Load Tests:** `locustperf.py` simulates concurrent users (requires Locust)

### Frontend Code Quality
- **ESLint** configured with TypeScript rules (`.eslintrc.cjs`)
- **Prettier** for code formatting
- **Husky** pre-commit hooks run lint-staged (`.husky/pre-commit`)
- **TypeScript** strict mode enforced

### No Jest/PyTest Config
Tests run directly with `pytest` (no `pytest.ini`). Frontend has no unit tests (integration testing via manual QA).

---

## Common Extension Points

### Adding a New LLM Provider
1. Add provider logic in `src/llm.py:get_llm()` function
2. Add env variable: `LLM_MODEL_CONFIG_<provider>_<model>`
3. Update `VITE_LLM_MODELS` in frontend `.env`
4. Add model option in `frontend/src/utils/Constants.ts`

### Adding a New Document Source
1. Create loader in `backend/src/sources/<source>_loader.py`
2. Implement `load_file()` function returning LangChain Documents
3. Add source handling in `src/main.py:extract_graph_from_file_local_file()`
4. Add source to `VITE_REACT_APP_SOURCES` in frontend `.env`
5. Add UI component in `frontend/src/components/DataSources/`

### Adding a New Chat Mode
1. Implement retrieval logic in `src/QA_integration.py`
2. Add mode to `VITE_CHAT_MODES` env variable
3. Update frontend `ChatModeToggle` component to show new mode

### Customizing Chunking Strategy
Modify `TokenTextSplitter` configuration in `src/main.py:extract_graph_from_file_local_file()`:
- `chunk_size`: Tokens per chunk (default: 2000)
- `chunk_overlap`: Overlap between chunks (default: 200)

---

## Deployment Notes

### Local Development
Run backend and frontend separately (see Development Commands). Requires:
- Python 3.x with pip
- Node.js 18+ with yarn
- Neo4j 5.23+ instance (Aura or local)

### Docker Compose
Single-command deployment for testing:
```bash
docker-compose up --build
```
Access at http://localhost:8080 (frontend proxies to backend:8000)

### Google Cloud Run
Deploy backend and frontend as separate services:
- Backend: Uses Dockerfile in `backend/` with uvicorn
- Frontend: Uses Dockerfile in `frontend/` with nginx
- Configure env variables via `gcloud run deploy --set-env-vars`

### Neo4j Requirements
- Version 5.23 or later (required for vector indexes)
- APOC plugin installed (for graph algorithms)
- Aura DB or Aura DS supported (community detection requires DS)

---

## Debugging Tips

### Backend Issues
- Check logs: Backend uses `CustomLogger` with structured logging
- Enable LangChain tracing: Set `LANGCHAIN_TRACING_V2=true`, `LANGCHAIN_API_KEY`
- Test Neo4j connection: Use `/health` endpoint or check `score.py` startup logs
- LLM errors: Check API keys and rate limits (logged in `src/llm.py`)

### Frontend Issues
- Check browser console for API errors
- Inspect Network tab for failed requests
- Verify `VITE_BACKEND_API_URL` matches backend location
- Check React DevTools for context state issues

### Graph Extraction Failures
- Check `Document.status` in Neo4j (should be "Completed", not "Failed")
- Review `Document.errorMessage` property for error details
- Verify LLM model supports required context window (large files need large windows)
- Check chunk size settings (too large chunks may timeout)

### Performance Issues
- Enable entity embeddings: `ENTITY_EMBEDDING=True` (slower but better retrieval)
- Adjust batch size: `UPDATE_GRAPH_CHUNKS_PROCESSED` (default: 20)
- Use Diffbot for faster extraction (pre-trained, no LLM calls for entity extraction)
- Monitor Neo4j memory and CPU (large graphs need tuning)

---

## Architecture Decisions

### Why Context API Instead of Redux?
Simpler state management for this scale. Only 4 contexts (Credentials, Files, Messages, Alerts) with minimal cross-component dependencies.

### Why LangChain?
Provides unified interface for 11+ LLM providers and built-in tools for:
- Document loading (50+ source types)
- Text splitting strategies
- Graph extraction (`LLMGraphTransformer`)
- Vector store integration
- Prompt templates

### Why FastAPI?
- Native async support for long-running extraction tasks
- Automatic OpenAPI docs generation (`/docs`)
- Type hints with Pydantic models
- SSE support for real-time progress updates

### Why Neo4j?
- Native graph database with Cypher query language
- Vector index support (semantic search)
- Full-text index support (keyword search)
- Graph algorithms (community detection, similarity)
- Scalable (Aura cloud offering)

### Why Vite Instead of CRA?
- Faster dev server (ES modules, no bundling in dev)
- Faster builds (Rollup-based)
- Better TypeScript support
- Modern tooling (PostCSS, Tailwind)
