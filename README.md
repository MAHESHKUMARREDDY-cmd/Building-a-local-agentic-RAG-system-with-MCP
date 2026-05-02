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






Here are the setup steps in order. Each step builds on the previous one.

---

## Step 1 — Check system prerequisites

Open a terminal and verify you have the required runtimes:

```bash
python3 --version      # need 3.11+
node --version         # need 18+
docker --version       # need 24+
git --version
```

If anything is missing:

```bash
# Python 3.11 (Ubuntu/Debian)
sudo apt update && sudo apt install -y python3.11 python3.11-venv python3-pip

# Node 20 via nvm (any OS)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 20

# Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # then log out and back in
```

---

## Step 2 — Install Ollama and pull models

Ollama runs your LLMs 100% locally.

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Start the Ollama daemon (runs on port 11434)
ollama serve &

# Pull the inference model (4.9 GB — grab a coffee)
ollama pull llama3.1:8b

# Pull the embedding model (274 MB)
ollama pull nomic-embed-text

# Verify both are loaded
ollama list
```

You should see `llama3.1:8b` and `nomic-embed-text` in the list. Test it quickly:

```bash
ollama run llama3.1:8b "Say hello in one sentence"
```

---

## Step 3 — Set up SearXNG (self-hosted web search)

SearXNG is your zero-cost, privacy-first web search engine. You'll run it in Docker.

```bash
# Create a directory for SearXNG config
mkdir -p ~/searxng && cd ~/searxng

# Pull the official image
docker pull searxng/searxng

# Generate a secret key
openssl rand -hex 32  # copy this output

# Create the settings file
cat > settings.yml << 'EOF'
use_default_settings: true
server:
  secret_key: "PASTE_YOUR_KEY_HERE"
  limiter: false
  image_proxy: false
search:
  safe_search: 1
  default_lang: "en"
engines:
  - name: google
    engine: google
    disabled: false
  - name: duckduckgo
    engine: duckduckgo
    disabled: false
  - name: wikipedia
    engine: wikipedia
    disabled: false
EOF
```

Now start the container:

```bash
docker run -d \
  --name searxng \
  -p 8888:8080 \
  -v ~/searxng/settings.yml:/etc/searxng/settings.yml \
  --restart unless-stopped \
  searxng/searxng
```

Test it in your browser: `http://localhost:8888` — you should see the SearXNG search UI. Also test via curl:

```bash
curl "http://localhost:8888/search?q=test&format=json" | python3 -m json.tool | head -30
```

---

## Step 4 — Create the project and Python environment

```bash
# Create project directory
mkdir -p ~/agentic-rag && cd ~/agentic-rag

# Create virtual environment
python3.11 -m venv .venv
source .venv/bin/activate

# Confirm you're inside the venv
which python   # should show ~/agentic-rag/.venv/bin/python
```

---

## Step 5 — Install Python dependencies

```bash
pip install --upgrade pip

pip install \
  langgraph \
  langchain \
  langchain-ollama \
  langchain-chroma \
  langchain-community \
  chromadb \
  flashrank \
  mcp \
  pymupdf \
  unstructured \
  rank-bm25 \
  python-dotenv \
  requests \
  httpx \
  pydantic
```

This will take 2–4 minutes. Verify key packages:

```bash
python -c "import langgraph; import flashrank; import chromadb; print('All imports OK')"
```

---

## Step 6 — Install the MCP servers

MCP servers are Node.js packages. Install them globally:

```bash
# Filesystem MCP server
npm install -g @modelcontextprotocol/server-filesystem

# SQLite MCP server (via uvx — install uv first if needed)
pip install uv
uvx mcp-server-sqlite --help   # confirms it's working

# SearXNG MCP bridge
pip install mcp-server-searxng
```

Test the filesystem MCP server manually:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | \
  npx @modelcontextprotocol/server-filesystem ~/Documents
```

If you see a JSON response with `protocolVersion`, the server is working.

---

## Step 7 — Set up your knowledge base directory

```bash
# Create the folder structure
mkdir -p ~/knowledge-base/{documents,reports,notes}

# Put a test file in it
cat > ~/knowledge-base/documents/test-policy.md << 'EOF'
# Q3 Procurement Policy

## Approved Vendors
- All vendors must be ISO 9001 certified.
- Payments above $10,000 require dual approval.

## Timeline
- Q3 runs July 1 – September 30, 2025.
- Purchase orders must be submitted by September 15.
EOF
```

Create the SQLite database for structured data:

```bash
python3 - << 'EOF'
import sqlite3, os
db_path = os.path.expanduser("~/knowledge-base/knowledge.db")
conn = sqlite3.connect(db_path)
conn.execute("""
    CREATE TABLE IF NOT EXISTS documents (
        id INTEGER PRIMARY KEY,
        title TEXT,
        content TEXT,
        category TEXT,
        created_at TEXT
    )
""")
conn.execute("""
    INSERT INTO documents (title, content, category, created_at) VALUES
    ('Q3 Budget Cap', 'Total procurement budget for Q3 is capped at $500,000 USD.', 'finance', '2025-07-01')
""")
conn.commit()
conn.close()
print("SQLite DB created at", db_path)
EOF
```

---

## Step 8 — Create the MCP configuration file

```bash
cd ~/agentic-rag

