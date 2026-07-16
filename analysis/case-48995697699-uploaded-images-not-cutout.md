# 项目 48995697699：用户上传图片未抠图的根因与解决方案

## 结论

项目中的用户上传商品图没有经历抠图失败，而是从一开始就没有进入抠图流程。Agent 将 PNG 文件直接视为可用于合成的商品素材，在检查图片内容后，把原始上传路径写入 HyperFrames HTML。整个链路没有调用 `cutout_image`，也没有进行真实透明通道检测、背景移除或抠图结果验收。

因此，问题本质是素材预处理决策和交付 QC 缺少硬性门禁，而不是抠图服务报错。

## 项目与链路概况

- 项目 ID：`48995697699`
- Trace ID：`e2e7ee2dd3ab74a34b3fe9225a344090`
- Trace 开始时间：`2026-07-15T12:34:30.737Z`
- 最终产物：`AlMazraa_8Flavors_Showcase_v1.mp4`
- 合成文件：`almazraa_showcase.html`
- 上传素材：Al Mazraa Logo 和 8 张果汁瓶商品图

## 关键证据

### 1. 素材检查只识别内容，没有检查背景和透明通道

Agent 对每张商品图调用了视觉分析，查询内容均类似：

```text
Describe this juice product: bottle shape, colors, label design, flavor name
```

被检查的原始上传素材包括：

- `a_1aX2oEu_Mango.png.png`
- `a_WmzmkBD_Orange.png.png`
- `a_p1x9bvP_Lemon-mint.png`
- `a_1ZHdf3U_Strawberry.png.png`
- `a_RLTk1AX_Pomegranate.png.png`
- `a_tLAFqgV_Kamruddin.png.png`
- `a_fbkY48t_Grapes.png.png`
- `a_of6CqNi_Mix_Fruits.png.png`
- `a_Fm1gRQW_Al-Mazraa-Logo.png`

分析问题没有要求确认：

- 图片是否具有真实 alpha 通道
- 背景是否完全透明
- 是否存在白底、色底或矩形画布
- 商品边缘是否适合直接叠加到新背景

文件后缀是 `.png` 不能证明图片已经去背；PNG 同样可以是完全不透明的 RGB/RGBA 图片。

### 2. 没有任何抠图工具调用

该 trace 共拉取到 699 条 observation。实际工具调用中没有发现以下任何动作：

- `cutout_image`
- remove background / background removal
- alpha channel inspection
- cutout asset manifest
- re-plate / plate-key
- 抠图结果边缘或透明区域验收

这说明系统没有尝试抠图，因此不存在“抠图调用失败后回退原图”的情况。

### 3. HTML 直接引用用户上传原图

Agent 创建 `almazraa_showcase.html` 后，直接将上述 `/projects/48995697699/workspace/assets/a_...` 原始路径用于商品瓶和 Logo 的 `<img>` 元素。后续修改集中在：

- GSAP 动画进出场
- track index 冲突
- 元素隐藏时机
- 画面布局和装饰粒子
- lint 与 render

链路中没有把原始路径替换为抠图后产生的新 `asset://...` 资产。

### 4. QC 没有检查背景残留

Agent 对 1 秒、4 秒、10 秒、19 秒和 27 秒等关键帧进行了视觉 QC，但检查项主要是：

- Logo 是否可见、位置是否正确
- 商品瓶是否清晰、突出
- 口味标签是否存在
- 分屏、栅格和背景色是否符合设计

没有检查：

- 商品图是否出现矩形底色
- 商品周围是否有白边或背景残留
- 图片是否以透明主体形式融入场景
- Logo 或瓶体是否仍带原图画布

因此，即使成片里出现明显未抠图效果，现有 QC 也不会将其判为失败。

## 根因

### 根因 1：把 PNG 后缀误当成透明素材证明

Agent 直接使用 `.png` / `.png.png` 上传文件，没有验证真实 alpha。素材类型判断停留在文件名和视觉内容层，没有进入图像属性检测。

### 根因 2：商品主体叠加场景缺少 cutout gate

该项目使用独立商品瓶叠加彩色动态图形背景，语义上属于典型的“主体层 + 新背景”场景，应在合成前强制完成主体归一化。但当前 motion workflow 没有把这一要求设为 render 前置条件。

