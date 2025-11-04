# Implementation Plan: GPT-5 Model Support

**Branch**: `001-openai-advanced-features` | **Date**: 2025-11-04 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-openai-advanced-features/spec.md`

## Summary

Add support for four GPT-5 model variants (gpt-5, gpt-5-mini, gpt-5-nano, gpt-5-chat) to the LLM Graph Builder application. GPT-5 models use standard OpenAI API interface with same parameters as GPT-4 (temperature, top_p, max_tokens), require no special parameter handling, and will be positioned at the top of model selection dropdowns in both extraction and chat settings. Implementation is primarily configuration-driven (adding model entries to frontend Constants.ts and backend environment variable documentation), with minimal code changes required. Backend leverages existing LangChain OpenAI integration; frontend adds four new dropdown options.

## Technical Context

**Language/Version**: Python 3.11+ (backend), TypeScript 5.x (frontend)
**Primary Dependencies**:
- Backend: FastAPI 0.115.12, LangChain 0.3.25, langchain-openai 0.3.23, OpenAI 1.86.0
- Frontend: React 18.3.1, Vite 4.x, Neo4j NDL 3.2.x, Axios 1.8.4

**Storage**: Neo4j 5.23+ (graph database), browser localStorage/sessionStorage (UI preferences)
**Testing**:
- Backend: pytest (existing tests in `backend/test_*.py`)
- Frontend: Manual testing for UI flows (model selection, extraction, chat)
- Integration: End-to-end document processing with each GPT-5 variant

**Target Platform**:
- Backend: Linux/Docker containers (Cloud Run compatible)
- Frontend: Modern browsers (Chrome, Firefox, Safari, Edge)

**Project Type**: Web application (React frontend + FastAPI backend)

**Performance Goals**:
- No regression in extraction throughput (target: ≥20 chunks/update, 1000 pages/hour)
- No regression in chat latency (vector search <5s p95)
- Model selection dropdown renders in <100ms

**Constraints**:
- Backward compatibility: 100% - existing GPT-4/o1/o3 configurations must continue working
- Zero downtime deployment: New models can be added via environment variable updates without code deployment
- API rate limits: GPT-5 subject to standard OpenAI rate limits (handled by existing retry logic)

**Scale/Scope**:
- 4 new model variants to add
- 2 locations to update (backend `llm.py`, frontend `Constants.ts`)
- 2 environment files to document (`.env.example` for backend/frontend)
- 1 README update (mention GPT-5 support and pricing)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### I. Understand Before Modifying
- [x] **Context Analysis**:
  - **Backend files to modify**:
    - `backend/src/llm.py` (OpenAI model initialization - lines 51-70 already handle openai models)
    - `backend/.env` example documentation (add GPT-5 model configs)
  - **Frontend files to modify**:
    - `frontend/src/utils/Constants.ts` (llms array - lines 11-41, add 4 GPT-5 entries at top)
    - `frontend/.env` example documentation (update VITE_LLM_MODELS)
  - **Dependencies traced**:
    - Backend: `llm.py:get_llm()` → LangChain ChatOpenAI → OpenAI API (no changes needed, passthrough)
    - Frontend: `Constants.ts:llms` → model dropdown components → user selection → backend API calls
    - No Neo4j query changes required (model selection doesn't affect graph structure)
  - **Configuration files**: `.env` examples, `CLAUDE.md` (add GPT-5 to architecture doc)

- [x] **Regression Risk**:
  - **Low risk identified**: GPT-5 is purely additive (no modifications to existing model handling)
  - Existing user flows unaffected:
    - GPT-4/o1/o3 users: No change in behavior (same initialization logic)
    - Extraction workflow: Identical (uses same LLMGraphTransformer)
    - Chat Q&A: Identical (uses same QA_integration logic)
  - Potential risks mitigated:
    - Dropdown ordering: GPT-5 models prepended to array (existing models maintain relative order)
    - Model name parsing: Reuses existing "openai" prefix detection (line 51 in llm.py)

- [x] **Configuration Impact**:
  - **Environment variables affected**:
    - Backend: Need to add 4 new vars: `LLM_MODEL_CONFIG_OPENAI_GPT_5`, `LLM_MODEL_CONFIG_OPENAI_GPT_5_MINI`, `LLM_MODEL_CONFIG_OPENAI_GPT_5_NANO`, `LLM_MODEL_CONFIG_OPENAI_GPT_5_CHAT`
    - Frontend: Update `VITE_LLM_MODELS` example to include: `openai_gpt_5,openai_gpt_5_mini,openai_gpt_5_nano,openai_gpt_5_chat`
  - **Config files**: `backend/.env.example`, `frontend/.env.example`
  - **Deployment manifests**: No changes (environment variables set at runtime)

### II. Testing Standards
- [x] **Test Coverage**:
  - **Tests required**:
    - Backend integration test: Process sample document with each GPT-5 variant, verify entities extracted to Neo4j
    - Frontend manual test: Verify all 4 variants appear in dropdowns (extraction + chat), verify ordering (GPT-5 at top)
    - End-to-end test: Select gpt-5, extract document, ask chat question - verify full workflow
  - **Tests NOT required**: No new unit tests (reusing existing ChatOpenAI logic from LangChain)

- [x] **Test Plan**:
  1. **Backend**: Add test case to `test_integrationqa.py` or create `test_gpt5_models.py`:
     - Mock OpenAI API responses for GPT-5 variants
     - Call `get_llm("openai_gpt_5")` and verify ChatOpenAI initialization
     - Verify temperature parameter included (FR-002)
  2. **Frontend**: Manual testing checklist:
     - Open extraction settings → verify GPT-5 variants at top
     - Select each variant → process document → verify success
     - Open chat settings → verify GPT-5 variants at top
     - Switch models during chat → verify model change applies
  3. **Integration**: Process 10-page PDF with each GPT-5 variant, compare entity extraction quality/cost

- [x] **Coverage Target**: Maintain current coverage (no decrease). New code is minimal (configuration additions only).

### III. User Experience Consistency
- [x] **Multi-Provider Support**:
  - N/A for this feature - GPT-5 is OpenAI-specific
  - Other providers (Gemini, Diffbot, Anthropic) unaffected
  - Existing multi-provider UX consistency preserved (same extraction/chat workflows)

- [x] **Error Handling**:
  - Error messages inherit from existing OpenAI error handling in `llm.py`:
    - API key missing: "OPENAI_API_KEY environment variable is not set" (line 57)
    - Rate limit: Standard OpenAI error propagated to user
    - Model access denied: OpenAI API returns 403, shown as "Model not available for your API key" (FR-017)
  - No new error types introduced by GPT-5

- [x] **Progress Tracking**:
  - Identical to existing OpenAI models:
    - Extraction: Chunk-based progress updates via SSE (`/sse_chat` endpoint)
    - Chat: Standard response streaming (if enabled)
  - No GPT-5-specific progress logic needed

- [x] **Configuration UX**:
  - New environment variables documented in `.env.example` files with:
    - Descriptive comments: "# GPT-5 models (August 2025 release) - 4 variants for cost/performance tradeoffs"
    - Pricing information: Input/output costs per 1M tokens
    - Example values: `LLM_MODEL_CONFIG_OPENAI_GPT_5=gpt-5,use_openai_api_key`
  - Frontend dropdown labels use readable names: "GPT-5", "GPT-5 Mini", "GPT-5 Nano", "GPT-5 Chat"

### IV. Performance and Cost Awareness
- [x] **Performance Targets**:
  - **No performance regression expected**:
    - GPT-5 uses same API interface as GPT-4 (same latency profile)
    - Extraction throughput: Same chunk processing logic (target: ≥20 chunks/update maintained)
    - Chat latency: Same retrieval modes (vector/graph/hybrid) - target: <5s p95 maintained
  - **Potential improvements** (GPT-5 advantage):
    - Unified reasoning may reduce need for retry logic on complex extractions
    - Improved accuracy may reduce post-processing corrections

- [x] **Cost Impact**:
  - **Cost visibility implemented**:
    - Pricing documented in spec (FR-007): gpt-5 ($1.25/$10), gpt-5-mini ($0.25/$2), gpt-5-nano ($0.05/$0.40)
    - Token usage logged via existing logging in `llm.py` (captures input/output tokens from OpenAI response)
    - Costs calculated in logs using pricing formula: `(input_tokens * input_price + output_tokens * output_price) / 1_000_000`
  - **Configurability**: Users choose model via dropdown (cost tradeoff visible in pricing documentation)

- [x] **Regression Detection**:
  - Existing performance tests will catch regressions:
    - `Performance_test.py`: Measures extraction throughput
    - `locustperf.py`: Load testing for concurrent users
  - Run tests with GPT-5 models before release, compare baseline (GPT-4o benchmarks)

### V. Backward Compatibility
- [x] **Breaking Changes**: **NONE**
  - GPT-5 is purely additive (new models added to existing list)
  - No changes to:
    - Existing environment variables (GPT-4, o1, o3 configs untouched)
    - API endpoints (no new parameters, no removed parameters)
    - Neo4j graph model (entity extraction logic identical)
    - Frontend URLs/routes (no routing changes)

- [x] **Migration Path**:
  - N/A - no migration needed
  - Users can adopt GPT-5 incrementally (keep using existing models, try GPT-5 when ready)

- [x] **Default Behavior**:
  - New users: Existing default model maintained (typically `openai_gpt_4o` or `openai_gpt_4o_mini`)
  - Existing users: Model selection preserved (no automatic switch to GPT-5)
  - Optional parameters: GPT-5 uses standard defaults (temperature=0, max_tokens=None, timeout=None)

### VI. Code Quality and Maintainability
- [x] **Modular Design**:
  - Backend: Leverages existing `llm.py:get_llm()` function (lines 21-70) - no new abstraction needed
  - Frontend: Uses existing Constants.ts pattern (model list configuration)
  - Follows CLAUDE.md architecture: "All LLM providers abstracted via `src/llm.py:get_llm()`"

- [x] **Code Standards**:
  - **Python**: Existing code already has type hints, uses Pydantic models, CustomLogger
  - **TypeScript**: Constants.ts follows existing patterns, ESLint compliant
  - **Documentation**: Add docstring comment to GPT-5 section in Constants.ts: "// GPT-5 models (2025) - unified reasoning, 4 variants"

- [x] **Anti-Patterns Avoided**:
  - No hardcoded configuration (uses environment variables, configurable model lists)
  - No direct API calls in frontend (goes through backend /extract and /chat_bot endpoints)
  - No provider-specific logic outside `llm.py` (GPT-5 handled in existing OpenAI branch)

### VII. Observability and Debugging
- [x] **Logging**:
  - **Backend structured logs** (existing logging in `llm.py` line 32):
    - `logging.info("Model: {}".format(env_key))` - logs selected model (e.g., "LLM_MODEL_CONFIG_OPENAI_GPT_5")
  - **Additional logging** (already present in LangChain):
    - Token usage: Captured from OpenAI API response metadata
    - Latency: Logged via FastAPI middleware
  - **Context fields**: user_id (from session), file_name (from request), model_name (from env_key)

- [x] **Error Handling**:
  - Custom exceptions: Reuses `LLMGraphBuilderException` (line 17 import) for API key errors
  - Frontend messages: Inherits from existing OpenAI error handling (shown via AlertContext toasts)
  - Specific errors logged:
    - Model config missing: "Environment variable 'LLM_MODEL_CONFIG_OPENAI_GPT_5' is not defined" (line 28-30)
    - API failures: OpenAI SDK exceptions propagated with context

- [x] **Monitoring**:
  - Debugging tools available:
    - LangChain tracing: `LANGCHAIN_TRACING_V2=true` captures GPT-5 prompts/responses
    - SSE endpoint: Real-time extraction progress visible at `/sse_chat`
    - Neo4j Bloom: Graph visualization for extracted entities
  - Metrics to monitor:
    - GPT-5 usage rate (% of requests using GPT-5 vs GPT-4)
    - Cost per extraction (compare GPT-5 variants)
    - Error rates (track API failures by model)

**Constitution Compliance Summary:**

**✅ PASS - All principles satisfied:**

- **Principle I**: Context fully analyzed (llm.py lines 51-70, Constants.ts lines 11-41), dependencies traced, no regression risks
- **Principle II**: Test plan defined (integration + manual), coverage maintained
- **Principle III**: UX consistency preserved (same workflows, clear error messages, configuration documented)
- **Principle IV**: Performance targets documented, cost visibility via logs, regression detection via existing tests
- **Principle V**: Zero breaking changes, backward compatibility 100%, no migration needed
- **Principle VI**: Follows modular design (llm.py abstraction), code standards maintained, anti-patterns avoided
- **Principle VII**: Observability via existing logging, error handling reused, debugging tools available

**No constitution violations or exceptions required.** This is a clean, additive feature following all established patterns.

## Project Structure

### Documentation (this feature)

```text
specs/001-openai-advanced-features/
├── spec.md              # Feature specification (already complete)
├── plan.md              # This file - implementation plan
├── research.md          # Phase 0: LangChain compatibility, OpenAI API details
├── data-model.md        # Phase 1: Model variant entity definitions
├── quickstart.md        # Phase 1: Quick setup guide for GPT-5
├── contracts/           # Phase 1: No new API contracts (reuses existing)
│   └── README.md        # Documents that existing /extract and /chat_bot APIs handle GPT-5
└── tasks.md             # Phase 2: NOT created by /speckit.plan (created by /speckit.tasks)
```

### Source Code (repository root)

```text
# Web application structure (existing)
backend/
├── src/
│   ├── llm.py                    # ⚠️ MODIFY: Add GPT-5 to OpenAI branch (lines 51-70)
│   ├── main.py                   # No changes (uses llm.py abstraction)
│   ├── QA_integration.py         # No changes (model-agnostic chat logic)
│   ├── graphDB_dataAccess.py     # No changes (graph operations unchanged)
│   └── shared/
│       └── common_fn.py          # No changes (utility functions unchanged)
├── .env.example                  # ⚠️ UPDATE: Add GPT-5 env var examples with pricing
└── tests/
    ├── test_integrationqa.py     # ✅ ADD: GPT-5 integration test cases
    └── test_gpt5_models.py       # ✅ NEW: GPT-5 specific unit tests (optional)

