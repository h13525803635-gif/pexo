# Case 66831923061 — 韩语对白 / 口型失配 / 人物漂移：根因与 Skill 修改方案

| 字段 | 值 |
|------|-----|
| 项目 ID | `66831923061` |
| Langfuse trace | `71ed890301eae31d6fafd16cdde1fc4d`（`trace-1-71ed8903.json`） |
| 用户 brief | 30s 16:9 真人实拍；公园长椅年轻男性**对镜说话**；B-roll 手表/手机；**旁白=角色自述**（普通话） |
| 成片 | `fitness_app_final.mp4` |
| 分析日期 | 2026-05-19 |
| 关联 case | 46021149558、62861322045、01425582579 |

---

## 1. 问题现象

| # | 用户反馈 | 技术表现 |
|---|----------|----------|
| P0-1 | 主角讲韩语 | TTS 为中文（`language_code: zh`）；韩语来自 Seq-1 Seedance **co-gen 嵌入对白** |
| P0-2 | 有人物时口型对不上旁白 | 画面按「说话」生成（嘴在动），声音为 co-gen 乱语/韩语 + 后期中文 TTS 叠轨 |
| P0-3 | 前后人物不一致 | 跳过 character_ref；三段 `video_generate` **并行纯文生**，无 `image_list` 锚定 |

---

## 2. 根因链

### 2.1 音频：规划与执行互斥（P0-1 / P0-2）

**用户需求**：旁白 = 角色自述 + 开场对镜说话 → 需要**单一 spoken 来源**且口型与中文一致。

**Agent 生产规划（内部）同时写了两套方案**：

| 来源 | 写法 |
|------|------|
| 序列表 | Seq-1 = `on_camera_sync_speech`（角色说出旁白） |
| 语音决策 | 采用 `post_tts_vo`：Seq-1 加 `no speech`，旁白全靠 TTS |

**实际 `video_generate`（seq1_park_presenter）**：

- `sound: "on"`
- Prompt 前半：`He speaks naturally... as he talks... as he speaks`
- Prompt 末尾：`No speech, no dialogue, no voice, no spoken words`
- **未传 `audio_list`**

→ Seedance 在「要说话」与「禁止说话」矛盾下 **co-gen 非中文对白**（常见为韩语/乱语），同时嘴型在动。

**TTS（正确的中文）**：

```text
audio_produce → vo_fitness_app
voice_id: 4VZIsMPtgggwNg7OXbPY (james_gao)
language_code: zh
text: 以前训练完，数据堆在手机里……
```

**合成阶段错误假设**：

> 「三段视频嵌入音频均为 SFX/环境音（非对话），无口型同步语音 → 安全叠加 VO」

实际 Seq-1 很可能含 co-gen 对白。混音将嵌入音降至 **0.25**，中文 TTS **1.0** 叠在上面 → **嘴型对韩语、耳朵听中文、双轨并存**。

这违反线上已有规则（但未被执行）：

- **Voice Source Exclusivity**：每句 spoken 只能有一个来源
- **Anti-Doubling**：嵌入 speech 存在时禁止再叠同文案 TTS
- **Character-sync contradiction check**：用户要求对镜/自述时，不得用 `post_tts_vo` + `no speech` 同时又 `speaks`

### 2.2 口型：应用 `audio_list_tts` 却未执行（P0-2）

**正确做法（用户理解正确）**：当画内人物需对上固定旁白时：

1. 先 `audio_produce` 生成中文 VO  
2. 再 `video_generate` 传入 `audio_list` + `image_list`（character_ref）  
3. Prompt：`lip sync to audio 1` + 引号内中文台词  
4. 合成：**使用嵌入对白**，不再叠同文案 TTS  

线上 API：`audio_list` 为参考音频（每段 2–15s，合计 ≤15s），须配合 image/video，不能单独使用。

66831923061 **未走此路径**，反而走「co-gen 瞎说 + post TTS」最差组合。

### 2.3 人物：跳过定妆、并行文生（P0-3）

