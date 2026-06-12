# Case 71316294507 成片问题分析

- **项目 / 会话 ID**：71316294507
- **Langfuse trace**：`pexo:71316294507`（`trace-1-c7801d26.json`）
- **用户需求**：使用 `02_education/04_coding_screen.png` 生成编程教程视频；在 00:15-00:20 加 50% 黑色半透明全屏遮罩，用矩形镂空高亮中间一行代码，并在右侧弹出 tooltip：`This is a list comprehension, equivalent to a for-loop with append`.
- **核心问题**：原始截图里没有 list comprehension 或 for-loop，但 Agent 仍高亮了 `with path.open('r', encoding='utf-8') as f:`，并把 tooltip 改成 list comprehension 说明，导致成片的“高亮代码”和“解释文案”语义不一致。除此之外，成片还存在底图静止、方框校准不准、无音频音效三个交付问题。

## 一、关键事实链

### 1. 图像分析阶段已经发现需求无法被素材直接满足

Trace 中第一次 `analyze_file_content` 明确返回：

- 图片为 3840 x 2160 的 VS Code 截图；
- Python 文件是 `helpers.py`；
- 可见代码主要是 JSON 读写工具函数；
- **没有 list comprehension，也没有 for-loop**；
- 可见循环/推导式相关内容不存在。

这一步已经足以触发“向用户澄清 / 换图 / 修改高亮目标 / 生成补充代码画面”的分支。

### 2. Agent 承认没有目标代码，但自行替代为 line 10

模型随后写道：

> There's no list comprehension visible, but I'll pick a meaningful line — line 10 (`with path.open('r', encoding='utf-8') as f:`) as the "highlighted" line...

这一步是根因转折点：Agent 把“不存在目标对象”当成“可自行找一个教学点替代”，没有征求用户确认，也没有把 tooltip 语义同步改成 `context manager`。

### 3. 预览校验只验证了视觉对齐，没有验证语义一致

后续多次 `render_frame` / `analyze_file_content` 的重点是：

- 是否有半透明遮罩；
- 矩形镂空是否对准单行；
- tooltip 是否在右侧；
- 最终确认为高亮 **line 10**：`with path.open('r', encoding='utf-8') as f:`

但这些校验没有检查：**line 10 是否真的是 list comprehension**。

### 4. 最终 tooltip 被改回用户指定文案，造成错误成片

早期 HTML 里 tooltip 曾是正确匹配 line 10 的解释：

`This is a context manager for safe file handling...`

但最后一次 `edit_file` 把 tooltip 改为：

`This is a list comprehension, equivalent to a for-loop with append.`

同时高亮仍是 line 10 `with path.open(...) as f:`。因此最终视频视觉上完成了遮罩/镂空/弹窗，但内容教学是错的。

### 5. 渲染侧本身成功

`submit_render` 返回 job `aslm5xat`，最终 `query_render` 成功：

- 输出文件：`/projects/71316294507/workspace/assets/coding_tutorial_final.mp4`
- 状态：`done`
- 渲染耗时：`94102ms`

所以问题不是 HyperFrames 渲染失败，而是 Agent 规划与验收逻辑失败。

## 二、问题表现

成片中 00:15-00:20 的效果大概率为：

- 黑色半透明遮罩覆盖全屏；
- 矩形镂空高亮了 `with path.open('r', encoding='utf-8') as f:`；
- 右侧 tooltip 显示 “This is a list comprehension, equivalent to a for-loop with append”；
- 用户会看到：**被高亮的代码不是 list comprehension，但解释说它是 list comprehension**。
- 全片底图只是原始截图，除遮罩、方框、tooltip 外没有镜头运动或代码演示变化；
- 方框位置经历多次手动改坐标，最终仅被校验为“大致高亮 line 10”，没有精确贴合用户想要的目标代码区域；
- Composition 没有添加 VO、BGM、点击音、弹窗音效或提示音，最终视频是静音教程动画。

这是典型的教学视频内容错配，属于高优先级语义错误。

## 三、根因分析

### 根因 1：缺少“目标对象不存在”的阻断规则

当素材中不存在用户指定的视觉/语义对象时，Agent 没有进入澄清或替代方案确认，而是直接自行选择了一个“看起来适合讲解”的代码行。

### 根因 2：视觉 QA 和语义 QA 脱节

后续校验只证明“镂空对准了某行代码”，没有证明“被高亮代码符合 tooltip 文案”。对教程类、代码类视频，语义一致性应是必检项。

### 根因 3：用户文案优先级被机械执行

Agent 最后为了满足用户给出的 tooltip 原文，把已经正确匹配 line 10 的 `context manager` 文案替换成 `list comprehension` 文案，但没有同步更换画面素材或代码内容。

### 根因 4：交付说明进一步固化错误

最终回复中写明高亮 line 10，同时又说 tooltip 是 list comprehension。这说明 Agent 在交付前已经拥有冲突事实，但没有拦截。

