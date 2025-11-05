# Tasks: GPT-5 Model Support

**Input**: Design documents from `/specs/001-openai-advanced-features/`
**Prerequisites**: plan.md (complete), spec.md (complete), research.md (complete), data-model.md (complete), contracts/ (complete)

**Tests**: Per spec.md test plan and Constitution Principle II, tests ARE required for this feature:
- Backend integration test: Verify GPT-5 model initialization
- Frontend manual test: Verify model dropdown ordering and selection
- End-to-end test: Document processing with each GPT-5 variant

**Organization**: Tasks grouped by user story (P1-P4) to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: User story label (US1, US2, US3, US4)
- All tasks include exact file paths

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Environment and configuration setup for GPT-5 models

- [ ] T001 Verify OpenAI API key has GPT-5 access (test API call to gpt-5-mini endpoint)
- [ ] T002 [P] Create backup of backend/.env and frontend/.env files before modifications
- [ ] T003 [P] Review existing model configurations in backend/src/llm.py lines 51-70 to understand OpenAI handling pattern, AND verify backend/src/shared/common_fn.py for any hard-coded model lists (e.g., MODEL_VERSIONS, OPENAI_MODELS constants) - if found, document that GPT-5 models need to be added to those lists; if not found, confirm configuration is purely env-variable-driven per FR-018

**Checkpoint**: Development environment ready for GPT-5 configuration

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: None required - GPT-5 is purely additive to existing infrastructure

**⚠️ CRITICAL**: No foundational tasks needed. Existing LangChain ChatOpenAI integration, error handling, and retry logic handle GPT-5 automatically via passthrough. User stories can begin immediately after Phase 1.

**Checkpoint**: Foundation ready (existing system) - user story implementation can begin in parallel

---

## Phase 3: User Story 1 - Select GPT-5 Models for Entity Extraction (Priority: P1) 🎯 MVP

**Goal**: Enable users to select and use all four GPT-5 variants (gpt-5, gpt-5-mini, gpt-5-nano, gpt-5-chat) for document entity extraction via model dropdown in extraction settings.

**Independent Test**: Open extraction settings model dropdown → verify 4 GPT-5 variants at top → select gpt-5-mini → upload 5-page PDF → click "Generate Graph" → verify successful extraction with no errors → check Neo4j for extracted entities.

### Backend Configuration for User Story 1

- [ ] T004 [P] [US1] Add GPT-5 environment variable documentation to backend/.env.example with pricing comments and example values (format: LLM_MODEL_CONFIG_OPENAI_GPT_5=gpt-5,use_openai_api_key)
- [ ] T005 [P] [US1] Add GPT-5-mini environment variable documentation to backend/.env.example
- [ ] T006 [P] [US1] Add GPT-5-nano environment variable documentation to backend/.env.example
- [ ] T007 [P] [US1] Add GPT-5-chat environment variable documentation to backend/.env.example
- [ ] T008 [US1] Configure actual backend/.env file with 4 GPT-5 model variables using OPENAI_API_KEY (copy examples from .env.example and set actual API key)

### Frontend Configuration for User Story 1

- [ ] T009 [US1] Update frontend/src/utils/Constants.ts to prepend 4 GPT-5 models to llms array at lines 14-41 (order: openai_gpt_5, openai_gpt_5_mini, openai_gpt_5_nano, openai_gpt_5_chat before existing models)
- [ ] T010 [P] [US1] Update frontend/.env.example to include GPT-5 models in VITE_LLM_MODELS example (comma-separated list with GPT-5 first)

### Testing for User Story 1

- [ ] T011 [US1] Restart backend (uvicorn score:app --reload) and verify logs show "Model: LLM_MODEL_CONFIG_OPENAI_GPT_5" entries for all 4 variants
- [ ] T012 [US1] Rebuild frontend (yarn build) and verify no TypeScript compilation errors
- [ ] T013 [US1] Manual test: Open extraction settings → verify GPT-5, GPT-5 Mini, GPT-5 Nano, GPT-5 Chat appear at top of model dropdown in that order
- [ ] T014 [US1] Manual test: Select gpt-5-mini → upload test document (5-10 pages) → click Extract/Generate Graph → verify extraction completes successfully
- [ ] T015 [US1] Manual test: Check backend logs for token usage and model name "gpt-5-mini" confirmation
- [ ] T016 [US1] Manual test: Verify extracted entities visible in Neo4j graph (use Neo4j Bloom or graph query)

