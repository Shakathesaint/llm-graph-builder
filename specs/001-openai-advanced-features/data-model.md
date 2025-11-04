# Data Model: GPT-5 Model Support

**Feature**: GPT-5 Model Support
**Date**: 2025-11-04
**Purpose**: Define entity structures for GPT-5 model variants and runtime configuration

---

## Entity 1: GPT-5 Model Variant

**Purpose**: Represents one of the four GPT-5 model options available to users for entity extraction and chat operations.

### Attributes

| Attribute | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `model_id` | string | ✅ Yes | Internal identifier used in frontend Constants.ts | `"openai_gpt_5"`, `"openai_gpt_5_mini"` |
| `api_model_name` | string | ✅ Yes | Exact model name passed to OpenAI API | `"gpt-5"`, `"gpt-5-mini"` |
| `display_name` | string | ✅ Yes | Human-readable name shown in UI dropdowns | `"GPT-5"`, `"GPT-5 Mini"` |
| `input_price_per_1m_tokens` | decimal | ✅ Yes | Cost per 1 million input tokens in USD | `1.25`, `0.25`, `0.05` |
| `output_price_per_1m_tokens` | decimal | ✅ Yes | Cost per 1 million output tokens in USD | `10.00`, `2.00`, `0.40` |
| `description` | string | ⬜ No | Brief description of model characteristics | `"Flagship model with best accuracy"` |

### Validation Rules

1. **Model ID Format**: Must match pattern `^openai_gpt_5(_mini|_nano|_chat)?$`
   - Valid: `openai_gpt_5`, `openai_gpt_5_mini`
   - Invalid: `gpt_5`, `openai_gpt5`, `OPENAI_GPT_5`

2. **API Model Name Format**: Must match pattern `^gpt-5(-mini|-nano|-chat)?$`
   - Valid: `gpt-5`, `gpt-5-mini`
   - Invalid: `gpt5`, `GPT-5`, `gpt-5-turbo`

3. **Pricing**: Must be positive decimals
   - Minimum: `0.01` (1 cent per 1M tokens)
   - Maximum: `1000.00` (reasonable upper bound)

4. **Display Name**: Must be unique across all models for user clarity
   - Length: 3-30 characters
   - Format: Title case with spaces allowed

### Relationships

- **Used by**: User (via model selection in extraction/chat settings)
- **Configured in**: Backend environment variables (`LLM_MODEL_CONFIG_{model_id}`)
- **Listed in**: Frontend Constants.ts `llms` array
- **Passed to**: LangChain ChatOpenAI as `model` parameter

### State Transitions

N/A - Models are static configuration (no lifecycle states)

### Storage Locations

| Location | Format | Example |
|----------|--------|---------|
| **Backend Config** | Environment variable | `LLM_MODEL_CONFIG_OPENAI_GPT_5=gpt-5,use_openai_api_key` |
| **Frontend Config** | TypeScript array | `llms = ['openai_gpt_5', 'openai_gpt_5_mini', ...]` |
| **User Selection** | Browser localStorage | `{"selectedModel_extraction": "openai_gpt_5"}` |

### Instance Data

| Model ID | API Name | Display Name | Input Price | Output Price | Description |
|----------|----------|--------------|-------------|--------------|-------------|
| `openai_gpt_5` | `gpt-5` | `GPT-5` | `$1.25` | `$10.00` | Flagship model with best accuracy and reasoning |
| `openai_gpt_5_mini` | `gpt-5-mini` | `GPT-5 Mini` | `$0.25` | `$2.00` | Balanced cost and performance |
| `openai_gpt_5_nano` | `gpt-5-nano` | `GPT-5 Nano` | `$0.05` | `$0.40` | Most cost-effective option |
| `openai_gpt_5_chat` | `gpt-5-chat` | `GPT-5 Chat` | `$1.25` | `$10.00` | Optimized for conversational interactions |

---

## Entity 2: LLM Configuration (for GPT-5)

**Purpose**: Represents runtime configuration for an OpenAI API call using a GPT-5 model.

### Attributes

