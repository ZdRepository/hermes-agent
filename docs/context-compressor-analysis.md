# `ContextCompressor.compress()` 四阶段压缩算法分析

源文件：`agent/context_compressor.py:1827`

---

## Phase 1 — 工具结果修剪（无 LLM 调用）`L1877`

`_prune_old_tool_results()` 对尾部保护窗口之外的消息做三轮处理：

1. **重复去重** — 相同内容的工具结果（MD5 hash）只保留最新一份，旧的替换为 `[Duplicate tool output...]`
2. **摘要替换** — 超过 200 字符的 tool result 替换为 1 行描述，如 `[terminal] ran 'npm test' -> exit 0, 47 lines`
3. **截断 tool_call 参数** — 超 500 字符的 arguments JSON 解析后对字符串字段截断，保持 JSON 合法（避免下游 provider 400）

截图（`image_url/input_image/image`）被替换为占位文本。

---

## Phase 2 — 确定压缩边界 `L1884`

两端保护：

| 端 | 方法 | 逻辑 |
|----|------|-------|
| **Head** | `_protect_head_size()` | system prompt + `protect_first_n`（默认 3）条消息 |
| **Tail** | `_find_tail_cut_by_tokens()` | 从末尾向前累积 token，直到超出 `tail_token_budget`（≈ `0.20 × threshold`） |

边界对齐规则：
- 向前推过孤立 tool result（`_align_boundary_forward`）
- 向后推避免拆断 tool_call/result 组（`_align_boundary_backward`）
- **强制保证最新 user message 在 tail 中**（`_ensure_last_user_message_in_tail`，修 #10896）

中间窗口 = `messages[compress_start:compress_end]`，同时搜索窗口内已有的 summary 用于迭代更新。

---

## Phase 3 — LLM 结构化摘要 `L1936`

`_generate_summary()` 调用辅助模型（或主模型）生成 markdown checkpoint：

- **首次压缩** → 全新 summary，模板含 13 个 section（见下表）
- **再次压缩** → 迭代更新：将 `_previous_summary` + 新 turns 一并发给 LLM，合并进度

### 模板 Section 说明（`_template_sections` L1268）

| # | Section | 含义 |
|---|---------|------|
| 1 | **Active Task** | **最重要字段**。用户最近一个未完成输入，逐字引用。是"用户还在等什么答复"，不是"当前在做什么" |
| 2 | **Goal** | 用户整体想达成的宏观目标，不随每步操作变化 |
| 3 | **Constraints & Preferences** | 用户编码风格偏好、约束条件、重要决策 |
| 4 | **Completed Actions** | 已执行的具体操作，编号列表，格式：`N. ACTION target — outcome [tool: name]` |
| 5 | **Active State** | 当前工作状态快照：分支、已改动文件、测试通过率、运行中的进程等 |
| 6 | **In Progress** | 压缩触发时正在进行中、尚未完成的工作 |
| 7 | **Blocked** | 尚未解决的阻塞项、错误、问题，含完整 error message |
| 8 | **Key Decisions** | 重要技术决策及其**原因**（Why，不只是 What） |
| 9 | **Resolved Questions** | 用户**已得到答复**的问题 + 答案，避免重复回答 |
| 10 | **Pending User Asks** | 用户提出但**尚未履行**的请求或问题 |
| 11 | **Relevant Files** | 被读取/修改/创建的文件，每条附简短说明 |
| 12 | **Remaining Work** | 还需要做什么（作为背景上下文，不是指令） |
| 13 | **Critical Context** | 若不显式保留则会丢失的关键值：配置细节、具体数值、error message。API key/密码写 `[REDACTED]` |

> **静态兜底模板**（`_build_static_fallback_summary` L1142）在第 12、13 之间额外插入 **Last Dropped Turns**（被压缩掉的最后几条对话摘录），共 14 个 section。

**迭代更新规则**（再次压缩时）：
- `In Progress` 已完成项 → 移入 `Completed Actions`
- `Pending User Asks` 已答项 → 移入 `Resolved Questions`
- `Active Task` 随最新用户输入更新，旧任务若已取消则丢弃

失败处理层级：
1. summary_model 失败 → 自动回退到主模型重试一次
2. 主模型也失败且 `abort_on_summary_failure=True` → 返回原始 messages 不压缩
3. 默认 → 插入 `_build_static_fallback_summary()`（本地确定性生成，无 LLM）

---

## Phase 4 — 组装压缩结果 `L1964`

1. 复制 head messages，向 system prompt 注入压缩说明注记
2. 插入 summary message（role 选 `user`/`assistant`，避免与 head/tail 相邻重复；或合并进第一条 tail 消息）
3. 追加 tail messages
4. `_sanitize_tool_pairs()` — 修复孤立 tool_call/result（移除无对应 call 的 result；为无 result 的 call 插入 stub）
5. `_strip_historical_media()` — 清除最新图片消息之前所有消息的 image 内容（防止历史图片重复发送）
6. 计算压缩率；连续两次 `<10%` 触发反抖动保护（后续跳过压缩）

---

## 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `threshold_percent` | `0.50` | 触发压缩时机（context 用量占比） |
| `summary_target_ratio` | `0.20` | tail/summary token 预算比例 |
| `protect_first_n` | `3` | head 保护条数（不含 system prompt） |
| `protect_last_n` | `20` | tool result 修剪时尾部保护条数 |
| `abort_on_summary_failure` | `False` | LLM 摘要失败时是否中止压缩 |
