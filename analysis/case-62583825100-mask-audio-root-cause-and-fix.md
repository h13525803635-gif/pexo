# 项目 62583825100：遮罩蒙版时序异常与成片无声问题分析

## 用户 Prompt

> 使用 `01_ecommerce/02_skincare_serum.png` 生成一条护肤产品视频。从 `00:08` 开始，在主视频上添加一个 `30%` 透明度的黑色遮罩蒙版。然后在右侧每隔 1 秒从下到上滑入用户评论气泡卡片：`Skin feels so smooth! ⭐⭐⭐⭐⭐`、`Buying my 3rd bottle`、`Highly recommended`。评论卡片采用错落的瀑布流布局，停留 5 秒后全部向右滑出。

## 问题现象

- 成片没有稳定按用户要求执行 `00:08` 开始的遮罩/评论卡片时序。
- 交付成片没有声音。

## Trace 证据

### 1. 底色视频生成时就是静音

底色视频生成调用使用的是 Kling reference-to-video，并显式设置了 `sound: "off"`：

```json
{
  "name": "skincare_serum_base",
  "provider": "kling",
  "model": "kling-v3-omni",
  "mode": "reference2video",
  "provider_param": {
    "kling_reference2video": {
      "duration": "15",
      "sound": "off"
    }
  }
}
```

生成出的底色视频资产为：

- `asset://a_abvLj5f`
- `/projects/62583825100/workspace/assets/skincare_serum_base_20260611T023405_624b4b14.mp4`

对该资产执行 `probe_media` 后，只看到 1 路视频流，没有音频流。

### 2. 最终 HTML 静音引用底色视频，且没有添加任何音频

合成 HTML 中引用底色视频时带有 `muted`：

```html
<video
  id="main-video"
  src="https://pexo-assets.../skincare_serum_base_20260611T023405_624b4b14.mp4"
  muted
  playsinline
></video>
```

合成文件中没有 `<audio>` 标签；trace 中也没有看到 BGM、VO、环境声或音效生成步骤。

最终交付资产为：

- render job：`4ivvw93m`
- final asset：`asset://a_8m3MKuM`

对最终资产执行 `probe_media` 后，同样只看到视频流，没有音频流。

### 3. 第一次成功渲染时，遮罩和评论卡片仍有时序告警

第一次成功 render 是在 overlay 时序告警修复前提交的。当时 HyperFrames 返回了这些 warning：

- `<div id="mask">` 缺少 `class="clip"`，触发 `timed_element_missing_clip_class`
- `<div id="cards-overlay">` 缺少 `class="clip"`，触发 `timed_element_missing_clip_class`
- 根 composition 缺少 `data-start`，触发 `root_composition_missing_data_start`

这些 warning 的含义是：带 `data-start` / `data-duration` 的元素如果没有 `class="clip"`，可能不会只在计划时间段内显示，而是全片可见。

后续 agent 确实修改了 HTML，补上了 `class="clip"` 和 root timing，但第二次 render job：

- `6bq9ray6`

一直处于 pending。最终交付的仍然是较早完成的 render job `4ivvw93m`，也就是基于带 warning 的 HTML 版本生成的成片。

## 根因

### 遮罩/评论卡片问题

遮罩和评论卡片是通过 HTML overlay 实现的，但第一次完成渲染的 HTML 版本缺少 HyperFrames 所需的时序元数据。具体来说，带时间控制的 `mask` 和 `cards-overlay` 元素缺少 `class="clip"`，导致它们无法被可靠地限制在 `data-start="8"` / `data-duration="7"` 的时间窗口内。

虽然后续 HTML 被修正了，但修正后的版本没有成功完成并被交付；最终交付的是修复前的 render 结果。

### 成片无声问题

成片无声是因为整条链路都按静音处理：

1. 底色视频生成时使用了 `sound: "off"`。
2. HTML 合成中用 `muted` 引用底色视频。
3. 合成文件没有添加 `<audio>` 音轨。
4. 没有生成 BGM、VO、环境声或音效资产。
5. 最终 `probe_media` 确认交付 MP4 没有音频流。

