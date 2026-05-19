# Case 62861322045：同角色自述片 — 三段音色不一致 + 人物漂移

**Langfuse**：`pexo:62861322045`（1 trace：`trace-1-920eeb52.json`）  
**用户 brief**：30s 16:9 真人实拍风；咖啡厅年轻人对镜说话 → B-roll 打字/点手机；旁白为**同角色自述**  
**成片**：`final_assembly_v1.mp4`  
**Trace 数据**：`analysis/langfuse-data/cases/62861322045/`

---

## 现象

| 用户反馈 | 技术表现 |
|----------|----------|
| 三段音频音色不一致 | Seg1 对白来自 Seedance 烘焙轨（vol 0.9）；Seg2/3 来自 ElevenLabs MP3 直叠（vol 1.0） |
| 前后人物变了 | 仅 Seg1 绑定 `cafe_presenter_ref`；Seg2/3 无 `image_list`，Seg3 拉远露脸重新采样 |

---

## 会话概览

| 阶段 | 动作 | 关键参数 |
|------|------|----------|
| 定妆 | `image_generate` → `cafe_presenter_ref.png` | Seedream 2K 人像 |
| 配音 | `audio_produce` ×3 → `vo_seg1/2/3` | 同一 `voice_id: 4VZIsMPtgggwNg7OXbPY`（james_gao） |
| 视频 | `video_generate` ×3 并行 | Seg1：`image_list` + `audio_list`(vo_seg1)；Seg2/3：无 ref |
| 合成 | `execute_edit_video` → `final_assembly_v1` | Seg1 嵌轨作主 VO，Seg2/3 外挂 TTS |

---

## 根因

### P0-1 — 混合 spoken lineage（音色断裂主因）

Agent 将「主播对镜 + B-roll 自述」规划为：

- Seg1：`audio_list_tts`（TTS 经 Seedance lipsync 烘焙进 MP4）
- Seg2/3：`post_tts_vo`（成片时间轴叠 MP3）

同一 `voice_id` 经 **两条音频管道** 交付，听感必然不一致（压缩/环境声/混响/响度不同）。

### P0-2 — 同角色参考未贯穿全片（人物漂移主因）

| 片段 | `image_list` | 是否露脸 |
|------|--------------|----------|
| seq1_presenter | ✅ `cafe_presenter_ref` | 是 |
| seq2_broll_typing | ❌ 无 | 仅手部 |
| seq3_broll_phone | ❌ 无 | 结尾拉远露脸 |

三段 **并行独立** Seedance 调用，Seg2/3 仅靠英文 prompt 描述「young Asian man」→ 每张脸重新生成。

### P1 — 三次独立 TTS 调用

`vo_seg1/2/3` 分三次 `eleven_v3` 生成，段间语气/气息有随机差（次因）。

### P1 — 混音参数不一致

Seg1 嵌入对白 vol **0.9** + BGM 0.12；Seg2/3 TTS vol **1.0** + 环境 vol 0.35。

---

## Skill 层解决办法

### 新增模式：`unified_character_narration`

当 brief 含「旁白即为该角色自述」且含 on-camera + B-roll 时，**script-skill** 必须声明：

- `narrator_mode: unified_character_narration`
- 全片单一 `spoken_lineage`（`post_tts_vo` 或 `audio_list_tts`，禁止按段混搭）
- 分镜表每段必填：`character_id`、`character_ref_assets[]`、`face_visible`、`image_list_required`

### generation-skill

| 文件 | 规则 |
|------|------|
| `references/voice-strategy-execution.md` | **禁止** Seg1 `audio_list` + Seg2/3 `post_tts`；B-roll 存在时默认全片 `post_tts_vo`，Seg1 不用 `audio_list` |
| `SKILL.md` / `video-generation-execution.md` | 同 `character_id` 各段 `image_list` 必须一致；`face_visible=yes` 时 ref 非空，否则 block |
| Handoff manifest | `narrator_mode` 下 mixed lineage → 禁止交给 assembly |

### assembly-skill

| 文件 | 规则 |
|------|------|
| `references/vo-design-principles.md` §4 | `unified_character_narration` + `post_tts_vo`：全部旁白只走 TTS MP3；**禁止** Seg1 嵌轨作主 VO 而 Seg2/3 外挂 TTS |
| | 全段 VO 同 volume + LUFS；推荐单次长 VO 再切分 |
| `SKILL.md` AUDIO AUDIT | 检测双管道即 fail |

### brainstorm-skill（P2）

用户表述「同一人自述」→ 打标 `same_character_narrator: true`，下游强制 `narrator_mode`。

### 模式选择（写入 skill）

| 优先级 | 策略 | 口型 | 音色 |
|--------|------|------|------|
| **默认** | 全片 `post_tts_vo` | 略弱 | **最稳** |
| 仅当用户明确要求口型 | 全片 `audio_list_tts` | 强 | 需全段统一，仍禁止混搭 |

---

## 实施优先级

| 优先级 | Skill 文件 |
|--------|------------|
| P0 | `generation-skill/references/voice-strategy-execution.md` |
| P0 | `generation-skill` 同角色 `image_list` parity |
| P0 | `assembly-skill/references/vo-design-principles.md` |
| P1 | `script-skill/SKILL.md` |
| P2 | `brainstorm-skill/SKILL.md` |

---

## 相关文件

- QA 看板：`analysis/langfuse-data/cases/62861322045/qa-report.html`
- 素材元数据：`analysis/langfuse-data/cases/62861322045/assets-with-prompts.json`
- 媒体：`analysis/langfuse-data/cases/62861322045/media/`
