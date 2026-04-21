# Extraction Pipeline — Plan of Action

> This document captures the architectural decisions made for the extraction pipeline that feeds Neo4j and Qdrant. It is intended as the reference for implementation and future contributors.

---

## Context

The current stack has Neo4j and Qdrant running and reachable on `llm-stack`, but both stores are empty. The inference layer (KoboldCPP → LiteLLM → Open WebUI) is functional. The missing piece is the ingestion side: nothing reads projects, extracts structure and semantics, generates embeddings, or writes the results into the two stores.

This pipeline is the prerequisite for the orchestration layer. Agents that traverse Neo4j and query Qdrant need real content to be useful. Building the agent layer before filling the stores produces a well-wired shell with nothing inside.

---

## Architecture Overview

The pipeline runs in three distinct layers, each feeding the same two downstream stores.

```
Code files
    └── Fraunhofer AISEC CPG ──────────────────────► Neo4j
                                                      (structural layer)

Docs / READMEs / ADRs / PDFs
    └── Neo4j LLM Graph Builder ──────────────────► Neo4j
         (via LiteLLM → 9B model)                    (semantic layer)

All content (code + docs)
    └── KoboldCPP --embeddingsmodel ──────────────► Qdrant
         (Qwen3-Embedding-0.6B Q8_0)                 (vector layer)
```

Neo4j and Qdrant are linked by a shared `neo4j_id` key. Every Qdrant payload carries the ID of its corresponding Neo4j node, enabling hybrid retrieval via `neo4j-graphrag-python`.

---

## Schema

### The Non-Negotiable: `project_id` Namespacing

Every Neo4j node carries a mandatory `project_ref` property pointing to the UUID of its parent `Project` node. All agent queries scope to a project boundary first. Cross-project edges are only created by an explicit separate pass, never during ingestion.

This prevents graph pollution as projects accumulate. Without namespacing, `AuthService` from three different codebases becomes candidates for the same node after enough projects are ingested, producing incoherent subgraphs that cause LLM hallucination through faithfully-reported corrupted context.

```cypher
-- All queries must be scoped. This is the wrong pattern:
MATCH (f:Function {name: "validate_token"})-[:CALLS]->(g:Function)
RETURN g

-- This is the correct pattern:
MATCH (f:Function {name: "validate_token", project_ref: $projectId})
      -[:CALLS]->(g:Function {project_ref: $projectId})
RETURN g
```

The schema design document must be completed before any pipeline code is written.

---

### Project Registry

The `Project` node is the anchor for all other nodes. Its `id` is a stable UUID that never changes. All mutable identifiers — repository URL, display name, remote host — are properties that can be updated without touching the rest of the graph.

```cypher
(:Project {
    id:                   "uuid",          // stable, never changes — FK for all child nodes
    slug:                 "auth-service",  // normalized, URL-safe, used in queries
    name:                 "Auth Service",  // display name, used by LLM in prompts
    project_id:           "lnyc:auth-service",  // derived from git remote, can change
    remote_url:           "github.com/lnyc/auth-service",
    last_ingested_commit: "a3f9c2d",       // HEAD at last successful ingestion run
    last_ingested_at:     datetime(),
    registered_at:        datetime()
})
```

`slug` and `id` are immutable after creation. `project_id`, `remote_url`, and `name` can change (repository migration, rename) without affecting graph topology.

---

### Base Properties — Every Node

All node types inherit these properties regardless of label:

```
id              // UUID — stable identity, never changes
name            // human-readable, used by LLM
project_ref     // UUID of parent Project node
created_at      // datetime — when this node was first ingested
updated_at      // datetime — last property change
deprecated_at   // datetime — null if active, set when entity is removed
source_layer    // "cpg" | "llm" | "git" | "manual"
last_seen_at    // datetime — last ingestion run that confirmed this node exists
```

---

### Base Properties — Every Edge

All relationship types carry these properties:

```
confidence      // 0.0–1.0 (1.0 = structural fact from AST, <1.0 = inferred)
source_layer    // "cpg" | "llm" | "git" | "manual"
created_at      // datetime
last_seen_at    // datetime — last ingestion that confirmed this edge still exists
```

