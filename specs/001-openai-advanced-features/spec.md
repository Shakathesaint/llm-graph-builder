# Feature Specification: GPT-5 Model Support

**Feature Branch**: `001-openai-advanced-features`
**Created**: 2025-11-04
**Status**: Draft
**Input**: User description: "Add support for the latest OpenAI GPT-5 models, which are currently not supported in the system"

## Clarifications

### Session 2025-11-04

- Q: The spec currently focuses on "OpenAI Advanced Features" (prompt caching, service tiers, reasoning models), but your clarification mentions GPT-5 models specifically. Which is the true scope? → A: The spec should be rewritten - the primary goal is adding GPT-5 model support only, not advanced features for existing models. GPT-5 models are already available as of August 2025 (gpt-5, gpt-5-mini, gpt-5-nano, gpt-5-chat).
- Q: GPT-5 has four API variants available. Which models should be supported in this feature? → A: Support all four variants (gpt-5, gpt-5-mini, gpt-5-nano, gpt-5-chat) to give users full flexibility for cost/performance tradeoffs.
- Q: The gpt-5-chat variant is specifically designed for conversational applications. Should it be available for all operations or only chat/Q&A? → A: Make gpt-5-chat available for all operations (entity extraction, chat, Q&A) - treat it the same as other variants.
- Q: GPT-5 features "unified reasoning" that automatically adjusts thinking time. Does this require special parameter handling in the backend configuration? → A: Use standard OpenAI configuration (temperature, top_p, max_tokens, etc.) - GPT-5's reasoning is handled internally by OpenAI's API.
- Q: Where should the new GPT-5 models appear in the frontend UI? → A: Add GPT-5 models to the existing OpenAI model dropdown/selection alongside gpt-4o, o1, o3 models, but with GPT-5 models ordered at the top (listed first) to highlight them as the newest generation.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Select GPT-5 Models for Entity Extraction (Priority: P1)

As a user processing documents for entity extraction, I need to select from the latest GPT-5 model variants (gpt-5, gpt-5-mini, gpt-5-nano, gpt-5-chat) in the settings, so I can leverage the most advanced AI capabilities with flexible cost/performance tradeoffs.

**Why this priority**: This is the core functionality - users need to be able to select and use GPT-5 models. All other stories depend on this working correctly.

**Independent Test**: Can be fully tested by opening model selection dropdown in extraction settings, verifying all four GPT-5 variants appear at the top of the list, selecting each one, and processing a document successfully.

**Acceptance Scenarios**:

1. **Given** I open the model selection dropdown in extraction settings, **When** I view the list, **Then** I see all four GPT-5 variants (gpt-5, gpt-5-mini, gpt-5-nano, gpt-5-chat) listed at the top before other OpenAI models
2. **Given** I select "gpt-5" and process a 50-page PDF document, **When** extraction completes, **Then** entities are successfully extracted and stored in Neo4j with no API errors
3. **Given** I select "gpt-5-mini" for cost-optimized extraction, **When** processing begins, **Then** the system uses the correct model and pricing ($0.25/$2 per 1M tokens)
4. **Given** I select "gpt-5-nano" for maximum cost savings, **When** extraction completes, **Then** the system correctly applies nano pricing ($0.05/$0.40 per 1M tokens)

---

### User Story 2 - Use GPT-5 Models in Chat/Q&A (Priority: P2)

As a user asking questions about my knowledge graph, I need to select GPT-5 models in chat settings, so I can get answers powered by the most advanced reasoning and accuracy improvements (45% fewer factual errors vs GPT-4o).

**Why this priority**: Chat is the second major use case after extraction. Users should be able to leverage GPT-5's improved accuracy and reasoning for Q&A operations.

**Independent Test**: Can be tested by opening chat model selection, selecting each GPT-5 variant, asking a complex question, and verifying accurate responses with proper citations.

**Acceptance Scenarios**:

1. **Given** I open the chat model selection dropdown, **When** I view the list, **Then** I see all four GPT-5 variants listed at the top
2. **Given** I select "gpt-5-chat" (optimized for conversations) and ask a question, **When** the system responds, **Then** I receive an accurate answer with source citations from my knowledge graph
3. **Given** I select "gpt-5" for highest quality responses, **When** I ask a complex multi-hop reasoning question, **Then** the system leverages GPT-5's unified reasoning to provide comprehensive answers
4. **Given** I switch between GPT-5 variants during a chat session, **When** I ask new questions, **Then** each response uses the newly selected model without requiring page refresh

