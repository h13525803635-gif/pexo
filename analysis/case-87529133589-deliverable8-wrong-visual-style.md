# 项目 87529133589：成片 #8 Wrong visual style — 原因与解决办法

**问题成片（#8）**：`angels_banner_clip_20260518T135537_315dba30.mp4`（6s，1280×720）  
**主 trace**：`analysis/langfuse-data/cases/87529133589/trace-10-d961357d.json`  
**项目名**：Gloveman Nightmare-to-Dream Ad  
**Langfuse 会话**：13 traces（含 memory-session）  
**分析日期**：2026-05-19  

---

## 一、成片 #8 在会话中的位置

本会话共 **8 次**独立 `show_final_video` 交付（按时间序）：

| # | 文件 | Trace | 说明 |
|---|------|-------|------|
| 1 | `gloveman_ad_final.mp4` | trace-1 | 首版 18s 主广告 |
| 2–5 | `gloveman_ad_v2` ~ `v5` | trace-1 | 人物/卧室一致性迭代 |
| 6 | `gloveman_ad_v6.mp4` | trace-7 | 延长至 25s + 结尾 hold |
| 7 | `gloveman_ad_v7.mp4` | trace-8 | S2 改为空仓库噩梦 |
| **8** | **`angels_banner_clip_...mp4`** | **trace-10** | **6s 天使 end card（本报告）** |

后续 trace-12 用户重做 angels 时明确要求 **3D lettering**，产出 `angels_3d_banner_v2`，可视为对 #8 视觉载体的纠正。

---

## 二、用户需求（trace-10 验收标准）

用户原文：

> Create a **comical** 6 second video of two angels in a sunny sky with lots of puffy clouds, flying around, **holding the lettering between them** `"www.gloveman.co.uk"` and `"01209 314759"`.

| 要素 | 含义 |
|------|------|
| 风格 | **comical** — 卡通喜剧天使（Agent 对此理解正确） |
| 载体 | **lettering** — 字本身由天使托举，不是横幅/卷轴/海报 |
| 文案 | 两行：`www.gloveman.co.uk` + `01209 314759` |
| 时长 | 6 秒 |

**未要求**：结尾 thumbs up、布料横幅、平面印刷黑字、与主片 v7 自动拼接（Agent 事后提议 stitch）。

---

## 三、实际交付（#8 真实路径）

### 3.1 调用时序（trace-10， angels 段落）

| # | 工具 | 说明 |
|---|------|------|
| 1 | `image_generate` | `angels_sky_ref`（Seedream，`cartoon-style` + **white banner scroll** + **bold black lettering**） |
| 2 | `analyze_file_content` | 验收：天使可见、banner 上 www 可读；**未验收是否为 3D/立体字、是否卷轴载体** |
| 3 | `video_generate` | `angels_banner_clip`（Seedance ref2video，沿用 **banner scroll + bold black lettering**） |
| 4 | `show_final_video` | 交付 #8 |

### 3.2 视觉实际参数（prompt 摘要）

**参考图 `angels_sky_ref`：**
```
Comical cartoon-style illustration … white banner scroll …
bold clean black lettering … www.gloveman.co.uk / 01209 314759
```

**视频 `angels_banner_clip`：**
```
Comical cartoon-style animation … large white banner scroll …
bold black lettering … [5-6s] thumbs up to camera  ← 用户未要求
```

**结论**：用户要 **lettering（立体字）**，成片做成 **横幅卷轴 + 2D 黑字** → QA 标 **Wrong visual style**（视觉载体/渲染方式错误，非「不该卡通」）。

---

## 四、问题清单（现象 → 根因 → 解决办法）

### P0-1：「lettering」被降级为 banner scroll

| 项 | 内容 |
|----|------|
| **现象** | 天使举的是白色布质卷轴，字为平面黑体，非立体字 |
| **根因** | Agent 未解析 lettering vs banner vs 3D text，直接写入 image/video prompt |
| **重做** | 使用 **3D gold extruded lettering**，天使左右托举字块（见 trace-12 `angels_3d_text_ref` 路径） |
| **Skill** | brainstorm/modification：遇到 `lettering` / `3D lettering` 时禁止默认 `banner` / `scroll` |

### P0-2：参考图验收未拦截错误载体

| 项 | 内容 |
|----|------|
| **现象** | `analyze_file_content` 返回「banner 可读、cherub 可爱」即通过 |
| **根因** | 验收 query 只问 legibility，未对照用户指定的 **lettering 形态** |
| **重做** | lettering 类需求：若描述含 `banner`/`scroll`/`flat black on fabric` → **fail，重生成** |
| **Skill** | generation-skill：参考图 QA checklist 增加 `carrier_type_match` |

### P1：擅自增加未请求表演

