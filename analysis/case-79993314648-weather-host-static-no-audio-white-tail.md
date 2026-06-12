# Case 79993314648：天气播报参考图未生视频、无音频与尾部白屏

## 结论摘要

项目 `79993314648` 的原始需求是：

```text
Generate a weather forecast broadcast video using 04_news/04_weather_forecast.png.
At 00:04, slide in three weather cards from the left edge to the right sequentially:
'Monday: Sunny 28°C', 'Tuesday: Heavy Rain 22°C', 'Wednesday: Cloudy 25°C'.
Overlay a localized animated rain sticker (looping blue raindrops) specifically on top of the Tuesday card.
```

参考图里有天气主持人，合理成片应是“主持人讲述天气 + 卡片动画 + Tuesday 雨滴贴纸”。实际链路没有让参考图进入视频生成模型，也没有生成/合成任何音频；它走的是 HyperFrames HTML 合成，把参考图当静态 `<img>` 背景，再叠加三张卡片和雨滴动画。因此：

- “参考图没有生视频”：成立。全链路 `video_generate = 0`。
- “没有音频/音效”：成立。全链路 `audio_produce = 0`、`music_generate = 0`，HTML 内也没有 `<audio>`。
- “最后两秒白屏”：高概率来自 HyperFrames 时长/clip 元数据错误：主 scene `data-duration="9"`，但提交渲染时已有 warning 指出 root 缺 `data-start`、scene 有 timing 却缺 `class="clip"`；agent 未 lint、未 frame preview、未 probe/视觉 QA 就交付。

本质不是视频模型生成失败，而是路由和门禁都错了：一个 reference image to weather broadcast video 的需求，被降级成静态 HTML 动效包装。

## Langfuse 证据

本地 trace：

- `analysis/langfuse-data/cases/79993314648/trace-1-fb14d38e.json`
- `analysis/langfuse-data/cases/79993314648/trace-index.json`

trace 信息：

```text
trace: pexo:79993314648
start_time: 2026-06-11T02:39:02.185Z
trace_count: 1
```

关键工具调用统计：

| 工具 | 次数 | 含义 |
|---|---:|---|
| `analyze_file_content` | 1 | 只分析了参考图内容 |
| `get_file_info` | 1 | 获取参考图 signed URL |
| `write_file` | 1 | 写入 HyperFrames HTML |
| `submit_render` | 1 | 提交 HTML 渲染 |
| `query_render` | 12 | 查询渲染状态 |
| `show_final_video` | 1 | 交付最终 MP4 |
| `video_generate` | 0 | 未图生视频 |
| `audio_produce` | 0 | 未生成主持人口播/旁白 |
| `music_generate` | 0 | 未生成 BGM/音效 |
| `execute_edit_video` | 0 | 未走后期合成音视频 |
| `render_frame` / `lint_composition` / `probe_media` | 0 | 未做有效视觉/音轨 QA |

最终交付：

```text
show_final_video("/projects/79993314648/workspace/assets/weather_forecast_broadcast.mp4")
-> asset_id: a_DYF7wAs
```

## 真实调用链路

```text
用户上传天气播报参考图 + 卡片/雨滴动画需求
  -> analyze_file_content：识别图中有女主持人、天气地图、电视播报 UI
  -> get_file_info：获取参考图 signed_url
  -> write_file：生成 weather_forecast.html
       <img class="bg" src="参考图 signed_url">
       三张 weather-card div
       JS 注入 rain-sticker raindrops
       GSAP timeline：卡片从 t=4s 开始滑入
  -> submit_render：HyperFrames 渲染 HTML 到 MP4
  -> query_render：渲染完成
  -> show_final_video：交付 weather_forecast_broadcast.mp4
```

这条链路没有任何视频生成模型调用。主持人、地图和背景都只是同一张 PNG 被铺满画面。

## 关键 HTML 证据

`write_file` 中的 HTML 只有静态图片背景：

```html
<img class="bg" src="https://.../24_04_weather_forecast.png?...">
```

没有视频或音频媒体：

```text
<video: 0
<audio: 0
```

主 scene 被写成 9 秒：

```html
<div id="scene-main" data-start="0" data-duration="9" data-track-index="0">
```

卡片动画从 4 秒开始：

```js
tl.to("#card-monday", { x: 0, opacity: 1, duration: 0.65 }, 4.0);
tl.to("#card-tuesday", { x: 0, opacity: 1, duration: 0.65 }, 4.3);
tl.to("#card-wednesday", { x: 0, opacity: 1, duration: 0.65 }, 4.6);
```

提交渲染时已有 warning：

```text
[timed_element_missing_clip_class]
<div id="scene-main"> has timing attributes but no class="clip".

[root_composition_missing_data_start]
Root composition "weather-forecast-v1" is missing data-start.
```

agent 没有修复 warning，也没有做 `render_frame` 或 `probe_media`，直接交付。

## 根因

### P0：路由错误，把“参考图生成播报视频”误走 HyperFrames 静态包装

