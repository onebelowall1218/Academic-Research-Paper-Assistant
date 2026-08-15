# Academic Research Paper Assistant — Agentic GraphRAG 🧠📚

An **AI research assistant** that automates searching, retrieving, and reasoning across hundreds of academic papers. It combines a **multi-agent crew** (CrewAI) with a **Neo4j knowledge graph** and retrieval-augmented generation (GraphRAG) — cutting manual literature-review time by an estimated **40–50%**.

> 📓 Full implementation:
> [`project.ipynb`](project.ipynb)

---

## What it does
- **Fetches** papers from the **arXiv API** on a topic you give it.
- **Builds a knowledge graph** in **Neo4j** (papers, authors, topics, relationships).
- **Answers questions** by having agents retrieve relevant subgraphs and reason over them — so answers are **grounded in real papers**, not hallucinated.

## Why it's interesting
Plain LLM Q&A over papers hallucinates. Routing retrieval through a **graph** (GraphRAG) keeps answers tied to actual sources and relationships between papers — a more reliable pattern for research and enterprise knowledge.

## Architecture
```
arXiv API ──► ingest ──► Neo4j knowledge graph
                                  ▲
        ┌──────────── agents (CrewAI) ─────────────┐
        │  planner · graph-retriever · reasoner · writer │
        └────────────────────────────────────────────────┘
                       ▲              │
                  LangChain      LLaMA 3.1  (Ollama / Groq)
```
**4 agents · 4 tools**, orchestrated end-to-end.

## Run it
```bash
git clone https://github.com/onebelowall1218/Academic-Research-Paper-Assistant
cd Academic-Research-Paper-Assistant
pip install -r requirements.txt

# Start a local Neo4j instance (Docker is easiest):
docker run -d --name neo4j -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/password neo4j:5

# Then open the notebook and run the cells:
jupyter notebook project.ipynb
```
<!-- TODO(anjum): confirm the Neo4j URI / credentials the notebook expects, and note any GROQ_API_KEY needed. -->

## Stack
`CrewAI` · `LangChain` · `Ollama` · **LLaMA 3.1** · `Groq` · `Neo4j` · `arXiv API` · `Python`

<!-- TODO(anjum): a short GIF of "ask a question → grounded answer" would make this pop for recruiters. -->
