# 发给 OpenClaw 的快速安装指令

**用法：** 把下面整段复制，粘贴到 OpenClaw 对话里发送，让 OpenClaw 按对话执行快速安装。

---

快速安装（通过openclaw）

一、安装这个记忆插件 https://github.com/jidechao/openclaw-plugins-memory-lancedb

要求：

1、embedding模型配置
   - api_key: <你的 SiliconFlow API Key>
   - base_url: https://api.siliconflow.cn/v1
   - model: BAAI/bge-m3

2、reranker模型配置
   - api_key: <你的 SiliconFlow API Key>
   - base_url: https://api.siliconflow.cn/v1
   - model: BAAI/bge-reranker-v2-m3

3、禁用memory-lancedb，同一时间只能启用一个 memory 插件。

二、设置记忆管理铁律规则

将下面的铁律也存入memory.md：

## LanceDB Operational Iron Rules

> Extracted from MEMORY.md -rules directly related to LanceDB
> read/write operations.

## 1.  Dual-Layer Memory Storage (IRON LAW)

Every pitfall/lesson learned → IMMEDIATELY store **TWO** memories to LanceDB before moving on:

- **Technical layer**: `Pitfall: [symptom]. Cause: [root cause]. Fix: [solution]. Prevention: [how to avoid]` (category: fact, importance ≥ 0.8)
- **Principle layer**: `Decision principle ([tag]): [behavioral rule]. Trigger: [when it applies]. Action: [what to do]` (category: decision, importance ≥ 0.85)
- After each store, **immediately `memory_recall`** with anchor keywords to verify retrieval. If not found, rewrite and re-store.
- Missing either layer = incomplete. Do NOT proceed to next topic until both are stored and verified.
- Also update relevant SKILL.md files to prevent recurrence.

## 2. **LanceDB Hygiene**

Entries must be short and atomic (< 500 chars). Never store raw conversation summaries, large blobs, or duplicates. Prefer structured format with keywords for retrieval.

## 3. Recall before retry

On ANY tool failure, repeated error, or unexpected behavior, ALWAYS `memory_recall` with relevant keywords (error message, tool name, symptom) BEFORE retrying. LanceDB likely already has the fix. Blind retries waste time and repeat known mistakes.

## 4. Confirm the target codebase before editing.

When working on memory plugins, confirm you are editing the intended package (e.g., `memory-lancedb-pro` vs built-in `memory-lancedb`) before making changes; use `memory_recall` + filesystem search to avoid patching the wrong repo.

## 5. Plugin code changes **must** clear the Jiti cache (**MANDATORY**).

After modifying ANY `.ts` file under `plugins/`, MUST run `rm -rf /tmp/jiti/` BEFORE `openclaw gateway restart`. jiti caches compiled TS; restart alone loads STALE code. This has caused silent bugs multiple times. Config-only changes do NOT need cache clearing.
