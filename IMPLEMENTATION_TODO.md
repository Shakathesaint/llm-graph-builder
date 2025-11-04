# TODO: Implementazione Modelli OpenAI Avanzati con Cost & Performance Optimization

## 🎯 Obiettivo Generale

Migliorare l'integrazione dei modelli OpenAI esistenti aggiungendo supporto per le funzionalità avanzate di OpenAI:
- **Prompt Caching**: Monitoraggio del caching automatico (50% sconto su token cached)
- **Priority Processing**: Supporto per service tier "priority" (40% più veloce) e "flex" (batch-like, costi ridotti)
- **Configuration Flexibility**: Timeout, retry, e custom headers configurabili
- **Reasoning Models Detection**: Gestione corretta di tutti i modelli o1/o3

**Benefici Attesi:**
- ⚡ Riduzione latenza con priority tier
- 💰 Riduzione costi con flex tier e prompt caching
- 🔍 Visibilità su cache hits per ottimizzazione
- 🛠️ Maggiore controllo su timeout e retry logic

---

## 📋 Checklist Implementazione

### Phase 1: Backend Core - LLM Configuration Enhancement

#### ✅ Task 1.1: Aggiornare `backend/src/llm.py`
**File:** `backend/src/llm.py`
**Linee:** 51-70 (sezione OpenAI)

**Obiettivo:** Aggiungere parametri avanzati a ChatOpenAI per supportare service_tier, timeout, retry e custom headers.

**Modifiche da fare:**
1. Importare `json` module all'inizio del file (se non già presente)
2. Sostituire il blocco `elif "openai" in model:` (linee 51-70) con logica migliorata
3. Aggiungere detection completa reasoning models (o1, o3)
4. Leggere environment variables per configurazione avanzata:
   - `OPENAI_SERVICE_TIER` (default: "auto")
   - `OPENAI_TIMEOUT` (default: 120 secondi)
   - `OPENAI_MAX_RETRIES` (default: 2)
   - `OPENAI_CUSTOM_HEADERS` (optional, JSON string)

**Codice da implementare:**
```python
elif "openai" in model:
    model_name, api_key_config_value = env_value.split(",")

    # Resolve API key (existing logic)
    if api_key_config_value.strip().lower() == "use_openai_api_key":
        actual_api_key = os.getenv("OPENAI_API_KEY")
        if not actual_api_key:
            raise LLMGraphBuilderException("OPENAI_API_KEY environment variable is not set")
    else:
        actual_api_key = api_key_config_value.strip()

    # Determine if reasoning model (o1, o3 series don't support temperature)
    is_reasoning_model = any(
        rm in model.lower()
        for rm in ["o1-", "o3-", "o1_", "o3_", "gpt-o1", "gpt-o3"]
    )

    # Read advanced configuration from environment
    service_tier = os.getenv("OPENAI_SERVICE_TIER", "auto")  # auto/default/priority/flex
    timeout = int(os.getenv("OPENAI_TIMEOUT", "120"))  # seconds
    max_retries = int(os.getenv("OPENAI_MAX_RETRIES", "2"))

    # Optional custom headers (for tracking, routing, etc.)
    custom_headers = {}
    custom_headers_env = os.getenv("OPENAI_CUSTOM_HEADERS")
    if custom_headers_env:
        try:
            custom_headers = json.loads(custom_headers_env)
        except json.JSONDecodeError:
            logging.warning(f"Invalid OPENAI_CUSTOM_HEADERS JSON: {custom_headers_env}")

    # Build ChatOpenAI kwargs
    llm_kwargs = {
        "api_key": actual_api_key,
        "model": model_name,
        "timeout": timeout,
        "max_retries": max_retries,
    }

    # Add service_tier if specified
    if service_tier in ["auto", "default", "priority", "flex"]:
        llm_kwargs["service_tier"] = service_tier

    # Add custom headers if provided
    if custom_headers:
        llm_kwargs["default_headers"] = custom_headers

    # Temperature only for non-reasoning models
    if not is_reasoning_model:
        llm_kwargs["temperature"] = 0

    llm = ChatOpenAI(**llm_kwargs)

    logging.info(f"OpenAI model {model_name} initialized with service_tier={service_tier}, timeout={timeout}s")
```

