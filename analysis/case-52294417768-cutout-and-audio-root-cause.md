# 项目 52294417768：首次不抠图且无声、二次只有人声无音效的根因与方案

## 结论摘要

本项目可确认的 Langfuse trace 有两次 HyperFrames render：

- 首次 render：`job_id=9ftf1tbf`，产物 `asset_id=a_6pr9F5d`，有 3 个 warning。
- 清洁重渲染：`job_id=c6jdcm67`，产物 `asset_id=a_xaefDi9`，最终 `show_final_video` 交付。

首次/清洁重渲染的问题本质一致：

1. **没有走抠图/主体归一化流程**，HTML 直接 `<img>` 引用了用户上传原图。
2. **没有生成或绑定音效/BGM/音频轨**，HTML 中没有 `<audio>`，最终也没有 `probe_media` 音频流校验。

用户描述的“第二次成片只有人声还是没有音效”，在当前成功拉取的 trace 中没有对应的第二轮用户反馈/二次修改 trace。按现象判断，第二轮大概率只补了 VO/TTS 轨或人声轨，但仍未把“黄金粒子爆炸/弹卡/震动”等 SFX 作为独立音效层加入时间线或 HTML。

## 用户需求

原始 prompt：

> On a solid purple gradient background, display `01_ecommerce/03_blind_box_toy.png` in the center at 00:00. Apply a violent shaking animation to the image from 00:02 to 00:04. At 00:04, hide the original image, play a golden particle explosion effect (screen overlay) in the same position, and simultaneously pop up a text card with a spring animation saying `Congratulations! Secret Edition Unlocked - No.088`.

用户上传图：

- `/projects/52294417768/workspace/assets/a_FPHnF4f_06_03_blind_box_toy.png`

该需求虽然没有显式写“去背景/抠图”，但语义是“在纯紫色渐变背景中展示玩具主体”，通常应理解为：用户图作为主体层叠到新背景上，不能保留原图底色/画布。

## 证据

### 1. 首次 HTML 直接引用原图，未抠图

trace 中只调用了 `get_file_info` 获取上传图 signed URL：

- `asset_id=a_FPHnF4f`
- `file_name=06_03_blind_box_toy.png`

随后 HTML 直接写入：

```html
<img id="toy-img" src="https://pexo-assets.../06_03_blind_box_toy.png" alt="Blind Box Toy" />
```

没有看到以下任一动作：

- `cutout_asset_manifest`
- re-plate / key-color plate
- canvas plate-key 抠图
- alpha/透明主体校验
- 背景移除后的新资产

而 HyperFrames skill 的规则明确要求：当用户上传产品/图标/贴纸需要作为干净主体层时，不能直接 `<img>` 原图，必须先有 cutout asset manifest 或走 re-plate/canvas key 流程。

### 2. 首次 render 有 warning，但修复范围只覆盖时序/定位，不覆盖抠图和音频

首次提交：

- `submit_render(html_file=asset://a_avz5xaD, output_name=blind_box_reveal)`
- `job_id=9ftf1tbf`
- warning：
  - `gsap_css_transform_conflict`
  - `timed_element_missing_clip_class`
  - `root_composition_missing_data_start`

后续修改只做了：

- 给 `scene-main` 加 `class="clip"`
- 给 root 加 `data-start="0"`
- 修 `#toy-wrapper` 的 CSS transform 冲突

清洁重渲染：

- `submit_render(html_file=asset://a_TE8NBnN, output_name=blind_box_reveal_v2)`
- `job_id=c6jdcm67`
- `warnings=[]`
- 最终展示：`/projects/52294417768/workspace/assets/blind_box_reveal_v2.mp4`
- `asset_id=a_xaefDi9`

这些修复能解决 HyperFrames 时序 warning，但不能解决“原图未抠图”和“无音频”。

### 3. 首次/清洁重渲染均没有音频绑定

真实工具调用列表中没有音频生产调用：

- 没有 `audio_produce`
- 没有 `music_generate`
- 没有 SFX 生成/上传
- 没有 `probe_media`

最终 HTML 中没有 `<audio>` 元素。实际交付前也没有执行“最终 MP4 必须包含 audio stream”的校验。

虽然 trace 文本里出现了 `audio_produce` / `music_generate` 字样，但它们来自工具 schema 或 skill 文档上下文，不是真实 TOOL 调用。真实 TOOL 调用只有 `write_file`、`edit_file`、`submit_render`、`query_render`、`show_final_video` 等。

## 根因

### 根因 1：路由到了 HyperFrames 视觉合成，但没有先做主体资产处理

Agent 将需求理解为 HTML/GSAP 动效合成：紫色背景、图片震动、粒子 canvas、弹卡。这条路线本身没错，但遗漏了前置的 subject/cutout 步骤。

因此最终画面是“原始图片整体作为矩形图片层显示在紫色背景上”，不是“玩具主体被抠出来后叠到紫色背景上”。