这不是最终 mux 阶段丢音轨，而是音频层从一开始就没有被创建或挂载。

## 当前项目修复方案

### 1. 只能基于修正后的 HTML 重新渲染

overlay 元素必须包含 `class="clip"` 和完整时间信息：

```html
<div
  class="mask-layer clip"
  id="mask"
  data-start="8"
  data-duration="7"
  data-track-index="1"
></div>

<div
  id="cards-overlay"
  class="clip"
  data-start="8"
  data-duration="7"
  data-track-index="2"
></div>
```

根 wrapper 也需要包含时间归属信息：

```html
data-start="0"
data-duration="15"
```

不要交付 job `4ivvw93m`，因为它是在时序修复前生成的。

### 2. 最终渲染前补充音频

用户没有要求静音，因此至少应添加一条音频层。对这条护肤产品视频，最稳妥的默认方案是生成一条 15 秒的高级感、轻奢护肤类 BGM。

最终 HTML 中应加入使用 signed URL 的音频元素，例如：

```html
<audio
  src="https://..."
  data-start="0"
  data-duration="15"
  data-track-index="3"
></audio>
```

如果使用环境声或音效代替音乐，也必须遵守同一原则：最终 composition 中必须存在音频资产。

### 3. 交付前最终校验

交付前必须完成以下检查：

- 使用最新修正后的 HTML 重新 render。
- 确认最终 render job 对应的是最新 HTML 版本。
- 对最终 MP4 执行 `probe_media`。
- 如果最终 MP4 没有音频流，阻断交付。
- 抽帧检查关键时间点：
  - `7.9s`：遮罩不应可见。
  - `8.0s` 到 `8.6s`：遮罩应淡入到 30% 透明度。
  - `8.3s`、`9.3s`、`10.3s`：评论卡片应依次进入。
  - `13.5s` 之后：评论卡片应向右滑出。

## 系统级防复发方案

### 1. 将 overlay 时序 warning 升级为阻断

当 HyperFrames 返回以下 warning 时，不允许最终交付：

- `timed_element_missing_clip_class`
- `root_composition_missing_data_start`
- timed overlay elements with `data-start` but no `class="clip"`

这些 warning 会直接影响用户可见的时序效果，不能作为普通 warning 放过。

### 2. 强制交付最新 HTML 对应的 render

如果 HTML 在某个 render job 提交后又被修改，那么这个 render job 就已经过期。即使该旧 job 成功完成，也不能作为最终结果交付。

最终资产选择必须满足：

- 来自最新 HTML revision。
- lint / render 都针对同一份最新 HTML。
- 最终媒体 probe 通过。

### 3. 增加音频意图门禁

除非用户明确要求静音：

- `audio_intent` 不应是 `explicit_silent`。
- 最终 composition 必须至少包含一个音频源。
- `probe_media(final)` 必须能看到音频流。

如果 `video_generate` 使用了 `sound: "off"`，则 assembly/composition 阶段必须补充 BGM、环境声、VO 或音效后才能交付。

### 4. 增加最终 QA 硬检查

对此类请求增加以下硬检查：

- 遮罩必须在用户要求的时间点开始。
- 遮罩透明度必须达到用户要求的目标值。
- 评论卡片必须按用户要求的节奏进入。
- 评论卡片必须按用户要求停留。
- 评论卡片必须按用户要求方向退出。
- 除非用户明确要求静音，最终视频必须包含音频流。
- 交付资产必须来自最新一次成功 render。

## 总结

该项目失败的直接原因有两个：第一，最终交付的 render 来自仍有 timed-overlay warning 的 HTML 版本；第二，音频层从生成到合成都没有被创建或挂载。即时修复方案是：基于补齐 `class="clip"` 的 overlay HTML 重新渲染，并添加 BGM 或其他音频轨。长期修复方案是：将 timed-overlay warning 和最终无音频流升级为交付阻断条件。
