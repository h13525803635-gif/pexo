# 项目 19598599427：第一次成片 Wrong visual style — 原因与解决办法

**成片（V1）**：`ubaid_to_dubai_animation.mp4`（14s，1920×1080）  
**主 trace**：`analysis/langfuse-data/cases/19598599427/trace-3-f8e08738.json`  
**Langfuse 会话**：6 traces（trace-1 ~ trace-6）  
**分析日期**：2026-05-19  
**QA 看板**：`analysis/langfuse-data/cases/19598599427/qa-report.html`

---

## 一、用户需求（验收标准）

用户原文（trace-1）：

> I need a video where it would be written - **Going to Ubaid** - and then the **(Ubaid)s letter D** will come forward to the word and it will become - **DUBAI** - and a **plane will cross the letter**

| 要素 | 含义 |
|------|------|
| 文案 | 整句「Going to Ubaid」，不是拆开的字母条 |
| 动效 | **Ubaid 里的 D** 前移到词首 → 读出 DUBAI |
| 飞机 | **穿过字母**（与字形同一画面、有交互感） |
| 风格 | **用户未指定**配色/写实/旅游风，仅描述动效逻辑 |

用户仅对 Agent 提议的 **16:9、10–15s** 回复 **「ok」**（格式确认，≠ 视觉风格确认）。

---

## 二、实际交付（V1 真实路径）

### 2.1 调用时序（trace-3）

| # | 时间 (UTC) | 工具 | 说明 |
|---|------------|------|------|
| 1 | 21:04:39 | `write_todos` | Keyframes → 飞机 → 组装 |
| 2 | 21:04:49 | `image_generate` ×3 | `frame_going_to_ubaid`、`frame_dubai_text`、`plane_icon`（gpt-image-2，深蓝极简） |
| 3 | 21:06:17 | `analyze_file_content` ×3 | 验收 PNG 可读性 |
| 4 | 21:06:33 | `video-editor__list_fonts` | 选 Montserrat |
| 5 | 21:06:50 | `video-editor__execute_edit_video` | 纯色底 + **编辑器文字层** + 飞机 PNG overlay |
| 6 | 21:08:34 | `show_final_video` | `ubaid_to_dubai_animation.mp4` |

**未调用**：`video_generate`、`brainstorm` 正式产出、`script-skill` 分镜、**风格 mood 确认**。

### 2.2 视觉实际参数

```
背景: #0a1628 纯色（非用户指定）
文字: Montserrat — "Going to" + "U B A I"（带空格）+ 独立 "D"
D 动效: #f0c040 金色，x 关键帧 +420 → -320
飞机: plane_icon 扁平白色剪影 PNG，overlay 横移（非穿字）
已生成但未进成片: frame_going_to_ubaid.png, frame_dubai_text.png
```

---

## 三、问题清单（现象 → 根因 → 解决办法）

### P0-1：视觉风格未经确认即锁定

| 项 | 内容 |
|----|------|
| **现象** | 深蓝 + 白字 + 金 D 的冷峻 MG，与用户「只描述动效」不符 → **Wrong visual style** |
| **根因** | Agent 将「ok」误认为同意整套 aesthetic；未跑 Style Gate |
| **重做** | 交付前展示 1 张 mood / 沿用已批 keyframe 作为唯一视觉标准 |
| **Skill** | brainstorm：`Format Gate` 与 `Style Gate` 分离；「ok」仅锁定 duration/aspect |

### P0-2：已验收 keyframe 未用于成片

| 项 | 内容 |
|----|------|
| **现象** | PNG 上是居中整句「Going to Ubaid」；成片是编辑器重排版 |
| **根因** | 先生成图、后改口「全程 video-editor text layers」，路径断裂 |
| **重做** | 二选一：**A)** 以已批 PNG 为底图序列 + 只 anim D/plane；**B)** 不生成 PNG，直接 editor 但 layout 对齐 script |
| **Skill** | generation/assembly：若 `analyze_file_content` 已通过 keyframe，禁止另起一套 typography |

### P0-3：文案编排破坏字谜

