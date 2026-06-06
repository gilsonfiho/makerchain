# CLAUDE.md — Guia de Desenvolvimento do Makerchain

## O que é este projeto

Makerchain é um sistema RAG (Retrieval-Augmented Generation) local para projetos maker. Indexa datasheets de hardware (ESP32, etc.) em um banco vetorial FAISS e responde perguntas técnicas em português usando modelos Ollama rodando localmente.

## Stack

- **Backend**: Python 3.11, FastAPI, Uvicorn
- **AI/LLM**: LangChain, LangChain-Ollama (modelo Mistral por padrão)
- **Vector DB**: FAISS (local, arquivo em disco)
- **Frontend**: Jinja2 + Bootstrap 5 (HTML/JS server-side rendering)
- **Container**: Docker + Docker Compose

## Estrutura do projeto

```
src/
  main.py          # App FastAPI — rotas e middleware de sessão
  qa_enginer.py    # QAEngine (carrega FAISS+LLM) e QAHandler (pipeline RAG)
  ingest.py        # Script standalone para vetorizar documentos
  templates/
    index.html     # UI única (Bootstrap 5, vanilla JS)
data/              # PDFs e outros documentos a indexar
vectorstore/
  db_faiss/        # Índice FAISS gerado pelo ingest.py
logs/
  prompts_log.txt  # Log plano de todas as interações
```

## Pré-requisitos

1. Ollama instalado e rodando com o modelo Mistral:
   ```bash
   ollama pull mistral
   ollama serve
   ```
2. Python 3.11+
3. (Opcional) Docker para containerização

## Configuração e execução local

```bash
# Instalar dependências
pip install -r requirements.txt

# Vetorizar documentos da pasta data/
python src/ingest.py

# Subir o servidor
python -m uvicorn src.main:app --reload --host 0.0.0.0 --port 8000
```

Acesse em `http://localhost:8000`.

## Execução com Docker

```bash
# Garanta que o Ollama está rodando no host antes de subir
docker compose up --build
```

A variável `OLLAMA_HOST` precisa apontar para onde o Ollama está acessível. Em Docker Desktop (Mac/Windows) use `http://host.docker.internal:11434`; em Linux nativo use o IP do host (`http://172.17.0.1:11434`).

## Variáveis de ambiente

| Variável | Padrão | Descrição |
|----------|--------|-----------|
| `OLLAMA_HOST` | `http://host.docker.internal:11434` | URL base do servidor Ollama |

## Pontos críticos de atenção

- **`ingest.py`** usa `DATA_DIR` hardcoded com caminho Windows — altere antes de rodar. O caminho correto é `data/` relativo à raiz do projeto.
- **Modelo de embeddings**: `ingest.py` usa `llama3`, mas `qa_enginer.py` usa `mistral`. O modelo de embeddings usado na ingestão **deve ser idêntico** ao usado na consulta, ou a busca vetorial retorna lixo.
- **Secret key de sessão** em `main.py` está com valor placeholder — defina via variável de ambiente em produção.
- **`QAHandler.history`** é um atributo de instância compartilhado entre todas as sessões (estado global no processo). Isso mistura histórico de usuários distintos.
- **`allow_dangerous_deserialization=True`** no FAISS só é seguro quando o arquivo `.pkl` é gerado pelo próprio projeto e não vem de fonte externa.
- **`qa_enginer.py`** — o nome tem um typo (deveria ser `qa_engine.py`). Corrigir exige atualizar o import em `main.py`.

## Fluxo RAG (consulta)

```
Usuário digita pergunta
        ↓
POST / (main.py)
        ↓
QAHandler.ask(pergunta)
  - monta contexto com últimas 3 trocas do histórico in-memory
  - chama RetrievalQA.run(prompt_com_contexto)
        ↓
FAISS retriever
  - busca os k chunks mais similares ao embedding da pergunta
        ↓
StuffDocumentsChain + LLMChain
  - concatena chunks como contexto
  - envia prompt ao ChatOllama (Mistral)
        ↓
Resposta → sessão → HTML
```

## Fluxo de ingestão

```
data/ (PDFs, .py, .md, .txt)
        ↓
load_documents() — PyPDFLoader / PythonLoader / TextLoader
        ↓
split_documents() — RecursiveCharacterTextSplitter ou SemanticChunker
        ↓
OllamaEmbeddings.embed_documents()
        ↓
FAISS.from_documents() → salva em vectorstore/db_faiss/
```

## Tarefas comuns

### Adicionar novos documentos
1. Coloque os arquivos em `data/`
2. Rode `python src/ingest.py` — isso **recria** o índice do zero

### Trocar o modelo LLM
Em `qa_enginer.py`, altere `model="mistral"` na instância de `ChatOllama` e `OllamaEmbeddings`. Garanta que o mesmo modelo foi usado no `ingest.py`.

### Ver histórico de interações
```bash
cat logs/prompts_log.txt
```

## Não existe ainda

- Testes automatizados (pytest)
- Validação de variáveis de ambiente na inicialização
- Upload de documentos pela UI
- Autenticação
- Health check endpoint
- CI/CD pipeline
