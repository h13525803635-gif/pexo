# Case 33300979274 — HyperFrames 成片无音频/无旁白根因与解决方案

## 结论

项目 `33300979274` 的最终视频没有音频、没有旁白，并不是音频在合成或导出阶段丢失，而是生产链路从一开始就没有创建任何音频资产。

这单走的是 HyperFrames HTML/SVG 动画路线：Agent 只制作了血液循环的视觉动画、箭头流向和 typewriter 风格文字标签，未调用 TTS、音乐或音效生成工具，也未在 HTML composition 中挂载 `<audio>` 元素。最终虽然尝试做媒体 probe，但两次 probe 都因资产引用错误失败；失败后没有阻断交付，仍然直接 `show_final_video`。

## 需求与实际交付

用户原始需求：

> Generate a 30-second explainer video about human blood circulation. Show the heart pumping blood through arteries and veins with animated arrow lines tracing the flow path. Add typewriter-style text labels for "Heart", "Lungs", "Arteries", "Veins" as each organ appears.

需求没有显式写 `voiceover` / `narration` / `music`，但它是一个 `30-second explainer video`。对 explainer 类视频，Agent 应先做 `audio_intent` 判断；如果不确定是否静音，应询问或默认规划旁白/BGM，而不是直接交付静音视频。

## Trace 证据

已下载 trace：

- `analysis/langfuse-data/cases/33300979274/trace-2-880fb778.json`
- 另有一条更早 trace `d7ad01851f1520012d0b51291221cf16`，Langfuse 详情接口返回 HTTP 500，未能下载；但已下载的交付 trace 足以解释最终成片无声。

关键工具调用统计：

| 工具 | 次数 | 含义 |
|---|---:|---|
| `write_file` | 1 | 写入 HyperFrames HTML skeleton |
| `edit_file` | 21 | 填充 SVG/HTML 场景与 GSAP timeline |
| `render_frame` | 5 | 渲染静帧预览 |
| `submit_render` | 5 | 提交 HyperFrames 渲染 |
| `query_render` | 91 | 查询渲染状态 |
| `get_file_info` | 1 | 获取最终 MP4 信息 |
| `show_final_video` | 1 | 交付最终视频 |
| `audio_produce` | 0 | 未生成 TTS/旁白 |
| `music_generate` | 0 | 未生成 BGM/音效 |

HTML 写入内容检查：

| 检查项 | 结果 |
|---|---:|
| `<audio` 元素 | 0 |
| `audio_produce` | 0 |
| `music_generate` | 0 |
| `narration` / `voiceover` / `voice` 相关规划 | 0 |

最终渲染与交付：

1. `submit_render` 最终成功，输出 `/projects/33300979274/workspace/assets/blood-circulation-explainer.mp4`。
2. Agent 随后尝试 `probe_media(file="asset://a_5mqilgtv")`，失败：`a_5mqilgtv` 不是合法 asset id。
3. 再尝试 `probe_media(file="file:///projects/33300979274/workspace/assets/blood-circulation-explainer.mp4")`，失败：渲染环境无法解析该 file 路径。
4. Agent 调用 `get_file_info` 得到真实 asset id：`a_RTwW5Td`。
5. 未使用正确 signed URL 或 asset id 重新 probe，直接 `show_final_video` 交付。

因此，最终视频无声的根因链路是：

```text
explainer 需求
  -> 未做 audio_intent 判断
  -> 直接进入 HyperFrames 视觉制作
  -> 未调用 audio_produce / music_generate
  -> HTML 中没有 <audio>
  -> submit_render 输出 video-only MP4
  -> probe_media 两次失败但未阻断
  -> show_final_video 交付静音成片
```

## 根因

### P0：音频意图门禁缺失

`30-second explainer video` 被当成纯视觉动画处理。Agent 没有在制作前声明：

- `audio_intent = explicit_silent`
- `audio_intent = narration_required`
- `audio_intent = bgm_or_sfx_required`
- `audio_intent = ask_user`

在没有 `explicit_silent` 的情况下，直接进入 HTML composition，是本次无旁白/无音频的第一根因。

### P0：HyperFrames HTML 没有音频挂载

