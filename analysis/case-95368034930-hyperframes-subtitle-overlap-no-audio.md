# Case 95368034930 — 开头字幕叠加乱码与无旁白音频根因

## 结论

本 case 的问题不是导出阶段丢音频，而是最终成片从一开始就是 **HyperFrames HTML 静音渲染链路**：画面、信息卡和底部 narration subtitle 都写进 HTML/GSAP 时间线，但没有先生成 TTS 旁白，也没有在最终 HTML 中绑定 `<audio>` 音轨。

开头下方多层字幕叠加的直接原因是：HTML 中 `sub-0` 到 `sub-9` 十条字幕都放在同一个底部绝对定位区域，默认 CSS 没有把它们隐藏；隐藏依赖 GSAP 在时间线 `0s` 执行 `opacity: 0`。当首帧/开头几帧捕获早于或未正确应用该时间线状态时，所有字幕会同时显示在同一位置，视觉上形成多层叠加乱码。

## 用户需求

用户要求创建一个板块运动科普视频：

- 从 Pangaea 到现代大陆漂移，慢速动画展示大陆分离。
- 每个阶段加入滑入信息卡，解释对应地质年代。
- 底部加入 scrolling subtitle narration throughout the video。

其中 “subtitle narration” 被执行成了可见底部字幕，而没有被落实成可听的旁白音频。

## 现象

1. 视频开头底部出现多条字幕叠在一起，读起来像乱码。
2. 全片只有底部字幕，没有旁白。
3. 成片没有任何可听音频。

## 关键证据

### 1. 最终交付来自 HyperFrames HTML

最终交付文件为：

```text
/projects/95368034930/workspace/assets/tectonic-plates-final.mp4
```

对应的最终渲染调用使用同一个 HTML 资产：

```json
{
  "html_file": "asset://a_X9k1upn",
  "output_name": "tectonic-plates-final",
  "quality": "standard"
}
```

渲染完成后通过 `show_final_video` 展示：

```text
/projects/95368034930/workspace/assets/tectonic-plates-final.mp4
```

### 2. 最终 HTML 中只有字幕 DOM，没有音频元素

HTML 中底部字幕层结构如下：

```html
<div id="subtitle-pip" class="clip"
  data-start="0" data-duration="68" data-track-index="1"
  style="position:absolute;bottom:0;left:0;right:0;height:110px;z-index:30;pointer-events:none;overflow:hidden;">
  <div id="sub-0">Welcome to a journey through deep time...</div>
  <div id="sub-1">250 million years ago...</div>
  ...
  <div id="sub-9">In another 250 million years...</div>
</div>
```

这些字幕都使用相同的底部绝对定位：

```css
position:absolute;
bottom:32px;
left:0;
right:0;
text-align:center;
```

但最终 HTML 没有看到任何：

```html
<audio src="...">
```

因此最终 MP4 没有旁白或 BGM 来源。

### 3. 字幕隐藏依赖 GSAP，而不是初始 CSS

字幕初始隐藏逻辑写在时间线里：

```js
tl.set("#sub-1,#sub-2,#sub-3,#sub-4,#sub-5,#sub-6,#sub-7,#sub-8,#sub-9", { opacity: 0 }, 0);
tl.set("#sub-0", { opacity: 0 }, 0);
tl.to("#sub-0", { opacity: 1, duration: 0.6, ease: "power2.out" }, 1.0);
```

问题在于 HTML/CSS 初始状态没有兜底隐藏。只要首帧渲染时 GSAP `tl.set(..., 0)` 没有先于截图状态生效，所有 `sub-*` 都会按默认可见状态叠在一起。

### 4. 音频生产没有进入最终交付链路

trace 中后段确实出现过 `audio_produce` / `music_generate` 工具 schema，但它们出现在最终 `tectonic-plates-final.mp4` 渲染完成记录之后；最终交付仍是从 `asset://a_X9k1upn` 渲染出来，且该 HTML 没有绑定音频。

换句话说，即使后续有尝试补音频，也没有重新写入最终 HTML、没有重新 mux 到交付文件，也没有在交付前做 audio stream 校验。

## 根因

### 根因 1：把 narration 误降级为纯字幕

用户的 “scrolling subtitle narration” 语义上包含 narration，但执行时只做了底部字幕条，没有生成 VO TTS，也没有进行语音与字幕对齐。

### 根因 2：HyperFrames 音频门禁未生效

已读到的 HyperFrames skill 规则要求：

