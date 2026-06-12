# Case 72319856063 — 卡片重叠与无音频音效旁白根因及解决方案

## 结论

本 case 的成片问题来自两个独立缺口：

1. 卡片布局使用固定绝对坐标，缺少碰撞检测、安全区校验和关键时间点截图 QA，导致卡片与卡片、卡片与其它信息层存在重叠风险。
2. 原始执行链路只按静音 HTML 动效执行，没有进入音频生产、音效生成、旁白生成或最终音轨校验链路，因此最终 MP4 没有音频、音效、旁白。

最终交付文件为：

```text
/projects/72319856063/workspace/assets/sector_donut_animation.mp4
```

最终渲染来自 HyperFrames HTML：

```json
{
  "html_file": "asset://a_L3jNTUd",
  "fps": 30,
  "quality": "standard",
  "format": "mp4",
  "output_name": "sector_donut_animation"
}
```

## 用户需求

用户要求使用参考图 `04_news/05_financial_data.png` 的视觉风格，制作一个深色金融数据动效：

- 中心播放增长中的 donut chart 动画。
- 数据为 Tech 26% / Finance 18% / Consumer 15% / Healthcare 12% / Other 29%。
- 00:03 后淡入图表周围的文字框。
- 用打字机效果输出结论：`Tech sector leads the growth, recommend overweighting`。

需求文本没有显式提出 voiceover、narration、sound effect、BGM 或 audio。

## 关键证据

### 1. 执行计划只覆盖视觉合成

trace 中 todo 计划为：

```text
Review the style reference image
List available fonts
Build the HTML composition with donut chart animation + typewriter text
Preview a frame to verify the layout
Render and deliver the final video
```

没有 audio production、music generation、sound design、voiceover scripting、mux audio 或 audio stream probe 步骤。

### 2. 实际工具调用未执行音频生产

结构化 trace 中实际调用的工具包括：

```text
analyze_file_content
list_fonts
write_file / edit_file / read_file
render_frame
submit_render
query_render
show_final_video
```

没有实际调用：

```text
audio_produce
music_generate
```

因此最终视频没有可用音频来源。

### 3. 最终交付直接展示静音 HTML 渲染结果

最终交付调用为：

```json
{
  "file_path": "/projects/72319856063/workspace/assets/sector_donut_animation.mp4"
}
```

在 `show_final_video` 前没有看到对最终 MP4 的 `probe_media`/ffprobe 校验，也没有阻断无音轨交付。

### 4. 卡片布局是硬编码绝对坐标

HTML 中卡片样式如下：

```css
.card {
  position: absolute;
  width: 240px;
  padding: 13px 18px;
  opacity: 0;
}

#c-tech     { left: 148px;  top: 170px; }
#c-finance  { left: 1532px; top: 170px; }
#c-consumer { left: 130px;  top: 660px; }
#c-health   { left: 1532px; top: 660px; }
#c-other    { left: 840px;  top: 56px;  }
```

这类布局没有根据文本实际宽高、边框、padding、图表半径、结论面板高度、legend 位置做统一计算。只要字体渲染、文本换行、容器高度或视觉层位置发生变化，就可能出现重叠。

### 5. 预览覆盖不足

trace 中做了关键帧预览：

```text
render_frame time=1.5
render_frame time=5
```

但卡片是在约 `3.0s` 开始入场，且 stagger 淡入持续到后续时间点。只预览 `1.5s` 和 `5s` 无法覆盖：

- 3.0s 卡片刚出现时的层叠状态。
- 3.3s 多张卡片同时淡入时的碰撞风险。
- 3.5s 结论面板和 typewriter 同步出现后的安全区冲突。

## 根因

### 根因 1：卡片布局没有布局引擎或碰撞检测

卡片是人工写死的 `left/top` 坐标。HTML 没有：

- 卡片 bounding box 计算。
- 卡片与卡片 overlap 检查。
- 卡片与 donut chart / legend / conclusion panel 的 overlap 检查。
- 最小间距约束。
- 安全区边界约束。