即使不生成旁白，若需要 BGM 或音效，也必须有 `<audio src="https://...">`。本单 HTML 中没有任何 `<audio>`，所以渲染器没有可混入的音频源。

### P1：交付前音轨 probe 失败未阻断

HyperFrames skill 中已有要求：非明确静音时，最终交付前必须 `probe_media`，确认成片包含 audio stream。本单 probe 失败后没有继续修正引用并复查，而是直接交付，导致 QA 防线失效。

### P2：用户需求未显式要求旁白，但 Agent 没有澄清

用户没有明确说“需要旁白”，但也没有说“silent”。对解释类视频，如果 Agent 选择不加音频，应在制作前澄清；否则至少应默认提供轻量旁白或 BGM。

## 解决方案

### 1. 增加 explainer 音频意图门禁

在 HyperFrames / 视频制作入口前增加判断：

```text
if request contains explainer/tutorial/educational/lesson/how it works:
  if user explicitly says silent/no audio/no voice:
    audio_intent = explicit_silent
  elif user mentions labels/text only and no audio:
    audio_intent = ask_user
  else:
    audio_intent = narration_required
```

对本单，推荐判定为：

```text
audio_intent = narration_required
```

至少应生成 30 秒英文旁白，或向用户确认是否要静音。

### 2. 非静音任务必须先产出音频资产再写 HTML

推荐链路：

```text
write narration script
  -> audio_produce(provider=elevenlabs, mode=text2speech)
  -> optional music_generate for subtle educational bed
  -> get_file_info for each audio asset
  -> embed <audio src="https://..."> in HyperFrames HTML
  -> submit_render
  -> probe_media final output
  -> show_final_video only if audio stream exists
```

HTML 中应显式挂载：

```html
<audio
  src="https://signed-url/voiceover.mp3"
  data-start="0"
  data-duration="30"
  data-track-index="10">
</audio>
```

如有 BGM：

```html
<audio
  src="https://signed-url/educational-bed.mp3"
  data-start="0"
  data-duration="30"
  data-track-index="11"
  data-volume="0.18">
</audio>
```

### 3. 渲染前 lint 增加音频断言

在 `submit_render` 前做 composition 检查：

```text
if audio_intent != explicit_silent:
  require html contains "<audio "
  require every planned VO/BGM/SFX asset has a matching <audio src="https://...">
  block submit_render if missing
```

### 4. 交付前 probe 必须用正确资产引用

本单错误地把 job id 拼成了 `asset://a_5mqilgtv`。正确做法：

1. 使用 `query_render` 返回的 file 路径调用 `get_file_info`。
2. 用 `get_file_info.signed_url` 或真实 `asset_id` 进行 `probe_media`。
3. 检查 streams 中包含 audio。
4. 如果没有 audio stream 且 `audio_intent != explicit_silent`，禁止 `show_final_video`。

阻断规则：

```text
if probe_media failed:
  do not deliver
  fix media reference and retry

if probe_media succeeds but no audio stream and audio_intent != explicit_silent:
  do not deliver
  regenerate/mux audio and retry
```

### 5. 对 explainer 的默认成片规格

本类教育解释视频建议默认：

- 旁白：1 条 30 秒英文 VO，语速约 130-150 wpm。
- BGM：可选，低音量、无主旋律抢占。
- 文字标签：保留 typewriter 样式，与旁白关键词同步。
- 最终 QA：`probe_media` 必须确认 `video + audio` 双流。

## 修复验收标准

同类任务只有同时满足以下条件才允许交付：

1. 有明确 `audio_intent`。
2. 若不是 `explicit_silent`，至少存在一条真实音频资产。
3. HTML 中至少有一个 `<audio src="https://...">`。
4. `submit_render` 前音频断言通过。
5. 最终 MP4 的 `probe_media` 成功。
6. 最终 MP4 streams 包含 audio。
7. `probe_media` 失败时不得调用 `show_final_video`。

## 一句话复盘

这单无声不是导出 bug，而是 Agent 把 explainer 视频当成了纯视觉 HTML 动画来做：没有音频规划、没有 TTS/BGM 生成、没有 `<audio>` 绑定，且最终音轨校验失败后仍然交付。