### 根因 3：视觉分析提示词不包含素材可合成性检查

视觉分析只关注瓶型、颜色、标签和口味，导致 Agent 得到了设计信息，却没有得到“这张图能否作为透明主体直接使用”的判断。

### 根因 4：交付 QC 偏向可见性和版式，不检查合成质量

QC 确认了瓶子“看得见”，但没有确认瓶子“已正确去背”。可见性检查不能替代透明度、边缘和背景残留检查。

## 当前项目修复方案

### 1. 对 9 张上传图逐一做真实 alpha 检测

不能依据扩展名判断。每张图片应检查：

- 是否存在 alpha 通道
- alpha 是否实际包含透明像素，而不是全 255
- 四角和主体外区域是否透明
- 是否有白底、色底、阴影底板或压缩残边

已有合格透明通道的图片可以保留；其余图片进入抠图流程。

### 2. 对不合格素材调用抠图并生成新资产

对每个需要处理的商品瓶和 Logo：

1. 将 workspace 文件解析为工具可访问的 `asset://...` 或有效 URL。
2. 如尺寸超出工具限制，先等比 resize。
3. 调用 `cutout_image`。
4. 保存 source asset 与 cutout asset 的映射。
5. 验证主体没有被裁断，瓶盖、瓶身、标签和细小边缘得到保留。

建议形成如下 manifest：

```json
{
  "source_asset": "asset://source",
  "cutout_asset": "asset://cutout",
  "has_real_alpha": true,
  "subject_preserved": true,
  "edge_verified": true
}
```

### 3. 替换 HTML 中全部原图引用

`almazraa_showcase.html` 中所有商品瓶和 Logo 的 `<img src>` 必须引用验证后的 cutout asset。主场景、分屏和最终 8 瓶栅格都要统一替换，避免某些镜头使用抠图图、另一些镜头又回退原图。

### 4. 重新渲染并做关键帧验收

至少检查：

- `1s`：Logo 周围没有矩形底色或白边。
- `4s`：Mango 瓶体以透明主体叠在背景上。
- `10s`：Lemon Mint 瓶体边缘干净，没有源图画布。
- `19s`：Kamruddin 和 Grapes 两张图在分屏中均已去背。
- `27s`：最终 8 瓶栅格全部使用抠图资产。

## 系统级防复发方案

### 1. 增加 render 前 cutout gate

满足以下任一条件时，强制验证 alpha 或提供 cutout manifest：

- 商品、人物、Logo、贴纸等主体叠加到新背景
- Prompt 包含 showcase、product reveal、center product、solid background、gradient background
- HTML 将用户上传图片作为独立 `<img>` 主体层使用

没有真实透明通道且没有 cutout 结果时，应阻断 render，而不是继续使用原图。

### 2. 扩展素材视觉检查提示词

除内容描述外，统一追加：

```text
Check whether the image has a clean transparent background and can be composited
as an isolated subject. Report any opaque background, rectangular canvas, halo,
white fringe, cropped edge, or unwanted shadow plate.
```

### 3. 不允许使用扩展名推断透明度

在素材处理规则中明确：

- `.png` 仅代表容器格式
- 必须读取像素或 alpha 统计
- alpha 通道存在但完全不透明，也视为未抠图

### 4. 增加合成质量 QC

商品主体项目的关键帧 QC 必须包含：

- 是否存在矩形底色
- 是否存在白边、黑边、色边或 halo
- 是否错误裁切主体
- 是否所有镜头统一使用 cutout asset
- 透明主体是否自然融入新背景

### 5. 建立原图引用阻断规则

当用户上传图被判断为需要抠图后，扫描最终 HTML：如果 `<img src>` 仍指向 source asset，而不是 manifest 中的 cutout asset，直接阻断交付。

## 验收标准

修复完成需同时满足：

1. 所有商品图和 Logo 都有明确的 alpha 检测结果。
2. 所有不透明素材均生成并使用抠图资产。
3. 最终 HTML 不再引用需要抠图的原始上传图。
4. 关键帧不存在矩形底色、明显白边或背景残留。
5. QC 记录包含透明度和边缘质量检查，而不仅是主体可见性检查。

