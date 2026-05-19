# 项目 45125516390：30s 山口彊短片 — 不符合用户要求全链路分析

**Langfuse trace**：`analysis/langfuse-data/cases/45125516390/trace-1-b1598742.json`（latency ≈ 479s，单次交付）  
**项目名称**：The Man Who Survived Two Atomic Bombs  
**分析日期**：2026-05-19  
**QA 看板**：`analysis/langfuse-data/cases/45125516390/qa-report.html`

---

## 一、用户需求（验收标准）

用户消息（原文要点）：

> Create a cinematic **30s** short with **movie-quality** visuals — feel free to expand or rewrite my script for stronger shots.

| 时间 | 节拍 | 必须出现的叙事要素 |
|------|------|-------------------|
| 0:00–0:05 | Hook | 「Meet the luckiest, yet unluckiest man in history」+ 1945 广岛上班 + 第一颗弹 |
| 0:05–0:15 | Twist | 烧伤存活 → **躲进避难所** → 次日回家 → 「Where was home? Nagasaki」 |
| 0:15–0:25 | Climax | 向老板描述广岛爆炸 → **窗外第二颗弹** |
| 0:25–0:30 | Outro | 再次存活 → **活到 93 岁** |

---

## 二、实际交付（trace 事实）

### 2.1 结构：4 节拍 → 3 段视频

| 段 | 时长 | 内容 |
|----|------|------|
| `yamaguchi_seq1_hiroshima` | 10.05s | 广岛清晨 → 爆炸 → 烧伤起身 |
| `yamaguchi_seq2_train_nagasaki` | 12.05s | 火车 → 长崎站 → 办公室第二颗弹（约 5s） |
| `yamaguchi_seq3_survivor` | 8.06s | 90 岁老人静坐 Outro |
| **合计** | **30.16s** | 硬切拼接 |

### 2.2 旁白 `yamaguchi_vo`

- 时长：**26.01s**（与画面 30.16s 不一致）
- **未包含**用户 Hook 金句「Meet the luckiest, yet unluckiest man in history」
- 起笔：`In 1945, Tsutomu Yamaguchi was in Hiroshima — on the wrong morning...`

### 2.3 生成参数

```
video_generate ×3:
  provider=seedance, model=doubao-seedance-2-0-260128
  mode=reference2video          ← 无 image_list
  sound=on                      ← 与 voiceover_narration 冲突
  duration: 10 + 12 + 8

execute_edit_video:
  视频轨 volume: 0.4 / 0.2 / 0.4
  VO vol=1.0 + BGM vol=0.08
  无 trim、无 duration enforcement
  未使用 subtitle_file
```

### 2.4 跳过的产线

- 未读 `subject-asset-skill`（无主角参考图）
- 非 VO-First（视频与 VO 并行生成）
- ASR 已跑但未用于混音 truth map

---

## 三、问题清单（现象 → 根因 → 解决办法）

### P0-1：Hook 金句从旁白消失

| 项 | 内容 |
|----|------|
| **现象** | 开场无用户脚本第一句 |
| **根因** | VO 被当作「故事摘要」重写；未触发 beat-locked / verbatim 模式 |
| **解决办法（重做）** | VO 第一句必须字面包含用户 Hook；生成前做句级 diff |
| **解决办法（skill）** | brainstorm/script：检测 `0:00-0:05` 等时间码 → 禁止删用户句 |

### P0-2：四节拍压成三段，Climax 被挤压

| 项 | 内容 |
|----|------|
| **现象** | 用户 Climax 10s → 实际仅 seq2 末 5s；Twist 与 Climax 同段 |
| **根因** | 为减少段数选 10+12+8，未尊重用户时间预算 |
| **解决办法（重做）** | 4 段建议 5+8+10+7s，Climax 独占一段 |
| **解决办法（skill）** | 用户时间码脚本禁止合并节拍 |

### P0-3：「躲进避难所」画面缺失

| 项 | 内容 |
|----|------|
| **现象** | VO 有 crawled to safety，画面无 shelter |
| **根因** | seq1 塞满爆炸戏；seq2 从火车开始 |
| **解决办法（重做）** | Twist 段专拍避难所/掩体再切火车 |
| **解决办法（skill）** | beat ↔ shot 校验表：每个 VO 意象至少 1 镜 |

### P1-4：音画时长不同步（26s vs 30s）

| 项 | 内容 |
|----|------|
| **现象** | 末尾约 4s 无旁白 |
| **根因** | 违反 VO-First；组装无 duration enforcement |
| **解决办法（重做）** | 先 VO → ffprobe → 再按句界分配段长 → trim 至 30.0s |
| **解决办法（skill）** | generation：voiceover 必须先 audio_produce；assembly：master=VO |

### P1-5：旁白模式 + co-gen 音轨叠加

