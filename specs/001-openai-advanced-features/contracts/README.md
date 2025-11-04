# API Contracts: GPT-5 Model Support

**Feature**: GPT-5 Model Support
**Date**: 2025-11-04
**Status**: No new contracts - reuses existing endpoints

---

## Summary

**No new API endpoints or contract changes required.** GPT-5 models leverage existing `/extract` and `/chat_bot` endpoints with identical request/response schemas. This document confirms compatibility and documents expected behavior.

---

## Endpoint 1: POST /extract

**Purpose**: Extract entities and relationships from uploaded documents using selected LLM.

### Request Schema (unchanged)

```json
{
  "uri_list": ["s3://bucket/document.pdf", "local://uploads/file.txt"],
  "model": "openai_gpt_5",          // ✅ NEW: Can be any GPT-5 variant
  "chunksize": 10000,
  "chunkoverlap": 200,
  "allowedNodes": ["Person", "Organization"],  // Optional
  "allowedRelationships": ["WORKS_FOR"],        // Optional
  "language": "en"
}
```

**GPT-5 Model Values**:
- `"openai_gpt_5"` → API calls `gpt-5`
- `"openai_gpt_5_mini"` → API calls `gpt-5-mini`
- `"openai_gpt_5_nano"` → API calls `gpt-5-nano`
- `"openai_gpt_5_chat"` → API calls `gpt-5-chat`

### Response Schema (unchanged)

```json
{
  "status": "completed",
  "message": "Extraction successful",
  "files_processed": 1,
  "entities_extracted": 150,
  "relationships_created": 200,
  "processing_time_seconds": 45.2,
  "chunks_processed": 25,
  "model_used": "gpt-5-mini"          // ✅ Reflects actual API model name
}
```

### Validation

- **Model name**: Backend `llm.py` validates model exists via environment variable lookup
- **Error on missing config**: Returns 400 if `LLM_MODEL_CONFIG_OPENAI_GPT_5` not set
- **Error on API failure**: Returns 500 if OpenAI API returns error (with user-friendly message)

### Backward Compatibility

✅ **100% compatible** - Existing GPT-4/o1/o3 requests continue working unchanged.

**Example**: Switching from GPT-4o to GPT-5
```json
// Before (still works)
{"model": "openai_gpt_4o", ...}

// After (new option)
{"model": "openai_gpt_5", ...}
```

---

## Endpoint 2: POST /chat_bot

**Purpose**: Ask questions about knowledge graph using selected LLM.

### Request Schema (unchanged)

```json
{
  "question": "What are the main entities in the contract?",
  "model": "openai_gpt_5_chat",     // ✅ NEW: GPT-5 variant for chat
  "mode": "graph+vector",
  "session_id": "abc123",
  "document_names": ["contract.pdf"],  // Optional: filter by documents
  "max_tokens": 1000                   // Optional: limit response length
}
```

**Chat Modes** (all supported by GPT-5):
- `"vector"`: Semantic search via embeddings
- `"graph"`: Graph traversal retrieval
- `"graph+vector"`: Hybrid approach
- `"fulltext"`: Keyword search
- `"entity_vector"`: Entity-focused retrieval

### Response Schema (unchanged)

```json
{
  "answer": "The main entities are John Doe (Person), Acme Corp (Organization)...",
  "sources": [
    {
      "chunk_id": "chunk_123",
      "text": "John Doe works for Acme Corp...",
      "document": "contract.pdf",
      "score": 0.95
    }
  ],
  "model_used": "gpt-5-chat",
  "tokens_used": {
    "input": 500,
    "output": 150,
    "total": 650
  },
  "processing_time_seconds": 2.3
}
```

### Token Usage Logging

Backend logs token usage with cost calculation:
```python
logging.info(f"GPT-5 chat completed: {tokens_used['total']} tokens, "
             f"cost: ${calculate_cost('gpt-5-chat', tokens_used['input'], tokens_used['output'])}")
```

### Backward Compatibility

✅ **100% compatible** - Existing chat sessions using GPT-4 continue without interruption.

---

## Endpoint 3: GET /models (if exists)

**Status**: Verify if application exposes model listing endpoint.