- 非 `explicit_silent` 意图必须先有 `vo_*` / `bgm_*` 资产。
- HTML 必须包含 `<audio src="https://...">`。
- 最终交付前必须 probe 成片确认存在 audio stream。

本 case 违反了这些门禁：没有音频资产绑定、没有 `<audio>`、也没有看到交付前阻断。

### 根因 3：字幕初始状态只靠时间线，不靠 CSS

所有字幕 DOM 在 HTML 中同时存在、同位置叠放；只有 GSAP 时间线负责把非当前字幕设为透明。首帧捕获或时间线 seek 状态异常时，没有 CSS 兜底，导致多字幕同屏。

### 根因 4：最终渲染资产反复复用，但没有重新注入音频

多次 `submit_render` 都指向 `asset://a_X9k1upn`。该资产代表的是无音频 HTML，所以无论渲染多少次，结果仍然是静音版本。

## 解决方案

### 方案 A：做成真正的有声旁白科普视频

适用于本 case，推荐采用。

1. 明确 `audio_intent = voiceover_required`，不要把 narration 当成纯字幕。
2. 用 `audio_produce` 生成完整 TTS 旁白。
3. 获取 TTS alignment/subtitle 时间戳，生成 SRT/VTT 或 cue JSON。
4. 用 `get_file_info` 获取音频 signed URL。
5. 在 HTML body 中加入：

```html
<audio src="https://..." data-start="0" data-duration="68"></audio>
```

6. 字幕由真实 TTS cue 驱动，而不是手写固定秒数猜测。
7. 重新渲染最终 MP4。
8. 交付前 probe：必须存在 audio stream，且音频时长覆盖主体视频。

### 方案 B：若明确要静音，则取消 narration 语义

如果用户明确要 silent visual version：

1. 不使用 “narration” 文案描述。
2. 底部字幕改成静态说明或阶段标题。
3. 交付说明中明确这是静音视频。
4. 不使用逐句/滚动 narration 效果，避免用户期待有声音。

### 字幕叠加修复

无论是否补音频，字幕初始状态都应由 CSS 兜底隐藏，只让当前 cue 显示：

```css
.subtitle-line {
  opacity: 0;
  visibility: hidden;
}
.subtitle-line.is-active {
  opacity: 1;
  visibility: visible;
}
```

或在 inline style 中直接写入初始隐藏：

```html
<div id="sub-0" class="subtitle-line" style="opacity:0;visibility:hidden;">...</div>
```

时间线中再显式切换：

```js
tl.set(".subtitle-line", { opacity: 0, visibility: "hidden" }, 0);
tl.set("#sub-0", { opacity: 1, visibility: "visible" }, 1.0);
tl.set("#sub-0", { opacity: 0, visibility: "hidden" }, 8.2);
tl.set("#sub-1", { opacity: 1, visibility: "visible" }, 8.8);
```

更稳的方式是只保留一个字幕容器，根据 cue 更新文本内容，避免多个同位置 DOM 同时存在：

```html
<div id="subtitle-current" class="subtitle-line"></div>
```

## 防复发规则

1. **音频意图门禁**：请求中出现 narration、voiceover、旁白、朗读、讲解、subtitle narration 等词时，默认不是静音需求。
2. **HyperFrames 提交前检查**：非静音视频在 `submit_render` 前必须检查 HTML 是否包含 `<audio src="https://...">`。
3. **字幕初始隐藏**：所有 subtitle cue DOM 必须在 CSS/inline style 层面默认隐藏，不能只靠 GSAP 首帧 `tl.set`。
4. **最终媒体校验**：`show_final_video` 前必须 probe final MP4，确认 audio stream 存在；若无音轨且非 explicit silent，阻断交付。
5. **资产复用检查**：如果最终渲染复用旧 HTML asset，必须确认该 asset 是最新版本，且包含音频绑定和字幕修复。

## 推荐返工

本 case 推荐按方案 A 返工：生成完整英文 TTS 旁白，用 TTS alignment 生成字幕 cue，把 VO 音轨和 cue 一起绑定进 HyperFrames HTML 后重渲染。这样可以同时解决“只有字幕无音频”和“字幕与旁白不同步/伪同步”的问题。

快速修复则至少应先做两件事：

1. 给所有 `sub-*` 初始加 `opacity:0; visibility:hidden`，避免首帧多字幕叠加。
2. 若暂不补音频，将交付标注为 silent visual version，并把 “narration” 字样改为 “bottom captions”。
