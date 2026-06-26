---
name: super-memory
description: Use when the user asks to remember something, save experience, create notes, record decisions, track bugs, set preferences, summarize work, start projects, mentions 记忆/模板/学习/经验/笔记, encounters errors, or at the start of any new conversation. Also trigger when user message contains known entity names from memory-index tags.
---

> **Super Memory v3.0** — 会话检查点 · 语义标签搜索 · 渐进式召回 · 知识图谱 · 情节/语义分离 · 自动巩固 · 记忆钩子 · 冲突检测

---

## ⚡ 快速参考

| 做什么 | 怎么做 |
|--------|--------|
| 搜记忆 | `memory(operation="search", query="<关键词>")` |
| 读记忆 | `memory(operation="read", name="<name>")` |
| 存记忆 | `remember(name="...", type="...", description="...", body="...")` |
| 删记忆 | `forget(name="...")` |
| 列出全部 | `memory(operation="list")` |
| 更新索引 | `bash` 写 `memory/memory-index.json` |

---

## 零、对话启动：会话检查点 + 主动回忆

**每个新对话的第一件事：**

```
memory(operation="list")         ← 列出所有记忆
memory(operation="search", query="session-checkpoint")  ← 找会话检查点
```

然后向用户报告：

> 🧠 上次对话摘要（<时间>）：<做了什么>、<关键决定>、<待办事项>
> 📎 本话题共 X 条记忆。最近活跃：<top3>。需要回顾吗？

---

## 一、记忆存储模型

### 1.1 记忆分类

| 类别 | 含义 | 例子 |
|------|------|------|
| `episodic` | 发生了什么（时间线） | "2024-03-15 调试登录超时" |
| `semantic` | 学到了什么（知识） | "JWT token 过期时间 30 分钟" |
| `procedural` | 怎么做（流程） | "部署前跑 test suite + 检查 .env" |
| `session_checkpoint` | 对话摘要（连续性） | "本次讨论了 X，决定 Y，待办 Z" |

### 1.2 索引结构 (memory-index.json)

```json
{
  "memories": {
    "<name>": {
      "category": "episodic|semantic|procedural|session_checkpoint",
      "priority": 3,
      "confidence": 3,
      "summary": "≤50字摘要",
      "key_points": ["要点1", "要点2"],
      "tags": ["概念标签1", "概念标签2"],
      "relations": [
        {"target": "<other-name>", "type": "derives_from|supersedes|contradicts|supports|depends_on|caused_by"}
      ],
      "access_count": 0,
      "last_access": "",
      "created": "",
      "consolidated": false
    }
  }
}
```

---

## 二、语义标签搜索（核心检索升级）

### 保存时自动生成标签

每次 `remember` 后，分析记忆内容，生成 **5-10 个概念标签**（不是关键词，是抽象概念）。写入 `memory-index.json` 的 `tags` 字段。

例：一条「Python asyncio 死锁」的记忆 →
`["concurrency", "event-loop", "deadlock", "async-await", "debugging", "python", "race-condition"]`

### 搜索时两级匹配

```
1. 关键词匹配标题/正文（原有逻辑）→ 命中 +10 分
2. 概念标签交集加权（新增）→ 每个匹配标签 +5 分
3. 按总分排序，同分按 priority 排序
```

搜索策略：
1. 先用当前任务关键词搜一次
2. 再把任务关键词扩展为概念标签后再搜一次
3. 综合两次结果，去重排序

---

## 三、渐进式召回（省 Token）

```
搜索结果
  │
  ├── Level 1（始终展示）
  │     记忆名称 · ⭐优先级 · 类别 · summary（一行）· 匹配标签
  │
  ├── Level 2（用户说「展开」或自动对 top3 展开）
  │     key_points（3-5条要点）
  │
  └── Level 3（用户说「详细」或需要直接引用时）
       完整正文
```

对话中引用记忆时，默认用 Level 1+2，只有需要原文细节时才读 Level 3。

---

## 四、知识图谱（关系升级）

### 关系类型

| 关系 | 含义 | 何时用 |
|------|------|--------|
| `derives_from` | B 是从 A 总结出来的 | 合并/巩固时 |
| `supersedes` | B 取代了 A | 旧方案被新方案替代 |
| `contradicts` | A 和 B 矛盾 | 冲突检测时 |
| `supports` | A 支撑 B 的结论 | 证据链 |
| `depends_on` | B 依赖 A | 前置条件 |
| `caused_by` | B 的根因是 A | Bug → 根因追溯 |

### 保存时自动建关系

1. 搜索相关记忆
2. 对每条相关记忆，LLM 判断关系类型（可多个）
3. 写入双方的 `relations` 字段

### 查询时追溯

用户问「这个为什么失败」→ 沿 `caused_by` 链追溯根因
用户问「后来怎么改的」→ 沿 `supersedes` 链找最新方案

---

## 五、读取模式

展示记忆时支持两种模式：

### Markdown 模式（默认）
原始格式，保留 `**加粗**`、列表、代码块等。

### 纯文本模式
用户说「纯文本」「可读模式」「去掉格式」时：

1. 读取记忆后，剥离所有 Markdown 标记（`**`、`- `、`# `、``` 等）
2. 用等宽空白替代表格，保留内容结构
3. 输出前标注「📄 纯文本模式」

**纯文本格式示例：**
```
记忆: howto-autonomous-learning
类型: reference | 优先级: ⭐4
时间: 2026-06-26

