# Architecture

> Assumes you've read [index.md](index.md), which covers the crew-level flow.
> This page goes one level down: layers, the graph data model, the ingest and retrieval internals, failure points, and scaling.

---

## High-Level Architecture

```mermaid
flowchart TB
    classDef client fill:#e3f2fd,stroke:#1565c0,color:#0d47a1
    classDef app fill:#ede7f6,stroke:#5e35b1,color:#311b92
    classDef db fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef ext fill:#fff3e0,stroke:#ef6c00,color:#e65100

    U([Jupyter user<br/>my_crew.kickoff]):::client

    subgraph L1["Orchestration · CrewAI"]
        CR[Crew · Process.sequential<br/>4 agents / 4 tasks]:::app
    end

    subgraph L2["Tools · @tool functions"]
        T1[get papers<br/>ingest]:::app
        T2[get paper by title<br/>exact lookup]:::app
        T3[get relevant context<br/>chunk vector search]:::app
        T4[get relevant summaries<br/>abstract vector search]:::app
    end

    subgraph L3["Helpers"]
        H1[download_arxiv_papers]:::app
        H2[get_text_chunks_from_pdf]:::app
        H3[generate_embedding]:::app
    end

    subgraph L4["Storage"]
        NEO[(Neo4j + GDS plugin)]:::db
        FS[(research_papers/*.pdf<br/>local disk)]:::db
    end

    subgraph L5["External services"]
        AX[arXiv API]:::ext
        OL[Ollama · nomic-embed-text]:::ext
        GQ[Groq · llama-3.3-70b-versatile]:::ext
    end

    U --> CR
    CR <--> GQ
    CR --> T1 & T2 & T3 & T4
    T1 --> H1 & H2 & H3
    T3 & T4 --> H3
    H1 --> AX
    H1 --> FS
    H2 --> FS
    H3 --> OL
    T1 & T2 & T3 & T4 --> NEO
```

## Component Responsibilities

| Layer | Component | Owns | Doesn't do |
|---|---|---|---|
| Orchestration | `my_crew` | Task order; passes each task's output to the next | Parallelism; validating agent output |
| Agents | `search_agent` | Decides the topic string for ingest | Answering |
| | `database_agent` | Exact paper lookup (`allow_delegation=True`) | Vector search (see note below) |
| | `question_answer_agent` | Final answer from top-k chunks | Seeing `{user_query}` directly |
| | `future_works_agent` | Research-gap suggestions | Adding new data |
| Tools | `add_paper_to_neo4j` | All graph writes | Deduplicating chunks |
| | `get_text_chunk_nodes_by_title_and_year` | Read-only lookup by exact title + year | Fuzzy matching |
| | `get_relevant_context` / `vector_search_summaries` | Embed query → cosine rank → concatenate text | Returning scores or metadata to the agent |
| Helpers | `download_arxiv_papers` | arXiv search, 5-year filter, PDF download | Retries or rate limiting |

> [!NOTE]
> `query_database_task` sets `tools=[vector_search_summaries]`, while `database_agent` has `tools=[get_text_chunk_nodes_by_title_and_year]`. In the recorded run the agent only called **`get paper by title`**. How CrewAI resolves task tools vs. agent tools in the installed version is **NEEDS VERIFICATION**.

## Graph Data Model

```mermaid
erDiagram
    Author ||--o{ SummaryNode : WROTE
    TextChunk }o--|| SummaryNode : BELONGS_TO

    Author {
        string name "MERGE key"
    }
    SummaryNode {
        string title
        string summary "arXiv abstract; used as the match key"
        list embedding "nomic-embed-text"
        datetime publish_date "naive LocalDateTime"
        string pdf_path
    }
    TextChunk {
        string text "up to 2000 chars"
        list embedding
        string title
        datetime publish_date
    }
```

- **`summary` is the effective primary key.** Author and chunk writes find their paper with `MATCH (n:SummaryNode {summary: $summary_text})`.
- There are **no constraints or indexes** anywhere in the repo, including vector indexes.

## Request Lifecycle — `get papers` (ingest)

This is the heaviest request in the system. It runs as a single tool call inside the search task.

```mermaid
sequenceDiagram
    autonumber
    participant Agent as search_agent
    participant Tool as add_paper_to_neo4j
    participant AX as arXiv
    participant FS as Local disk
    participant OL as Ollama
    participant Neo as Neo4j

    Agent->>Tool: {"topic": "superconductors"}
    Tool->>AX: Search(query, max_results=10, sort=Relevance)
    loop each result published in the last 5 years
        Tool->>AX: download_pdf
        AX-->>FS: research_papers/<title>.pdf
    end
    loop each paper
        Tool->>OL: embed(abstract)
        Tool->>Neo: MERGE SummaryNode {all props}
        loop each author
            Tool->>Neo: MERGE Author, MERGE (a)-[:WROTE]->(n)
        end
        Tool->>FS: PyMuPDFLoader → split 2000/100
        loop each chunk
            Tool->>OL: embed(chunk)
            Tool->>Neo: CREATE TextChunk, MERGE [:BELONGS_TO]
        end
    end
    Tool-->>Agent: "Data added to neo4j graph database"
```

> Every step here is **synchronous and serial**: one HTTP call to Ollama and one Cypher statement per chunk.

## Data Flow