### 根因 2：音频意图没有被结构化，导致默认静音出片

需求里出现了：

- violent shaking
- golden particle explosion
- spring animation card

这些都天然对应 SFX：

- 00:02-00:04 震动/摇晃声
- 00:04 爆炸/金币粒子 burst
- 00:04 弹卡 pop / sparkle

但 agent 没有把它们转成 audio_intent / sfx_plan，也没有在写 HTML 前生成音频资产。最终 HTML 只有视觉元素，没有 `<audio>`。

### 根因 3：交付前缺少音频流阻断

HyperFrames skill 已写明：

- 非 `explicit_silent` 时，HTML 中必须有至少一个 `<audio src="https://...">`
- 最终 render 后应 `probe_media`
- 如果无 audio stream，应阻断交付

本 case 最终直接 `show_final_video`，没有 `probe_media`，所以无声视频被交付。

### 根因 4：第二次只补人声的可能原因

当前可拉取 trace 未包含“第二次只有人声”的二次修改链路，不能从本 trace 实锤。但从现象看，最可能是修复时只补了 VO/TTS/人声轨，未补 SFX 轨。

也就是说，修复路径把“没有声音”误收敛成“加一条人声/旁白”，而没有还原用户动效里真正需要的音效设计：震动、爆炸、粒子、弹卡等同步 SFX。

## 当前项目修复方案

### 1. 重新准备干净主体图

对 `a_FPHnF4f_06_03_blind_box_toy.png` 先做主体归一化：

- 若源图有真实 alpha：验证边缘和透明区域。
- 若源图无真实 alpha：用 cutout/re-plate 流程得到“玩具主体 + 可 key 的纯色底板”或真实透明 PNG。
- 产出 `cutout_asset_manifest`，记录：
  - source asset
  - normalized/cutout asset
  - plate color 或 alpha 状态
  - subject_preserved
  - centered_verified

HTML 中只能引用处理后的主体资产，不能再直接引用原始上传图。

### 2. 补齐音效，不要只补人声

这条视频建议至少 3 层 SFX：

- `00:02.0-00:04.0`：快速 shake / rattle，跟随震动强度增强。
- `00:04.0`：golden burst / magical explosion hit。
- `00:04.05-00:05.2`：card pop + sparkle chimes，随文字 stagger 轻微点缀。

如果需要音乐，可加一条低音量 magical/product reveal BGM，但不是必须。人声不是这个需求的核心，除非用户明确要求读出卡片文案。

### 3. HTML 或剪辑时间线绑定音频

如果继续走 HyperFrames，HTML 应包含音频层，例如：

```html
<audio src="https://.../shake_rattle.mp3" data-start="2" data-duration="2" data-track-index="10"></audio>
<audio src="https://.../golden_burst.mp3" data-start="4" data-duration="1.2" data-track-index="11"></audio>
<audio src="https://.../card_pop_sparkle.mp3" data-start="4.05" data-duration="1.6" data-track-index="12"></audio>
```

如果使用 video-editor 合成，则应在 audio tracks 中显式放入这些 SFX，并与视觉关键帧对齐。

### 4. 交付前必须校验

交付前做三类检查：

1. 视觉抽帧：
   - `0.5s`：只有玩具主体在紫色背景上，无原图底色矩形。
   - `2.5s`：主体震动仍居中，不因 transform 冲突偏移。
   - `4.0s`：原图隐藏，金色粒子爆发。
   - `4.5s`：卡片弹出，文字清晰。
2. 音频检查：
   - `probe_media` 最终 MP4，必须有 audio stream。
   - 检查 2-4 秒、4 秒附近音量峰值，确认不是只有人声。
3. 语义检查：
   - 若用户未要求旁白，不要用人声代替音效。
   - SFX 与视觉事件逐点对齐。

## 系统级防复发

1. **产品/贴纸/Logo 叠到新背景时强制 cutout gate**
   - 触发词包括：solid background、gradient background、center display、product reveal、logo reveal、sticker、toy on background。
   - 如果使用原始上传图直接 `<img>`，且没有 alpha/cutout manifest，应阻断 render。

2. **视觉事件自动生成音频意图**
   - shake、explosion、burst、pop、spring、sparkle、impact 等关键词必须生成 `sfx_plan`。
   - 不得把“音频缺失”默认修成“加人声”。

3. **非静音项目必须有音频流校验**
   - 用户没有明确要求 silent 时，最终成片无 audio stream 直接阻断。
   - 如果 prompt 包含 explosion / sound-like visual events，最终音轨不能只有 VO/TTS，必须有 SFX 或明确说明用户选择静音。

4. **真实工具调用与 skill 文档区分**
   - 分析/监控时不能只用全文搜索判断 `audio_produce` 是否发生。
   - 必须以 observation `type=TOOL` 且 `name=audio_produce/music_generate/...` 为准。

