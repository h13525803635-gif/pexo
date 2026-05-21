# Case 34944715782 — 成片1 中文字幕全方框（缺 CJK 字体）

| 字段 | 值 |
|------|-----|
| 项目 ID | `34944715782` |
| 项目名 | 深圳天使母基金介绍视频 |
| 首版 trace | `a690d4e2`（`trace-1-a690d4e2.json`） |
| 首版成片 | `szangelfund_final`（约 47s，16:9，1280×720） |
| 日期 | 2026-05-20（首版）/ 2026-05-21（修复迭代） |

## 用户需求

搜索「深圳天使母基金」信息并制作约 45 秒机构介绍片：旁白 + 烧录字幕，专业机构气质。

## 问题摘要

| # | 现象 | 严重度 |
|---|------|--------|
| P1 | 成片1 烧录字幕**全部为方框**（□ / tofu） | P1 |
| P2 | 第二幕 B-roll 出现外籍面孔（与中国基金语境不符） | P2 |
| P3 | v2 字幕约 30s 处时间窗重叠/叠影 | P2 |
| P4 | v2 使用字符级「打字机」字幕，可读性差 | P3 |

---

## 根因链（P1 方框）

### 现象

- 旁白正常、字幕时间轴存在，但画面汉字全部显示为 **□**。
- 用户反馈（trace-3）：「1、字幕有问题，要做好；2、中间的不要外国人，这是中国基金」。

### 直接原因

首版 `execute_edit_video`（`name: szangelfund_final`）字幕轨 `kind: subtitle`，`source.file` 正确指向 TTS 返回的 `vo_seg*_subtitle_*.json`，但 **`text_opts` 未设置 `font`**：

```json
"text_opts": {
  "fontsize": "34",
  "fontcolor": "white",
  "borderw": "2",
  "bordercolor": "black",
  "alignment": "2",
  "margin_v": "36"
}
```

渲染器回退到**无 CJK 字形**的默认字体 → 中文码点无 glyph → 满屏方框。

### 对比：v2 修复后

同结构下增加 CJK 字体（trace-3 `szangelfund_v2`）：

```json
"text_opts": {
  "font": "noto-sans-cjksc",
  "font_weight": "500",
  "fontsize": "36",
  "fontcolor": "white",
  "borderw": "2",
  "bordercolor": "black",
  "alignment": "2",
  "margin_v": "50"
}
```

Agent 在修复前调用 `video-editor__list_fonts`，确认 `noto-sans-cjksc` 支持 `zh-Hans`。

### 非原因（已排除）

| 假设 | 结论 |
|------|------|
| TTS 未生成字幕 | ❌ 三段 VO 均返回 `subtitle_file` |
| 字幕 JSON 非中文/乱码 | ❌ Agent 确认为中文、字符级时间戳 |
| 时间轴算错导致方框 | ❌ 方框与 VO 时长无关，v1 时间基于 VO 绝对时间 |
| 选错 font id（如 inter） | ❌ v1 **完全未指定** font |

### 流程归因

- `assembly-skill` 要求合成前读 `references/video-editor-tool-guide.md`，首版仍漏写 `text_opts.font`。
- 技能未将「含 CJK 字幕且 `font` 缺失 → BLOCKED」写成硬停。

---

## 版本迭代

| 版本 | 文件 | 修复点 |
|------|------|--------|
| 成片1 | `szangelfund_final` | —（缺 CJK font） |
| v2 | `szangelfund_v2` | `noto-sans-cjksc` + 第二段中国面孔 B-roll |
| v3 | `szangelfund_v3` | 字幕段间 +200ms gap，消重叠 |
| v4 | `szangelfund_v4` | 整句静态字幕（替代逐字 typewriter） |

---

## 解决方案

### A. 本项目（立刻可用）

1. **对外交付 v4**（或至少 v2+），勿再展示 `szangelfund_final` 为默认成片。
2. **仅重烧字幕**：保留原视频/VO/BGM/`subtitle_file`，重跑 `execute_edit_video` 并在每条 subtitle clip 的 `text_opts` 加 `font: "noto-sans-cjksc"`。

### B. 合成规范（技术）

1. `audio_produce` → 使用返回的 **`subtitle_file`** 作为 `kind: subtitle` 的 `source.file`。
2. 合成前 **`video-editor__list_fonts`**，按 `spoken_language` 选 font id。
3. 简体中文默认：**`font: "noto-sans-cjksc"`**，**`font_weight: "500"`**（或 400）。
4. **禁止**仅写 `fontsize`/`fontcolor` 而不写 `font` 就提交含中文的烧录字幕。

### C. Skill 建议（防再犯）

在 **`assembly-skill` → Subtitle Hard Stops** 增加：

- 旁白/字幕含 **zh-Hans** 时，`text_opts.font` **必填**，且来自 `list_fonts` 且 `languages` 含对应语种。
- 提交 `execute_edit_video` 前自检：CJK 文案 + 缺 `font` → **BLOCKED**。

在 **`video-editor-tool-guide.md`** 增加常见错误：**中文满屏方框 = 未指定 CJK 字体**。

handoff 增加：`spoken_language: zh-Hans`、`subtitle_font: noto-sans-cjksc`。

### D. 产品兜底（可选）

`kind: subtitle` + `subtitle_file` 含 CJK 且未指定 `font` 时，渲染层默认 `noto-sans-cjksc`，避免静默回退西文字体。

---

## 关键证据（trace）

- 首版 edit spec：`analysis/langfuse-data/cases/34944715782/trace-1-a690d4e2.json`
- 用户反馈 + v2 修复：`trace-3-06d87ca0.json`（用户消息含「字幕有问题」；`list_fonts` → `noto-sans-cjksc`）
- v3/v4：`trace-5-f3871b0f.json`、`trace-7-e830ee2d.json`

---

## 交叉引用

- 同类方框 case（已选 font 仍方框，偏执行端映射）：`analysis/case-12336464341-字幕方框问题说明.md`
- 字幕重叠：`analysis/case-14194336910-字幕重叠问题说明.md`

---

## 修订记录

| 日期 | 说明 |
|------|------|
| 2026-05-21 | 初版：根因、方案、版本迭代（Langfuse trace 34944715782） |
