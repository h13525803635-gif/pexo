# Pexo 项目 53505424835：图片请求被交付为 1 秒静止视频

## 分析范围与数据完整性

- 项目 ID：`53505424835`
- Langfuse Trace ID：`a86c59eee99793c138985816a2407171`
- Trace 开始时间：`2026-07-28T09:01:24.541Z`
- 发现 Trace 数：1
- 成功拉取 Trace 数：1
- 拉取失败数：0
- 已检查 Observation 数：180
- 产物检查：根因判断不需要额外检查媒体文件；工具结果和 composition 元数据已直接证明最终产物类型及其时长

原始 Langfuse 导出包含临时签名资源 URL 及可能敏感的执行上下文，因此未提交至 GitHub。

## 用户可见问题

用户明确要求制作一张简单的 16:9 Transformer Attention 教育**图片**，最终收到的却是 `Transformer_Attention_Explainer.mp4`：一段只有静止画面的 1 秒视频。

## 一句话根因

**已确认：** Agent 为纯图片请求选择了 HTML-to-video 工作流。虽然期间已成功生成并质检通过用户需要的 PNG，但 Agent 没有交付该图片，而是继续执行视频专用的 `submit_render -> show_final_video` 路径；同时 composition 被显式设置为 1 秒，所有动画又在渲染前被移除，因此最终必然得到 1 秒静止 MP4。

## 因果链

```text
用户明确请求图片
  -> 未锁定交付类型，错误选择 motion-skill
  -> HTML composition 被显式设置为 data-duration="1"
  -> t=0 预览暴露动画初始状态不可见问题
  -> 为保证图片在 t=0 可见，移除全部入场动画
  -> 正确 PNG 已生成并通过质量检查
  -> PNG 未被交付
  -> submit_render 生成 MP4
  -> show_final_video 交付 1 秒静止视频
```

## 时间线证据

| 时间（UTC） | Observation | 证据 | 含义 |
|---|---|---|---|
| 09:01:24 | Trace 输入 | 用户要求制作一张“16:9 educational image” | 用户要求的交付类型明确是图片。 |
| 09:01:29 | `read_file`，obs `a1df2155161744d2` | Agent 加载 `motion-skill`；该技能定义为 HTML-to-video composition，标准流程以 MP4 交付结束 | 这是最早使故障成为必然的错误路由决策。 |
| 09:04:44 | `write_file`，obs `e4e656c77aeb0b56` | composition 根节点和场景节点都设置了 `data-duration="1"` | 1 秒时长由 Agent 显式写入，并非渲染器意外截断。 |
| 09:04:51 | `render_frame`，obs `8867d973e3b2f2fa` | 在 `time: 0` 生成 `attn_preview_frame.png` | 当前工作流已经具备直接生成 PNG 的能力。 |
| 09:04:59 | `analyze_file_content`，obs `bda2a58e67df768f` | 质量检查发现大部分预期内容在动画初始状态不可见 | 暴露的是取帧时机问题，而不是图片生成失败。 |
| 09:05:23 | `edit_file`，obs `6776938b39c21d29` | Agent 用静态 timeline 替换入场动画，并明确写下 `static frame - no entrance animation` | 此操作保证后续视频不包含任何运动。 |
| 09:05:26 | `render_frame`，obs `1321124afb3e38be` | 成功生成 `attn_frame_v2.png` | 视频渲染前已经存在可交付的正确图片。 |
| 09:05:33 | `analyze_file_content`，obs `b8ecec6a1f06498c` | 质量检查确认标题、步骤、热力图、说明、公式、色彩和可读性全部符合预期 | 用户要求的图片已经完成并可直接交付。 |
| 09:05:54 | `submit_render`，obs `43484a4f899f4864` | 以 30 fps 提交 HTML composition，名称为 `Transformer Attention Explainer` | 工作流不必要地把已验收图片转换成视频。 |
| 09:06:16 | `show_final_video`，obs `b7e0d0a85f47e790` | 交付 `Transformer_Attention_Explainer.mp4` | 最终产物类型与用户请求直接冲突。 |

