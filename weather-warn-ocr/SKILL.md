---
name: weather-warn-ocr
description: "气象预警截图 OCR 子 skill：只负责读图 + 生成 Frozen Facts，不做审核、不查模板。"
metadata:
  {
    "openclaw":
      {
        "emoji": "📸"
      }
  }
---

# 气象预警 OCR 子 Skill（2026-08-01 新增）

由主 skill `weather-warn-pipeline` 派发。
**只**负责读图 + 逐字段 OCR + 生成 Frozen Facts。
**不做**审核 / **不做**模板对比 / **不做**报告生成。

## 受 FACT LOCK / Evidence Priority 约束（引用主 skill）

🚨 **铁律**：遵守主 skill 的 FACT LOCK（事实锁）+ Evidence Priority（证据优先级）。

特别要点：
- 图片 = 唯一事实来源（违反 = 整个审核失真）
- OCR 完成 = 立即冻结（后续 audit / template 子 skill 不得修改）

## 读图步骤（强制顺序 · Step0~Step4）

🚨 **铁律**：必须按以下顺序执行，**不得跳步**：

| Step | 动作 | 说明 |
|------|------|------|
| **Step 0** | 重新读取图片 | 用 `read` 工具重新读图，**不依赖短时记忆** |
| **Step 1** | 逐字段 OCR | 14 个字段（含 `warn_id`）逐个 OCR |
| **Step 2** | 生成 Frozen Facts | 按主 skill 字段定义整理为 JSON |
| **Step 3** | 冻结 | OCR 结果**立即冻结**，**不得再修改** |
| **Step 4** | 输出 | 返回 Frozen Facts JSON 给主 agent |

## OCR 输出约束

🚨 **铁律**：OCR 阶段**禁止**以下行为：

- ❌ 修改文字
- ❌ 补全文字
- ❌ 猜测模糊文字
- ❌ 依据模板修改文字
- ❌ 多数决覆盖图片

处理原则：

| 情形 | 处理 |
|------|------|
| 能确认 | **原样摘抄** |
| 不能确认 | 输出 `null` 或 `[无法确认]` |
| 模糊 | **不得猜测** |

🚨 **铁律**：OCR 结果**只能**从图片读出，**不得**从模板、案例、推理补全。

## 🚨 预警内容读取边界（防读多内容 · 7/28 23:20 老板纠错）

**预警正文编辑框前几行常出现平台系统模板注释**（带 `*` 号），**不是**实际预警正文——必须**只摘抄**从「气象台发布…信号」开始的实际预警文字。

| 类别 | 特征 | 例子 |
|------|------|------|
| 平台系统注释 | 行首 `*` + 操作/界面提示 | `*请接收企业用户接入至固定二级再进行签发。` |
| 平台系统注释 | 行首 `*` + UI 显示提示 | `*如接收预警地区右名字体，请滚动条拖拽，请核查。` |

**读取铁律**（4 条）：

1. 🚨 **只摘抄**从「[气象台名]…发布[灾种][颜色]预警信号：」开始的实际预警文字
2. 🚨 **不**把平台注释算入字数
3. 🚨 **不**对平台注释做错别字/语病/重复审核
4. 🚨 **不**对平台注释报警

🚨 OCR 完成后，必须在 JSON 输出里单列 `platform_comments_excluded` 字段记录**已排除**的平台注释（让主 agent 可追溯）。

## 输出格式（Frozen Facts JSON）

```json
{
  "warn_id": "2026-08-01-XXXX",
  "title": "...",
  "disaster_type": "...",
  "category": "...",
  "publisher": "...",
  "publish_time": "2026-08-01T18:30:00+08:00",
  "validity_hours": 6,
  "affected_areas_display": "白碱滩区 +2",
  "content": "...",
  "defense_guide": "...",
  "warning_standard": "...",
  "audience": "...",
  "signer": "...",
  "screenshot_source": "qqbot/downloads/...",
  "platform_comments_excluded": ["...", "..."],
  "ocr_confidence": {
    "high": ["title", "disaster_type", "publisher"],
    "low": ["content"]
  }
}
```

🚨 **铁律**：
- 字段名严格按主 skill 字段定义（14 个必填字段）
- 时间格式 ISO 8601 + 时区（如 `2026-08-01T18:30:00+08:00`）
- 文本字段不修改、不补全、不猜测
- 模糊字段标 `null` 或 `[无法确认]`
- **必带** `platform_comments_excluded`（即使为空数组 `[]`）
- **必带** `ocr_confidence`（标注每个字段置信度）

## SOP-A · 强制重新读图（最重要）

- 每次输出 JSON 前，**必须**调 `read` 工具重新读图（**不依赖短时记忆**）
- 重新读图后**逐字校验**所有 14 个字段
- 输出前，对照"读图结果"与"JSON 字段"——不一致就重读

## SOP-B · 二次怀疑机制

每报告一个字段前，自问：

1. **"我是真的从图里看到的吗？还是我假设的？"**
2. **"我用 `read` 工具重新读图了吗？"**

🚨 **铁律**：2 问里有 1 个没把握 → **不写入 JSON**，重新核对。

## SOP-C · 雪球效应熔断器

- 一旦发现**自己开始找证据支持假设**（而非客观比对），立即停止
- 触发关键词警觉 ⚠️："果然……""确实……""这正是……"
- 重读图像 + 重新列差异，**不预设结论**

## 触发场景

- **由主 skill `weather-warn-pipeline` 派发**（`taskName="ocr-{warn_id}"`）
- ❌ **不**直接被老板调用
- ❌ **不**直接接收老板的预警截图（截图由主 agent 转发）

## 修订记录

| 日期 | 修订 | 触发 |
|------|------|------|
| 2026-08-01 | 初版：从原 weather-warn-audit 剥离 OCR + 读图 SOP + Frozen Facts 生成；作为子 skill 由主 skill 派发；输出 14 字段 JSON 含 `platform_comments_excluded` + `ocr_confidence` | 老板指令：拆分 audit skill，OCR 独立 |