```mermaid
flowchart LR
    classDef db fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef ext fill:#fff3e0,stroke:#ef6c00,color:#e65100

    subgraph Write["Write path"]
        Q1[topic] --> AX[arXiv results]:::ext
        AX --> PDF[PDF files]:::db
        AX --> ABS[abstract + authors + date]
        PDF --> TXT[full text] --> CH[2000-char chunks]
        ABS --> E1[embed]
        CH --> E2[embed]
    end

    E1 --> G[(Neo4j graph)]:::db
    E2 --> G

    subgraph Read["Read path"]
        Q2[query_text] --> E3[embed] --> COS[cosine vs. every node]
        COS --> TOP[top 5] --> STR[concatenated text]
        STR --> LLM[Groq LLM prompt]:::ext
    end

    G --> COS
```

## State Management

| State | Lives in | Changes when | Persistent? |
|---|---|---|---|
| Papers, authors, chunks, embeddings | Neo4j (`NEO4J_DATABASE`) | Each `get papers` call | ✅ Across runs, and it **accumulates** |
| PDF files | `./research_papers/` | Each download; same title → overwritten | ✅ |
| Neo4j driver, `llm`, agents, tasks | Notebook kernel globals | Cell execution | ❌ Lost on kernel restart |
| Inter-task context | CrewAI's in-memory task outputs | Each task completes | ❌ Per `kickoff` |
| Config | `.env` → `os.getenv` | Kernel start (`override=True`) | File-based |

**Persistence boundary:** only Neo4j and the PDF folder survive a run. The final answer is returned as `result` and is not saved anywhere.

## Async / Event Flow

Not applicable. The code has no async calls, queues, background workers, or event handlers; everything runs sequentially in the notebook kernel. The only "events" are CrewAI's verbose logs printed to stdout.

## Failure Boundaries

```mermaid
flowchart TB
    classDef bad fill:#ffebee,stroke:#c62828,color:#b71c1c

    K[kickoff] --> S[Search task]
    S -->|arXiv down / rate-limited| F1[Exception in tool]:::bad
    S -->|Ollama not running| F2[Embedding connection error]:::bad
    S -->|bad PDF / odd filename| F3[Loader or file error]:::bad
    S --> D[Database task]
    D -->|title/year not found| F4[Empty string → agent may<br/>fabricate content]:::bad
    D --> Q[Q&A task]
    Q -->|GDS plugin missing| F5[Unknown function<br/>gds.similarity.cosine]:::bad
    Q -->|malformed tool input| F6[CrewAI 'not a valid key,<br/>value dictionary' → agent retries]:::bad
    Q --> FW[Future-works task]
    FW --> R[result]
```

| Boundary | Handling in code | Observed? |
|---|---|---|
| arXiv / Ollama / Neo4j errors | None (no try/except). Only `max_retries=3` on the search task | Not observed |
| Tool input parsing | CrewAI returns an error to the agent, and the agent tries again | ✅ Cell 54 output |
| Empty retrieval | Tool returns `""`; nothing prevents fabrication | ✅ Cell 54: invented "John Doe" paper |
| Partial ingest | No transaction around a paper; a failure mid-loop leaves a partial graph | Inferred from code |

## Scaling Model

**Current model:** one process, one notebook, serial I/O, and a single Neo4j instance.

| Bottleneck | Why | At 10× load (≈100 papers/run, 10× graph) | What would need to change |
|---|---|---|---|
| Ingest time | One Ollama call and one Cypher write per chunk, all in series | Grows roughly linearly with the number of chunks | Batch embeddings; `UNWIND` batched writes; parallel downloads |
| Vector search | `MATCH (t:TextChunk)` + GDS cosine = full scan | Query cost grows linearly with the number of chunks | Neo4j vector index (`db.index.vector.queryNodes`) |
| Duplicate chunks | `CREATE` on each run | Graph grows with every re-run of a topic | `MERGE` on a chunk hash, or uniqueness constraints |
| LLM context | Top-5 × 2000 chars sent to Groq in each tool call | Stays constant per query, but more agents means more tokens | Re-ranking; summarizing context first |
| Groq rate limits | Hosted API | **UNKNOWN / NEEDS VERIFICATION** (depends on plan) | Backoff, caching, or a local model (an Ollama LLM is already stubbed in cell 30) |

No benchmarks exist in the repository.

## Architecture Decisions

| Decision | Why | Tradeoff | Evidence |
|---|---|---|---|
| **GraphRAG on Neo4j** instead of a standalone vector DB | Keeps relationships (author ↔ paper ↔ chunk) and vectors together | Brute-force similarity; GDS plugin needed | Cells 12, 19, 22 |
| **Embeddings as node properties** + GDS cosine | No index setup; works on any GDS-enabled instance | Linear scan per query | `gds.similarity.cosine(t.embedding, $query_embedding)` |
| **Groq-hosted Llama 3.3 70B** | Large model, fast inference; `llama-3.1-70b-versatile` in the copy notebook suggests a model upgrade | External dependency; hard-coded key | Cell 29; `project_copy.ipynb` |
| **Local Ollama embeddings** | Free, private embedding step | Ollama must be running locally | Cell 4 |
| **Sequential crew** | Deterministic order: store → look up → answer → extend | No parallel steps; errors carry forward | Cell 53 `Process.sequential` |
| **Abstract as the node key** | arXiv provides no other ID in the stored fields | Fragile; an arXiv entry ID would be sturdier | `MATCH (n:SummaryNode {summary: ...})` |
| **Ollama as an alternative LLM** (commented out) | Option to run fully local | Not active | Cell 30 |