**Expected Behavior**: If `/models` endpoint exists, it should dynamically read from:
- Backend: Environment variables matching `LLM_MODEL_CONFIG_*`
- Frontend: `Constants.ts` `llms` array

**Expected Response** (after GPT-5 addition):
```json
{
  "available_models": [
    "openai_gpt_5",
    "openai_gpt_5_mini",
    "openai_gpt_5_nano",
    "openai_gpt_5_chat",
    "openai_gpt_4o",
    "openai_gpt_4o_mini",
    "gemini_2.0_flash",
    "diffbot"
  ],
  "default_model": "openai_gpt_4o"
}
```

**Action**: If endpoint doesn't exist, no changes needed. If it exists, verify it reads from configuration (not hard-coded).

---

## Error Responses

All existing error codes apply to GPT-5:

### 400 Bad Request - Missing Model Config
```json
{
  "error": "Environment variable 'LLM_MODEL_CONFIG_OPENAI_GPT_5' is not defined as per format or missing",
  "model_requested": "openai_gpt_5",
  "suggestion": "Add LLM_MODEL_CONFIG_OPENAI_GPT_5=gpt-5,use_openai_api_key to .env file"
}
```

### 403 Forbidden - Model Access Denied
```json
{
  "error": "OpenAI API error: Model 'gpt-5' not available for your API key",
  "model_requested": "gpt-5",
  "suggestion": "Verify your OpenAI account has access to GPT-5 models"
}
```

### 429 Too Many Requests - Rate Limit
```json
{
  "error": "Rate limit exceeded for model 'gpt-5-mini'",
  "retry_after_seconds": 20,
  "suggestion": "Reduce request frequency or upgrade OpenAI tier"
}
```

### 500 Internal Server Error - API Failure
```json
{
  "error": "OpenAI API call failed: Connection timeout",
  "model_used": "gpt-5",
  "suggestion": "Check OpenAI API status and retry"
}
```

---

## Contract Compatibility Matrix

| Endpoint | GPT-4 Behavior | GPT-5 Behavior | Breaking Change? |
|----------|----------------|----------------|------------------|
| `POST /extract` | ✅ Works | ✅ Works (identical logic) | ❌ No |
| `POST /chat_bot` | ✅ Works | ✅ Works (identical logic) | ❌ No |
| `GET /models` | Returns GPT-4 list | Returns GPT-4 + GPT-5 list | ❌ No (additive) |
| Error responses | Standard format | Standard format | ❌ No |

---

## Testing Checklist

### Backend Integration Tests

- [ ] Call `/extract` with `model: "openai_gpt_5"` → Verify successful extraction
- [ ] Call `/extract` with `model: "openai_gpt_5_mini"` → Verify cost calculation in logs
- [ ] Call `/extract` with missing config → Verify 400 error with helpful message
- [ ] Call `/chat_bot` with `model: "openai_gpt_5_chat"` → Verify chat response
- [ ] Call `/chat_bot` with invalid model → Verify error handling

### Frontend Manual Tests

- [ ] Open extraction settings → Verify GPT-5 models at top of dropdown
- [ ] Select `openai_gpt_5` → Process document → Verify success
- [ ] Open chat settings → Verify GPT-5 models at top of dropdown
- [ ] Select `openai_gpt_5_chat` → Ask question → Verify response

### Contract Regression Tests

- [ ] Existing GPT-4o extraction still works (no changes to behavior)
- [ ] Existing GPT-4o chat still works (no session interruption)
- [ ] Error messages still user-friendly for all models

---

## OpenAPI Schema Updates

**Not required** - Existing OpenAPI spec (available at `/docs`) dynamically accepts any model name as string parameter. No schema modifications needed.

**Verification**: After deployment, access `/docs` and confirm:
- `/extract` endpoint accepts `model` parameter (type: string)
- `/chat_bot` endpoint accepts `model` parameter (type: string)
- No hard-coded enum restricting model values

---

## Summary

**API Contract Status**: ✅ **NO CHANGES REQUIRED**

- GPT-5 models use existing endpoints
- Request/response schemas unchanged
- Error handling reuses existing logic
- 100% backward compatible
- Additive only (no breaking changes)

**Ready for implementation**: All contracts validated for GPT-5 compatibility.

---

**Contracts review completed**: 2025-11-04
**Phase 1 status**: Contracts validated - no API changes needed