**Test di verifica:**
```bash
# Nel backend directory
python -c "from src.llm import get_llm; llm, name = get_llm('openai_gpt_4o'); print(f'✓ Model loaded: {name}')"
```

---

#### ✅ Task 1.2: Aggiornare `backend/src/QA_integration.py`
**File:** `backend/src/QA_integration.py`
**Funzione:** `get_total_tokens()` (linee 74-100 circa)

**Obiettivo:** Aggiungere monitoraggio dei token cachati per visibilità sui risparmi da prompt caching.

**Modifiche da fare:**
1. Modificare la funzione per estrarre e loggare `cached_tokens` da response metadata
2. Restituire sia total_tokens che cached_tokens (tuple)
3. Aggiungere logging quando prompt caching è attivo

**Codice da implementare:**
```python
def get_total_tokens(ai_response, llm):
    """
    Extract token usage from LLM response metadata.
    Returns: (total_tokens, cached_tokens)
    """
    total_tokens = 0
    cached_tokens = 0

    try:
        if isinstance(llm, (ChatOpenAI, AzureChatOpenAI, ChatFireworks, ChatGroq)):
            usage = ai_response.response_metadata.get('token_usage', {})
            total_tokens = usage.get('total_tokens', 0)

            # Extract cached tokens (OpenAI prompt caching feature)
            cached_tokens = usage.get('cached_tokens', 0)

            if cached_tokens > 0:
                cache_percentage = (cached_tokens / total_tokens * 100) if total_tokens > 0 else 0
                logging.info(
                    f"✓ Prompt caching active: {cached_tokens}/{total_tokens} tokens cached "
                    f"({cache_percentage:.1f}% - 50% cost reduction on cached portion)"
                )

        elif isinstance(llm, ChatVertexAI):
            usage = ai_response.response_metadata.get('usage_metadata', {})
            total_tokens = usage.get('total_token_count', 0)
            cached_tokens = usage.get('cached_content_token_count', 0)

        # ... rest of existing elif blocks for other providers

    except Exception as e:
        logging.warning(f"Failed to extract token usage: {e}")

    return total_tokens, cached_tokens
```

**Modifiche downstream:**
- Aggiornare chiamate a `get_total_tokens()` per gestire tuple return value
- Esempio: `total_tokens, cached_tokens = get_total_tokens(ai_response, llm)`

**Test di verifica:**
```python
# Dopo aver fatto alcune query ripetute, verificare nei log:
# "✓ Prompt caching active: 2500/5000 tokens cached (50.0% - 50% cost reduction)"
```

---

### Phase 2: Configuration Files

#### ✅ Task 2.1: Aggiornare `backend/example.env`
**File:** `backend/example.env`

**Obiettivo:** Documentare le nuove variabili di configurazione per funzionalità avanzate OpenAI.

**Modifiche da fare:**
Aggiungere la seguente sezione dopo le configurazioni OpenAI esistenti:

```bash
##############################################
# OpenAI Advanced Configuration (Optional)
##############################################

# Service Tier for Priority Processing
# - "auto" (default): OpenAI decides based on usage patterns
# - "default": Standard processing speed and pricing
# - "priority": 40% faster processing, premium pricing
# - "flex": Batch-like flexible timing, reduced cost (beta)
# Docs: https://platform.openai.com/docs/guides/priority-processing
OPENAI_SERVICE_TIER=auto

# API Timeout (seconds)
# Increase for large document processing or reasoning models
OPENAI_TIMEOUT=120

# Max Retries on API Errors
# Number of automatic retries for rate limits or transient errors
OPENAI_MAX_RETRIES=2

# Custom HTTP Headers (optional, JSON format)
# Use for request tracking, routing, or custom authentication
# Example: OPENAI_CUSTOM_HEADERS={"X-Request-ID": "my-app-v1", "X-User-ID": "user123"}
# OPENAI_CUSTOM_HEADERS={}

##############################################
# Prompt Caching (Automatic - No Config Needed)
##############################################
# OpenAI automatically caches prompts > 1,024 tokens for:
# - gpt-4o, gpt-4o-mini (all dated versions)
# - o1-preview, o1-mini, o3-mini
#
# Benefits:
# - 50% discount on cached input tokens
# - 80% latency reduction on cache hits
# - 5-10 min cache lifetime (max 1 hour)
#
# Monitoring: Check application logs for "Prompt caching active" messages
# Docs: https://platform.openai.com/docs/guides/prompt-caching
```

