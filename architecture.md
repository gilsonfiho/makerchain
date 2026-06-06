# Arquitetura do Makerchain

## Visão geral

Makerchain é um sistema RAG (Retrieval-Augmented Generation) que conecta documentos técnicos de hardware a um LLM local, respondendo perguntas em português sem depender de APIs externas.

```
┌─────────────────────────────────────────────────────────────┐
│                        Usuário (Browser)                    │
└─────────────────────────┬───────────────────────────────────┘
                          │ HTTP (porta 8000)
┌─────────────────────────▼───────────────────────────────────┐
│                    FastAPI (main.py)                        │
│  GET /          → renderiza index.html com histórico        │
│  POST /         → recebe `pergunta`, retorna resposta       │
│  POST /clear    → limpa sessão                              │
│                                                             │
│  SessionMiddleware (Starlette) — histórico por cookie       │
│  Jinja2Templates — SSR, arquivo único index.html           │
└─────────────────────────┬───────────────────────────────────┘
                          │ método .ask()
┌─────────────────────────▼───────────────────────────────────┐
│                   QAHandler (qa_enginer.py)                 │
│  - Mantém histórico in-memory (últimas 10 trocas)           │
│  - Monta prompt com contexto das últimas 3 trocas           │
│  - Grava log em logs/prompts_log.txt                        │
│  - Delega ao QAEngine.qa (RetrievalQA chain)                │
└─────────────────────────┬───────────────────────────────────┘
                          │
          ┌───────────────┴──────────────┐
          │                              │
┌─────────▼──────────┐      ┌────────────▼────────────────────┐
│  FAISS Retriever   │      │  LangChain RetrievalQA Chain     │
│                    │      │                                   │
│  vectorstore/      │      │  StuffDocumentsChain             │
│  db_faiss/         │      │    + LLMChain                    │
│  ├── index.faiss   │      │    + PromptTemplate (PT-BR)      │
│  └── index.pkl     │      └────────────┬────────────────────┘
│                    │                   │
│  OllamaEmbeddings  │                   │ HTTP (porta 11434)
│  (model: mistral)  │      ┌────────────▼────────────────────┐
└────────────────────┘      │       Ollama (processo local)    │
                            │       modelo: mistral            │
                            └─────────────────────────────────┘
```

## Componentes

### `src/main.py` — Camada Web

Ponto de entrada da aplicação. Responsável exclusivamente por:
- Receber requisições HTTP
- Gerenciar o histórico no cookie de sessão (máx. 10 itens)
- Renderizar o template HTML com o resultado

Não contém lógica de negócio de IA.

### `src/qa_enginer.py` — Camada de IA

Dividida em duas classes:

**`QAEngine`** — inicialização stateless do pipeline RAG:
- Carrega o índice FAISS do disco
- Instancia `ChatOllama` e `OllamaEmbeddings`
- Constrói a chain: `LLMChain` → `StuffDocumentsChain` → `RetrievalQA`
- Prompt fixo em português: `Contexto: {context}\nPergunta: {question}`

**`QAHandler`** — estado conversacional e logging:
- Mantém `self.history` em memória (lista de dicts `{pergunta, resposta}`)
- Injeta as 3 últimas trocas no prompt antes de cada query
- Grava cada interação em `logs/prompts_log.txt`

> Atenção: `QAHandler` é instanciado uma vez globalmente em `main.py`. Seu `self.history` é compartilhado entre todas as sessões de usuário simultâneas.

### `src/ingest.py` — Pipeline de Ingestão (offline)

Script standalone executado manualmente para recriar o índice vetorial:

```
Arquivos em data/  →  Loaders  →  Splitter  →  Embeddings  →  FAISS (disco)
```

Suporta dois modos de chunking (configurado via `SPLITTER_TYPE`):
- `recursive`: `RecursiveCharacterTextSplitter` (chunk_size=500, overlap=100)
- `semantic`: `SemanticChunker` (divide por similaridade semântica entre sentenças)

### `src/templates/index.html` — Frontend

Single-page app server-side renderizado. Funcionalidades via vanilla JS:
- Envio do formulário com exibição de spinner
- Download da resposta como arquivo `.txt`
- Exibição do histórico de sessão

## Fluxo de dados — consulta

```
1. Usuário submete pergunta via POST /
2. main.py chama qa_handler.ask(pergunta)
3. QAHandler monta prompt: histórico (últimas 3) + pergunta atual
4. RetrievalQA.run(prompt) é chamado
5. FAISS retriever embeds o prompt e busca k chunks mais próximos
6. StuffDocumentsChain concatena os chunks como {context}
7. LLMChain envia ao ChatOllama → Mistral gera resposta
8. Resposta retorna pelo stack até o template HTML
9. main.py salva {pergunta, resposta} na sessão (cookie)
10. QAHandler salva no histórico in-memory e no arquivo de log
```

## Fluxo de dados — ingestão

```
1. python src/ingest.py
2. load_documents() percorre data/ recursivamente
3. Cada arquivo gera uma lista de Document objects
4. split_documents() fragmenta em chunks menores
5. OllamaEmbeddings.embed_documents() gera vetores para cada chunk
6. FAISS.from_documents() constrói o índice
7. vectorstore.save_local() persiste index.faiss + index.pkl
```

## Decisões de design

| Decisão | Justificativa |
|---------|---------------|
| FAISS local (sem servidor) | Zero infraestrutura extra; adequado para corpus pequeno (<100k chunks) |
| Ollama local | Privacidade total; sem custo por token; funciona offline |
| SSR com Jinja2 | Simples; sem build frontend; sem dependência de npm |
| Histórico em sessão (cookie) | Stateless no servidor; escala horizontalmente sem banco |
| Logging em arquivo texto | Auditoria simples; sem dependência de banco de dados |

## Limitações atuais

| Limitação | Impacto |
|-----------|---------|
| `QAHandler.history` global | Mistura contexto conversacional de usuários distintos |
| `DATA_DIR` hardcoded (Windows) em `ingest.py` | Ingestão falha em ambientes não-Windows sem edição manual |
| Modelos diferentes entre ingestão (`llama3`) e consulta (`mistral`) | Embeddings incompatíveis degradam silenciosamente a qualidade da busca |
| Secret key hardcoded em `main.py` | Inseguro em produção |
| `OLLAMA_HOST` ausente no `docker-compose.yml` | Container Docker falha em conectar ao Ollama sem configuração manual |
| `RetrievalQA` e `LLMChain` depreciados no LangChain | Ficará incompatível com versões futuras |
| Sem isolamento de histórico por usuário | Um usuário contamina o contexto de outro |
| Sem testes | Regressões silenciosas |

## Dependências externas em tempo de execução

```
Aplicação FastAPI (porta 8000)
    └── Ollama daemon (porta 11434)
            └── Modelo mistral (baixado previamente)
```

O FAISS é carregado do disco na inicialização; após isso, consultas a ele são locais e não requerem rede.
