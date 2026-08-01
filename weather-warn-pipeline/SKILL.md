---
name: weather-warn-pipeline
description: "气象预警审核主流程：派发 3 个独立子 agent（OCR → audit → template），主 agent 只做派发 / 收集 / 报告生成。"
metadata:
  {
    "openclaw":
      {
        "emoji": "⚡"
      }
  }
---

# 气象预警审核 Pipeline（主入口 · 2026-08-01 新增）

气象台预警信息录入「突发预警信息发布平台」前的最后一道审核。
通过线性派发 3 个独立子 agent（OCR → audit → template）完成审核，**主 agent 只做派发 / 收集 / 报告生成**。

## FACT LOCK（事实锁 · 跨子 skill 总铁律）

> 🚨 **本章节为 4 个 skill 最高优先级条款。所有子 skill 必须遵守。**

事实锁原则（7 条 · 不可覆盖）：

① **图片内容是唯一事实来源**。

② OCR 完成以后，图片读取结果立即成为「**冻结事实（Frozen Facts）**」。

③ 后续任何步骤**不得修改 OCR 结果**。

④ 模板库**只能参考**，**不能覆盖**图片内容。

⑤ 历史案例**只能参考**，**不能覆盖**图片内容。

⑥ 推理**不能覆盖**图片内容。

⑦ 如果模板与图片冲突：

→ **必须保留图片读取内容**。

→ 输出：「模板与图片不一致，请人工复核。」

→ 🚨 **禁止**：自动修正图片文字。

## Evidence Priority（证据优先级 · 跨子 skill 总铁律）

证据等级（从高到低）：

| 优先级 | 证据来源 | 说明 |
|--------|----------|------|
| **P0** | 🖼 图片 | 最高优先级 · 唯一事实来源 |
| P1 | 📄 OCR 结果 | 图片读取后冻结 |
| P2 | ⌨️ 用户输入文字 | 仅作参考 |
| P3 | 📚 模板库 | 用语参考 |
| P4 | 🗂 历史案例 | 用语参考 |
| P5 | 🧠 AI 推理 | 最低优先级 |

🚨 **铁律**：

> **低优先级证据禁止覆盖高优先级证据。**

## 审核流程（强制线性顺序）

🚨 **铁律**：流程必须按以下顺序执行，**不得颠倒**：

```
图片
↓
[子 agent #1] weather-warn-ocr  → Frozen Facts
↓
[主 agent] 收集 OCR 结果
↓
[子 agent #2] weather-warn-audit  → 审核问题
↓
[主 agent] 收集审核结果
↓
[子 agent #3] weather-warn-template  → 模板参考
↓
[主 agent] 汇总生成最终报告（7 段固定模板）
↓
老板
```

各阶段定位：

| 阶段 | 作用 | 谁能做 |
|------|------|--------|
| 图片 | 事实唯一来源 | — |
| OCR | 读图 + Frozen Facts | **子 agent #1**（weather-warn-ocr）|
| OCR 结果收集 | 接收 Frozen Facts | **主 agent** |
| 审核 | 6 条规则 + 一致性核对 | **子 agent #2**（weather-warn-audit）|
| 审核结果收集 | 接收审核问题 | **主 agent** |
| 模板对比 | 模板查询 + 推荐 | **子 agent #3**（weather-warn-template）|
| 最终报告生成 | 7 段完整版 | **主 agent** |

🚨 **铁律**：
- **模板参考必须放在审核之后**。**不得**放在 OCR 之前。
- **主 agent 不得读图 / 不得审核 / 不得查模板**——只能派发 / 收集 / 汇总。
- **3 个子 agent 必须串行**（前一个完成才能派下一个）。

## Frozen Facts 字段定义（主 skill 定义 · ocr 子 skill 生成 · 后续子 skill 读取）