### 根因 2：预览 QA 只看首中尾，不看问题发生点

卡片重叠发生在卡片入场阶段和最终信息密集态。实际只采样了 `1.5s` 和 `5s`，没有针对 `3.0s` 后的卡片入场做逐帧或多点检查。

### 根因 3：音频意图没有被推断或确认

用户需求没有明确音频，但最终交付被用户评价为“没有音频音效旁白”。这说明执行链路缺少对视频交付类型的确认：

- 如果是纯数据动效，应明确交付为 silent animation。
- 如果是金融解读视频，应默认补充 VO/BGM/SFX。

本次执行既没有确认静音，也没有生产音频。

### 根因 4：最终交付前缺少 audio gate

非明确静音的视频，在 `show_final_video` 前应检查是否存在 audio stream。本 case 没有看到这类校验，所以静音 MP4 被直接交付。

## 解决方案

### 方案 A：修复为有声金融解读视频

推荐用于返工。

1. 增加旁白脚本，例如：

```text
Technology leads portfolio growth at 26 percent, followed by finance and consumer sectors. With tech momentum still dominant, the recommended stance is to overweight technology while keeping broad sector balance.
```

2. 使用 `audio_produce` 生成 6-8 秒 VO。
3. 可选使用 `music_generate` 生成低音量科技金融 BGM。
4. 在 HTML 中绑定音频，或渲染后 mux 到 MP4。
5. 根据 VO 时长校准动画总时长和 typewriter 节奏。
6. 交付前执行媒体探测，确认最终 MP4 有 audio stream。

### 方案 B：修复为明确静音数据动效

如果产品定义就是静音图表动画：

1. 交付说明中明确标注 silent animation。
2. 不承诺音效、旁白、BGM。
3. 仍需修复卡片布局和多时间点 QA。

### 卡片布局修复

建议改成固定安全区布局，而不是散落坐标：

```css
.card {
  position: absolute;
  width: 240px;
  min-height: 92px;
  box-sizing: border-box;
}

#c-tech     { left: 180px;  top: 190px; }
#c-consumer { left: 180px;  top: 620px; }
#c-finance  { right: 180px; top: 190px; }
#c-health   { right: 180px; top: 620px; }
#c-other    { left: 50%; top: 170px; transform: translateX(-50%); }
```

同时给 chart、legend、conclusion panel 预留不可侵入区域：

```text
chart safe box:      x=690..1230, y=270..760
legend safe box:     x=520..1400, y=850..910
conclusion safe box: x=500..1420, y=940..1040
card minimum gap:    32px
```

更稳的方式是用 JS 在渲染前做 bounding box 检查：

```js
function overlaps(a, b, gap = 24) {
  return !(
    a.right + gap < b.left ||
    b.right + gap < a.left ||
    a.bottom + gap < b.top ||
    b.bottom + gap < a.top
  );
}
```

如果检测到 overlap，自动降级为左右两列列表布局。

## 防复发规则

1. 需求未明确静音时，交付前必须确认是否需要 VO/BGM/SFX。
2. 出现“解读、结论、推荐、新闻、金融播报、narration、voiceover、旁白、音效”等语义时，默认进入音频链路。
3. `show_final_video` 前必须 probe final MP4：
   - 有声需求：必须存在 audio stream。
   - 静音需求：必须在交付说明中写明 silent。
4. 卡片、字幕、标签、信息面板类动效必须做多时间点截图 QA，不只看最终帧。
5. 对所有绝对定位元素执行 bbox overlap 检查，至少保证 24-32px 间距。
6. HTML 资产复用或重渲染前，必须确认使用的是最新修复版本。

## 推荐返工路径

1. 修复卡片布局：引入安全区 + overlap check。
2. 生成一段 6-8 秒英文旁白。
3. 增加轻量 UI tick / whoosh / low-volume finance bed，或至少 VO + BGM。
4. 重新渲染。
5. 采样 `1.5s / 3.0s / 3.3s / 3.6s / 5.0s / 6.5s` 截图检查布局。
6. probe final MP4 确认存在音轨后再交付。

