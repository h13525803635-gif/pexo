# Case 50055018517：成片静图但后台四段视频正常 — 问题与解决办法

- **项目 ID**：50055018517  
- **Langfuse trace**：`pexo:50055018517`（`trace-1-c744808d`，trace id `c744808d41fe7fa6ee5414f3fbb47176`）  
- **用户需求**：45s / 16:9，Memphis 风「Zero to Hero: Photoshop & Illustrator」讲解，动态展示从空白画布到成品海报  
- **成片资产**：`/projects/50055018517/workspace/assets/zero_to_hero_final.mp4`  
- **分析日期**：2026-06-01  

---

## 一、现象

用户侧：成片**看起来像一张静止图片 + 完整英文旁白**，没有预期的四段动态画面切换。

后台：4 段 `video_generate` 均 **ok**，`show_final_video` 指向 `zero_to_hero_final.mp4`，Agent 报告合成成功（45.29s）。

---

## 二、结论（一句话）

**4 段视频生成正常；问题出在 `video-editor__execute_edit_video` 合成/export 把视频轨压成近乎静帧，音频轨（VO + BGM + SFX）正常。** 不是「没生成视频」，而是「合成没把视频拼进去」。

---

## 三、验证数据（本地复现核对）

分析数据目录：`analysis/langfuse-data/cases/50055018517/`

| 对比项 | seg1–4 源视频 | 成片 `zero_to_hero_final` | ffmpeg 正确拼接测试片 `reassembled_test.mp4` |
|--------|---------------|-------------------------|---------------------------------------------|
| 文件大小 | 各 ~4–5 MB | **~1.5 MB** | **~18 MB** |
| 视频码率 | ~2.5–3.5 Mbps | **~75 kbps** | **~3.2 Mbps** |
| 0s vs 11s 画面差异 (MAD) | seg2 段内 ~15 | **~0.17（几乎不动）** | **~20（正常切场）** |
| 11s 画面 vs seg2 内容 | 绿底工具分屏 | **仍像 seg1 黄底画布** | **与 seg2 一致** |
| 45s 内不同画面数 (mpdecimate) | — | **约 4 帧** | 正常多场景 |

源段 mid-frame 内容（抽帧确认）：

- **seg1**：黄底 + 中央空白画布  
- **seg2**：左右分屏工具图标（Ps / Ai 示意）  
- **seg3**：草图 → 上色海报对比  
- **seg4**：蓝色 CTA 卡片 + 星形爆发  

成片在 0s / 11s / 23s / 36s 抽帧均为 **seg1 同款黄底画布静帧**，未出现 seg2–4 画面。

---

## 四、根因分解

### 4.1 合成时间线配置错误（Agent / assembly）

**A. 多余的 `overlay` 轨**

合成参数包含 `kind: overlay`，素材为 **`style_ref_s1_blank_canvas.png`**（开场静图），`opacity: 0`，`end_ms: 1`。  
成片视觉即该静图样式。若合成器对 `opacity` / `end_ms` 处理异常，可能**整段 45s 被 PNG 盖住**，底层 4 段视频不可见。

**B. SFX 音轨缺少 `start_ms`**

台账规划 SFX 应对齐 0 / 10.042s / 22.084s / 35.126s，但 `execute_edit_video` JSON 中从 4 个 seg mp4 抽取的 **audio clip 均无 `start_ms`**（4 轨叠在 0ms），仅 VO 轨有 `start_ms`。  
对比正常 case（如 21670845003），从视频抽 SFX 必须带时间轴偏移。违反 `video-editor-tool-guide` 时间线规则。

**C. 合成后未做抽帧验收**

Agent 仅确认 `success: true` 与时长 45s，未执行 assembly-skill 要求的「成片抽帧 vs 各 seg 对比」，静图成片未被拦截。

### 4.2 合成器 export 缺陷（工程）

- 视频轨写了 4 个 clip + cut，但输出表现为「每段仅首帧或极少帧 + 音频正常」  
- 符合 `execute_edit_video` **多 clip 拼接 / 重编码 / overlay 交互** 类 bug  

### 4.3 生成阶段（次要，非本次静图主因）

4 段均为 Seedance `reference2video`，每段仅 1 张 `style_ref` 静图作锚点；段内运动幅度有限。  
但 **源 seg2–4 画面内容与成片完全不同**，故静图主因仍是合成，而非「生成只有静图」。

