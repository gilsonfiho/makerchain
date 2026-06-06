# Plano de Melhoria — Makerchain

## Prioridade 1 — Correções críticas (bugs e segurança)

Estas são falhas que afetam corretude ou segurança hoje. Devem ser feitas antes de qualquer nova feature.

### 1.1 Corrigir caminho hardcoded no `ingest.py`

**Problema**: `DATA_DIR` está com caminho absoluto Windows (`C:\Users\gilso\...`). O script falha em qualquer outro ambiente.

**Solução**:
```python
# ingest.py
import pathlib

BASE_DIR = pathlib.Path(__file__).parent.parent
DATA_DIR = str(BASE_DIR / "data")
```

### 1.2 Unificar o modelo de embeddings

**Problema**: `ingest.py` usa `OllamaEmbeddings(model="llama3")` e `qa_enginer.py` usa `OllamaEmbeddings(model="mistral")`. Embeddings gerados com modelos diferentes são incompatíveis — a busca vetorial retorna resultados aleatórios.

**Solução**: Centralizar o nome do modelo em uma variável de ambiente (ou constante compartilhada) e garantir que ambos os scripts usem o mesmo valor.

```python
# config.py (novo arquivo)
import os

OLLAMA_BASE_URL = os.getenv("OLLAMA_HOST", "http://host.docker.internal:11434")
EMBED_MODEL = os.getenv("EMBED_MODEL", "mistral")
LLM_MODEL = os.getenv("LLM_MODEL", "mistral")
DB_FAISS_PATH = os.getenv("DB_FAISS_PATH", "vectorstore/db_faiss")
```

### 1.3 Tirar a secret key hardcoded

**Problema**: `app.add_middleware(SessionMiddleware, secret_key="sua_chave_secreta_aqui")` expõe sessões a ataques de falsificação.

**Solução**:
```python
import os, secrets

secret = os.getenv("SESSION_SECRET_KEY") or secrets.token_hex(32)
app.add_middleware(SessionMiddleware, secret_key=secret)
```
Adicionar `SESSION_SECRET_KEY` ao `.env.example` e ao `docker-compose.yml`.

### 1.4 Adicionar `OLLAMA_HOST` ao `docker-compose.yml`

**Problema**: A variável está definida no código mas ausente no compose, então containers Docker sempre usam o default.

```yaml
# docker-compose.yml
services:
  makerchain:
    environment:
      - OLLAMA_HOST=${OLLAMA_HOST:-http://host.docker.internal:11434}
      - SESSION_SECRET_KEY=${SESSION_SECRET_KEY}
      - LLM_MODEL=${LLM_MODEL:-mistral}
      - EMBED_MODEL=${EMBED_MODEL:-mistral}
```

### 1.5 Isolar histórico conversacional por sessão

**Problema**: `QAHandler.history` é um atributo da instância global. Requisições de usuários diferentes compartilham o mesmo histórico, contaminando o contexto.

**Solução**: Mover o histórico para a sessão HTTP (já existe na sessão para o histórico de display). O `QAHandler.ask()` deve receber o histórico como parâmetro e retornar o histórico atualizado, sem armazenar estado interno.

---

## Prioridade 2 — Qualidade e manutenibilidade

### 2.1 Renomear `qa_enginer.py` para `qa_engine.py`

Typo simples que gera estranhamento. Requer atualizar o import em `main.py`.

### 2.2 Migrar para a API moderna do LangChain (LCEL)

`RetrievalQA`, `LLMChain` e `StuffDocumentsChain` estão depreciados desde LangChain 0.2. Migrar para LangChain Expression Language (LCEL):

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | pt_prompt
    | llm
    | StrOutputParser()
)
```

### 2.3 Criar suite de testes com pytest

Estrutura sugerida:
```
tests/
  test_ingest.py      # testa load_documents() e split_documents()
  test_qa_engine.py   # testa QAEngine com FAISS mockado
  test_routes.py      # testa endpoints FastAPI com TestClient
  conftest.py         # fixtures compartilhadas (FAISS mock, LLM mock)
```

Usar `unittest.mock` para mockar chamadas ao Ollama, permitindo testes sem o daemon rodando.

### 2.4 Adicionar validação na inicialização

Validar variáveis obrigatórias e disponibilidade do Ollama na subida do servidor:

```python
@app.on_event("startup")
async def startup_check():
    if not pathlib.Path(DB_FAISS_PATH).exists():
        raise RuntimeError(f"Índice FAISS não encontrado em {DB_FAISS_PATH}. Execute ingest.py primeiro.")
