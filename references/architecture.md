# Super Memory v3.0 — 架构详解

## 数据模型

```
memory-index.json
  ├── entity_names[]          ← 实体清单（驱动记忆钩子）
  └── memories:
      ├── category            ← episodic | semantic | procedural | session_checkpoint
      ├── priority: 1-5       ← 自动升降（访问次数 + 遗忘曲线）
      ├── confidence: 1-5     ← 5=用户明确声明, 3=推断, 2=单次提及
      ├── summary             ← ≤50字摘要（Level 1 召回）
      ├── key_points[]        ← 3-5条要点（Level 2 召回）
      ├── tags[]              ← 5-10个概念标签（语义搜索核心）
      ├── relations[]         ← 三元组: {target, type}
      ├── access_count        ← 访问计数
      ├── last_access         ← 最后访问时间（遗忘曲线）
      └── consolidated        ← 已被合并标记
```

## 知识图谱

```
architecture-memory-system ──supersedes──→ howto-autonomous-learning
         │                                      │
         └──────derives_from────────────────────┘
```

## 搜索流程

```
用户消息
  │
  ├── 记忆钩子：消息中含已知实体？→ 自动 memory.search(实体名)
  │
  ├── 正常搜索：memory.search(任务关键词)
  │       │
  │       ├── 关键词匹配标题/正文 → +10分
  │       ├── 概念标签交集 → 每个匹配 +5分
  │       └── 按总分排序 → 同分按 priority
  │
  └── 展示：Level 1 (summary) → 用户展开 → Level 2 (key_points) → 需要时 Level 3 (全文)
```

## 记忆生命周期

```
创建 (priority=3, confidence=3)
  │
  ├── 被访问 ≥3 次 → priority +1（最高5）
  ├── 用户说「重要」→ priority = 5, confidence = 5
  │
  ├── 30天未访问 → priority -1
  ├── 90天未访问 → 建议清理
  │
  ├── 同主题 ≥5 条 → 自动巩固 → 综合记忆 priority+1
  └── 发现冲突 → 对比展示 → 用户裁决 → supersedes/contradicts
```

## 工具依赖

全部基于 Reasonix 内置工具，零外部依赖：
- `memory(operation="search|read|list")` — 检索
- `remember(name, type, description, body)` — 保存
- `forget(name)` — 删除
- `bash` — 读写 memory-index.json