问题: 用户想让 Reasonix 跨对话自主学习
方案: 创建了 autonomous-learning 技能
核心: Memory Compiler 学策略不学方案
```

### 记忆翻译
用户要求翻译记忆时：先读取 → 翻译为目标语言 → 以纯文本输出（翻译后 Markdown 标记无意义）

---

## 六、记忆钩子（实体自动触发）

维护实体清单（从所有记忆的 `tags` + 正文提取的项目名/人名/技术名）。

每次收到用户消息时：
1. 扫描消息中是否命中已知实体
2. 命中 → 自动 `memory.search` 该实体
3. 回复时附上：「📎 相关：<记忆名> — <summary>」

不等待用户主动说「回顾」，而是自动触发。但限制每次最多展示 3 条，避免噪音。

---

## 七、自动巩固（≥5 条同主题自动合并）

对话结束或手动触发「巩固记忆」时：
1. 扫描所有记忆的 `tags`，找出标签重叠 ≥3 个的记忆群
2. 若群内 ≥5 条 → 触发巩固
3. LLM 合并为一条综合记忆（`derives_from` 关联原来的每条）
4. 综合记忆 `priority` = max(原子记忆) + 1
5. 原子记忆标记 `consolidated: true`，priority 降为 1

---

## 八、冲突检测

保存新记忆时：
1. 搜索 `tags` 重叠 ≥2 的记忆
2. LLM 判断是否矛盾
3. 矛盾 → 对比展示 + 标注置信度 + 询问用户
4. 用户确认后 → 旧记忆标记 `supersedes` 关系，新记忆 priority 更高

---

## 九、会话检查点

**每次对话结束前**（用户说再见 / 超时 / 新话题开始时）：

```
remember(
  name="checkpoint-<date>",
  type="project",
  category="session_checkpoint",
  description="会话检查点 <日期>",
  body="**本次讨论了:** <主题>
**关键决定:** <决定>
**待办事项:** <待办>
**新学到的:** <知识点>")
```

下次对话启动时自动读取最近 1-2 个检查点。

---

## 十、初始化

首次运行（`memory-index.json` 不存在时），用 `bash` 创建：

```json
{"version":"3.0","entity_names":[],"memories":{},"stats":{"total":0}}
```

文件路径：`<项目工作区>/memory/memory-index.json`

---

## 十一、更新已有记忆

用户说「XX 不是 30 分钟，是 45 分钟」等修正时：

1. 搜索找到原记忆 → 读取
2. 如果改动小（单个数值/事实修正）：
   - `forget` 旧记忆 → `remember` 新版本（同名）
   - 在 `memory-index.json` 中保留原 `access_count` 和 `relations`
3. 如果改动大（新增重要发现）：
   - 保留原记忆 → 新建记忆 → 添加 `supersedes` 关系
   - 旧记忆 priority 降 1

原则：小改动直接替换，大改动保留历史。

---

## 十二、记忆健康检查

用户说「记忆健康」或每 10 次记忆操作后，自动扫描：

```
🏥 记忆健康报告

📊 总计: X 条
⭐ 平均优先级: X.X
📅 最旧记忆: <时间>
🕐 最近活跃: <时间>

⚠️ 需要关注:
- 低优先级(≤2)未清理: N 条
- 沉睡(>90天未访问): N 条
- 可合并(标签重叠≥4): N 组
- 矛盾关系未解决: N 对

💡 建议: <清理/合并/回顾>
```

---

## 十三、六种模板 + 类别自动匹配

触发词 → 自动套用模板 + 自动分配合适的 `category`：

| 触发 | 模板 | category |
|------|------|----------|
| 新项目、"开始做XX" | 项目启动 | procedural |
| Bug、"又出问题了" | 踩坑记录 | episodic |
| "我喜欢/不喜欢" | 偏好记录 | semantic |
| "总结"、"本周" | 周期总结 | session_checkpoint |
| "这个写法"、"套路" | 通用模式 | procedural |
| "为什么选"、"决策" | 决策记录 | semantic |

---

## 十四、导出 / 导入 / 备份

- **导出**：收集所有 .md + memory-index.json → 一个 JSON 文件 → 保存到桌面
- **导入**：解析 JSON → 逐条 remember → 重建索引
- **备份**：「备份记忆」→ 复制整个 memory/ 目录到桌面

---

## 十五、防偷懒强化版

| 你在想 | 真相 |
|--------|------|
| 「标签太麻烦」 | 保存时让 LLM 自动生成，不需要你动手 |
| 「3 级加载有用吗」 | 100 条记忆时省 90% token，必须做 |
| 「检查点没必要」 | 隔三天回来，检查点是唯一让你接上上下文的东西 |
| 「钩子会吵」 | 上限 3 条 + 去重 + 10 轮冷却，不吵 |

---

## 十六、版本

v3.1 — 记忆健康检查 · UPDATE操作 · 会话检查点触发明确 · 搜索评分校准 · 钩子去重 · 初始化流程 · 快速参考卡

---

## 十七、什么时候不记

| 不记的内容 | 原因 |
|-----------|------|
| 闲聊寒暄 | "你好""今天天气不错" — 无长期价值 |
| 一次性任务细节 | "帮我打开XX文件" — 任务特定，不可复用 |
| 已在 AGENTS.md 中的规则 | 避免重复 |
| 用户明确说「不用记」 | 尊重用户选择 |
| 超过 500 字的原文引用 | 用摘要替代，节省空间 |
