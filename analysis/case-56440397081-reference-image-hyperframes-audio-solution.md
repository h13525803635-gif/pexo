# Case 56440397081：参考图视频生成误解与音效缺失解决方案

**Langfuse**：`pexo:56440397081`  
**首次 trace 时间**：`2026-06-11T02:31:12.963Z`  
**问题类型**：参考图到视频链路误解；静图动效包装被交付为视频；音效意图缺失  
**本地 trace 数据**：`analysis/langfuse-data/cases/56440397081/trace-{1,2}-*.json`

## 一句话结论

该项目不是“参考图没有生成视频”。首次执行已经用牛排参考图渲染出 `steak_promo_v1.mp4`，但它走的是 HyperFrames/HTML 动效渲染路径：把参考图作为主背景，再叠加蒸汽和优惠券动画。它没有调用 `video_generate(image2video/reference2video)`，因此主视觉本质上是静态牛排图的动效包装；同时原始 prompt 未声明音效/音乐，最终视频默认为静音。

## 用户原始需求

```text
Generate a close-up video of a steak using 01_ecommerce/06_steak_food.png.
Overlay a continuous rising steam effect (screen blend mode) specifically over the food area.
At 00:03, pop up a coupon card in the bottom right corner with an elastic scale animation
reading "$30 Off for New Customers | Valid until June 30".
Add a slight left-right rocking motion to the card to grab attention.
<original-image>/projects/56440397081/workspace/assets/a_MAkBUMx_08_06_steak_food.png</original-image>
```

用户后续追问：

```text
Why is there no sound effects?
```

## 实际调用链路

### Trace 1：视频已成功生成

```text
用户 prompt + 牛排参考图
  -> analyze_file_content：分析参考图构图和牛排区域
  -> get_file_info：获取参考图 signed_url
  -> write_file：写入 /projects/56440397081/workspace/steak_promo.html
       <img id="bg-img" src="参考图 signed_url">
       CSS/GSAP 蒸汽动效
       CSS/GSAP 优惠券弹出和摇摆
  -> submit_render(html_file=asset://a_ghDB5QN, output_name=steak_promo_v1)
  -> query_render：done，输出 asset_id=a_GTzeMav
  -> render_frame：成功生成预览帧
  -> show_final_video：/projects/56440397081/workspace/assets/steak_promo_v1.mp4
```

关键证据：

| 节点 | 结果 |
|------|------|
| 参考图 asset | `a_MAkBUMx` / `08_06_steak_food.png` |
| HTML composition | `/projects/56440397081/workspace/steak_promo.html` |
| render job | `3d1pbbi0` |
| 输出视频 asset | `a_GTzeMav` |
| 输出文件 | `steak_promo_v1.mp4` |
| `show_final_video` | 成功 |

### Trace 2：用户询问无音效

第二轮用户问 `Why is there no sound effects?`。Agent 回复原 brief 没有要求 audio，因此默认静音，并建议添加 sizzling/crackling、soft restaurant ambience、subtle whoosh/pop 等声音方向。

这说明后续问题不是“没出视频”，而是“视频没有声音/音效”。

## 容易误判的点

### 1. 生成了 MP4，但不是图生视频模型生成

首次链路没有 `video_generate`。视频来自 HyperFrames 渲染：

```text
上传参考图 -> HTML <img> 主背景 -> GSAP 蒸汽/卡片动画 -> submit_render -> MP4
```

这类产物确实是视频文件，但主视觉不是 AI 视频模型把牛排画面动态化，而是静态图 + 确定性 overlay 动画。

### 2. 中途有资产解析错误，但不是生成失败

渲染完成后，Agent 做媒体校验时曾错误调用：

```text
probe_media(file="asset://steak_promo_v1")
```

该值不是合法 asset id。合法 asset id 必须类似 `asset://a_GTzeMav`，因此返回：

```text
hyperframes.asset_resolve_failed
invalid id type prefix: expected 'a', got 's'
```

这个错误发生在 probe/校验阶段。真实视频已经由 `submit_render` 成功生成，并通过 `show_final_video` 展示。

### 3. 没有音效是音频意图缺失