**Checkpoint**: User Story 1 complete - users can select and extract documents with GPT-5 models

---

## Phase 4: User Story 2 - Use GPT-5 Models in Chat/Q&A (Priority: P2)

**Goal**: Enable users to select GPT-5 variants in chat settings and ask questions about knowledge graph using GPT-5's improved reasoning.

**Independent Test**: Open chat settings model dropdown → verify 4 GPT-5 variants at top → select gpt-5-chat → ask question "What are the main entities?" → verify response with source citations → switch to gpt-5 → ask another question → verify model change applies without page refresh.

### Implementation for User Story 2

- [ ] T017 [US2] Verify chat model dropdown reads from same Constants.ts llms array (already updated in T009) - no code changes needed
- [ ] T018 [US2] Manual test: Open chat tab → verify GPT-5, GPT-5 Mini, GPT-5 Nano, GPT-5 Chat appear at top of chat model dropdown
- [ ] T019 [US2] Manual test: Select gpt-5-chat → ask test question "What are the main entities in the document?" → verify response with source citations
- [ ] T020 [US2] Manual test: Select gpt-5 (flagship model) → ask complex question requiring multi-hop reasoning → verify comprehensive answer
- [ ] T021 [US2] Manual test: Switch from gpt-5-chat to gpt-5-mini during same session → ask new question → verify response uses new model (check backend logs for model name)
- [ ] T022 [US2] Manual test: Refresh page → verify selected model (gpt-5-chat or gpt-5) persists in dropdown (localStorage check)

**Checkpoint**: User Stories 1 AND 2 both work independently - users can use GPT-5 for extraction and chat

---

## Phase 5: User Story 3 - View Cost Transparency for GPT-5 Usage (Priority: P3)

**Goal**: Provide visibility into token usage and cost for each GPT-5 variant to help administrators optimize spending.

**Independent Test**: Process document with gpt-5 → review backend logs → verify cost calculation showing $1.25/$10 pricing → process same document with gpt-5-nano → verify logs show $0.05/$0.40 pricing → compare costs → confirm nano is significantly cheaper.

### Implementation for User Story 3

- [ ] T023 [P] [US3] Add GPT-5 pricing constants dictionary to backend/src/shared/common_fn.py or create backend/src/pricing.py with GPT5_PRICING dict (gpt-5: 1.25/10, gpt-5-mini: 0.25/2, gpt-5-nano: 0.05/0.40, gpt-5-chat: 1.25/10)
- [ ] T024 [US3] Add calculate_gpt5_cost() function in backend (takes model_name, input_tokens, output_tokens, returns USD cost using pricing dict)
- [ ] T025 [US3] Update backend logging in extraction completion handler to call calculate_gpt5_cost() and log "GPT-5 cost: $X.XX (input: $Y.YY, output: $Z.ZZ)" when GPT-5 model detected
- [ ] T026 [US3] Update backend logging in chat/Q&A completion handler to log GPT-5 cost breakdown for chat operations

### Testing for User Story 3

- [ ] T027 [US3] Manual test: Process 10-page PDF with gpt-5 → check logs for "GPT-5 cost: $X.XX" with $1.25/$10 pricing calculation
- [ ] T028 [US3] Manual test: Process same document with gpt-5-nano → verify logs show significantly lower cost (~96% cheaper)
- [ ] T029 [US3] Manual test: Review logs after processing multiple documents with different GPT-5 variants → verify cost differences clear (gpt-5 > gpt-5-mini > gpt-5-nano)
- [ ] T030 [US3] Manual test: Perform chat Q&A with gpt-5-chat → verify logs show token usage and cost for chat operation

**Checkpoint**: All cost logging implemented - administrators can track GPT-5 spending via logs

