# Guida alla Risoluzione dei Problemi di Configurazione Docker

Questa guida documenta i problemi riscontrati durante la configurazione iniziale dell'applicazione LLM Graph Builder con Docker Compose e le soluzioni implementate per risolverli.

## Sintomo Iniziale

Dopo aver eseguito `docker compose up`, l'applicazione restituiva il seguente errore:

```
AuthenticationError: Error code: 401 - {'error': {'message': 'Incorrect API key provided: openai_a**_key. You can find your API key at https://platform.openai.com/account/api-keys.', 'type': 'invalid_request_error', 'param': None, 'code': 'invalid_api_key'}}
```

La chiave API di OpenAI era stata correttamente configurata nel file `backend/.env`, ma l'applicazione non riusciva a leggerla.

---

## Problema 1: Sintassi Errata nel File `.env`

### Causa

Il file `backend/.env` conteneva **spazi attorno al segno uguale** nella definizione delle variabili:

```bash
# ❌ ERRATO
OPENAI_API_KEY = sk-proj-...
EMBEDDING_MODEL = "all-MiniLM-L6-v2"
NEO4J_URI = "neo4j+s://..."
```

I file `.env` richiedono la sintassi **senza spazi** e **senza virgolette** (a meno che non facciano parte del valore effettivo):

```bash
# ✅ CORRETTO
OPENAI_API_KEY=sk-proj-...
EMBEDDING_MODEL=all-MiniLM-L6-v2
NEO4J_URI=neo4j+s://...
```

### Soluzione

Rimuovere tutti gli spazi attorno al segno `=` nel file `backend/.env`:

**Prima:**
```bash
OPENAI_API_KEY = sk-proj-abc123
EMBEDDING_MODEL = "all-MiniLM-L6-v2"
```

**Dopo:**
```bash
OPENAI_API_KEY=sk-proj-abc123
EMBEDDING_MODEL=all-MiniLM-L6-v2
```

### Perché Funziona

I parser di file `.env` (sia in Docker Compose che nelle librerie Python come `python-dotenv`) interpretano:
- `KEY=value` → variabile `KEY` con valore `value`
- `KEY = value` → variabile `KEY ` (con spazio finale) con valore ` value` (con spazio iniziale)

Gli spazi vengono inclusi letteralmente nei nomi e valori delle variabili, causando mismatch.

---

## Problema 2: Docker Compose Non Legge `backend/.env`

### Causa

Il servizio `backend` nel file `docker-compose.yml` originale **non aveva** la direttiva `env_file`:

```yaml
# docker-compose.yml - Configurazione ORIGINALE
services:
  backend:
    build:
      context: ./backend
    volumes:
      - ./backend:/code
    # ❌ Manca env_file!
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY-}
      - NEO4J_URI=${NEO4J_URI-neo4j://database:7687}
```

Docker Compose **non legge automaticamente** i file `.env` nelle sottodirectory. Il file `backend/.env` viene ignorato.

### Come Funziona Docker Compose con i File `.env`

Docker Compose ha **due meccanismi distinti** per gestire le variabili d'ambiente:

1. **File `.env` nella root** (stesso livello di `docker-compose.yml`):
   - Letto **automaticamente** da Docker Compose
   - Usato per **sostituire** le variabili `${VAR}` nel `docker-compose.yml`
   - **NON** passato direttamente ai container

2. **Direttiva `env_file:`** nei servizi:
   - Specifica file `.env` da **caricare dentro il container**
   - Può essere in qualsiasi percorso
   - Deve essere dichiarato esplicitamente

### Soluzione

Aggiungere la direttiva `env_file` al servizio backend:

```yaml
services:
  backend:
    build:
      context: ./backend
    volumes:
      - ./backend:/code
    env_file:              # ✅ AGGIUNTO
      - ./backend/.env     # ✅ AGGIUNTO
    container_name: backend
```

### Perché Funziona

Con `env_file: - ./backend/.env`, Docker Compose:
1. Legge tutte le variabili da `backend/.env`
2. Le passa come variabili d'ambiente al container backend
3. L'applicazione Python può accedervi tramite `os.environ.get()`

---

## Problema 3: Conflitto tra `env_file` e `environment`

### Causa

Anche dopo aver aggiunto `env_file`, l'errore persisteva. Il problema era la **sezione `environment:`** che **sovrascriveva** i valori caricati da `env_file`:

```yaml
services:
  backend:
    env_file:
      - ./backend/.env     # Carica OPENAI_API_KEY=sk-proj-...
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY-}  # ❌ SOVRASCRIVE con ""!
```

### Come Funziona l'Ordine di Priorità