| 项 | 内容 |
|----|------|
| **现象** | 显示「Going to」+「U B A I」+「D」，不是「Ubaid」里抽出 D |
| **根因** | `edit_spec` 拆成 3 个 text track，未按单词建模 |
| **重做** | 单层或联动层：`Ubaid` 保留为词，`D` 从词内 index 前移；或整句 PNG morph |
| **Skill** | script-skill：typographic pun 类需求 → 禁止拆成 spaced caps |

### P0-4：飞机为扁平贴纸，非「穿字」

| 项 | 内容 |
|----|------|
| **现象** | 白色剪影在 Z 轴上层横扫，与字母无遮挡关系 |
| **根因** | `plane_icon` prompt 要求 flat silhouette + masking；实现为 overlay 位移 |
| **重做** | 飞机层与字层同合成：mask / track matte / 字被翼遮挡；或 3D 透视穿过字缝 |
| **Skill** | 用户说 cross the letter → 强制「与字形同一 compositing 空间」，禁 flat icon sweep |

### P1：跳过 script / 风格文档

| 项 | 内容 |
|----|------|
| **现象** | 读了 skill 文件但未产出 sequence breakdown |
| **根因** | 误判为「简单 MG」可直做 |
| **解决办法** | 即使 &lt;15s，typographic + 多 beat 仍走 script（timing + visual anchor） |

---

## 四、失败因果链

```
用户只描述动效，未给风格
  → Agent 单方面定为 dark navy / gold MG
  → 用户「ok」仅确认 16:9 时长
  → image_generate 做 keyframe 但 execute_edit_video 改用 Montserrat 重排
  → 「U B A I」+ 独立 D，字谜失效
  → 扁平 plane_icon overlay，非穿字
  → V1 Wrong visual style
```

---

## 五、推荐重做产线（V1 正确版）

```
1. 意图澄清：动效逻辑 + 1 张 style mood（或确认「极简 MG / 旅游风 / 写实」三选一）
2. script-skill：
   - Beat1: 「Going to Ubaid」整句 hold
   - Beat2: D 从 Ubaid 内前移 → 「DUBAI」
   - Beat3: 飞机与字同层穿过（定义 mask 关系）
3. 生产（二选一，全程一致）：
   A) 已批 keyframe PNG 为底 + editor 只 anim D/plane/mask
   B) 纯 editor：单句 text + 子字符串 D 的 position anim + plane mask
4. 禁止：无参考的 flat icon 横扫；禁止 keyframe 与成片两套字体
5. show_final_video → 用户确认后再考虑 Habibi/写实飞机等 V2 需求
```

### edit_spec 要点示例（方向）

```json
{
  "text": "Going to Ubaid",
  "plane": "mask_composite_with_text_not_overlay_icon",
  "style_source": "approved_frame_going_to_ubaid.png OR user_mood_ref"
}
```

---

## 六、Skill 改进建议（防复发）

| 规则 | 写入位置 | 要点 |
|------|----------|------|
| Style Gate 独立于 Format Gate | brainstorm-skill | duration/aspect 与 palette/写实度分开确认 |
| Typographic pun 编排 | script-skill | 禁止 `U B A I` 空格拆分；D 必须来自词内 |
| Keyframe 一致性 | generation-skill / assembly-skill | 已批 PNG 不得被 editor 另一套字体替代 |
| cross the letter | assembly-skill / modification-skill | 飞机须 mask/matte 与字同层，禁 flat overlay sweep |
| 短项目仍要 script | brainstorm-skill | ≤15s 但有 ≥2 beat + typographic → 必须 sequence breakdown |

---

## 七、数据文件索引

| 文件 | 说明 |
|------|------|
| `analysis/langfuse-data/cases/19598599427/trace-3-f8e08738.json` | V1 主 trace |
| `analysis/langfuse-data/cases/19598599427/trace-1-a171f5b9.json` | 用户首条需求 + Agent 风格提议 |
| `analysis/langfuse-data/cases/19598599427/assets-with-prompts.json` | 素材与 prompt 映射 |
| `analysis/langfuse-data/cases/19598599427/qa-report.html` | 可视化 QA 看板 |
| `analysis/langfuse-data/cases/19598599427/media/` | 已下载媒体（plane 片段等） |