`last_seen_at` on edges is critical for detecting deleted relationships. A `CALLS` edge with a stale `last_seen_at` indicates the call was removed; without this field, you cannot distinguish a deleted relationship from an ingestion gap.

---

### Node Types

#### Structural Layer (produced by Fraunhofer AISEC CPG — confidence 1.0)

| Label | Description | Natural key for upsert |
|---|---|---|
| `Directory` | Filesystem folder | `project_ref + path` |
| `File` | Source file | `project_ref + path` |
| `Module` | Logical code unit / package | `project_ref + qualified_name` |
| `Class` | Class or struct definition | `project_ref + file_path + class_name` |
| `Interface` | Interface, trait, or protocol | `project_ref + file_path + name` |
| `Function` | Function or method | `project_ref + file_path + function_name + normalized_body_hash` |
| `Variable` | Field, parameter, or global | `project_ref + file_path + name + scope` |

**Function natural key note:** `normalized_body_hash` is SHA256 of the function body after stripping trailing whitespace per line, normalizing line endings, and collapsing consecutive blank lines. This makes the key stable across file moves (caught by git rename detection) and pure formatting changes, while correctly treating substantial rewrites as new entities.

#### Semantic Layer (produced by Neo4j LLM Graph Builder — confidence varies)

| Label | Description |
|---|---|
| `Document` | Source document (README, ADR, PDF, transcript) |
| `Chunk` | Text fragment with embedding, child of Document |
| `Concept` | High-level idea or domain concept extracted from prose |
| `ArchitectureDecision` | Architecture decision record or design rationale |

#### Change Tracking

| Label | Description |
|---|---|
| `ChangeEvent` | Records a specific property mutation on a node |

```cypher
(:ChangeEvent {
    id:          "uuid",
    timestamp:   datetime(),
    change_type: "UPDATE" | "DEPRECATE" | "MOVE",
    field:       "name",
    old_value:   "Google OAuth v1",
    new_value:   "Google OAuth v2"
})
```

#### Community (schema present — clustering deferred)

| Label | Description |
|---|---|
| `Community` | Cluster of densely connected nodes produced by Leiden algorithm |

The `Community` node type is defined in the schema now to avoid a migration later. The Leiden clustering pass and LLM summary generation that populates these nodes is deferred until the graph has sufficient density across multiple projects to produce meaningful communities.

```cypher
(:Community {
    id:         "uuid",
    name:       "Authentication Layer",   // LLM-generated, populated by clustering pass
    summary:    "...",                    // LLM-generated, populated by clustering pass
    level:      1,                        // Leiden hierarchy level
    created_at: datetime()
})
```

#### Future Extension Layers (schema reserved — not implemented in v1)

These node types are reserved in the schema to avoid conflicts when implemented. No ingestion code targets them in v1.

**ML projects:**
`Dataset`, `Experiment`, `MLModel`, `Checkpoint`, `Metric`, `TrainingPipeline`

**Infrastructure:**
`Service`, `Endpoint`, `APIContract`, `ConfigFile`, `EnvVariable`

---

### Relationship Types

#### Structural (confidence 1.0 — from Fraunhofer CPG)

| Relationship | Direction | Meaning |
|---|---|---|
| `CONTAINS` | Directory→File, File→Class, Class→Function | Hierarchy containment |
| `IMPORTS` | File→File | Import / require statement |
| `CALLS` | Function→Function | Call graph edge |
| `INHERITS_FROM` | Class→Class | Inheritance |
| `IMPLEMENTS` | Class→Interface | Interface implementation |
| `DECLARES` | Class→Function/Variable | Member declaration |
| `DEPENDS_ON` | Module→Module | Module-level dependency |
| `RETURNS` | Function→Type | Return type reference |
| `USES` | Function→Variable | Variable usage |

#### Lexical (confidence 1.0 — from Neo4j LLM Graph Builder)

