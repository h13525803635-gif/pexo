# Case 68552524367 — 参考图未生成视频 & 模特长相不一致

## 结论

本 case 的核心问题有两层：

1. **参考图未驱动视频/人脸生成**：用户上传的 `06_cosmetics_set.png` 仅被用作底部产品卡片与 HTML 合成装饰层，全程 **0 次 `video_generate`**，妆前/妆后人脸也是凭空文生图，未走图生视频或图生图管线。
2. **妆前/妆后模特不一致**：两次 `image_generate` 均为独立 `text2image`，妆后图只在 prompt 中写了 "same woman as natural look"，**未将妆前图作为参考传入**，导致左右两张脸明显不是同一人。

一句话根因：**Agent 将「妆前妆后对比视频」误判为 HyperFrames 静态图遮罩动画任务，跳过 reference-skill / image-skill 的人物一致性流程，用两次无锚定的文生图硬拼对比画面。**

## 元数据

| 字段 | 值 |
|------|-----|
| 项目 ID | `68552524367` |
| Langfuse trace | `pexo:68552524367` / `fc4a7f2baa22a993d264134ed0ec42d6` |
| 会话 | `thread_75127832422_68552524367_01KTT8MKDAP9E4HW2EQ009933G` |
| 时间 | `2026-06-11T02:37:04Z`（test） |
| 用户 ID | `75127832422` |
| 迭代次数 | 1（单轮交付） |
| 本地 trace | `analysis/langfuse-data/cases/68552524367/trace-1-fc4a7f2b.json` |
| 本地素材 | `analysis/langfuse-data/cases/68552524367/media/` |

## 用户需求

> Use 03_brand/06_cosmetics_set.png as the bottom product display layer. Generate a before-and-after makeup comparison video above it, using a vertical dividing line mask animation to create a left-to-right sliding comparison effect. During the sliding process, fade in a product display card at the bottom referencing 03_brand/06_cosmetics_set.png with the text 'Full look with just 5 items - Click to shop'.

用户附带参考图：`a_yX5HvsZ_20_06_cosmetics_set.png`（玫瑰金彩妆 flat lay，粉色大理石背景，**无人物**）。

## 真实调用链路

```
用户 prompt + cosmetics_set.png
  → read_file hyperframes-skill
  → analyze_file_content(cosmetics_set.png)  # 仅提取产品/配色
  → image_generate ×2 (text2image, seedream-4-5)
       before_makeup_look  # 独立文生图
       after_makeup_look   # 独立文生图，prompt 含 "same woman"
  → write_file makeup_comparison.html
  → submit_render → makeup_before_after_comparison.mp4
  → render_frame (frame_check_5s)
  → show_final_video
```

**全程未出现：**

- `video_generate`
- `image2image` / `reference2image`
- `reference-skill` / `image-skill` / `creative-skill` 读取
- `execute_edit_video`

## 工具调用统计

| 工具 | 次数 | 说明 |
|------|------|------|
| `image_generate` | 2 | 均为 `mode: text2image` |
| `submit_render` | 1 | HyperFrames HTML 渲染 |
| `render_frame` | 1 | 5s 抽帧验收 |
| `show_final_video` | 1 | 交付成片 |
| `video_generate` | **0** | — |

## 问题 1：参考图未生成视频

### 现象

用户期望参考图参与视频内容生成；实际参考图只出现在底部产品卡片缩略图，妆前/妆后对比区完全是 AI 凭空生成的人脸静态图 + GSAP 遮罩动画。

### 根因

| 层级 | 说明 |
|------|------|
| 路由误判 | Agent 首读 `hyperframes-skill`，将「对比视频」理解为 HTML 遮罩动效，而非 `video_generate` 或图生图 |
| 参考图角色错位 | `cosmetics_set.png` 是产品 flat lay（无模特），Agent 正确用于底部展示，但未补生成/请求角色参考，也未用参考图驱动人脸 |
| 跳过标准流程 | 未走 `reference-skill` 做素材缺口分析（缺模特参考）和 `image-skill` 的人物一致性规范 |
| 交付形态 | 成片是 HyperFrames 渲染的静态图动画，不是 AI 视频生成产物 |

### 证据

- `analyze_file_content` 返回：彩妆产品、玫瑰金包装、flat lay 布局，**未识别到人物**
- `image_generate` 两次调用 `mode: "text2image"`，无 `image_list` / 参考图参数
- 最终 `makeup_before_after_comparison.mp4` 来自 `submit_render(html_file=makeup_comparison.html)`

## 问题 2：模特长相不一致

### 现象

