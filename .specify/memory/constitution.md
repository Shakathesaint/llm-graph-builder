# LLM Graph Builder Constitution

<!--
SYNC IMPACT REPORT
==================
Version: NEW → 1.0.0 (Initial constitution)
Ratification Date: 2025-11-04
Last Amended: 2025-11-04

CHANGE SUMMARY:
- Created initial constitution from template
- Established 7 core principles for LLM Graph Builder project
- Added Performance Standards and Development Workflow sections
- Defined governance rules for technical decisions

MODIFIED PRINCIPLES: N/A (initial version)
ADDED SECTIONS:
  - Core Principles (7 principles)
  - Performance Standards
  - Development Workflow
  - Governance

REMOVED SECTIONS: N/A (initial version)

TEMPLATES REQUIRING UPDATES:
  ✅ .specify/templates/plan-template.md - Constitution Check section validates principles
  ✅ .specify/templates/spec-template.md - User scenarios align with UX Consistency principle
  ✅ .specify/templates/tasks-template.md - Test-first tasks align with Testing Standards principle

FOLLOW-UP TODOS: None

RATIONALE FOR VERSION 1.0.0:
- Initial constitution establishing foundational principles
- MAJOR version reflects first formal governance structure
-->

## Core Principles

### I. Understand Before Modifying (NON-NEGOTIABLE)

**Trial and error is prohibited.** Before making any code change:

- **MUST** read and understand the complete context of files being modified
- **MUST** trace dependencies and identify all affected components (backend ↔ frontend, LLM providers, Neo4j queries)
- **MUST** verify existing functionality will not break by reviewing related code paths
- **MUST** check environment variables, configuration files, and deployment manifests for affected settings
- **MUST** consult CLAUDE.md for architectural patterns and extension points before implementing

**Rationale:** This project integrates 11+ LLM providers, complex graph transformations, and dual frontend/backend architecture. Uninformed changes cascade into subtle bugs (e.g., breaking one LLM provider while fixing another, or causing frontend/backend API contract mismatches).

**Validation:** Every pull request MUST include a "Context Analysis" section documenting:
- Files read before changes
- Dependency trace results
- Regression risk assessment

### II. Testing Standards

**Tests are mandatory for:**
- New LLM provider integrations (contract tests validating API responses)
- Graph transformation logic changes (integration tests with Neo4j)
- API endpoint modifications (contract tests for request/response schemas)
- Critical user journeys (chat Q&A, file upload → graph generation)

**Test requirements:**
- **Backend:** pytest for unit/integration tests, coverage MUST NOT decrease
- **Frontend:** Manual testing required for UX flows (file upload, chat interface, graph visualization)
- **Performance:** Load tests via Locust for extraction throughput regression detection

**When tests are optional:**
- Pure documentation updates
- Configuration file changes (MUST be validated via manual testing)
- UI styling adjustments (MUST be validated via visual review)

**Rationale:** The application processes unstructured data with unpredictable LLM outputs. Tests act as regression safeguards, especially for graph extraction logic where subtle changes can corrupt entity relationships.

### III. User Experience Consistency

**All 11+ LLM providers MUST deliver consistent UX:**
- Same extraction workflow regardless of provider (OpenAI, Gemini, Diffbot, etc.)
- Error messages MUST be user-friendly and actionable (avoid raw API errors)
- Progress tracking MUST work identically across providers
- Chat modes (vector, graph, hybrid, fulltext) MUST behave consistently

**Configuration UX rules:**
- Environment variables MUST have descriptive names and documented defaults (see example.env)
- Frontend settings MUST reflect backend capabilities (model dropdown synced with LLM_MODEL_CONFIG_*)
- Validation errors MUST surface immediately (frontend form validation + backend API validation)

**Rationale:** Users should not need to understand provider-specific quirks. Abstraction via `src/llm.py:get_llm()` ensures provider swapping doesn't break workflows.

**Validation:** Feature specs MUST include acceptance scenarios testing multiple LLM providers.

### IV. Performance and Cost Awareness

**Performance targets (MUST NOT regress):**
- Extraction: ≥ 20 chunks processed per update interval
- Chat response: < 5 seconds for vector search (p95)
- File upload: Chunked upload with progress tracking, no frontend blocking
- Graph queries: < 2 seconds for visualization (1000 nodes, typical document)

