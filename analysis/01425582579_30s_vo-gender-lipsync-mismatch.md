# Case 01425582579 — 旁白女声 / 画面男声 / 无口型

| 字段 | 值 |
|------|-----|
| Conversation ID | `01425582579` |
| Trace | `12101510`（1 trace，~30min session） |
| 成片 | `final_explainer_30s.mp4`（30s，16:9） |
| 日期 | 2026-05-19 |

## 用户需求

30 秒真人实拍风解释片：开头**年轻男性**在咖啡厅**对镜头说话**；B-roll 打字、点手机；旁白为**角色第一人称自述**（非画外解说员）。

## 问题摘要

| # | 现象 | 严重度 |
|---|------|--------|
| P1 | 旁白听感像女声，画面为男性 | P1 |
| P2 | 开场人物无与旁白一致的口型 | P1 |
| P3 | 「自述」被做成画外 VO 叠在静音出镜上 | P1 |
| P4 | 首次 `audio_list` 口型片段 ASR 内容与脚本不符 | P2 |

---

## 根因链

### P1 声线不符

- TTS 使用 `voice_id`: **`BybEfHMQu0fyclQR7lfh`**
- 技能目录中 Kevin Tu（中文男声）为 **`BrbEfHMQu0fyclQR7lfh`**（`Brb` ≠ `Byb`）
- `BybEf…` **不在 Approved Voice Catalog**，违反「不得使用目录外 ID」
- 结果：音色与「年轻男性自述」不匹配

**推荐替换（本 brief）：**

| 名称 | voice_id | 说明 |
|------|----------|------|
| James Gao | `4VZIsMPtgggwNg7OXbPY` | 标准普通话、产品讲解旁白（首选） |
| Danyu Zhao | `BWN0mOtkGHghA3CYFzFK` | 教程/解释类自述 |
| Kevin Tu | `BrbEfHMQu0fyclQR7lfh` | 若需台普男声（注意 Brb 拼写） |

### P2/P3 无口型

1. `seq1_talking_head` + `audio_list`(vo_seq1_lipsync) 生成成功
2. ASR：`对对对。呃，其实我在这边也…` ≠ 脚本「以前我一天要花…」
3. Agent **未按规则重生口型片段**，改为 `seq1_presenter_silent`（`no speech`）
4. 成片：`execute_edit_video` 将 `character_vo_full` 从 0s 铺轨 → **画外旁白 + 出镜点头**

---

## 解决方案

### A. 本项目重制（立即可执行）

```
1. audio_produce × character_vo_full
   voice_id = 4VZIsMPtgggwNg7OXbPY  # James Gao, zh male
   text = （原脚本不变）

2. audio_produce × vo_seq1_opening
   text = 开场两句（约 8s）
   同一 voice_id

3. video_generate × seq1_talking_head_v2
   mode = reference2video
   image_list = character_ref
   audio_list = vo_seq1_opening.mp3
   prompt 含引号内对白 + speaking to camera
   duration 跟随 TTS 实测

4. ASR 门禁：转写须与脚本一致，否则重生（禁止 silent presenter 降级）

5. seq2/seq3 保持 B-roll + speech exclusion

6. assembly：
   - Seq1：仅用 embedded lipsync（volume 1.0），禁止再叠同文案 VO
   - Seq2+：叠 vo 续段（若需）或全片一条 master VO 从 Seq1 口型结束处接续
```

### B. Skill 规则补丁（防复发）

| 文件 | 改动 |
|------|------|
| `assembly-skill/references/vo-design-principles.md` §6 | TTS 前强制 `voice_id` 与目录表逐字比对；`Byb`/`Brb` 类 typo 视为 BLOCK |
| `assembly-skill/SKILL.md` §1 Audio Audit | 当 brief 含「自述/对镜说话」且 ASR 失败：**禁止** `presenter_silent + post VO` 作为默认降级 |
| `generation-skill/references/video-generation-execution.md` | `audio_list` 口型 prompt 必须含引号对白；ASR 不一致时最多重试 2 次再上报 |

---

## 时间线

| 阶段 | 动作 | 结果 |
|------|------|------|
| T0 | 用户 brief：年轻男性 + 自述 + 对镜说话 | — |
| T1 | TTS `BybEf…` + character_ref（男） | 声画性别隐患 |
| T2 | seq1_talking_head + audio_list | ASR 内容错误 |
| T3 | seq1_presenter_silent | 无口型备选 |
| T4 | assembly + VO 全片铺轨 | 成片：男画女音、无对口型 |

---

## 相关文件

- Trace: `analysis/langfuse-data/cases/01425582579/trace-1-12101510.json`
- QA 看板: `analysis/langfuse-data/cases/01425582579/qa-report.html`
- 媒体: `analysis/langfuse-data/cases/01425582579/media/`
