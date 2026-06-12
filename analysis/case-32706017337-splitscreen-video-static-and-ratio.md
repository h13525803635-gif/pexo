# Case 32706017337: 首轮静图视频与二轮左右比例不一致

**项目 ID**: `32706017337`  
**Langfuse traceName**: `pexo:32706017337`  
**时间**: 2026-06-11  
**问题类型**: 首轮视频主视觉为静态图片；二轮图生视频后左右人物 framing 不一致

## 一句话结论

首轮并不是没有渲染 MP4，而是把两张静态图放进 HyperFrames/HTML 里做成了“静图动效视频”：左侧是新生成的 Before 图片，右侧是用户上传的 After 图片，全程没有真实 `video_generate`。第二轮才生成了左右两个真实视频，但两个视频是独立 reference-to-video 任务，人物大小、站位和镜头距离没有被锁定；再用同样的 `object-fit: cover` 塞进左右半屏后，视觉比例就不一致。

## 用户原始需求

用户要求：

```text
Use 05_social/05_outfit_ootd.png as the 'After' outfit on the right,
and generate a casual outfit for the 'Before' on the left,
creating a 1:1 split-screen video.
Overlay a vertical dividing line with a lightning flash effect
(using screen blend mode) in the middle.
At the bottom center, overlay a product link card saying
'Get this blazer for $299 - Click to buy'
with a continuous breathing micro-animation throughout the video.
```

这里的关键是用户明确要求 `split-screen video`，但首轮只做了图片面板的 HTML 渲染。

## Trace 数据

本地抓取结果：

```text
analysis/langfuse-data/cases/32706017337/trace-1-1c32e7e9.json
analysis/langfuse-data/cases/32706017337/trace-3-415eb972.json
analysis/langfuse-data/cases/32706017337/trace-index.json
```

Langfuse 成功返回 2 条 trace，另有 1 条中间 trace 详情接口返回 500：

| idx | trace_id | start_time | 状态 |
|---|---|---:|---|
| 1 | `1c32e7e9870f73642503ddc674e7e9b3` | `2026-06-11T02:28:46.603Z` | 成功抓取 |
| 2 | `37879e56f2a46a976b5234837cd05755` | `2026-06-11T02:47:38.258Z` | Langfuse API 500 |
| 3 | `415eb9724a610c310c48747c099524e6` | `2026-06-11T06:14:20.115Z` | 成功抓取 |

## 首轮真实链路

trace-1 的实际调用链：

```text
analyze_file_content(uploaded After image)
  -> image_generate(Before casual outfit, 9:16)
  -> get_file_info(Before image + After image)
  -> write_file(outfit_splitscreen.html)
       <img> Before
       <img> After
       CSS/GSAP divider, lightning flash, product card breathing
  -> render_frame(preview)
  -> submit_render(outfit_splitscreen_final)
  -> query_render(done)
  -> show_final_video -> asset a_u5Qbn9B
```

关键证据：

- `image_generate` 生成了 `before_casual_outfit_20260611T022938_7b46dc71.png`，asset `a_ZKXTefJ`。
- 首轮 HTML 中左右面板使用 `<img>`，不是 `<video>`。
- 首轮没有成功的 `video_generate` 调用。
- 最终确实渲染了 `outfit_splitscreen_final.mp4`，asset `a_u5Qbn9B`，但它是“静态图面板 + 动效 overlay”的 MP4。

## 为什么用户感知为“只有图没有视频”

首轮交付的是 MP4 容器，但主视觉没有真实视频素材。左右服装人物都是静图，只有分割线、闪电和商品卡片在动。

这类产物从技术上是视频文件，但从用户语义上不是“图生视频”或“人物动起来的视频”。根因是路由把 `video` 理解成“把 HTML 动效渲染成 MP4”，而不是“先生成动态 footage，再做 overlay”。

## 问题根因

### 根因 1：首轮路由把“生成视频”误判成“HTML 动效渲染成 MP4”

