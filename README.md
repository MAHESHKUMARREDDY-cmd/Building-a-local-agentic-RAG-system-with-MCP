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