**Test di verifica:**
```bash
# Verificare che le variabili siano leggibili
grep "OPENAI_SERVICE_TIER" backend/.env
```

---

#### ✅ Task 2.2: Aggiornare `backend/.env` (se necessario)
**File:** `backend/.env` (file locale, non committato)

**Obiettivo:** Aggiungere le nuove variabili al file di configurazione locale.

**Modifiche da fare:**
1. Copiare le nuove variabili da `example.env`
2. Configurare valori appropriati per il proprio environment:
   - Usare `priority` se serve velocità massima
   - Usare `flex` se si vuole risparmiare sui costi
   - Lasciare `auto` per bilanciamento automatico

**Esempio configurazione production (alta priorità):**
```bash
OPENAI_SERVICE_TIER=priority
OPENAI_TIMEOUT=180
OPENAI_MAX_RETRIES=3
```

**Esempio configurazione development (costi ridotti):**
```bash
OPENAI_SERVICE_TIER=flex
OPENAI_TIMEOUT=300  # Flex può essere più lento
OPENAI_MAX_RETRIES=2
```

---

### Phase 3: Frontend Synchronization

#### ✅ Task 3.1: Verificare `frontend/src/utils/Constants.ts`
**File:** `frontend/src/utils/Constants.ts`
**Linee:** 11-41

**Obiettivo:** Assicurarsi che tutti i modelli backend siano esposti nel frontend e sincronizzati.

**Azioni da fare:**
1. Leggere i modelli configurati in `backend/.env` (LLM_MODEL_CONFIG_*)
2. Confrontare con l'array `llms` in Constants.ts
3. Aggiungere eventuali modelli mancanti
4. Verificare naming consistency (es: `openai_gpt_4o` vs `openai-gpt-4o`)

**Modelli attualmente nel backend da verificare:**
```typescript
// Verificare che siano tutti presenti in frontend/src/utils/Constants.ts
const expectedModels = [
    'openai_gpt_3.5',
    'openai_gpt_4o_mini',
    'openai_gpt_4o',
    'openai_gpt_4.1_mini',
    'openai_gpt_4.1',
    'openai_gpt_o3_mini',
];
```

**Se mancano modelli, aggiungerli all'array `llms`:**
```typescript
export const llms =
  process.env?.VITE_LLM_MODELS?.trim() != ''
    ? (process.env.VITE_LLM_MODELS?.split(',') as string[])
    : [
        'openai_gpt_4o',
        'openai_gpt_4o_mini',
        'openai_gpt_4.1',
        'openai_gpt_4.1_mini',
        'openai_gpt_o3_mini',
        'openai_gpt_3.5',  // ← verificare se presente
        // ... altri modelli
      ];
```

**Test di verifica:**
```bash
cd frontend
yarn run dev
# Aprire browser e verificare dropdown modelli LLM nel Settings
```

---

#### ✅ Task 3.2: Aggiornare `frontend/.env` (opzionale)
**File:** `frontend/.env`

**Obiettivo:** Configurare i modelli visibili nel frontend per development vs production.

**Modifiche da fare:**
Verificare/aggiornare:
```bash
# Development - tutti i modelli disponibili
VITE_LLM_MODELS=openai_gpt_4o,openai_gpt_4o_mini,openai_gpt_4.1,openai_gpt_o3_mini,gemini_2.0_flash,diffbot

# Production - solo modelli ottimizzati
VITE_LLM_MODELS_PROD=openai_gpt_4o,openai_gpt_4o_mini,gemini_2.0_flash,diffbot
```

---

### Phase 4: Documentation

#### ✅ Task 4.1: Aggiornare `CLAUDE.md`
**File:** `CLAUDE.md`

