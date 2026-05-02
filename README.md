# Agentic RAG — MCP Hybrid Intelligence System

A production-ready, zero-cost Agentic RAG system that answers queries by
orchestrating between your **private local files** and **live web data**
using the Model Context Protocol (MCP). Runs 100% locally — no OpenAI,
no API keys, no monthly bills.

---

## What it does

- Routes your query to local files, the web, or both — automatically
- Reads your Markdown and PDF files via Filesystem MCP
- Queries a self-hosted SearXNG instance for live web results
- Re-ranks all chunks locally using FlashRank cross-encoder
- Synthesizes an answer using Ollama (Llama 3.1 8B)
- Runs a Chain-of-Verification critique to catch hallucinations
- Resolves conflicts between private and public data with arbitration rules

## Architecture
User Query
│
▼
Intent Classifier (LangGraph)
│
├──────────────────────┐
▼                      ▼
Local MCP Stack        Web MCP Stack
(Filesystem + SQLite)  (SearXNG)
│                      │
└──────────┬───────────┘
▼
FlashRank Re-ranker
│
▼
Synthesis Node
(Ollama LLM)
│
▼
Critique Node
(Chain-of-Verification)
│
┌───────┴────────┐
▼                ▼
CoVe Resolution   Final Output
(if conflict)     (if verified)
│
└──► re-synthesize (cyclic)


## Tech stack

| Layer | Tool | License |
|---|---|---|
| Orchestration | LangGraph | Apache 2.0 |
| LLM inference | Ollama + Llama 3.1 8B | MIT / Meta |
| Embeddings | nomic-embed-text | Apache 2.0 |
| Vector store | ChromaDB | Apache 2.0 |
| Re-ranking | FlashRank | Apache 2.0 |
| Local retrieval | Filesystem MCP + SQLite MCP | MIT |
| Web retrieval | SearXNG + MCP bridge | AGPL |
| Document parsing | PyMuPDF + Unstructured | AGPL / Apache |

**Total recurring cost: $0**

---

## Prerequisites

- Python 3.11+
- Node.js 18+
- Docker 24+
- 8 GB RAM minimum (16 GB recommended)
- 10 GB free disk space for models

---

## Setup

See **[docs/setup.md](docs/setup.md)** for the full step-by-step guide.

Quick start after setup:

```bash
# Activate environment
source .venv/bin/activate

# Start services
ollama serve &
docker start searxng

# Ingest your documents
python ingest.py

# Run the agent
python agent.py
```

---

## Configuration

Copy `.env.example` to `.env` and edit the paths:

```bash
cp .env.example .env
```

Edit `mcp.json` to point to your knowledge base directory.

---

## Project structure


agentic-rag-mcp/
├── README.md           # this file
├── agent.py            # main LangGraph agent
├── ingest.py           # document ingestion into ChromaDB
├── mcp.json            # MCP server configuration
├── requirements.txt    # Python dependencies
├── .env.example        # environment variable template
└── docs/
└── setup.md        # full step-by-step setup guide

## License

MIT — use freely, modify, and build on it.