---

## Phase 6: User Story 4 - Verify Backward Compatibility with Existing Models (Priority: P4)

**Goal**: Ensure GPT-4, o1, o3, and other existing OpenAI models continue working unchanged after GPT-5 additions.

**Independent Test**: Select gpt-4o in extraction settings → process document → verify successful extraction identical to pre-GPT-5 behavior → select o1-preview → verify temperature parameter excluded (existing reasoning model logic) → verify all existing models appear below GPT-5 in dropdown.

### Testing for User Story 4

- [ ] T031 [US4] Manual test: Select gpt-4o in extraction settings → process test document → verify extraction succeeds with no errors
- [ ] T032 [US4] Manual test: Select gpt-4o-mini in chat settings → ask question → verify chat response works correctly
- [ ] T033 [US4] Manual test: Select o1-preview (reasoning model) → verify backend initializes without temperature parameter (check logs for absence of temperature=0)
- [ ] T034 [US4] Manual test: Verify dropdown ordering: GPT-5 variants at top, then gpt-4o, gpt-4o-mini, o1, o3 models below in original relative order
- [ ] T035 [US4] Manual test: Process document with gpt-4o before and after GPT-5 feature deployment → compare results → verify identical entity extraction behavior
- [ ] T036 [US4] Manual test: Verify existing environment variables (LLM_MODEL_CONFIG_OPENAI_GPT_4O, etc.) still work without modification

**Checkpoint**: All existing models working correctly - 100% backward compatibility confirmed

---

## Phase 7: Backend Integration Test (Cross-Story Validation)

**Purpose**: Automated integration test covering all GPT-5 variants

- [ ] T037 [P] Create backend/tests/test_gpt5_models.py integration test file with mock OpenAI API responses for GPT-5 variants
- [ ] T038 Test case: Call get_llm("openai_gpt_5") and verify ChatOpenAI object returned with model="gpt-5" and temperature=0
- [ ] T039 Test case: Call get_llm("openai_gpt_5_mini") and verify correct model name
- [ ] T040 Test case: Call get_llm("openai_gpt_5_nano") and verify correct model name
- [ ] T041 Test case: Call get_llm("openai_gpt_5_chat") and verify correct model name
- [ ] T042 Test case: Mock OpenAI API error (403 model not available) → verify LLMGraphBuilderException raised with helpful error message
- [ ] T043 Run pytest backend/tests/test_gpt5_models.py and verify all tests pass

**Checkpoint**: Automated backend tests passing - GPT-5 integration validated

---

## Phase 8: Documentation & Polish

**Purpose**: Update documentation and perform final validation

- [ ] T044 [P] Update README.md to mention GPT-5 support in LLM providers section (add bullet: "GPT-5 (4 variants: gpt-5, gpt-5-mini, gpt-5-nano, gpt-5-chat)")
- [ ] T045 [P] Update README.md to include GPT-5 pricing table in configuration section
- [ ] T046 [P] Update CLAUDE.md "Adding a New LLM Provider" section to include GPT-5 as example (already partially done by agent script)
- [ ] T047 [P] Add GPT-5 example to backend/.env.example with detailed comments explaining each variant's use case
- [ ] T048 Review quickstart.md from specs/001-openai-advanced-features/quickstart.md and verify all setup steps are accurate
- [ ] T049 Run complete manual test suite following quickstart.md: setup → extraction → chat → cost verification → backward compatibility
- [ ] T050 [P] Optional: Create Docker Compose example showing GPT-5 environment variable configuration in docker-compose.yml

**Checkpoint**: Documentation complete and validated - feature ready for deployment

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies - start immediately ✅
- **Phase 2 (Foundational)**: None needed - existing infrastructure handles GPT-5 ✅
- **Phase 3 (US1 - Extraction)**: Depends on Phase 1 → Can start immediately after Setup
- **Phase 4 (US2 - Chat)**: Depends on Phase 3 (uses same Constants.ts) → Can start after US1 T009 complete
- **Phase 5 (US3 - Cost Logging)**: Independent of US1/US2 → Can start after Phase 1
- **Phase 6 (US4 - Backward Compat)**: Depends on Phase 3 → Can start after US1 config complete
- **Phase 7 (Integration Tests)**: Depends on Phase 3 backend config → Can start after US1 T008 complete
- **Phase 8 (Documentation)**: Depends on all user stories being validated → Final phase