---

### User Story 3 - View Cost Transparency for GPT-5 Usage (Priority: P3)

As a system administrator managing AI costs, I need to understand the cost implications of each GPT-5 variant, so I can make informed decisions about which model to use for different workloads.

**Why this priority**: Cost visibility helps administrators optimize spending. However, it's lower priority than core functionality (model selection and usage).

**Independent Test**: Can be tested by processing documents with each GPT-5 variant and reviewing logs or UI cost indicators to verify correct pricing calculations.

**Acceptance Scenarios**:

1. **Given** I process a document with "gpt-5", **When** extraction completes, **Then** logs show token usage and cost calculated at $1.25/$10 per 1M input/output tokens
2. **Given** I process the same document with "gpt-5-nano", **When** extraction completes, **Then** logs show significantly lower cost ($0.05/$0.40 per 1M tokens)
3. **Given** I review system logs after using multiple GPT-5 variants, **When** I compare costs, **Then** I can clearly identify cost differences between variants (gpt-5 > gpt-5-mini > gpt-5-nano)
4. **Given** I review system logs after processing documents with GPT-5 variants, **When** I search for "GPT-5 cost" entries, **Then** I see accurate cost calculations for each operation with pricing breakdown (input cost + output cost = total cost)

#### Future Acceptance Scenarios (Post-MVP)

- **AS-5 (Future Enhancement)**: **Given** I am selecting a model in the extraction settings dropdown, **When** I hover over a GPT-5 model option, **Then** I see a tooltip displaying pricing information: "Input: $X.XX/1M tokens, Output: $Y.YY/1M tokens" to help me compare costs before selection

---

### User Story 4 - Verify Backward Compatibility with Existing Models (Priority: P4)

As a user with existing workflows using GPT-4o or other OpenAI models, I need assurance that adding GPT-5 support doesn't break my current configurations, so I can continue working while gradually migrating to GPT-5.

**Why this priority**: This is a safety/quality requirement. While critical for production stability, it's lower priority than delivering new GPT-5 functionality.

**Independent Test**: Can be tested by processing documents with existing models (gpt-4o, gpt-4o-mini, o1, o3) before and after the GPT-5 implementation, verifying identical behavior.

**Acceptance Scenarios**:

1. **Given** I have existing extraction jobs configured with "gpt-4o", **When** the GPT-5 feature is deployed, **Then** my jobs continue working without modification or errors
2. **Given** I am using "o1-preview" for reasoning-heavy extractions, **When** I process documents after GPT-5 deployment, **Then** the model initializes correctly (temperature parameter still excluded)
3. **Given** I have not explicitly selected a GPT-5 model, **When** I process documents, **Then** the system uses my previously configured model (no automatic migration)
4. **Given** I view the model selection dropdown, **When** GPT-5 models are added, **Then** all existing models remain available in their original relative order (below GPT-5 variants)

---

### Edge Cases

- **Invalid Model Selection**: What if a user has "gpt-5" selected but their OpenAI API key doesn't have access to GPT-5 models? System should return a clear error message indicating insufficient model access and suggest using available models.
- **Model Deprecation**: What if OpenAI deprecates GPT-5 model naming (e.g., renames variants)? System should gracefully handle API errors and log deprecation warnings for administrators.
- **Cost Calculation Accuracy**: What if OpenAI changes GPT-5 pricing? System should use actual costs from API responses (if available) rather than hard-coded pricing to stay accurate.
- **Model Name Collisions**: What if future OpenAI models have similar naming (e.g., "gpt-5-pro")? System should be extensible to add new variants without code changes (configuration-driven model lists).
- **Backward Compatibility**: Do existing model configurations (gpt-4o, o1, o3) continue working? Yes - GPT-5 is additive only; no changes to existing model handling.
- **UI Ordering Stability**: What if new OpenAI models are released between GPT-4 and GPT-5? GPT-5 variants should remain at the top; new models inserted in chronological order below.
- **Chat Model Persistence**: If a user selects "gpt-5-chat" and refreshes the page, does selection persist? Yes - model selection should be stored in user preferences/session storage.
- **Large Document Performance**: Do GPT-5 models handle very large documents (1000+ pages) without timeouts? GPT-5 has the same context window considerations as GPT-4; existing chunking and timeout logic applies.
- **API Rate Limits**: Are GPT-5 models subject to different rate limits than GPT-4? Likely yes - system should respect rate limit headers from API and apply existing retry logic.
- **Concurrent Model Usage**: Can multiple users use different GPT-5 variants simultaneously? Yes - each request is independent; no shared state between model selections.

