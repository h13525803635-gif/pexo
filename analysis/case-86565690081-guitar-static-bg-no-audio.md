# 项目 86565690081：吉他底图未图生视频、只有乐谱动效、成片无声

## 结论

本 case 的问题不是图生视频失败，也不是最终渲染把音轨丢了，而是执行链路一开始就走成了 **HyperFrames 静态 HTML 动效合成**：

1. 用户要求是 `Generate a guitar playing tutorial video using 02_education/06_guitar_playing.png`，主图应作为图生视频输入。
2. 实际执行没有调用真实 `video_generate`，吉他图只被写入 HTML 的 `<img id="bg-img">`，所以背景人物/吉他本身不会动。
3. 画面里唯一持续运动的是 HTML/GSAP 做的乐谱横向滚动和和弦图高亮，因此观感上变成“只有乐谱在动”。
4. 没有调用 `audio_produce` / `music_generate`，HTML 中也没有 `<audio src=...>`，最终交付前没有 `probe_media` 音轨校验，所以成片无声。

## 用户原始诉求

> Generate a guitar playing tutorial video using 02_education/06_guitar_playing.png. Overlay an animated guitar chord chart at the bottom center, showing the transition from C → G → Am → F. At the top of the screen, overlay a strip of sheet music and apply a continuous right-to-left scrolling animation that lasts for the entire duration of the video.

关键意图包括：

- 用 `06_guitar_playing.png` 生成吉他教学视频。
- 顶部叠加持续滚动的乐谱。
- 底部叠加 C → G → Am → F 和弦图动画。
- 教学/吉他演奏场景天然需要声音，用户没有要求静音。

## 实际执行链路

trace 只有 1 条，核心工具时间线如下：

| 时间 | 工具 | 结果 |
|---|---|---|
| 2026-06-11 02:34:19 UTC | `analyze_file_content` | 分析上传图内容 |
| 2026-06-11 02:34:38 UTC | `get_file_info` | 获取 `14_06_guitar_playing.png` signed URL |
| 2026-06-11 02:35:34 UTC | `write_file` | 写入 `/projects/86565690081/workspace/guitar_tutorial.html` |
| 2026-06-11 02:39:30 UTC | `submit_render` | 首次 lint 失败：两个 label opacity no-op |
| 2026-06-11 02:40:00 UTC | `submit_render` | 第二次提交成功，job `3l33jy79` |
| 2026-06-11 02:40:08 UTC | `render_frame` | 预览图成功 |
| 2026-06-11 02:47:44 UTC | `query_render` | 渲染完成，asset `a_68G6g1R` |
| 2026-06-11 02:47:56 UTC | `show_final_video` | 交付 `/projects/86565690081/workspace/assets/guitar_tutorial_v1.mp4` |

本次没有真实视频生成链路：

- 无 `video_generate` 工具调用。
- 无 `audio_produce` / `music_generate` 工具调用。
- 无 `probe_media` / `ffprobe` 对最终 mp4 做音轨校验。

## 为什么吉他图没有生视频

HTML 里主图的用法是静态图片背景：

```html
<img id="bg-img" src="https://pexo-assets.../14_06_guitar_playing.png?..." alt="guitar" />
```

它不是 `<video>`，也没有先通过 `video_generate(image_list=[uploaded image])` 得到动态底片。因此最终画面中吉他/人物/手部不会发生真实演奏动作，只会作为静态背景被渲染 32 秒。

正确链路应该是：

```text
uploaded guitar image
  -> video_generate(image_list=[guitar image], prompt=吉他教学/演奏动作)
  -> generated base video
  -> HyperFrames 或 video-editor 叠加乐谱/和弦图
  -> 加 BGM/示范音/旁白
  -> probe_media 校验有 video + audio
```

## 为什么只有乐谱移动

实际成片的运动全部来自 HTML/GSAP：

- 顶部 `#sheet-scroll-track` 做右到左滚动。
- 底部和弦卡片做高亮、缩放、描边变化。
- 标签和进度文字做进出场。

这些都是后期二维 overlay 动效；底层吉他图没有被图生视频，所以用户看到的是“乐谱在动，底图不动”。

## 为什么没有声音

根因是音频在计划和执行阶段都缺失：

1. 没有先声明 `audio_intent`，也没有把“吉他教学”识别成需要音乐/示范音/旁白的非静音任务。
2. 没有调用 `audio_produce` 做教学旁白。
3. 没有调用 `music_generate` 或生成/绑定吉他示范音。
4. HTML composition 中没有任何 `<audio src="https://...">`。
5. `show_final_video` 前没有 `probe_media`，没有发现最终 mp4 是 video-only。

这违反了 HyperFrames skill 自身的防线：非 `explicit_silent` 时，HTML 需要至少一个 `<audio>`，最终交付前也应 probe 确认有 audio stream。

## 根因归类

| 问题 | 类型 | 说明 |
|---|---|---|
| 吉他图未动 | 路由错误 | “Generate video using image” 被误走成 HTML 静态包装 |
| 只有乐谱动 | 表现结果 | 只有 overlay 是 GSAP 动效，主视觉不是视频 |
| 无声音 | 音频规划缺失 | 未生产音频、未挂载 `<audio>`、未做最终音轨校验 |
| 未阻断交付 | QA 缺口 | 缺少 `video_generate` 与 audio stream 的交付前检查 |

## 修复建议

1. 对包含 `Generate video using <image>` / `using ...png` 的任务，先走 `video_generate`，除非用户明确只要动态图文包装。
2. HyperFrames 只负责后期叠加乐谱、和弦图、字幕等包装层，不应替代主底图的图生视频链路。
3. “教学/演奏/教程”类视频默认不是静音：至少需要 BGM、示范音或旁白之一，并在 HTML 中显式 `<audio src="https://...">` 挂载。
4. `show_final_video` 前必须 `probe_media`：非明确静音但没有 audio stream 时阻断交付。
5. 若最终走 HTML 合成，lint 通过后还应做语义 QA：主视觉是否是 `<video>` 或由 `video_generate` 输出，而不是只有 `<img>` 静态背景。

一句话总结：这次失败的本质是 **把“用吉他图生成教学视频并叠加乐谱/和弦图”的任务，执行成了“静态吉他图片 + HTML 乐谱和弦动效”的包装视频；音频则从生产计划到最终合成都没有进入链路。**
