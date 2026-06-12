# Case 35307705381 — 足球进球成片问题根因与解决方案

## 结论

本 case 的主要问题不是底层视频素材生成失败，而是成片组装链路选错并失控：

- Kling `reference2video` 已成功从源图生成动态足球视频。
- Agent 随后没有直接交付该动态视频，而是进入 HyperFrames HTML 二次合成，制作 `soccer_goal_freeze_final.mp4`。
- HTML 渲染阶段反复提交同一 composition，多个 render job 长时间轮询，最终才交付一条 HTML 合成产物。

一句话根因：**动态素材已生成成功，但 Agent 过度使用 HyperFrames freeze-frame 二次包装；当 HTML render 长时间不收敛时没有及时 fail-fast 或降级到已成功的视频素材，导致成片链路耗时失控并增加最终画面异常风险。**

## 证据链

完整 trace 详情接口对本项目返回 HTTP 500，因此本分析改用 Langfuse observations 还原调用链。

- 项目 ID：`35307705381`
- Trace ID：`77a96b1b83dbe2eb74c55dad0ff60f7f`
- Trace 起始时间：`2026-06-11T02:38:03.789Z`
- 本地 observation 摘要：`analysis/langfuse-data/cases/35307705381/observations-fullish.json`

关键调用如下：

1. 读取源图  
   `a_jVFmPvA_22_02_soccer_goal.png`

2. 调用 Kling 图生视频  
   `video_generate` 入参：
   - `provider`: `kling`
   - `model`: `kling-v3-omni`
   - `mode`: `reference2video`
   - 输出目标名：`soccer_goal_video`

3. 成功得到动态视频素材  
   `soccer_goal_video_20260611T024344_9f7605fb.mp4`

4. 进入 HyperFrames HTML 组装  
   生成/多次修改：
   `soccer_goal_composition.html`

5. 多次提交 HTML render  
   多个 job 被反复轮询：
   - `46narvjd`
   - `6eur6bv6`
   - `756aj4jm`
   - `1uga5fts`
   - `e0n1abcx`

6. 最终交付  
   `soccer_goal_freeze_final.mp4`

## 问题拆解

### 1. 链路选择错误

用户意图更接近「将足球图片生成动态视频/进球瞬间」。此类任务中，Kling 图生视频成功后，本应优先直接交付该动态视频。

但本 case 继续进入 HyperFrames，制作 freeze-frame/标注式包装，导致交付物从自然动态视频变成「动态视频 + 定格/叠层/HTML 组合」的复合结构。这增加了画面异常、静态段过长、渲染失败和最终体验偏离用户预期的风险。

### 2. HTML 渲染状态机缺少 fail-fast

同一个 HTML asset `asset://a_wGawtuG` 被多次提交渲染。尤其 `756aj4jm` 从 `2026-06-11T03:21:52Z` 开始被反复查询，直到 `2026-06-11T04:20:57Z` 仍在轮询。

这说明流程缺少明确的超时策略：

- 未在长时间 pending 后终止该 job。
- 未自动切换到 draft/低复杂度渲染的稳定路径。
- 未降级交付已成功生成的 Kling 视频素材。

### 3. 最终 QA 门禁不足

Trace 中有抽帧预览：

- `freeze_frame_preview_t0.png`
- `freeze_frame_preview_t14.png`

但没有看到对最终 mp4 的完整验收门禁，例如：

- 时长是否符合预期。
- 首帧/中段/尾帧是否非黑屏。
- 是否存在音轨。
- 画面是否持续运动而非异常冻结。
- 最终交付 asset 是否确为最后一次成功 render 产物。

## 解决方案

### 短期修复

1. 对「单图生成动态视频」类请求，默认直接交付 `video_generate` 成功产物。
2. 只有用户明确要求字幕、讲解卡、标注、定格分析、信息图包装时，才进入 HyperFrames。
3. 若 HTML render job 超过 10-15 分钟仍未完成，应停止轮询并切换方案。
4. 已有成功 `video_generate` 产物时，HTML render 失败或超时时应降级交付该视频，而不是继续等待。

### 中期修复

1. 在 planner 阶段增加路由判断：
   - `image-to-video only`：走视频生成直出。
   - `video + overlays/explainer`：才走 HyperFrames。
2. 在 `html_render_video` 调用层增加超时和最大重试次数。
3. 禁止对同一 HTML asset 无限制重复提交 standard render。
4. 成片交付前强制执行 probe 与抽帧 QA。

### 推荐 QA 清单

交付前至少校验：

- `ffprobe` 确认存在 video stream。
- 非明确静音时确认存在 audio stream。
- 首/中/尾抽帧非黑屏、非透明、非异常静态。
- 最终时长与计划时长偏差在可接受范围内。
- 若用户要动态视频，至少抽检多帧差异，确认不是静态幻灯片。

## 建议的产品规则

当请求满足以下条件时：

- 输入是单张图片；
- 用户要求「变成视频」「动起来」「生成进球/动作/场景运动」；
- 未要求字幕、讲解卡、标注、分镜包装；

则默认路径应为：

```text
source image -> video_generate(reference2video) -> probe/QA -> show_final_video
```

不应默认进入：

```text
source image -> video_generate -> HTML composition -> html_render_video -> show_final_video
```

后者只适用于明确需要 UI/字幕/信息层/教学标注的成片。
