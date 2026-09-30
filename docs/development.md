# Development Guide

> How to run, configure, and change the project. For *why* it is built this way, see [architecture.md](architecture.md).

---

## Prerequisites

| Requirement | Why | Notes |
|---|---|---|
| Python **3.10** | Notebook kernel metadata says `3.10.15` | Other versions are untested |
| Jupyter | The whole project is `project.ipynb` | |
| **Neo4j 5** + **Graph Data Science** plugin | Storage and `gds.similarity.cosine` | Local Docker or Aura. Aura needs a GDS-enabled tier (**NEEDS VERIFICATION**) |
| **Ollama**, running locally | Embeddings (`nomic-embed-text`) | Default `http://localhost:11434` |
| Groq API key | LLM (`llama-3.3-70b-versatile`) | |

## Installation

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Imported by the notebook but missing from requirements.txt:
pip install langchain langchain-community langchain-ollama langchain-groq langchain-text-splitters pytz

ollama pull nomic-embed-text
```

> [!IMPORTANT]
> No package versions are pinned. The notebook uses `from langchain.document_loaders import ...` and `from langchain.text_splitter import ...`, which newer LangChain releases have moved or removed. The versions that worked originally are **UNKNOWN / NEEDS VERIFICATION**. If an import fails, switch to `langchain_community.document_loaders` and `langchain_text_splitters`.

**Unused packages in `requirements.txt`:** `google-api-python-client`, `google-auth-*`, `google`, `duckduckgo-search`, `streamlit`, `fastapi`, `unstructured`, `tools`, `pyright`, `ruff`. The first cell imports the Google auth modules, but nothing uses them.

## Configuration

Start Neo4j with GDS:

```bash
docker run -d --name neo4j -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/password \
  -e NEO4J_PLUGINS='["graph-data-science"]' \
  neo4j:5
```

Create `.env` from the template (the notebook loads `.env` from its working directory):

```bash
cp .env_example .env
```

> [!WARNING]
> **The Groq key is hard-coded** in cell 29 (`llm = LLM(model=..., api_key="gsk_...")`) of both notebooks. Change it to read from the environment:
> ```python
> llm = LLM(model="groq/llama-3.3-70b-versatile", api_key=os.getenv("GROQ_API_KEY"))
> ```

## Environment Variables

| Variable | Used by | Example | Read in code? |
|---|---|---|---|
| `NEO4J_URI` | `GraphDatabase.driver` | `bolt://localhost:7687` | ✅ cell 6 |
| `NEO4J_USERNAME` | driver auth | `neo4j` | ✅ |
| `NEO4J_PASSWORD` | driver auth | `password` | ✅ |
| `NEO4J_DATABASE` | `driver.session(database=...)` | `neo4j` | ✅ |
| `GROQ_API_KEY` | Groq LLM | `gsk_...` | ❌ Listed in `.env_example` but not read; see the warning above |

## Running Locally

1. Start Neo4j (with GDS) and Ollama.
2. Open `project.ipynb` and **run all cells**. The last code cell starts the crew:

```python
result = my_crew.kickoff(inputs={
    'user_query': 'What is currently going on in the field of superconductors?',
    'topics': 'superconductors',
})
```

3. Watch the verbose agent trace in the cell output. `result` holds the future-works agent's final answer.
4. Check the graph at <http://localhost:7474>:

```cypher
MATCH (a:Author)-[:WROTE]->(s:SummaryNode)<-[:BELONGS_TO]-(t:TextChunk)
RETURN a, s, t LIMIT 50
```

> 💡 **Ingest only:** `storage_crew` (cells 50–51, commented out) runs just the search task. Uncomment it to fill the graph without running the Q&A agents.

## Important APIs

There is no HTTP API. The "API" is the set of **CrewAI tools** the agents can call.

