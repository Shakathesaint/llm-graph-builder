# Quickstart: GPT-5 Model Support

**Feature**: GPT-5 Model Support
**Setup Time**: ~5-10 minutes
**Prerequisites**: OpenAI API key, LLM Graph Builder deployed

---

## Prerequisites

✅ **Required**:
- OpenAI API key with GPT-5 access (standard API key as of August 2025, no beta access needed)
- LLM Graph Builder backend running (Python 3.11+, FastAPI)
- LLM Graph Builder frontend running (React 18, Vite)
- Neo4j instance configured and accessible

⬜ **Optional**:
- Docker/Docker Compose for containerized deployment
- Cloud deployment (GCP Cloud Run, AWS ECS, Azure Container Apps)

---

## Step 1: Backend Configuration (3 minutes)

### 1.1 Add GPT-5 Environment Variables

Edit your `backend/.env` file:

```bash
# GPT-5 Models (August 2025 release)
# Pricing: gpt-5 ($1.25/$10), gpt-5-mini ($0.25/$2), gpt-5-nano ($0.05/$0.40) per 1M input/output tokens

LLM_MODEL_CONFIG_OPENAI_GPT_5=gpt-5,use_openai_api_key
LLM_MODEL_CONFIG_OPENAI_GPT_5_MINI=gpt-5-mini,use_openai_api_key
LLM_MODEL_CONFIG_OPENAI_GPT_5_NANO=gpt-5-nano,use_openai_api_key
LLM_MODEL_CONFIG_OPENAI_GPT_5_CHAT=gpt-5-chat,use_openai_api_key

# Ensure OPENAI_API_KEY is set
OPENAI_API_KEY=sk-proj-your-actual-key-here
```

**Alternative**: Use different API key per model:
```bash
LLM_MODEL_CONFIG_OPENAI_GPT_5=gpt-5,sk-different-key-here
```

### 1.2 Restart Backend

```bash
cd backend
source envName/bin/activate  # Activate venv (Linux/Mac)
# OR
envName\Scripts\activate     # Windows

uvicorn score:app --reload
```

**Verification**: Check logs for successful model loading:
```
INFO: Model: LLM_MODEL_CONFIG_OPENAI_GPT_5
INFO: Model: LLM_MODEL_CONFIG_OPENAI_GPT_5_MINI
INFO: Model: LLM_MODEL_CONFIG_OPENAI_GPT_5_NANO
INFO: Model: LLM_MODEL_CONFIG_OPENAI_GPT_5_CHAT
```

---

## Step 2: Frontend Configuration (2 minutes)

### 2.1 Update Model List

