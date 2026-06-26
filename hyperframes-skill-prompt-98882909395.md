# 项目 98882909395 — HyperFrames 合成时的提示词

## 概述

项目 98882909395 在使用 HyperFrames 合成时，LLM 读取了以下 skill 文件作为指导提示词：

1. `/.skills/0/hyperframes-skill/SKILL.md` — 主文档（47,272 字符）
2. `/.skills/0/hyperframes-skill/references/composition-rules.md` — 合成规则
3. `/.skills/0/hyperframes-skill/references/composition-skeleton.md` — HTML 骨架模板（43,931 字符）
4. `/.skills/0/hyperframes-skill/references/design-motion.md` — 动效设计原则（11,207 字符）
5. `/.skills/0/hyperframes-skill/references/tech-html.md` — HTML 技术规范
6. `/.skills/0/hyperframes-skill/references/tech-tools.md` — 工具使用规范（38,208 字符）

---

## SKILL.md 核心内容

### 名称与描述

```
name: hyperframes-skill
description: "HTML-as-source-of-truth video composition via the hyperframes MCP.
Author a single self-contained HTML composition (all scenes + PiP overlays inlined
as direct child divs of the root wrapper), then lint and render via the MCP."
```

### Quick Reference Card（5 条最重要规则）

| 关注点 | 规则 |
|--------|------|
| **Brand-mark cutout layers** | 使用 `remove_background` 工具，不要在 HTML 中手动实现抠图 |
| **Font selection** | 始终使用 `hyperframes.list_fonts`，绝不使用 `video-editor.list_fonts`（两个目录完全不重叠） |
| **`html_file` 参数** | 传入 `write_file` 写入的虚拟路径，中间件自动转换为 `asset://` |
| **HTML body 中的媒体引用** | 必须是 `https://` 签名 URL（通过 `get_file_info` 获取），虚拟路径和 `asset://` 在 body 中不起作用 |
| **`write_file` / `edit_file` 内容大小** | 每次调用硬限制 ≤ 30 KB，安全余量 ≤ 25 KB |
| **Entry gate** | 不能在 `summarize_url` 之后直接进入 hyperframes，需要先确定 `audio_intent`。当 intent ≠ `explicit_silent` 时，`vo_*`/`bgm_*` 资产必须在第一次 `write_file` 之前存在 |

### Strategy Gene

**Domain**: HTML composition authoring, GSAP timeline scheduling, `data-*` clip metadata, font catalog selection, `asset://` HTML upload contract, lint-before-render discipline, NVENC GPU render submission, asynchronous render polling, burned-in subtitle integration.

**Summary**: Write an HTML composition with a paused GSAP timeline, upload to asset-service, lint to zero errors, then submit an async GPU render and poll until done.

### Reference Read Order

1. `html-file-contract.md` — 虚拟路径 → `asset://` 中间件转换
2. `iterative-build.md` — 大于 25KB 的合成需要迭代构建
3. `composition-skeleton.md` — 根属性、timeline 注册、单文件结构
4. `composition-rules.md` — Layout-Before-Animation、Timeline Contract、确定性规则、场景过渡
5. `typography-and-fonts.md` — 字体选择和排版
6. `motion-principles.md` — 动效设计（缓动、时长、错开间隔）
7. `design-captions.md` — 字幕支持
8. `design-camera-cursor.md` — 虚拟镜头运动
9. `error-handling.md` — 错误处理

### Tool Use Logic

1. `hyperframes.list_fonts` — 枚举字体（绝不使用 video-editor.list_fonts）
2. `get_file_info` — 获取签名 URL 用于 HTML body 中的媒体引用
3. `hyperframes.lint_composition` — 每次编辑后、每次渲染前必须 lint
4. `hyperframes.render_frame` — 单帧验证
5. `hyperframes.probe_media` — 验证媒体时长/编码
6. `hyperframes.submit_render` — 提交渲染（lint 通过后）
7. `hyperframes.query_render` — 轮询渲染状态

### Decision Principles

1. `html_file` scheme 是最常见的失败源
2. Lint 成本 < 渲染成本（lint 2秒 vs 渲染 30-90秒）
3. Composition root 需要三个结构属性
4. GSAP timeline 是 paused + frame-seek 驱动，不是 autoplay
5. 工具结果的 `agent_hint` 字段是规范的修复指令
6. Asset references 必须使用 resolver-aware URLs
7. Video 和 audio 是独立的媒体元素
8. 动画 wrapper divs，不要直接动画媒体元素
9. Brand-mark cutout 使用 `remove_background` 工具

### AVOID（硬性禁止）

- 在没有 `audio_intent` 的情况下进入 hyperframes
- 当 intent ≠ `explicit_silent` 时交付静音视频
- 当 HTML 没有 `<audio>` 元素时提交渲染（intent ≠ explicit_silent）
- 在 lint 成功之前提交渲染
- 传入 `file:///...` 路径
- 在 HTML body 中嵌入虚拟路径
- 单次 write_file 超过 30KB
- 使用 `data-composition-src`（多文件不可行）
- 跳过 `read_file` 验证
- 使用 `video-editor.list_fonts`
- 使用 `setTimeout`/`requestAnimationFrame`/`Math.random()` 等非确定性 API
- 在 `<video>` 或 `<audio>` 上调用 `.play()`/`.pause()`
- 直接动画 `<video>` 或 `<audio>` 元素
- 使用 `<br>` 换行（应使用 CSS `max-width`）
- 在非最后一个场景上使用退出动画

### Entry Gate（音频入口门控）

```
Never first production step after summarize_url.
Require script audio_intent.
When intent ≠ explicit_silent, vo_*/bgm_* assets must exist before first composition write_file.
```

### 渲染后验证

```
probe_media(final output) → streams must include audio unless explicit_silent;
video-only → re-lint HTML audio refs or escalate mux
```

---

## 该项目的实际执行

**项目内容**：手写动画风格的动态排版视频（Kinetic Typography）
- 依次手写出现 4 个词：THINK → CREATE → BUILD → MAKE IT HAPPEN
- 黑底白字，Montserrat 粗体
- SVG stroke-dashoffset 动画模拟手写效果
- 总时长约 18.5~20 秒
- 使用 HyperFrames 合成（HTML + GSAP 动画）

**LLM 读取的 skill 文件**：
1. `SKILL.md`（10:56:08）
2. `composition-rules.md`（10:56:08）
3. `composition-skeleton.md`（10:56:08 & 10:56:14）
4. `design-motion.md`（10:56:24）
5. `tech-html.md`（10:56:24）
6. `tech-tools.md`（10:58:49）