原始请求只要求画面、蒸汽、优惠券动画和摇摆动作，没有明确要求 sound effect、BGM、VO 或 ambience。Agent 没有先做 audio intent 判断，也没有主动生成音效轨，导致交付的是静音 MP4。

## 根因

| 层级 | 说明 |
|------|------|
| 路由 | Agent 将需求理解为“参考图 + UI/动效包装”，优先走 HyperFrames，而不是 `video_generate(image2video)` |
| 主视觉 | 参考图只进入 HTML `<img>`，没有进入视频模型的 `image_list` |
| 用户预期 | 用户说 `Generate ... video using image`，可能期待图生视频；系统把“渲染成 MP4”视作满足视频需求 |
| 音频 | 没有 audio intent gate，原 prompt 未要求声音时直接静音交付 |
| 校验 | probe 阶段错误使用 `asset://steak_promo_v1`，应使用真实 asset id 或 signed URL |

## 项目级补救方案

如果用户期望“牛排画面本身也动起来”，应重做为两阶段：

```text
1. video_generate
   - mode: image2video 或 reference2video
   - image_list: [asset://a_MAkBUMx]
   - 目标：牛排热气、肉汁微动、镜头轻推近、食物质感动态
   - duration: 6-8s
   - aspect_ratio: 9:16 或按交付场景选择

2. HyperFrames overlay
   - 背景从 <img> 改为生成的 <video>
   - 保留连续 rising steam overlay
   - 00:03 弹出 "$30 Off" coupon card
   - 添加卡片轻微左右摇摆
   - 加入 sizzling/crackling sound effect 或轻餐厅 ambience
   - submit_render / show_final_video
```

如果用户只需要“电商广告式动效包装”，首次结果路径可以接受，但应在交付时明确说明：

```text
已基于参考图制作静态主图动效视频，包含蒸汽和优惠券动画；未做 AI 图生视频主体运动。
```

## 系统级解决方案

### A. 增加主视觉动态意图判断

当用户说：

```text
generate/create/make video using image
close-up video using <original-image>
product video / food video / footage
```

系统应判断用户是否期待主视觉动态化：

- 若期待动态 footage：必须调用 `video_generate`，并把上传图放入 `image_list`。
- 若只期待卡片、贴纸、文字、蒸汽等 overlay 动效：可以走 HyperFrames，但需标注主图是静态底图。

### B. 交付前增加静态主视觉拦截

```text
if user_intent contains video/footage/product video
and main visual is an uploaded PNG/JPG covering most of the timeline
and no successful video_generate exists:
    require explicit confirmation or run image2video first
```

### C. 音频意图 gate

在进入 HyperFrames 或 assembly 前增加 audio intent：

```text
if prompt contains food/steam/sizzle/pop/coupon/ad
and user did not explicitly request silent:
    propose or auto-add lightweight SFX/BGM
```

推荐本案默认音效：

- 低音量 `sizzling/crackling steak` 环境音贯穿全片
- `00:03` 优惠券弹出时添加短 `pop/whoosh`
- 背景可选轻餐厅 ambience 或低强度广告 BGM

### D. 资产引用校验

修复 probe/render 后校验规范：

```text
错误：asset://steak_promo_v1
正确：asset://a_GTzeMav
或：使用 get_file_info 返回的 signed_url
```

规则：

- `asset://` 后面只能是真实 `a_` 前缀 asset id。
- 文件名不能伪造成 asset id。
- `probe_media` 失败时不能误判为视频未生成，应回查 `query_render` 和 `show_final_video` 结果。

## 推荐回归用例

输入：

```text
Generate a close-up product video using an uploaded steak image.
Add rising steam over the food.
At 00:03 show a coupon card and make it rock slightly.
Add realistic sizzling sound effects.
```

期望：

- 至少一次成功 `video_generate`。
- `image_list` 包含上传图 asset。
- HyperFrames 主视觉使用生成的 `.mp4`，不是原始 `.png`。
- HTML overlay 包含蒸汽、优惠券弹出和摇摆。
- 输出视频 probe 有 audio stream。
- `show_final_video` 交付最终 MP4。

## 修订记录

| 日期 | 说明 |
|------|------|
| 2026-06-12 | 初版：真实链路、误判点、补救方案与系统级修复 |
