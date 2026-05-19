# Case 46021149558：健身 App 自述片 — 女声旁白 + 画内男声双轨

**Langfuse**：`pexo:46021149558`（1 trace：`trace-1-739643c9.json`）  
**用户 brief**：30s 16:9 真人实拍风；公园长椅年轻**男性**对镜说话 → B-roll 手表/手机；**旁白即为该角色的自述**  
**成片**：`fitness_app_explainer_v1.mp4`  
**Trace 数据**：`analysis/langfuse-data/cases/46021149558/`

---

## 现象

| 用户反馈 | 技术表现 |
|----------|----------|
| 既有旁白又有画内主角讲话 | Seq1：Seedance 协生男声对白 + ElevenLabs 女声 TTS 同时叠在成片 |
| 旁白是女声、画面是男生 | 全片 TTS 使用 `voice_id: siqi_liu`（女声），未按角色性别选男声 |

---

## 会话概览

| 阶段 | 动作 | 关键参数 |
|------|------|----------|
| 定妆 | `image_generate` → `char_ref_main.png` | 年轻东亚男性运动休闲装 |
| 配音 | `audio_produce` ×4 → `vo_seq1/2/3` | 同一 `voice_id: siqi_liu`（女声） |
| 视频 | `video_generate` ×3 并行 | Seq1：`sound:on` + prompt「speaks…mouth moving」；Seq2/3：`No speech` |
| 合成 | `execute_edit_video` | 视频轨保留内嵌音 + 独立 VO 轨叠 TTS，**未 mute Seq1 对白** |

---

## 根因

### P0-1 — 规划与执行不一致（双 spoken 轨）

Agent 推理中决定 Seq1 用 `audio_list_tts`（TTS 驱动口型），实际 `video_generate` **未传 `audio_list`**，反而：

- `sound: "on"`
- Prompt：`He speaks directly… mouth visibly moving as he talks`

→ Seedance **协生男声对白**烘焙进 MP4（ffprobe：`nb_streams: 2`，含 AAC）。

同时又 `audio_produce` 生成 TTS，合成时叠在视频轨上 → **开场段双声源**。

### P0-2 — 音色性别与角色不匹配

Brief / 定妆均为年轻男性，TTS 却选 `siqi_liu`（女声 catalog），未按 `vo-design-principles` §6 做 gender 匹配。

用户感知：画内男生 + 画外女声解说，违背「旁白 = 角色自述」。

### P0-3 — 音频策略最差组合

思考链在 `co_gen_dialogue` / `post_tts_vo` / `audio_list_tts` 间切换，最终落地为：

**co_gen（Seq1）+ post_tts（全段）**，违反 `voice source exclusivity / anti-doubling`。

### P1 — 误用画外 VO 指南

`vo-design-principles.md` scope 为 **off-screen VO**；本案例为画内角色自述，应走 `dialogue-monologue-design-kb` / `unified_character_narration`。

---

## 解决办法

### 对本项目重制

**推荐：全片 `post_tts_vo` + 男声**

1. 换男声 `voice_id`（如 `james_gao` / `4VZIsMPtgggwNg7OXbPY`）
2. 三段 VO 同一 voice，建议单次长 VO 再切分
3. **重生成 Seq1**：prompt 加 `No speech, no dialogue`；或 `sound: off` + 仅环境音
4. 合成：**mute Seq1 视频内嵌对白**，只保留环境 + TTS + BGM

**若强口型**：全片 `audio_list_tts`，Seq1 传入 TTS `audio_list`，禁止同时 `sound:on` + speaks prompt。

### Skill 层（同 62861322045）

| 优先级 | 改动 |
|--------|------|
| P0 | `unified_character_narration`：brief 含「旁白即为角色自述」→ 单一 `spoken_lineage` |
| P0 | 禁止 `co_gen_dialogue` + `post_tts_vo` 同片；assembly 检测双 spoken 轨 fail |
| P0 | `voice_id` 必须匹配 `character_gender` |
| P1 | script-skill 分镜表必填 `spoken_lineage`、`narrator_mode` |

---

## 相关文件

- 素材元数据：`analysis/langfuse-data/cases/46021149558/assets.json`
- Trace：`analysis/langfuse-data/cases/46021149558/trace-1-739643c9.json`
- 媒体：`analysis/langfuse-data/cases/46021149558/media/`
