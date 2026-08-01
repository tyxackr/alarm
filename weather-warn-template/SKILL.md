---
name: weather-warn-template
description: "气象预警模板子 skill：模板查询 + 推荐 + 入库。不得参与事实生成，不得覆盖 Frozen Facts。"
metadata:
  {
    "openclaw":
      {
        "emoji": "📚"
      }
  }
---

# 气象预警模板 子 Skill（2026-08-01 新增）

由主 skill `weather-warn-pipeline` 派发。
**只**负责模板查询 + 推荐 + 入库。
**不做** OCR / **不做**审核 / **不做**最终报告生成。

## 边界

- **输入**：OCR 子 skill 的 Frozen Facts + audit 子 skill 的审核问题清单
- **输出**：模板参考（同类案例 + 句式对比 + 推荐）
- **不**读图 / **不**做审核 / **不**改 Frozen Facts

## 受 FACT LOCK / Evidence Priority 约束（引用主 skill）

🚨 **铁律**：遵守主 skill 的 FACT LOCK（事实锁）+ Evidence Priority（证据优先级）。

特别要点：
- 模板**永远不得参与事实生成**
- 模板与 Frozen Facts 冲突时，**必须保留 Frozen Facts**

## 🚨 模板参与边界（最强约束 · 2026-08-01 新增）

**核心原则**：

> 📚 **模板永远不得参与事实生成。**

模板**只能**参与：

- ✓ 审核（判断句式是否符合惯例、常用表达）
- ✓ 提示（推荐历史案例）
- ✓ 案例推荐

模板**不得**参与：

- ✗ 修正 OCR
- ✗ 修改图片文字
- ✗ 自动补全文字
- ✗ 多数决覆盖图片
- ✗ OCR
- ✗ 字段提取
- ✗ 正文摘抄

🚨 **铁律**：

> **模板仅提供参考。不得参与 OCR 结果生成。不得覆盖 Frozen Facts。**

## 互动 1：审核前查同类参考

收到 Frozen Facts + 审核问题 → 自动 grep 模板库（同灾种/类别/时效）→ 列出同类已通过案例作为参考：

```
📚 模板库参考（同 冰雹·橙色 IV 级·3h）
- [2026-07-28-0415] ✅ 预计未来3小时，克拉玛依区、白碱滩区的部分区域...
- [2026-07-28-0809] ✅ 预计未来3小时，克拉玛依区、白碱滩区的部分区域...
- (反例) [2026-07-22-0422] ❌ 大风·黄色 III 级·7h · 时间不一致 + 语病
```

模板库路径：`~/.openclaw/workspace/data/warn-templates/`

## 互动 2：审核后入库（🚨 必须人工复核）

🚨 **铁律**：**AI 审核完不直接入库**，必须等老板复核结果才入。

### 流程

1. AI 完成审核报告（主 agent 汇总）
2. **AI 主动问老板**：人工复核结果 = 通过 / 不通过 / 需修改？
3. 老板回复「**通过**」→ 调 `add-warn.py --review-result passed` 入库
4. 老板回复「**不通过**」→ 调 `add-warn.py --review-result failed` 入库（含审出问题）
5. 老板回复「**需修改**」→ **不入库**，等老板修改后重新审核

### 触发词

| 老板说 | 触发 |
|--------|------|
| 「通过」「确认通过」「入库」「入模板」 | → 入 passed.jsonl |
| 「不通过」「驳回」「打回」「不录入」 | → 入 failed.jsonl |
| 「需修改」「重审」「这条不要」 | → 不入库，等下次 |
| 「通过但不入库」「审核通过但不归档」 | → **审核通过，但不调 `add-warn.py`，不写 `passed.jsonl`**（2026-07-30 14:30 新增）|

## 互动 3：模板推荐（≥3 条同类时触发）

同一灾种+类别通过案例 ≥ 3 条时，自动列出：

```
💡 模板推荐
本次审核的是 [冰雹·橙色 IV 级·3h]，模板库已有 N 条同类参考：
- 2026-07-28-0415 ✅ 正文 48 字
- 2026-07-28-0809 ✅ 正文 48 字
- ...
参考正文句式：「预计未来3小时，[地区]的部分区域可能出现冰雹伴随雷电天气...」
```

## 入库脚本

```bash
# 通过案例
python3 ~/.openclaw/workspace/data/warn-templates/scripts/add-warn.py \
  --id 2026-07-29-XXXX \
  --灾种 雷电 \
  --类别 黄色预警[IV级/一般] \
  --时效小时 6 \
  --标题 "克拉玛依市气象台发布雷电黄色预警[IV级/一般]" \
  --正文 "预计未来6小时..." \
  --影响地区栏 "白碱滩区 +2（共 3 区）" \
  --影响地区正文 "克拉玛依区,白碱滩区,乌尔禾区" \
  --发布时间 "2026-07-29 XX:XX:XX" \
  --截图 "qqbot/downloads/..." \
  --12379短信勾选 true \
  --review-result passed

# 不通过案例（必带 --审出问题）
python3 ~/.openclaw/workspace/data/warn-templates/scripts/add-warn.py \
  ... \
  --review-result failed \
  --审出问题 "[严重] 时间不一致,[严重] 语病"
```

## 查询脚本

```bash
python3 ~/.openclaw/workspace/data/warn-templates/scripts/query-warn.py --all
python3 ~/.openclaw/workspace/data/warn-templates/scripts/query-warn.py "冰雹"
python3 ~/.openclaw/workspace/data/warn-templates/scripts/query-warn.py --type 雷电 --passed
python3 ~/.openclaw/workspace/data/warn-templates/scripts/query-warn.py --hours 3
```

## 输出格式（模板参考 JSON）

```json
{
  "warn_id": "2026-08-01-XXXX",
  "similar_cases": [
    {
      "case_id": "2026-07-28-0415",
      "disaster_type": "冰雹",
      "category": "橙色预警[IV级/一般]",
      "validity_hours": 3,
      "content_excerpt": "...",
      "passed": true
    }
  ],
  "expression_pattern": "预计未来3小时，[地区]的部分区域可能出现冰雹伴随雷电天气...",
  "recommendation": "...",
  "conflicts_with_frozen_facts": []
}
```

🚨 **铁律**：
- `conflicts_with_frozen_facts` **必填**——如果模板与 Frozen Facts 字段冲突，列出冲突点 + 标注「保留 Frozen Facts · 请人工复核」
- `similar_cases` 按相关性排序，最多列 5 条
- 模板库查询结果为 0 条时，**不**强行推荐——`recommendation` 留空或标「无同类参考」

## 触发场景

- **由主 skill `weather-warn-pipeline` 派发**（`taskName="template-{warn_id}"`）
- ❌ **不**直接被老板调用
- **人工复核入库**流程：由主 agent 收到老板触发词后调 `add-warn.py`（template 子 skill 提供脚本定义）

## 修订记录

| 日期 | 修订 | 触发 |
|------|------|------|
| 2026-08-01 | 初版：从原 weather-warn-audit 剥离「与模板库互动」章节 + 互动 1/2/3 + 入库脚本 + 查询脚本；新增「模板参与边界」最强约束；输出模板参考 JSON 含 `conflicts_with_frozen_facts` 字段 | 老板指令：拆分 audit skill，template 独立 |