```

### 2.5 Adicionar health check endpoint

```python
@app.get("/health")
async def health():
    return {"status": "ok", "faiss_loaded": qa_engine.db is not None}
```

Essencial para Docker HEALTHCHECK e orquestradores como Kubernetes.

### 2.6 Substituir logging manual por `logging` padrão

O log atual grava texto plano sem timestamp, severidade ou estrutura. Substituir por `logging` do Python com formatação JSON facilita ingestão por Loki, Datadog, etc.

---

## Prioridade 3 — Novas funcionalidades

### 3.1 Upload de documentos pela interface web

Adicionar endpoint `POST /upload` que:
1. Recebe arquivo via `multipart/form-data`
2. Salva em `data/`
3. Aciona reindexação incremental (adiciona ao FAISS existente sem recriar do zero)

```python
@app.post("/upload")
async def upload_document(file: UploadFile = File(...)):
    dest = Path("data") / file.filename
    dest.write_bytes(await file.read())
    # reindexar de forma assíncrona (background task)
    background_tasks.add_task(reindex_document, dest)
    return {"message": f"{file.filename} enviado e será indexado em breve"}
```

### 3.2 Suporte a múltiplos modelos Ollama (selecionável pelo usuário)

Adicionar um seletor na UI para o usuário escolher o modelo (entre os disponíveis no Ollama local). Endpoint `GET /models` consulta `ollama list` e retorna a lista.

### 3.3 Reindexação incremental

Atualmente `ingest.py` recria o índice completo. Implementar adição incremental:
```python
existing_db = FAISS.load_local(DB_FAISS_PATH, embeddings)
existing_db.add_documents(new_split_docs)
existing_db.save_local(DB_FAISS_PATH)
```

### 3.4 Streaming de respostas via SSE

Respostas longas (20+ segundos) travam a UI. Usar Server-Sent Events com `ChatOllama` em modo streaming:

```python
from fastapi.responses import StreamingResponse

@app.post("/stream")
async def stream_answer(pergunta: str = Form(...)):
    async def generate():
        async for chunk in chain.astream(pergunta):
            yield f"data: {chunk}\n\n"
    return StreamingResponse(generate(), media_type="text/event-stream")
```

### 3.5 Suporte a `.env` com validação via Pydantic Settings

```python
# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    ollama_host: str = "http://localhost:11434"
    llm_model: str = "mistral"
    embed_model: str = "mistral"
    db_faiss_path: str = "vectorstore/db_faiss"
    session_secret_key: str

    class Config:
        env_file = ".env"
```

---

## Prioridade 4 — Infraestrutura e DevOps

### 4.1 Adicionar GitHub Actions CI

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.11"}
      - run: pip install -r requirements.txt pytest
      - run: pytest tests/ -v
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ruff
      - run: ruff check src/
```

### 4.2 Adicionar HEALTHCHECK ao Dockerfile e docker-compose

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s \
  CMD curl -f http://localhost:8000/health || exit 1
```

### 4.3 Multi-stage Dockerfile para imagem menor

```dockerfile
FROM python:3.11-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS runtime
COPY src/ src/
COPY vectorstore/ vectorstore/
EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 4.4 Criar `.env.example`

Documentar todas as variáveis esperadas para facilitar onboarding de novos contribuidores.

---

## Roadmap resumido

| Sprint | Itens | Valor entregue |
|--------|-------|---------------|
| 1 (1 semana) | 1.1 + 1.2 + 1.3 + 1.4 | Projeto funciona em qualquer OS, Docker estável, sem bugs silenciosos |
| 2 (1 semana) | 1.5 + 2.1 + 2.3 + 2.5 | Isolamento de sessão correto, testes básicos, projeto seguro |
| 3 (2 semanas) | 2.2 + 2.4 + 2.6 + 4.1 | Base de código moderna, CI rodando, observabilidade |
| 4 (2 semanas) | 3.1 + 3.3 + 3.5 | Upload de docs pela UI, reindexação sem downtime |
| 5 (2 semanas) | 3.2 + 3.4 + 4.2 + 4.3 | Streaming, multi-modelo, Docker production-ready |
