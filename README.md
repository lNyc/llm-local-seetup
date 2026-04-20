# llm-local-setup

> Self-hosted LLM stack with observability, built for consumer GPUs.

Runs a full local inference pipeline — model server, API proxy, chat UI, vector store, knowledge graph, and GPU-aware observability — entirely in Docker. No cloud dependencies, no API keys required for inference.

---

## Stack

| Layer | Service | Role |
|---|---|---|
| **Backend** | [KoboldCPP (hybrid-cpp)](https://github.com/Elypha/hybrid-cpp) | GGUF model server with hybrid/recurrent checkpoint support |
| **Middleware** | [LiteLLM](https://github.com/BerriAI/litellm) | OpenAI-compatible proxy and model router |
| **Frontend** | [Open WebUI](https://github.com/open-webui/open-webui) | Chat interface |
| **Observability** | [OpenLIT](https://github.com/openlit/openlit) | LLM observability dashboard |
| **Telemetry** | [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) | Trace/metric/log pipeline |
| **Vector store** | [Qdrant](https://qdrant.tech/) | Semantic similarity search and RAG document storage |
| **Graph store** | [Neo4j](https://neo4j.com/) | Knowledge graph — entities, relationships, multi-hop traversal |
| **Database** | [ClickHouse](https://clickhouse.com/) | Telemetry storage backend |

---

## Architecture

```
                         ┌──────────────────────────────────────────────────┐
                         │                    llm-stack                      │
                         │                                                   │
          Browser        │   ┌──────────┐      ┌──────────────────┐         │
        ──────────────►  │   │ Open WebUI│────► │     LiteLLM      │         │
         :2601           │   └─────┬────┘      └────────┬─────────┘         │
                         │         │ (RAG)              │                    │
          Browser        │         ▼                    ▼                    │
        ──────────────►  │   ┌──────────┐    ┌──────────────────┐           │
         :3000           │   │  Qdrant  │    │   KoboldCPP      │           │
                         │   └──────────┘    │  (hybrid-cpp)    │           │
                         │                   └────────┬─────────┘           │
                         │   ┌──────────┐             │ CUDA                │
                         │   │  Neo4j   │             ▼                     │
                         │   └──────────┘            GPU                    │
                         │         ↑                  │                     │
                         │    (pending agent layer)   │                     │
                         │                            │                     │
                         │   ┌──────────┐             │                     │
                         │   │  OpenLIT │◄──────────────────────────┐       │
                         │   └────┬─────┘                           │       │
                         │        ▼                                 │       │
                         │   ┌──────────┐   ┌──────────────────┐   │       │
                         │   │ClickHouse│◄──│  OTEL Collector  │◄──┘       │
                         │   └──────────┘   └────────▲─────────┘           │
                         │                           │                      │
                         │                  ┌────────┴────────┐             │
                         │                  │  GPU Collector  │             │
                         │                  └─────────────────┘             │
                         └──────────────────────────────────────────────────┘
```

Inference path: **Open WebUI → LiteLLM → KoboldCPP → GPU**

RAG path: **Open WebUI → Qdrant** (vector similarity search)

Graph path: **Agent → Neo4j** (knowledge graph traversal — pending agent layer)

Observability path: **KoboldCPP + GPU collector → OTEL Collector → ClickHouse → OpenLIT**

---

## Hardware

Tested on:

| Component | Spec |
|---|---|
| CPU | AMD Ryzen 9 3900X |
| RAM | 48 GB DDR4 |
| GPU | NVIDIA RTX 3080 10GB (LHR) |

VRAM at runtime: ~8.4 GB for KoboldCPP, ~9 GB total system usage.

> **Different GPU?** See [`docs/hardware-tuning.md`](docs/hardware-tuning.md) for context size and KV quantization tradeoffs across GPU tiers.

---

## Prerequisites

- Docker with the Compose plugin
- NVIDIA drivers + [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- A compatible GGUF model file (see [`docs/model-selection.md`](docs/model-selection.md))

---

## Quick Start

**1. Clone the repo**
```bash
git clone https://github.com/lNyc/llm-local-setup.git
cd llm-local-setup
```

**2. Create the shared Docker network**
```bash
docker network create llm-stack
```

**3. Configure environment**
```bash
cp .env.example .env
```
Open `.env` and fill in the required values — at minimum `CLICKHOUSE_PASSWORD`, `NEXTAUTH_SECRET`, and `NEO4J_PASSWORD`. See [`docs/configuration.md`](docs/configuration.md) for a full variable reference.

**4. Drop in your model**
```bash
# Place your GGUF file here:
Infrastructure/backend/models/<your-model>.gguf
```
Then update the `--model` flag in `Infrastructure/backend/koboldocpp_Elypha/koboldcpp.yml` to match the filename.

**5. Build and start**
```bash
docker compose up -d --build
```
The first run builds the KoboldCPP image and pulls all other images. This takes a few minutes.

**6. Open the interfaces**

| Interface | URL |
|---|---|
| Chat (Open WebUI) | http://localhost:2601 |
| Observability (OpenLIT) | http://localhost:3000 |

---

## Services & Ports

| Service | Host Port | Internal Address | Description |
|---|---|---|---|
| Open WebUI | `2601` | `openwebui:8080` | Chat interface |
| OpenLIT | `3000` | `openlit:3000` | Observability dashboard |
| LiteLLM | — | `litellm:4000` | API proxy (internal only) |
| KoboldCPP | — | `model-server:5001` | Model inference (internal only) |
| Qdrant | — | `qdrant:6333` | Vector store HTTP (internal only) |
| Neo4j | — | `neo4j:7687` | Knowledge graph Bolt (internal only) |
| ClickHouse HTTP | — | `clickhouse:8123` | DB HTTP API (internal only) |
| ClickHouse TCP | — | `clickhouse:9000` | DB native protocol (internal only) |
| OTEL Collector gRPC | — | `otel-collector:4317` | Telemetry ingress (internal only) |
| OTEL Collector HTTP | — | `otel-collector:4318` | Telemetry ingress (internal only) |

LiteLLM, KoboldCPP, Qdrant, Neo4j, ClickHouse, and the OTEL collector are intentionally not exposed to the host — all traffic flows through the `llm-stack` Docker network.

---

## Stopping the Stack

```bash
docker compose down
```

To also remove volumes (wipes all data including chat history, vector store, knowledge graph, and telemetry):
```bash
docker compose down -v
```

---

## Documentation

- [`docs/architecture.md`](docs/architecture.md) — why each layer exists and how they connect
- [`docs/hardware-tuning.md`](docs/hardware-tuning.md) — VRAM budget, context size, KV quantization by GPU tier
- [`docs/model-selection.md`](docs/model-selection.md) — compatible model families, quantization formats, ChatML requirement
- [`docs/observability.md`](docs/observability.md) — GPU metrics, inference traces, OpenLIT dashboard walkthrough
- [`docs/configuration.md`](docs/configuration.md) — full `.env` variable reference

> 🤖 Generated by [Claude](https://claude.ai) — review before committing.(I did not review)