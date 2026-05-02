# Setup Guide — Step by Step

Follow each step in order. Every step builds on the previous one.

---

## Step 1 — Check system prerequisites

```bash
python3 --version      # need 3.11+
node --version         # need 18+
docker --version       # need 24+
git --version
```

Install missing tools:

```bash
# Python 3.11
sudo apt update && sudo apt install -y python3.11 python3.11-venv python3-pip

# Node 20 via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 20

# Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

---

## Step 2 — Install Ollama and pull models

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama serve &
ollama pull llama3.1:8b
ollama pull nomic-embed-text
ollama list
```

---

## Step 3 — Set up SearXNG

```bash
mkdir -p ~/searxng && cd ~/searxng
docker pull searxng/searxng
openssl rand -hex 32    # copy this value

cat > settings.yml << 'EOF'
use_default_settings: true
server:
  secret_key: "PASTE_YOUR_KEY_HERE"
  limiter: false
search:
  safe_search: 1
  default_lang: "en"
EOF

docker run -d \
  --name searxng \
  -p 8888:8080 \
  -v ~/searxng/settings.yml:/etc/searxng/settings.yml \
  --restart unless-stopped \
  searxng/searxng
```

Test: open `http://localhost:8888` in your browser.

---

## Step 4 — Create the project environment

```bash
git clone https://github.com/YOUR_USERNAME/agentic-rag-mcp.git
cd agentic-rag-mcp
python3.11 -m venv .venv
source .venv/bin/activate
```

---

## Step 5 — Install Python dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
python -c "import langgraph; import flashrank; import chromadb; print('OK')"
```

---

## Step 6 — Install MCP servers

```bash
npm install -g @modelcontextprotocol/server-filesystem
pip install uv
pip install mcp-server-searxng
```

---

## Step 7 — Set up your knowledge base

```bash
mkdir -p ~/knowledge-base/{documents,reports,notes}
```

Drop your Markdown and PDF files into `~/knowledge-base/documents/`.

---

## Step 8 — Configure environment

```bash
cp .env.example .env
```

Edit `.env` — replace `YOUR_USERNAME` with your actual Linux username.
Edit `mcp.json` — replace `USER` with your actual Linux username.

---

## Step 9 — Ingest documents

```bash
python ingest.py
```

---

## Step 10 — Run the agent

```bash
python agent.py
```

---

## After a reboot

```bash
ollama serve &
docker start searxng
source .venv/bin/activate
python agent.py
```