Edit `frontend/.env` (create if it doesn't exist):

```bash
# List GPT-5 models first to show at top of dropdowns
VITE_LLM_MODELS=openai_gpt_5,openai_gpt_5_mini,openai_gpt_5_nano,openai_gpt_5_chat,openai_gpt_4o,openai_gpt_4o_mini,gemini_2.0_flash,diffbot
```

### 2.2 Rebuild Frontend

```bash
cd frontend
yarn build
```

**Development mode** (hot reload):
```bash
yarn run dev
```

### 2.3 Verify in Browser

1. Open browser → Navigate to `http://localhost:5173` (dev) or `http://localhost:8080` (production)
2. Go to **Settings** or **Model Selection**
3. Confirm GPT-5 models appear at top of dropdown:
   - GPT-5
   - GPT-5 Mini
   - GPT-5 Nano
   - GPT-5 Chat

---

## Step 3: Test Entity Extraction (5 minutes)

### 3.1 Select GPT-5 Model

1. Open LLM Graph Builder UI
2. Navigate to **Settings** → **Model Selection** (Extraction tab)
3. Select **GPT-5 Mini** (recommended for first test - good balance of cost/performance)

### 3.2 Upload Test Document

1. Click **Upload** or drag-and-drop a document:
   - **Recommended**: 5-10 page PDF (quick test)
   - **Supported formats**: PDF, DOCX, TXT, CSV
2. Click **Generate Graph** or **Extract Entities**

### 3.3 Monitor Extraction

Watch console/logs for:
```
INFO: Processing document with model: gpt-5-mini
INFO: Chunks processed: 5/10
INFO: Extraction completed - 45 entities, 60 relationships
INFO: Token usage: 8,500 input, 2,300 output (total: 10,800 tokens)
INFO: Cost: $0.027 (input: $0.002, output: $0.005)
```

**Expected results**:
- ✅ Extraction completes successfully
- ✅ Entities visible in Neo4j graph
- ✅ Cost logged in backend logs

---

## Step 4: Test Chat/Q&A (3 minutes)

### 4.1 Select GPT-5 Chat Model

1. Navigate to **Chat** tab
2. Select **GPT-5 Chat** from model dropdown (optimized for conversations)
3. Choose retrieval mode: **graph+vector** (recommended)

### 4.2 Ask Question

Example questions:
- "What are the main entities in the document?"
- "Summarize the key relationships between people and organizations"
- "What is the document about?"

### 4.3 Review Response

Expected output:
- ✅ Accurate answer with context from extracted graph
- ✅ Source citations showing which chunks informed the answer
- ✅ Token usage and cost logged

---

## Cost Optimization Guide

### Model Selection Strategy

| Use Case | Recommended Model | Why |
|----------|-------------------|-----|
| **Production extraction** (general) | GPT-5 Mini | 80% cost savings vs GPT-5, similar quality |
| **Complex documents** (legal, medical, technical) | GPT-5 | Best accuracy for nuanced entity extraction |
| **Batch processing** (thousands of documents) | GPT-5 Nano | 96% cost savings, acceptable for simple documents |
| **Interactive chat** (user-facing Q&A) | GPT-5 Chat | Optimized for conversational flow |

### Cost Comparison Example

**50-page PDF document** (~50K input tokens, ~10K output tokens):

| Model | Input Cost | Output Cost | **Total** |
|-------|-----------|-------------|----------|
| GPT-5 | $0.0625 | $0.100 | **$0.16** |
| GPT-5 Mini | $0.0125 | $0.020 | **$0.03** ✅ 81% savings |
| GPT-5 Nano | $0.0025 | $0.004 | **$0.01** ✅ 94% savings |

**Monthly cost** (1,000 documents/month):
- GPT-5: $160/month
- GPT-5 Mini: $30/month ✅ **$130 savings**
- GPT-5 Nano: $6/month ✅ **$154 savings**

---

## Troubleshooting

### Error: "Environment variable 'LLM_MODEL_CONFIG_OPENAI_GPT_5' is not defined"

**Cause**: Backend `.env` file missing GPT-5 configuration.

**Fix**:
1. Add the 4 environment variables from Step 1.1
2. Restart backend: `uvicorn score:app --reload`
3. Verify logs show "Model: LLM_MODEL_CONFIG_OPENAI_GPT_5"

---

### Error: "Model 'gpt-5' not available for your API key"

**Cause**: OpenAI account doesn't have GPT-5 access.

**Fix**:
1. Check OpenAI dashboard → **API** → **Models**
2. Verify "gpt-5" appears in available models list
3. If not listed:
   - Contact OpenAI support for access
   - Use GPT-4o as fallback: select "openai_gpt_4o" in dropdown

---

### GPT-5 Models Not Appearing in Dropdown

**Cause**: Frontend `.env` not updated or frontend not rebuilt.

**Fix**:
1. Verify `frontend/.env` contains GPT-5 models in `VITE_LLM_MODELS`
2. Rebuild: `cd frontend && yarn build`
3. Clear browser cache: Ctrl+Shift+R (Windows/Linux) or Cmd+Shift+R (Mac)
4. Refresh page

---

### Cost Higher Than Expected

**Cause**: Using gpt-5 instead of gpt-5-mini for routine extractions.

**Fix**:
1. Review logs for token usage: `grep "Token usage" backend.log`
2. Switch to GPT-5 Mini for 80% cost reduction (similar quality)
3. Reserve GPT-5 for complex documents only

**Enable cost tracking**:
```bash
# Add to backend/.env
LOG_LEVEL=INFO  # Ensures token usage logged
LANGCHAIN_TRACING_V2=true  # Detailed LLM tracing (optional)
```

---

### Extraction Timeout

**Cause**: Large document + default timeout (120s).

**Fix**: Increase timeout in backend `.env`:
```bash
# Add custom timeout for large documents
OPENAI_TIMEOUT=300  # 5 minutes
```

Then restart backend.

---

## Performance Tips

### 1. Chunking Strategy

**Default** (works for most cases):
- Chunk size: 10,000 tokens
- Overlap: 200 tokens

**For GPT-5** (experiment with larger chunks):
```python
# In backend extraction settings or code
chunk_size = 15000  # GPT-5's reasoning handles larger contexts better
chunk_overlap = 300
```

### 2. Batch Processing

For processing many documents:
1. Use **GPT-5 Nano** (cheapest)
2. Process overnight (use "flex" tier if available for additional savings)
3. Monitor logs for cost tracking

### 3. Monitoring

**Enable structured logging**:
```bash
# backend/.env
LOG_FORMAT=json  # Structured logs for easier parsing
LANGCHAIN_TRACING_V2=true  # Trace LLM calls
LANGCHAIN_PROJECT=gpt5-testing  # Organize traces
```

**Track costs**:
```bash
# Search logs for cost information
grep "total_cost" backend.log | awk '{sum+=$NF} END {print "Total cost: $" sum}'
```

### 4. Quality Validation

**A/B testing** (compare GPT-5 Mini vs GPT-5):
1. Process same document with both models
2. Compare entity extraction accuracy
3. Use GPT-5 only if Mini quality insufficient (rare for most documents)

---

## Advanced Usage

### Docker Compose Deployment

Add GPT-5 config to `docker-compose.yml`:

```yaml
services:
  backend:
    environment:
      - LLM_MODEL_CONFIG_OPENAI_GPT_5=gpt-5,${OPENAI_API_KEY}
      - LLM_MODEL_CONFIG_OPENAI_GPT_5_MINI=gpt-5-mini,${OPENAI_API_KEY}
      - LLM_MODEL_CONFIG_OPENAI_GPT_5_NANO=gpt-5-nano,${OPENAI_API_KEY}
      - LLM_MODEL_CONFIG_OPENAI_GPT_5_CHAT=gpt-5-chat,${OPENAI_API_KEY}
      - OPENAI_API_KEY=${OPENAI_API_KEY}

  frontend:
    environment:
      - VITE_LLM_MODELS=openai_gpt_5,openai_gpt_5_mini,openai_gpt_5_nano,openai_gpt_5_chat,openai_gpt_4o
```

Then deploy:
```bash
docker-compose up --build
```

### Cloud Deployment (GCP Cloud Run)

Set environment variables via `gcloud`:
```bash
gcloud run deploy llm-graph-builder-backend \
  --set-env-vars="LLM_MODEL_CONFIG_OPENAI_GPT_5=gpt-5,use_openai_api_key" \
  --set-env-vars="LLM_MODEL_CONFIG_OPENAI_GPT_5_MINI=gpt-5-mini,use_openai_api_key" \
  --set-env-vars="OPENAI_API_KEY=sk-your-key-here"
```

---

## Next Steps

### Optimization
- [ ] Run A/B tests: GPT-5 vs GPT-5 Mini extraction quality
- [ ] Benchmark performance: Processing time per document
- [ ] Establish baseline costs: Track first week of GPT-5 usage

### Monitoring
- [ ] Set up log aggregation (CloudWatch, Stackdriver, ELK)
- [ ] Create cost dashboard (aggregate token usage + costs)
- [ ] Set budget alerts in OpenAI dashboard

### Experimentation
- [ ] Test GPT-5 on complex documents (legal, medical, technical)
- [ ] Try larger chunk sizes (15K-20K tokens) with GPT-5
- [ ] Compare chat quality: GPT-5 Chat vs GPT-5 vs GPT-5 Mini

### Documentation
- [ ] Share findings with team (cost savings, quality trade-offs)
- [ ] Update internal wiki with GPT-5 best practices
- [ ] Document cost optimization strategies

---

## Support Resources

- **Documentation**: [CLAUDE.md](../../CLAUDE.md) - Architecture details
- **Spec**: [spec.md](../spec.md) - Feature requirements
- **Data Model**: [data-model.md](../data-model.md) - Entity definitions
- **Contracts**: [contracts/README.md](../contracts/README.md) - API details
- **Issues**: Report bugs at project GitHub Issues page
- **Community**: Join project Slack/Discord for discussions

---

## FAQ

**Q: Can I mix GPT-5 and GPT-4 in the same workflow?**
A: Yes! Model selection is per-operation. Extract with GPT-5 Mini, chat with GPT-4o - fully supported.

**Q: Does GPT-5 work with all document sources (local, S3, GCS, YouTube, web)?**
A: Yes! Source loading is model-agnostic. GPT-5 works with all 6+ supported sources.

**Q: Can I change models mid-extraction?**
A: No - model is set when extraction starts. To switch models, start a new extraction job.

**Q: What happens to existing extractions after adding GPT-5?**
A: Nothing! Existing graphs remain unchanged. GPT-5 only affects new extractions.

**Q: Is GPT-5 faster than GPT-4?**
A: Similar latency per request. GPT-5's "unified reasoning" may reduce retry needs, potentially improving overall throughput for complex extractions.

---

**Quickstart completed**: 2025-11-04
**Ready to use**: GPT-5 models available for extraction and chat