| Attribute | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `model_name` | string | ✅ Yes | N/A | API model name (e.g., `"gpt-5"`) |
| `api_key` | string | ✅ Yes | N/A | OpenAI API key or `"use_openai_api_key"` placeholder |
| `temperature` | float | ⬜ No | `0` | Sampling temperature (0.0-2.0) for deterministic output |
| `top_p` | float | ⬜ No | `1.0` | Nucleus sampling parameter (0.0-1.0) |
| `max_tokens` | int | ⬜ No | `None` | Maximum output tokens (None = model decides) |
| `timeout` | int | ⬜ No | `120` | API request timeout in seconds |
| `max_retries` | int | ⬜ No | `2` | Maximum retry attempts for transient errors |

### Validation Rules

1. **API Key**: Must be non-empty string or `"use_openai_api_key"` placeholder
2. **Temperature**: Range 0.0-2.0 (0 = deterministic, 2 = maximum randomness)
3. **Top-p**: Range 0.0-1.0 (nucleus sampling threshold)
4. **Max Tokens**: Positive integer or None
5. **Timeout**: 10-600 seconds (10s minimum, 10 minutes maximum)
6. **Max Retries**: 0-5 retries (0 = no retries, 5 = maximum for safety)

### Relationships

- **Generated from**: User's model selection + system defaults
- **Passed to**: LangChain `ChatOpenAI` constructor
- **Used by**: OpenAI API client (via LangChain wrapper)

### Lifecycle

1. **Creation**: When user selects GPT-5 model for extraction/chat
2. **Initialization**: `llm.py:get_llm()` creates ChatOpenAI instance with config
3. **Execution**: LangChain passes config to OpenAI SDK for API call
4. **Completion**: Config discarded after operation (stateless)

### Code Example

```python
# backend/src/llm.py (lines 51-70)
elif "openai" in model:
    model_name, api_key_config_value = env_value.split(",")
    actual_api_key = (
        os.getenv("OPENAI_API_KEY")
        if api_key_config_value.strip().lower() == "use_openai_api_key"
        else api_key_config_value.strip()
    )

    llm = ChatOpenAI(
        api_key=actual_api_key,        # ← LLM Configuration
        model=model_name,               # ← "gpt-5", "gpt-5-mini", etc.
        temperature=0,                  # ← Deterministic for entity extraction
    )
```

---

## Entity 3: Token Usage Record (for GPT-5)

**Purpose**: Tracks actual token consumption and cost for a completed GPT-5 operation.

### Attributes

| Attribute | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `model_used` | string | ✅ Yes | API model name | `"gpt-5"` |
| `input_tokens` | int | ✅ Yes | Number of input tokens consumed | `5000` |
| `output_tokens` | int | ✅ Yes | Number of output tokens generated | `1200` |
| `total_cost` | decimal | ✅ Yes (calculated) | Total cost in USD | `$0.18` |
| `operation_type` | enum | ✅ Yes | Type of operation | `"extraction"`, `"chat"`, `"qa"` |
| `timestamp` | datetime | ✅ Yes | When operation completed | `"2025-11-04T10:30:00Z"` |
| `user_id` | string | ⬜ No | User identifier (if available) | `"user_123"` |
| `file_name` | string | ⬜ No | Document processed (for extraction) | `"contract.pdf"` |

### Validation Rules

1. **Token counts**: Must be non-negative integers
2. **Total cost**: Calculated via formula (see below)
3. **Operation type**: Must be one of: `"extraction"`, `"chat"`, `"qa"`
4. **Timestamp**: ISO 8601 format

### Cost Calculation Formula

```python
# Pricing constants (from research.md)
GPT5_PRICING = {
    "gpt-5": {"input": 1.25, "output": 10.00},
    "gpt-5-mini": {"input": 0.25, "output": 2.00},
    "gpt-5-nano": {"input": 0.05, "output": 0.40},
    "gpt-5-chat": {"input": 1.25, "output": 10.00},
}

def calculate_cost(model_used, input_tokens, output_tokens):
    pricing = GPT5_PRICING.get(model_used, {"input": 0, "output": 0})
    input_cost = (input_tokens * pricing["input"]) / 1_000_000
    output_cost = (output_tokens * pricing["output"]) / 1_000_000
    return input_cost + output_cost
```

**Example**:
- Model: `gpt-5-mini`
- Input tokens: `5000`
- Output tokens: `1200`
- Cost: `(5000 * 0.25 + 1200 * 2.00) / 1,000,000 = $0.00365` (0.365 cents)

### Relationships

