# 多项目音轨重合问题汇总与解决方案

本文档汇总对 Langfuse 生产 trace 的归因结论与可执行对策，涉及项目：**84529123884**、**51593088086**、**32711810012**。原始 trace 仅保存在本地分析目录，不入库。

---

## 一、问题概览

| 项目 ID | 主要现象 | 根因摘要 |
|---------|----------|----------|
| **84529123884** | 多段人声/音轨糊在一起 | 8 段 `video_generate` 均为 Kling **`sound=on`**（片内 AAC），同时又叠多条 **`audio_produce`（TTS）**；成片 `execute_edit` 中 VO 的 `start` 近似按固定间隔（约 10s）摆放，**单条 TTS 实际时长常大于间隔**，导致相邻 VO 时间重叠。 |
| **51593088086** | 全片旁白与画面声音「双层」 | 8 段 **`sound=on`** + **一条超长全片 TTS**（约 1892 字）贯穿时间轴；首次成片 `edit_spec` 中**视频轨 `volume` 均为 1**，片内同期声与旁白同时满音量混合。同项目后续 trace 将视频轨 **`volume` 改为 0**，属正确修复方向。另：曾出现 **16 段视频为同一组 8 段素材重复接两遍**的编辑结构，易引发重复叙事，需单独排查。 |
| **32711810012** | 教程感「音轨叠在一起」 | 7 段 **`sound=on`**；成片视频轨 **`volume=0.5`**（片内声仍明显）；分段 TTS + **BGM 从 `start=0` 全片铺**，与 VO 长期同轨混音，听感拥挤。 |

**共性结论**：在存在后期 **TTS / 分段旁白 / 全片 VO** 的前提下，仍保留 **`sound=on` 片内对白轨** 且混音时未将视频轨视为「纯画面」，导致 **两路（或多路）主力人声/解说** 与 **BGM** 不合理叠加。

---

## 二、解决方案（按落地优先级）

### 1. 硬性策略：后期要铺 TTS 时，片内不再保留对白轨

- **推荐**：所有将参与组片、且计划叠 TTS 的 `video_generate` 使用 **`sound=off`**，叙事人声只走 `audio_produce`。
- **若必须保留片内口型/同期声**：则**不得**再叠同角色的长段 TTS 作为主线解说；配乐可走独立 BGM 轨。

**禁止模式**：多段 `sound=on` + 全片或分段 TTS 同时作为「主叙事人声」。

### 2. 混音：`execute_edit` 视频轨默认静音或极低环境

- 存在独立 VO / TTS 时，视频轨各 clip 设 **`volume: 0`**（与 515930 修复版一致）。
- 若产品需要极弱环境声，用 **0.05–0.15** 量级并明确为「仅环境」，避免使用 **0.5** 作为折中。

### 3. 分段 VO 排轨：用 ffprobe 实测时长，禁止固定时间网格

- 每条 `audio_produce` 后 **`ffprobe` 取 `format.duration`**，再计算下一条的 `start`（帧）。
- 保证：`start[i+1] ≥ start[i] + duration_frames(i) + gap_frames`（建议预留 **12–24 帧** 气口）。
- 文案过长则 **改稿或拆视频段**，勿在同一画面长度内硬塞超长 VO。

### 4. BGM 与 VO：ducking 写死

- VO 存在时：BGM **低音量**（如 **0.08–0.15**）+ **`fade_in` / `fade_out`**。
- 避免 BGM 与第一条 VO 同起点却电平过大导致「糊」。

### 5. 全片单条旁白（515930 类）

- **全片一条 VO**：必须 **`sound=off` + 成片视频轨 `volume: 0`**。
- 禁止「8 段 sound-on + 一条盖全片的长 TTS」且视频轨仍为满音量。

### 6. 成片前自检清单

1. 是否有 `audio_produce` / 多段 VO？→ 是则相关 **`video_generate` 须 `sound=off`**，或编辑里 **视频 `volume=0`**。
2. 相邻 VO 的 **`start` 间隔** 是否 ≥ **ffprobe 时长 + 气口**？
3. BGM 与 VO 同起点时，BGM 是否足够低、是否有淡入淡出？
4. 视频 clip 的 **`fade_out`** 与轨间 **`crossfade`** 是否互斥（避免同刀口双效果，参见 generation-skill / video-models-scheduling 既有规则）。

### 7. 建议写入 Skill 的一句话（供粘贴）

> 当计划使用 `audio_produce`（含全片或分段 VO）在 `execute_edit_video` 中混音时：所有对应 `video_generate` 必须使用 **`sound=off`**；若素材已为 sound-on，则成片 `edit_spec` 中视频轨所有 clip 的 **`volume` 必须为 `0`**。禁止片内同期对白与 TTS 解说同时作为主力人声。VO 的 **`start` 必须基于 ffprobe 实测时长逐条累加**，禁止使用固定时间网格摆放相邻 VO。

---

## 三、数据与追溯说明

- 分析数据来源：Langfuse 拉取的 trace（本地路径示例：`analysis/langfuse-data/cases/<项目ID>/`）。
- 关键工具链字段：`video_generate` 的 `sound`；`audio_produce` 文本长度；`video-editor__execute_edit_video` 的 `edit_spec.tracks`（`kind: video|audio`、`volume`、`start`、`in`/`out`）。

---

## 四、修订记录

| 日期 | 说明 |
|------|------|
| 2026-04-20 | 初版：三项目归因 + 解决方案汇总 |