| 字段 | 类型 | 说明 |
|------|------|------|
| `warn_id` | string | 预警 ID（YYYY-MM-DD-HHMM 格式）|
| `title` | string | 标题 |
| `disaster_type` | string | 灾种 |
| `category` | string | 类别（如「黄色预警[IV级/一般]」）|
| `publisher` | string | 发布单位 |
| `publish_time` | string | 发布时间（精确到秒 · ISO 8601 + 时区）|
| `validity_hours` | int | 信息时效（整数小时）|
| `affected_areas_display` | string | 影响地区栏（折叠显示，如「白碱滩区 +2」）|
| `content` | string | 预警内容（正文 · 原文摘抄 · **不准改字**）|
| `defense_guide` | string | 防御指南（平台预设不审）|
| `warning_standard` | string | 预警标准（平台预设不审）|
| `audience` | string | 受众群体（平台自动推送不审）|
| `signer` | string | 签发人 |
| `screenshot_source` | string | 截图源（本地路径 / QQ 下载时间戳）|

🚨 **铁律**：
- Frozen Facts **一旦生成不得修改**
- 后续 audit / template 子 skill **只能读取 Frozen Facts**——不得重新 OCR / 不得重新推理 / 不得重新生成

## 子 agent 派发协议

主 agent 通过 OpenClaw `sessions_spawn` 派发独立子 agent（**模型统一 `minimax/MiniMax-M3`**）：

### Step 1 · 派发 OCR 子 agent

```python
result_ocr = sessions_spawn(
  task=f'''
    按 ~/.openclaw/skills/weather-warn-audit/weather-warn-ocr/SKILL.md 处理预警截图：
    
    截图路径：{screenshot_path}
    
    🚨 必须遵守：
    - 主 skill FACT LOCK（事实锁）
    - 读图 Step0~Step4 强制顺序
    - OCR 输出约束（不修改 / 不补全 / 不猜测）
    - 输出严格按 Frozen Facts JSON schema（见主 skill 字段定义）
    
    不要做任何审核 / 模板查询。
  ''',
  taskName=f"ocr-{warn_id}",
  runtime="subagent",
  mode="run",
  agentId="main",
  model="minimax/MiniMax-M3"
)
```

### Step 2 · 派发 audit 子 agent

```python
result_audit = sessions_spawn(
  task=f'''
    按 ~/.openclaw/skills/weather-warn-audit/SKILL.md 审核：
    
    Frozen Facts：
    {json.dumps(ocr_result, ensure_ascii=False, indent=2)}
    
    🚨 必须遵守：
    - 主 skill FACT LOCK（事实锁）
    - 主 skill Evidence Priority（证据优先级）
    - Frozen Facts 不得修改（只读取）
    
    只输出审核问题清单（严重 / 重大 / 提示 + 已合规项）。
    不要做模板查询 / 不要输出最终报告格式。
  ''',
  taskName=f"audit-{warn_id}",
  runtime="subagent",
  mode="run",
  agentId="main",
  model="minimax/MiniMax-M3"
)
```

### Step 3 · 派发 template 子 agent

```python
result_template = sessions_spawn(
  task=f'''
    按 ~/.openclaw/skills/weather-warn-audit/weather-warn-template/SKILL.md 查询模板：
    
    Frozen Facts：
    {json.dumps(ocr_result, ensure_ascii=False, indent=2)}
    
    审核问题：
    {json.dumps(audit_result, ensure_ascii=False, indent=2)}
    
    🚨 必须遵守：
    - 主 skill FACT LOCK（事实锁）
    - 模板参与边界（模板不得参与事实生成）
    - 模板不得修改 / 覆盖 Frozen Facts
    
    只输出模板参考（同类案例 + 句式对比 + 推荐）。
    不要做审核 / 不要做 OCR。
  ''',
  taskName=f"template-{warn_id}",
  runtime="subagent",
  mode="run",
  agentId="main",
  model="minimax/MiniMax-M3"
)
```

### 派发规则

🚨 **铁律**：
- **3 个子 agent 必须串行**（前一个完成才能派下一个）
- **每个子 agent 必须独立 session**（`runtime="subagent"`）
- **主 agent 不直接调用 OCR / 审核 / 模板工具**——只能派发子 agent
- **模型统一使用 `minimax/MiniMax-M3`**（避免多模型混淆）

