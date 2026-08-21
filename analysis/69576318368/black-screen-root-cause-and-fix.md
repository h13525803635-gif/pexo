# Pexo 项目 69576318368：视频黑屏根因与解决方案

## 结论

项目的源视频已经成功生成，但这段单素材视频本不需要进入最终 HTML 成片合成。错误路由把它送入 Motion 合成后，约 12 秒的源视频被异常输出为 24 秒、约 213 kbps 的第一次成片；动态 `<video>` 素材没有得到有效像素，但渲染和 HLS 任务仍被标记成功，最终黑屏文件被交付。

后续流程把动态视频层替换成静态风格帧并重新渲染，绕过了黑屏，但丢失了原本的视频运动。这是降级处理，不是根因修复。

## 调查范围与证据完整性

- 项目 ID：`69576318368`
- 调查时间：2026-08-13（Asia/Shanghai）
- Langfuse 发现轨迹：6 条
- 已获取：6 条轨迹的观测数据
- Metabase 数据源：数据库 3（`pg-server`，Asia/Shanghai）
- Metabase 查询表：`public.projects`、`public.project_assets`、`public.asset_hls_generation_jobs`
- 关键轨迹：
  - `56fd3d77b2f4467b5c63d68e949897d6`：生成源视频、写入 HTML、首次最终渲染
  - `ebd69a92df809af151029a8966644f8e`：单独检查源视频内容
  - `994a51b38f00534dede7211791f2ff73`：检查黑屏帧、替换动态层、二次渲染
- 报告未保存原始轨迹、签名 URL、凭据或用户数据。

## 已确认事实

1. `video_generate` 成功产出源视频资产 `a_foducxe`，时长 `12.097007s`、尺寸 `720×1280`、H.264、码率约 `2.64 Mbps`、文件约 `3.99 MB`，资产和预览状态均为 `READY/FINAL`。
2. 该 MP4 后续被单独送入视觉分析，说明源文件存在且可被读取；问题不在视频生成请求失败。
3. 首次合成创建了包含动态 `<video id="wolf-clip">` 的 `composition.html`，随后产出最终资产 `a_4V6PBjM`。
4. 第一次成片时长为 `24s`、尺寸 `1080×1920`、码率仅约 `213 kbps`、文件约 `640 KB`。它与 `12.097007s` 的源视频存在明确的时长和信息量异常。
5. 第一次成片的 HLS 任务状态为 `ready`，错误字段为空；说明 HLS 成功只验证文件处理完成，不验证画面内容有效。
6. 用户反馈后，流程检查了输出帧和源视频，并编辑 `composition.html`，将原动态视频节点替换为静态图节点。
7. 第二次最终资产 `a_z1RDG1n` 为 `12.010667s`、`1080×1920`、码率约 `12.52 Mbps`、文件约 `18.80 MB`；HLS 同样为 `ready`。这与静态替代后的重新渲染时间线一致。
8. 数据库记录 `final_videos_count = 2`，但项目最终 `execution_status = INTERRUPTED` 且 `completed_at` 为空。

## Metabase 资产对照

| 阶段 | Asset ID | 时长 | 尺寸 | 码率 | 文件大小 | 数据库状态 | 含义 |
|---|---|---:|---:|---:|---:|---|---|
| 源生成视频 | `a_foducxe` | 12.097007s | 720×1280 | 2.64 Mbps | 3.99 MB | FINAL / READY | 源视频正常生成并可预览 |
| 第一次最终成片 | `a_4V6PBjM` | 24s | 1080×1920 | 0.213 Mbps | 0.64 MB | FINAL / HLS ready | 时长翻倍且信息量异常，黑屏成片仍被当作成功资产 |
| 第二次最终成片 | `a_z1RDG1n` | 12.010667s | 1080×1920 | 12.52 Mbps | 18.80 MB | FINAL / HLS ready | 静态替代后的第二次渲染 |

## 根因