- **Created after**: Each OpenAI API call completion
- **Logged to**: Application logs (structured JSON format)
- **Aggregated for**: Cost reporting and monitoring

### Storage

| Location | Format | Purpose |
|----------|--------|---------|
| **Application Logs** | JSON structured log | Primary storage, searchable |
| **Console Output** | Formatted string | Real-time visibility during development |
| **Future**: Database | Table row | Analytics dashboard (optional enhancement) |

### Log Format Example

```json
{
  "timestamp": "2025-11-04T10:30:00Z",
  "level": "INFO",
  "message": "GPT-5 operation completed",
  "model_used": "gpt-5-mini",
  "operation_type": "extraction",
  "file_name": "contract.pdf",
  "user_id": "user_123",
  "input_tokens": 5000,
  "output_tokens": 1200,
  "total_cost": 0.00365,
  "cost_breakdown": {
    "input_cost": 0.00125,
    "output_cost": 0.0024
  }
}
```

---

## Entity 4: Model Selection UI State

**Purpose**: Represents the user's current model choice in the frontend, persisted across sessions.

### Attributes

| Attribute | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `selected_model_id` | string | ✅ Yes | Model ID from Constants.ts | `"openai_gpt_5_mini"` |
| `context` | enum | ✅ Yes | Where model is used | `"extraction"`, `"chat"` |
| `persisted_in_storage` | boolean | ✅ Yes | Whether selection is saved | `true` |
| `last_updated` | datetime | ⬜ No | When selection last changed | `"2025-11-04T10:30:00Z"` |

### Validation Rules

1. **Selected Model ID**: Must exist in Constants.ts `llms` array
2. **Context**: Must be `"extraction"` or `"chat"`
3. **Persisted**: Boolean flag (true = saved to localStorage)

### Relationships

- **Stored in**: Browser localStorage (key: `selectedModel_{context}`)
- **Read by**: Model selection dropdown components
- **Passed to**: Backend API calls (as `model` parameter in request body)

### Storage Keys

| Context | LocalStorage Key | Example Value |
|---------|------------------|---------------|
| Extraction | `selectedModel_extraction` | `"openai_gpt_5"` |
| Chat | `selectedModel_chat` | `"openai_gpt_5_chat"` |

### Lifecycle

1. **Initialization**: On page load, read from localStorage or use default
2. **Update**: When user selects different model from dropdown
3. **Persistence**: Save to localStorage immediately after selection
4. **Usage**: Include in API request body when calling `/extract` or `/chat_bot`

### Code Example

```typescript
// Frontend: Reading model selection
const getSelectedModel = (context: 'extraction' | 'chat'): string => {
  const storageKey = `selectedModel_${context}`;
  const stored = localStorage.getItem(storageKey);
  return stored || 'openai_gpt_4o'; // Fallback to default
};

// Frontend: Saving model selection
const setSelectedModel = (context: 'extraction' | 'chat', modelId: string): void => {
  const storageKey = `selectedModel_${context}`;
  localStorage.setItem(storageKey, modelId);
};
```

---

## Data Flow Diagram

```
User Action (Select GPT-5 in dropdown)
    ↓
Model Selection UI State (localStorage)
    ↓
Frontend API Call (/extract or /chat_bot)
    ↓
Backend llm.py:get_llm(model_id)
    ↓
LLM Configuration (ChatOpenAI initialization)
    ↓
OpenAI API Call (with gpt-5 model name)
    ↓
API Response (with token usage)
    ↓
Token Usage Record (logged with cost calculation)
```

---

## Summary

### Entities Defined

1. **GPT-5 Model Variant** - 4 instances (gpt-5, mini, nano, chat)
2. **LLM Configuration** - Runtime config for API calls
3. **Token Usage Record** - Cost tracking and monitoring
4. **Model Selection UI State** - User preferences persistence

### Key Design Decisions

1. **Pricing**: Hard-coded in application (from research.md findings)
2. **Storage**: Configuration-driven (environment variables + Constants.ts)
3. **Persistence**: Browser localStorage for user selections
4. **Logging**: Structured JSON for token usage and costs

### No Breaking Changes

All entities are **additive** - existing GPT-4/o1/o3 models continue working unchanged.

---

**Data model completed**: 2025-11-04
**Phase 1 status**: Data model ready for implementation
