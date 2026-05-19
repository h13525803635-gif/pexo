# Case 73077293918 — 音频重叠（含 L2 英文叠音）根因分析

| 字段 | 值 |
|------|-----|
| 项目 ID | `73077293918` |
| Langfuse trace | `32880e9f60e59c043b4eb391a12db599`（本地 `trace-1-32880e9f.json`，约 45MB，未入库） |
| 用户 brief | 30s 16:9 真人实拍健身 App；公园自述 + B-roll；**旁白=角色自述** |
| 成片 | `fitness_app_final.mp4`（27.17s） |
| 分析日期 | 2026-05-19 |
| 关联 case | 35802589870、66831923061 |

---

## 1. 问题现象

| # | 现象 | 时间轴（约） |
|---|------|----------------|
| P0-1 | **L2 段中英文旁白重叠** | 14.1s–19.2s（S3 手机 B-roll） |
| P0-2 | 多轨混音发糊 / 双声感 | 全片，开场与 B-roll 尤甚 |
| P1 | L2/L3 切点双旁白 | 19.18s–19.26s（约 76ms） |
| P1 | S1 内嵌旁白含剧本外台词 | 0–9s |
| P2 | S4 结语口型与中文不同步 | 19.2s–27.2s（重拍 v2 后） |

---

## 2. L2 英文重叠（核心）

### 2.1 定位

L2 文案：「后来用了这款 App……我终于能看懂自己的数据了。」对应 **S2（手表）+ S3（手机）** 两段 B-roll。

| 片段 | STT 结果 | `language_probability` | 结论 |
|------|----------|------------------------|------|
| S2 `s2_broll_watch` | `[birds chirping]` | 0.56（环境事件） | 无人声，正常 |
| S3 `s3_broll_phone` | `So this is Altman who be a thewhode...` | **0.966** | **整段英文 co-gen 人声** |

S3 英文覆盖片内 **0.16s–4.62s**（几乎满 5s）。

### 2.2 Agent 审计误判（直接放行）

Trace 内审计结论：

> S3 ✅ — 有轻微噪声被 STT 误识别为乱码英文，但实际上是环境音（very low language probability）

与 STT JSON **矛盾**（英文置信度 96.6%，且为连续 `type: word`）。应判为 **speech-bearing segment**，需重生成或合成静音。

### 2.3 成片叠音机制

`fitness_app_final` 在 L2 时段同时播放：

| 轨 | 内容 | L2 时段行为 |
|----|------|-------------|
| 3 | `vo_seg2_broll.mp3` 中文 L2 | `volume: 1.0`，9056–19256ms |
| 2 | S2/S3 视频内嵌音 | S3 段 `volume: 0.35`，**含英文人声** |
| 4 | BGM | `volume: 0.12` 全片 |

**S3 段听感** = 中文 L2（100%）+ 英文 co-gen（约 35%）+ BGM。

### 2.4 为何 S3 生出英文

Prompt 虽含 `No speech, no dialogue, no voice, no spoken words`，但同时：

- `sound: "on"`
- 强语义：`smartphone` / `fitness tracking app` / `app interface` / `scrolling`

Seedance 对「操作 App 界面」镜头易默认 **英文解说式口播**（与 S4 首版 `"So listen to your body..."` 同类）。

---

## 3. 其他音频重叠

### 3.1 S4 首版：英文内嵌 + 计划叠中文

- `s4_close_talk` STT：`So listen to your body and keep moving.`
- `audio_list` 注入失败 → 模型自生成英文
- 若叠 `vo_seg4_close` → 中英双轨（后重拍 `s4_close_v2` 缓解，但失去唇同步）

### 3.2 S1：TTS 烘焙 + 模型加戏

STT 检出 L1 外额外句：「喝多了酒第二天也照跑不误」——剧本无此句 → `audio_list_tts` 未完全独占声源。

### 3.3 L2/L3 切点 76ms 重叠

- `vo_seg2_broll`：9056 + 10200 = **19256ms** 结束  
- `vo_seg4_close`：**19180ms** 开始  
- **19180–19256ms**：L2 与 L3 同时播放

### 3.4 策略层：同旁白双路径

- S1/S4：`audio_list_tts` 烘焙进视频  
- S2/S3：后期 `vo_seg2_broll`  
- 另生成 `vo_full_narration` 再切片 → 边界与审计复杂度上升

---

## 4. 时间轴与合成 DSL（证据）

```
总时长: 9.056 + 5.062 + 5.062 + 8.057 = 27.237s

S1  0      – 9056   内嵌中文 L1 (audio_list_tts)
S2  9056   – 14118  环境音 + 中文 L2 (vo_seg2)
S3  14118  – 19180  环境音+英文co-gen + 中文 L2  ← L2英文重叠区
S4  19180  – 27237  环境音 + 中文 L3 (vo_seg4) + BGM
```

合成 `execute_edit_video` 名称：`fitness_app_final`，5 轨（1 video + 4 audio）。

---

## 5. Skill 归因

| 规则来源 | 应触发 | 本 case |
|----------|--------|---------|
| script-skill：B-roll + `no speech` | S3 禁止人声 | 失败，出英文 |
| video-generation-execution：Voice Source Exclusivity | 有 `audio_list` 时禁止 co-gen speech | S1/S4 部分违反 |
| assembly-skill：AUDIO AUDIT | STT 检出 speech → mute/regen | **S3 误判放行** |
| assembly-skill：Speech overlap rule | speech 片段不得再叠同语种 VO @0.35 | S3 违反 |
| assembly-skill：anti-doubling | `audio_list_tts` 成功不再叠 VO | S1 遵守；S3 未识别 |

---

## 6. 修复建议

1. **S3**：STT `language_probability > 0.8` 且存在 word 级 transcript → 强制重生成或合成 `volume: 0`。  
2. **L2**：仅保留一条中文 VO；B-roll 内嵌轨若含 speech 则静音，仅保留极低环境音（或 `sound: off` 重生成）。  
3. **切点**：`vo_seg4.start_ms >= vo_seg2.start_ms + vo_seg2.duration`。  
4. **审计门禁**：禁止将高置信英文 transcript 标注为「环境音误识别」。

---

## 7. 仓库内文件索引

| 路径 | 说明 |
|------|------|
| [本报告](./73077293918_audio-overlap-root-cause.md) | 根因分析（当前文件） |
| [qa-report.html](./langfuse-data/cases/73077293918/qa-report.html) | 可视化 QA 看板 |
| [cases/73077293918/](./langfuse-data/cases/73077293918/) | 素材、url_map、媒体 |
| Langfuse | trace `32880e9f`，session `thread_70491211668_73077293918_*` |

---

## 8. 关键证据摘录

**S3 STT（应判 speech，实际判正常）：**

```json
{
  "language_code": "eng",
  "language_probability": 0.9656668305397034,
  "text": "So this is Altman who be a thewhode and the thirfood is a tatnon your fuvs"
}
```

**合成轨 2/3（S3 内嵌 0.35 + 中文 L2 @1.0）：**

- 轨 2：`s3_broll_phone.mp4`，`start_ms: 14118`，`volume: 0.35`  
- 轨 3：`vo_seg2_broll.mp3`，`start_ms: 9056`，`volume: 1.0`