**Obiettivo:** Documentare le nuove funzionalità avanzate di OpenAI per futuri sviluppatori.

**Modifiche da fare:**
Aggiungere una nuova sezione nella parte "LLM Provider Integration" (dopo linea 60 circa):

```markdown
### OpenAI Advanced Features

#### Automatic Prompt Caching
OpenAI automatically caches prompts longer than 1,024 tokens for supported models (gpt-4o, gpt-4o-mini, o1, o3 series).

**How it works:**
- First request: Full token cost + processing time
- Repeated similar prompts within 5-10 min: 50% discount on cached portion, 80% latency reduction
- Common use case: Entity extraction from multiple similar document chunks

**Monitoring:**
The application logs cache hits automatically:
```
✓ Prompt caching active: 2500/5000 tokens cached (50.0% - 50% cost reduction)
```

**No configuration needed** - caching is automatic at the OpenAI API level.

**Docs:** https://platform.openai.com/docs/guides/prompt-caching

---

#### Priority Processing (Service Tiers)
Control processing speed and cost with the `OPENAI_SERVICE_TIER` environment variable.

**Available Tiers:**
- `auto` (default): OpenAI dynamically routes based on usage patterns
- `default`: Standard processing speed and pricing
- `priority`: 40% faster processing, premium pricing (~1.2x cost)
- `flex` (beta): Batch-like flexible timing, reduced cost (~0.6x cost)

**Configuration:**
```bash
# In backend/.env
OPENAI_SERVICE_TIER=priority  # For time-sensitive graph extraction
# or
OPENAI_SERVICE_TIER=flex      # For large batch processing at lower cost
```

**Use Cases:**
- **Priority**: Real-time user-facing extraction, urgent document processing
- **Flex**: Overnight batch processing, non-urgent large document sets
- **Auto**: Balanced approach, good for most scenarios

**Docs:** https://platform.openai.com/docs/guides/priority-processing

---

#### Advanced Configuration Options

Additional OpenAI parameters configurable via environment variables:

```bash
OPENAI_TIMEOUT=120           # API timeout in seconds (increase for large docs)
OPENAI_MAX_RETRIES=2         # Automatic retries on rate limits/errors
OPENAI_CUSTOM_HEADERS={}     # Custom HTTP headers (JSON format)
```

**Example - Custom Headers for Request Tracking:**
```bash
OPENAI_CUSTOM_HEADERS={"X-Request-ID": "llm-graph-builder", "X-Environment": "production"}
```
```

**Test di verifica:**
```bash
# Verificare che la documentazione sia chiara e completa
grep -A 5 "Prompt Caching" CLAUDE.md
```

---

#### ✅ Task 4.2: Creare `docs/OPENAI_COST_OPTIMIZATION.md` (opzionale)
**File:** `docs/OPENAI_COST_OPTIMIZATION.md` (nuovo file)

**Obiettivo:** Guida dettagliata per ottimizzare i costi e le performance con i modelli OpenAI.