用户明确说 `creating a 1:1 split-screen video`，并且要求 Before/After 服装对比。正确语义应该是：

```text
参考图/生成图 -> video_generate 生成动态人物 footage -> HyperFrames 叠加 divider/card 动效 -> render MP4
```

但首轮实际走成了：

```text
参考图/生成图 -> <img> 静态面板 -> HyperFrames 叠加 divider/card 动效 -> render MP4
```

也就是说系统只满足了“MP4 文件”和“overlay 动效”，没有满足“主视觉是动态视频”。

### 根因 2：工具选择缺少 `video_generate`

首轮只调用了 `image_generate` 来生成 Before 图片，然后把 Before 图片和 After 上传图写入 HTML。没有把两张图作为 `image2video` 或 `reference2video` 的输入。

因此首轮最终文件虽然是 `.mp4`，但主画面本质仍是两张静图。这个问题属于“视频容器成功，视频语义失败”。

### 根因 3：交付前缺少静态主视觉拦截

在 `submit_render` / `show_final_video` 前，系统没有检查：

```text
用户是否要求 video
主视觉是否只有 <img>
是否存在成功的 video_generate
```

如果这个校验存在，首轮应该被拦截，并要求先补 `video_generate`，而不是直接交付 `outfit_splitscreen_final.mp4`。

### 根因 4：第二轮视频生成没有锁定左右构图

第二轮虽然补了 `video_generate`，但 Before 和 After 是两个独立任务。两个 prompt 没有强制约束：

- same camera distance
- same body scale
- same subject bbox height
- same head/feet safe margins
- no zoom-in / no crop drift

视频模型因此各自决定人物大小和镜头距离，导致两个 `960x960` 视频内部的主体比例不同。

### 根因 5：最终合成依赖 `object-fit: cover` 放大了差异

v2 合成把两个 `960x960` 正方形视频放进 `540x1080` 的竖长半屏，并使用 `object-fit: cover`。这个 CSS 会用裁切来填满容器。

当两个输入视频本身的人物位置和大小不一致时，同样的 `cover` 策略不会把人物比例统一，反而会把 framing 差异更明显地暴露出来。

### 根因 6：缺少 split-screen 前的归一化步骤

Before/After 对比类视频需要在合成前做统一规格处理：

```text
detect subject bbox -> normalize scale/crop/pad -> exact half-screen video -> final split-screen
```

本案直接把两个独立生成的视频塞进左右面板，没有做人物 bbox 对齐、统一缩放和安全区裁切，所以最终左右视觉比例不一致。

## 第二轮真实链路

trace-3 的 session summary 记录：用户第二轮要求把静态图片面板换成 live animated video panels。

实际生成的视频资产：

| 侧边 | 工具 | 文件 | asset | 探测信息 |
|---|---|---|---|---|
| Before | `video_generate(reference2video)` | `before_outfit_video_20260611T025546_d60cf806.mp4` | `a_RrYtB49` | 约 6.04s, 960x960, silent |
| After | `video_generate(reference2video)` | `after_outfit_video_20260611T025501_8d8b1392.mp4` | `a_KTQY2Aq` | 约 6.04s, 960x960, silent |

v2 HTML 的关键布局：

```css
#vid-before {
  position: absolute;
  top: 0; left: 0;
  width: 540px; height: 1080px;
  object-fit: cover;
  object-position: center center;
}

#vid-after {
  position: absolute;
  top: 0; left: 540px;
  width: 540px; height: 1080px;
  object-fit: cover;
  object-position: center center;
}
```

最终渲染：

```text
submit_render(outfit_splitscreen_v2_final)
  -> query_render(done)
  -> show_final_video -> asset a_ia5UzBf
```

## 为什么二轮左右比例不一致

HTML 容器本身左右是对称的：

- 总画布：`1080x1080`
- 左半屏：`540x1080`
- 右半屏：`540x1080`
- 两侧都是 `object-fit: cover`