**Cost optimization requirements:**
- LLM features with cost impact (prompt caching, service tiers) MUST be configurable via environment variables
- Token usage MUST be logged and visible (total tokens, cached tokens, cost savings)
- Batch processing MUST be preferred for large document sets (UPDATE_GRAPH_CHUNKS_PROCESSED tuning)

**Monitoring obligations:**
- Log structured metrics for: token usage, cache hit rates, extraction throughput, API errors
- Performance tests MUST run before releasing changes to extraction or chat logic

**Rationale:** LLM costs scale with token usage. Prompt caching (50% savings) and service tier selection (priority vs flex) directly impact operating costs. Users need visibility to optimize.

### V. Backward Compatibility

**Breaking changes are prohibited without migration path:**
- Environment variable renames MUST support legacy names for ≥ 1 release
- API endpoint changes MUST maintain old schemas via versioning or adapters
- Neo4j graph model changes MUST include migration Cypher scripts
- Frontend changes MUST NOT break existing bookmarked URLs or deep links

**Configuration changes:**
- New optional parameters MUST have sensible defaults (preserve existing behavior)
- Removing parameters MUST trigger deprecation warnings for ≥ 1 release before removal

**Rationale:** Users deploy this application in production environments. Breaking changes force costly redeployment and configuration audits.

**Validation:** Pull requests MUST document:
- Compatibility impact assessment
- Migration instructions (if breaking change justified)
- Deprecation timeline for removed features

### VI. Code Quality and Maintainability

**Code organization:**
- Backend: Modular structure via `src/` submodules (llm, sources, graphDB_dataAccess, QA_integration, post_processing)
- Frontend: Component-based React with clear context boundaries (UserCredentials, Files, Messages, Alerts)
- Shared logic MUST be extracted into utilities (`shared/common_fn.py`, `frontend/src/utils/`)

**Code standards:**
- **Python:** Type hints for public functions, Pydantic models for API contracts, logging via CustomLogger
- **TypeScript:** Strict mode enforced, ESLint compliance, Prettier formatting
- **Documentation:** Docstrings for non-trivial functions, inline comments for complex logic (graph queries, LLM transformations)

**Anti-patterns to avoid:**
- Hardcoded configuration (use environment variables)
- Direct Neo4j queries in UI code (use backend API)
- LLM provider-specific logic outside `src/llm.py`
- Copy-paste code across components (extract shared utilities)

**Rationale:** This codebase supports 11+ LLM providers and 6+ document sources. Maintainability requires strict boundaries and abstraction layers.

### VII. Observability and Debugging

**Logging requirements:**
- **Backend:** Structured logs with context (user_id, file_name, model_name, operation)
- **Frontend:** Console errors for debugging, toast notifications for user-facing errors (AlertContext)
- **LLM calls:** Log prompts (sanitized), responses (truncated), token usage, latency, cache hits

**Error handling:**
- **Backend:** Custom `LLMGraphBuilderException` for domain errors, structured API responses via `create_api_response()`
- **Frontend:** User-friendly error messages (avoid technical jargon), retry mechanisms for transient failures
- **LLM failures:** Graceful degradation (e.g., fall back to simpler model, skip entity extraction for problematic chunks)

**Debugging tools:**
- Enable LangChain tracing via `LANGCHAIN_TRACING_V2=true` for LLM debugging
- SSE endpoint (`/sse_chat`) for real-time progress monitoring
- Neo4j Bloom integration for graph visualization debugging

**Rationale:** LLM-based applications have non-deterministic behavior. Observability is critical for diagnosing extraction failures, cost anomalies, and performance bottlenecks.

## Performance Standards

**Extraction throughput:**
- Target: 1000 pages/hour (typical PDF documents, gpt-4o-mini)
- Bottleneck monitoring: Token limits, API rate limits, Neo4j write throughput
- Optimization levers: Chunk size (MAX_TOKEN_CHUNK_SIZE), batch size (UPDATE_GRAPH_CHUNKS_PROCESSED), service tier (priority/flex)

**Chat response latency:**
- Vector search: < 5s (p95)
- Graph traversal: < 10s (p95, depends on graph size)
- Hybrid modes: < 15s (p95, combines multiple retrievals)

