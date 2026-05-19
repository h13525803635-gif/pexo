# Case 90921184221 — 音频重叠（画内音 + 后期旁白）

| 字段 | 值 |
|------|-----|
| 项目 ID | `90921184221` |
| Langfuse trace | `a284c3a58f554ef70fc95bec3d6ebeac`（本地 `trace-1-a284c3a5.json`，约 25MB，未入库） |
| 用户 brief | 30s 16:9 真人实拍记账 App；家居书桌自述 + B-roll；旁白=角色自述 |
| 成片 | `final_accounting_app_ad.mp4`（约 30.2s） |
| 分析日期 | 2026-05-19 |
| 关联 case | 73077293918、66831923061 |

---

## 1. 问题现象

| # | 现象 | 时间轴（约） |
|---|------|----------------|
| P0-1 | **Seq1 画内 co-gen 乱码中文 + TTS 旁白双轨** | 0.2s–9.6s |
| P0-2 | 多轨混音发糊 / 双声感 | 全片，开场尤甚 |
| P1 | Seq4 旁白与环境内嵌音叠层 | 22.5s–26.3s（无人声冲突，但有混音层叠） |

---

## 2. Seq1 重叠（核心）

### 2.1 定位

Agent 规划：Seq1/Seq4 使用 **后期 TTS**（`post_tts_vo`），生成时排除 speech，只保留环境音。

实际 `seq1_main_open` STT（`asr_seq1`）：

| 字段 | 值 |
|------|-----|
| `language_code` | `zho` |
| `language_probability` | **0.9997** |
| `text` | 「我在星火多元的日子的和汪两情……」（乱码，与剧本不符） |
| 人声时段 | 约 **0.62s–9s+** |

### 2.2 生成阶段矛盾

Prompt 同时包含：

- `No speech, no dialogue, no voice, no spoken words`
- **`lips moving naturally as if speaking`**
- `sound: "on"`

Seedance 仍 co-gen 整段中文人声（嘴型 + 声源同时出现）。

### 2.3 合成叠音机制

`final_accounting_app_ad` 在 Seq1 时段同时播放：

| 轨 | 内容 | Seq1 行为 |
|----|------|-----------|
| 2 | `seq1_main_open` 内嵌音（含 co-gen 乱码人声） | `volume: 0.1` |
| 3 | `vo_part_a` 中文旁白（正确剧本） | `volume: 1.0`，`start_ms: 200`，长 9360ms |
| 4 | `bgm_guitar` | `volume: 0.1` 全片 |

**听感** = 10% 乱码画内音 + 100% TTS 旁白 + BGM。

Agent 备注承认「P1 VO vs P2 dialogue 冲突」，但选择将内嵌轨降至 0.1 而非 **mute（0）**，导致重叠仍可闻。

---

## 3. 时间轴与合成 DSL（证据）

```
总时长: 10.042 + 5.062 + 7.059 + 8.057 ≈ 30.22s

Seq1  0      – 10042  内嵌乱码中文 @0.1 + vo_part_a @1.0  ← 核心重叠区
Seq2  10042  – 15104  环境音 @0.5（无旁白）
Seq3  15104  – 22163  环境音 @0.5（无旁白）
Seq4  22163  – 30220  环境音 @0.45 + vo_part_b @1.0（22500–26340ms）
```

合成 `execute_edit_video` 名称：`final_accounting_app_ad`，4 轨（1 video + 3 audio）。

**vo_part_a**：`start_ms: 200`，`out_ms: 9360` → 结束约 **9560ms**  
**vo_part_b**：`start_ms: 22500`，`out_ms: 3840` → 结束约 **26340ms**

---

## 4. Skill 归因

| 规则来源 | 应触发 | 本 case |
|----------|--------|---------|
| script-skill：主镜头 + post TTS | 禁止 co-gen speech | 失败，Seq1 出乱码中文 |
| video-generation-execution：Voice Source Exclusivity | 有后期 VO 时禁止画内 speech | 违反 |
| assembly-skill：AUDIO AUDIT | STT 检出 speech → mute/regen | 识别到但只降音量 |
| assembly-skill：anti-doubling | 有 TTS 时画内 speech 轨 volume=0 | Seq1 用 0.1 |

---

## 5. 修复建议

1. **Seq1 重生成**：去掉「嘴在说话」类描述；或 `sound: off` / `audio_list` 烘焙 TTS。  
2. **合成**：含 speech 的画内轨 **`volume: 0`**，不要 0.1。  
3. **审计门禁**：`language_probability > 0.8` 且 word 级 transcript → 禁止叠同语种 VO。  
4. **B-roll**：对 Seq2/Seq3 补 STT，防止漏检 co-gen 人声。

---

## 6. 仓库内文件索引

| 路径 | 说明 |
|------|------|
| [本报告](./90921184221_audio-overlap-root-cause.md) | 根因分析（当前文件） |
| [cases/90921184221/](./langfuse-data/cases/90921184221/) | 素材、url_map、媒体 |
| Langfuse | trace `a284c3a5`，session `thread_70491211668_90921184221_*` |

---

## 7. 关键证据摘录

**Seq1 STT（应判 speech，实际仅降音量）：**

```json
{
  "language_code": "zho",
  "language_probability": 0.9997097849845886,
  "text": "我在星火多元的日子的和汪两情，没有陷到楼毅的深巨弱，可在我想要这场过辞节，就更紧嘛。"
}
```

**合成轨 2/3（Seq1 内嵌 0.1 + 中文 VO @1.0）：**

- 轨 2：`seq1_main_open.mp4`，`start_ms: 0`，`volume: 0.1`  
- 轨 3：`vo_part_a.mp3`，`start_ms: 200`，`volume: 1.0`