| 项 | 事实 |
|----|------|
| Todo | 含「准备视觉参考素材」→ **未完成** |
| 规划 | 「无用户参考素材，纯文字生成」 |
| 执行 | 3 段并行 `reference2video`，均无 `image_list` |
| 结果 | Seq-1 / Seq-3 各生成一名「young East Asian man」→ 脸/发型/衣着漂移 |

会话摘要曾写下一步应 `image_generate` character_ref，实际被跳过。

---

## 3. 正确技术路径（验收用）

### 方案 A — 强口型（推荐，符合「对镜自述」）

```
character_ref (image_generate)
    ↓
VO 中文 TTS (≤15s/段，按句切分)
    ↓
video_generate(
  image_list = [character_ref],
  audio_list = [vo_chunk.mp3],
  sound: on,
  prompt: 对镜 + 引号中文台词 + sync to audio 1
  # 禁止 speaks 与 no speech 并存
)
    ↓
assembly: 嵌入音为主，禁止再叠同文案 TTS
```

### 方案 B — 弱口型（音色最稳）

```
character_ref 仍必填
video_generate: 无 speaks；sound: off；仅环境音
单次 master VO → 合成叠轨；Seq1 嵌入对白 mute
```

**本案不应**：A 的画面 + B 的音频（当前错误）。

---

## 4. Skill 修改方案（待落地）

> 说明：线上 skill **已有** Voice Source Exclusivity、`audio_list`、Character-sync 等条文，66831923061 **违反的是执行**。本方案以 **补门禁、消歧、强制 subject-asset** 为主。

### 4.1 问题 → 改动映射

| ID | 根因 | 严重度 | Skill 改动方向 |
|----|------|--------|----------------|
| P0-1 | co-gen + TTS 双轨 | P0 | Preflight 双 spoken 禁止；assembly ASR 门禁 |
| P0-2 | 未用 audio_list_tts | P0 | Unified Character Narration Playbook |
| P0-3 | 无 character_ref 并行文生 | P0 | subject-asset Character Anchor Gate |
| P1-1 | 旁白=自述 走错 VO 路径 | P1 | brainstorm 打标 + script narrator_mode |
| P1-2 | 合成误判嵌入音为环境音 | P1 | assembly 禁止无 ASR 叠 TTS |

### 4.2 新增模式：`unified_character_narration`

**触发词**：`旁白即为`、`自述`、`第一人称`、`对着镜头说话`、`口播`、`口型`

**script-skill 必须先选 `spoken_lineage`（全片二选一，禁止按段混搭）**：

| spoken_lineage | 口型 | 适用 |
|----------------|------|------|
| `post_tts_vo` | 弱 | 用户未强调口型；画内静默 |
| `audio_list_tts` | 强 | 用户要求对镜口述/口型 |

**66831923061 应选**：`audio_list_tts`（非 `post_tts_vo`）。

### 4.3 分文件修改清单

#### P0 — `brainstorm-skill/SKILL.md`（**新增**）

- Handoff 字段：`same_character_narrator: true`、`narrator_mode: unified_character_narration`
- 禁止仅写 off-screen `voiceover_narration` 而不区分画内自述

#### P0 — `script-skill/SKILL.md`（**新增 + 改表**）

**分镜表必填列**：`narrator_mode`、`spoken_lineage`、`character_id`、`character_gender`、`voice_id`、`face_visible`、`planned_source`、`character_ref_assets[]`、`image_list_required`、`audio_list_asset`

**BLOCK**：

- 同 `character_id` 禁止 Seg-A `audio_list_tts` + Seg-B `post_tts_vo`
- `post_tts_vo` 禁止填 `on_camera_sync_speech`
- `face_visible=yes` 且 `audio_list_tts` 必须有 `audio_list_asset`

**改 `references/video-generation-execution.md`**：co-gen-first 增加 unified_character_narration 例外框

#### P0 — `subject-asset-skill/SKILL.md`（**新增**）

**Character Anchor Gate (BLOCK)**：