### 首要根因：单素材视频被错误路由到最终成片合成

该项目只有一个已经生成好的 9:16 视频片段，无字幕、无文字叠加、无多镜头拼接、无复杂视觉布局。它应当直接交付源 MP4；如果只需处理声音，应走音频混合/封装，而不是重新录制 HTML 页面。

因此，最早使黑屏成为可能的错误是 Assembly/Motion 路由没有识别“单素材可直出”，不必要地引入了二次加载、解码、seek 和编码链路。

### 已确认的合成异常：12 秒源素材被输出为 24 秒

Metabase 直接证明第一次成片时长为源素材的近两倍。第一次成片码率和文件大小也远低于源视频及第二次成片，这与大量低信息量或黑色帧被编码一致。

这说明 `composition.html` 或 `submit_render` 的总时长没有正确继承源视频的实际可播放时长。24 秒异常是已确认的配置/时间轴故障，不应只归因于浏览器解码。

## 工具层定位：12 秒为什么变成 24 秒

Metabase `project_messages` 中保存的工具调用证明，24 秒不是视频生成工具产出的，也不是调用 `submit_render` 时显式传入的。

### 1. `media_probe` 已得到正确源时长

首次合成前，`media_probe(mode="info")` 返回：

```text
video_playable_seconds = 12.041667
container_duration = 12.098
video_fps = 24
```

因此源素材和探测工具均正常，Assembly 在写 HTML 前已经持有正确的可播放时长。

### 2. `write_file` 写入了无时间边界的 sequence wrapper

首次 `composition.html` 的关键结构是：

```html
<div data-composition-id="wolf-howl"
     data-start="0"
     data-duration="12">
  <div class="seq" data-hf-sequence>
    <video id="wolf-clip"
           data-duration="12"
           src="wolf_howl_clip...mp4">
    </video>
  </div>
</div>
```

错误点：

- 中间 `data-hf-sequence` 没有 `data-start` 和 `data-duration`；
- `<video>` 没有 `data-start`；
- 有边界的 12 秒视频被嵌套在一个无边界 sequence 中；
- 工具调用记录中没有在 `write_file` 后执行 `lint_composition`。

### 3. `submit_render` 没有传入 24 秒

第一次 `submit_render` 参数只有：

```json
{
  "fps": 24,
  "quality": "high",
  "html_file": "composition.html",
  "output_name": "Wolf Howl - Final",
  "output_resolution": "portrait"
}
```

调用方没有传 `duration`，所以渲染器只能从 HTML 结构推导总时长。它最终输出了 24 秒。

外层 composition 和内层 video 各声明 12 秒，结果恰好是 24 秒。**强推断**是渲染器对无边界 `data-hf-sequence` 的时长推导将二者累计或重复计入；该内部算法仍需用 render service 源码或同结构最小复现确认，不能写成已直接证明的实现细节。

### 4. 渲染后已经检测到 24 秒，但仍继续交付

`query_render` 返回 `done` 后，Agent 调用 `media_probe(mode="audio")`，工具明确返回：

```text
duration = 24
```

随后 Agent仍将任务标记为完成并调用 `show_final_video`。因此还有一个独立的后置校验缺陷：工具已经暴露时长翻倍，但工作流没有把实际时长与目标 12 秒比较。

### 5. 后续修复印证 wrapper 是故障点

用户反馈后，Agent读取原 HTML 并明确判断 `data-hf-sequence` wrapper 缺少时间属性会导致内部视频无法被渲染器正确驱动。随后 `edit_file` 将 `<video>` 移出 wrapper，并补上：

```html
data-start="0" data-duration="12"
```

第二次成片恢复为 `12.010667s`。这强力支持首次无边界 wrapper 是 24 秒与视频层异常的直接工具层触发点。

### 工具层根因总结

