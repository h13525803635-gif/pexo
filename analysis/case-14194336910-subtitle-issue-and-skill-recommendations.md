# 项目 14194336910：字幕问题说明与 assembly-skill 修改建议

**Langfuse trace**：`analysis/langfuse-data/cases/14194336910/trace-1-4e087206.json`（体积较大，未纳入 Git；可本地用 `case-analyzer` 的 `fetch-case.py` 按项目 ID 重新拉取。）  
**分析日期**：2026-04-24  

---

## 一、用户侧现象

- 字幕**重叠**或切换别扭。  
- 字幕应与**旁白（TTS）同期**；在**无旁白**的片段（仅画面/音乐）字幕仍不消失，或上一句已结束而字还在。  

---

## 二、根因（基于生产 trace 归因）

### 1. 未使用 TTS 返回的 `subtitle_file` 真时间轴

`audio_produce` 对每段旁白返回了 **`subtitle_file`（JSON）** 和 **`duration`**。Agent 在成片 **没有**用该文件驱动烧录，而是对 `execute_edit_video` 使用 **`kind: "text"`** 并**手写**每行 `start_ms` / `out_ms`，按文意分句后**粗估/均分**毫秒。  
因此字幕与真实朗读节奏**不对齐**，易出现：句已念完字还在、多行时间窗与语音错位、听感上像「没旁白仍有字」。

### 2. `subtitle` 轨首次 DSL 不符合渲染器要求

首版使用 **`kind: "subtitle"`**，`file` 指向各段 **MP3**，仅有 **`start_ms`**，缺少 **`out_ms`**。  
渲染失败，错误信息示例：`subtitle clip[0]: out_ms not set and no probe function available`（`video_editor.dsl_conversion_failed` / `fix_edit_spec`）。  
之后改为手搓 `text` 轨，把「系统约束问题」转成了「人工时间轴精度问题」。

### 3. 手写时间与 VO 总时长可核对不一致

例如 seg1：VO 从 timeline **1000ms** 起，`duration` 约 **8.627s**，语音约在 **9627ms** 结束；手写最后一行字幕曾落在 **6200–9800ms** 等区间，**尾段易长于实际发声**，表现为无旁白时字幕仍短暂停留。

### 4. 其它（非字幕主因，同 trace 中可见）

- 成片首版 BGM 轨**分段重复/堆叠**导致 `duration` 被拉到约 **106s**，后改为单条连续 BGM；属音轨编排问题。  
- 非最终交付版本时，用户可能看到的是中间产物。  

---

## 三、对 assembly-skill 的修改建议（可落地）

以下为建议写入 **`assembly-skill/SKILL.md`** 或独立 **`references/assembly-subtitles.md`**（由 SKILL 引用）的条文化内容。

### 1. 时间轴真源（硬规则）

- 凡经 **`audio_produce`** 的旁白，**必须优先**用返回的 **`subtitle_file`（JSON）** 生成烧录或字幕轨。  
- **禁止**以「将全文按句均分毫秒」作为唯一手段；若 `subtitle_file` 不可用，须以 **ffprobe 实测**每段 MP3 的 `format.duration` 为边界，并显式约束每句的 `out_ms`。

### 2. 多段 VO 与字幕对齐

- 每条 cue 的**时间线** = 该段 VO 在 `execute_edit_video` 中的 **`start_ms`** + cue 在 `subtitle_file` 内的**相对时间**（若产品约定 JSON 为绝对时间，应在 skill 中**写死一种**换算方式）。  
- 与既有「**相邻 VO 的 start 由 ffprobe 逐条累加**」规则一致，避免与音轨错位。

### 3. DSL 约束：`*subtitle*` 与 `*text*`

- **`kind: "subtitle"`** 若须引用源文件，须满足渲染器对 **`out_ms`（或等价）** 的要求，不得仅 `file: *.mp3` + `start_ms` 且无结束时间。  
- **更推荐**在说明中写清：烧录应来自 **解析 `subtitle_file` 后的 cue 表**，而不是把 MP3 当作未指定长度的字幕源。

### 4. 明确禁止

- 禁止：`*subtitle`* + 仅 MP3 路径 + 仅有 `start_ms` 且无 `out_ms`/probe。  
- 禁止：无 `subtitle_file`/duration 支撑的手拍「均匀切分」`*text`* 作为默认方案。

### 5. 调用 `execute_edit_video` 前自检（短清单）

- 每段 VO 的 **duration** 已来自 API 或 ffprobe。  
- 每条字幕的 `[start, end)` 落在此段 VO 的**有效语音区间内**；段与段**静音间隙**内无字幕 clip。  
- 若使用多段 `*text`*，**时间窗不得相交**。

### 6. 与画面内文字的关系

- 若需求为「清晰可读英文字幕」，以**后期烧录/轨**为主；避免在 `video_generate` 里同时堆大量画内说明字，与后期字幕叠在观感上误为「叠字」。

### 7. 交叉引用

- 在 `references/video-models-scheduling.md` 的 Post-assembly / 混音小节，**加一句**指向本字幕规范，保证改音量/VO 的同事同步看到字幕真源要求。

---

## 四、追溯说明

- 本仓库若未包含该 trace 的 JSON，可用环境变量 `LANGFUSE_*` 运行：  
  `~/.cursor/skills/case-analyzer/scripts/fetch-case.py --conversation-id 14194336910 --output-dir analysis/langfuse-data`  
- 远程技能更新流程见 case-analyzer skill 中 `update-skills.sh` 与 admin API 说明。  

---

## 五、修订记录

| 日期 | 说明 |
|------|------|
| 2026-04-24 | 初版：问题归因 + assembly-skill 修改建议，便于入库与分享 |
