---
name: weather-warn-ocr
description: "气象预警截图 OCR 子 skill：用 PaddleOCR 3.x（paddlepaddle 3.2.2）做字符级 OCR，输出 Frozen Facts。"
metadata:
  {
    "openclaw":
      {
        "emoji": "📸"
      }
  }
---

# 气象预警 OCR 子 Skill（2026-08-01 切换 PaddleOCR）

由主 skill `weather-warn-pipeline` 派发。
**只**负责读图 + 调用Paddle OCR工具 逐字段 OCR + 生成 Frozen Facts。

**OCR 工具**：PaddleOCR 3.x + paddlepaddle 3.2.2（CPU 推理）。

## 受 FACT LOCK / Evidence Priority 约束（引用主 skill）

🚨 **铁律**：遵守主 skill 的 FACT LOCK + Evidence Priority。

## 读图步骤（强制顺序 · Step0~Step4 · 不得跳步）

| Step | 动作 | 说明 |
|------|------|------|
| **Step 0** | 读取图片 | 用 `Paddle OCR` 工具读图，禁止使用其他方式读图 |
| **Step 1** | 逐字段 OCR | 14 个字段（含 `warn_id`）逐个 OCR |
| **Step 2** | 生成 Frozen Facts | 按主 skill 字段定义整理为 JSON |
| **Step 3** | 冻结 | OCR 结果**立即冻结**，**不得再修改** |
| **Step 4** | 输出 | 返回 Frozen Facts JSON 给主 agent |

## OCR 输出约束

🚨 **禁止**：修改文字 / 补全 / 猜测模糊 / 依据模板修改 / 多数决覆盖图片。

| 情形 | 处理 |
|------|------|
| 能确认 | **原样摘抄** |
| 不能确认 | `null` 或 `[无法确认]` |
| 模糊 | **不得猜测** |

## 预警内容读取边界（防读多内容 · 7/28 23:20 老板纠错）

**只摘抄**从「[气象台名]…发布[灾种][颜色]预警信号：」开始的实际预警文字。平台系统注释（行首 `*` 或 UI 提示）**不算正文**，必须在 JSON `platform_comments_excluded` 字段记录**已排除**的内容。

## PaddleOCR 调用规范

### 安装

```bash
# 必须 paddlepaddle==3.2.2（3.3.x 有 OneDNN + PIR bug：NotImplementedError）
# 见 GitHub Issue: PaddlePaddle/Paddle#77340 + PaddlePaddle/PaddleOCR#18162
pip install --break-system-packages --ignore-installed PyYAML paddleocr paddlepaddle==3.2.2
```

### 调用

```python
from paddleocr import PaddleOCR

engine = PaddleOCR(
    lang='ch',
    use_doc_orientation_classify=False,
    use_doc_unwarping=False,
    use_textline_orientation=False,
)

result = list(engine.predict(image_path))   # generator → list
page = result[0]                            # 单图
texts = page['rec_texts']    # list[str]   识别文本
scores = page['rec_scores']  # list[float] 置信度
boxes = page['rec_boxes']    # np.ndarray shape=(N, 4) 矩形 [x1,y1,x2,y2]
```

🚨 **字段名注意**（PaddleOCR 3.x）：
- ✅ `rec_texts` / `rec_scores` / `rec_boxes`（复数）
- ❌ 不是 `rec_text` / `rec_score` / `rec_box`（老版本）

### 已知问题

- **paddlepaddle 3.3.x 不能用** —— 必须 3.2.2
- `ReduceMeanCheckIfOneDNNSupport` 警告 = **无害**（OneDNN 路径正常工作日志）
- OneDNN 不能简单关掉（PaddleX 强制用）

## 输出格式（Frozen Facts JSON · 14 字段）

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
  "platform_comments_excluded": [],
  "ocr_confidence": {"high": ["title"], "low": ["content"]}
}
```

🚨 **铁律**：时间 ISO 8601+时区 · 模糊字段 `null` 或 `[无法确认]` · 必带 `platform_comments_excluded` + `ocr_confidence`。

## 触发场景

- **由主 skill `weather-warn-pipeline` 派发**（`taskName="ocr-{warn_id}"`）
- ❌ **不**直接被老板调用 / ❌ **不**直接接收截图

## 修订记录

| 日期 | 修订 | 触发 |
|------|------|------|
| 2026-08-01 20:02 | 修订：删除老板 GitHub 版本遗留下的 `### ` 空标题（L81） | 报告瑕疵后老板指令"修复，并上传" |
| 2026-08-01 19:48 | 重大修订：删除 SOP-A · 强制重新读图 / SOP-B · 二次怀疑机制 / SOP-C · 雪球效应熔断器 三个章节 | 老板指令："把 OCR skill 的 sop 全部去除"，后续精修"只删 SOP-A/B/C，其余不删" |
| 2026-08-01 19:21 | **切换 OCR 工具**：RapidOCR → PaddleOCR 3.x；新增安装/调用/字段名规范；强调 `paddlepaddle==3.2.2` workaround | 老板指令："修改 skill，使用 paddleocr 识图" |
| 2026-08-01 | 初版：基于 RapidOCR + MiniMax-VL | 老板指令：拆分 audit skill |