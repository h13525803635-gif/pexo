# Case 26513781249：协作 App 30s 自述片 — 女声旁白 + 未用 audio_list 口型驱动

**Langfuse**：`pexo:26513781249`（1 trace：`trace-1-0ccb6a0b.json`）  
**用户 brief**：30s 16:9 真人实拍风；落地窗前年轻**男性**对镜说话 → B-roll 平板批注 / 视频会议；**旁白即为该角色的自述**  
**成片**：`final_explainer_30s.mp4`（30.1s）  
**Trace / 媒体**：`analysis/langfuse-data/cases/26513781249/`

---

## 现象

| 用户反馈 | 技术表现 |
|----------|----------|
| 旁白听起来是女声 | B-roll 段 TTS 使用 `voice_id: siqi_liu`（catalog 标 male，但 warm/soft，听感偏柔） |
| 不像同一男性在自述 | 开场用 Seedance **协生对白**，B-roll 才叠 **后期 TTS**，两套发声来源 |
| 口型/音色与画面男性不符 | 三条 `video_generate` **均未传 `audio_list`**，未走 TTS 驱动口型 |

---

## 会话概览

| 阶段 | 动作 | 关键参数 |
|------|------|----------|
| 定妆 | `image_generate` → `character_ref` | 年轻商务男性，共享办公区落地窗 |
| 视频 | `video_generate` ×3 并行 | Seq1：`sound:on` + prompt 中文台词；Seq2/3：`no speech` |
| 配音 | `audio_produce` → `voiceover_main` | `eleven_v3`，`voice_id: siqi_liu`，约 25.2s |
| BGM | `music_generate` → `bgm_acoustic` | 原声吉他，35s instrumental |
| 合成 | `execute_edit_video` | Seq1 内嵌音 volume 1.0；TTS 从 ~10.5s 起叠在 B-roll，**未覆盖开场** |

---

## 根因

### P0-1 — 未执行 `audio_list_tts`（应传音频给 Seedance 对口型）

Brief 要求「对着镜头自信地说话」且「旁白即为角色自述」。Skill 规定画内可见说话应走：

- 先 `audio_produce` 定稿 TTS（固定 `voice_id`）
- 再 `video_generate` 传入 **`audio_list`** → `planned_source: audio_list_tts`
- 口型由音频驱动，合成时**禁止再叠同文案 TTS**

Agent 推理中写过「开场 `on_camera_sync_speech`，B-roll `post_tts_vo`」，但执行时：

| 镜头 | 应有 | 实际 |
|------|------|------|
| seq1_presenter | `audio_list` + 男声 TTS | `audio_list: null`，`sound: on`，prompt 内写台词 → **协生对白** |
| seq2/3 B-roll | 静音画面 + 同音色 post TTS | 画面静音，剪辑叠 `siqi_liu` TTS |

### P0-2 — 音色选择与角色不匹配

Brief / 定妆均为年轻男性。更贴合的 catalog 男声包括 `james_gao`（产品叙述）、`haoran`（磁性广告）、`danyu_zhao`（讲解类）。

实际选用 `siqi_liu`（warm, soft，情感叙事向），用户易听成女声，且与「商务男性自述」气质不符。

> 注：catalog 将 `siqi_liu` 标为 male，但 timbre 偏柔，存在 **标注 gender vs 听感** 偏差。

### P0-3 — 合成时间线分裂「旁白」

| 时间段 | 成片听到的说话/旁白 |
|--------|---------------------|
| 0–10s | Seq1 视频内嵌协生对白（非 TTS） |
| 10s+ | `voiceover_main` TTS 切片叠在 B-roll（`start_ms: 10054` 起） |

违背「全片同一角色自述」的单一声线预期。

### P1 — 误用画外 VO 路径

读了 `vo-design-principles.md`（scope：**off-screen VO**）。本案应走 `dialogue-monologue-design-kb` / **unified character narration**：可见说话 `audio_list_tts`，画外延续同 `voice_id` 的 `post_tts_vo`。

---

## 正确重制流程（推荐）

```text
1. audio_produce → 男声 TTS（如 james_gao），可按 seq 切段或一条 master 再切
2. seq1_presenter:
     video_generate(
       image_list=[character_ref],
       audio_list=[seq1 TTS 文件],    ← 口型驱动
       prompt: 镜头/表演/环境，不写台词、不依赖协生对白
     )
3. seq2/seq3: no speech + 静音生成
4. 合成:
     - seq1 保留内嵌 TTS（勿再叠同文案 audio 轨）
     - B-roll 铺同 voice_id 的 post_tts
     - BGM + 环境音
```

**禁止**：`co_gen_dialogue`（sound:on + speaks prompt）与 `post_tts_vo` 同句双轨。

---

## Skill 改进建议

| 优先级 | 改动 |
|--------|------|
| P0 | brief 含「旁白即为角色自述」→ 强制 `unified_character_narration`，单一 spoken lineage |
| P0 | 对镜说话必须 `audio_list_tts`；generation 校验 `audio_list` 非空 |
| P0 | `voice_id` 必须匹配 `character_gender` + 场景（商务自述 → james_gao/haoran 等） |
| P0 | 禁止 co_gen + post_tts 同片；assembly 双 spoken 轨 fail-closed |
| P1 | script-skill 分镜表必填 `planned_source`、`assembly_guard` |

---

## 关键工具参数摘录

**TTS（实际调用）**

```json
{
  "name": "voiceover_main",
  "model": "eleven_v3",
  "voice_id": "siqi_liu",
  "text": "以前开完会，我要花半小时整理跟进事项……"
}
```

**Seq1 视频（缺 audio_list）**

```json
{
  "name": "seq1_presenter",
  "mode": "reference2video",
  "audio_list": null,
  "sound": "on",
  "prompt": "... speaking candidly: \"以前开完会……\""
}
```

---

## 相关文件

| 文件 | 说明 |
|------|------|
| 本报告 | `analysis/26513781249_30s-collab-explainer-audio-route.md` |
| 素材元数据 | `analysis/langfuse-data/cases/26513781249/assets.json` |
| 带 prompt 映射 | `analysis/langfuse-data/cases/26513781249/assets-with-prompts.json` |
| Trace | `analysis/langfuse-data/cases/26513781249/trace-1-0ccb6a0b.json` |
| 本地媒体 | `analysis/langfuse-data/cases/26513781249/media/` |
| 同类 case | `analysis/46021149558_30s-fitness-narrator-voice-doubling.md` |