因此比例不一致不是左右容器宽高写错，而是输入视频的视觉 framing 不一致：

1. Before 和 After 是两个独立的视频生成任务。
2. prompt 只描述了服装、动作和环境，没有硬性约束人物 bbox、头顶/脚底位置、镜头距离和身体占画面比例。
3. 两个输出视频虽然都是 `960x960`，但人物在画面中的大小、站位和裁切边界不同。
4. 正方形视频被 `object-fit: cover` 放进 `540x1080` 竖长半屏时会被水平/垂直裁切，原本的 framing 差异被放大。

## 项目级补救方案

最快补救不需要重跑整条创作链，直接使用现有两个视频资产重做标准化和合成：

```text
1. 输入现有视频
   - Before: asset://a_RrYtB49
   - After:  asset://a_KTQY2Aq

2. 对两个视频做构图归一化
   - 输出目标：540x1080
   - 人物中心线对齐
   - 人物 bbox 高度统一，例如占画面高度 82%
   - 头顶、脚底安全边距统一
   - 必要时对一侧做 scale/crop/pad

3. 重新合成 split-screen v3
   - 左右直接使用已标准化的 540x1080 视频
   - CSS 不再依赖 object-fit: cover 做主裁切
   - 保留 divider、lightning flash、product card breathing overlay

4. 交付 outfit_splitscreen_v3_final.mp4
```

如果没有自动人体 bbox 检测能力，短期可用人工参数补救：预览第 0s、3s、6s 三帧，按人物高度较小的一侧统一缩放另一侧，并固定 `object-position`。

## 系统级修复方案

### 1. 视频意图路由门禁

当用户明确要求 `video`、`footage`、`live animated video panels`、人物/产品需要动起来时：

```text
必须至少有一次成功 video_generate；
最终主视觉不能只有 <img> / PNG / JPG；
除非用户明确要求 slideshow/static poster animation。
```

如果只做静图 HTML 渲染，交付文案必须明确说明这是“静态图动效视频”，不能说成真实 animated footage。

### 2. HTML 交付前校验

在 `submit_render` 或 `show_final_video` 前检查：

```text
if user_intent contains video
and main visual tags are <img>
and no successful video_generate exists:
    block delivery
    require image-to-video/reference-to-video first
```

### 3. 双栏视频一致性校验

对于 split-screen before/after、comparison、左右对比类视频，增加一条构图一致性检查：

```text
if layout is split-screen
and both sides are independently generated videos:
    require same aspect ratio
    require same target crop size
    require subject bbox height delta <= 5%
    require subject center x/y delta <= threshold
```

不满足时先做 `scale + crop + pad` 归一化，再进入最终合成。

### 4. 生成 prompt 强约束

两个 `video_generate` prompt 应共享构图约束：

```text
same camera distance
same full-body framing
same mirror selfie pose
subject centered
subject occupies exactly 82% of frame height
head top and shoes bottom visible
no zoom-in
no camera crop change
no change in body scale
```

### 5. 推荐标准链路

```text
uploaded After image
  -> generate Before reference image
  -> normalize Before/After reference images to same 9:16 composition
  -> video_generate Before with locked framing
  -> video_generate After with locked framing
  -> normalize generated videos to exact 540x1080
  -> HyperFrames overlay split-screen UI
  -> submit_render
  -> show_final_video
```

## 回归用例

输入：

```text
Use uploaded outfit image as After on the right.
Generate casual Before on the left.
Create a 1:1 split-screen video.
Both people should be animated and have the same body scale.
```

期望：

- 首轮就有成功 `video_generate`。
- 最终 HTML 主视觉包含 `<video>`，不是只有 `<img>`。
- 左右视频进入最终合成前已归一化到相同半屏尺寸。
- 人物 bbox 高度差不超过 5%。
- 交付文案不把静态图动效包装成真实视频 footage。

## 修订记录

| 日期 | 说明 |
|---|---|
| 2026-06-12 | 初版：Langfuse 链路分析、根因和修复方案 |
