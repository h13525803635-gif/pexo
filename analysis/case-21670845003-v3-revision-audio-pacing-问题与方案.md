# 项目 21670845003：V3 成片 Revision made it worse / Audio issue / Bad pacing — 原因与解决办法

**主成片（V2，QA 对照）**：`silent_hearts_full_v2.mp4`（约 195s，720×1280）  
**V3 状态**：已启动 `v3_seg1–10` 无声重生，**未交付** `silent_hearts_full_v3.mp4`（trace-37 用户积分耗尽）  
**Langfuse 会话**：38 traces（trace-1 ~ trace-38，2026-05-19）  
**分析日期**：2026-05-19  
**QA 看板**：`analysis/langfuse-data/cases/21670845003/qa-report.html`

---

## 一、用户需求（验收标准）

| 阶段 | 用户原意（摘自 trace input） |
|------|------------------------------|
| 启动 | 电影感棚拍歌手表演，Silent Hearts 美学，角色参考图 |
| Lipsync | 「已有音乐，要与嘴同步」→ 上传 `Waiting_On_To_Stay_ok.mp3` |
| 预览 | 「先看前几段 validate」 |
| 音频 QA | 批注：**AUDIO DUPLICADO. EXCLUIR O AUDIO ADIANTADO.**（重复音频，去掉超前音轨） |
| 节奏 QA | **视频比音乐慢，口型不同步** |
| 策略 | 选用 **co-generated 音频**（口型准），原曲暂不作 BGM |
| 全片 | 「可以 montar」 |
| 修订 V2 | **像很多碎片拼起来**；3:20 出现错误 **let's go**，要求删掉 |
| 歌词 | 粘贴完整英文歌词，要求 **整首同步到人物声音** |
| 修订 V3 | **去掉音乐，做无音频视频，只要歌手按歌词做表情**（口型/表演） |
| 收尾 | 抱怨生成太多、**积分耗尽** |

隐含验收：**一条连贯 MV、口型与歌词/音乐一致、无重复镜头、无叠音、修订后变好而非变差**。

---

## 二、实际交付与版本演进

### 2.1 成片文件对照

| 版本 | 文件 | 说明 |
|------|------|------|
| 初版 | `silent_hearts_performance.mp4` | 3 段表演 + BGM，非全曲 lipsync |
| V1 | `silent_hearts_full_clip.mp4` | 16 段 lipsync 硬拼（含 seg15 用 3 次等） |
| **V2** | `silent_hearts_full_v2.mp4` | 删 seg15 + 12× crossfade 800ms；**仍用重复段填洞** |
| V2.5 | `v2_lip_seg1–10` + `silent_hearts_vocals` | **已生成，从未写入最终时间线** |
| V3 | `v3_seg1–10` | 无声重生进行中，**无 full_v3 交付** |

### 2.2 V2 时间线（trace-19 `execute_edit_video` 实测）

13 个 clip，每段 **固定 15s**（`out_ms: 15000`），**无独立 `audio_tracks`**，仅靠各段内嵌 co-generated 音频拼接：

| # | 素材 | 说明 |
|---|------|------|
| 1–3 | seg1, seg2, seg3 | 正常 |
| 4–5 | **seg5, seg5** | 重复（缺 seg4） |
| 6–7 | **seg7, seg7** | 重复（缺 seg6、seg8） |
| 8 | seg9 | 正常 |
| 9–10 | **seg10, seg10** | 重复（缺 seg11） |
| 11–13 | seg12, seg13, seg14 | 正常；已删错误歌词的 seg15 |

Agent 在 trace-17 曾明确计划：缺段时用相邻段 **复制顶替**（seg4→5、seg6/8→7、seg11→10 等），导致 QA **Bad pacing** 与 **Revision made it worse**。

---

## 三、问题清单（现象 → 根因 → 解决办法）

### P0-1：Revision made it worse（修订越改越差）

| 项 | 内容 |
|----|------|
| **现象** | V2 仍碎片化、重复镜头；V3 烧积分全量重生且无新成片；TTS/新 lipsync 未生效 |
| **根因** | 修订只删 seg15、加 crossfade，**未补全缺失段、未去重**；V3 把「去音乐」理解成 **10 段 video_generate 重生** 而非后期 mute |
| **重做** | ① 仅生成缺失的 seg4/6/8/11；② 新时间线 **零 duplicate**；③ TTS 路线须 **新 execute_edit** 交付，禁止继续展示旧 full_v2 |
| **Skill** | `assembly-skill`：禁止用同一 clip 文件填洞；`modification-skill`：「去音乐」= mute/strip，默认不 bulk regen |

### P0-2：Audio issue（音频问题）

