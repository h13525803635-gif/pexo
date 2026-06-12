# Case 16670575106 — 底图未生成视频且成片无音频

## 结论

项目 `16670575106` 的问题不是视频模型生成失败，而是执行链路从一开始就没有进入图生视频或音频生成路径。Agent 将用户的“生成历史课堂视频”误路由为 HyperFrames 静态 HTML 动效合成：直接把上传的课堂 PNG 写成 `<img>` 背景，再叠加时间轴和文字卡片，最后由 HyperFrames render 成 MP4。

因此最终文件虽然是 MP4，但底层画面不是由 `video_generate` 生成的动态底色视频；音频也不是合成丢失，而是从未生成、从未绑定。

## 用户需求

用户原始需求：

> Generate a history lecture video using `02_education/02_history_classroom.png`. Pan the main video to the left to leave the right 1/3 of the screen blank. Starting at 00:03, reveal a vertical timeline axis on the right using a wipe-down animation. Then, at 00:05, 00:08, and 00:11, fade in text cards next to the timeline nodes: `221 BC: Qin Dynasty Unifies China`, `618 AD: Tang Dynasty Established`, `1368 AD: Ming Dynasty Established`.

上传素材：

- `/projects/16670575106/workspace/assets/a_SAjhDL8_10_02_history_classroom.png`

这个需求里有两个关键语义：

1. `Generate a history lecture video using ... png`：应先用图片生成底色视频。
2. `Pan the main video to the left`：用户指的是主视频素材，而不是只把静态图片裁到左侧。

## Trace 证据

### 1. 实际没有调用视频生成

真实工具链只有：

- `get_file_info`
- `write_file`
- `render_frame`
- `submit_render`
- `query_render`
- `show_final_video`

没有以下任何调用：

- `video_generate`
- reference-to-video / image-to-video
- `execute_edit_video`

Agent 初始 todo 是：

- `Author the HTML composition with panned video, timeline axis, and animated text cards`

但随后变成：

- `Author the HTML composition with panned image, timeline axis wipe-down, and animated text cards`

这里发生了关键降级：从“panned video”变成了“panned image”。

### 2. HTML 直接引用上传 PNG

写入的 `history-lecture.html` 中，背景是静态图片：

```html
<img id="bg-img"
  src="https://pexo-assets.../10_02_history_classroom.png"
  alt="" />
```

没有 `<video>` 背景元素，也没有任何底色视频 signed URL。

最终产物：

- render job: `8udlhkia`
- final asset: `a_yPN3Mmp`
- file: `/projects/16670575106/workspace/assets/history_lecture_timeline.mp4`

这说明最终 MP4 是 HyperFrames 对 HTML/GSAP 静态图片动效的录制结果，不是“先图生视频、再叠加时间轴”的结果。

### 3. 实际没有调用音频生成，也没有绑定音频

trace 中没有以下调用：

- `audio_produce`
- `music_generate`
- TTS / VO 生成
- SFX 生成

HTML 中也没有：

```html
<audio src="...">
```

最终交付前没有 `probe_media` 校验 audio stream。因此无声不是 mux 阶段丢音轨，而是整条链路没有创建音频层。

### 4. HyperFrames warning 被放过

`submit_render` 返回了 warning：

- `timed_element_missing_clip_class`：`<div id="scene-main">` 有 timing attributes 但没有 `class="clip"`。
- `root_composition_missing_data_start`：root composition 缺少 `data-start="0"`。

这些 warning 本应在交付前修复，但实际仍继续 query render 并 `show_final_video`。

## 根因

### 根因 1：路由把“生成视频”误收敛成“输出 MP4”

Agent 看到右侧时间轴、文字卡片、wipe-down、fade-in 等动效需求后，选择了 HyperFrames。这个路由适合做后期 overlay，但不应替代底色视频生成。

错误点在于：HyperFrames render 也会输出 MP4，于是 Agent 把“输出一个 MP4”当成满足“生成视频”，忽略了用户要求的 `main video` 必须先从上传图生成动态底视频。

