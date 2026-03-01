# 🧠 openclaw-plugins-memory-lancedb

**[OpenClaw](https://github.com/openclaw/openclaw) 增强型 LanceDB 长期记忆插件**

混合检索（Vector + BM25）· 多 Reranker（Jina / SiliconFlow / 轻量级）· 多 Scope 隔离 · 自动备份 · 管理 CLI

[![OpenClaw Plugin](https://img.shields.io/badge/OpenClaw-Plugin-blue)](https://github.com/openclaw/openclaw)
[![LanceDB](https://img.shields.io/badge/LanceDB-Vectorstore-orange)](https://lancedb.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

[English](README.md) | **简体中文**

---

## 为什么需要这个插件？

OpenClaw 内置的 `memory-lancedb` 插件仅提供基本的向量搜索。**openclaw-plugins-memory-lancedb** 在此基础上进行了全面升级：

| 功能 | 内置 `memory-lancedb` | **openclaw-plugins-memory-lancedb** |
|------|----------------------|----------------------|
| 向量搜索 | ✅ | ✅ |
| BM25 全文检索 | ❌ | ✅ |
| 混合融合（Vector + BM25） | ❌ | ✅ |
| 跨编码器 Rerank（Jina） | ❌ | ✅ |
| SiliconFlow Rerank | ❌ | ✅ |
| 轻量级 Rerank（cosine） | ❌ | ✅ |
| 时效性加成 | ❌ | ✅ |
| 时间衰减 | ❌ | ✅ |
| 长度归一化 | ❌ | ✅ |
| MMR 多样性去重 | ❌ | ✅ |
| 多 Scope 隔离 | ❌ | ✅ |
| 噪声过滤 | ❌ | ✅ |
| 自适应检索 | ❌ | ✅ |
| 记忆原地更新 | ❌ | ✅ |
| 自动备份（每日 JSONL） | ❌ | ✅ |
| Embedding 缓存（LRU） | ❌ | ✅ |
| 管理 CLI | ❌ | ✅ |
| Session 记忆 | ❌ | ✅ |
| Task-aware Embedding | ❌ | ✅ |
| 任意 OpenAI 兼容 Embedding | 有限 | ✅（OpenAI、Gemini、Jina、Ollama 等） |

---

## 架构概览

```
┌─────────────────────────────────────────────────────────┐
│                   index.ts (入口)                        │
│  插件注册 · 配置解析 · 生命周期钩子 · 自动备份服务        │
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

### 文件说明

| 文件 | 用途 |
|------|------|
| `index.ts` | 插件入口。注册到 OpenClaw Plugin API，解析配置，挂载 `before_agent_start`（自动回忆）、`agent_end`（自动捕获）、`command:new`（Session 记忆）等钩子。管理每日自动备份服务 |
| `openclaw.plugin.json` | 插件元数据 + 完整 JSON Schema 配置声明（含 `uiHints`） |
| `package.json` | NPM 包信息，依赖 `@lancedb/lancedb`、`openai`、`@sinclair/typebox` |
| `cli.ts` | CLI 命令实现：`memory list/search/stats/delete/delete-bulk/export/import/reembed/migrate` |
| `src/store.ts` | LanceDB 存储层。表创建 / FTS 索引 / Vector Search / BM25 Search / CRUD / 批量删除 / 统计 |
| `src/embedder.ts` | Embedding 抽象层。兼容 OpenAI API 的任意 Provider（OpenAI、Gemini、Jina、Ollama 等），支持 task-aware embedding（`taskQuery`/`taskPassage`）和 LRU 缓存（256 条目，30 分钟 TTL） |
| `src/retriever.ts` | 混合检索引擎。Vector + BM25 → RRF 融合 → Rerank（Jina / SiliconFlow / 轻量级）→ Recency Boost → Importance Weight → Length Norm → Time Decay → Hard Min Score → Noise Filter → MMR Diversity |
| `src/scopes.ts` | 多 Scope 访问控制。支持 `global`、`agent:<id>`、`custom:<name>`、`project:<id>`、`user:<id>` 等 Scope 模式 |
| `src/tools.ts` | Agent 工具定义：`memory_recall`、`memory_store`、`memory_forget`、`memory_update`（核心）+ `memory_stats`、`memory_list`（管理，需开启） |
| `src/noise-filter.ts` | 噪声过滤器。过滤 Agent 拒绝回复、Meta 问题、寒暄等低质量记忆 |
| `src/adaptive-retrieval.ts` | 自适应检索。判断 query 是否需要触发记忆检索（跳过问候、命令、简单确认等） |
| `src/migrate.ts` | 迁移工具。从旧版 `memory-lancedb` 插件迁移数据到 Pro 版 |

---

## 核心特性

### 1. 混合检索 (Hybrid Retrieval)

```
Query → embedQuery() ─┐
                       ├─→ RRF 融合 → Rerank → 时效加成 → 重要性加权 → 过滤
Query → BM25 FTS ─────┘
```

- **向量搜索**: 语义相似度搜索（cosine distance via LanceDB ANN）
- **BM25 全文搜索**: 关键词精确匹配（LanceDB FTS 索引）
- **融合策略**: Vector score 为基础，BM25 命中给予 15% 加成（非传统 RRF，经过调优）
- **可配置权重**: `vectorWeight`、`bm25Weight`、`minScore`

### 2. 多 Reranker 支持

三种重排策略，支持优雅降级：

| 模式 | 提供商 | 模型 | 超时 | 适用场景 |
|------|--------|------|------|----------|
| `cross-encoder` | Jina Reranker API | `jina-reranker-v2-base-multilingual` | 5s | 精度最高，多语言 |
| `siliconflow` | SiliconFlow API | `BAAI/bge-reranker-v2-m3` | 10s | 性价比高，中文支持好 |
| `lightweight` | 本地 cosine 相似度 | — | — | 零延迟，无需 API |
| `none` | 禁用 | — | — | 最快，仅向量评分 |

- **混合评分**: 60% cross-encoder/reranker 分数 + 40% 原始融合分
- **降级策略**: API 失败或超时时，自动回退到轻量级 cosine similarity rerank

### 3. 多层评分管线

| 阶段 | 公式 | 效果 |
|------|------|------|
| **时效加成** | `exp(-ageDays / halfLife) * weight` | 新记忆分数更高（默认半衰期 14 天，权重 0.10） |
| **重要性加权** | `score *= (0.7 + 0.3 * importance)` | importance=1.0 → ×1.0，importance=0.5 → ×0.85 |
| **长度归一化** | `score *= 1 / (1 + 0.5 * log2(len/anchor))` | 短条目（≤锚点）不受惩罚，长条目被降权（锚点：500 字符） |
| **时间衰减** | `score *= 0.5 + 0.5 * exp(-ageDays / halfLife)` | 旧条目逐渐降权，下限 0.5×（60 天半衰期） |
| **硬最低分** | 低于阈值直接丢弃 | 移除不相关结果（默认 0.35） |
| **MMR 多样性** | cosine 相似度 > 0.85 → 降级 | 防止近似重复结果 |

### 4. 多 Scope 隔离

- **内置 Scope 模式**: `global`、`agent:<id>`、`custom:<name>`、`project:<id>`、`user:<id>`
- **Agent 级访问控制**: 通过 `scopes.agentAccess` 配置每个 Agent 可访问的 Scope
- **默认行为**: Agent 可访问 `global` + 自己的 `agent:<id>` Scope

### 5. 自适应检索

- 跳过不需要记忆的 query（问候、slash 命令、简单确认、emoji）
- 强制检索含记忆相关关键词的 query（"remember"、"之前"、"上次"等）
- 支持 CJK 字符的更低阈值（中文 6 字符 vs 英文 15 字符）

### 6. 噪声过滤

在自动捕获和工具存储阶段同时生效：
- 过滤 Agent 拒绝回复（"I don't have any information"）
- 过滤 Meta 问题（"do you remember"）
- 过滤寒暄（"hi"、"hello"、"HEARTBEAT"）

### 7. 自动备份

- 每日自动将所有记忆导出为 JSONL 格式
- 备份目录：`{dbPath}/../backups/memory-backup-{date}.jsonl`
- 保留最近 7 个备份，旧备份自动清理
- 插件启动 1 分钟后执行首次备份，之后每 24 小时一次

### 8. Session 记忆

- `/new` 命令触发时可保存上一个 Session 的对话摘要到 LanceDB
- 默认关闭（`enabled: false`），因为 OpenClaw 已有原生 .jsonl 会话保存
- 开启会导致大段摘要污染检索质量，建议仅在需要语义搜索历史会话时开启
- 可配置消息数量（默认 15 条）

### 9. 自动捕获 & 自动回忆

- **Auto-Capture**（`agent_end` hook）: 从对话中提取 preference/fact/decision/entity，去重后存储（每次最多 3 条）
  - 可选择同时捕获 Assistant 消息（`captureAssistant: true`）
- **Auto-Recall**（`before_agent_start` hook）: 注入 `<relevant-memories>` 上下文（最多 3 条），标注来源（vector/BM25/reranked）

### 10. Embedding 缓存

- LRU 缓存，256 条目，30 分钟 TTL
- 减少重复或相似 Embedding 的 API 调用
- 对所有 Embedding 操作（query、passage、batch）透明生效

---

## Agent 工具

### 核心工具（始终启用）

| 工具 | 参数 | 说明 |
|------|------|------|
| `memory_recall` | `query`（必需）、`limit`（默认 5，最大 20）、`scope`、`category` | 使用混合检索搜索记忆，返回带来源标记的结果 |
| `memory_store` | `text`（必需）、`importance`（0-1，默认 0.7）、`category`、`scope` | 保存记忆。自动去重（相似度 > 0.98）、噪声过滤 |
| `memory_forget` | `query` 或 `memoryId`、`scope` | 删除记忆。支持搜索或直接 ID 删除，模糊匹配时返回候选列表 |
| `memory_update` | `memoryId`（必需）、`text`、`importance`、`category` | 原地更新记忆。保留原始时间戳，文本更新时自动重新嵌入 |

### 管理工具（需 `enableManagementTools: true`）

| 工具 | 参数 | 说明 |
|------|------|------|
| `memory_stats` | `scope` | 获取统计信息：总数、Scope 分布、类别分布、检索模式 |
| `memory_list` | `limit`（默认 10，最大 50）、`scope`、`category`、`offset` | 列出最近记忆，按时间戳降序 |

---

## 安装

1. 克隆到 OpenClaw 插件目录：

```bash
cd /path/to/your/openclaw/workspace
git clone https://github.com/win4r/openclaw-plugins-memory-lancedb.git plugins/openclaw-plugins-memory-lancedb
cd plugins/openclaw-plugins-memory-lancedb
npm install
```

2. 在 OpenClaw 配置中添加（`openclaw.json`）：

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

3. 重启 Gateway：

```bash
openclaw gateway restart
```

> **注意：** 如果之前使用了内置的 `memory-lancedb`，启用本插件时需同时禁用它。同一时间只能有一个 memory 插件处于活动状态。

---

## 配置

<details>
<summary><strong>完整配置示例（点击展开）</strong></summary>

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
      "global": { "description": "共享知识库" },
      "agent:discord-bot": { "description": "Discord 机器人私有" }
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

### 配置参考

| 键 | 类型 | 默认值 | 说明 |
|----|------|--------|------|
| `embedding.apiKey` | string | — | **必需。** Embedding API key，支持 `${ENV_VAR}` 占位符 |
| `embedding.model` | string | `text-embedding-3-small` | Embedding 模型名称 |
| `embedding.baseURL` | string | — | OpenAI 兼容 API 的 Base URL |
| `embedding.dimensions` | number | 自动检测 | 向量维度（自动检测或手动指定） |
| `embedding.taskQuery` | string | — | 查询任务类型（如 `retrieval.query`） |
| `embedding.taskPassage` | string | — | 文档任务类型（如 `retrieval.passage`） |
| `embedding.normalized` | boolean | false | 请求归一化嵌入（Jina v5） |
| `dbPath` | string | `~/.openclaw/memory/lancedb-pro` | 数据库存储路径 |
| `autoCapture` | boolean | `true` | 自动从对话中捕获记忆 |
| `autoRecall` | boolean | `true` | Agent 启动前自动召回相关记忆 |
| `captureAssistant` | boolean | `false` | 同时捕获 Assistant 消息（不仅限 User） |
| `enableManagementTools` | boolean | `false` | 启用 `memory_stats` 和 `memory_list` 工具 |
| `retrieval.mode` | string | `hybrid` | `"hybrid"` 或 `"vector"` |
| `retrieval.vectorWeight` | number | 0.7 | 融合时向量搜索的权重 |
| `retrieval.bm25Weight` | number | 0.3 | 融合时 BM25 搜索的权重 |
| `retrieval.minScore` | number | 0.3 | 最低分数阈值（rerank 前） |
| `retrieval.rerank` | string | `cross-encoder` | `"cross-encoder"`、`"siliconflow"`、`"lightweight"` 或 `"none"` |
| `retrieval.rerankApiKey` | string | — | Jina Reranker API key |
| `retrieval.rerankSFApiKey` | string | — | SiliconFlow API key |
| `retrieval.rerankModel` | string | 按提供商默认 | Reranker 模型名称 |
| `retrieval.rerankBaseURL` | string | 按提供商默认 | 自定义 rerank API base URL |
| `retrieval.candidatePoolSize` | number | 20 | Rerank 前的候选数量 |
| `retrieval.recencyHalfLifeDays` | number | 14 | 时效加成半衰期（0 = 禁用） |
| `retrieval.recencyWeight` | number | 0.1 | 时效加成权重 |
| `retrieval.filterNoise` | boolean | `true` | 对结果启用噪声过滤 |
| `retrieval.lengthNormAnchor` | number | 500 | 长度归一化锚点（字符数，0 = 禁用） |
| `retrieval.hardMinScore` | number | 0.35 | 硬最低分数（管线后） |
| `retrieval.timeDecayHalfLifeDays` | number | 60 | 时间衰减半衰期（0 = 禁用） |

### Embedding 提供商

本插件支持 **任意 OpenAI 兼容的 Embedding API**：

| 提供商 | 模型 | Base URL | 维度 |
|--------|------|----------|------|
| **Jina**（推荐） | `jina-embeddings-v5-text-small` | `https://api.jina.ai/v1` | 1024 |
| **OpenAI** | `text-embedding-3-small` | `https://api.openai.com/v1` | 1536 |
| **Google Gemini** | `gemini-embedding-001` | `https://generativelanguage.googleapis.com/v1beta/openai/` | 3072 |
| **Ollama**（本地） | `nomic-embed-text` | `http://localhost:11434/v1` | 768 |

---

## CLI 命令

```bash
# 列出记忆
openclaw memory list [--scope global] [--category fact] [--limit 20] [--json]

# 搜索记忆
openclaw memory search "query" [--scope global] [--limit 10] [--json]

# 查看统计
openclaw memory stats [--scope global] [--json]

# 按 ID 删除记忆（支持 8+ 字符前缀）
openclaw memory delete <id>

# 批量删除（--scope 必填，至少指定一个 scope）
openclaw memory delete-bulk --scope global [--before 2025-01-01] [--dry-run]

# 导出 / 导入
openclaw memory export [--scope global] [--output memories.json]
openclaw memory import memories.json [--scope global] [--dry-run]

# 使用新模型重新生成 Embedding
openclaw memory reembed --source-db /path/to/old-db [--batch-size 32] [--skip-existing]

# 从内置 memory-lancedb 迁移
openclaw memory migrate check [--source /path]
openclaw memory migrate run [--source /path] [--default-scope global] [--dry-run] [--skip-existing]
openclaw memory migrate verify [--source /path]
```

---

## 数据库 Schema

LanceDB 表 `memories`：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string (UUID) | 主键 |
| `text` | string | 记忆文本（FTS 索引） |
| `vector` | float[] | Embedding 向量 |
| `category` | string | `preference` / `fact` / `decision` / `entity` / `other` |
| `scope` | string | Scope 标识（如 `global`、`agent:main`） |
| `importance` | float | 重要性分数 0-1 |
| `timestamp` | int64 | 创建时间戳 (ms) |
| `metadata` | string (JSON) | 扩展元数据 |

---

## 依赖

| 包 | 用途 |
|----|------|
| `@lancedb/lancedb` ≥0.26.2 | 向量数据库（ANN + FTS） |
| `openai` ≥6.21.0 | OpenAI 兼容 Embedding API 客户端 |
| `@sinclair/typebox` 0.34.48 | JSON Schema 类型定义（工具参数） |

---

## License

MIT

---