# 项目 53740015906：底图未走图生视频链路且成片无音频

## 结论

本 case 的问题不是图生视频失败，也不是音频在最终合成阶段丢失，而是从规划阶段就走错了生产链路：

1. 主底图 `09_01_physics_experiment.png` 被当作 HTML 静态背景图写入 HyperFrames composition。
2. 任务没有调用 `video_generate`，因此底图没有进入 `image_list`，也就没有机会生成动态视频。
3. 任务没有调用 `audio_produce` 或 `music_generate`，HTML 中也没有 `<audio src=...>`，最终交付前未做音轨校验，因此成片无声。

用户原始需求是：

> Generate a physics class video using 02_education/01_physics_experiment.png. Overlay a Picture-in-Picture window in the top right corner showing a close-up of a sliding block experiment. From 00:05 to 00:10, overlay a dynamic red arrow pointing towards the PiP area. Simultaneously, fade in a formula card at the bottom of the screen reading 'F = ma | μ = 0.3 | a = 2.94 m/s²'.

这个需求同时包含两类信号：

- 图生视频信号：`Generate ... video using ...png`
- 后期动效信号：PiP、红箭头、指定时间段、公式卡

Agent 过度抓住了后期动效信号，直接选择 HyperFrames 进行 HTML/GSAP 合成，忽略了“使用底图生成视频”的主链路意图。

## Trace 证据

Langfuse trace：`pexo:53740015906`，trace id `a42b03a57193171ba4f4ecb6293c0534`，开始时间 `2026-06-11T02:31:42Z`。

关键工具调用顺序：

| 时间 | 调用 | 说明 |
| --- | --- | --- |
| 02:31:51 | `read_file /.skills/0/hyperframes-skill/SKILL.md` | 一开始进入 HyperFrames 静态 composition 链路 |
| 02:31:51 | `analyze_file_content` | 分析用户上传的物理实验图片 |
| 02:32:26 | `get_file_info` | 仅获取主底图 signed URL |
| 02:32:26 | `image_generate` | 生成右上角 PiP 的滑块实验静态图 |
| 02:33:33 | `write_file physics_class.html` | 写 HTML composition |
| 02:33:37 | `render_frame` | 验证 HTML 单帧 |
| 02:34:44 | `submit_render` | 提交 HTML 渲染成 MP4 |
| 02:38:04 | `query_render` | 渲染完成，asset `a_sNrNKrg` |
| 02:38:22 | `show_final_video` | 交付最终视频 |

缺失的关键调用：

- 没有 `video_generate`
- 没有 `audio_produce`
- 没有 `music_generate`
- 没有最终 `probe_media` 音轨校验

HTML composition 中主底图实际写法为：

```html
<div id="scene-bg" data-start="0" data-duration="15" data-track-index="0">
  <img id="bg-img" src="https://..." alt="" />
  <div id="bg-overlay"></div>
</div>
```

这说明主图只是一个 15 秒静态 `<img>` 背景层，而不是图生视频输出。

PiP 也只是静态图：

```html
<div id="pip-wrapper" data-start="0.5" data-duration="14.5" data-track-index="1">
  <img id="pip-img" src="https://..." alt="" />
  <div id="pip-label">Close-up: Sliding Block</div>
</div>
```

HTML 中没有任何 `<video>` 或 `<audio>` 资产层。

## 为什么没有走“底图图生视频”链路

根因是路由规则缺少硬门槛。

当前 agent 在看到 PiP、箭头、公式卡、时间点叠加时，把任务归类成了 HyperFrames 动效合成任务。HyperFrames 确实适合做 overlay、卡片、箭头、字幕和时间轴动画，但它不负责把上传图片动态化。

正确的拆解应是：

```text
uploaded image
  -> video_generate(image_list=[uploaded image])
  -> generated base video
  -> HyperFrames / assembly overlay PiP + arrow + formula card
  -> audio bind + final probe
```

实际链路变成了：

```text
uploaded image
  -> get_file_info
  -> HTML <img> background
  -> HyperFrames render MP4
```

因此底图没有被生成成动态视频。

## 为什么无音频

音频不是被合成阶段丢掉，而是没有被创建和挂载。

Trace 中没有任何音频生产调用：

- 没有 `audio_produce`
- 没有 `music_generate`
- 没有 BGM/SFX/环境声资源
- HTML 中没有 `<audio src="https://...">`

HyperFrames 不会凭空生成音频。HTML composition 没有音频元素时，输出 MP4 自然只有视频流。

另外，最终 `show_final_video` 前没有 `probe_media` 校验音轨，导致无声视频没有被阻断。

## 修复方案

### 1. 增加主视觉图片到视频的路由硬门槛

当满足以下条件时，必须先走图生视频：

- 用户说 `generate/create/make video using image`
- 主视觉是用户上传的 `png/jpg/jpeg`
- 用户没有明确要求“静态海报动画”“只做图片动效”“slideshow”

则必须执行：

```text
video_generate(image_list=[uploaded_primary_image])
```

HyperFrames 只能作为后续 overlay/composition 层，不能直接消费上传图作为静态背景交付。

### 2. 对本 case 的正确生产链路

建议重新生成：

1. 用主底图生成 15 秒基础视频：
   - 教室场景保持一致；
   - 轻微镜头推进或稳定手持感；
   - 老师、学生、摆锤有细微运动；
   - 物理实验氛围真实自然。
2. 生成或准备右上角 PiP 滑块实验素材：
   - 如果用户期待 PiP 也动起来，PiP 也应走 `video_generate`；
   - 如果只需要说明性窗口，静态 PiP 图片可接受。
3. 用 HyperFrames 或 assembly 叠加：
   - 右上角 PiP；
   - 00:05-00:10 红色箭头；
   - 00:05 起底部公式卡 `F = ma | μ = 0.3 | a = 2.94 m/s²`。
4. 生成并挂载音频：
   - 可用轻微 classroom ambience、实验室环境声、柔和教学 BGM 或短 SFX；
   - 非明确静音时必须写入 `<audio src="https://...">`。
5. 交付前用 `probe_media` 校验：
   - 有 video stream；
   - 非静音需求必须有 audio stream；
   - 时长约 15 秒；
   - 关键 overlay 出现在 00:05-00:10。

### 3. 增加交付阻断规则

在 `submit_render` 或 `show_final_video` 前增加检查：

```text
if user_intent != explicit_silent:
    require audio asset exists
    require HTML contains <audio src="https://...">
    require final probe_media shows audio stream
```

否则禁止交付，必须补音频或向用户明确说明当前是 silent 版本。

### 4. 增加 trace/QA 自动检测

可在 QA 阶段增加以下自动判定：

- 用户请求包含 `video using image` 且 trace 无 `video_generate`：标记为 route_error。
- 主 HTML 只有 `<img>` 无 `<video>` 且任务不是静态图片动画：标记为 static_background_error。
- 非静音任务无 `audio_produce/music_generate/<audio>`：标记为 missing_audio_error。
- `show_final_video` 前无 `probe_media`：标记为 delivery_validation_missing。

## 一句话复盘

这次失败的本质是：agent 把“用底图生成视频并叠加动效”的任务误执行成了“静态图片 HTML 动效包装”；音频则是从生产计划到最终 HTML 都没有生成或挂载。
