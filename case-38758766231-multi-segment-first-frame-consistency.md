# Case 38758766231 — 美甲一致性问题分析

**需求**：美甲穿戴甲 UGC 广告（人物+参考视频+美甲样式，15s 竖屏）  
**时间**：2026-04-20 13:13 / 2026-04-21 03:21–03:32（3 个 trace）  
**核心问题**：作为视频唯一产品主体的美甲，在三段视频中外观不一致

---

## 项目概况

三段结构：

| 段 | 时长 | 内容 |
|----|------|------|
| `nail_ugc_seg1` | 4s | 正脸开场，兴奋说「已经有三个人问我指甲在哪做的了——」 |
| `nail_ugc_seg2` | 7s | 手部特写，手指在阳光下转动展示美甲闪光效果 |
| `nail_ugc_seg3` | 4s | 回到正脸，笑着说「链接在主页，真的！」 |

美甲款式：克莱因蓝+古铜金 3D 浮雕+镭射虹彩+酒红爱心，赛博朋克×Y2K 风格。

---

## 根因分析

### 1. seg1 完全没有传入美甲参考图（最直接原因）

| 段 | `image_list` | 美甲参考图 |
|----|-------------|-----------|
| `nail_ugc_seg1` | `[a_31vdnWr]`（仅人物图） | **缺失** |
| `nail_ugc_seg2` | `[a_31vdnWr, a_b592vRD]` | ✓ |
| `nail_ugc_seg3` | `[a_31vdnWr, a_b592vRD]` | ✓ |

seg1 是正脸开场，但手部在画面中可见。没有美甲参考图，Seedance 完全自由发挥指甲外观，与 seg2/seg3 展示的克莱因蓝 3D 美甲毫无关联。seg1 的 prompt 里对美甲的描述也几乎为零，进一步放大了这个问题。

### 根因溯源：参考图在 creative → generation 交接时丢失

通过 trace 还原完整流程，发现美甲参考图的丢失有更具体的发生节点：

**概念片阶段（creative-skill，trace-2）**：

`ugc_nail_preview` 的 image_list = `[a_31vdnWr, a_b592vRD]`，人物图 + 美甲参考图**都正确传入**。但概念片里的美甲外观并非真正锚定——Seedance 是依靠 prompt 文字描述（"Klein blue, gold metal 3D sculptural decorations…"）自由生成的，图像仅作风格参考。用户看到概念片后确认通过，**实际上确认的是 prompt 描述出的效果，而非被参考图约束的效果**。

**生产阶段（generation-skill，trace-3）**：

generation-skill 接手后将视频拆成三段独立生成。在编排 seg1（正脸开场段）时，agent 认为「此段重点在人物，不需要美甲参考」，只保留了人物图，将 `a_b592vRD` 从 seg1 的 image_list 中删除。seg2/seg3 因为内容明确涉及美甲才保留了参考图。

**本质**：generation-skill 在多段拆分时，对「哪些段需要哪些参考图」做了主观判断，而不是将 creative-skill 确定的 image_list 原样带下去。正脸段因手部不是主体而被认为无需美甲参考，但 Seedance 生成时手部仍然可见，美甲外观因此完全失控。

### 2. seg2/seg3 有参考图但锚定仍然很弱

即便 seg2/seg3 都传入了美甲参考图 `a_b592vRD`：

- `image_list` 同时包含人物图和美甲图，Seedance 对两张图的权重分配不受控
- 美甲参考图是**静态照片**，seg2 的核心动作是「手指转动闪烁」，动态视角下的美甲细节 Seedance 会自行生成，难以还原 3D 浮雕的具体造型、爱心位置、蓝色分布
- seg2 和 seg3 独立生成，即便参考同一张图，生成的美甲细节也因采样随机性而各异

### 3. 生成策略把美甲当背景元素处理

用 `reference2video` + `image_list` 的方式对美甲进行参考，本质是「风格借鉴」，不是「精确复现」。复杂 3D 美甲款式无法通过这个路径精确还原——每次独立生成都是一套不同的美甲。

---

## 正确的处理方式

### seg1：必须加入美甲参考图

```python
# 错误
image_list = [person_ref]

# 正确
image_list = [person_ref, nail_ref]
```

即便 seg1 以正脸为主，手部在画面中仍可见，必须传入美甲参考图约束外观。

### seg2（美甲特写段）：应换为 image2video 模式

纯美甲特写段更适合以美甲图作为 `first_frame`，让模型从实际美甲样子出发做动效：

```python
# 当前（reference2video，美甲从零生成）
mode = 'reference2video'
image_list = [person_ref, nail_ref]

# 更优（image2video，以美甲图为首帧）
mode = 'image2video'
first_frame = nail_ref   # 美甲直接作为视频起点
```

### 三段 image_list 必须完全一致

所有段都应传入相同的参考图组合，防止 Seedance 在不同段对同一产品的外观理解出现偏差。

---

## 系统性问题

这不是单个 case 的偶发问题。generation-skill 在**「产品特写+人物出镜」的多段结构**中，缺少对核心产品视觉一致性的保障机制：

- 没有检查各段 image_list 是否包含产品参考图
- 没有识别「产品特写段」并切换为 image2video 模式
- 没有在多段生成后校验产品外观是否一致再合成
