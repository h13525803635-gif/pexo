# Case 35802589870 — 无画内口述、旁白与口型不一致：根因分析

| 字段 | 值 |
|------|-----|
| 项目 ID | `35802589870` |
| Langfuse trace | `c43b25476431709e66a5def99dd42f0e`（`trace-1-c43b2547.json`） |
| 用户 brief | 30s 16:9 真人实拍；公园长椅年轻男性**对镜说话**；B-roll 手表/手机；**旁白=角色自述** |
| 成片 | `fitness_app_explainer_30s.mp4` |
| 分析日期 | 2026-05-19 |
| 关联 case | 66831923061、46021149558、62861322045 |

---

## 1. 问题现象

| # | 用户反馈 | 技术表现 |
|---|----------|----------|
| P0-1 | 没有口述（画内口播） | Seq1/Seq4 画面「在说话」，但片段内**无人声** |
| P0-2 | 口述应与旁白一致 | 仅有后期 `voiceover_main` TTS；口型与中文旁白不同步 |
| P1 | 旁白存在但像画外音 | 全片叠一条 Post-TTS，非角色嘴型驱动 |

---

## 2. 根因链

### 2.1 误判「旁白=自述」为画外 VO（P0-1 / P0-2）

用户原文：

> 对着镜头轻松地说话 … **旁白即为该角色的自述**

应走 **单一说话人 + 画内同步口播**（`on_camera_sync_speech` 或 `audio_list_tts`），而非 off-screen `post_tts_vo`。

**Agent 内部计划却写：**

- 视觉：Seq1/Seq4「男主对镜头自述 / 微笑说话」
- 音频：**旁白：Post-TTS**，`voice_id=siqi_liu`

→ 规划层就把「自述」拆成「画面演口型 + 画外配音」，与用户意图不符。

### 2.2 视频 prompt 禁止人声（P0-1）

`seq1_park_bench_talk`、`seq4_closing_smile` 同时包含：

- `He moves his hands slightly as he speaks` / `he speaks with a warm confident smile`
- **`No speech, no dialogue, no voice, no spoken words`**

且**未**传入 `audio_list`，也**未**在 prompt 写入 L1/L3 台词做 co-gen 对白。

### 2.3 仅有后期旁白轨（P0-2）

| 步骤 | 结果 |
|------|------|
| `audio_produce` → `voiceover_main` | 26.7s 中文 TTS ✅ |
| `video_generate` seq1/seq4 | 仅环境音 ❌ |
| ASR（scribe_v2）seq1/seq4 | 仅 `[birds chirping]`，无对白 |
| `execute_edit_video` | 视频轨 + `voiceover_main` + BGM + 字幕 |

成片 = 静音口型视频 + 全片 post VO → **无画内口述，旁白与嘴型不一致**。

---

## 3. 正确做法（本案例）

1. 用同一 `voice_id` 生成 L1、L3（及 B-roll 段 L2）TTS。
2. **Seq1 / Seq4**：`audio_list` + 对应台词 → Seedance 口型同步（`audio_list_tts`）。
3. **Seq2 / Seq3**：B-roll 静音或仅环境音，同一条 VO 在剪辑时间轴对齐铺过去。
4. **禁止**在「对镜说话」镜头写 `No speech, no dialogue...`。
5. 若片段已嵌入同文案 speech，组装时不得再叠 `voiceover_main`（anti-doubling）。

---

## 4. Skill 归因

| 规则来源 | 应触发但未遵守 |
|----------|----------------|
| brainstorm-skill Speech-role lens | 「旁白=角色自述」→ 画内表演，非画外 narrator |
| script-skill Single speaker ownership gate | 同一角色不得 split 为 visible speak + post VO |
| script-skill Character-sync hard stop | 对镜/自述不得 `post_tts_vo` + `no speech` + `speaks` |
| video-generation-execution | 画内对白优先 co-gen 或 `audio_list`，不用 post 替代 on-screen dialogue |
| assembly-skill AUDIO AUDIT | ASR 已证 silent，但应用 `audio_list` 重生成而非仅叠 VO |

---

## 5. 关键证据（trace）

- 用户 input：`旁白即为该角色的自述`
- 计划：`旁白：Post-TTS`
- seq1 prompt 末尾：`No speech, no dialogue, no voice, no spoken words`
- ASR seq1/seq4：`"text":"[birds chirping]"`
- 合成：`voiceover_main_*.mp3` 全片 `start_ms:0`，无 `audio_list` 口型绑定

原始 trace：`analysis/langfuse-data/cases/35802589870/trace-1-c43b2547.json`
