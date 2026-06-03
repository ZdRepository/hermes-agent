# Hermes Agent 自学习功能分析

## 一、学习机制全景

### 1. 全息记忆（核心学习模块）

`plugins/memory/holographic/`

这是最核心的自学习模块，基于 SQLite 存储结构化"事实"，并通过**信任评分（Trust Score）**实现反馈学习：

```python
# store.py
_HELPFUL_DELTA   = +0.05   # 标记为有帮助 → 信任分上升
_UNHELPFUL_DELTA = -0.10   # 标记为无用   → 信任分下降（惩罚更大）
```

检索时加权：`final_score = relevance × trust_score`，高信任事实优先排名，低信任事实逐渐淘汰。

**反馈入口**是一个 Agent 可主动调用的工具：

```python
# __init__.py
FACT_FEEDBACK_SCHEMA = {
    "name": "fact_feedback",
    "description": "Rate a fact after using it. Mark 'helpful' if accurate, "
                   "'unhelpful' if outdated. This trains the memory — "
                   "good facts rise, bad facts sink.",
    "parameters": { "action": {"enum": ["helpful", "unhelpful"]}, "fact_id": int }
}
```

### 2. 自动事实提取

会话结束时，`on_session_end()` 钩子自动扫描对话内容，通过正则模式提取偏好和决策：

```python
# __init__.py
_PREF_PATTERNS = [
    r'\bI\s+(?:prefer|like|love|use|want|need)\s+(.+)',
    r'\bmy\s+(?:favorite|preferred|default)\s+\w+\s+is\s+(.+)',
]
_DECISION_PATTERNS = [
    r'\bwe\s+(?:decided|agreed|chose)\s+(?:to\s+)?(.+)',
    r'\bthe\s+project\s+(?:uses|needs|requires)\s+(.+)',
]
```

### 3. 技能系统（程序化记忆）

`tools/skill_manager_tool.py` + `agent/curator.py`

Agent 可自主创建、编辑技能（SKILL.md 文件），存储在 `~/.hermes/skills/`。策展器（Curator）在后台负责技能的生命周期管理：

| 状态 | 条件 |
|------|------|
| `fresh` → `active` | 被使用后 |
| `active` → `stale` | 30 天未使用 |
| `stale` → `archived` | 90 天未使用 |

### 4. 记忆持久化层

`tools/memory_tool.py` + `agent/memory_manager.py`

两个持久化文件：`MEMORY.md`（Agent 观察）和 `USER.md`（用户偏好），注入系统提示，在会话间保持"冻结快照"以保持前缀缓存稳定。

### 5. 检索策略（混合搜索）

`plugins/memory/holographic/retrieval.py`

1. **FTS5 全文搜索** → 候选事实
2. **Jaccard 重排名** → 词符重叠相关性
3. **信任加权** → 乘以 trust_score
4. **时间衰减（可选）** → `decay = 0.5^(age_days / half_life)`

---

## 二、学习闭环数据流

```
用户交互 → Agent 使用事实/技能
           ↓
      fact_feedback("helpful"/"unhelpful")
           ↓
      trust_score ±0.05 / -0.10
           ↓
    下次检索：高信任事实排名靠前
           ↓
      系统自我收敛，持续改进
```

---

## 三、关键文件一览

| 文件 | 作用 |
|------|------|
| `plugins/memory/holographic/store.py` | SQLite 事实存储 + 信任评分 |
| `plugins/memory/holographic/retrieval.py` | 混合检索 + 信任加权 |
| `plugins/memory/holographic/__init__.py` | 全息记忆提供者 + `fact_feedback` 工具 |
| `plugins/memory/holographic/holographic.py` | HRR 向量操作（语义编码） |
| `tools/memory_tool.py` | MEMORY/USER.md 持久化 |
| `agent/memory_manager.py` | 多提供者协调 + 生命周期钩子 |
| `agent/curator.py` | 技能生命周期管理 |
| `tools/skill_manager_tool.py` | 技能 CRUD |
| `trajectory_compressor.py` | 轨迹压缩（用于训练数据生成） |

---

## 四、补充：完整架构细节

### 全息记忆数据库架构（SQLite）

```
- facts 表: fact_id, content, category, tags, trust_score,
            retrieval_count, helpful_count, created_at, updated_at, hrr_vector
- entities 表: 实体名称、类型、别名
- fact_entities 表: 事实与实体的关联
- memory_banks 表: HRR 向量库
- facts_fts 表: SQLite FTS5 全文索引
```

### HRR 向量操作（holographic.py）

| 函数 | 作用 |
|------|------|
| `encode_atom()` | 确定性相位向量（SHA-256 基础） |
| `bind()` | 循环卷积（相位加法） |
| `unbind()` | 循环相关（相位减法） |
| `bundle()` | 叠加（圆形均值） |
| `encode_text()` | 词袋编码 |
| `encode_fact()` | 结构化事实编码 |

### 技能目录结构

```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md           # 主要指令（必需）
│   ├── references/        # 支持文档
│   ├── templates/         # 输出模板
│   ├── scripts/           # 可执行脚本
│   └── assets/            # 补充文件
└── category-name/
    └── another-skill/
        └── SKILL.md
```

### 可选记忆插件

| 目录 | 功能 |
|------|------|
| `plugins/memory/honcho/` | Honcho 用户建模 |
| `plugins/memory/hindsight/` | Hindsight 记忆 |
| `plugins/memory/mem0/` | Mem0 集成 |
| `plugins/memory/holographic/` | 全息记忆（主要） |

---

**总结**：hermes 的自学习不是模型微调，而是一套**运行时反馈驱动的知识管理系统**——通过信任评分筛选事实质量、自动提取偏好、技能自主创建与策展，形成跨会话的持续学习闭环。