## Requirements *(mandatory)*

### Functional Requirements

#### Backend Requirements

- **FR-001**: System MUST add support for four GPT-5 model variants: "gpt-5", "gpt-5-mini", "gpt-5-nano", "gpt-5-chat" in the backend LLM configuration (`backend/src/llm.py`)
- **FR-002**: System MUST initialize GPT-5 models with standard OpenAI configuration parameters (temperature, top_p, max_tokens, etc.) without special parameter exclusions
- **FR-003**: System MUST use the exact model names as provided by OpenAI API: "gpt-5", "gpt-5-mini", "gpt-5-nano", "gpt-5-chat" (no name transformation)
- **FR-004**: System MUST handle GPT-5 API responses identically to existing OpenAI models (GPT-4, o1, o3) for token usage, metadata, and error handling
- **FR-005**: System MUST log GPT-5 model selection and usage including model name, token counts (input/output), and operation type (extraction, chat, Q&A)
- **FR-006**: System MUST apply existing retry logic and error handling to GPT-5 models (rate limits, timeouts, network failures)
- **FR-007**: System MUST calculate costs for GPT-5 usage using pricing: gpt-5 ($1.25/$10), gpt-5-mini ($0.25/$2), gpt-5-nano ($0.05/$0.40) per 1M input/output tokens
- **FR-008**: System MUST support GPT-5 models for all operations: entity extraction, chat/Q&A, document processing, and graph generation
- **FR-009**: System MUST maintain backward compatibility - existing GPT-4, o1, o3 model configurations continue working without modification
- **FR-010**: System MUST preserve existing LLM provider functionality (Gemini, Diffbot, Azure, Anthropic, etc.) - changes are OpenAI-specific

#### Frontend Requirements

- **FR-011**: Frontend MUST display all four GPT-5 variants in model selection dropdowns for both extraction settings and chat settings
- **FR-012**: Frontend MUST order GPT-5 models at the TOP of OpenAI model dropdowns, before GPT-4, o1, o3, and other existing models
- **FR-013**: Frontend MUST use the following display order for GPT-5 variants: gpt-5, gpt-5-mini, gpt-5-nano, gpt-5-chat
- **FR-014**: Frontend MUST persist GPT-5 model selection in user preferences/session storage so selection survives page refreshes
- **FR-015**: Frontend MUST allow switching between GPT-5 variants and other OpenAI models without page reload or system restart
- **FR-016**: Frontend MUST show clear error messages when GPT-5 API calls fail (e.g., "Model not available for your API key", "Rate limit exceeded")
- **FR-017**: Frontend MUST update environment variable documentation to include GPT-5 models in VITE_LLM_MODELS configuration

#### Configuration Requirements

- **FR-018**: System MUST add GPT-5 models to backend configuration constants (if model lists are hard-coded in `backend/src/shared/common_fn.py` or similar)
- **FR-019**: System MUST add GPT-5 models to frontend configuration constants (`frontend/src/utils/Constants.ts` or model list definitions)
- **FR-020**: System MUST update example `.env` files to show GPT-5 models in LLM_MODEL_CONFIG examples and VITE_LLM_MODELS lists
- **FR-021**: System MUST update README or documentation to mention GPT-5 support and pricing differences between variants

### Future Enhancements (Post-MVP)

- **FE-001**: Frontend could display pricing information for each GPT-5 variant via tooltips on model dropdown items, hover states, or a dedicated model details panel to help users compare costs before selection. Pricing format: "Input: $X.XX/1M tokens, Output: $Y.YY/1M tokens". This enhancement is deferred to a future release; MVP provides cost visibility through backend logs only (per US3)

### Key Entities

- **GPT-5 Model Variant**: Represents one of the four GPT-5 model options available to users. Attributes: model_id (e.g., "gpt-5", "gpt-5-mini"), display_name, input_price_per_1m_tokens, output_price_per_1m_tokens, description. Relationships: Used in both extraction and chat operations; selected by users via frontend dropdowns.

- **LLM Configuration**: Represents the runtime configuration for an OpenAI API call using a GPT-5 model. Attributes: model_name (e.g., "gpt-5"), api_key, temperature, top_p, max_tokens, timeout, max_retries. Relationships: Generated from user's model selection and system defaults; passed to OpenAI API.