### 根因 5：底图被当成静态背景，没有做教程视频化处理

HTML 中核心画面是一个全屏 `<img id="bg-img">`，时间轴只驱动：

- 15s 遮罩淡入；
- 15.2s 高亮框出现；
- 15.5s tooltip 滑入；
- 19.5s 后这些元素淡出；
- 24.3s 整体淡出。

底图本身没有 pan / zoom / cursor / code typing / scroll / line reveal 等运动，所以用户看到的是“静态截图上叠层”，不是完整的编程教程视频。

### 根因 6：方框定位依赖人工估算和二次视觉描述，没有坐标级校准

Trace 中先按 3840x2160 到 1920x1080 的 0.5 缩放估算 line 10，结果预览高亮到了 line 7-8；随后又通过 `preview_baseline` 反推 line 10 的 Y 坐标，再手动改为 `y=358, x=212, w=740, h=34`。这个流程有两个问题：

- 原始目标 “list comprehension” 不存在，所以即使框准 line 10，也不符合用户目标；
- 高亮框没有基于 OCR / DOM / 代码行 bbox 做精确计算，只是靠截图视觉反馈迭代，容易出现上下偏移、包含空行或框宽不合适。

### 根因 7：未进行音频设计

本次走的是 HyperFrames 静态 HTML composition，Trace 中没有 TTS、BGM、SFX 生成，也没有 `<audio>` 元素或混音/装配工具调用。Agent 的 todo 只包含分析图像、读 HyperFrames、写 HTML、渲染交付，没有音频任务，因此最终无声。

## 四、解决方案

### A. 对当前 case 的直接修复

语义层二选一：

1. **保持原图不变**：把 tooltip 改为 context manager 解释，例如：
   `This is a context manager, used to open and safely close a file.`

2. **保持用户原 tooltip 不变**：需要先修改画面内容，让截图中间实际出现 list comprehension，例如：
   `squares = [x * x for x in numbers]`
   然后重新定位镂空到这行代码。

推荐方案 2，因为用户明确指定了 list comprehension 教学点。

同时补齐视频化和音频：

1. **底图动起来**
   - 0-5s：轻微 zoom-in 到编辑器区域；
   - 5-12s：加入鼠标指针或代码行扫描高亮；
   - 12-15s：聚焦到目标代码行；
   - 15-20s：遮罩、镂空、tooltip 出现；
   - 20-25s：退出高亮并收尾。

2. **方框精确校准**
   - 若继续用截图，先对目标代码行做 OCR / bbox 定位；
   - 方框应只覆盖目标行文本和少量 padding，不跨空行；
   - 最终 render 前必须重新检查最终帧，而不是只检查改文案前的帧。

3. **添加音频音效**
   - 添加短 VO，例如 “This line is a list comprehension...”；
   - 添加轻 BGM 或低音量 ambient tech bed；
   - 遮罩出现、框选、tooltip 弹出分别添加 subtle whoosh / click / pop；
   - 音频覆盖 0-25s，避免静音成片。

### B. Agent 规则侧修复

1. **素材语义预检**
   - 如果用户要求高亮某类对象（list comprehension、函数、变量、按钮、Logo 等），图像分析必须返回 `target_found: true/false`。
   - `target_found=false` 时禁止直接进入制作，必须澄清或提供明确替代方案。

2. **替代目标需要用户确认**
   - 允许 Agent 提议：“当前截图没有 list comprehension，我可以改为讲 context manager，或生成/编辑一张包含 list comprehension 的代码截图。”
   - 不能默认替代。

3. **教程类视频增加语义验收**
   - 交付前检查：`highlighted_code_semantics` 是否匹配 `tooltip_claim`。
   - 若 tooltip 声称 list comprehension，但高亮代码不包含 `[...] for ... in ...` 等结构，应阻断。

4. **最终 render 前必须做最后一帧语义复核**
   - 本 case 在 tooltip 文案替换后没有再次对最终帧做“文案 + 高亮代码”一致性检查。
   - 规则应要求最后一次关键文案修改后重新 `render_frame` 并校验。

5. **交付说明冲突检测**
   - 若最终说明同时出现“高亮 `with path.open`”和“这是 list comprehension”，应触发自检失败。

6. **教程视频必须有 motion/audio plan**
   - 用户说“tutorial video”时，不能只做静态截图叠层；
   - 除非用户明确要求静音，否则需要至少规划 VO/BGM/SFX 之一；
   - 如果不做音频，应在交付前说明并征得确认。

## 五、结论

本 case 的成片问题是：**Agent 已识别原素材没有 list comprehension，却绕过需求不可满足检查，自行高亮了 context manager 行，并在最终 tooltip 中仍使用 list comprehension 文案**。此外，Agent 把教程视频做成了“静态截图 + 遮罩弹窗”，没有底图运动，没有坐标级方框校准，也没有音频音效。渲染链路成功，问题发生在素材理解、需求澄清、动效/音频规划、方框校准和语义一致性验收多个环节。
