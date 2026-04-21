# Case 97468242117 — 音频叠加与成片分析

**品牌**：Cheetah Airlines  
**需求**：将 AI 生成的女性办公室代理形象制作成说话人动画视频，带精准口型同步和专业配音  
**时间**：2026-04-20 12:39–12:52（约 13 分钟，3 个 trace）

---

## 制作流程

| Trace | 时间 | 用户输入 | Agent 动作 |
|-------|------|---------|-----------|
| trace-1 | 12:39 | 详细 brief：角色动画 + 口型同步 + voiceover 文本 | 分析 brief，推荐 16:9/16s 格式；尝试生成 style frame 预览图（**失败 6 次**）；向用户展示创意方向等待确认 |
| trace-2 | 12:41 | "yes please" | 生成 TTS 配音（ElevenLabs 22.83s）；生成 2 段 Kling 视频（各 10.04s）；合成时视频轨 volume=0.3，**两路声音叠加** |
| trace-3 | 12:51 | "please remove the overlapping voice that doesn't match the lips" | 将视频轨 volume 改为 0，重新渲染 `cheetah_airlines_v2.mp4`，修复叠声问题 |

---

## 关键问题

### 1. 音频叠加（已修复）

**根因**：Trace-2 合成时视频轨 `volume=0.3`，保留了 Kling co-generate 的声音，与 TTS VO 同时播放。  
**修复**：Trace-3 将视频轨 volume 设为 0，只保留 TTS 轨。

**规避方式**：多轨合成时，凡使用 TTS 作为主音轨，视频轨 co-gen 音频应默认 `volume=0`。

### 2. Style frame 预览图生成失败

`image_generate` 连续 6 次报错：`image billing requires size or image_size`。  
参数缺少 `size` 或 `image_size` 字段。影响：无法向用户展示可视化预览，但不影响最终成片。

### 3. VO 与视频时长不对齐

- TTS 生成时长：22.83s
- 视频总时长：20.08s（两段 Kling 各 10.04s）
- 尾部约 2.75s 配音被截断

脚本朗读时长估算偏短，应在生成 TTS 后先 ffprobe 测量时长，再决定视频段数。

### 4. 口型同步为近似同步，非精准同步

Kling 靠 prompt 描述生成口型动作，而非以 TTS 音频为驱动。Agent 在 Trace-3 末尾主动提出可用 TTS 作为 reference input 重新生成，用户未继续跟进。

---

## 最终成片参数

| 项目 | 值 |
|------|---|
| 文件 | `cheetah_airlines_v2.mp4` |
| Asset ID | `a_N2eSCba` |
| 规格 | 1280×720，24fps，20.08s |
| 视频 | 2 段 Kling kling-v3-omni，各 10s，hard cut 拼接 |
| 音频 | ElevenLabs eleven_v3，女声英语，截至 20.08s |
