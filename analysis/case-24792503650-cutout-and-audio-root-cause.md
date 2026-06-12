# Case 24792503650 — 首次成片不抠图且无声音根因

## 结论

项目 `24792503650` 首次成片同时出现“不抠图”和“无声音”，是两个独立漏项叠加：

1. 首版直接把用户上传的原始手机 PNG 作为 `<img>` 放进 HyperFrames HTML，没有先做 cutout normalization，也没有使用 green-plate/canvas key-color 去底。
2. 首版没有生成或挂载任何音频资产；trace 中没有 `audio_produce`、`music_generate`，HTML 中也没有 `<audio src=...>`，最终交付前也没有 `probe_media` 校验音轨。

## 时间线

| 时间 | 关键动作 | 影响 |
|---|---|---|
| 2026-06-11 02:30 | 用户要求生成 360 度旋转手机展示视频，上传 `07_04_smartphone.png` | 需求是产品展示动效，没有明确说明静音 |
| 2026-06-11 02:31 | `get_file_info` 获取原图签名 URL | 原图被直接作为 HTML `<img>` 素材 |
| 2026-06-11 02:31 | 写入 `smartphone_showcase.html` | HTML 直接引用原始 PNG，未抠图 |
| 2026-06-11 02:32 | `submit_render` 首次提交渲染 | 未见任何音频生成或 `<audio>` 绑定 |
| 2026-06-11 02:35 | `show_final_video` 交付 `smartphone_showcase_final.mp4` | 首版成片：背景未去除，且无音轨 |
| 2026-06-11 04:02 | `image_generate` 将原图背景替换为纯绿 `#00FF00` | 后续才开始正确的 cutout 预处理 |
| 2026-06-11 04:03-04:04 | QA 纯绿底与预览图，替换 HTML 素材为 cutout 版本 | 抠图问题在后续版本被修正 |
| 2026-06-11 06:13-06:18 | 重新提交并交付 `a_3sixNy9` | 抠图版重新渲染完成；trace 中仍未见补音频 |

## 不抠图根因

首版 HTML 直接使用：

- `/projects/24792503650/workspace/assets/a_aNZg16U_07_04_smartphone.png`
- asset id: `a_aNZg16U`
- 文件名: `07_04_smartphone.png`

该图经 `analyze_file_content` 判断为带深色渐变背景的产品渲染图，不是透明背景素材。首版没有执行 subject-asset-skill 的 cutout normalization，也没有生成纯色 plate manifest，因此 HyperFrames 只能把整张图片渲染进画面。

后续修复动作是正确方向：

- `image_generate` 生成 `smartphone_cutout_exact_20260611T040312_22745b5f.png`
- 背景被 QA 为纯绿 `#00FF00`
- HTML 将素材替换为 green-plate cutout 版本
- preview QA 显示无绿底残留，手机浮在深色背景上

## 无声音根因

首版 trace 中没有以下任何音频生产或绑定动作：

- `audio_produce`
- `music_generate`
- `<audio src="...">`
- `probe_media` 校验最终 mp4 是否含 audio stream

因此首版不是“音频被合成丢了”，而是“从编排阶段就没有创建或挂载音轨”。HyperFrames 渲染本身不会凭空生成 BGM/SFX，HTML 没有 `<audio>` 时输出 mp4 就会是无声视频。

## 应修复的流程门禁

1. 产品/主体浮层类 HTML composition：若用户上传图不是已验证透明或 cutout manifest，必须先走 cutout normalization，禁止直接 `<img>` 原图。
2. 对 HyperFrames 成片：除非用户明确要求 silent，否则在首版 composition 前必须确定 `audio_intent`，并生成/绑定 BGM 或 SFX。
3. `submit_render` 前检查 HTML：非静音需求必须存在至少一个 `<audio src="https://...">`。
4. `show_final_video` 前必须 `probe_media`：若无 audio stream 且不是 explicit silent，应阻断交付并补音频。

## 对用户解释口径

这次首版的问题不是模型能力限制，而是执行链路漏了两个必要步骤：先把带背景的手机图规范化成可抠的纯色 plate，再把音频资产显式挂进 HyperFrames HTML。后续版本已经补了抠图链路；如还需要声音，需要再补 BGM/SFX 或旁白并重新渲染。
