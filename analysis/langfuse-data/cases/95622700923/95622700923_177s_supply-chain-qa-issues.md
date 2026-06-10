# Case 95622700923 — 供应链科普片 QA 问题根因与解决方案

| 字段 | 值 |
|------|-----|
| 项目 ID | `95622700923` |
| Langfuse trace | `52e20c0043187c89a9085fc2d796091a`（737 observations，trace 本体 >100MB 未完整入库） |
| 主题 | Supply Chain Disruptions 科普片（英文旁白 + B-roll） |
| 成片 | `Supply_Chain_Disruptions_v2.mp4`（177s 时间轴） |
| 合成方式 | HyperFrames HTML → `submit_render` / `query_render` |
| 分析日期 | 2026-06-10 |

---

## 1. 问题现象

| # | 现象 | 严重程度 |
|---|------|----------|
| P1 | 字幕重叠且跟音频不对位 | P1 |
| P2 | 卡片覆盖在画面上方 | P1 |
| P3 | 会议情节人物突然讲中文，且音频重叠 | P1 |
| P4 | 后半段视频与视频黑屏间隔特别长 | P1 |
| P5 | 最后约 20 秒黑屏 | P1 |

---

## 2. 真实调用路径

```
用户请求
  └─ Agent (Claude Sonnet 4.6)
       ├─ [09:47] video_generate ×13  (happyhorse-1.0-t2v, dashscope, 各 ~10s)
       ├─ [09:53] music_generate ×1  → sc_bgm
       ├─ [09:54–09:55] audio_produce ×2 → sc_vo_part1 (79.75s) + sc_vo_part2 (96.955s) + SRT
       ├─ [09:56] probe_media ×3  → 全部失败（未验证真实时长）
       ├─ [09:59–10:02] write_file → supply_chain_video.html
       ├─ [10:00] lint_composition → 工具不存在，校验跳过
       ├─ [10:00] submit_render #1 → Supply_Chain_Disruptions.mp4 (job 39cfxocs)
       ├─ [10:04] edit_file 多次修补（字幕、clip class、签名 URL）
       ├─ [10:17] submit_render #2 → Supply_Chain_Disruptions_v2.mp4 (job 65ezhcjk)
       └─ [10:31] show_final_video → v2 交付
```

**注意**：本项目未走 `execute_edit_video`，全程 HyperFrames HTML 合成。

---

## 3. 根因分析

### P1 — 字幕重叠且跟音频不对位

**现象链**：
1. Agent 先写旁白、再按「凑满 177s」硬切 13 段 B-roll，**未按 SRT 句点对齐画面**。
2. 多层文字叠加：0–5s title card 副标题 + SRT burn-in；22s/50s/74s chapter 卡片（bottom:120px）与 SRT（bottom:60px）底部区域重叠。
3. v1 渲染 `sub_vo1` 使用无效 `asset://` 引用，part2 字幕 initially 缺失；v2 补上但未修时间轴对齐。

**关键证据**（VO vs 画面）：

| 时间 | 旁白 (SRT) | 实际画面 |
|------|-----------|----------|
| 70.3s | "empty shelves" | S7 会议室 (62–74s) |
| 75.7s | "longer lead times" | S8 供应商网络 (74–86s) |
| 79.8s | Part2 recovery 开始 | 仍在 S8 |

SRT 与 TTS 音频同步（part1 79.75s + part2 96.955s = 176.705s），但 burn-in 字幕配的是 B-roll 切换时间，观感为「声画字幕三方错位」。

---

### P2 — 卡片覆盖在画面上方

**根因**：v2 修补时给 chapter/title 卡片加了 `class="clip"`，触发全屏 CSS：

```css
.clip { position:absolute; top:0; left:0; width:100%; height:100%; opacity:0; }
```

`.chapter-card` 原本为左下角小标签（`left:80px; bottom:120px`），叠加 `.clip` 后变成 100%×100% 全屏容器。GSAP 将 opacity 拉到 1 时，半透明背景盖住整个画面。

Title card 本身也是全屏遮罩（`rgba(0,0,0,0.55)`），0–5s 整屏覆盖 B-roll。

---

### P3 — 会议情节中文对话 + 音频重叠

**根因 A**：`sc_s7_boardroom_crisis` 使用 HappyHorse，`prompt` 含 "intense discussion and strategy session"，模型生成 **带中文对话的 native audio**。HTML 写了 `muted`，但 HyperFrames 渲染可能仍混入视频音轨。

**根因 B**：违反 VO 独占原则——Track 20 英文 TTS 旁白与 Track 1 S7 潜在 embedded 中文对话在 62–74s 同时播放。