| 项 | 内容 |
|----|------|
| **现象** | 重复音频；画面慢于音乐；co-gen 歌曲≠用户上传曲；V3 走向完全无声 |
| **根因** | 段内 Seedance 音频 + 用户 MP3 **双轨叠加**（trace-9）；固定 15s 未对齐节拍（trace-11）；选 co-gen 后 **无用户 master 轨**；V2 edit spec **无 audio_tracks** |
| **重做** | `sound=off` 生成；`audio_tracks[0]` = 用户 MP3 或校对后 TTS；段内 embedded 音频 **全部 mute** |
| **Skill** | `assembly-skill`：合成前 probe 每段音频，强制单 master；`generation-skill`：lipsync 默认 `sound=off` |

### P0-3：Bad pacing（节奏差）

| 项 | 内容 |
|----|------|
| **现象** | 约 195s 成片 vs ~4min 原曲；同一段画面播两遍；切换生硬 |
| **根因** | **15s 均匀切格**；缺 5 段 + **重复 5/7/10**；11/16 段未齐仍先 montar；crossfade 无法弥补重复镜头 |
| **重做** | 按 **歌词时间轴** 切 in/out；缺段则缩短或占位，**禁止 neighbor duplicate**；支持 per-clip `speed` 对齐音乐 |
| **Skill** | `script-skill` / `creative-skill`：先有 segment map 再批量生成；`assembly-skill`：duplicate 校验器 |

### P1：积分与迭代失控

| 项 | 内容 |
|----|------|
| **现象** | trace-15 单次 trace 内 28 次 `video_generate`；V2/V3 多轮重生后积分耗尽 |
| **根因** | 未「预览锁定方案」即批量生成；修订未最小 diff |
| **解决办法** | 长 MV：45s 预览 + 锁定 segment 表后再全量生成；修订优先 edit，后补生成 |

---

## 四、失败因果链

```
用户上传原曲 → 要求 lipsync
  → 段内 co-gen 音 + MP3 叠加（重复音频）
  → 固定 15s + 视频慢于音乐
  → 改 co-gen 路 + 批量生成，部分段失败
  → 用 seg5/7/10 复制填洞 montar（缺 4/6/8/11）
  → V2：删 seg15 + crossfade，仍有重复段
  → 用户给歌词 → TTS + v2_lip 生成但未组装
  → V3：误解「无音频」→ v3_seg 全量重生
  → 积分耗尽，无 full_v3
  → QA：Revision worse + Audio + Bad pacing
```

---

## 五、推荐重做产线（正确版）

```
1. 从用户 MP3 分析歌词/节拍 → segment map（起止 ms，禁止 uniform 15s）
2. 盘点已有 lipsync 段：仅补生成缺失 index（4,6,8,11…），sound=off
3. execute_edit_video：
   - video：按 map 顺序，无 duplicate source.file
   - audio_tracks[0]：用户 MP3 或 TTS 整轨
   - 各 clip：mute 段内 embedded 音频
4. 若用户要「只要表情、不要音乐」：在同一时间线上 export mute 版，不 regen v3_seg
5. show_final_video → 用户确认后再迭代
```

### edit_spec 要点（方向）

```json
{
  "video_clips": "按歌词时间轴，无重复 file",
  "audio_tracks": [{ "source": "Waiting_On_To_Stay_ok.mp3", "volume": 1.0 }],
  "clip_audio": "muted",
  "forbid": ["duplicate_neighbor_fill", "stack_co_gen_with_master"]
}
```

---

## 六、Skill 改进建议（防复发）

| 规则 | 写入位置 | 要点 |
|------|----------|------|
| 禁止 duplicate 填洞 | assembly-skill | 缺段 → 生成或缩短，不得复制相邻 clip |
| 单 master 音轨 | assembly-skill | lipsync 必先 probe，再 mute 段内音 + 铺 master |
| sound=off 默认 | generation-skill | lipsync 段不内嵌 BGM/人声 |
| 歌词时间轴切分 | script-skill | 禁止 15s 网格硬切 |
| 修订最小 diff | modification-skill | 去音乐 = mute；须新 edit 才展示新版本 |
| 长 MV 预览门 | generation-skill | 45s 锁定后再批量，避免 trace-15 式爆量生成 |

---

## 七、数据文件索引

| 文件 | 说明 |
|------|------|
| `analysis/langfuse-data/cases/21670845003/trace-19-9b47d519.json` | V2 组装与「去 seg15」主 trace |
| `analysis/langfuse-data/cases/21670845003/trace-25-31b8c329.json` | V3「无音频只要表情」用户输入 |
| `analysis/langfuse-data/cases/21670845003/trace-21-c443286d.json` | TTS + v2_lip 生成（未进成片） |
| `analysis/langfuse-data/cases/21670845003/trace-9-7467d248.json` | 重复音频用户批注 |
| `analysis/langfuse-data/cases/21670845003/assets.json` | 素材元数据 |
| `analysis/langfuse-data/cases/21670845003/qa-report.html` | 可视化 QA 看板 |
| `analysis/langfuse-data/cases/21670845003/media/` | 已下载媒体样本 |
| `analysis/langfuse-data/cases/21670845003/trace-*.json` | 全量 38 条 Langfuse trace |