| Relationship | Direction | Meaning |
|---|---|---|
| `CONTAINS` | Document→Chunk | Document chunking |
| `NEXT_CHUNK` | Chunk→Chunk | Sequential chunk order |
| `DESCRIBES` | Document→File | Document documents a specific file |
| `HAS_ENTITY` | Chunk→(any node) | Chunk references an entity |

#### Semantic (confidence varies — from Neo4j LLM Graph Builder)

| Relationship | Direction | Meaning |
|---|---|---|
| `IMPLEMENTS_CONCEPT` | Function/Class→Concept | Code realises a concept |
| `RATIONALE_FOR` | ArchitectureDecision→Function/Module | Explains design choice |
| `RELATED_TO` | Concept→Concept | General semantic link |
| `DOCUMENTED_BY` | Function/Class→Chunk | Links code entity to its documentation chunk |
| `OWNED_BY` | Module/File→Team/Person | Ownership / maintainership |

#### Community (deferred — populated by clustering pass)

| Relationship | Direction | Meaning |
|---|---|---|
| `SUMMARIZES` | Community→(any node) | Community covers this node |
| `CHILD_OF` | Community→Community | Leiden hierarchy level |

#### Change Tracking

| Relationship | Direction | Meaning |
|---|---|---|
| `HAS_CHANGE` | (any node)→ChangeEvent | Audit trail entry |
| `SUPERSEDED_BY` | (any node)→(same label) | Version chain |
| `MOVED_FROM` | (any node)→(same label) | Git rename detected |

#### Cross-Project (deferred — populated by explicit second pass)

| Relationship | Direction | Meaning |
|---|---|---|
| `DEPENDS_ON` | Service→Service | Cross-project service dependency |
| `SHARES_PATTERN_WITH` | Module→Module | Similar implementation, explicit |
| `CONSUMED_BY` | Endpoint→Service | Which service calls which API |

---

### Incremental Ingestion and Change Detection

The pipeline uses git diff as the primary signal for detecting changes between ingestion runs. The last ingested commit is stored on the `Project` node and updated at the end of each successful run.

```
1. Read Project.last_ingested_commit from Neo4j
2. git diff --find-renames -M85% {last_commit} HEAD --name-status
3. For each changed file:
     ADDED    → Fraunhofer CPG on file → CREATE nodes
     DELETED  → MATCH nodes by file_path → SET deprecated_at
     RENAMED  → MATCH nodes by old path → UPDATE source_file, CREATE ChangeEvent
     MODIFIED → Fraunhofer CPG on file → UPSERT nodes by natural key
4. SET Project.last_ingested_commit = current HEAD
```

**Upsert logic is mandatory.** The pipeline must always attempt to find an existing node by its natural key before creating a new one. Blind `CREATE` is forbidden — it produces duplicate nodes on re-ingestion.

**Git rename detection threshold:** `-M85%` requires 85% content similarity to classify a change as a rename rather than delete+add. This is stricter than git's default (50%) to reduce false positives on large refactors.

**Semantic pass follows the same diff.** Only documents that changed since `last_ingested_commit` are re-processed by Neo4j LLM Graph Builder. This keeps the semantic pass fast on incremental runs.

---

## Layer 1 — AST Extraction

### Tool: Fraunhofer AISEC CPG

**Repository:** https://github.com/Fraunhofer-AISEC/cpg

**What it does:** Parses source code into a Code Property Graph (CPG) — a supergraph combining the AST, control flow graph, data flow graph, and call graph. Writes directly to Neo4j via the `cpg-neo4j` submodule.

**Why this tool:**
- Active development since ~2017, Apache 2.0 licensed. PRs and dev meetings through January 2026 confirm it is not abandoned despite infrequent versioned releases.
- Language coverage: C/C++, Java, Go, Python, TypeScript, Ruby, and any language that compiles to LLVM-IR.
- Ships a `cpg-neo4j` CLI tool that takes source paths and a Neo4j connection and writes the full graph directly. No translation layer required.
- Produces richer structural data than Tree-sitter alone: control flow edges, data flow edges, and cross-language resolution are included in the same graph.

**What it does not do:** Semantic enrichment. It extracts structure, not meaning. A function named `process` tells you nothing about what it does — that is handled by Layer 2.