| 项 | 内容 |
|----|------|
| **现象** | sound:on + 独立 VO + BGM，视频轨仍 0.2–0.4 |
| **根因** | co-gen 优先与 voiceover_narration 未二选一；ASR 未驱动音量 |
| **解决办法（重做）** | sound:off；合成视频 vol=0 |
| **解决办法（skill）** | 有独立 VO → 默认 sound:off；ASR 含 speech → vol=0 或重生 |

### P1-6：`reference2video` 无参考图

| 项 | 内容 |
|----|------|
| **现象** | mode=reference2video，image_list 为空 |
| **根因** | 复制模板未按 pure text2video 改路由 |
| **解决办法（重做）** | 改 text2video；有角色图后再 reference2video+image_list |
| **解决办法（skill）** | image_list 为空 → 强制 text2video |

### P1-7：跨段主角一致性无保障

| 项 | 内容 |
|----|------|
| **现象** | 三段并行文生，易三张脸 |
| **根因** | 跳过 subject-asset-skill |
| **解决办法（重做）** | 生成 yamaguchi 设定图 → 各段挂同一 ref |
| **解决办法（skill）** | ≥2 段同主人公 → 必读 subject-asset |

### P2-8：未 duration enforcement / 未用字幕

| 项 | 内容 |
|----|------|
| **现象** | 全长硬拼；subtitle_file 未烧录 |
| **根因** | assembly 未执行 skill 中的 enforcement 与字幕轨 |
| **解决办法** | trim 至 30s；可选烧 subtitle_file |

### P2-9：movie-quality 仅 720p

| 项 | 内容 |
|----|------|
| **现象** | 输出 1280×720 |
| **根因** | 未锁定更高分辨率；单 prompt 过长 |
| **解决办法** | creative lock 分辨率；每段单 beat 短 prompt |

---

## 四、调用时序（真实路径）

| # | 时间 (UTC) | 工具 | 说明 |
|---|------------|------|------|
| 1 | 21:10+ | read_file | brainstorm / script / generation / assembly skills |
| 2 | 21:11:46 | video_generate ×3 + audio_produce | 并行 batch，非 VO-First |
| 3 | 21:16:37 | ffprobe ×3 + ASR ×3 | 未读 transcript 调音量 |
| 4 | 21:17:00 | music_generate | BGM 29.99s |
| 5 | 21:17:49 | execute_edit_video | 硬切，无 trim |
| 6 | 21:18:06 | show_final_video | yamaguchi_final.mp4 |

---

## 五、失败因果链

```
用户 4 节拍 30s 分镜
  → Agent 误判为可自由改写 brief
  → 3 段 10+12+8（Hook/避难所/Climax 时长全偏）
  → 跳过 subject-asset + reference2video 无图
  → VO 与视频并行（26s vs 30s）
  → sound:on + 独立 VO + 固定音量混音
  → 成片不符合用户要求
```

---

## 六、推荐重做产线

```
brainstorm（锁定 30s / 16:9 / 4beat / voiceover_narration）
  → script（4 sequences ↔ 用户时间码 1:1）
  → subject-asset（主角设定图）
  → audio_produce（含 Hook 原句）→ ffprobe
  → video_generate ×4（text2video 或 ref+image_list, sound:off）
  → assembly（ASR truth map, 视频 vol=0, VO master, trim 30s）
  → show_final_video
```

### VO 结构示例

```text
[0-5s]  Meet the luckiest, yet unluckiest man in history. In 1945, Tsutomu Yamaguchi was in Hiroshima for work when the first atomic bomb struck.
[5-15s] He survived with burns. He crawled to a shelter. Somehow, the next day, he made it home. Where was home? Nagasaki.
[15-25s] While describing the Hiroshima blast to his boss, the second bomb detonated—right outside his window.
[25-30s] He survived that, too. He lived to the age of ninety-three.
```

---

## 七、Skill 改进建议（防复发）

| 规则 | 写入位置 | 要点 |
|------|----------|------|
| 时间码脚本检测 | brainstorm / Format Gate | ≥2 段时间码 → beat-locked，禁止合并 |
| Hook 保留 | vo-design-principles | 第一句与用户脚本字面一致 |
| VO-First | generation-skill | voiceover 必须先 VO 再视频 |
| sound 互斥 | video-generation-execution | 有独立 VO → sound:off |
| reference 空参 | generation-skill | 无 image_list → text2video |
| 多段同角色 | subject-asset-skill | ≥2 段同主人公必读 |
| ASR truth map | assembly-skill | 必读 transcript 再定 volume |
| duration enforcement | assembly-skill | 成片 = 用户目标 ±0.2s |

---

## 八、数据文件索引

| 文件 | 说明 |
|------|------|
| `analysis/langfuse-data/cases/45125516390/trace-1-b1598742.json` | 主 trace |
| `analysis/langfuse-data/cases/45125516390/assets-with-prompts.json` | 素材与 prompt |
| `analysis/langfuse-data/cases/45125516390/media/` | 下载的 mp4/mp3 |
| `analysis/langfuse-data/cases/45125516390/qa-report.html` | 可视化 QA 看板 |

---

*由 Langfuse trace 复盘生成，用于团队对齐根因与改进方向。*
