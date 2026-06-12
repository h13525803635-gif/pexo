# Case 24354459773 — 无音频下逐词字幕异常根因与解决方案

## 结论

本 case 的问题不是“音频在导出时丢失”，而是**制作链路从一开始就没有生成或绑定可听音轨**；同时字幕仍按预设时间轴执行了逐词 karaoke 高亮，因此形成了“无声视频里字幕一个词一个词变化/观感像变小”的异常体验。

## 用户需求

用户要求：

- 将 `02_education/03_vocab_flashcard.png` 放在纯色背景中作为词卡正面。
- 3 秒处做 3D Y 轴翻转，背面展示 `Elephant /ˈelɪfənt/ n.`。
- 右上角保留圆形 PiP 嘴型视频，展示发音。
- 底部添加例句字幕：`The elephant is the largest land animal.`
- 字幕使用 karaoke-style word-by-word highlight effect。

## 现象

1. 成片没有音频。
2. 底部字幕仍然按词逐个变化。
3. 因为当前词、已读词、未读词颜色/发光状态不同，用户观感上会觉得“一个单词变小一个”或字号/粗细不一致。

## 根因

### 1. 音频没有生成或绑定

trace 中 mouth PiP 的视频生成参数明确设置了：

```json
"sound": "off"
```

后续 HTML 中嵌入 PiP 视频时又写了：

```html
<video src="..." muted playsinline loop></video>
```

同时没有看到 `audio_produce`、`text_to_music` / `music_generate` 等音频生产调用，最终 HTML 中也没有 `<audio src="...">`。因此最终无声是编排阶段的结果，不是渲染器把音频丢掉。

### 2. 字幕动画与音频完全解耦

字幕不是由真实音频时间戳驱动，而是在 HTML/GSAP 时间轴里硬编码了逐词状态切换：

```html
<span class="k-word" id="kw-0">The&nbsp;</span>
<span class="k-word" id="kw-1">elephant&nbsp;</span>
...
```

并在脚本中按固定秒数切换 active/done：

```js
tl.set("#kw-0", { className: "+=active" }, 4.5);
tl.set("#kw-0", { className: "-=active" }, 5.0);
tl.set("#kw-1", { className: "+=active" }, 5.0);
```

这意味着即使没有任何音频，视频时间轴照常播放，字幕也会继续逐词变化。

### 3. karaoke 效果在无声场景下是不合理降级

用户要求里确实包含 word-by-word highlight，但该效果天然暗示“正在朗读/演唱/发音”。当系统没有生成 TTS、没有保留 PiP 原声、也没有绑定音轨时，继续展示逐词 karaoke 会造成错误预期：观众会以为字幕跟着声音走，但实际没有声音。

## 为什么会看起来像“一个词变小”

代码里 `.k-word` 的基础字号是统一的 `52px`，但不同状态使用了不同颜色和发光：

```css
.k-word {
  font-size: 52px;
  color: rgba(255,255,255,0.45);
}
.k-word.active {
  color: #FEFCBF;
  text-shadow: 0 0 18px rgba(246,224,94,0.9);
}
.k-word.done {
  color: rgba(255,255,255,0.8);
}
```

因此严格说不一定是实际 `font-size` 改了，而是逐词高亮造成视觉权重变化：当前词更亮、更厚、更有光晕，其他词更淡，用户会感知为大小或层级不一致。

## 解决方案

### 方案 A：保留 karaoke，但必须补音频

适用于用户确实想要“发音/朗读 + 逐词高亮”的教育视频。

应执行：

1. 生成例句 TTS：`The elephant is the largest land animal.`
2. 获取 TTS 的 subtitle/alignment 时间戳，避免手写猜测时间。
3. 在 HyperFrames HTML 中显式加入 `<audio src="https://...">`。
4. PiP mouth 视频如果需要发音声，不应使用 `sound: off`，也不应在 HTML 中 `muted`。
5. 渲染前检查：非静音需求下 HTML 必须包含至少一条 `<audio src="https://...">`。
6. 最终交付前 probe 成片，确认存在 audio stream。

### 方案 B：无音频版本取消逐词 karaoke

适用于用户只要视觉词卡，不需要听觉朗读。

应执行：

1. 去掉逐词 active/done 时间轴。
2. 字幕改为整句静态显示或轻微淡入。
3. 可保留 mouth PiP 作为视觉发音提示，但不要让字幕表现得像在跟读。
4. 若明确静音，应在交付说明中标注这是 silent visual version。

推荐静态字幕样式：

```html
<div id="subtitle-bar" class="clip" data-start="4.5" data-duration="5.5" data-track-index="2">
  <div id="sentence-caption">The elephant is the largest land animal.</div>
</div>
```

只做整体淡入：

```js
tl.from("#subtitle-bar", {
  y: 24,
  opacity: 0,
  duration: 0.35,
  ease: "power2.out"
}, 4.5);
```

## 防复发规则

1. **音频意图门禁**：若需求包含 pronounce、read、voice、karaoke、sing、旁白、朗读、发音等词，默认不是静音需求，必须先明确音频来源。
2. **karaoke 门禁**：没有真实音频或 alignment 时间戳时，不应交付逐词 karaoke，只能交付静态/整句字幕。
3. **HyperFrames 音频检查**：非 explicit silent 的 HTML composition，在 `submit_render` 前必须包含 `<audio src="https://...">`。
4. **mouth PiP 检查**：嘴型 PiP 如果承担“发音”语义，不能同时 `sound: off` 和 `muted`，除非明确做静音视觉演示。
5. **交付前媒体校验**：最终 mp4 必须 probe，确认有 audio stream；无音轨且非用户明确静音时阻断交付。

## 修复建议

对本 case 最合理的返工方式是方案 A：补一条例句 TTS，并用 TTS alignment 驱动字幕。因为原始需求包含“mouth pronouncing the word”和“karaoke-style word-by-word highlight”，这两个信号都指向有声发音教学视频。

如果只做快速修复，则采用方案 B：移除逐词变化，改成整句静态字幕，避免无声状态下出现伪同步字幕。
