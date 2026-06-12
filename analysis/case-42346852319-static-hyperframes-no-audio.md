# 42346852319 静止图、快转场、无音频根因

## 结论

本 case 最终交付的 `coffee-bean-to-cup-explainer.mp4` 不是由图生视频/文生视频片段合成，而是：

1. 用 Seedream `text2image` 生成 8 张静态图：4 个阶段各一张 raw + processed。
2. 写入 `/projects/42346852319/workspace/coffee-explainer.html`。
3. 多次调用 Hyperframes 将同一个 HTML 资产 `asset://a_6ZGLTqc` 渲染为 MP4。
4. 最终交付 `/projects/42346852319/workspace/assets/coffee-bean-to-cup-explainer.mp4`。

因此观感上是“静态图片做网页动画/幻灯片”，不是动态视频素材。

## 现象归因

### 1. 为什么都是静止图

Trace 中只看到 `media-orchestrator` 的 Seedream `text2image` 调用，生成：

- `harvest_raw`
- `harvest_processed`
- `roast_raw`
- `roast_processed`
- `grind_raw`
- `grind_processed`
- `brew_raw`
- `brew_processed`

没有看到 `video_generate` / 图生视频 / 文生视频调用。后续动作是把这些 PNG 放进 HTML 里，用 CSS/Hyperframes 渲染成 MP4，所以画面主体不会有真实运动，只会有平移、缩放、遮罩、文字等网页动画。

### 2. 为什么转场显得快

HTML 时间线被写成约 53 秒：

- title: `0-5s`
- harvest: `5-16s`
- roast: `16-27s`
- grind: `27-38s`
- brew: `38-48s`
- end: `48-53s`

主体段每段约 10-11 秒，但每段内部用的是“raw -> processed”的静图切换，加上场景切换/遮罩动画。因为没有视频镜头运动承接，任何图片替换都会显得像快速转场或 PPT 翻页。

### 3. 为什么没有音频

Trace 中没有看到音频生产或音频合成链路：

- 没有 `audio_produce`
- 没有 `music_generate`
- 没有 TTS/VO 生成
- 没有混音/音轨合成
- HTML 写入和 Hyperframes 渲染参数里也没有音频文件输入

所以最终 MP4 是无声渲染。

## 正确修复路径

如果用户要“真正的视频”，生产链路应改为：

1. 每个阶段先生成关键帧参考图。
2. 每个阶段调用 `video_generate` / image-to-video 生成 5-8 秒动态视频段。
3. 再做成片合成，转场控制在 0.4-0.8 秒或按节奏表设置。
4. 同时生成或接入 VO/BGM/SFX，最后混音进成片。
5. 交付前检查：视频轨不应只来自 HTML 静图渲染，音轨必须存在且可听。

