---
name: lesson
description: Store a lesson learned from the current conversation. Triggered by /lesson command. Use when Master signals that the recent conversation contains a pitfall, fix, or key insight that should be persisted to long-term memory.
---

# Lesson Extraction & Storage

When triggered by `/lesson` command, extract and store lessons from the **recent conversation context** following the Dual-Layer Memory Storage rule.

## When to Use

- Encountered and fixed a bug/pitfall
- Learned something important about how a system works
- Made a design decision that should be remembered
- User explicitly says "记住这个教训" or similar

## Steps

### 1. Scan Context

Identify from recent messages:
- **Symptom**: What went wrong / what was observed
- **Root Cause**: Why it happened
- **Solution**: How it was fixed
- **Prevention**: How to avoid in the future

### 2. Store Technical Layer (fact, importance ≥ 0.8)

```
Pitfall: [symptom]. Cause: [root cause]. Fix: [solution]. Prevention: [how to avoid].
```

Example:
```
Pitfall: AI Daily Digest 脚本调用 DeepSeek API 时无限卡住。Cause: fetch 请求没有设置超时。Fix: 添加 AbortController + setTimeout(60s)。Prevention: 所有外部 API 调用必须设置超时。
```

### 3. Store Principle Layer (decision, importance ≥ 0.85)

```
Decision principle ([tag]): [behavioral rule]. Trigger: [when]. Action: [what to do].
```

Example:
```
Decision principle (api-timeout): 所有外部 API 调用必须设置超时。Trigger: 编写任何 fetch 调用外部服务的代码时。Action: 使用 AbortController + setTimeout(30-60s)，并在 catch 中处理 AbortError。
```

### 4. Verify Storage

Use `memory_recall` with anchor keywords to confirm both entries are retrievable:
- Call with keywords from the pitfall (e.g., "timeout fetch API")
- Verify both entries appear in results

### 5. Report to Master

Brief summary of what was stored:
```
✅ 已存储今天的教训

📌 **技术层** (fact): [one-line summary]
📌 **原则层** (decision): [one-line summary]

✅ 验证通过，两条记忆均可召回。
```

## Rules

- Keep entries **short and atomic** (< 500 chars each)
- Use **English tags** for decision principles (e.g., `api-timeout`, `lancedb-dual-store`)
- Include **keywords** for retrieval (tool names, error messages, concepts)
- If lesson affects a SKILL.md or checklist, **update those files too**
- If no clear lesson found, **ask Master** what to store

## Example Session

```
User: /lesson

Agent: 扫描最近的对话，发现以下教训：

**问题**: AI Daily Digest 脚本调用 DeepSeek API 时无限超时
**原因**: callOpenAICompatible 函数没有设置 fetch 超时
**修复**: 添加 AbortController + 60秒超时

正在存储双层记忆...

[stores fact layer]
[stores decision layer]
[verifies with memory_recall]

✅ 已存储今天的教训

📌 **技术层**: API 调用无限超时 → 需设置 AbortController
📌 **原则层**: api-timeout 原则 → 所有 fetch 必须有超时

✅ 验证通过。
```

## Related

- See `MEMORY.md` section "Dual-Layer Memory Storage" for the iron law
- See `MEMORY.md` section "Auto-Capture vs Manual Storage" for when to use `/lesson` vs auto-capture