用户说的是 `Generate a weather forecast broadcast video using ...png`，并且参考图里存在主持人。正确策略应至少评估：

- 是否需要 image-to-video / reference-to-video 让主持人、手势、镜头、背景产生视频运动。
- 是否需要主持人口播音频。
- 卡片/雨滴是否作为后期 overlay 合成。

但实际 agent 直接选择 HyperFrames，把参考图当背景图，没有调用 `video_generate`。这导致“主持人讲述天气”的核心画面不可能出现。

### P0：音频意图门禁缺失

天气播报、主持人、broadcast video 都强烈暗示需要口播或至少播报类音效/BGM。实际没有：

- 主持人口播脚本。
- TTS/voiceover。
- BGM/天气音效。
- HTML `<audio>`。
- 最终音轨 probe。

因此无声不是导出丢音轨，而是压根没有音频资产。

### P1：HyperFrames clip 元数据错误导致尾部白屏风险

HTML 使用了 timing attributes，但主 scene 缺 `class="clip"`，root 缺 `data-start="0"`。submit_render 明确返回 warning，说明 runtime 的可见性和播放起点可能不稳定。

主 scene `data-duration="9"`，而需求没有明确总时长；卡片 t=4s 进入，雨滴按 `9 - RAIN_START` 只覆盖到 9s。如果渲染器输出时长超过有效 scene，比如 11s，后两秒就没有可靠可见内容，容易露出默认空白底。

### P1：交付前 QA 防线缺失

agent todo 写了 “Lint and frame-preview the composition”，但真实工具调用里没有 `lint_composition`、`render_frame`、`probe_media`。这使三个问题都没被阻断：

- 没有发现末尾白屏。
- 没有发现音轨缺失。
- 没有发现 warning 未修复。

## 正确解决方案

### 1. 正确生产链路

本案应拆成“生成主持人视频 + 后期卡片包装 + 音频合成”：

```text
参考图
  -> video_generate(image2video/reference2video)
       prompt: weather presenter speaks and gestures toward forecast map
       duration: 9-12s
       image_list: [04_weather_forecast.png]
  -> audio_produce
       生成 9-12s 天气播报口播
  -> optional music_generate / sfx
       轻量新闻床乐、Tuesday 雨滴音效
  -> execute_edit_video 或 HyperFrames overlay
       主视频轨：主持人播报视频
       overlay：三张天气卡片 t=4s 顺序滑入
       overlay：Tuesday 卡片雨滴 sticker
       audio：VO + BGM/SFX
  -> probe_media + 抽帧 QA
  -> show_final_video
```

如果必须使用 HyperFrames 做卡片动画，也应把 video_generate 输出作为 `<video>` 背景，而不是把原始 PNG 当 `<img>` 背景。

### 2. 路由门禁

增加规则：

```text
if user asks "generate video using image" and image contains person/host/presenter:
  require video_generate unless user explicitly asks for static motion graphics

if request contains broadcast/news/weather forecast/presenter:
  require audio_intent in {voiceover_required, explicit_silent, ask_user}
  default to voiceover_required unless user says silent/no audio
```

### 3. 音频门禁

非 explicit silent 时：

```text
require audio asset before render/assembly
require final probe contains audio stream
block show_final_video if audio stream missing
```

对本案可生成类似 10 秒英文口播：

```text
"Here is your three-day outlook. Monday will be sunny and warm at 28 degrees.
Tuesday brings heavy rain with a cooler 22 degrees. Wednesday turns cloudy, reaching 25."
```

### 4. HyperFrames 修复

如果保留 HyperFrames 合成：

- root wrapper 加 `data-start="0"`。
- timed scene 加 `class="clip"`。
- composition 总时长与 scene/animation 时长一致，例如统一 10s 或 12s。
- 背景层覆盖完整总时长；最后至少 freeze 主视频/图像，不留空白尾段。
- submit_render warning 必须视为阻断，修复后再渲染。

示例：

```html
<div data-composition-id="weather-forecast-v1"
     data-width="1920"
     data-height="1080"
     data-start="0"
     data-duration="10"
     class="wrapper clip">

  <div id="scene-main"
       class="clip"
       data-start="0"
       data-duration="10"
       data-track-index="0">
```

### 5. 交付验收标准

同类任务必须同时满足：

1. 至少一次 `video_generate`，且 `image_list` 包含用户参考图。
2. 非静音时至少一次 `audio_produce` 或明确 BGM/SFX 资产。
3. 最终成片 `probe_media` 有 video + audio 双流。
4. 抽帧检查 `0s / 中段 / 最后 1s` 不为空白、不黑屏、不白屏。
5. HyperFrames render warnings 为 0；如有 warning，不得交付。

## 一句话复盘

这单坏在第一步路由：agent 把“天气主持人参考图生成播报视频”做成了“静态图片 + HTML 卡片动画”，所以主持人不会动、不会说话；同时没有音频资产和交付前 QA，末尾白屏也没有被阻断。