### 失败处理

| 失败情形 | 处理 |
|----------|------|
| OCR 子 agent 失败 | 立即报错，**不**进入下一步 · 提示老板重试 |
| audit 子 agent 失败 | 使用 OCR 结果 + 标注「审核失败，请人工复核」 |
| template 子 agent 失败 | 使用 OCR + audit 结果 + 标注「模板参考不可用」 |

## 输出格式（最终报告 · 固定完整版 7 段 · 一字不改）

🚨 **铁律**：最终报告**严格按此模板输出 7 个 section**，**不允许省略任何 section**。

```
📸 读图结果（基于 [文件名]）
- 标题：[Frozen Facts.title]
- 灾种：[Frozen Facts.disaster_type]
- 类别：[Frozen Facts.category]
- 发布单位：[Frozen Facts.publisher]
- 发布时间：[精确到秒]
- 信息时效：[小时数]
- 影响地区栏：[Frozen Facts.affected_areas_display]
- 预警内容：[Frozen Facts.content · 原文摘抄]
- 防御指南：[Frozen Facts.defense_guide · 平台预设不审]
- 预警标准：[Frozen Facts.warning_standard · 平台预设不审]
- 受众群体：[Frozen Facts.audience · 平台自动推送不审]
- 签发人：[Frozen Facts.signer]
- 截图源：Frozen Facts.screenshot_source

📚 模板库参考
- [template 子 agent 返回的同类案例 + 句式对比 + 推荐]

📋 [预警类型] 审核报告

🚨 严重问题（X 处）
1. [位置] 问题描述 → 修改建议
2. ...

⚠️ 重大问题（X 处）
1. [位置] 问题描述 → 修改建议

💡 提示（X 条）
- [位置] 问题描述 → 优化建议

✅ 已合规项
- 标题 / 时间 / 灾种 / 地区 / 受众 / 时效

🎯 结论：可发布 / 需修改后发布
```

🚨 **铁律**：
- 严重问题 0 处 → 显示「（0 处）无」不省略
- 重大问题 0 处 → 显示「（0 处）无」不省略
- 提示 0 条 → 显示「（0 条）无」不省略
- 已合规项必须逐项列出（6 项）
- 结论必须填「可发布」或「需修改后发布」（二选一）

### 配套铁律（同时生效）

1. **必须返回读图结果**（7/26 20:03 老板明示）—— 不可跳到结论
2. **必须按完整模板输出**（7/28 19:43 老板明示）—— 不允许省略任何 section
3. **必须等人工复核结果才入模板库**（7/28 19:38 老板明示）—— AI 结论不直接入库

## 触发场景

- 老板说「审核一下」「录完前审查」「看一下」「重新审核」
- 老板发送预警信息截图或草稿
- 任何气象台预警录入平台前
- **老板说「通过」「不通过」「入库」「确认通过」** → 触发人工复核后入库流程

## 子 Skill 索引

| 子 Skill | 路径 | 职责 |
|----------|------|------|
| `weather-warn-ocr` | `weather-warn-ocr/SKILL.md` | 只读图 + OCR + Frozen Facts |
| `weather-warn-audit` | `SKILL.md`（根目录）| 只审核（6 条规则 + 一致性核对）|
| `weather-warn-template` | `weather-warn-template/SKILL.md` | 模板查询 + 推荐 + 入库 |

## 修订记录

| 日期 | 修订 | 触发 |
|------|------|------|
| 2026-08-01 | 初版：拆分原 weather-warn-audit 为 pipeline + ocr + audit + template 4 个 skill；FACT LOCK / Evidence Priority / 审核流程 / Frozen Facts 字段定义 / 7 段输出格式 全部归主 skill；主 agent 派发 3 个独立子 agent 串行执行；模型统一 `minimax/MiniMax-M3` | 老板指令：拆解 skill + 主 agent 派发独立子 agent + 4 个 skill 都进 alarm repo + 用主模型 |