**Confidence tagging:** All edges produced by Fraunhofer CPG are structural facts derived from the source — equivalent to `EXTRACTED` (confidence 1.0) in Graphify's taxonomy. They are not inferences.

**Rejected alternatives:**

| Tool | Reason rejected |
|---|---|
| Graphify | One week old at evaluation, flat schema, semantic pass hardcoded to Claude API |
| Joern | Moved away from Neo4j to custom in-memory DB (OverflowDB) |
| Sourcetrail | Discontinued 2021 |
| jQAssistant | Java/JVM-first; weaker Python/Go coverage |
| Custom Tree-sitter wrapper | Fraunhofer already solved this more completely |

**Language modules to enable:**

```kotlin
implementation("de.fraunhofer.aisec", "cpg-core", cpgVersion)
implementation("de.fraunhofer.aisec", "cpg-language-go", cpgVersion)
implementation("de.fraunhofer.aisec", "cpg-language-python", cpgVersion)
implementation("de.fraunhofer.aisec", "cpg-language-java", cpgVersion)
implementation("de.fraunhofer.aisec", "cpg-language-typescript", cpgVersion)
```

---

## Layer 2 — Semantic Extraction

### Tool: Neo4j LLM Graph Builder

**Repository:** https://github.com/neo4j-labs/llm-graph-builder

**What it does:** Ingests unstructured documents (PDFs, markdown, web pages, YouTube transcripts), chunks them, computes embeddings, extracts entity and relationship triples via LLM, and writes a lexical graph (Document → Chunk) and entity graph into Neo4j. Includes built-in entity resolution to merge near-duplicate nodes.

**Why this tool:**
- Maintained by Neo4j Labs, not a community fork. Long-term support is credible.
- Docker Compose deployable, configurable schema, supports local LLMs via the Ollama endpoint format — wired to the existing LiteLLM proxy instead of a cloud provider.
- Handles the content types Fraunhofer CPG ignores: READMEs, architecture decision records, design documents, PDFs, meeting transcripts.
- The entity resolver (merging `AuthService`, `auth-svc`, and `authentication module` into one node) is built in, not a custom implementation task.

**LLM backend:** The existing 9B model via LiteLLM. Semantic extraction runs as low-priority batched jobs during idle time, not inline or blocking the inference path. GPU is shared — extraction and interactive inference do not run simultaneously.

**What it does not do:** AST extraction, code structure, call graphs. Those are handled by Layer 1.

**`project_id` enforcement:** The LLM Graph Builder's schema is configurable. `project_id` must be added as a mandatory node property in the schema config before the first run. The tool does not enforce namespacing by default.

---

## Layer 3 — Embeddings

### Model: Qwen3-Embedding-0.6B

**GGUF source:** `Qwen/Qwen3-Embedding-0.6B-GGUF` on HuggingFace

**Quantization: Q8_0** (639 MB)

**Why Q8_0 over FP16:**
- The Ryzen 9 3900X (Zen 2) supports AVX2 but not native FP16 compute. FP16 on this CPU upcasts to FP32 for arithmetic, making it slower than Q8_0.
- Q8_0 uses INT8 arithmetic, which AVX2 accelerates efficiently.
- For embedding models (single forward pass → normalized vector), Q8_0 is near-lossless. Quantization noise does not compound across autoregressive steps the way it does in generative models.
- The 560 MB savings over FP16 is irrelevant at 48 GB RAM, but the inference speed advantage is real.

**Why this model over alternatives:**

| Model | Params | Code | Prose | Verdict |
|---|---|---|---|---|
| nomic-embed-code | 7B | ✅ best | ❌ code-only | GPU required, infeasible |
| CodeRankEmbed-137M | 137M | ✅ good | ❌ | Code-only, needs companion |
| nomic-embed-text v1.5 | 137M | ⚠️ weak | ✅ | Prose-only, needs companion |
| **Qwen3-Embedding-0.6B** | **0.6B** | **✅ 75.41 MTEB-Code** | **✅ 61.82 MTEB-R** | **Single model, both domains** |

