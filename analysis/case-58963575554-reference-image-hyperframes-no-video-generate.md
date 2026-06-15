# Case 58963575554: 参考图未生成视频而是静态垫图 + HTML 渲染

**项目 ID**: `58963575554`  
**Langfuse traceName**: `pexo:58963575554`  
**Trace 时间**: `2026-06-11T02:37:35.575Z`  
**环境**: `test`  
**问题类型**: 路由错误；参考图被当作 HyperFrames 静态背景，未进入 `video_generate`

## 一句话结论

用户要求「使用 `01_news_anchor.png` 生成 financial news anchor broadcast video」，Agent 全程未调用 `video_generate`，而是走 HyperFrames/HTML 路径：把参考 PNG 作为整段 `<img>` 背景，再叠加 K 线动画和底部 ticker，最后 `submit_render` 成 MP4。最终产物是「静态主播图 + UI 动效」，不是 AI 图生视频的动态播报 footage。

## 用户原始需求

```text
Generate a financial news anchor broadcast video using 04_news/01_news_anchor.png.
Overlay a dynamic candlestick stock chart animation (green for up, red for down)
on the left half of the screen.
At the very bottom, add a 60px high dark blue background strip and seamlessly
scroll the text from right to left:
'AAPL +1.2% ▲ | TSLA -0.5% ▼ | MSFT +2.1% ▲ | NVDA +3.8% ▲'.
```

用户附带参考图：`/projects/58963575554/workspace/assets/a_cYJtTts_21_01_news_anchor.png`（专业新闻主播棚拍静态图）。

## 实际执行链路

Trace 中实际工具链（共 1 轮、60 条 message）：

```text
write_todos
  -> read_file(hyperframes-skill)
  -> analyze_file_content(21_01_news_anchor.png)
  -> read_file(composition-skeleton / composition-rules)
  -> get_file_info(a_cYJtTts)          # 仅取 signed_url
  -> list_fonts
  -> write_file(financial_news.html)
  -> edit_file(BLOCK_SCENE_CONTENT)    # 插入 anchor-bg <img> + chart + ticker
  -> edit_file(BLOCK_TIMELINE)         # GSAP 动画
  -> submit_render
  -> query_render × 8 + sleep × 8
  -> show_final_video(financial_news_broadcast.mp4)
```

**全程零次 `video_generate` / `image_generate` 调用。**

参考图在 HTML 中的用法：

```html
<img id="anchor-bg"
     src="https://pexo-assets.oss-us-east-1.aliyuncs.com/.../21_01_news_anchor.png?..."
     alt="" />
```

参考图只通过 `get_file_info` 拿到 OSS 签名 URL，写入 `<img>` 标签，**未进入 `video_generate.image_list`**。

## 交付产物

| 资产 | 路径 / ID | 说明 |
|------|-----------|------|
| 参考图 | `asset://a_cYJtTts` | 4.4MB PNG，新闻主播棚拍 |
| 最终视频 | `financial_news_broadcast.mp4` → `asset://a_hW4MEpY` | HyperFrames 渲染，约 15s，1920×1080 |
| HTML 合成 | `financial_news.html` | GSAP + 静态 `<img>` 背景 |

渲染 job `dwawqi6a` 成功完成（`render_ms`: 90954）。

## 根因

### 1. 路由把「UI 动效包装」当成了主任务

Agent 优先满足 K 线、ticker 等确定性前端动效，选择了 **hyperframes-skill**，而非 **generation-skill** 的图生视频路径。HyperFrames 适合 overlay / 字幕 / 跑马灯，但不负责把静态人物图生成动态 footage。

### 2. 「Generate video using image」被误读

用户说 **generate video using image**，系统应走：

```text
参考图 → video_generate(image2video/reference2video) → 动态主播 footage → overlay
```

实际走了：

```text
参考图 → HTML <img> 静态背景 → submit_render → MP4
```

「渲染成 MP4」被误当作「图生视频」。

### 3. 主播人物不会动

参考图是静态棚拍图。HyperFrames 只能让 K 线、ticker 动起来，**主播不会说话、做手势或播报**，与「news anchor broadcast video」的用户预期不符。

### 4. 次要问题

| 问题 | 详情 |
|------|------|
| Lint 未执行 | Todo 标记「Lint and frame preview check」为 completed，但未调用 `lint_composition` / `render_frame` |
| 渲染 warning | `[timed_element_missing_clip_class]`、`[root_composition_missing_data_start]` |
| 布局遮挡 | 左侧 K 线面板 960px 宽 + 72% 不透明，遮挡主播左半边 |
| 无音频 | 新闻播报类未生成口播 / BGM |

## 与同类 Case 对照

| Case | 现象 | 根因 |
|------|------|------|
| [96559468147](case-96559468147-livestream-static-background-root-cause.md) | 直播 highlight 首次成片静态垫图 | 同上：HyperFrames `<img>` 替代 `video_generate` |
| [56440397081](case-56440397081-reference-image-hyperframes-audio-solution.md) | 牛排 close-up 未图生视频 | 同上 |
| [79993314648](case-79993314648-weather-host-static-no-audio-white-tail.md) | 天气主播静态 + 无音频 | 同上 + 音频门禁缺失 |

## 正确补救方案

### 项目级重做

```text
1. video_generate
   - mode: image2video 或 reference2video
   - image_list: [asset://a_cYJtTts]
   - prompt: 新闻主播对镜头播报财经新闻，自然手势，专业演播室
   - duration: 10-15s, aspect_ratio: 16:9, sound: on

2. audio_produce（可选）
   - 生成英文财经播报口播

3. HyperFrames overlay
   - 背景从 <img> 改为 video_generate 输出的 <video>
   - 左侧 K 线动画 + 底部 60px ticker
   - submit_render → show_final_video
```

### 系统级修复建议

#### A. 路由门禁

```text
当用户请求 create/make/generate video using image，
或需求包含 broadcast / news anchor / footage / presenter：
  - 上传图必须进入 video_generate.image_list
  - 禁止仅用 <img> PNG/JPG 作为整段主视觉渲染交付
  - 除非用户明确要求静态海报 / 纯 UI 动效
```

#### B. 交付前静态主视觉检查

```text
if user_intent contains video/footage/broadcast/anchor
and main visual is uploaded PNG/JPG covering most of timeline
and no successful video_generate exists:
    block delivery or require image-to-video first
```

#### C. 工具分工约束

- `video_generate`：动态人物、动态场景、播报 footage
- HyperFrames/HTML：K 线、ticker、字幕、卡片等确定性图层
- `show_final_video`：仅交付已满足主视觉动态要求的最终产物

## 推荐回归用例

输入：

```text
Generate a financial news anchor broadcast video using [uploaded anchor image].
Overlay candlestick chart on the left half.
Add scrolling stock ticker at the bottom.
```

期望：

- 至少一次成功 `video_generate`，`image_list` 包含上传图 asset
- HyperFrames 主视觉使用生成的 `.mp4`，不是原始 `.png`
- 主播有动态播报动作（非静态垫图）
- 可选：口播 / BGM 音轨