cat > mcp.json << 'EOF'
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/home/USER/knowledge-base"
      ],
      "description": "Local Markdown and PDF retrieval"
    },
    "sqlite": {
      "command": "uvx",
      "args": [
        "mcp-server-sqlite",
        "--db-path",
        "/home/USER/knowledge-base/knowledge.db"
      ],
      "description": "Structured data retrieval"
    },
    "searxng": {
      "command": "python",
      "args": ["-m", "mcp_server_searxng"],
      "env": {
        "SEARXNG_BASE_URL": "http://localhost:8888",
        "SEARXNG_MAX_RESULTS": "10"
      },
      "description": "Live web search via SearXNG"
    }
  }
}
EOF

# Replace USER with your actual username
sed -i "s/USER/$(whoami)/g" mcp.json

cat mcp.json   # verify paths look correct
```

---

## Step 9 — Create the `.env` file

```bash
cat > .env << 'EOF'
OLLAMA_BASE_URL=http://localhost:11434
CHROMA_DB_PATH=./chroma_db
KNOWLEDGE_BASE_PATH=/home/USER/knowledge-base
SEARXNG_URL=http://localhost:8888
LOG_LEVEL=INFO
EOF

sed -i "s/USER/$(whoami)/g" .env
```

---

## Step 10 — Ingest your documents into ChromaDB

Create and run the ingestion script:

```bash
cat > ingest.py << 'EOF'
import os, glob
from dotenv import load_dotenv
from langchain_ollama import OllamaEmbeddings
from langchain_chroma import Chroma
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import (
    UnstructuredMarkdownLoader, PyMuPDFLoader
)

load_dotenv()

KNOWLEDGE_BASE = os.getenv("KNOWLEDGE_BASE_PATH")
CHROMA_PATH    = os.getenv("CHROMA_DB_PATH", "./chroma_db")

embedder = OllamaEmbeddings(model="nomic-embed-text")
splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100)

docs = []

# Load Markdown files
for path in glob.glob(f"{KNOWLEDGE_BASE}/**/*.md", recursive=True):
    loader = UnstructuredMarkdownLoader(path)
    raw = loader.load()
    chunks = splitter.split_documents(raw)
    for c in chunks:
        c.metadata["source_type"] = "private_file"
        c.metadata["file_path"]   = path
    docs.extend(chunks)
    print(f"  Loaded {len(chunks)} chunks from {path}")

# Load PDF files
for path in glob.glob(f"{KNOWLEDGE_BASE}/**/*.pdf", recursive=True):
    loader = PyMuPDFLoader(path)
    raw = loader.load()
    chunks = splitter.split_documents(raw)
    for c in chunks:
        c.metadata["source_type"] = "private_file"
        c.metadata["file_path"]   = path
    docs.extend(chunks)
    print(f"  Loaded {len(chunks)} chunks from {path}")

print(f"\nTotal chunks to embed: {len(docs)}")
print("Embedding via nomic-embed-text (local)... this may take a moment")

vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embedder,
    persist_directory=CHROMA_PATH,
    collection_name="private_kb",
)

print(f"\nDone. {vectorstore._collection.count()} vectors stored in ChromaDB.")
EOF

python ingest.py
```

You should see output like `3 vectors stored in ChromaDB` (more as you add real documents).

---

## Step 11 — Save the agent script

```bash
# Copy the full agent code from the earlier response into agent.py
# Then do a quick sanity check on imports:

python -c "
from langchain_ollama import ChatOllama, OllamaEmbeddings
from langchain_chroma import Chroma
from flashrank import Ranker, RerankRequest
from langgraph.graph import StateGraph, END
print('All agent imports OK')
"
```

---

## Step 12 — Run a test query

```bash
python - << 'EOF'
from dotenv import load_dotenv
load_dotenv()

from langchain_ollama import ChatOllama

llm = ChatOllama(model="llama3.1:8b")
response = llm.invoke("In one sentence, confirm you are running locally.")
print("LLM response:", response.content)

from langchain_ollama import OllamaEmbeddings
from langchain_chroma import Chroma

embedder = OllamaEmbeddings(model="nomic-embed-text")
vs = Chroma(
    collection_name="private_kb",
    embedding_function=embedder,
    persist_directory="./chroma_db"
)
results = vs.similarity_search("procurement policy", k=2)
print(f"\nChroma retrieval test: found {len(results)} chunks")
for r in results:
    print(" →", r.page_content[:80])

import requests
r = requests.get("http://localhost:8888/search?q=EU+supply+chain+2025&format=json")
print(f"\nSearXNG status: {r.status_code}, results: {len(r.json().get('results', []))}")
EOF
```

All three checks should pass — LLM responds, ChromaDB returns chunks, SearXNG returns web results.

---

## Step 13 — Run the full agent

```bash
python agent.py
```

You should see the agent walk through classification → retrieval → re-ranking → synthesis → critique → output and print the final verified answer to your terminal.

---

## Quick reference — what runs where

| Service | Port | How to start | How to check |
|---|---|---|---|
| Ollama | 11434 | `ollama serve` | `curl localhost:11434` |
| SearXNG | 8888 | `docker start searxng` | `localhost:8888` in browser |
| ChromaDB | embedded | auto (no daemon) | check `./chroma_db` folder exists |
| MCP servers | stdio | spawned by agent | runs automatically |

After a reboot, just run `ollama serve &` and `docker start searxng` — everything else starts on demand when you run `python agent.py`.