## 触发原因、传播路径与检测缺口

### 触发原因

Agent 把“educational image”理解为可以使用 motion composition 制作，但没有先持久化 `deliverable_type=image`。使用 HTML 制图本身没有问题，问题在于制作方式被错误地等同于最终交付类型。

### 传播路径

所选技能的标准完成路径要求依次调用 `submit_render` 和 `show_final_video`。进入该路径后，Agent 将成功生成的 PNG 仅视为预览和质量检查帧，而没有把它视为最终交付物。

### 检测缺口

最终交付前不存在“用户请求类型与产物类型一致性”校验。Lint 和视觉质量检查只验证 composition 与画面内容，不验证输出格式。任务列表虽然写着“render the final image”，实际最终工具仍然是仅用于视频的 `show_final_video`，这种状态矛盾没有触发阻断。

## 已排除的其他解释

- **图片生成失败：已排除。** `attn_frame_v2.png` 已成功生成，并通过详细视觉质量检查。
- **渲染器意外把长视频截成 1 秒：已排除。** composition 根节点和场景节点都显式声明了 1 秒时长。
- **导出时动画失效：已排除为主要原因。** Agent 在导出前主动删除入场动画，把 composition 改成了静止画面。
- **旧版本或错误 revision 被交付：无证据支持。** 最终 MP4 直接由 PNG 验收后的最新静态 HTML 生成。

## 解决方案

### 1. 在意图识别阶段锁定交付类型

- **负责人：** 意图路由器 / 任务规划器
- **触发条件：** 用户明确使用 `image`、`picture`、`poster`、`infographic`、`still`、`PNG`、`JPG`、`WebP` 等词，且没有提出动画或视频要求
- **正确行为：** 持久化 `deliverable_type=image`，路由到支持图片交付的工作流
- **防护规则：** 下游技能可以改变制作方式，但未经用户明确同意不得改变交付类型

### 2. 为 HTML composition 增加静态图片分支

- **负责人：** `motion-skill` 或后续承接 HTML composition 的技能
- **触发条件：** `deliverable_type=image`
- **正确行为：** 编写并 lint HTML，在预期最终状态调用 `render_frame`，完成图片质量检查后直接交付 PNG
- **防护规则：** 图片任务禁止调用 `submit_render`、`query_render` 和 `show_final_video`

### 3. 增加最终交付契约校验

- **负责人：** 最终交付层
- **触发条件：** 任意 `show_final_*` 调用之前
- **正确行为：** 比较用户请求的交付类型与文件扩展名、MIME 类型
- **防护规则：** 当 `deliverable_type=image` 时拒绝 `.mp4`、`.mov`、`.webm`；当 `deliverable_type=video` 时拒绝图片文件

### 4. 将图片质检成功设为终止状态

- **负责人：** Agent 工作流 / 任务状态机
- **触发条件：** 图片产物已经存在，并通过所有用户要求的视觉断言
- **正确行为：** 将图片任务标记为完成，并交付通过质检的同一资产
- **防护规则：** 后续格式转换必须由用户请求或明确的兼容性要求驱动

## 回归检查

1. 输入“制作一张 16:9 教育图片”时，最终调用图片交付工具，产物 MIME 类型必须以 `image/` 开头。
2. HTML 制作的信息图可以调用 `render_frame`，但不得调用 `submit_render`。
3. `render_frame` 返回的 PNG 通过质量检查后，最终必须交付同一个 asset ID。
4. `deliverable_type=image` 与 MP4 产物不一致时，必须在交付前失败并阻断。
5. 用户增加“彩色”“步骤”“attention grid”等内容要求时，不得因此推断用户需要动态视频。
6. 用户明确要求“把信息图做成动画”时，应正常进入视频路径，不受图片专用防护规则影响。

## 结论

这是一次路由和交付契约故障，不是内容生成故障。最早且最有效的修复点是在意图识别阶段锁定用户要求的产物类型；最强的兜底是在最终交付前执行 MIME/类型断言，从机制上禁止为明确的图片请求交付视频。