```text
media_probe 得到正确的 12.041667s
  -> write_file 写入无时间边界的 data-hf-sequence
  -> 未调用 lint_composition 拦截
  -> submit_render 未显式传 duration，按错误 HTML 推导时间轴
  -> 输出 24s
  -> media_probe 已返回 24s
  -> 工作流未比较计划时长与实际时长
  -> show_final_video 继续交付
```

## 工具责任判定与优化需求

### 这是不是工具写错了

准确结论是：**时间轴 HTML 由 Agent 写错，但工具链也没有履行校验和阻断责任。**

- `media_probe` 没有错：它已经返回正确的 `12.041667s` 可播放时长。
- `write_file` 没有擅自修改内容：它忠实写入了 Agent 生成的 HTML；直接错误是 Agent 创建了无边界的 `data-hf-sequence`。
- `lint_composition` 流程有错：写入后没有调用，因而没有发现无边界 sequence。
- `submit_render` 防御不足：它接受了时间轴不完整的 HTML，也没有要求或校验计划时长。
- 渲染后 QA 有错：`media_probe` 已返回 `24s`，工作流仍把任务标记完成并交付。

因此不能只提“修复 `write_file`”。真正需要优化的是从 Agent 编排、composition schema、提交前校验到交付后 QA 的完整契约。

### 建议需求标题

> HyperFrames 单素材时间轴契约校验与成片时长一致性保护

### P0 功能需求

#### 1. 时间轴必须只有一种明确语义

- composition、sequence 和 media node 的 `start`、`duration` 必须定义为父级相对时间，禁止未声明边界的 sequence 参与总时长推导。
- 容器用于分组时不得额外贡献时长；总时长必须由显式 composition duration 或子节点最大结束时间确定，不能把父子 duration 相加。
- 单素材场景默认生成扁平结构，不创建没有实际编排作用的 sequence wrapper。

本案例的正确结构应为：

```html
<div data-composition-id="wolf-howl" data-start="0" data-duration="12.041667">
  <video id="wolf-clip"
         data-start="0"
         data-duration="12.041667"
         src="...">
  </video>
</div>
```

#### 2. `submit_render` 内置强制 preflight

`submit_render` 不能依赖 Agent 自觉调用 lint。提交时必须自动执行同等校验，并在以下情况硬失败：

- sequence 缺少可确定的 `data-start` 或 `data-duration`；
- media node 缺少时间边界；
- composition 推导时长与计划时长不一致；
- 单素材无延长意图，但成片时长超过素材时长一个渲染帧；
- 子节点结束时间越过 composition 边界。

建议返回结构化错误：

```json
{
  "code": "HF_SEQUENCE_TIME_BOUNDS_MISSING",
  "element": "div[data-hf-sequence]",
  "message": "Sequence must define or deterministically inherit start and duration"
}
```

时长不一致统一返回 `DURATION_MISMATCH`，并包含 `planned_duration`、`derived_duration`、`source_duration` 和 `fps`。

#### 3. Agent 必须执行固定状态机

```text
media_probe
  -> 生成 timeline ledger
  -> write_file / edit_file
  -> lint_composition
  -> render_frame 抽查
  -> submit_render
  -> query_render
  -> media_probe(final)
  -> duration + visual QA
  -> show_final_video
```

任一步失败均不得跳到 `show_final_video`。其中 timeline ledger 至少保存：

```text
source_video_duration = 12.041667
planned_composition_duration = 12.041667
expected_output_duration = 12.041667
allowed_tolerance = 1 / 24s
```

#### 4. 渲染后时长必须再次对账

即使 render job 返回 `done`，也必须探测最终文件并校验：

```text
abs(actual_output_duration - expected_output_duration) <= 1 / fps
```

本案例 `abs(24 - 12.041667)` 远超 `1/24s`，必须标记为失败，禁止交付。

### 工具执行正确的验收用例

