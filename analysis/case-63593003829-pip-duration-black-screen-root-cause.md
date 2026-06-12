# Case 63593003829：画中画时长误判与尾部黑屏问题

## 结论摘要

项目 `63593003829` 的原始需求是制作一条运动鞋产品展示视频：

- 主画面：鞋子的 close-up rotating shot。
- 画中画：右下角展示穿着这双鞋跑步的画面。
- 叠加信息：从左侧滑入产品参数卡，包含 `280g`、`Cushion Tech`、`Breathable Mesh`。

用户反馈了两个问题：

1. 首次成片看起来只给了一张图。
2. 用户追问后拿到的视频，后面有很长一段黑屏。

根因是：**成片时长规划错误 + 交付前视觉 QA 不足**。

核心问题可以概括为：

> 系统生成了两段 8 秒视频，但误把它们当成可以顺序相加的时长素材，即近似 `8s + 8s`。实际需求是画中画，主画面和 PiP 画面应该并行播放，共享同一段时间轴。最终 HyperFrames 被渲染成 18 秒，但可见视频素材本身只覆盖约 8 秒；如果 HTML 没有显式设置循环、冻结尾帧或兜底产品图，素材播完后就会露出黑底，形成尾部长黑屏。

## Langfuse 证据

本地导出路径：

- `analysis/langfuse-data/cases/63593003829/trace-2-fb3a282a.json`
- `analysis/langfuse-data/cases/63593003829/trace-index.json`

重新拉取后的 trace 索引：

```json
{
  "fetched": [
    {
      "idx": 2,
      "trace_id": "fb3a282a12766747788078f5b392a883",
      "start_time": "2026-06-11T06:14:29.960Z",
      "file": "analysis/langfuse-data/cases/63593003829/trace-2-fb3a282a.json"
    }
  ],
  "failed": [
    {
      "idx": 1,
      "trace_id": "c94fc20eda733f5aecd5bcf7c0968220",
      "start_time": "2026-06-11T02:27:16.059Z",
      "error": "GET failed after 5 attempts: HTTP Error 500: Internal Server Error"
    }
  ]
}
```

第一条 trace 很可能对应首次交付，但 Langfuse trace detail 接口连续返回 HTTP 500，无法取回完整详情。第二条 trace 是用户追问后的链路，用户消息是：

```text
Please deliver the video.
```

第二条 trace 中恢复出的原始视频需求是：

```text
Generate a product showcase video using the sneaker image. Start with a close-up rotating shot of the shoe, then add a picture-in-picture window in the bottom-right showing the shoe being worn while running. Overlay a product spec card sliding in from the left with key features: weight 280g, cushion tech, breathable mesh.
```

## 调用时间线

以下时间均为 `2026-06-11` UTC。

| 时间 | 事件 | 含义 |
|---|---|---|
| `06:14:29` | 用户发送 `Please deliver the video.` | 用户在首次交付后再次要求交付视频。 |
| `06:14:34` | `query_render(job_id="0x7otdim")` | Agent 尝试恢复上一轮旧渲染任务。 |
| `06:14:34` | 返回 `hyperframes.job_not_found` | 旧渲染任务不存在或已失效。 |
| `06:14:42` | `submit_render(html_file="/projects/63593003829/workspace/sneaker_showcase.html", fps=30, output_name="sneaker_product_showcase")` | Agent 重新提交 HTML 合成渲染。 |
| `06:22:52` | render status `done` | 新渲染任务完成。 |
| `06:23:01` | `probe_media(asset://a_D3UAoia)` | Agent 只检查了媒体容器元数据。 |
| `06:23:15` | `show_final_video("/projects/63593003829/workspace/assets/sneaker_product_showcase.mp4")` | 最终视频被交付。 |

最终视频的 `probe_media` 元数据：

```text
file: asset://a_D3UAoia
duration: 18.000000 seconds video, 18.048000 seconds audio
resolution: 1920x1080
fps: 30
codec: H.264 + AAC stereo
```

本地缓存中的生成素材：

- `sneaker_rotating_closeup_20260611T023503_c04bb093.mp4`
- `sneaker_running_pip_20260611T023442_564acae9.mp4`
- `sneaker_bgm_20260611T023523_75c34f08.mp3`
- `sneaker_product_showcase.mp4`

## 为什么首次成片像是只给了一张图

第二条可见 trace 显示，Agent 一开始尝试查询上一轮旧渲染任务：

```text
query_render(job_id="0x7otdim") -> hyperframes.job_not_found
```

这说明首次尝试没有留下一个可恢复、可交付的有效视频渲染任务。因为第一条 trace 因 Langfuse HTTP 500 无法取回，无法 100% 还原首次最终动作；但现有证据能支持以下判断：

- 用户后续明确追问 `Please deliver the video`。
- 追问后的链路试图恢复旧渲染任务。
- 旧任务返回 `job_not_found`。
- Agent 随后从 HTML 重新提交渲染。