Docker Compose applica le variabili in questo ordine (dal più basso al più alto):

1. `env_file:` (priorità bassa)
2. `environment:` (priorità alta) ← **SOVRASCRIVE env_file**

### Come Funziona `${VARIABILE-default}`

La sintassi `${OPENAI_API_KEY-}` cerca la variabile in quest'ordine:

1. **Variabili d'ambiente del sistema host** (Windows/Linux dove esegui `docker compose up`)
2. **File `.env` nella root del progetto** (stesso livello di `docker-compose.yml`)
3. Se non trova niente, usa il valore dopo il `-` (in questo caso stringa vuota `""`)

**IMPORTANTE:** NON cerca in `backend/.env` perché quello è usato solo da `env_file:`.

### Flusso del Problema

```
1. env_file carica:     OPENAI_API_KEY="sk-proj-abc123" ✅
2. Docker Compose processa environment: ${OPENAI_API_KEY-}
   - Cerca nell'host Windows → NON trovata
   - Cerca in .env root → NON esiste
   - Usa default → "" (stringa vuota)
3. environment setta:   OPENAI_API_KEY="" ❌ (sovrascrive!)
4. Container riceve:    OPENAI_API_KEY="" ❌
```

### Soluzione

Rimuovere completamente la sezione `environment:` dal servizio backend:

```yaml
services:
  backend:
    env_file:
      - ./backend/.env     # ✅ Unica fonte delle variabili
    # ❌ RIMOSSO: environment: ...
    container_name: backend
```

### Verifica delle Variabili Mancanti

Prima di rimuovere `environment:`, abbiamo verificato che tutte le variabili fossero presenti in `backend/.env`:

**Variabili nella sezione `environment:` originale:**
- NEO4J_URI, NEO4J_USERNAME, NEO4J_PASSWORD
- OPENAI_API_KEY
- DIFFBOT_API_KEY
- EMBEDDING_MODEL
- LANGCHAIN_* (API_KEY, PROJECT, ENDPOINT, TRACING_V2)
- KNN_MIN_SCORE, IS_EMBEDDING
- GEMINI_ENABLED, GCP_LOG_METRICS_ENABLED
- UPDATE_GRAPH_CHUNKS_PROCESSED, NUMBER_OF_CHUNKS_TO_COMBINE
- ENTITY_EMBEDDING, GCS_FILE_CACHE
- LLM_MODEL_CONFIG_ollama_llama3

**Variabile mancante trovata:** `DIFFBOT_API_KEY`

Abbiamo aggiunto al `backend/.env`:
```bash
DIFFBOT_API_KEY=
```

### Perché Funziona

Con solo `env_file:` e senza `environment:`:
- Tutte le variabili vengono caricate da `backend/.env`
- Nessun override o conflitto
- Configurazione centralizzata in un unico file

---

## Problema 4: Placeholder nelle Configurazioni LLM (Problema Principale)

### Causa Reale dell'Errore 401

Analizzando i log del container, abbiamo scoperto il problema **reale**:

```
2025-11-03 18:08:20,338 - Model: LLM_MODEL_CONFIG_openai_gpt_4o
2025-11-03 18:08:24,532 - HTTP Request: POST https://api.openai.com/v1/chat/completions "HTTP/1.1 401 Unauthorized"
2025-11-03 18:08:24,533 - Error: 'Incorrect API key provided: openai_a**_key'
```

L'applicazione usava `LLM_MODEL_CONFIG_openai_gpt_4o`, che nel file `backend/.env` era configurato come:

```bash
LLM_MODEL_CONFIG_openai_gpt_4o="gpt-4o-2024-11-20,openai_api_key"
```

Il valore `openai_api_key` è un **placeholder/esempio**, non un riferimento alla variabile `OPENAI_API_KEY`.

### Perché Esistono Due Variabili Diverse?

L'applicazione usa OpenAI per **due scopi diversi**:

#### 1. `OPENAI_API_KEY` → Embeddings (Vector Search)

Nel file `src/shared/common_fn.py`:

```python
def load_embedding_model(embedding_model_name: str):
    if embedding_model_name == "openai":
        embeddings = OpenAIEmbeddings()  # ← Cerca automaticamente OPENAI_API_KEY
```

La classe `OpenAIEmbeddings()` di LangChain **cerca automaticamente** la variabile `OPENAI_API_KEY` nell'ambiente.

**Verifica nei log:** Gli embeddings funzionavano correttamente:
```
2025-11-03 18:08:04,651 - Embedding: Using OpenAI Embeddings, Dimension:1536 ✅
```

#### 2. `LLM_MODEL_CONFIG_openai_gpt_4o` → Chat Models (Q&A)