| 场景 | 输入 | 正确工具行为 |
|---|---|---|
| 单素材无叠加 | 一个 12.041667s MP4 | 直接交付，不生成 composition |
| 单素材必须合成 | 一个 12.041667s MP4 | 生成扁平时间轴，输出约 12.041667s |
| 无边界 wrapper | sequence 无 start/duration | `submit_render` 自动拒绝，返回 `HF_SEQUENCE_TIME_BOUNDS_MISSING` |
| 父子均为 12s | composition 12s、video 12s | 总时长仍为 12s，不得累计为 24s |
| 渲染结果 24s | 预期 12.041667s | 后置对账失败，返回 `DURATION_MISMATCH`，不得交付 |
| Agent 漏调 lint | 直接调用 `submit_render` | submit 内置 preflight 仍能阻断错误 HTML |

### 建议监控指标

- `render_duration_mismatch_rate`：成片与计划时长不一致比例；
- `sequence_bounds_validation_failure_rate`：无边界时间轴被拦截比例；
- `render_done_but_qa_failed_rate`：渲染成功但 QA 失败比例；
- `single_asset_composition_rate`：本可直出的单素材仍进入 HTML 合成的比例；
- `static_fallback_rate`：动态素材被静态降级的比例。

首期上线后应重点观察 `single_asset_composition_rate` 是否下降，以及是否还出现源素材约 12 秒但成片约 24 秒的同类异常。

### 下游故障：最终合成没有得到有效的动态视频帧

黑屏由 Motion/HTML 合成中的动态视频层触发。渲染任务完成时，目标时间点上的 `<video>` 没有向页面提供有效像素，最终画布只剩黑色背景。

结合调用顺序，视频层失败仍可能涉及媒体就绪或 seek 同步缺失：渲染器在 `loadeddata`、`seeked` 或解码后的下一次绘制完成之前抓取画面。签名 URL 加载、跨域配置、编码兼容性也可能触发同一症状。但 Metabase 新证据已经确认总时间轴错误，不能把 seek/解码描述为唯一原因。

> 证据等级：错误进入最终合成为 **confirmed**；动态视频层在该合成中没有产出有效像素为 **confirmed**；具体是加载、CORS、解码还是 seek 的哪一个底层事件失败为 **strongly supported**，仍需在渲染器中记录媒体事件或使用同一 HTML 复现才能进一步细分。

### 流程根因：任务成功、HLS 成功与视觉成功没有分开

`submit_render` / `query_render` 的 `done` 和 HLS `ready` 只证明渲染、封装或转码任务完成，没有证明输出画面非黑、素材加载成功、主体可见或时长正确。流程在第一次交付前没有检查时长一致性，也没有检查首帧、中间帧和尾帧。

### 恢复策略缺陷：静默降级为静态图

发现问题后，系统替换了动态视频层，而不是让动态媒体渲染失败并修复。这个降级改变了交付语义，却没有要求用户确认，也没有验证最终结果仍包含运动。

## 因果链

```text
源视频成功生成且满足直出条件
  -> 路由错误：进入不必要的最终成片合成
  -> composition.html 引用动态视频
  -> 总时间轴错误：12.097 秒源素材被配置/输出为 24 秒
  -> 动态视频层未贡献有效画面
  -> 黑色画布被正常编码
  -> render job 返回 done，HLS 返回 ready
  -> 缺少时长一致性、黑帧与运动校验
  -> 黑屏视频被交付
  -> 后续用静态图绕过，未修复动态媒体链路
```

## 解决方案

### P0：增加单素材直出路由

Owner：Assembly/router。

当满足以下条件时，跳过 HTML/Motion 合成，直接交付源 MP4：

```text
visual_clip_count == 1
&& 无 DOM 字幕/文字/贴图
&& 无多层视觉布局、转场或拼接
&& 源画幅、时长和编码符合交付要求
=> direct_media_delivery
```

若只需配乐、旁白或音效，使用媒体音频混合/替换并保留源视频画面，不进入 HTML 重渲染。

### P0：增加合成时长一致性门禁

