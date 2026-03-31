# Modular RAG MCP Server

A pluggable, modular RAG (Retrieval-Augmented Generation) service framework exposing tool interfaces via MCP (Model Context Protocol), enabling direct integration with AI assistants such as GitHub Copilot and Claude Desktop.

## Architecture

```
┌─────────────┐     MCP Protocol     ┌──────────────────────────────────┐
│  AI Client  │ ◄──────────────────► │         MCP Server               │
│  (Copilot/  │                      │  ┌─────────────────────────────┐  │
│   Claude)   │                      │  │        Query Pipeline       │  │
└─────────────┘                      │  │  Hybrid Search + Rerank     │  │
                                     │  │  → LLM Generation           │  │
┌─────────────┐                      │  └─────────────────────────────┘  │
│  Dashboard  │ ◄──────────────────► │  ┌─────────────────────────────┐  │
│  (Streamlit)│      REST API        │  │     Ingestion Pipeline      │  │
└─────────────┘                      │  │  PDF → Chunk → Embed → Store │  │
                                     │  └─────────────────────────────┘  │
                                     └──────────┬───────────────────────┘
                                                │
                                     ┌──────────▼───────────────────────┐
                                     │       Pluggable Backends        │
                                     │  LLM / Embedding / Reranker     │
                                     │  VectorStore / Evaluator        │
                                     └─────────────────────────────────┘
```

## Core Features

### Hybrid Search + Rerank
- **Sparse Retrieval (BM25)**: Precise matching for domain-specific terms
- **Dense Retrieval (Embedding)**: Semantic matching for synonymous expressions
- **RRF Fusion**: Reciprocal Rank Fusion to balance recall and precision
- **Rerank**: Optional Cross-Encoder / LLM-based reranking for top-K refinement

### Full Pipeline Ingestion
- PDF parsing → Markdown conversion → Semantic chunking → Metadata enrichment → Embedding → Vector store
- **Multimodal support**: Vision LLM generates image descriptions (diagrams, screenshots) and stitches them into chunks, enabling "search text, retrieve images"
- Idempotent document management ensuring data consistency on updates

### MCP Integration
Exposes standardized tool interfaces via Model Context Protocol:
- `query_knowledge_hub` — Semantic search with hybrid retrieval
- `list_collections` — Browse available knowledge bases
- `get_document_summary` — Document-level summary retrieval

### Observability
- Full-chain tracing for both Ingestion and Query pipelines
- Every intermediate state is transparent and inspectable
- Structured logging with configurable levels

### Evaluation
- **Ragas framework** integration: Faithfulness, Relevancy, Recall metrics
- **Golden Test Set** regression testing for data-driven quality iteration
- Custom evaluator extension points

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | Python 3.10+ |
| Vector Store | ChromaDB |
| Sparse Retrieval | BM25 (jieba tokenizer) |
| Dense Retrieval | OpenAI / Azure OpenAI Embeddings |
| Reranker | Cross-Encoder / LLM |
| Protocol | MCP (Model Context Protocol) |
| Dashboard | Streamlit |
| Evaluation | Ragas |
| Config | YAML (factory pattern, zero-code backend switching) |

## Quick Start

### Prerequisites
- Python >= 3.10
- An LLM API key (OpenAI / Azure OpenAI / DeepSeek / Ollama)

### Install

```bash
git clone https://github.com/<your-username>/modular-rag-mcp-server.git
cd modular-rag-mcp-server
pip install -e ".[dev]"
```

### Configure

Edit `config/settings.yaml` to set your provider and API key:

```yaml
llm:
  provider: "openai"
  model: "gpt-4o"
  api_key: "your-api-key"

embedding:
  provider: "openai"
  model: "text-embedding-ada-002"
  api_key: "your-api-key"
```

### Run

```bash
# Start dashboard
streamlit run scripts/start_dashboard.py

# Ingest documents
python scripts/ingest.py --input-dir ./data/documents

# Run evaluation
python scripts/evaluate.py --test-set ./data/eval/golden_test_set.json
```

## Pluggable Architecture

Every core component defines an abstract interface with factory pattern registration. Switch backends via YAML config, zero code changes:

```
src/
├── libs/
│   ├── llm/          # LLM providers (OpenAI, Azure, Ollama, DeepSeek)
│   ├── embedding/    # Embedding providers
│   ├── reranker/     # Reranking strategies
│   └── vectorstore/  # Vector store backends
├── pipeline/
│   ├── ingestion/    # Document ingestion pipeline
│   └── retrieval/    # Query + retrieval pipeline
├── mcp_server/       # MCP protocol server
├── dashboard/        # Streamlit dashboard
└── evaluation/       # Ragas + custom evaluation
```

## Testing

```bash
# Run all tests
pytest

# Unit tests only
pytest -m unit

# Integration tests
pytest -m integration

# Exclude LLM-dependent tests
pytest -m "not llm"
```

## License

MIT