生成时未显式 `sound=off`，未做 speech audit（assembly-skill 要求 B-roll 不得叠 spoken audio）。

---

### P4 — 后半段黑屏间隔长

**根因**：HTML 时间轴 slot 时长 >> 素材实际时长（HappyHorse 源片 ~10s），且 **未配置 loop**。

| 片段 | Slot | 超出 10s | 黑屏起始 |
|------|------|----------|----------|
| S6 | 12s | 2s | 60s |
| S7 | 12s | 2s | 72s |
| S8 | 12s | 2s | 84s |
| S9 | 14s | 4s | 96s |
| S11 | 14s | 4s | 120s |
| S12 | 14s | 4s | 134s |
| **S13** | **39s** | **29s** | **148s** |

注释写 `39s = loop` 但无 `loop` 属性。`probe_media` 三次全部失败，Agent 从未验证真实 clip 时长。

---

### P5 — 结尾约 20 秒黑屏

**根因**：S13 占 138–177s（39s slot），源素材 ~10s，**148s 起约 29s 无画面**。VO/BGM 在 176.7s 结束，但视频轨 148s 后已黑屏 → 声画不同步 + 长尾黑屏（用户感知 ~20s）。

---

## 4. Skill 归因

| 规则来源 | 应触发 | 本 case |
|----------|--------|---------|
| generation-skill：B-roll `sound=off` | 禁止 co-gen speech | 未设置 |
| video-models-routing：slot ≤ 源时长或 loop | 时长映射 | 违反（S13 等） |
| assembly-skill：probe 后校验时长 | 写 HTML 前 probe | probe 全失败仍继续 |
| assembly-skill：VO-driven 切点 | 按 SRT 对齐 B-roll | 未做 |
| assembly-skill：speech audit | 检出 speech → mute/regen | 未做 |
| hyperframes guide：卡片勿复用 `.clip` 全屏样式 | CSS 隔离 | 违反 |
| lint_composition | 合成前校验 | 工具不可用，跳过 |

---

## 5. 解决方案

### 5.1 generation-skill / video-generation-execution

| 改动 | 内容 |
|------|------|
| **新增** | B-roll（含会议/讨论类场景）必须显式 `sound=off`；prompt 禁用 discussion/dialogue/meeting speech 等暗示词，改用 "silent boardroom strategy review" |
| **新增** | HappyHorse 生成后 mandatory `probe_media`；probe 失败则 block 进入 assembly |
| **新增** | `data-duration` 不得超过 probe 返回时长；超出时必须 `data-loop="true"` 或换更长源素材 |

### 5.2 assembly-skill / hyperframes assembly guide

| 改动 | 内容 |
|------|------|
| **新增** | **VO-first 时间轴**：以 SRT cue 时间为锚点切 B-roll，禁止先等分画面再贴旁白 |
| **新增** | Chapter/title 卡片使用独立 class（如 `.overlay-card`），**禁止**叠加 `.clip` 全屏样式 |
| **新增** | 字幕层 z-index 与 chapter 卡片互斥时段检查；title card 时段（0–5s）暂停 SRT burn |
| **新增** | 合成前 lint：任一 video slot > probe duration 且无 loop → hard block |
| **修改** | `muted` 不够时，对已知 speech-bearing 片段在合成层强制 volume=0 或换素材 |

### 5.3 本 case 修复清单（重制时）

1. 重新 probe 全部 13 段视频，按真实时长重算时间轴（总长约 130s 或对各段 loop 填充至 177s）。
2. 按 SRT 句点重排 B-roll：如 "empty shelves" → S2，"boardroom/crisis" → S7。
3. S7 重生成：`sound=off` + 无对话暗示 prompt；或换 Seedance 无声 B-roll。
4. 移除 chapter 卡片上的 `.clip` class，恢复 `left/bottom` 定位。
5. S13：要么 `loop` 填满 39s，要么缩短 slot 至 ≤10s 并延长 BGM/VO 结尾画面。

---

## 6. 文件索引

| 文件 | 说明 |
|------|------|
| `analysis/langfuse-data/cases/95622700923/observations-index.json` | 737 条 observation |
| `analysis/langfuse-data/cases/95622700923/tool-timeline.json` | 296 次 tool 调用序列 |
| `analysis/langfuse-data/cases/95622700923/supply_chain_video.html` | 合成 HTML（v1 基线） |
| `analysis/langfuse-data/cases/95622700923/sc_vo_part1.srt` | 旁白 part1 字幕 |
| `analysis/langfuse-data/cases/95622700923/sc_vo_part2.srt` | 旁白 part2 字幕 |
| `analysis/langfuse-data/cases/95622700923/qa-report.html` | 可视化 QA 看板 |
