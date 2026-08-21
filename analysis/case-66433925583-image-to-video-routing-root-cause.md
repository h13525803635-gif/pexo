# Case 66433925583: 图片内容动画被误路由为 Motion 排版动画

## 结论

项目 `66433925583` 的顶层技能选择并不是唯一问题。真正导致错误成片的最早决策发生在 Script 阶段：Agent 把用户要求的“每个画面上的内容得动起来”解释成卡片、标题、徽章和 SVG 图层的 DOM/Motion 入场动画，并将生产路线锁定为“pure dynamic typography/card animation，no AI video generation needed”。

但用户实际需要的是：使用已经生成的 8 张模块图片作为参考图，分别调用 image/reference-to-video，让图片中的人物、设备和场景产生语义动作，再拼接成视频。错误路线最终生成了静态卡片切换和排版动效，因此用户认为成片像 PPT。

准确分类为：**Script 生产路线判断错误 + 语义动作意图未结构化 + 错误状态被会话摘要持续传播**。Motion 主要是在执行上游的错误 handoff。

## 数据范围与完整性

- 项目：`66433925583`
- 项目名称：`图片模块介绍`
- 创建时间：2026-08-11 11:40:19（Asia/Shanghai）
- Agent schema：`0.1.1`
- 数据源：Metabase `pg-server` 项目与消息记录；Langfuse `pexo:66433925583` traces
- 已发现执行 trace：14 个
- 重点 trace：
  - `807882b0ae29aefee9d327010528731b`：Script、图片卡片规划与生成
  - `1464526d387d11e78210c823ebcae03f`：错误的 Motion/DOM 重建和渲染
  - `0f4eb708e570d475d5d80023aa5ba270`：用户纠正后切换到 Generation/reference-to-video

结论置信度：**confirmed**。关键生产路线、工具调用和用户纠正均有直接 trace 证据。

## 用户意图

用户最初要求：

> 基于这张图做一条 15 秒的视频，保留图片风格，核心内容是图片里各个模块的介绍。

后续约束进一步明确：

- 9:16 竖版。
- 不显示原始整张信息图。
- 将原图中的 8 个模块拆成独立画面。
- 每个画面的内容需要动起来，不能只是 PPT 切换。
- 使用前面已经生成的模块图片来生成视频素材。

最后两条已经明确指向 image/reference-to-video，而不是纯 DOM 排版动画。

## 实际调用链

### 1. Brainstorm 先提出了错误的整图镜头方案

Agent 最初建议从整张信息图俯视并逐模块拉近。用户随即纠正：不要镜头拉近，不要出现整张原图，要把模块拆开。

这一步不是最终路由错误，但说明 Agent 没有从“模块介绍”中稳定提取素材拆分约束。

### 2. Agent 生成了 8 张独立模块卡片

Agent 使用 GPT Image 2 image-to-image，生成并迭代了 8 张 9:16 模块卡片。这一阶段基本符合用户要求，且用户确认了卡片风格。

### 3. Script 将生产路线错误锁成 Motion/DOM

会话 handoff 中出现了决定性的错误状态：

> Pure dynamic typography/card animation (no AI video generation needed - motion design approach)

该结论没有来自用户确认。用户只确认了视觉风格和卡片布局，并没有同意以排版动画代替图片内容动画。

### 4. Motion 使用静态图片和 DOM/SVG 动画制作视频

Agent 读取 `motion-skill`，写入 `composition.html`，采用两种实现：

1. 将 8 张静态图片放入 sequence，做交叉淡化、滑入和缩放。
2. 后续又重画 DOM/SVG 卡片，让徽章、插画、标题和副标题分别入场。

这些实现可以让“图层”运动，但没有让图片中的人物、安检门、摄像头、机器人等主体执行语义动作。产物因此仍然表现为动态 PPT。

### 5. 用户明确纠正后才进入 Generation

用户明确指出：

> 不是让你用 motion 重新生成，是用你前面生成的图片来生成视频素材。

此后 Agent 才读取 `generation-skill` 和视频模型路由文档，并对 8 张模块图分别调用 Seedance `reference2video`。典型调用参数为：

```json
{
  "provider": "seedance",
  "model": "doubao-seedance-2-0-260128",
  "mode": "reference2video",
  "image_list": [
    { "file": ".../card_02_health_checks_20260811T035846_40347fab.png" }
  ],
  "aspect_ratio": "9:16",
  "duration": "5",
  "sound": "off"
}
```

这证明正确路线在系统中可用，失败不是模型能力缺失，而是前置路线判断错误。

## 根因分析

### 根因 1：没有区分排版运动和语义主体运动

“内容动起来”至少有两种完全不同的含义：

| 动作类型 | 例子 | 正确路线 |
|---|---|---|
| 排版/图层运动 | 标题淡入、卡片滑入、徽章弹出、图标缩放 | Motion / HyperFrames |
| 语义主体运动 | 人穿过安检门、摄像头扫描、行李移动、机器人工作 | Generation image/reference-to-video |

Script 没有把这一差异结构化，直接选择了第一类。

### 根因 2：视觉风格确认被错误扩展为生产路线确认

用户确认的是蓝白线稿风格、卡片布局和文案密度。Agent 将这个确认错误地扩展为“同意使用纯 Motion 动画”，违反了确认边界。

### 根因 3：会话摘要固化了错误路线

会话压缩摘要将以下内容写成已确认事实：

> no AI video generation needed

