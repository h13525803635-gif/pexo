# Case 90504488194：旧书店自述片 — 双轨人声 + 交付文案编造「好久不见」

**Langfuse**：`pexo:90504488194`（1 trace：`trace-1-29c2edd1.json`）  
**用户 brief**：30s 16:9 真人实拍；风衣女性**书店门口对镜笑着说话** → B-roll 书脊/抱书出巷；**旁白为角色自述**  
**成片**：`bookstore_short_film_30s.mp4`  
**Trace 数据**：`analysis/langfuse-data/cases/90504488194/`

---

## 现象

| 用户反馈 | 技术表现 |
|----------|----------|
| 旁白与口述音色不一致 | 0–10s Seq-A **Seedance 协生对白** + 10s 起 **ElevenLabs TTS**（`5qr5FEpvZGzmVOPBS55W`），双 spoken 轨 |
| 开头应按规划说「好久不见」 | 交付文案声称已说；**生成参数中从未写入该句** |

---

## 会话概览

| 阶段 | 动作 | 关键参数 |
|------|------|----------|
| 定妆 | `image_generate` ×2 | `char_ref_trenchcoat`、`scene_ref_bookstore` |
| 视频 | `video_generate` ×3 并行 | Seq-A：`sound:on`，嘴动但无具体中文台词；B/C：`No speech` |
| 配音 | `audio_produce` → `vo_narration` | 从「我也说不清楚…」起，**无「好久不见」** |
| 合成 | `execute_edit_video` | Seq-A 内嵌音 volume=1.0；VO 从 10054ms 叠入 |

---

## 根因

### P0-1 — 角色自述片误用双 spoken 来源

Brief 要求「旁白 = 角色自述」。Agent 规划为：

- Seq-A：`on_camera_sync_speech`（co-gen）
- Seq-B/C：`post_tts_vo`（`zi_yue` / ElevenLabs）

合成保留开场 co-gen + 后段 TTS → 听感为两个说话人。违反 **voice source exclusivity**（同 46021149558、62861322045）。

### P0-2 — 「好久不见」仅存在于交付文案（confabulation）

`show_final_video` 后发给用户的结构回顾写明：

> 0–10s … 对着镜头笑着说了句「好久不见」

但执行阶段：

| 产物 | 是否含「好久不见」 |
|------|-------------------|
| 内部制作规划 | ❌ |
| Seq-A `video_generate` prompt | ❌（仅 `as if warmly saying something`） |
| `audio_produce` text | ❌ |

Agent **事后描述**与**生成参数**脱节，用户按 UI 规划验收会判定失败。

### P1 — Seq-A 未锁定台词与口型

对镜说话应使用 `audio_list_tts`（先 TTS「好久不见」，再 `lip sync to audio 1`），或全片统一 post-TTS 并 mute Seq-A 对白。均未执行。

---

## 解决办法

### 对本项目重制

1. 统一 VO 稿：`好久不见。我也说不清楚，可能就是那种味道……`（同一 `voice_id`）
2. **方案 A**：Seq-A 传 `audio_list` + 引号内「好久不见」，口型同步；B/C 无 speech；合成不叠第二条 TTS
3. **方案 B**：全片 post-TTS 从 0s 起播；Seq-A 重生成并 `No speech`，mute 内嵌对白

### Skill 层

| 优先级 | 改动 |
|--------|------|
| P0 | brief 含「旁白即为角色自述」→ `unified_character_narration`，禁止 co-gen + post-TTS 同片 |
| P0 | 交付前 **spoken-line audit**：用户可见台词须在 prompt/TTS 中可检索 |
| P1 | 对镜台词必填 `spoken_text` 或 `audio_list`，禁止仅写「嘴在动」 |

---

## 相关文件

- 分析正文：本文件
- Trace：`analysis/langfuse-data/cases/90504488194/trace-1-29c2edd1.json`
- 媒体：`analysis/langfuse-data/cases/90504488194/media/`