因此，首次问题本质是**交付状态失败**：系统没有在第一轮稳定交付完成的视频 asset，用户侧可能只看到了预览图、封面图或类似图片的结果。

## 为什么第二次视频后面有长黑屏

生产链路生成了两段 8 秒视频：

1. 主画面：鞋子旋转 close-up。
2. PiP 画面：穿着鞋跑步的动态镜头。

但这两段不是顺序场景。用户明确要求的是画中画，因此正确结构应该是：

```text
0s ---------------- 8s
主画面旋转鞋子:  [================]
PiP 跑步画面:        [=============]
参数卡叠加:             [==========]
```

错误结构则把它们当成了时长库存：

```text
主画面 8s + PiP 8s + 片头/片尾包装 ~= 18s
```

这对画中画合成是不成立的，因为主视频和 PiP 视频是在同一时间段里叠加播放，而不是先后播放。最终渲染时长是 18 秒，但视觉素材本身只覆盖约前 8 秒；如果 HTML 没有显式延展素材，后半段就会露出黑底。

当时缺少这些兜底：

- 没有给嵌入视频设置 `loop`。
- 没有在源视频结束后冻结最后一帧。
- 没有在尾部切换到产品静帧或品牌 end card。
- 没有在交付前做黑帧/暗帧视觉 QA。

Agent 虽然运行了 `probe_media`，但它只能证明文件技术上有效：

```text
18s, 1920x1080, 30fps, H.264 video + AAC audio
```

它不能证明画面内容在 18 秒内都正常可见。

## 正确的需求理解

这个需求应该被理解为多图层并行合成：

- 旋转鞋子 close-up 是主背景层。
- 跑步镜头是右下角 PiP 叠加层。
- 产品参数卡是另一个动画叠加层。
- 三者应该共处同一个时间轴。

因此，两段 8 秒视频不能相加来决定总时长。

如果最终视频一定要 18 秒，则至少需要以下一种处理：

- 生成真正 18 秒的主画面素材。
- 对 8 秒主画面和 PiP 素材做自然循环。
- 在 8 秒后冻结最后一帧。
- 切到静态产品 end card。
- 或者把最终成片裁到真实有效画面时长。

## 解决方案

### 1. 时间线规划规则

为 PiP、split-screen、多图层并行合成增加规划规则：

```text
当多个 clip 是并行叠加关系时，最终时长取各图层时长的最大值，而不是求和。
```

本 case 中：

```text
max(主画面 8s, PiP 8s) = 8s 可用视觉时长
```

18 秒目标只有在额外生成、循环、冻结或补 end card 的情况下才成立。

### 2. HyperFrames 写作规则

每个 HTML 内嵌 `<video>` 都必须明确源视频结束后的行为：

- `loop`
- 冻结最后一帧
- 替换为 poster / end card
- fade 到品牌静帧
- 或裁短合成时长

除非黑底是明确设计的一部分，否则不能允许 root background 成为尾部唯一可见层。

### 3. 渲染任务恢复规则

如果 `query_render` 返回：

```text
hyperframes.job_not_found
```

则必须：

- 不再沿用旧任务状态。
- 重新提交渲染。
- 保存新的 job id。
- 只交付新的已完成 asset。
- 不把旧预览图或旧中间 asset 当作最终结果暴露给用户。

### 4. 交付前视觉 QA

`ffprobe` 或 `probe_media` 不足以作为最终 QA。`show_final_video` 前应增加逐帧视觉检查：

- 抽取 `0s`、`25%`、`50%`、`75%`、`duration - 1s` 等关键帧。
- 计算平均亮度和非黑像素比例。
- 如果尾部连续帧接近纯黑，判定渲染失败。
- 自动重渲染或裁掉黑屏后再交付。

建议规则：

```text
如果最终视频最后 20% 的连续帧亮度极低，且没有显式 fade-to-black 设计标记，则阻止交付。
```

### 5. 本项目补救方案

针对项目 `63593003829`，建议修复步骤：

1. 重新打开 `sneaker_showcase.html`。
2. 把 `sneaker_rotating_closeup` 作为主画面层。
3. 把 `sneaker_running_pip` 作为并行 PiP 层。
4. 如果保留 18 秒，就对素材做 loop / freeze / end card；否则将成片缩短到约 8-10 秒。
5. 重新渲染。
6. 执行黑帧视觉 QA。
7. 确认无黑屏后再交付新的 mp4。

## 最终诊断

这不是用户素材问题。

问题由三点共同导致：

1. 首次渲染任务丢失或过期，导致第一次交付看起来不完整。
2. 对画中画关系的时长规划错误，把两段并行 8 秒视频当成了可顺序相加的 16 秒素材。
3. 交付前只检查了视频编码和时长，没有检查画面是否存在尾部黑屏。