**Contenuto suggerito:**
```markdown
# OpenAI Cost & Performance Optimization Guide

## Cost Optimization Strategies

### 1. Use Prompt Caching (Automatic)
- **Savings:** 50% on cached input tokens
- **How:** Process similar documents in batches within 10 minutes
- **Best for:** Large document sets with similar structure

### 2. Use Flex Tier for Batch Processing
- **Savings:** ~40% cost reduction vs standard
- **Configuration:** `OPENAI_SERVICE_TIER=flex`
- **Best for:** Non-urgent processing, overnight jobs
- **Tradeoff:** Variable completion time (up to 24h)

### 3. Choose Right Model for Task
- **gpt-4o-mini:** 15x cheaper than gpt-4o, good for simple extraction
- **gpt-4o:** Best accuracy for complex documents
- **o3-mini:** Reasoning model for complex entity relationships

## Performance Optimization Strategies

### 1. Use Priority Tier for Real-Time Processing
- **Speed:** 40% faster than standard
- **Configuration:** `OPENAI_SERVICE_TIER=priority`
- **Best for:** User-facing features, time-sensitive extraction
- **Tradeoff:** ~20% cost increase

### 2. Optimize Chunk Size
- **Large chunks:** Better context, higher accuracy, more caching
- **Small chunks:** Faster processing, lower per-request cost
- **Recommended:** 5000-8000 tokens for optimal caching benefit

### 3. Monitor Cache Hit Rate
Check logs for cache efficiency:
```
grep "Prompt caching active" backend/logs/*.log | wc -l
```

## Cost Calculation Example

**Scenario:** Extract entities from 100 PDF documents (500 pages each)

**Setup:**
- Model: gpt-4o-mini ($0.15 per 1M input tokens)
- Chunk size: 5000 tokens per chunk
- Average: 10 chunks per document
- Total tokens: 5,000,000 input tokens

**Cost Comparison:**

| Configuration | Speed | Cost | Notes |
|--------------|-------|------|-------|
| Standard (no optimization) | Baseline | $0.75 | Full price, standard speed |
| Standard + Prompt Caching | Baseline | $0.56 | 25% savings (assume 50% cache hit rate) |
| Flex Tier + Caching | 2-24h | $0.34 | 55% savings, batch processing |
| Priority Tier + Caching | 0.6x baseline | $0.67 | 11% savings, 40% faster |

**Recommendation:**
- **Development/Testing:** Use flex tier with caching
- **Production:** Use auto tier (balanced) or priority for user-facing features
- **Batch Jobs:** Use flex tier with large chunk sizes for maximum savings
```

---

### Phase 5: Testing & Validation

#### ✅ Task 5.1: Test Priority Tier Processing Speed
**Obiettivo:** Verificare che il priority tier effettivamente riduca la latenza.

**Steps:**
1. Configurare `OPENAI_SERVICE_TIER=default` nel `.env`
2. Estrarre grafo da un documento di test di dimensione media (~50 pagine)
3. Annotare tempo totale di processing
4. Configurare `OPENAI_SERVICE_TIER=priority`
5. Estrarre lo stesso documento
6. Confrontare tempi: priority dovrebbe essere ~40% più veloce

**Comando test:**
```bash
# Backend
cd backend
python -c "
import time
from src.main import extract_graph_from_file_local_file
start = time.time()
# ... run extraction ...
print(f'Processing time: {time.time() - start:.2f}s')
"
```

**Expected results:**
- Default tier: ~120s
- Priority tier: ~70s (40% faster)

---

#### ✅ Task 5.2: Test Prompt Caching Effectiveness
**Obiettivo:** Verificare che il prompt caching funzioni e venga monitorato correttamente.

**Steps:**
1. Estrarre grafo da un documento
2. Verificare nei log se appaiono messaggi "Prompt caching active"
3. Processare un secondo documento simile entro 10 minuti
4. Verificare aumento della percentuale di cache hits

**Comando test:**
```bash
# Verificare log per cache hits
tail -f backend/logs/*.log | grep "Prompt caching"
```

**Expected results:**
```
✓ Prompt caching active: 2450/4800 tokens cached (51.0% - 50% cost reduction)
✓ Prompt caching active: 3200/4800 tokens cached (66.7% - 50% cost reduction)
```

---

#### ✅ Task 5.3: Test Flex Tier for Cost Reduction
**Obiettivo:** Verificare che il flex tier riduca effettivamente i costi (indicato nei response metadata).

**Steps:**
1. Configurare `OPENAI_SERVICE_TIER=flex`
2. Sottomettere un batch di documenti per estrazione
3. Verificare che le richieste vengano accettate
4. Monitorare completion time (può essere più lungo)

**Note:**
- Flex tier è in beta, potrebbe non essere disponibile per tutti gli account
- Se ricevi errore "service_tier not supported", usare auto o priority

---

#### ✅ Task 5.4: Test Timeout Configuration
**Obiettivo:** Verificare che il timeout configurabile funzioni correttamente.

**Steps:**
1. Configurare `OPENAI_TIMEOUT=10` (timeout molto breve)
2. Provare a estrarre un documento grande
3. Dovrebbe andare in timeout con errore appropriato
4. Riconfigurare `OPENAI_TIMEOUT=180` (timeout lungo)
5. La stessa estrazione dovrebbe completare con successo

---

#### ✅ Task 5.5: Test Reasoning Model Detection
**Obiettivo:** Verificare che i modelli o1/o3 non passino il parametro temperature.

**Steps:**
1. Testare con `openai_gpt_o3_mini`
2. Verificare nei log che venga inizializzato senza temperature
3. Verificare che non ci siano errori API relativi a parametri non supportati

**Test code:**
```python
from src.llm import get_llm
llm, model_name = get_llm('openai_gpt_o3_mini')
print(f"Model: {model_name}")
print(f"Has temperature: {'temperature' in str(llm)}")
# Should print: Has temperature: False
```

---

### Phase 6: Monitoring & Optimization

#### ✅ Task 6.1: Creare Dashboard Metrics (opzionale)
**Obiettivo:** Aggiungere endpoint API per esporre metriche di utilizzo caching e service tier.

**File:** `backend/score.py`

**Nuovo endpoint da aggiungere:**
```python
@app.get("/api/metrics/llm_usage")
async def get_llm_usage_metrics():
    """
    Get LLM usage statistics including cache hits and cost savings
    """
    # Query database or logs for metrics
    return {
        "total_requests": 1250,
        "cached_requests": 620,
        "cache_hit_rate": 49.6,  # %
        "estimated_cost_savings": 127.50,  # USD
        "service_tier_distribution": {
            "auto": 800,
            "priority": 200,
            "flex": 250
        }
    }
```

---

#### ✅ Task 6.2: Aggiungere Logging Strutturato (opzionale)
**Obiettivo:** Migliorare il logging per facilitare analisi dei costi e performance.

**File:** `backend/src/llm.py` e `backend/src/QA_integration.py`

**Modifiche da fare:**
Aggiungere structured logging per ogni chiamata LLM:
```python
logging.info(json.dumps({
    "event": "llm_request_complete",
    "model": model_name,
    "service_tier": service_tier,
    "total_tokens": total_tokens,
    "cached_tokens": cached_tokens,
    "cache_hit_rate": cached_tokens / total_tokens if total_tokens > 0 else 0,
    "estimated_cost": calculate_cost(model_name, total_tokens, cached_tokens),
    "duration_ms": duration_ms
}))
```

---

## 🎯 Summary & Success Criteria

### Definizione di "Done"

L'implementazione è completa quando:

1. ✅ **Backend configurabile**: Parametri service_tier, timeout, max_retries funzionanti
2. ✅ **Prompt caching monitorato**: Log mostrano cache hits e savings
3. ✅ **Reasoning models gestiti**: o1/o3 inizializzati senza temperature
4. ✅ **Documentazione aggiornata**: CLAUDE.md e example.env completi
5. ✅ **Test superati**: Tutti i test in Phase 5 passano con successo
6. ✅ **Frontend sincronizzato**: Tutti i modelli backend sono visibili nel UI

### Expected Performance Improvements

- **Latency**: 40% riduzione con priority tier
- **Cost**: 30-50% riduzione con flex tier + prompt caching
- **Reliability**: Meno timeout con configurazione timeout appropriata
- **Visibility**: Log chiari su cache usage e cost savings

---

## 📚 References

- [OpenAI Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching)
- [OpenAI Priority Processing](https://platform.openai.com/docs/guides/priority-processing)
- [LangChain ChatOpenAI API](https://python.langchain.com/docs/integrations/chat/openai)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference/chat)

---

## 🔄 Rollback Plan

Se l'implementazione causa problemi:

1. **Rimuovere le nuove env variables** da `.env`
2. **Revert changes** in `llm.py` usando git:
   ```bash
   git checkout HEAD -- backend/src/llm.py
   ```
3. **Restart backend** per applicare old configuration
4. **Verificare** che i modelli esistenti funzionino come prima

Il rollback è safe perché:
- Tutti i nuovi parametri hanno valori default
- Le modifiche sono backward compatible
- I modelli esistenti continuano a funzionare senza config aggiuntiva

---

**Created:** 2025-01-04
**Status:** Ready for implementation
**Estimated Time:** 3-4 hours (con testing completo)