---

## 五、Trace 关键事实

| 步骤 | 工具 | 结果 |
|------|------|------|
| 风格静图 | `image_generate` ×4 | `style_ref_s1` … `s4` .png 成功 |
| 分段视频 | `video_generate` ×4 | `seg1`…`seg4` .mp4 均 ok |
| 旁白 | `audio_produce` ×4 | vo_line1–4 .mp3 |
| BGM | `music_generate` | bgm_upbeat_synth .mp3 |
| 合成 | `execute_edit_video` | `zero_to_hero_final.mp4`，`success: true` |
| 交付 | `show_final_video` | 指向 final mp4 |

合成视频轨 clips（节选）：

- seg1: `out_ms` 10042  
- seg2: `out_ms` 12042  
- seg3: `out_ms` 13042  
- seg4: `out_ms` 10042  

---

## 六、解决办法

### 6.1 立刻救急（推荐，不重生 4 段）

对项目 **50055018517** 用已有 4 段 mp4 **重新合成**：

1. **删除整个 `overlay` 轨**（禁止用 `style_ref_*.png` 铺底）  
2. **保留视频轨** 4 clip + cut  
3. **SFX 轨** 为每段补 `start_ms`：

| Clip | start_ms | 说明 |
|------|----------|------|
| seg1 SFX | 0 | 0–10.1s |
| seg2 SFX | 10042 | 10.1–22.1s |
| seg3 SFX | 22084 | 22.1–35.1s |
| seg4 SFX | 35126 | 35.1–45.2s |

4. VO + BGM 轨保持现有参数  
5. 合成后执行 **§6.4 验收清单**，通过后再 `show_final_video`

本地验证：对 4 段 mp4 做 ffmpeg concat 得到 `media/reassembled_test.mp4`，画面与码率正常，证明素材可复用。

### 6.2 工程修复（治本）

| 项 | 动作 |
|----|------|
| 多 clip 拼接 | 修复「每段只出 1 帧」；校验 `nb_frames`、场景切变数 |
| overlay | 修复 `opacity: 0` / `end_ms` 异常；或禁止 style_ref 全屏 overlay |
| 导出质检 | 成片视频码率不得低于源片均值 30%；45s 成片场景切变 ≥ 3 |
| 失败策略 | 质检不通过返回失败，禁止交付 |

### 6.3 技能 / Agent 规则（防再发）

写入 **assembly-skill** / **video-editor-tool-guide**：

1. 禁止用 `style_ref` / `image_generate` PNG 作全片 overlay  
2. 从 mp4 抽 SFX 时每条 clip **必须有 `start_ms`**  
3. `execute_edit_video` 成功后必做抽帧对比（0 / 25% / 50% / 75%  vs 各 seg）  
4. 相邻 10s 抽帧 MAD < 5 → 判定「静图成片」，BLOCK 交付  

### 6.4 合成后验收清单

```
□ 成片时长 ≈ 计划时长（本 case 45s）
□ 成片视频码率 > 500 kbps（本 case 实际 ~75 kbps → FAIL）
□ 在 10s / 22s / 35s 抽帧，画面分别接近 seg2 / seg3 / seg4
□ 任意相邻 10s 抽帧 MAD > 5（本 case ~0.17 → FAIL）
□ 无 overlay 使用 style_ref 全屏铺底
```

---

## 七、相关文件

| 路径 | 说明 |
|------|------|
| `analysis/langfuse-data/cases/50055018517/trace-1-c744808d.json` | 完整 trace |
| `analysis/langfuse-data/cases/50055018517/url_map.json` | OSS 签名 URL |
| `analysis/langfuse-data/cases/50055018517/media/*seg*.mp4` | 4 段源视频（分析时下载） |
| `analysis/langfuse-data/cases/50055018517/media/zero_to_hero_final_dl.mp4` | 问题成片（分析时下载） |
| `analysis/langfuse-data/cases/50055018517/media/reassembled_test.mp4` | ffmpeg 正确拼接对照 |

---

## 八、操作顺序建议

1. **今天**：按 §6.1 重合成，替换 `zero_to_hero_final.mp4`  
2. **本周**：修 `execute_edit_video` + 导出质检（§6.2）  
3. **后续**：assembly 抽帧门禁（§6.3）；教程类需求优化多关键帧生成策略  

---

*文档由 Langfuse trace 与本地抽帧/码率对比生成。*