- `same_character_narrator` 或任一段 `face_visible=yes` → 必须先产出 `character_ref`
- 禁止「无素材 → 跳过定妆直接并行文生」

#### P0 — `generation-skill/SKILL.md`（**修改**）

**每次 `video_generate` 前 Preflight（一条不过即 STOP）**：

| 检查 | 规则 |
|------|------|
| A. Lineage | `planned_source` 与 handoff `spoken_lineage` 一致 |
| B. audio_list | `audio_list_tts` → `audio_list` 非空，VO ≤15s |
| C. post_tts 静默 | `post_tts_vo` + 露脸 → 无 speak/talk，`sound: off` |
| D. Sound-on 矛盾（**新**） | `sound:on` 与 `no speech` 或 `speaks` 互斥 → BLOCK |
| E. 双 spoken | 有 `audio_list` 则 assembly 禁止再叠同文案 TTS |
| F. image_list | `image_list_required=yes` → 含 character_ref |
| G. 并行 | 同 character 无 ref → 禁止并行 video_generate |

#### P0 — `generation-skill/references/voice-strategy-execution.md`（**新增专章**）

**Unified Character Narration — Execution Playbook**（audio_list_tts / post_tts_vo 标准工序）

**禁止降级**：`audio_list` ASR 失败后禁止默认 `presenter_silent` + post VO（除非用户放弃口型）

#### P0 — `generation-skill/references/video-generation-execution.md`（**修改**）

- Prompt 互斥：禁止 `speaks` + `No speech` 同条调用
- `audio_list_tts` 必须传 `audio_list`，禁止依赖自由 co-gen 对白

#### P0 — `assembly-skill/SKILL.md`（**修改**）

- 合成前 ASR：有嵌入 speech 且 handoff=`post_tts_vo` → BLOCK 叠 TTS
- 禁止未 ASR 即写「嵌入音仅为 SFX」
- `audio_list_tts` 段：以嵌入轨为主 VO

#### P1 — `assembly-skill/references/vo-design-principles.md`（**修改**）

- §0：`unified_character_narration` 时画内走 dialogue/audio_list，不以 off-screen VO 默认覆盖
- §6：`voice_id` 与目录逐字比对（ typo BLOCK）

#### P2 — `modification-skill/SKILL.md`（**修改**）

- 口型/外语反馈 → 优先重生 talking-head（audio_list_tts），禁止只调 volume

### 4.4 实施优先级

| 优先级 | 文件 |
|--------|------|
| P0 | generation-skill（SKILL + voice-strategy + video-generation-execution） |
| P0 | script-skill、assembly-skill |
| P1 | subject-asset-skill、brainstorm-skill、vo-design-principles |
| P2 | modification-skill |

若修改 `video-models-scheduling.md`，需同步 generation / creative / modification 三份副本。

---

## 5. 本案重制检查表

- [ ] `narrator_mode = unified_character_narration`
- [ ] `spoken_lineage = audio_list_tts`（若用户要口型）
- [ ] `image_generate` → `character_ref`
- [ ] TTS 男声 + `language_code: zh`，与角色 gender 一致
- [ ] Seq-1：`audio_list` + `image_list`，prompt 引号中文，无 `no speech` + `speaks` 并存
- [ ] Seq-2/3：同 `character_ref`；B-roll 旁白用同 master VO 策略（禁止 Seg1 烘焙 + Seg2/3 外挂混搭音色）
- [ ] 合成：Seq-1 不叠重复 TTS；ASR/语言检查通过后再混 BGM

---

## 6. 数据与媒体路径（本仓库）

| 资源 | 路径 |
|------|------|
| Trace JSON | `analysis/langfuse-data/cases/66831923061/trace-1-71ed8903.json` |
| 素材元数据 | `analysis/langfuse-data/cases/66831923061/assets.json` |
| 本地媒体 | `analysis/langfuse-data/cases/66831923061/media/` |

---

## 7. 修订记录

| 日期 | 说明 |
|------|------|
| 2026-05-19 | 初版：根因分析 + Skill 修改方案（case-analyzer Phase B） |