后续 Agent 将摘要当作持久状态继续执行，即使用户已经反馈“不是 PPT 切换”，仍优先修补 Motion composition，而没有立即重算生产路线。

### 根因 4：Motion 缺少上游路线一致性门禁

Motion 接收到静态图片和动态视频需求后，没有检查：

- 用户是否要求图片内部主体动作。
- Script 是否提供 `semantic_subject_motion`。
- 是否存在 Generation 产出的动态视觉素材。

因此 Motion 可以用静态图片或重画 SVG 替代本应生成的视频主体。

### 根因 5：交付验证只检查“有没有动”，没有检查“什么在动”

QC 验证了图标、徽章、标题、副标题是否显示和入场，却没有验证用户要求的动作，例如：

- 旅客是否穿过安检门。
- 人脸验证是否完成。
- 摄像头扫描光线和识别框是否出现。
- 机器人和行李是否发生可理解的运动。

因此技术渲染成功被误判为用户意图满足。

## 正确解决方案

### 1. 在 Script 中增加动作语义分类

Script handoff 应增加：

```yaml
motion_intent:
  layer_motion: true
  semantic_subject_motion: true
  required_actions:
    - traveler_passes_health_scanner
    - face_verification_succeeds
    - traveler_enters_camera_scan_zone
    - robot_moves_luggage
```

路由规则：

- `semantic_subject_motion: true`：必须进入 Generation。
- 只有 `layer_motion: true` 且 `semantic_subject_motion: false`：可以选择 `hf_dom_only`。
- 两者同时存在：Generation 负责主视觉，Motion 只负责文字、品牌、转场和最终包装。

### 2. 为已有图片的视频请求增加硬路由规则

当同时满足以下条件时，默认路由到 image/reference-to-video：

- 已有用户图片或已生成图片。
- 用户要求“让图片/人物/物体/场景动起来”。
- 用户描述了主体行为，或明确否定 PPT、轮播、静态切换。

建议规则：

```text
IF source_images_exist
AND semantic_motion_requested
THEN visual_route = aigc_image_to_video
AND hf_dom_only = forbidden
```

### 3. 限制摘要写入未确认的生产结论

会话摘要应区分：

- `user_confirmed`
- `agent_proposed`
- `rejected`
- `superseded`

“no AI video generation needed”没有用户确认，不应写入 `confirmed creative direction`。用户出现“不是 Motion”“不是 PPT”“用图片生成视频素材”等纠正后，旧路线必须标记为 `superseded`。

### 4. 给 Motion 增加阻断门禁

Motion 在写 HTML 前应检查：

```text
IF semantic_subject_motion = true
AND no generated_video_assets
THEN BLOCK and route to generation-skill
```

此外，当主视觉全部是 `<img>` 或静态 SVG，而用户要求人物/物体动作时，应阻断 `submit_render`。

### 5. 将 QC 从元素可见性升级为动作验收

每个生成片段必须有动作断言，例如：

```yaml
action_assertions:
  - segment: health_checks
    assertion: traveler visibly passes through scanner
  - segment: digital_identification
    assertion: face verification changes to approved state
  - segment: movement_tracking
    assertion: traveler enters frame before scan rays and tracking box appear
```

最终交付前需要检查动作是否在对应时间窗口真实发生，不能只检查画面非空、文字清晰或图层有位移。

## 推荐的正确生产链路

```text
用户原图
  -> Image Production：生成 8 张风格一致的模块参考图
  -> Script：为每张图定义具体主体动作和时长
  -> Generation：8 次 image/reference-to-video
  -> Assembly：裁剪到目标节奏并拼接，处理 BGM
  -> Motion：仅添加必要文字、品牌和转场
  -> Action QC：逐段验证语义动作
  -> 最终交付
```

Motion 不应重新绘制或替代主视觉，只应包装 Generation 已生成的视频素材。

## 回归测试

### 路由测试

1. 输入：“让这张卡片整体滑入，标题淡入。”
   - 预期：`hf_dom_only`。

2. 输入：“让图里的人拖着行李走过安检门。”
   - 预期：image/reference-to-video，禁止纯 Motion。

3. 输入：“用刚才生成的 8 张图片做视频，每张图里的内容都要动，不要像 PPT。”
   - 预期：Generation 生成 8 个视频片段，Motion 只能作为可选包装层。

4. 输入：“图片保持静态，只让编号和文案出现。”
   - 预期：Motion。

### 状态测试

1. Agent 提议 Motion，但用户未确认。
   - 摘要不得将其写为 confirmed。

2. 用户说“不是 Motion”。
   - 原路线必须立即标记 `superseded`，下一步重新进入 Script/Generation。

### 交付门禁测试

1. `semantic_subject_motion=true` 且只有 PNG/SVG。
   - `submit_render` 必须失败。

2. 已生成视频，但动作断言不成立。
   - 禁止交付，重新生成对应片段。

## 对用户的解释口径

这次不是用户描述有问题，也不是视频模型无法完成。问题发生在生产路线判断：Agent 把“图片中的内容需要动起来”误解成“卡片和文字做入场动画”，所以用了 Motion 将静态图片包装成 MP4，结果看起来像 PPT。正确做法是把已经生成的 8 张模块图逐张送入图生视频模型，让人物、设备和场景产生具体动作，再由 Motion 负责文字和最终包装。

## 修订记录

| 日期 | 说明 |
|---|---|
| 2026-08-11 | 初版：确认 Script 生产路线误判、摘要传播和 Motion 门禁缺失 |