### 根因 2：静态图片 `<img>` 被用作底视频替身

用户说的是 `main video`，但实际 HTML 只做了静态 PNG 的左侧裁切和轻微入场动画。这个实现只能产生静态背景上的 UI 动效，不会有真实课堂镜头运动、人物/场景动态或视频质感。

### 根因 3：音频意图没有结构化

历史 lecture 类视频默认不应静音，至少需要一条教育类 BGM，或旁白 + BGM。Agent 没有建立 `audio_intent`，也没有生成/选取 BGM、VO 或 SFX。

### 根因 4：交付门禁缺失

两类门禁都没有阻断：

1. 非静音视频无 `<audio>`、无 `probe_media`。
2. HyperFrames timing warning 未修复仍交付。

## 当前项目修复方案

### 1. 重新生成底色视频

使用上传图 `a_SAjhDL8_10_02_history_classroom.png` 先调用图生视频：

- 时长：14s 左右。
- 画幅：16:9 / 1920x1080。
- 视觉：历史课堂、轻微镜头 pan left、保留右侧 1/3 干净空间用于后期时间轴。
- 约束：不要在底视频中提前生成右侧时间轴和文字卡片。

得到底色视频后，HyperFrames HTML 应使用：

```html
<video
  src="https://.../history_classroom_base.mp4"
  data-start="0"
  data-duration="14"
  playsinline>
</video>
```

不能再用上传 PNG 的 `<img>` 作为主画面。

### 2. 重新叠加时间轴和卡片

用 HyperFrames 只负责 overlay：

- `00:03` 时间轴 wipe-down。
- `00:05` 第一个节点和卡片 fade in。
- `00:08` 第二个节点和卡片 fade in。
- `00:11` 第三个节点和卡片 fade in。

root 和 scene 元素必须补齐：

```html
data-start="0"
class="clip"
```

### 3. 补齐音频

如果用户没有明确要求静音，至少生成并绑定一条 14s 教育/历史讲解类 BGM：

```html
<audio
  src="https://.../history_lecture_bgm.mp3"
  data-start="0"
  data-duration="14"
  data-track-index="10">
</audio>
```

如果需要更贴合 lecture，可追加英文/中文旁白；但不能只用静音交付。

### 4. 交付前校验

交付前必须完成：

1. `probe_media` 底色视频，确认有视频流且时长约 14s。
2. `probe_media` BGM/VO，确认音频非空。
3. `render_frame` 检查 3s、5s、8s、11s 的 overlay 时序。
4. `probe_media` 最终 MP4，确认同时包含 video stream 和 audio stream。
5. 若最终无音频且用户未明确 silent，阻断交付。

## 系统级修复建议

1. **路由 gate**
   - 当用户出现 `Generate ... video using <image>`、`main video`、`base video`、`pan the video` 等语义时，必须先产出 base video。
   - HyperFrames 只能作为后期叠加，不允许直接用 `<img>` 替代底视频。

2. **静态替代检测**
   - 若 prompt 要求生成视频，但 composition 主视觉只有 `<img>` 且没有 `<video>` / `video_generate` 产物，应阻断 `submit_render`。

3. **音频 gate**
   - 非明确 silent 时，必须生成或选择 BGM/VO/SFX。
   - `submit_render` 前 HTML 必须包含至少一个 `<audio src="https://...">`。
   - `show_final_video` 前必须 `probe_media`，无 audio stream 则阻断。

4. **warning gate**
   - `timed_element_missing_clip_class`
   - `root_composition_missing_data_start`

   这类 warning 会影响最终可见效果，应升级为不可交付问题。

## 对用户解释口径

这次用户要求没有问题，问题出在执行路由：Agent 把“生成视频”理解成“用 HTML 动效渲染出一个 MP4”，于是直接用静态 PNG 做背景，没有先调用图生视频模型；音频也没有被规划和挂载。正确做法是先把上传课堂图生成 14 秒底色视频，再用 HyperFrames 添加右侧时间轴和卡片，并补 BGM/旁白后重新渲染。