**Resource limits:**
- Backend memory: < 2GB per worker (for Cloud Run deployment)
- Frontend bundle size: < 5MB (gzipped)
- Neo4j queries: < 100k nodes traversed per query (prevent graph scan timeouts)

**Scalability requirements:**
- Support concurrent file uploads (10+ users)
- Handle large documents (1000+ pages) without timeout
- Manage large graphs (100k+ nodes) with acceptable query performance

**Performance regression detection:**
- Run `Performance_test.py` before releases
- Compare extraction throughput and chat latency against baseline
- Flag regressions > 20% for investigation

## Development Workflow

**Feature development process:**
1. **Specification:** Create feature spec in `specs/[###-feature-name]/spec.md` using `/speckit.specify`
2. **Planning:** Generate implementation plan via `/speckit.plan` (includes constitution check)
3. **Task breakdown:** Generate tasks via `/speckit.tasks` with clear dependencies
4. **Implementation:** Follow tasks sequentially, commit after each logical unit
5. **Testing:** Write tests (if applicable), validate manually, run performance tests
6. **Review:** Pull request with Context Analysis, compatibility assessment, test results
7. **Deployment:** Update environment configs, run migration scripts (if needed), monitor logs

**Constitution compliance gates:**
- **Planning stage:** Verify feature aligns with principles (constitution check in plan.md)
- **Implementation stage:** Follow "Understand Before Modifying" before every code change
- **Review stage:** Reviewer validates Context Analysis and compatibility assessment
- **Deployment stage:** Monitor observability metrics for regressions

**Branch strategy:**
- Feature branches: `[###-feature-name]` (number from issue tracker)
- Main branch: Protected, requires pull request approval
- Deployment branches: Separate for dev/staging/production (if applicable)

**Code review checklist:**
- [ ] Context Analysis provided (files read, dependencies traced)
- [ ] Tests added/updated (if applicable) and passing
- [ ] Backward compatibility maintained or migration path documented
- [ ] Environment variables documented in example.env
- [ ] Frontend/backend API contracts aligned
- [ ] Performance impact assessed (extraction throughput, chat latency)
- [ ] Observability: Logs added for new operations, errors handled gracefully
- [ ] UX consistency: Tested across multiple LLM providers (if applicable)

## Governance

**Constitution authority:**
- This constitution supersedes informal practices and undocumented conventions
- When principles conflict with expedience, principles win (escalate to project lead if blocking)
- Violating non-negotiable principles (I, II for critical paths) requires explicit exception approval

**Amendment process:**
1. Propose amendment via pull request to `.specify/memory/constitution.md`
2. Document rationale: Why existing principle insufficient? What problem does amendment solve?
3. Update version per semantic versioning (MAJOR for breaking changes, MINOR for additions, PATCH for clarifications)
4. Update dependent templates (plan-template.md, spec-template.md, tasks-template.md)
5. Require project lead approval for MAJOR/MINOR versions
6. Announce amendment to team with migration guidance (if applicable)

**Versioning policy:**
- **MAJOR (X.0.0):** Removing principles, redefining non-negotiable rules, incompatible governance changes
- **MINOR (x.Y.0):** Adding principles, expanding sections, new quality gates
- **PATCH (x.y.Z):** Clarifications, typo fixes, examples, non-semantic improvements

**Compliance review:**
- Pull requests MUST reference applicable principles in description
- Reviewers MUST validate constitution compliance (use review checklist)
- Quarterly constitution review: Are principles still relevant? Are they being followed?

**Complexity justification:**
- If violating principles (e.g., adding 4th architectural layer, skipping tests for time-sensitive fix), MUST document in "Complexity Tracking" section of plan.md:
  - Which principle violated
  - Why necessary (business justification)
  - What simpler alternatives were considered and rejected
  - Timeline for remediation (if technical debt)

**Runtime guidance:**
- Use `CLAUDE.md` for detailed architectural patterns, extension points, debugging tips
- Use constitution for high-level principles and decision-making governance
- If conflict between CLAUDE.md and constitution, constitution wins (update CLAUDE.md)

---

**Version:** 1.0.0 | **Ratified:** 2025-11-04 | **Last Amended:** 2025-11-04