Owner：Assembly / Motion submit validator。

- 单素材合成时，默认 `composition_duration = video_playable_seconds`。
- 如果没有明确的留白、循环或延长需求，禁止成片时长超过源视频一个渲染帧。
- 提交渲染前比较 `planned_duration`、`composition_duration`、`video_playable_seconds`。
- 本案例应在 `12.097007s -> 24s` 时直接返回 `DURATION_MISMATCH`，不得提交渲染。

### P0：强制 lint composition 时间轴结构

Owner：Motion skill / `submit_render` preflight。

- `write_file` 或 `edit_file` 后必须调用 `lint_composition`。
- 每个 `data-hf-sequence` 必须声明可确定的 `data-start` 和 `data-duration`，或由 schema 明确定义继承规则。
- sequence 内的 `<video>` 必须有 `data-start`、`data-duration`，并满足 `data-duration <= video_playable_seconds - data-media-start`。
- `submit_render` 不得接受含无边界 sequence 的 HTML。

### P1：给必须合成的动态媒体增加确定性的就绪门禁

Owner：Motion/HTML renderer。

每次采样或编码一个时间点前，对所有可见 `<video>`：

1. 调用 `load()` 并等待 `loadedmetadata`。
2. 等待 `readyState >= HAVE_CURRENT_DATA`；生产环境建议要求 `HAVE_FUTURE_DATA`。
3. 将目标时间限制在实际可播放时长内。
4. 设置 `currentTime`，等待 `seeked`。
5. 再等待至少两个 `requestAnimationFrame`，保证解码帧已经绘制。
6. 监听 `error`、`stalled`、`abort` 和超时；任何失败都必须终止渲染，不能输出黑色视频。

示意代码：

```js
async function prepareVideoFrame(video, targetTime, fps, timeoutMs = 15000) {
  const wait = (event) => onceWithTimeout(video, event, timeoutMs);

  video.crossOrigin = "anonymous";
  video.preload = "auto";
  video.load();

  if (video.readyState < HTMLMediaElement.HAVE_METADATA) {
    await wait("loadedmetadata");
  }
  if (video.readyState < HTMLMediaElement.HAVE_CURRENT_DATA) {
    await wait("loadeddata");
  }

  const lastPlayableFrame = Math.max(0, video.duration - 1 / fps);
  video.currentTime = Math.min(Math.max(targetTime, 0), lastPlayableFrame);
  await wait("seeked");

  await new Promise(requestAnimationFrame);
  await new Promise(requestAnimationFrame);

  if (video.readyState < HTMLMediaElement.HAVE_CURRENT_DATA || video.videoWidth === 0) {
    throw new Error("video_frame_unavailable");
  }
}
```

### P1：把媒体加载失败变成明确失败

Owner：Motion render API。

- 返回结构化错误，例如 `MEDIA_LOAD_FAILED`、`MEDIA_DECODE_FAILED`、`MEDIA_SEEK_TIMEOUT`。
- 错误必须包含元素 ID、资源资产 ID、目标时间和媒体事件，不记录完整签名 URL。
- 禁止在媒体失败时继续编码黑色背景并返回 `done`。

### P1：增加交付前黑帧门禁

Owner：最终交付/QA 层。

渲染完成后抽取至少三个时间点：`0.5s`、`50%`、`结束前 0.5s`。检测：

- 平均亮度和非黑像素占比；
- 连续黑帧比例；
- 帧间差异，防止动态视频被静态图替代；
- 对主体明确的任务，验证主体在至少一个采样帧可见。

不满足条件时返回 `VISUAL_VALIDATION_FAILED`，不得调用最终交付。

HLS `ready` 之后仍必须执行该视觉门禁；HLS 状态不能替代内容 QA。

建议阈值需要用正常夜景样本校准，不能仅使用固定亮度阈值，否则会误伤合法的暗场视频。应组合使用亮度、边缘、局部对比度和帧间差异。

### P2：规范输入媒体

