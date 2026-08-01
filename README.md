# ⚠️ weather-warn

> **气象预警审核 4 Skill 套件** —— 气象台预警信息录入「突发预警信息发布平台」前的最后一道审核。

适用于 OpenClaw / Hermes 等 agent 框架。

## 📦 套件组成（4 个 Skill）

| Skill | 类型 | 职责 | 路径 |
|-------|------|------|------|
| **`weather-warn-pipeline`** | 🎯 主入口 | 派发 3 个子 agent + 收集结果 + 生成最终报告 | [`weather-warn-pipeline/SKILL.md`](weather-warn-pipeline/SKILL.md) |
| **`weather-warn-ocr`** | 📸 子 skill | 只读图 + OCR + Frozen Facts（≤200 行） | [`weather-warn-ocr/SKILL.md`](weather-warn-ocr/SKILL.md) |
| **`weather-warn-audit`** | ⚠️ 子 skill | 只审核（6 条规则 + 一致性核对） | [`SKILL.md`](SKILL.md) |
| **`weather-warn-template`** | 📚 子 skill | 模板查询 + 推荐 + 入库 | [`weather-warn-template/SKILL.md`](weather-warn-template/SKILL.md) |

## 🔄 审核流程（线性派发）

```
老板触发
  ↓
[主 agent] pipeline
  ↓ sessions_spawn subagent #1
[子 agent] ocr  → Frozen Facts
  ↓
[主 agent] 收集 + 派发 subagent #2
[子 agent] audit  → 审核问题
  ↓
[主 agent] 收集 + 派发 subagent #3
[子 agent] template  → 模板参考
  ↓
[主 agent] 汇总 → 7 段完整报告 → 老板
```

🚨 **铁律**：
- 3 个子 agent 必须**串行**执行（前一个完成才能派下一个）
- 主 agent **只做**派发 / 收集 / 报告生成（**不读图 / 不审核 / 不查模板**）
- 所有子 agent 使用主模型 `minimax/MiniMax-M3`

## 🔒 FACT LOCK（事实锁 · 跨子 skill 总铁律）

- 🖼 图片 = 唯一事实来源
- OCR 完成 = 立即冻结（Frozen Facts）
- 后续步骤不得修改 OCR 结果
- 模板/案例/推理均不得覆盖图片
- 冲突时输出「模板与图片不一致，请人工复核」
- 🚫 **禁止**：自动修正图片文字

详见 [`weather-warn-pipeline/SKILL.md`](weather-warn-pipeline/SKILL.md) FACT LOCK 章节。

## 🚀 安装

### 方式 A · OpenClaw（推荐）

```bash
# 整个套件（4 个 skill）一起装
cp -r . ~/.openclaw/skills/weather-warn-audit/

# 或软链（推荐用于开发）
ln -s "$(pwd)" ~/.openclaw/skills/weather-warn-audit
```

**OpenClaw skill loader 行为**：
- 根目录 `SKILL.md` → 加载为 `weather-warn-audit` skill
- 子目录 `weather-warn-pipeline/SKILL.md` 等 → 加载为对应 skill

### 方式 B · Hermes / 其他框架

参考对应框架的 skill 加载路径，确保 4 个 SKILL.md 都在可加载目录里。

### 调用方式

```
老板说"审核一下" / "录完前审查" / 发预警截图
  ↓
[主 agent 加载 weather-warn-pipeline]
  ↓
派发 3 个子 agent
  ↓
[最终报告返回老板]
```

主 agent 触发后，**无需**手动调用任何子 skill——pipeline 主 skill 内部完成派发。

## 📋 各 Skill 详情

### 🎯 weather-warn-pipeline（主入口）

- FACT LOCK / Evidence Priority 跨子 skill 总铁律
- Frozen Facts 14 字段定义
- 3 个子 agent 派发协议（sessions_spawn）
- 最终报告 7 段固定模板（📸读图 → 📚模板 → 📋审核 → 🚨严重 → ⚠️重大 → 💡提示 → ✅合规 → 🎯结论）

### 📸 weather-warn-ocr（读图子 skill）

- 读图 Step0~Step4 强制顺序
- OCR 输出约束（不修改 / 不补全 / 不猜测）
- Frozen Facts JSON 输出（14 字段 + `platform_comments_excluded` + `ocr_confidence`）
- SOP-A 强制重新读图 / SOP-B 二次怀疑 / SOP-C 雪球熔断
- ≤200 行

### ⚠️ weather-warn-audit（审核子 skill · 根目录）

- 6 条审核规则（严重 1/2/4/5/6 + 重大 3 + 提示）
- 文字审核 SOP（针对 Frozen Facts.content）
- 一致性核对清单（发布单位 / 渠道 / 标题 vs 灾种 vs 正文 / 级别 / 时间 / 术语 / 地区 / 防御指南）
- 4 个关联案例（2026-07-26 大风 / 雷电 / 2026-07-28 冰雹 / 雷电入库）
- 输出审核问题清单 JSON

### 📚 weather-warn-template（模板子 skill）

- 互动 1：审核前查同类参考
- 互动 2：审核后入库（🚨 必须人工复核）
- 互动 3：模板推荐（≥3 条同类时触发）
- 入库脚本 + 查询脚本
- 模板参与边界最强约束
- 输出模板参考 JSON（含 `conflicts_with_frozen_facts`）

## 🛠 模板库（运行时）

模板库**不属于**本仓库（运行时数据，`.gitignore` 已排除）：

- 数据目录：`~/.openclaw/workspace/data/warn-templates/`
- 入库脚本：`scripts/add-warn.py`
- 查询脚本：`scripts/query-warn.py`

详见 [`weather-warn-template/SKILL.md`](weather-warn-template/SKILL.md)。

## 📜 修订记录

| 日期 | 修订 |
|------|------|
| 2026-08-01 | 🎯 **重大重构**：拆分原 weather-warn-audit 为 pipeline + ocr + audit + template 4 个 skill；FACT LOCK / Evidence Priority / 审核流程 / Frozen Facts 字段定义 / 7 段输出格式 全部归主 skill；主 agent 派发 3 个独立子 agent 串行执行；模型统一 `minimax/MiniMax-M3` |
| 2026-07-30 14:30 | 新增触发词「通过但不入库」（保留在 template 子 skill） |
| 2026-07-30 14:26 | 新增「级别一致性」永久规则（保留在 audit 子 skill） |
| 2026-07-29 16:28 | 「模板仅作用语参考，不得覆盖读图结果」（已加强为「模板参与边界」最强约束） |
| 2026-07-29 15:20 | 新增「读图 SOP（抗字形误读）」（已移至 ocr 子 skill） |
| 2026-07-28 23:20 | 新增「预警内容读取边界（防读多内容）」（已移至 ocr 子 skill） |
| 2026-07-28 23:12 | 新增「文字审核 SOP（针对预警正文）」（保留在 audit 子 skill） |
| 2026-07-28 19:44 | 新增第 6 条「12379 短信必须勾选」（保留在 audit 子 skill） |
| 2026-07-28 19:43 | 锁定：输出格式改为「固定完整版 · 一字不改」（已移至主 skill） |
| 2026-07-28 19:38 | 新增「与模板库互动」章节（已移至 template 子 skill） |
| 2026-07-28 04:24 | 明确「+N」含义（保留在 audit 子 skill） |
| 2026-07-26 20:03 | 输出格式新增「读图结果」section（已移至主 skill） |
| 2026-07-26 19:00 | 初版 |

## 📄 License

MIT · tyxackr