### User Story Dependencies

- **US1 (Extraction - P1)**: No dependencies - Can start after Phase 1 ✅
- **US2 (Chat - P2)**: Depends on US1 T009 (Constants.ts update) - Minimal dependency
- **US3 (Cost Logging - P3)**: Independent - Can run in parallel with US1/US2
- **US4 (Backward Compat - P4)**: Depends on US1 config - Starts after US1 T008/T009

### Critical Path

1. Phase 1: Setup (T001-T003) → ~15 min
2. Phase 3: US1 Backend Config (T004-T008) → ~20 min ⚠️ BLOCKS US2, US4
3. Phase 3: US1 Frontend Config (T009-T010) → ~10 min ⚠️ BLOCKS US2
4. Phase 3: US1 Testing (T011-T016) → ~1 hour
5. Phase 4: US2 Testing (T017-T022) → ~45 min
6. Phase 5: US3 Implementation (T023-T026) → ~1 hour
7. Phase 5: US3 Testing (T027-T030) → ~30 min
8. Phase 6: US4 Testing (T031-T036) → ~45 min
9. Phase 7: Integration Tests (T037-T043) → ~1.5 hours
10. Phase 8: Documentation (T044-T050) → ~1 hour

**Total Sequential Time**: ~7-8 effective work hours (2-3 calendar days accounting for PR review, testing, context switching per SC-010)

### Parallel Opportunities

**Phase 1** (all can run in parallel):
- T002 (backup files) + T003 (review code)

**Phase 3 (US1 Backend Config)** - all run in parallel:
- T004 (gpt-5 .env.example)
- T005 (gpt-5-mini .env.example)
- T006 (gpt-5-nano .env.example)
- T007 (gpt-5-chat .env.example)
- T010 (frontend .env.example)

**Phase 5 (US3 Cost Logging)** - parallel tasks:
- T023 (pricing constants)
- T024 (cost calculation function)

**Phase 7 (Integration Tests)** - T037 creates file, then tests run in parallel:
- T038, T039, T040, T041 (model initialization tests)

**Phase 8 (Documentation)** - all can run in parallel:
- T044 (README providers)
- T045 (README pricing)
- T046 (CLAUDE.md)
- T047 (.env.example comments)
- T050 (Docker example)

**Cross-Story Parallelization**:
- After US1 T009 completes: US2 AND US3 can both start in parallel
- US4 can start after US1 T008/T009 complete
- Phase 7 (tests) can overlap with Phase 5/6 (US3/US4)

---

## Parallel Example: User Story 1 Backend Config

```bash
# Launch all .env.example documentation tasks together:
Task T004: "Add GPT-5 env var docs to backend/.env.example"
Task T005: "Add GPT-5-mini env var docs to backend/.env.example"
Task T006: "Add GPT-5-nano env var docs to backend/.env.example"
Task T007: "Add GPT-5-chat env var docs to backend/.env.example"
Task T010: "Update frontend/.env.example with GPT-5 models"

# These can all be done simultaneously (different sections of files)
```

---

## Parallel Example: User Story 3 Cost Logging