Nel file `src/llm.py`:

```python
def get_llm(model: str):
    env_key = f"LLM_MODEL_CONFIG_{model}"
    env_value = os.environ.get(env_key)

    if "openai" in model:
        model_name, api_key = env_value.split(",")  # ← Split manuale
        llm = ChatOpenAI(
            api_key=api_key,  # ← Usa direttamente il valore dopo la virgola
            model=model_name
        )
```

Il codice:
1. Legge `LLM_MODEL_CONFIG_openai_gpt_4o`
2. Splitta sulla virgola: `["gpt-4o-2024-11-20", "openai_api_key"]`
3. Usa `"openai_api_key"` (stringa letterale) come chiave API ❌

### Perché i File `.env` Non Supportano Riferimenti a Variabili

Un file `.env` è una **lista di coppie chiave=valore**, non uno script che interpreta variabili.

Quando scrivi:
```bash
OPENAI_API_KEY=sk-proj-abc123
LLM_MODEL_CONFIG_openai_gpt_4o="gpt-4o,openai_api_key"
```

Il valore di `LLM_MODEL_CONFIG_openai_gpt_4o` è **letteralmente**:
```
"gpt-4o,openai_api_key"
```

La stringa `openai_api_key` **NON** viene sostituita con il valore di `OPENAI_API_KEY`. È solo testo.

### Soluzione

Sostituire i placeholder `openai_api_key` con la **chiave API reale** in tutte le configurazioni `LLM_MODEL_CONFIG_openai_*`:

**Prima:**
```bash
LLM_MODEL_CONFIG_openai_gpt_4o="gpt-4o-2024-11-20,openai_api_key"
LLM_MODEL_CONFIG_openai_gpt_4o_mini="gpt-4o-mini-2024-07-18,openai_api_key"
```

**Dopo:**
```bash
LLM_MODEL_CONFIG_openai_gpt_4o="gpt-4o-2024-11-20,sk-proj-TUA_CHIAVE_REALE"
LLM_MODEL_CONFIG_openai_gpt_4o_mini="gpt-4o-mini-2024-07-18,sk-proj-TUA_CHIAVE_REALE"
```

### Configurazioni da Aggiornare

Sostituire il placeholder in **tutte** le configurazioni OpenAI:

```bash
LLM_MODEL_CONFIG_openai_gpt_3.5="gpt-3.5-turbo-0125,sk-proj-..."
LLM_MODEL_CONFIG_openai_gpt_4o_mini="gpt-4o-mini-2024-07-18,sk-proj-..."
LLM_MODEL_CONFIG_openai_gpt_4o="gpt-4o-2024-11-20,sk-proj-..."
LLM_MODEL_CONFIG_openai_gpt_4.1_mini="gpt-4.1-mini,sk-proj-..."
LLM_MODEL_CONFIG_openai_gpt_4.1="gpt-4.1,sk-proj-..."
LLM_MODEL_CONFIG_openai_gpt_o3_mini="o3-mini-2025-01-31,sk-proj-..."
```

### Perché Funziona

Il codice in `src/llm.py` ora riceve:
```python
env_value = "gpt-4o-2024-11-20,sk-proj-abc123"
model_name, api_key = env_value.split(",")
# model_name = "gpt-4o-2024-11-20"
# api_key = "sk-proj-abc123"  ✅ Chiave reale!

llm = ChatOpenAI(api_key=api_key)  ✅ Funziona!
```

---

## Riepilogo delle Modifiche Finali

### 1. File `backend/.env`

```bash
# Sintassi corretta (niente spazi, niente virgolette inutili)
OPENAI_API_KEY=sk-proj-TUA_CHIAVE_REALE
DIFFBOT_API_KEY=
EMBEDDING_MODEL=all-MiniLM-L6-v2
RAGAS_EMBEDDING_MODEL=openai
IS_EMBEDDING=TRUE
KNN_MIN_SCORE=0.94
GEMINI_ENABLED=False
GCP_LOG_METRICS_ENABLED=False
NUMBER_OF_CHUNKS_TO_COMBINE=6
UPDATE_GRAPH_CHUNKS_PROCESSED=20

# Neo4j
NEO4J_URI=neo4j+s://TUO_URI
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=TUA_PASSWORD
NEO4J_DATABASE=neo4j

# LLM Model Configs - SOSTITUIRE openai_api_key con chiave reale
LLM_MODEL_CONFIG_openai_gpt_3.5="gpt-3.5-turbo-0125,sk-proj-TUA_CHIAVE_REALE"
LLM_MODEL_CONFIG_openai_gpt_4o_mini="gpt-4o-mini-2024-07-18,sk-proj-TUA_CHIAVE_REALE"
LLM_MODEL_CONFIG_openai_gpt_4o="gpt-4o-2024-11-20,sk-proj-TUA_CHIAVE_REALE"

# ... altre configurazioni ...
```