| Tool name (agent-facing) | Function | Input | Returns |
|---|---|---|---|
| `get papers` | `add_paper_to_neo4j` | `topic: str` | `"Data added to neo4j graph database"` |
| `get paper by title` | `get_text_chunk_nodes_by_title_and_year` | `title: str`, `year: int` | All chunk texts joined with newlines (`""` if no match) |
| `get relevant context` | `get_relevant_context` | `query_text: str` | Top-5 chunk texts joined together |
| `get relevant summaries` | `vector_search_summaries` | `query_text: str` | Top-5 abstracts joined together |

| Agent → Task | Tools attached |
|---|---|
| `search_agent` → `search_and_store_task` | `get papers` |
| `database_agent` → `query_database_task` | Agent: `get paper by title`; task: `get relevant summaries` |
| `question_answer_agent` → `q_and_a_task` | `get relevant context` |
| `future_works_agent` → `future_works_task` | `get relevant context` |

To call a tool directly while debugging:

```python
print(get_relevant_context.invoke("Explain the working of superconducting qubits"))
```

This pattern appears in the commented-out cell 20. Depending on the CrewAI version, you may need `.run(...)` instead (**NEEDS VERIFICATION**).

## Important Code Paths

| Path | Cells | What to know before changing it |
|---|---|---|
| **Ingest** | 3 → 9 → 12 | `max_results`, the date window, and the chunk size are hard-coded. Chunks use `CREATE`, so re-runs duplicate them |
| **Vector search** | 19, 22 | Cypher `ORDER BY similarity DESC LIMIT $top_k`. To use a vector index, swap `MATCH` + `gds.similarity.cosine` for `db.index.vector.queryNodes` |
| **Prompt wiring** | 32–47 | `{user_query}` / `{topics}` are filled into agent `goal` strings. Add `{user_query}` to the Q&A and future-works task descriptions so those agents see the question |
| **LLM choice** | 29–30 | Groq is active; the local Ollama `llama3.1` option is commented out in cell 30 |

> `project_copy.ipynb` is an older snapshot. The only code difference is the model name (`llama-3.1-70b-versatile`) and the API key. Treat `project.ipynb` as the source of truth.

## Testing

No tests exist. There is no test directory, test framework, or CI config. `pyright` and `ruff` are in `requirements.txt`, but no config for them exists (**NEEDS VERIFICATION** whether they were ever used).

To check each piece by hand, use the commented example cells (10, 16–17, 20–21, 23–26). They call each helper or tool on its own.

## Debugging

| Symptom | Where to look |
|---|---|
| Agent picks the wrong tool, or passes a list instead of a dict | `verbose=True` trace in the kickoff cell output |
| Answer mentions papers that aren't in the graph | Check the database task output. An empty tool result can lead to invented content (see [index → limitations](index.md#known-limitations)) |
| Suspected duplicate data | `MATCH (t:TextChunk) RETURN t.title, count(*)` |
| Start over from an empty graph | `MATCH (n) DETACH DELETE n` ⚠️ deletes everything in the database |

## Deployment

The repository has no deployment setup: no Dockerfile, server, CI, or hosting config. The project runs only as a local notebook.

## Common Problems

| Error / symptom | Likely cause | Fix |
|---|---|---|
| `Unknown function 'gds.similarity.cosine'` | Neo4j is running without the GDS plugin | Add `NEO4J_PLUGINS='["graph-data-science"]'` |
| Connection refused on `:11434` | Ollama isn't running | `ollama serve`; `ollama pull nomic-embed-text` |
| `ModuleNotFoundError: langchain_ollama` (or `langchain_groq`, `pytz`) | Missing from `requirements.txt` | See [Installation](#installation) |
| Groq auth / model error | Hard-coded key is invalid, or the model has been retired | Use your own key; check Groq's current model list |
| `ServiceUnavailable` from the Neo4j driver | Wrong `NEO4J_URI`, or `.env` isn't in the working directory | Start Jupyter from the repo root |
| PDF download fails on Windows | Filenames only have spaces and `/` replaced; characters like `:` or `?` stay in (inferred) | Clean the filename more thoroughly in `download_arxiv_papers` |