```bash
# Launch pricing setup tasks together:
Task T023: "Create GPT5_PRICING constants dict"
Task T024: "Create calculate_gpt5_cost() function"

# Then after both complete:
Task T025: "Update extraction logging with cost calculation"
Task T026: "Update chat logging with cost calculation"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only) - ~2 hours

1. **Phase 1**: Setup (T001-T003) → 15 min
2. **Phase 3**: User Story 1 complete (T004-T016) → 1.5 hours
3. **STOP and VALIDATE**: Test extraction with gpt-5-mini
4. **Deploy** to development environment for early feedback

**MVP Delivers**: Users can select and use all 4 GPT-5 models for document extraction

### Incremental Delivery - ~7 hours total

1. **Foundation** (Phase 1) → 15 min
2. **US1: Extraction** (Phase 3) → 1.5 hours → Test independently → Deploy ✅
3. **US2: Chat** (Phase 4) → 45 min → Test independently → Deploy ✅
4. **US3: Cost Logging** (Phase 5) → 1.5 hours → Test independently → Deploy ✅
5. **US4: Backward Compat** (Phase 6) → 45 min → Validate
6. **Integration Tests** (Phase 7) → 1.5 hours
7. **Documentation** (Phase 8) → 1 hour → Final deploy ✅

Each increment adds value without breaking previous features.

### Parallel Team Strategy (2 developers)

**Developer A**:
1. Phase 1: Setup
2. Phase 3: US1 (Extraction) - config and testing
3. Phase 5: US3 (Cost Logging) - implementation
4. Phase 7: Integration Tests

**Developer B** (starts after US1 T009):
1. Phase 4: US2 (Chat) - testing
2. Phase 6: US4 (Backward Compatibility) - testing
3. Phase 8: Documentation

**Timeline**: ~4 hours with 2 developers working in parallel

---

## Task Summary

| Phase | User Story | Task Count | Estimated Time |
|-------|-----------|-----------|---------------|
| Phase 1 | Setup | 3 tasks | 15 min |
| Phase 2 | Foundational | 0 tasks | 0 min (existing infra) |
| Phase 3 | US1 (Extraction - P1) | 13 tasks | 1.5 hours |
| Phase 4 | US2 (Chat - P2) | 6 tasks | 45 min |
| Phase 5 | US3 (Cost Logging - P3) | 8 tasks | 1.5 hours |
| Phase 6 | US4 (Backward Compat - P4) | 6 tasks | 45 min |
| Phase 7 | Integration Tests | 7 tasks | 1.5 hours |
| Phase 8 | Documentation & Polish | 7 tasks | 1 hour |
| **TOTAL** | **4 User Stories** | **50 tasks** | **~7-8 hours** |

### Parallel Opportunities Identified

- **12 tasks marked [P]**: Can run in parallel (24% of tasks)
- **User Stories**: US3 can run parallel to US1/US2 after Phase 1
- **Team Parallelization**: 2 developers can complete in ~4 hours

### Independent Test Criteria

- ✅ **US1**: Process document with gpt-5-mini → verify entities in Neo4j
- ✅ **US2**: Ask chat question with gpt-5-chat → verify response with citations
- ✅ **US3**: Review logs → verify cost calculations for all variants
- ✅ **US4**: Process document with gpt-4o → verify unchanged behavior

### MVP Scope (Recommended)

**MVP = User Story 1 only** (13 tasks, ~1.5 hours)
- Delivers: All 4 GPT-5 models selectable for extraction
- Validates: Configuration approach, model initialization, dropdown ordering
- Enables: Early user feedback on GPT-5 quality vs GPT-4

---

## Format Validation

✅ **All tasks follow required checklist format**:
- Checkbox: `- [ ]` ✅
- Task ID: T001-T050 sequential ✅
- [P] marker: 12 parallel tasks marked ✅
- [Story] label: US1, US2, US3, US4 on appropriate tasks ✅
- File paths: All implementation tasks include exact paths ✅

✅ **Organization by user story**: Each story has dedicated phase ✅

✅ **Independent testing**: Each story has validation criteria ✅

---

## Notes

- **[P] tasks**: Different files or sections, no dependencies - safe to parallelize
- **[Story] labels**: Track which user story each task belongs to for traceability
- **File paths**: Exact paths provided for all code/config tasks
- **Tests**: Integration test (Phase 7) covers all variants, manual tests per story
- **Commits**: Recommended after each user story phase completion
- **Checkpoints**: Stop after each phase to validate story independently
- **MVP flexibility**: Can deploy after US1 only, or incrementally add US2/US3/US4

**Ready to implement**: Tasks are specific enough for LLM or developer to execute without additional context. Each task clearly states what file to modify and what change to make.