frontend/
├── src/
│   ├── utils/
│   │   └── Constants.ts          # ⚠️ MODIFY: Add 4 GPT-5 models to llms array (prepend to lines 14-41)
│   ├── components/               # No changes (model dropdown reads from Constants.ts)
│   ├── context/                  # No changes (FileContext, MessageContext unchanged)
│   └── services/                 # No changes (API calls model-agnostic)
├── .env.example                  # ⚠️ UPDATE: Add GPT-5 to VITE_LLM_MODELS example
└── tests/                        # Manual testing (no automated frontend tests exist)

.specify/
└── memory/
    └── constitution.md           # No changes (feature complies with all principles)

README.md                         # ⚠️ UPDATE: Mention GPT-5 support in LLM providers section
CLAUDE.md                         # ⚠️ UPDATE: Add GPT-5 to LLM provider integration examples
```

**Structure Decision**: This is a web application (React frontend + FastAPI backend). GPT-5 implementation touches 2 core files (`llm.py`, `Constants.ts`) plus documentation updates. No new architectural layers needed - leverages existing LLM abstraction pattern documented in CLAUDE.md ("All LLM providers abstracted via `src/llm.py:get_llm()`").

## Complexity Tracking

**No violations to report** - This implementation complies with all constitution principles without exceptions.

| Violation | Why Needed | Simpler Alternative Rejected Because | Remediation Timeline |
|-----------|------------|-------------------------------------|---------------------|
| N/A | N/A | N/A | N/A |

---

**Implementation proceeds to Phase 0 (Research) - No blockers identified.**

**Next steps after plan approval**:
1. Execute Phase 0 research tasks (generate research.md)
2. Execute Phase 1 design tasks (generate data-model.md, contracts/, quickstart.md)
3. Run agent context update script
4. Re-evaluate constitution check
5. Run `/speckit.tasks` to generate actionable task breakdown
