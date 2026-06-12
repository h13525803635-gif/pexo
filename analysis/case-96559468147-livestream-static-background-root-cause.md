# Case 96559468147: 首次成片参考图未生成直播视频而是静态垫图

**项目 ID**: `96559468147`  
**Langfuse traceName**: `pexo:96559468147`  
**首次 trace 时间**: `2026-06-11T02:28:17.644Z`  
**问题类型**: 首次成片路由错误；参考图被当作静态背景图，而非图生视频输入

## 一句话结论

首次成片没有把参考图中的主播生成成动态直播 footage，而是走了 HyperFrames/HTML 渲染路径：把上传图作为整段 `<img>` 背景垫在底层，再叠加顶部 banner、Flash Sale sticker 和底部跑马灯，最后渲染成 MP4。真正的 `video_generate` 是用户第二轮指出“背景人物也需要正常直播销售 footage”之后才调用的。

## 用户原始需求

用户要求：

- 使用 `01_ecommerce/07_livestream_host.png` 创建 livestream highlight video
- 顶部固定展示 `Mid-Year Mega Sale`
- `00:10` 时中心弹出 `Flash Sale` sticker，并抖动 3 秒
- 底部添加半透明黑色背景层和连续左滚跑马灯文案

这类需求中的主视觉是“直播 highlight video”，上传图应作为人物/场景参考进入图生视频或参考生视频链路；UI 动效只应作为后期 overlay。

## 实际首次链路

首次 trace 中实际工具链为：

```text
get_file_info
  -> write_file(livestream_highlight.html)
  -> render_frame
  -> add_attachments(preview frames)
  -> submit_render
  -> query_render
  -> show_final_video(livestream_highlight_mid_year_sale.mp4)
```

关键点：

- 首次 trace 没有真实 `video_generate` 工具调用。
- HTML composition 中主画面是上传图 `<img>`，并设置为 20 秒整段背景。
- 顶部 banner、Flash Sale sticker、底部 marquee 都由 HTML/CSS/GSAP 完成。
- 最终 MP4 来自 HTML 渲染，不是 AI 视频模型生成的人物直播 footage。

因此用户看到的是“静态主播图片 + 卡片/贴纸/跑马灯动效”，而不是“主播在直播间带货的动态视频”。

## 第二轮发生了什么

用户第二轮反馈：

```text
The character in the background image also needs regular live-stream sales footage.
```

之后 trace-2 才正确调用了 `video_generate`：

```text
video_generate(
  provider = "seedance",
  model = "doubao-seedance-2-0-260128",
  mode = "reference2video",
  image_list = [asset://a_AcYvDAa],
  duration = "10",
  aspect_ratio = "9:16",
  sound = "on"
)
```

输出视频资产为 `asset://a_fvtwyTj`。这说明系统具备正确路径，但首次路由没有进入该路径。

## 根因

### 1. 路由把“视频包装动效”当成了主任务

Agent 优先满足 banner、sticker、marquee 这些确定性的前端动效要求，选择了 HyperFrames/HTML 渲染路径。这个路径适合做 UI overlay、卡片动效、字幕和跑马灯，但不负责把静态人物图生成动态 footage。

### 2. 上传图被当成静态背景素材

上传图只通过 `get_file_info` 获取签名 URL，然后直接写进 HTML `<img>`。它没有进入 `video_generate.image_list`，所以视频模型没有机会把主播人物动起来。

### 3. 缺少“图片做视频”门禁

当用户说 `create ... video using <image>`、`livestream highlight video`、`footage` 时，系统没有强制要求：

```text
uploaded image -> video_generate(image/reference to video) -> overlay/render
```

而是允许：

```text
uploaded image -> HTML <img> background -> render MP4
```

这导致“渲染成 MP4”被误当作“生成视频 footage”。

### 4. 交付前缺少静图主视觉检查

首次交付前没有检查最终视频的主视觉来源。如果最终 composition 的主要画面是 PNG/JPG 且没有成功的 `video_generate`，系统应该阻断交付或至少询问用户是否接受静态幻灯片/海报式视频。

## 解决方案

### 项目级补救

正确重做链路应拆成两步：

```text
1. video_generate
   - 使用上传图 asset://a_AcYvDAa
   - mode: reference2video
   - aspect_ratio: 9:16
   - 生成直播主播正常带货、对镜头介绍商品、自然手势的动态 footage

2. HyperFrames / HTML overlay
   - 背景从 <img> 改为 <video>
   - 顶部固定 Mid-Year Mega Sale
   - 00:10 弹出并抖动 Flash Sale sticker
   - 底部持续左滚跑马灯
   - submit_render / show_final_video
```

也就是：`video_generate` 负责人物和场景动起来，HyperFrames 只负责确定性图层和 UI 动效。

### 系统级修复

#### A. 路由规则

增加硬规则：

```text
当用户请求 create/make/generate video using image，或需求包含 footage、highlight video、livestream、人物/角色需要动起来时：
- 上传图必须进入 video_generate 的 image_list；
- 除非用户明确要求静态海报、幻灯片、卡片动画或纯 UI 动效；
- 禁止仅用 <img> / PNG / JPG 作为整段主视觉渲染交付。
```

#### B. 渲染前校验

在 `submit_render` 或最终交付前增加检查：

```text
if user_intent contains video/footage/livestream/highlight
and main visual is image/png or image/jpeg
and no successful video_generate exists:
    block delivery
    require image-to-video generation first
```

#### C. HTML composition 检查

如果 HTML 中出现整段主场景：

```html
<img src="...uploaded/reference image..." />
```

并且该 `<img>` 覆盖全屏或持续全片，应标记为 `static_main_visual_risk`。除非用户明确接受静态图文视频，否则不能直接交付。

#### D. 分工约束

明确工具边界：

- `video_generate`: 生成动态人物、动态场景、直播 footage、产品/角色运动。
- HyperFrames/HTML: 叠加 banner、贴纸、字幕、跑马灯、卡片、进度条等确定性图层。
- `show_final_video`: 只能交付已满足主视觉动态要求的最终产物。

## 推荐回归用例

输入：

```text
Create a livestream highlight video using [uploaded host image].
Keep a sale banner fixed at the top.
At 00:10 show a Flash Sale sticker.
Add a scrolling bottom marquee.
```

期望：

- 至少一次成功 `video_generate`。
- `image_list` 包含上传图 asset。
- 最终 HTML/时间线主视觉使用生成的 `.mp4`，不是原始 `.png`。
- Banner/sticker/marquee 作为 overlay 叠加在生成视频上。

## 本地证据

本地已拉取 Langfuse 数据：

- `analysis/langfuse-data/cases/96559468147/trace-1-fde91371.json`
- `analysis/langfuse-data/cases/96559468147/trace-2-3c97d1ef.json`
- `analysis/langfuse-data/cases/96559468147/trace-index.json`

注意：原始 trace 可能包含签名 URL 和内部明细，不建议直接提交到公开仓库。