Qwen3-Embedding-0.6B scores higher than nomic-embed-text on English retrieval (61.82 vs ~53-55) and handles code retrieval in the same model. It supports 100+ languages including programming languages, has a 32K context window (relevant for large code nodes from Fraunhofer CPG), and supports Matryoshka dimensions for storage flexibility.

**Serving:** KoboldCPP's `--embeddingsmodel` flag loads a second GGUF alongside the main inference model. No additional service, no new container, no GPU usage. Embedding runs on CPU via the 3900X.

```yaml
# Addition to koboldcpp.yml
- --embeddingsmodel
- /models/Qwen3-Embedding-0.6B-Q8_0.gguf
```

**Note on the Elypha fork:** The current backend is the Elypha hybrid-cpp fork of KoboldCPP, used to correctly handle Qwen3.5 recurrent checkpoint state — a problem not yet fixed in upstream llama.cpp. The `--embeddingsmodel` flag is an upstream KoboldCPP feature that Elypha carries. When llama.cpp resolves native hybrid architecture support, the Dockerfile will migrate to upstream; no other stack components are affected by that migration.

**Rejected approach — cloud embedding APIs:** Cohere's free tier offers 1,000 API calls per month (insufficient for a single project ingestion run) and prohibits production use. All providers send data to external servers, breaking the local-first architecture. Local Q8_0 inference on a 0.6B model produces embeddings faster than a cloud API round-trip including network latency.

---

## Neo4j ↔ Qdrant Bridge

### Library: `neo4j-graphrag-python`

**Repository:** https://github.com/neo4j/neo4j-graphrag-python

The official Neo4j Python library ships a `QdrantNeo4jRetriever` that handles the join between Qdrant vector results and Neo4j nodes natively. A Qdrant similarity search returns vector matches with payloads containing `neo4j_id`; the retriever uses those IDs to fetch the corresponding Neo4j nodes and traverse their graph context.

**Linking key:** Every Qdrant point carries a payload field `neo4j_id` matching the Neo4j node's namespaced ID (`{project_id}::{local_id}`). This is the only custom piece — the ingestion script that writes to Qdrant must populate this field correctly.

```python
from neo4j_graphrag.retrievers import QdrantNeo4jRetriever

retriever = QdrantNeo4jRetriever(
    driver=neo4j_driver,
    client=QdrantClient(url="http://qdrant:6333"),
    collection_name="project_chunks",
    id_property_external="neo4j_id",
    id_property_neo4j="id",
    embedder=embedder,  # points at KoboldCPP /v1/embeddings
)
```

---

## What You Own (Glue Code)

The three tools handle their respective layers. The custom code is minimal:

1. **Ingestion orchestration script** — takes a project path and project slug, registers the project in Neo4j if new, runs Fraunhofer CPG (writes structural nodes), runs Neo4j LLM Graph Builder (writes semantic nodes), chunks content, calls `/v1/embeddings` (KoboldCPP), writes vectors to Qdrant with `neo4j_id` payload. Updates `Project.last_ingested_commit` on success.

2. **`project_ref` injection** — Fraunhofer CPG and Neo4j LLM Graph Builder both need `project_ref` added to their output nodes. This is a schema configuration for the LLM Graph Builder and a post-processing step (or CPG plugin) for Fraunhofer.

3. **Git diff driver** — reads `Project.last_ingested_commit` from Neo4j, runs `git diff --find-renames -M85%`, routes changed files to the correct pipeline action (create / deprecate / update / upsert).

4. **Cross-project relationship pass** — runs separately after ≥2 projects are ingested. Creates typed cross-project edges (`SHARES_PATTERN_WITH`, `CONSUMED_BY`, etc.) based on explicit rules. Not part of the initial ingestion pipeline.

5. **Leiden clustering trigger** — a standalone script that runs the Neo4j GDS Leiden algorithm and calls the LLM to generate community summaries. Run manually when the graph has sufficient density. Populates `Community` nodes and `SUMMARIZES` edges. Not part of the ingestion pipeline.

---

## Build Order

Execute in sequence. Each step validates the next layer's dependencies before committing to it.

