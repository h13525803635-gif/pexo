# Case 30929014795：跳剪、音画不同步、字幕与配音不对位 — 问题与解决方法

> 项目 ID（Langfuse / Pexo）：**30929014795**  
> 分析依据：Langfuse trace `2e922ad9`（Kanban explainer 成片）  
> 文档日期：2026-05-09

---

## 一、现象（用户侧）

1. **画面跳剪**：两段视频衔接处观感突兀。  
2. **音画不同步**：旁白与画面节奏对不上。  
3. **字幕与配音不对位**：烧录字幕与口播时间不一致。

---

## 二、根因（技术侧）

### 2.1 跳剪

- `execute_edit_video` 中视频轨两段之间显式配置 **`transitions: [{"type": "cut"}]`**，即硬切。  
- **硬切本身不保证视觉连续**：未配置「前段尾帧 → 后段首帧 / 参考图」的跨段 handoff 时，两段可视为独立镜头，观感即「跳剪」。

### 2.2 音画不同步（主因）

| 事实 | 说明 |
|------|------|
| 两段 `video_generate` 各约 **10.054s**，总长 **≈20.108s** | 与编辑里 `out_ms: 10054`×2 一致。 |
| TTS `kanban_vo_final` 返回 **`duration`: 22.476s** | 实际旁白文件长度。 |
| 时间线上 VO 使用 **`in_ms: 0`, `out_ms: 20000`** | 只播放前 **20s**，约 **2.48s** 口播被裁掉。 |
| 两段视频 **`sound: on`**，成片视频轨 **`volume: 0.45`** | 片内声与独立 TTS 叠加，听感易乱。 |

结论：**时间线用「约 20s」当旁白全长，与真实 TTS 时长不一致**，且存在片内声 + 解说双轨，加重「不同步」体感。

### 2.3 字幕与配音不对位（根因）

- **字幕**来自 `execute_edit_video` 的 **`text` 轨**：每条 **`start_ms` / `end_ms` 写死**，相当于脚本/拍脑袋时间轴。  
- **配音**来自真实 TTS 波形；且存在 **裁切（只用到 20s）**、**`speed: 1.15`** 等，真实字句起止与写死毫秒不一致。  
- TTS 返回的 **`subtitle_file`（按发音的时间戳）未用于驱动烧录字幕** → **声与字两套「时间真相」**，必然不对位。

---

## 三、解决方法（可落地产研 / Skill）

### 3.1 硬切 + 视觉连续（多段视频）

- 在 **generation / handoff** 中显式要求：后段 **`video_generate`** 必须带入 **前段尾帧**（或等价锚图）作为 **`first_frame` / `image_list`**，再谈是否用 `cut`。  
- 「硬切」只解决剪辑形态，**不替代**跨段视觉锚点。

### 3.2 旁白与成片时长一致

- **`audio_produce` 之后**：用接口返回的 **`duration`** 或 **ffprobe** 作为 **`out_ms` / 总片长的单一真相源**，禁止用「段数×标称秒数」硬编码（如固定 20000）。  
- 若旁白 **长于** 当前视频总长：**改稿、调速、或加长/补段视频**，禁止静默裁掉尾部旁白。

### 3.3 独立解说时的片内声

- 存在全片或主线 **TTS/VO** 时：相关 **`video_generate` 使用 `sound: off`**；若素材已是 sound-on，成片里视频轨 **`volume: 0`**（仅保留极弱环境声时再单独约定电平）。  
- 避免「片内解说级声音 + 同角色 TTS」双主线（与仓库内 `analysis/audio-overlap-cases-summary.md` 原则一致）。

### 3.4 字幕必须跟配音走

- **优先**：用 TTS 返回的 **`subtitle_file`**（或统一对齐管线）生成烧录字幕的时间码。  
- **若必须手铺**：在 **TTS 定稿且裁切方案确定后**，按 **最终进轨的 MP3** 重算每条 `start_ms`/`end_ms`；**VO 轨 `out_ms` 与字幕共用同一时长源**。  
- TTS 使用 **`speed ≠ 1`** 时，禁止沿用常速假设的字幕表。

### 3.5 工程门禁（建议）

- `execute_edit_video` 提交前校验：**VO `out_ms` ≤ 源文件时长** 且 **与 `subtitle`/字幕轨一致**；**总视频时长与 VO/BGM 尾对齐策略**显式可查。  
- 对「目标总时长」类需求：在 trace 中记录 `target_duration`、`vo_probe_duration`、`vo_timeline_out_ms`，便于复盘。

---

## 四、流程澄清（回应常见误解）

- **配音顺序**：本 case 中已是 **先两段 `video_generate`，再 `audio_produce`，再剪辑**；裁短 **不是**「配音早于画面」导致，而是 **时间线写错 `out_ms`**。  
- **硬切与尾帧**：硬切 **不会自动** 触发尾帧接龙；需 **单独规格与工具链**。

---

## 五、参考数据摘录（便于对照）

- 成片名：`kanban_explainer_final`  
- 视频轨：两段各 `0–10054` ms，`transitions: cut`  
- VO 轨：`kanban_vo_final`，时间线 `0–20000` ms；TTS 元数据 **duration 22.476s**  
- BGM：`0–20108` ms（与视频总长接近）  
- `video_generate`：`kanban_seg1` / `kanban_seg2`，`duration` 标称 10s，`sound: on`  
- TTS：`eleven_v3`，`speed: 1.15`

---

## 六、本地 Trace（可选，不入库）

完整 trace JSON 体积较大，可保留在本地：

`analysis/langfuse-data/cases/30929014795/trace-1-2e922ad9.json`

拉取命令见 `~/.cursor/skills/case-analyzer/SKILL.md`（`run-fetch-case.sh --project-id 30929014795`）。
