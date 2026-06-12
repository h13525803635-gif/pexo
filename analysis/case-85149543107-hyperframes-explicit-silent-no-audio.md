# 项目 85149543107：HyperFrames explainer 被误判为 explicit_silent 导致无音频

## 结论

项目 `85149543107` 没有音频，不是最终渲染或交付阶段把音轨丢了，而是生产计划阶段就把该视频标记成了 `audio_intent: explicit_silent`。

因此后续链路只执行了 HTML / GSAP 视觉动效制作与 HyperFrames 渲染，没有生成 VO、BGM 或 SFX，也没有在 HTML 中挂载 `<audio>` 元素。最终 MP4 是按视觉-only composition 正常渲染并交付的。

## 证据

本地拉取到的 Langfuse trace 显示：

- 项目共有 5 条 trace，其中第 3、4、5 条成功拉取；第 1、2 条详情接口返回 500。
- 最终交付路径为：
  - `/projects/85149543107/workspace/assets/internet-explainer-final.mp4`
- 最终渲染 job：
  - `job_id: 0k6mb1w8`
  - `status: done`
  - `asset_id: a_cC3TXJq`
  - `render_ms: 150781`
- 会话摘要明确记录：
  - `audio_intent: explicit_silent (no audio elements needed/requested)`
- 真实工具链路主要是：
  - `write_file` / `edit_file`
  - `submit_render`
  - `query_render`
  - `show_final_video`
- 没有看到真实执行的 `audio_produce` 或 `music_generate` 调用。
- `audio_produce` / `music_generate` 字样只出现在工具 schema 或 skill 文档上下文中，不代表真实工具调用。

## 根因

### 1. 音频意图判断错误

用户请求是“how data travels across the internet”的 explainer video。虽然用户没有显式说要旁白、音乐或音效，但 explainer 类视频通常需要至少一种音频层：

- VO 讲解；
- 轻量 BGM；
- 数据包流动、节点连接等 UI / tech SFX。

Agent 却把该项目归类为 `explicit_silent`，使音频生产分支完全跳过。

### 2. HyperFrames 入口 gate 没有拦住默认静音

HyperFrames skill 中虽然有规则要求：

- 当 `audio_intent != explicit_silent` 时，必须先生成并绑定音频；
- 非静音时 HTML 需要 `<audio src="https://...">`；
- 交付前应 probe 最终产物是否有 audio stream。

但本 case 的问题在更前面：`audio_intent` 被错误设置成 `explicit_silent`，导致这些规则没有触发。

### 3. 交付前缺少“用户体验合理性”复核

最终 render 成功后直接 `show_final_video`。系统没有再问：

- 这是 explainer，静音是否合理？
- 用户是否明确要求 silent？
- 是否至少需要补一条 BGM 或 VO？

因此一个技术上成功、体验上不完整的静音视频被交付。

## 不是根因的项

- 不是 HyperFrames 渲染失败：`submit_render` 和 `query_render` 最终成功。
- 不是 show_final_video 丢音轨：交付的 MP4 本身就是按无音频 composition 生成。
- 不是音频文件生成后未绑定：trace 中没有真实音频生成调用。
- 不是音频工具不可用：工具 schema 中有 `audio_produce` / `music_generate`，只是没有被调用。

## 解决方案

### 1. 增加 audio intent 分类门禁

在进入 HyperFrames `write_file` 前必须结构化判断音频意图：

```json
{
  "audio_intent": "explicit_silent | implied_audio | needs_clarification",
  "reason": "...",
  "planned_audio_layers": ["vo", "bgm", "sfx"]
}
```

判断规则：

- 只有用户明确说 silent / no sound / mute，才允许 `explicit_silent`。
- explainer、教程、广告、产品介绍、剧情短片默认不能直接归为 `explicit_silent`。
- 用户没提音频时，应归为 `implied_audio` 或 `needs_clarification`。

### 2. Explainer 默认音频策略

对 explainer 类视频建议默认策略：

- 如果有脚本文案：生成 VO，并按视觉时长压缩或扩展脚本。
- 如果没有脚本文案：生成轻量 BGM + 少量 UI SFX。
- 如果用户明确不想要旁白：至少询问是否要 BGM / SFX。

本 case 合理补救版本可以是：

- 30-32 秒英文 VO，解释数据从 laptop 到 router、server、phone 的传输；
- 低音量 tech ambient BGM；
- packet whoosh、node ping、connection pulse 等轻量 SFX。

### 3. HTML audio bind 阻断

当 `audio_intent != explicit_silent` 时，渲染前必须检查 HTML：

```text
require HTML contains <audio src="https://...">
require all planned VO/BGM/SFX assets are referenced
block submit_render if audio assets exist but no audio element is present
```

### 4. 交付前 probe 阻断

最终 `show_final_video` 前必须执行媒体流校验：

```text
if audio_intent != explicit_silent:
    require final mp4 has audio stream
else:
    require explicit user-facing silent rationale
```

如果无 audio stream 且非明确静音，禁止交付，必须补音频或向用户确认 silent 版本。

### 5. 避免把“未提音频”当成“明确静音”

核心产品规则应调整为：

```text
unspecified audio != explicit_silent
```

尤其是 explainer / tutorial / marketing / story / product video，未提音频只能表示“音频未指定”，不能表示“用户不要音频”。

## 修复优先级

1. **P0**：`explicit_silent` 只能来自用户明确静音要求。
2. **P0**：非静音 intent 的 HyperFrames composition 没有 `<audio>` 时阻断渲染。
3. **P0**：非静音 intent 的最终 MP4 没有 audio stream 时阻断交付。
4. **P1**：explainer 默认生成 VO/BGM/SFX 计划。
5. **P1**：交付文案中标明 silent 版本，仅在用户明确选择静音时允许。

## 一句话复盘

这次无音频的本质是：agent 把一个默认应有讲解或背景声的 explainer 视频误判成 `explicit_silent`，导致音频生产、HTML 音频绑定和最终音轨校验都没有进入执行路径。