- **Token Usage Record**: Represents actual token consumption for a completed GPT-5 operation. Attributes: model_used, input_tokens, output_tokens, total_cost, operation_type (extraction/chat), timestamp. Relationships: Created after each API call; logged for cost tracking and monitoring.

- **Model Selection UI State**: Represents the user's current model choice in the frontend. Attributes: selected_model_id, context (extraction/chat), persisted_in_storage. Relationships: Stored in browser session/local storage; determines which model is used for operations.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All four GPT-5 model variants (gpt-5, gpt-5-mini, gpt-5-nano, gpt-5-chat) are selectable in both extraction and chat settings dropdowns, with 100% availability

- **SC-002**: GPT-5 models appear at the top of OpenAI model dropdowns in the exact order: gpt-5, gpt-5-mini, gpt-5-nano, gpt-5-chat (verified via UI inspection)

- **SC-003**: Users can successfully process a 50-page PDF document using each GPT-5 variant with entities correctly extracted to Neo4j, with 100% success rate (no API errors)

- **SC-004**: Users can successfully perform chat/Q&A operations using each GPT-5 variant with accurate responses and source citations, with 100% success rate

- **SC-005**: Cost calculations for GPT-5 usage are accurate in logs, showing correct pricing for each variant: gpt-5 ($1.25/$10), gpt-5-mini ($0.25/$2), gpt-5-nano ($0.05/$0.40) per 1M tokens

- **SC-006**: Model selection persists across page refreshes - if a user selects "gpt-5-mini" and refreshes, the selection remains "gpt-5-mini" (verified in 100% of test cases)

- **SC-007**: Existing GPT-4, o1, o3 model configurations continue working without modification after GPT-5 deployment, with 100% backward compatibility (zero regressions)

- **SC-008**: API error messages for GPT-5 failures are clear and actionable (e.g., "GPT-5 not available for your API key"), enabling users to self-diagnose issues

- **SC-009**: Documentation is updated to include GPT-5 support with pricing information, model descriptions, and configuration examples in README and `.env` files

- **SC-010**: Implementation is completed within 2-3 days by a single developer (low complexity - primarily configuration changes rather than algorithmic work)

### Assumptions

1. **GPT-5 API Availability**: Assumes GPT-5 models (gpt-5, gpt-5-mini, gpt-5-nano, gpt-5-chat) are generally available via OpenAI API as of August 2025 and accessible with standard OpenAI API keys (no special beta access required)

2. **API Interface Stability**: Assumes GPT-5 models use the same API interface as GPT-4 models (same request/response format, same parameters supported: temperature, top_p, max_tokens, etc.)

3. **Pricing Stability**: Assumes GPT-5 pricing remains as announced: gpt-5 ($1.25/$10), gpt-5-mini ($0.25/$2), gpt-5-nano ($0.05/$0.40) per 1M input/output tokens, or that OpenAI API responses include actual cost data

4. **Model Naming Persistence**: Assumes OpenAI will not rename or deprecate these model IDs in the near term (6-12 months), allowing stable integration without frequent updates

5. **Unified Reasoning Transparency**: Assumes GPT-5's "unified reasoning" feature works automatically without requiring client-side configuration changes (i.e., the model decides internally when to use extended reasoning)

6. **Context Window Size**: Assumes GPT-5 models have context windows comparable to or larger than GPT-4o (128K+ tokens), supporting the same document sizes without additional chunking logic

7. **Rate Limits**: Assumes GPT-5 models are subject to standard OpenAI rate limits (per-minute and per-day token limits) and that existing retry/backoff logic handles GPT-5 rate limit errors appropriately

8. **Backward Compatibility**: Assumes existing OpenAI model integrations (GPT-4, o1, o3) continue working unchanged - GPT-5 is purely additive (no removal or modification of existing models)

9. **LangChain Compatibility**: Assumes LangChain library (used by the backend) supports GPT-5 model names either explicitly or via passthrough to OpenAI API (no LangChain code updates required)

10. **Frontend Model List Configuration**: Assumes the frontend uses a configurable model list (in Constants.ts or environment variables) rather than hard-coded model names, enabling easy addition of GPT-5 without major refactoring

11. **No New Error Types**: Assumes GPT-5 does not introduce new error types beyond existing OpenAI errors (rate limit, timeout, invalid request, insufficient quota), allowing reuse of existing error handling

12. **Cost Monitoring Approach**: Assumes log-based cost monitoring is acceptable for initial release - no requirement for real-time dashboards or database-stored cost analytics