妆前（左）与妆后（右）面部差异明显，不像同一人做妆容变换：

| 特征 | 妆前 `before_makeup_look` | 妆后 `after_makeup_look` |
|------|---------------------------|--------------------------|
| 肤色 | 偏暖橄榄色，有明显雀斑 | 偏冷白皙，无雀斑 |
| 五官 | 鼻翼较宽，眼型/脸型不同 | 鼻梁更窄，眼型/骨相不同 |
| 发型 | 深色自然发 | 偏红棕，带金色闪粉 |

### 根因

两次 `image_generate` 并行发起，均为独立文生图：

**妆前 prompt（节选）：**
> Beauty portrait photo, young woman with completely bare natural skin, no makeup, clean face, slight freckles visible...

**妆后 prompt（节选）：**
> Beauty portrait photo, young woman with full glamorous rose gold makeup look... **same woman as natural look**

问题点：

1. 妆后仅在文本中声明 "same woman"，**未传入妆前图作为 `image2image` 参考**
2. 未先生成 `character_ref` 再派生妆后变体
3. 未读取 `image-skill` 中妆前/妆后同一人应走角色参考链的规则
4. Seedream `text2image` 跨次生成无法保证人物 ID 一致

### 视觉证据

本地抽帧 `frame_check_5s.png` 可见左右分屏对比，分割线中间有滑块，底部产品卡片已正确展示 `cosmetics_set` 缩略图与 CTA 文案，但左右人脸不一致。

## 解决方案

### 短期修复（重制本条视频）

1. **先锁定角色参考**
   - 生成一张 `character_ref` 素颜正脸照（或请用户上传模特图）
   - 记录 asset_id 供后续所有变体引用

2. **妆后用图生图，不用独立文生图**
   - `after_makeup_look` 应以 `before_makeup_look` 或 `character_ref` 为 `image_list` 参考
   - `mode: image2image`，prompt 仅描述妆容变化（眼影、唇色、高光），禁止重述五官

3. **动效层用 HyperFrames（此路线合理）**
   - 两张**人物一致**的静态人脸 + 竖线遮罩从左向右滑动
   - 底部 `cosmetics_set.png` 产品卡片淡入
   - 此场景不需要 `video_generate`，但前提是上游人脸已一致

4. **交付前做人脸一致性 QA**
   - 对比妆前/妆后：肤色、雀斑、鼻型、眼型、发际线
   - 抽帧检查遮罩滑动中间态是否对齐

### 中期修复（Skill / Agent）

1. **路由规则**：「妆前/妆后对比」「before-after」「slider reveal」类请求
   - 必须先走 `reference-skill` 建立角色参考
   - 禁止两次独立 `text2image` 生成同一人变体
   - HyperFrames 只负责动效合成，不负责「造人」

2. **image-skill 门禁**
   - 同一人物的多状态图（素颜/化妆/换装）必须用 `image2image` 或共享 `character_ref`
   - prompt 中的 "same person" 不能替代参考图参数

3. **hyperframes-skill 补充**
   - 当 composition 含人物对比时，要求上游已产出 character-consistent 素材
   - 禁止在 HyperFrames 流程内用 `text2image` 凭空生成人脸

4. **素材缺口预检**
   - 用户只提供产品图、无人物参考时，应主动生成或询问模特基准照
   - `cosmetics_set.png` 无人物 ≠ 可以跳过角色参考

## 建议的产品规则

当请求满足：

- 需要妆前/妆后、before/after、slider comparison；
- 左右/上下为**同一人**的不同状态；

则默认路径应为：

```text
character_ref (生成或用户上传)
  → before_look (text2image 或直接用 ref)
  → after_look (image2image, ref=before_look)
  → HyperFrames 遮罩动画合成
  → show_final_video
```

禁止：

```text
text2image(before) + text2image(after, prompt="same woman") → HyperFrames
```

## 归因

| 问题 | 严重度 | 归因 |
|------|--------|------|
| 参考图未驱动视频/人脸生成 | P1 | Agent 路由到 HyperFrames；产品参考图无人物但未补角色参考 |
| 妆前/妆后模特不一致 | P1 | 两次独立 text2image，无 image2image 锚定；跳过 reference-skill / image-skill |

## 关联 Case

- [88897877704](case-88897877704-上传图未图生视频与解决方案.md) — 上传图未挂 image_list、未走图生视频
- [65160007362](case-65160007362-election-static-hyperframes-no-video-audio-card-overlap.md) — 参考图未 video_generate、HyperFrames 静态图合成
- [16096173371](case-16096173371-v6-misunderstood-inconsistent-ref.md) — 跨段参考不一致
