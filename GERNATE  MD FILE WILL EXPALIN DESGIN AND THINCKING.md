<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# GERNATE  MD FILE WILL EXPALIN DESGIN AND THINCKING ABOUT THIS DESGIN SO WILL SHARE WITH aI ASSITENCT FOR CONTEXT

Here’s a `DESIGN_OVERVIEW.md` you can add to the repo and share with any AI assistant as context.

***

```markdown
# Design Overview — Copilot MCP Codebase Agent

This document explains the **design, architecture, and reasoning** behind the Copilot MCP Codebase Agent implemented in this repository. It is written for AI assistants and engineers who need deep context about how and why the system is structured this way.

---

## 1. Problem Statement and Goals

We want an internal agent that:

- Understands a **large Java codebase** (OMS / trading systems).
- Can answer structural questions like:
  - “Where is order validation implemented?”
  - “What calls `SequencerQueue.enqueue`?”
  - “What could be impacted if I change this method/class?”
- Can be used **directly from VS Code** via **GitHub Copilot** (MCOP) as an MCP server.
- **Does not depend on external LLM keys** on the backend:
  - No OpenAI / Anthropic calls from our infra.
  - The LLM reasoning is done by Copilot’s models.
- Keeps **source code and embeddings on-prem**, stored in systems we control.

In other words: this service is a **knowledge + graph + tools backend**, and Copilot is the **LLM frontend** that uses it.

---

## 2. High-Level Architecture

At a high level:

- **Ingestion layer**:
  - Walks the Java codebase.
  - Splits code into chunks (currently file-level; later method-level).
  - Embeds each chunk using a **local sentence-transformers model**.
  - Writes embeddings + metadata to **Qdrant**.
  - Writes basic nodes to **Neo4j** (and later CALLS edges).

- **Storage layer**:
  - **Qdrant** = vector DB for semantic code search.
  - **Neo4j** = graph DB for call graph and relationship queries.

- **API layer (FastAPI)**:
  - `/api/index` — triggers full indexing (eager, no lazy init).
  - `/webhook/push` — placeholder for later incremental indexing via git hooks.
  - `/health/ready` — readiness check.
  - `/mcp/tools/list` and `/mcp/tools/call` — MCP tooling interface.

- **MCP tools**:
  - `search_code(query, layer?)` — semantic search over code.
  - `find_callers(method)` — graph query to find who calls a method.
  - `find_callees(method)` — graph query to find what a method calls.

- **Client**:
  - GitHub Copilot / MCOP in VS Code, acting as an **MCP client**.
  - Copilot’s LLM orchestrates tool calls and reasoning:
    - Calls our tools to get code and relationships.
    - Produces natural language answers and code edits.

This architecture keeps the backend **stateless w.r.t. LLM** and focuses on **high-quality retrieval and graph information**.

---

## 3. Key Design Decisions

### 3.1 Python + FastAPI instead of Java backend

Although the original design document was for a Java + Spring Boot backend, this project uses **Python + FastAPI** for the MCP agent because:

- Python has strong ecosystems for:
  - **Qdrant** (`qdrant-client`).
  - **Neo4j** (`neo4j` driver).
  - **Sentence-transformers** / HuggingFace models.
- Fast iteration and integration with LangChain / RAG patterns.
- The backend’s job is not performance-sensitive business logic; it’s data retrieval and lightweight processing.

The **Java codebase being indexed does not require the agent itself to be written in Java**; we treat the code as data.

### 3.2 No external LLM on backend

We explicitly avoid using OpenAI/Anthropic in the backend because:

- The organization may not have or want LLM API keys at infra level.
- GitHub Copilot already provides LLM capabilities at the *client* side.
- We can shift the “reasoning” responsibility to Copilot’s LLM:
  - The backend only returns **code snippets + graph info**.
  - Copilot’s LLM decides how to answer the user.

This leads to a simpler, more secure backend:

- No outbound LLM network calls.
- No need to manage or rotate LLM API keys.
- Clear separation of responsibilities:
  - Backend: **data + tools**.
  - Copilot: **intelligence**.

### 3.3 Qdrant + Neo4j as the persistent knowledge base

We use:

- **Qdrant** for:
  - Storing embeddings of each code chunk.
  - Performing semantic search on user queries.
- **Neo4j** for:
  - Representing call relationships and (later) module/layer relationships.
  - Handling impact analysis and flow queries via Cypher.

Reasoning:

- Qdrant is an efficient vector DB with good Python support and is battle-tested for RAG [web:34][web:37].
- Neo4j is designed for graph queries like “what calls what” and multi-hop path finding [web:20][web:23].
- Keeping both separate allows:
  - Semantic retrieval via vectors.
  - Precise structural reasoning via graph queries.

### 3.4 Local embeddings model

The embedding layer uses **sentence-transformers/all-MiniLM-L6-v2** by default:

- No API keys required.
- Good trade-off between quality and speed for code/comment-like text.
- Can be swapped for any other sentence-transformers model.

This aligns with the “no external LLM key” requirement while still enabling semantic search.

### 3.5 MCP-first interface

We designed the backend **specifically as an MCP server**:

- Primary endpoints:
  - `GET /mcp/tools/list` — tool discovery.
  - `POST /mcp/tools/call` — tool execution.
- Tool outputs are simple `{"content": [{"type": "text", "text": "..."}]}` structures that MCP-aware clients expect.
- Authentication via `Authorization: Bearer <MCP_AUTH_TOKEN>` to keep tools private.

This makes GitHub Copilot / MCOP integration straightforward and follows the emerging pattern of “MCP tools + LLM client”.

---

## 4. Data Model and Ingestion Strategy

### 4.1 Code chunk model

We define a `CodeChunk` dataclass:

- `id` — stable identifier for the chunk.
- `file_path` — full path of the source file.
- `class_name` / `method_name` / `package_name` — optional (reserved for richer parsing).
- `layer` — optional (e.g. validator, service, repository).
- `annotations` — optional list of annotations.
- `calls` — optional list of callee method names.
- `chunk_type` — `file` for now, later `method`, `class`, etc.
- `content` — actual text used for embedding.
- `repo`, `start_line`, `end_line` — reserved for advanced usage.

This is inspired by the original Java design doc’s method-level chunking but implemented in a simpler form initially so the system is usable end-to-end.

### 4.2 Current parsing strategy

For now, `java_parser.py`:

- Treats each `.java` file as **a single chunk**.
- Reads file text, computes a hash for `id`.
- Does not yet extract:
  - Methods.
  - Call lists.
  - Layer metadata.

**Reasoning**:

- This lets us have a **working system quickly**:
  - Semantic search over files works.
  - Qdrant and MCP wiring can be tested.
- It keeps parsing logic intentionally simple and local, so we can later swap in:
  - Tree-sitter-based Java parsing from Python.
  - Or calls to an existing Java-based AST microservice (e.g. JavaParser).

The data model is **future-proofed**: `CodeChunk` already has fields for method-level info.

### 4.3 Future: method-level and call graph enrichment

The plan is to evolve ingestion to:

- Parse methods and calls.
- Populate Neo4j as:

  - `(Class)-[:HAS_METHOD]->(Method)`
  - `(Method)-[:CALLS]->(Method)`
  - Optionally `(Class)-[:DEPENDS_ON]->(Class)`

This will enable:

- Accurate call graph queries (`find_callers`, `find_callees`, `trace_path`).
- Richer impact analysis (which modules are affected if a method changes).

The current `CallGraph` class is structured to support this future extension.

---

## 5. Qdrant and Neo4j Initialization

We want **no lazy initialization** at first MCP tool call.

### 5.1 Eager init at startup

In `app.main` startup event we:

1. Instantiate `QdrantStore` and call `ensure_collection()`:
   - If the collection doesn’t exist, create it with correct vector size and distance.
2. Instantiate `CallGraph` and run a simple `RETURN 1` query to verify Neo4j connectivity.
3. Warm the embedding model by calling `_embedding_model()` once.

**Reasoning**:

- Fail fast if Qdrant or Neo4j are not reachable.
- Ensure the vector collection exists before any indexing or search.
- Avoid embedding model load latency on the first MCP call.

We do **not** load all vectors or graph nodes into Python memory; Qdrant and Neo4j manage their own data.

### 5.2 Indexing

Indexing is triggered via:

- `POST /api/index` — manual call or CI/CD hook.

Indexing:

- Walks the source tree from `source.local_path`.
- Applies include/exclude filters.
- For each file:
  - Creates one or more `CodeChunk` objects.
  - Embeds and upserts chunks to Qdrant.
  - Updates Neo4j nodes via `CallGraph.update_from_chunk`.

The data persists in:

- `./data/qdrant` (mounted volume).
- `./data/neo4j`.

So subsequent restarts do not require recomputing everything; we can later add index metadata to skip re-indexing when nothing changed.

---

## 6. MCP Tools and Usage Pattern

### 6.1 Tools exposed

Currently:

- `search_code`:
  - Input:
    - `query: string`
    - `layer?: string` (optional future filter).
  - Behaviour:
    - Embeds `query` with local model.
    - Qdrant search.
    - Returns concatenated text blocks of file/method + content.

- `find_callers`:
  - Input:
    - `method: string`.
  - Behaviour:
    - Neo4j query for `(caller)-[:CALLS]->(target)`.
    - Returns a formatted list of caller method names.

- `find_callees`:
  - Input:
    - `method: string`.
  - Behaviour:
    - Neo4j query for `(source)-[:CALLS]->(callee)`.
    - Returns a formatted list of callee method names.

These are deliberately **low-level tools**; Copilot’s LLM does higher-level explanations.

### 6.2 Why this split?

We intentionally keep tools:

- **Deterministic and side-effect free**.
- Returning **data**, not opinions.

This makes it easier for Copilot’s LLM to:

- Compose multiple tool results.
- Cross-check information.
- Build explanations, proposed refactorings, or impact analyses based on the returned code and graph information.

---

## 7. Security and Isolation

Security considerations:

- Qdrant and Neo4j are internal-only services exposed via Docker Compose:
  - Bound to `localhost` ports.
  - Persist data to local volumes (not cloud).
- MCP endpoints require:
  - `Authorization: Bearer <MCP_AUTH_TOKEN>`.
- Git webhooks (if used) are verified using `WEBHOOK_SECRET`.

The backend does **not** communicate with any external LLM or embedding API, which minimizes data leakage risk.

---

## 8. Extensibility and Future Work

Key extension points:

1. **Richer Java parsing**:
   - Swap `java_parser.py` for Tree-sitter or JavaParser-sidecar to fill:
     - `class_name`, `method_name`, `layer`, `calls`, line ranges.
   - Enhance Neo4j to store `CALLS` edges and module/layer tags.

2. **Impact analysis tools**:
   - New MCP tools, e.g.:
     - `impact_of_method(class, method)` — uses graph queries.
     - `trace_path(from, to)` — returns call chains.

3. **Index metadata**:
   - Store last indexed commit/time to avoid re-indexing on every startup.

4. **Multi-repo support**:
   - Extend `agent_config.yaml` to support multiple repositories with separate Qdrant collections or a `repo` field in payloads.

5. **Non-Java languages**:
   - Add language-specific parsers for other languages and tag chunks by language.

---

## 9. Summary for AI Assistants

When using this project as context:

- **Assume there is no backend LLM**. All reasoning should be done by the external LLM (e.g. Copilot).
- The backend’s role is:
  - Indexing and storing code & relationships in Qdrant and Neo4j.
  - Exposing well-defined MCP tools to query:
    - Semantic code matches (`search_code`).
    - Call graph relationships (`find_callers`, `find_callees`).
- Any explanations, refactorings, or feature proposals should:
  - Use these tools to fetch relevant code/data.
  - Avoid assuming extra backend intelligence beyond those tools.
- The current parser is file-level, so method-level relationships are limited but the data model is prepared for future AST-based enrichment.

This document should be treated as the **canonical description of architecture and intent** for this codebase.
```

If you want, I can also generate a shorter “Prompt primer” MD file specifically tailored for how you’d ask Copilot/Claude to use these tools optimally.