### 2. File `docker-compose.yml`

```yaml
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    volumes:
      - ./backend:/code
    env_file:              # ✅ AGGIUNTO
      - ./backend/.env
    # ❌ RIMOSSA sezione environment:
    container_name: backend
    extra_hosts:
      - host.docker.internal:host-gateway
    ports:
      - "8000:8000"
    networks:
      - net
```

### 3. File `.env` nella root (opzionale, come backup)

Creato un file `.env` nella root del progetto (stesso livello di `docker-compose.yml`) con le variabili principali, utile come backup e per eventuali sostituzioni `${VAR}` future nel docker-compose.yml.

---

## Checklist per la Configurazione

Segui questi step per configurare correttamente l'applicazione:

### Step 1: Verifica `backend/.env`

- [ ] Nessuno spazio attorno al segno `=`
- [ ] Nessuna virgoletta inutile (a meno che non facciano parte del valore)
- [ ] `OPENAI_API_KEY` impostata con la chiave reale
- [ ] `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD` configurati

### Step 2: Aggiorna Configurazioni LLM

- [ ] Sostituire `openai_api_key` con la chiave reale in **tutte** le configurazioni `LLM_MODEL_CONFIG_openai_*`
- [ ] Verificare che non ci siano altri placeholder (es. `azure_api_key`, `anthropic_api_key`) se usi altri provider

### Step 3: Aggiorna `docker-compose.yml`

- [ ] Aggiunto `env_file: - ./backend/.env` al servizio backend
- [ ] Rimossa la sezione `environment:` dal servizio backend (o commentata)

### Step 4: Build e Avvio

```bash
# Ferma eventuali container in esecuzione
docker compose down

# Ricostruisci e avvia
docker compose up --build
```

### Step 5: Verifica

Controlla i log per confermare che:
- [ ] Nessun errore 401 da OpenAI
- [ ] Backend avviato correttamente
- [ ] Connessione a Neo4j stabilita
- [ ] Frontend accessibile su http://localhost:8080

---

## Verifica della Configurazione nel Container

Per verificare che le variabili siano caricate correttamente nel container backend:

```bash
# Controlla la chiave OpenAI
docker exec backend printenv | grep OPENAI_API_KEY

# Output atteso:
# OPENAI_API_KEY=sk-proj-...  (chiave completa, non mascherata)
```

Se vedi `OPENAI_API_KEY=` o `OPENAI_API_KEY=openai_api_key`, le configurazioni non sono state applicate correttamente.

---

## Domande Frequenti

### Q: Perché non posso usare `${OPENAI_API_KEY}` in `LLM_MODEL_CONFIG_openai_gpt_4o`?

I file `.env` non supportano la sostituzione di variabili. La stringa viene interpretata letteralmente.

### Q: Devo duplicare la chiave API in ogni `LLM_MODEL_CONFIG_openai_*`?

Sì, purtroppo il design dell'applicazione richiede che ogni configurazione LLM contenga la propria chiave API. Questo permette di usare chiavi diverse per modelli diversi, se necessario.

### Q: Posso lasciare la sezione `environment:` nel docker-compose.yml?

No, se hai `env_file:` e `environment:` insieme, `environment:` sovrascriverà i valori di `env_file:`. Rimuovila o commentala.

### Q: Cosa succede se aggiungo variabili d'ambiente al sistema Windows?

Docker Compose le userà per sostituire le variabili `${VAR}` nel `docker-compose.yml`, ma dovrai comunque avere `env_file:` per caricarle nel container.

### Q: Posso usare il file `.env` nella root invece di `backend/.env`?

Tecnicamente sì, ma dovrai aggiornare `env_file: - ./.env` nel docker-compose.yml. Tuttavia, è preferibile mantenere le configurazioni del backend separate da quelle del frontend.

---

## Note Finali

Questa configurazione è stata testata con:
- Docker Compose v3
- Python 3.10
- LangChain
- Neo4j Aura (cloud)
- Windows 10/11

Se incontri ancora problemi dopo aver seguito questa guida, verifica:
1. Che la chiave OpenAI sia valida e attiva su https://platform.openai.com
2. Che il billing sia attivo sul tuo account OpenAI
3. Che Neo4j sia accessibile dall'URL configurato
4. I log completi del container: `docker logs backend`

---

**Autori:** Documentazione creata durante il processo di debug e configurazione
**Data:** 2025-11-04
**Versione:** 1.0