Owner：Assembly/media preprocessing。

动态素材进入 HTML 合成前统一规范为：

- MP4；
- H.264 视频；
- `yuv420p`；
- CFR；
- `faststart`；
- 音频为 AAC 或移除不需要的音轨。

用 `video_playable_seconds` 而不是容器 duration 约束 `data-duration`，并确保结束时间至少预留一个渲染帧。

### P2：验证远程资源契约

Owner：资产服务和渲染器。

- 资源返回正确的 `Content-Type: video/mp4`；
- 支持 Range 请求；
- CORS 允许渲染页面来源；
- 签名 URL 的有效期覆盖排队、加载和完整渲染；
- HTML 中引用发布后的稳定资产，而不是即将过期的临时链接。

### P1：禁止静默静态降级

Owner：Motion skill / workflow policy。

当用户要求视频运动而动态素材失败时：

- 自动重试或走兼容转码；
- 重试仍失败则明确报错并停止；
- 只有用户明确接受时才允许以静态帧替代；
- 降级后必须将交付标记为 `static_fallback`，不能标记为正常动态视频成功。

## 回归测试

| 用例 | 输入条件 | 期望结果 |
|---|---|---|
| 单素材直出 | 无合成需求 | 不启动 HTML/Motion，源 MP4 直接交付 |
| 合成时长不一致 | 12.097s 源素材、24s composition，无显式延长需求 | 返回 `DURATION_MISMATCH`，不提交渲染 |
| 无边界 sequence | `data-hf-sequence` 缺少 start/duration | `lint_composition` 失败，禁止 `submit_render` |
| 正常远程 MP4 | 必须合成且加载正常，可 seek | 首、中、尾帧均有有效画面 |
| 慢速资源 | `loadeddata` 延迟 | 渲染等待，不提前截取黑帧 |
| CORS 拒绝 | 视频不可跨域读取 | 返回 `MEDIA_LOAD_FAILED`，不产出成片 |
| 不兼容编码 | 浏览器无法解码 | 预处理转码或返回 `MEDIA_DECODE_FAILED` |
| seek 超时 | 目标帧无法到达 | 返回 `MEDIA_SEEK_TIMEOUT` |
| 暗场视频 | 合法夜景 | QA 不因平均亮度低而误判黑屏 |
| 静态替代 | 动态层被换成图片 | 帧间差异校验失败或标记为显式降级 |
| 接近片尾 | 目标时间靠近 duration | 时间被限制到最后可播放帧，无尾部黑帧 |

## 验收标准

修复只有在以下条件全部满足时才算完成：

1. 本案例按直出路径交付源 MP4，不启动 HTML/Motion 合成。
2. 必须合成的单素材任务中，成片时长与 `video_playable_seconds` 一致，除非有显式延长需求。
3. 无边界 `data-hf-sequence` 在 `submit_render` 前被 lint 拒绝。
4. 首、中、尾采样帧非黑，且至少两个采样帧存在可观测运动差异。
5. 加载、解码或 seek 失败时任务明确失败，不生成可交付文件。
6. HLS `ready` 不会绕过时长和视觉 QA。
7. 日志能区分路由、时间轴结构、时长、资源加载、解码、seek 和视觉 QA 失败，但不泄露签名 URL。
8. 夜景等低亮度合法内容通过回归测试。

## 修复顺序建议

1. 先修正单素材直出路由，移除本案例不必要的合成步骤。
2. 强制 `lint_composition` 并拒绝无边界 `data-hf-sequence`。
3. 增加合成时长一致性门禁，阻止 `12.097s -> 24s` 这类异常进入渲染或交付。
4. 为仍需合成的任务增加黑帧门禁，且 HLS `ready` 后仍必须执行视觉 QA。
5. 修复渲染器媒体事件与 seek 同步，并增加输入转码和远程资源契约校验。
6. 移除静默静态降级，补齐结构化错误与监控指标。
