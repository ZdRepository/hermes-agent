# `prompt_caching.py` Prompt 缓存策略分析

源文件：`agent/prompt_caching.py`

---

## 概述

单一策略 `system_and_3`：向 API 消息列表注入最多 **4 个** `cache_control` 断点，覆盖 system prompt + 最后 3 条非 system 消息。在同一 session 的多轮对话中可将输入 token 成本降低约 **75%**。

模块为纯函数，无类状态，无 AIAgent 依赖。

---

## 公共入口：`apply_anthropic_cache_control()`

```
apply_anthropic_cache_control(
    api_messages,
    cache_ttl="5m",        # "5m"（默认）或 "1h"
    native_anthropic=False  # True = 原生 Anthropic 格式；False = OpenAI 兼容格式
) -> List[Dict]
```

返回 `api_messages` 的深拷贝（不修改原列表），注入断点后交给上游发送。

**断点放置规则：**

| 优先级 | 目标消息 | 条件 |
|--------|---------|------|
| 1 | `messages[0]`（system prompt） | role == "system" |
| 2~4 | 最后 3 条非 system 消息 | 从末尾向前取 |

---

## 内部函数

### `_build_marker(ttl)`

构造 `cache_control` 字典：

| TTL | 生成值 |
|-----|--------|
| `"5m"`（默认） | `{"type": "ephemeral"}` |
| `"1h"` | `{"type": "ephemeral", "ttl": "1h"}` |

### `_apply_cache_marker(msg, cache_marker, native_anthropic)`

将 `cache_control` 注入单条消息，处理三种 content 格式：

| 情况 | 处理方式 |
|------|---------|
| `role == "tool"` 且 `native_anthropic=True` | 直接设 `msg["cache_control"]` |
| `role == "tool"` 且 `native_anthropic=False` | **跳过**（OpenAI 格式不支持 tool 消息缓存） |
| `content` 为字符串 | 转为 `[{"type": "text", "text": ..., "cache_control": ...}]` |
| `content` 为列表 | 在最后一个 block 上追加 `cache_control` |

---

## `native_anthropic` 标志的作用

`native_anthropic=True` 时走原生 Anthropic API 格式，`tool` 消息允许顶层 `cache_control`；`False` 时走 OpenAI 兼容格式（如 OpenRouter），tool 消息跳过，避免 provider 拒绝请求。

---

## 与 ContextCompressor 的关系

`apply_anthropic_cache_control()` 在消息发送前（压缩之后）调用，作用于最终发出的消息列表。两者职责分离：

- **ContextCompressor** — 控制消息数量，保证不超出 context window
- **prompt_caching** — 在剩余消息上标记断点，降低重复前缀的计费 token 数