### Step 1 — Neo4j Constraints and Indexes
Apply uniqueness constraints and indexes derived from the schema. Enforce `id` uniqueness on all node labels. Enforce `slug` uniqueness on `Project`. Create composite indexes on `(project_ref, file_path, function_name)` for upsert lookups. Do not write ingestion code until constraints are verified.

### Step 2 — Fraunhofer CPG → Neo4j
Run Fraunhofer CPG on a single representative project. Validate:
- Neo4j write path works
- `project_id` is present on all nodes
- Call graph and data flow edges are present
- Cypher queries return expected structural results

### Step 3 — Neo4j LLM Graph Builder → Neo4j
Wire LLM Graph Builder to LiteLLM endpoint. Run on the same project's documentation. Validate:
- Entity resolution merges expected near-duplicates
- `project_id` is enforced in schema config
- Semantic nodes link correctly to structural nodes from Step 2

### Step 4 — Embeddings
Verify `--embeddingsmodel` flag in Elypha build. Add Qwen3-Embedding-0.6B Q8_0 to `koboldcpp.yml`. Validate:
- `/v1/embeddings` endpoint responds
- Vector dimensions match Qdrant collection config (1024)

### Step 5 — Qdrant Population
Write ingestion script. Chunk content, embed, write to Qdrant with `neo4j_id` payload. Validate:
- Qdrant collection populated
- Payload `neo4j_id` values match Neo4j node IDs exactly

### Step 6 — End-to-End Retrieval
Test `QdrantNeo4jRetriever` with a real query. Validate:
- Vector search returns relevant chunks
- Neo4j node fetch succeeds from returned IDs
- Graph traversal from retrieved nodes returns coherent context

### Step 7 — Second Project
Ingest a second project. Validate:
- No cross-project node contamination
- Scoped Cypher queries return only project-specific results
- Both projects independently queryable

Cross-project relationship pass is deferred until Step 7 is validated.

---

## Hardware Reference

| Component | Spec | Role in pipeline |
|---|---|---|
| CPU | AMD Ryzen 9 3900X (12c/24t) | Fraunhofer CPG, embedding inference, LLM Graph Builder |
| RAM | 48 GB DDR4 | Hybrid checkpoint slots, embedding model, pipeline buffers |
| GPU | RTX 3080 10GB | 9B inference model only — not used for embeddings |

The GPU is committed to the 9B inference model. All pipeline workloads (AST extraction, semantic extraction batches, embedding) run on CPU. Semantic extraction (Layer 2) and interactive inference are not scheduled simultaneously.

---

## Future Work

Once the extraction pipeline is validated, the following items are unblocked or deferred by design.

**Unblocked by the extraction pipeline:**
- **Orchestration layer** — agents can traverse Neo4j and query Qdrant with real content. Tool registration for SearXNG, Neo4j, Qdrant, and a code execution sandbox.

**Deferred by design — schema is ready, implementation is not:**
- **Community clustering pass** — run Neo4j GDS Leiden algorithm on the populated graph, generate LLM summaries per community, populate `Community` nodes and `SUMMARIZES` edges. Deferred until the graph has sufficient density across multiple projects.
- **Cross-project relationship pass** — explicit typed edges between projects (`DEPENDS_ON`, `SHARES_PATTERN_WITH`, `CONSUMED_BY`). Deferred until at least two projects are ingested and independently validated.
- **ML extension layer** — `Dataset`, `Experiment`, `MLModel`, `Checkpoint`, `Metric`, `TrainingPipeline` node types. Reserved in schema, no ingestion code targets them in v1.
- **Infrastructure extension layer** — `Service`, `Endpoint`, `APIContract`, `ConfigFile`, `EnvVariable` node types. Reserved in schema, no ingestion code targets them in v1.
- **Embedding similarity reconciliation** — secondary pass using vector similarity to detect function identity across large refactors that git rename detection misses. Deferred until real ingestion data shows where the gaps appear.
- **Codebase ingestion pipeline** — automated per-repository ingestion triggered by git hooks or a scheduler, rather than manual script invocation.