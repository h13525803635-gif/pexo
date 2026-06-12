# Case 65160007362：考图未图生视频、无音频音效、卡片重叠

## 结论

本 case 的核心问题不是视频模型生成失败，而是执行路径一开始就从「图片生成新闻视频」偏成了 **HyperFrames 静态图片 + HTML/GSAP 数据图表动画合成**。最终交付的 `election_news_video.mp4` 是 HTML 渲染结果，不是把 `04_news/03_election_scene.png` 送入 `video_generate` 后得到的图生视频。

## 证据

- 用户需求：`Generate an election news video using 04_news/03_election_scene.png...`
- 交付 trace：`analysis/langfuse-data/cases/65160007362/trace-2-4cd8a76f.json`
- 工具调用统计：
  - `submit_render`: 1 次
  - `query_render`: 9 次
  - `show_final_video`: 1 次
  - `sleep`: 8 次
  - `write_todos`: 1 次
- 未出现：
  - `video_generate`
  - TTS / voiceover 生成
  - BGM / 音效生成
  - `execute_edit_video`
  - `probe_media` / 最终音视频流验收

最终渲染链路为：

1. 旧 job `304uh692` 查询失败：`hyperframes.job_not_found`
2. 重新提交 HyperFrames：`submit_render(html_file='asset://a_pWL3W42', output_name='election_news_video')`
3. 轮询 job `9pg1oo1y`
4. 渲染完成，资产 `a_8MrRyaF`
5. `show_final_video('/projects/65160007362/workspace/assets/election_news_video.mp4')`

## 根因

### 1. 「考图」没有生视频

Agent 把需求理解成「对静态 election scene 图片叠加新闻图表包装」，而不是「基于图片生成真实新闻视频」。它只把原图作为 HTML `<img id="bg-img">` 背景铺满画面，然后用右侧面板、饼图、候选人卡片做动画。

这导致画面底图本身没有人物动作、镜头运动、现场氛围变化；用户看到的是静态图上的动态图形包装。

### 2. 没有音频/音效

HyperFrames composition 内没有 `<audio>` 轨，也没有先生成 VO/BGM/SFX 资产。交付 trace 里完全没有 TTS、BGM、SFX 或混音工具调用。

更深一层是 skill 入口门没有生效：HyperFrames skill 明确要求当意图不是 `explicit_silent` 时，第一次写 composition 前应先确定 `audio_intent`，并准备 `vo_*` / `bgm_*` 资产。但本次直接进入 HTML composition 和 render。

### 3. 卡片与卡片之间重叠/挤压

右侧面板使用固定尺寸排版：

- `#right-panel`: `top: 100px; bottom: 76px; width: 580px; gap: 24px`
- `#chart-container`: `300px × 300px`
- `#panel-title`: `margin-top: 28px; padding-bottom: 16px`
- `.cand-card`: `width: calc(100% - 48px); padding: 18px 24px`
- 卡片动画：`tl.set("#card-a", { opacity: 0, y: 30 }, 4.9)`，`tl.set("#card-b", { opacity: 0, y: 30 }, 4.9)`

问题点：

- 右侧面板没有用 CSS grid 明确划分「标题 / 图表 / 图例 / 卡片区」高度。
- 候选人卡片没有固定高度，也没有卡片容器负责 `gap`、`overflow` 和安全区。
- 两张卡片用相同的 `y: 30` 入场位移，若在中间帧或渲染采样时布局空间不足，会出现视觉上互相压住或贴得过近。
- 未看到针对 7.2s、8.2s 或最终态的帧级 QA 门禁；Agent 曾提到 8 秒预览正确，但没有最终资产级视觉验收。

## 解决方案

### 立即修复这条视频

1. 重新路由为图生视频：
   - 使用 `03_election_scene.png` 作为 `image_list`
   - 调用 `video_generate`，例如 Seedance/Kling reference2video
   - prompt 明确：投票站新闻现场、轻微推镜/横移、人员微动作、新闻现场氛围、5 秒后为后期图表留出右侧构图空间

2. 补音频：
   - 若要新闻播报：生成一条 16s 左右 VO
   - 生成或选择 news bed BGM
   - 加入投票站环境声/新闻提示音
   - 合成时设置视频原声、VO、BGM、SFX 的音量优先级

3. 重新做图表卡片叠加：
   - 底层用图生视频结果，不再用静态 `<img>` 作为唯一画面
   - 右侧 overlay 用 grid：
     - title 固定区
     - chart 固定区
     - legend 固定区
     - cards stack 固定区
   - card stack 用 `display: flex; flex-direction: column; gap: 16px;`
   - 卡片固定 `min-height` / `height`，文字长时缩小或换行
   - 入场动画只动 opacity 和 transform，不改变布局占位

4. 验收：
   - `probe_media` 检查最终 mp4 有 video stream + audio stream
   - 抽帧检查 0s、5s、7.2s、8.2s、12s
   - 检查右侧卡片 bounding boxes 无交叠
   - 检查音轨非静音

### 系统性修复

1. 路由规则：
   - 用户说 `generate a video using <image>` 时，默认必须进入 `video_generate`。
   - 只有用户明确要求「信息图、动态图表、海报动效、HTML motion graphic」时，才允许纯 HyperFrames。
   - 如需图表叠加，正确路线是 `video_generate -> HyperFrames overlay/assembly`，不是 `static image -> HyperFrames only`。

2. HyperFrames 入口门：
   - `audio_intent != explicit_silent` 时，禁止直接 `submit_render`。
   - 必须先完成 VO/BGM/SFX 资产规划或明确记录用户选择无声。

3. 布局门禁：
   - 叠加图表/卡片类 composition 必须使用布局容器，不允许靠自然流 + 固定大元素硬塞。
   - render 前必须至少抽取「所有元素都可见」的关键帧做视觉检查。
   - 对 repeated cards 做自动 bbox overlap 检查。

4. 交付门禁：
   - 最终交付前必须 probe 最终 asset，而不是只看 render job 成功。
   - 如果用户要求视频且不是 explicit silent，最终文件无音轨应阻断交付。