| 项 | 内容 |
|----|------|
| **现象** | 5–6s 双臂 thumbs up |
| **根因** | prompt 自行加「cheerful sign-off」 |
| **重做** | 删除未在用户 brief 中的动作节拍 |
| **Skill** | video prompt：禁止添加用户未写的 gesture / 表情收尾 |

### P1：与主片 v7 拼接未做风格门禁

| 项 | 内容 |
|----|------|
| **现象** | Agent 提议将 cartoon end card stitch 到 photoreal `gloveman_ad_v7` |
| **根因** | end card 虽符合「comical」，但与主片 cinematic photoreal 硬切；未先确认用户接受风格断裂 |
| **重做** | 若拼接：先确认 end-card 子风格 + 是否需 0.5s 过渡/LUT；或保持独立交付 |
| **Skill** | assembly：跨渲染族（photoreal + cartoon）须用户显式确认 |

### P2：模型路由不利于立体字

| 项 | 内容 |
|----|------|
| **现象** | Seedream 参考图倾向横幅+平面字；Seedance 沿用以至成片 |
| **根因** | 未对 in-frame 3D 文案走 gpt-image-2 / Kling 等更稳路径 |
| **重做** | 立体字 + 精确文案：优先 `gpt-image-2` ref → Seedance i2v（trace-12 已验证） |
| **Skill** | video-models-routing：in-frame 3D text → 指定 L4 / image-2 分支 |

---

## 五、失败因果链（#8）

```
用户: "holding the lettering between them" + comical angels
  → Agent 理解为 white banner scroll + bold black 2D text
  → Seedream ref 通过「banner 可读」验收
  → Seedance 视频 + 擅自 thumbs up
  → show_final_video #8 = Wrong visual style
  → 用户 trace-12 重做，明确 "3D lettering" + Visit/or Call 完整文案
```

---

## 六、推荐重做产线（#8 正确版）

```
1. 解析载体：lettering → 3D extruded gold text（非 banner）
2. 文案锁定：行1 "www.gloveman.co.uk" / 行2 "01209 314759"（或用户后续完整句式）
3. image_generate（gpt-image-2）：
   - cherub angels + 两行 3D 金字，16:9，comical but legible
4. analyze_file_content（硬门禁）：
   - FAIL if banner/scroll/flat print
   - FAIL if 任一行文案缺失或不可读
5. video_generate（Seedance ref2video）：
   - 仅复现 ref 载体；不加 thumbs up
6. show_final_video → 若需拼 v7，先问用户是否接受 cartoon↔photoreal 硬切
```

### Prompt 方向示例（参考图）

```
Comical cartoon cherub angels in sunny cloudy sky, each holding one end of
two lines of bold shiny 3D gold extruded lettering (NOT a fabric banner):
Line 1: "www.gloveman.co.uk"
Line 2: "01209 314759"
```

---

## 七、Skill 改进建议（防复发）

| 规则 | 写入位置 | 要点 |
|------|----------|------|
| Lettering 载体词典 | brainstorm-skill / modification-skill | lettering → 3D text；禁止默认 banner/scroll |
| 参考图 carrier QA | generation-skill | lettering 需求下 banner/scroll = fail |
| 禁止擅自加戏 | generation-skill | 用户未写 gesture 不得写入 prompt |
| 跨风格拼接确认 | assembly-skill | photoreal 主片 + cartoon end card 须显式确认 |
| 3D 文案模型路由 | video-models-routing | in-frame 3D text → gpt-image-2 / Kling |

---

## 八、附：主广告线其它风格问题（非 #8，供对照）

首版主广告 `gloveman_ad_final` 另存在 **photoreal 卧室 + cartoon 梦云 + vampire 反派** 混用问题；用户后续在 trace-8 将噩梦改为空仓库（v7）。**#8 问题独立于此**，专指 angels end card 的 lettering 载体错误。

| 主广告问题 | 与 #8 关系 |
|------------|------------|
| 跳过 Style Gate、卡通云/吸血鬼反派 | 主片 v1–v7 范畴 |
| 人物卧室不一致 | 主片迭代范畴 |
| **#8 banner 非 lettering** | **仅 end card trace-10** |

---

## 九、数据文件索引

| 文件 | 说明 |
|------|------|
| `analysis/langfuse-data/cases/87529133589/trace-10-d961357d.json` | #8 主 trace（angels_banner_clip） |
| `analysis/langfuse-data/cases/87529133589/trace-12-e285b770.json` | 用户重做 3D lettering（angels_3d_banner_v2） |
| `analysis/langfuse-data/cases/87529133589/trace-8-d880f13a.json` | v7 空仓库版主广告 |
| `analysis/langfuse-data/cases/87529133589/trace-1-633d4b45.json` | 主广告首版与 v2–v5 迭代 |
