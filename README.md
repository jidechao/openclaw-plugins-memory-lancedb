# 🧠 openclaw-plugins-memory-lancedb

**Enhanced Long-Term Memory Plugin for [OpenClaw](https://github.com/openclaw/openclaw)**

Hybrid Retrieval (Vector + BM25) · Multi-Reranker (Jina / SiliconFlow / Lightweight) · Multi-Scope Isolation · Auto-Backup · Management CLI

[![OpenClaw Plugin](https://img.shields.io/badge/OpenClaw-Plugin-blue)](https://github.com/openclaw/openclaw)
[![LanceDB](https://img.shields.io/badge/LanceDB-Vectorstore-orange)](https://lancedb.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

**English** | [简体中文](README_CN.md)

---


## Why This Plugin?

The built-in `memory-lancedb` plugin in OpenClaw provides basic vector search. **openclaw-plugins-memory-lancedb** takes it much further:

| Feature | Built-in `memory-lancedb` | **openclaw-plugins-memory-lancedb** |
|---------|--------------------------|----------------------|
| Vector search | ✅ | ✅ |
| BM25 full-text search | ❌ | ✅ |
| Hybrid fusion (Vector + BM25) | ❌ | ✅ |
| Cross-encoder rerank (Jina) | ❌ | ✅ |
| SiliconFlow rerank | ❌ | ✅ |
| Lightweight rerank (cosine) | ❌ | ✅ |
| Recency boost | ❌ | ✅ |
| Time decay | ❌ | ✅ |
| Length normalization | ❌ | ✅ |
| MMR diversity | ❌ | ✅ |
| Multi-scope isolation | ❌ | ✅ |
| Noise filtering | ❌ | ✅ |
| Adaptive retrieval | ❌ | ✅ |
| Memory update (in-place) | ❌ | ✅ |
| Auto-backup (daily JSONL) | ❌ | ✅ |
| Embedding cache (LRU) | ❌ | ✅ |
| Management CLI | ❌ | ✅ |
| Session memory | ❌ | ✅ |
| Task-aware embeddings | ❌ | ✅ |
| Any OpenAI-compatible embedding | Limited | ✅ (OpenAI, Gemini, Jina, Ollama, etc.) |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   index.ts (Entry Point)                │
│  Plugin Registration · Config Parsing · Lifecycle Hooks │
│                · Auto-Backup Service                    │
└────────┬──────────┬──────────┬──────────┬───────────────┘
         │          │          │          │
    ┌────▼───┐ ┌────▼───┐ ┌───▼────┐ ┌──▼──────────┐
    │ store  │ │embedder│ │retriever│ │   scopes    │
    │ .ts    │ │ .ts    │ │ .ts    │ │    .ts      │
    └────────┘ └────────┘ └────────┘ └─────────────┘
         │                     │
    ┌────▼───┐           ┌─────▼──────────┐
    │migrate │           │noise-filter.ts │
    │ .ts    │           │adaptive-       │
    └────────┘           │retrieval.ts    │
                         └────────────────┘
    ┌─────────────┐   ┌──────────┐
    │  tools.ts   │   │  cli.ts  │
    │ (Agent API) │   │ (CLI)    │
    └─────────────┘   └──────────┘
```

### File Reference

| File | Purpose |
|------|---------|
| `index.ts` | Plugin entry point. Registers with OpenClaw Plugin API, parses config, mounts `before_agent_start` (auto-recall), `agent_end` (auto-capture), and `command:new` (session memory) hooks. Manages daily auto-backup service |
| `openclaw.plugin.json` | Plugin metadata + full JSON Schema config declaration (with `uiHints`) |
| `package.json` | NPM package info. Depends on `@lancedb/lancedb`, `openai`, `@sinclair/typebox` |
| `cli.ts` | CLI commands: `memory list/search/stats/delete/delete-bulk/export/import/reembed/migrate` |
| `src/store.ts` | LanceDB storage layer. Table creation / FTS indexing / Vector search / BM25 search / CRUD / bulk delete / stats |
| `src/embedder.ts` | Embedding abstraction. Compatible with any OpenAI-API provider (OpenAI, Gemini, Jina, Ollama, etc.). Supports task-aware embedding (`taskQuery`/`taskPassage`) and LRU caching (256 entries, 30-min TTL) |
| `src/retriever.ts` | Hybrid retrieval engine. Vector + BM25 → RRF fusion → Rerank (Jina / SiliconFlow / Lightweight) → Recency Boost → Importance Weight → Length Norm → Time Decay → Hard Min Score → Noise Filter → MMR Diversity |
| `src/scopes.ts` | Multi-scope access control. Supports `global`, `agent:<id>`, `custom:<name>`, `project:<id>`, `user:<id>` |
| `src/tools.ts` | Agent tool definitions: `memory_recall`, `memory_store`, `memory_forget`, `memory_update` (core) + `memory_stats`, `memory_list` (management, opt-in) |
| `src/noise-filter.ts` | Noise filter. Filters out agent refusals, meta-questions, greetings, and low-quality content |
| `src/adaptive-retrieval.ts` | Adaptive retrieval. Determines whether a query needs memory retrieval (skips greetings, slash commands, simple confirmations, emoji) |
| `src/migrate.ts` | Migration tool. Migrates data from the built-in `memory-lancedb` plugin to Pro |

---

## Core Features

### 1. Hybrid Retrieval

```
Query → embedQuery() ─┐
                       ├─→ RRF Fusion → Rerank → Recency Boost → Importance Weight → Filter
Query → BM25 FTS ─────┘
```

- **Vector Search**: Semantic similarity via LanceDB ANN (cosine distance)
- **BM25 Full-Text Search**: Exact keyword matching via LanceDB FTS index
- **Fusion Strategy**: Vector score as base, BM25 hits get a 15% boost (tuned beyond traditional RRF)
- **Configurable Weights**: `vectorWeight`, `bm25Weight`, `minScore`

### 2. Multi-Reranker Support

Three reranking strategies with graceful degradation:

| Mode | Provider | Model | Timeout | Best For |
|------|----------|-------|---------|----------|
| `cross-encoder` | Jina Reranker API | `jina-reranker-v2-base-multilingual` | 5s | Highest accuracy, multilingual |
| `siliconflow` | SiliconFlow API | `BAAI/bge-reranker-v2-m3` | 10s | Cost-effective, good Chinese support |
| `lightweight` | Local cosine similarity | — | — | Zero-latency, no API dependency |
| `none` | Disabled | — | — | Fastest, vector-only scoring |

- **Hybrid Scoring**: 60% cross-encoder/reranker score + 40% original fused score
- **Graceful Degradation**: On API failure or timeout, automatically falls back to lightweight cosine similarity reranking

### 3. Multi-Stage Scoring Pipeline

| Stage | Formula | Effect |
|-------|---------|--------|
| **Recency Boost** | `exp(-ageDays / halfLife) * weight` | Newer memories score higher (default: 14-day half-life, 0.10 weight) |
| **Importance Weight** | `score *= (0.7 + 0.3 * importance)` | importance=1.0 → ×1.0, importance=0.5 → ×0.85 |
| **Length Normalization** | `score *= 1 / (1 + 0.5 * log2(len/anchor))` | Prevents long entries from dominating (anchor: 500 chars) |
| **Time Decay** | `score *= 0.5 + 0.5 * exp(-ageDays / halfLife)` | Old entries gradually lose weight, floor at 0.5× (60-day half-life) |
| **Hard Min Score** | Discard if `score < threshold` | Removes irrelevant results (default: 0.35) |
| **MMR Diversity** | Cosine similarity > 0.85 → demoted | Prevents near-duplicate results |

### 4. Multi-Scope Isolation

- **Built-in Scopes**: `global`, `agent:<id>`, `custom:<name>`, `project:<id>`, `user:<id>`
- **Agent-Level Access Control**: Configure per-agent scope access via `scopes.agentAccess`
- **Default Behavior**: Each agent accesses `global` + its own `agent:<id>` scope

### 5. Adaptive Retrieval

- Skips queries that don't need memory (greetings, slash commands, simple confirmations, emoji)
- Forces retrieval for memory-related keywords ("remember", "previously", "last time", etc.)
- CJK-aware thresholds (Chinese: 6 chars vs English: 15 chars)

### 6. Noise Filtering

Filters out low-quality content at both auto-capture and tool-store stages:
- Agent refusal responses ("I don't have any information")
- Meta-questions ("do you remember")
- Greetings ("hi", "hello", "HEARTBEAT")

### 7. Auto-Backup

- Automatic daily JSONL backup of all memory entries
- Backup directory: `{dbPath}/../backups/memory-backup-{date}.jsonl`
- Retains the most recent 7 backups, older ones are automatically cleaned up
- First backup runs 1 minute after plugin start, then every 24 hours

### 8. Session Memory

- Triggered on `/new` command — saves previous session summary to LanceDB
- Disabled by default (OpenClaw already has native `.jsonl` session persistence)
- Configurable message count (default: 15)

### 9. Auto-Capture & Auto-Recall

- **Auto-Capture** (`agent_end` hook): Extracts preference/fact/decision/entity from conversations, deduplicates, stores up to 3 per turn
  - Optionally captures assistant messages too (`captureAssistant: true`)
- **Auto-Recall** (`before_agent_start` hook): Injects `<relevant-memories>` context (up to 3 entries) with source markers (vector/BM25/reranked)

### 10. Embedding Cache

- LRU cache with 256 entries and 30-minute TTL
- Reduces redundant API calls for repeated or similar embeddings
- Transparent to all embedding operations (query, passage, batch)

---

## Agent Tools

### Core Tools (always enabled)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `memory_recall` | `query` (required), `limit` (default 5, max 20), `scope`, `category` | Search memories using hybrid retrieval. Returns results with source markers |
| `memory_store` | `text` (required), `importance` (0–1, default 0.7), `category`, `scope` | Save a memory. Auto-deduplicates (similarity > 0.98), applies noise filtering |
| `memory_forget` | `query` or `memoryId`, `scope` | Delete a memory by search or direct ID. Returns candidate list on fuzzy match |
| `memory_update` | `memoryId` (required), `text`, `importance`, `category` | Update an existing memory in-place. Preserves original timestamp. Auto re-embeds on text change |

### Management Tools (opt-in via `enableManagementTools: true`)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `memory_stats` | `scope` | Get statistics: total count, scope distribution, category distribution, retrieval mode |
| `memory_list` | `limit` (default 10, max 50), `scope`, `category`, `offset` | List recent memories ordered by timestamp (descending) |

---

## Installation

1. Clone into your OpenClaw plugins directory:

```bash
cd /path/to/your/openclaw/workspace
git clone https://github.com/win4r/openclaw-plugins-memory-lancedb.git plugins/openclaw-plugins-memory-lancedb
cd plugins/openclaw-plugins-memory-lancedb
npm install
```

2. Add to your OpenClaw config (`openclaw.json`):

```json
{
  "plugins": {
    "load": {
      "paths": ["plugins/openclaw-plugins-memory-lancedb"]
    },
    "entries": {
      "openclaw-plugins-memory-lancedb": {
        "enabled": true,
        "config": {
          "embedding": {
            "apiKey": "${JINA_API_KEY}",
            "model": "jina-embeddings-v5-text-small",
            "baseURL": "https://api.jina.ai/v1",
            "dimensions": 1024,
            "taskQuery": "retrieval.query",
            "taskPassage": "retrieval.passage",
            "normalized": true
          }
        }
      }
    },
    "slots": {
      "memory": "openclaw-plugins-memory-lancedb"
    }
  }
}
```

3. Restart the gateway:

```bash
openclaw gateway restart
```

> **Note:** If you previously used the built-in `memory-lancedb`, disable it when enabling this plugin. Only one memory plugin can be active at a time.

---

## Configuration

<details>
<summary><strong>Full Configuration Example (click to expand)</strong></summary>

```json
{
  "embedding": {
    "apiKey": "${JINA_API_KEY}",
    "model": "jina-embeddings-v5-text-small",
    "baseURL": "https://api.jina.ai/v1",
    "dimensions": 1024,
    "taskQuery": "retrieval.query",
    "taskPassage": "retrieval.passage",
    "normalized": true
  },
  "dbPath": "~/.openclaw/memory/lancedb-pro",
  "autoCapture": true,
  "autoRecall": true,
  "captureAssistant": false,
  "enableManagementTools": false,
  "retrieval": {
    "mode": "hybrid",
    "vectorWeight": 0.7,
    "bm25Weight": 0.3,
    "minScore": 0.3,
    "rerank": "cross-encoder",
    "rerankApiKey": "jina_xxx",
    "rerankSFApiKey": "sf_xxx",
    "rerankModel": "jina-reranker-v2-base-multilingual",
    "rerankBaseURL": "https://api.siliconflow.cn/v1",
    "candidatePoolSize": 20,
    "recencyHalfLifeDays": 14,
    "recencyWeight": 0.1,
    "filterNoise": true,
    "lengthNormAnchor": 500,
    "hardMinScore": 0.35,
    "timeDecayHalfLifeDays": 60
  },
  "scopes": {
    "default": "global",
    "definitions": {
      "global": { "description": "Shared knowledge" },
      "agent:discord-bot": { "description": "Discord bot private" }
    },
    "agentAccess": {
      "discord-bot": ["global", "agent:discord-bot"]
    }
  },
  "sessionMemory": {
    "enabled": false,
    "messageCount": 15
  }
}
```

</details>

### Configuration Reference

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `embedding.apiKey` | string | — | **Required.** Embedding API key. Supports `${ENV_VAR}` placeholders |
| `embedding.model` | string | `text-embedding-3-small` | Embedding model name |
| `embedding.baseURL` | string | — | OpenAI-compatible API base URL |
| `embedding.dimensions` | number | auto | Vector dimensions (auto-detected or manual) |
| `embedding.taskQuery` | string | — | Task type for query embedding (e.g., `retrieval.query`) |
| `embedding.taskPassage` | string | — | Task type for passage embedding (e.g., `retrieval.passage`) |
| `embedding.normalized` | boolean | false | Request normalized embeddings (Jina v5) |
| `dbPath` | string | `~/.openclaw/memory/lancedb-pro` | Database storage path |
| `autoCapture` | boolean | `true` | Auto-capture memories from conversations |
| `autoRecall` | boolean | `true` | Auto-recall relevant memories before agent starts |
| `captureAssistant` | boolean | `false` | Also capture assistant messages (not just user) |
| `enableManagementTools` | boolean | `false` | Enable `memory_stats` and `memory_list` tools |
| `retrieval.mode` | string | `hybrid` | `"hybrid"` or `"vector"` |
| `retrieval.vectorWeight` | number | 0.7 | Vector score weight in fusion |
| `retrieval.bm25Weight` | number | 0.3 | BM25 score weight in fusion |
| `retrieval.minScore` | number | 0.3 | Minimum score threshold (pre-rerank) |
| `retrieval.rerank` | string | `cross-encoder` | `"cross-encoder"`, `"siliconflow"`, `"lightweight"`, or `"none"` |
| `retrieval.rerankApiKey` | string | — | Jina Reranker API key |
| `retrieval.rerankSFApiKey` | string | — | SiliconFlow API key |
| `retrieval.rerankModel` | string | per-provider | Reranker model name |
| `retrieval.rerankBaseURL` | string | per-provider | Custom rerank API base URL |
| `retrieval.candidatePoolSize` | number | 20 | Number of candidates before reranking |
| `retrieval.recencyHalfLifeDays` | number | 14 | Recency boost half-life (0 = disable) |
| `retrieval.recencyWeight` | number | 0.1 | Recency boost weight |
| `retrieval.filterNoise` | boolean | `true` | Enable noise filtering on results |
| `retrieval.lengthNormAnchor` | number | 500 | Length normalization anchor in chars (0 = disable) |
| `retrieval.hardMinScore` | number | 0.35 | Hard minimum score cutoff (post-pipeline) |
| `retrieval.timeDecayHalfLifeDays` | number | 60 | Time decay half-life (0 = disable) |

### Embedding Providers

This plugin works with **any OpenAI-compatible embedding API**:

| Provider | Model | Base URL | Dimensions |
|----------|-------|----------|------------|
| **Jina** (recommended) | `jina-embeddings-v5-text-small` | `https://api.jina.ai/v1` | 1024 |
| **OpenAI** | `text-embedding-3-small` | `https://api.openai.com/v1` | 1536 |
| **Google Gemini** | `gemini-embedding-001` | `https://generativelanguage.googleapis.com/v1beta/openai/` | 3072 |
| **Ollama** (local) | `nomic-embed-text` | `http://localhost:11434/v1` | 768 |

---

## CLI Commands

```bash
# List memories
openclaw memory list [--scope global] [--category fact] [--limit 20] [--json]

# Search memories
openclaw memory search "query" [--scope global] [--limit 10] [--json]

# View statistics
openclaw memory stats [--scope global] [--json]

# Delete a memory by ID (supports 8+ char prefix)
openclaw memory delete <id>

# Bulk delete with filters
openclaw memory delete-bulk --scope global [--before 2025-01-01] [--dry-run]

# Export / Import
openclaw memory export [--scope global] [--output memories.json]
openclaw memory import memories.json [--scope global] [--dry-run]

# Re-embed all entries with a new model
openclaw memory reembed --source-db /path/to/old-db [--batch-size 32] [--skip-existing]

# Migrate from built-in memory-lancedb
openclaw memory migrate check [--source /path]
openclaw memory migrate run [--source /path] [--dry-run] [--skip-existing]
openclaw memory migrate verify [--source /path]
```

---

## Database Schema

LanceDB table `memories`:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Primary key |
| `text` | string | Memory text (FTS indexed) |
| `vector` | float[] | Embedding vector |
| `category` | string | `preference` / `fact` / `decision` / `entity` / `other` |
| `scope` | string | Scope identifier (e.g., `global`, `agent:main`) |
| `importance` | float | Importance score 0–1 |
| `timestamp` | int64 | Creation timestamp (ms) |
| `metadata` | string (JSON) | Extended metadata |

---

## Dependencies

| Package | Purpose |
|---------|---------|
| `@lancedb/lancedb` ≥0.26.2 | Vector database (ANN + FTS) |
| `openai` ≥6.21.0 | OpenAI-compatible Embedding API client |
| `@sinclair/typebox` 0.34.48 | JSON Schema type definitions (tool parameters) |

---

## License

MIT

---