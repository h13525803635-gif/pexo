# Case 16540703029 — 行星静止、无声、两段未合成

## 结论

项目 `16540703029` 的用户需求是一个太阳系行星尺寸科普视频：左侧展示行星球体和 3D flip reveal，右侧展示直径对比柱状图。最终交付出现三个问题：

1. 左侧行星不是用户预期的“持续自转”形态。
2. 成片没有声音。
3. 最终只交付 `solar-part-a.mp4` 和 `solar-part-b.mp4` 两段，没有合成单个文件。

这三个问题不是用户需求冲突，而是执行链路选择和交付门禁缺失导致。

## Trace 证据

### 1. 生成路线是 HyperFrames HTML 动效，不是真 3D/视频生成

trace 中用户需求为：

> Make a science explainer comparing planet sizes in our solar system. Start with Mercury and progressively grow each planet's sphere to scale. Use a 3D flip transition between each planet reveal. Show a growing bar chart on the side comparing diameters as each planet appears.

Agent 选择了 `hyperframes-skill`，构建 HTML/GSAP composition。后续摘要记录：

- `composition.html`：完整 76s HTML composition。
- `part-a.html`：intro + Mercury-Mars。
- `part-b.html`：Jupiter-Neptune + finale。

这说明行星视觉是 DOM/CSS/GSAP 合成出来的球体，不是通过 3D 引擎或视频模型生成的真实自转行星素材。

### 2. 自转语义没有落实为持续旋转动画

用户说“行星在画面左边应该是自转的形态”，但 trace 摘要只记录了：

- `3D flip card transitions per planet reveal`
- `planet spheres scaling to accurate relative sizes`
- `growing bar chart`

实现重点放在 reveal 翻转、大小缩放和柱状图增长。没有看到明确的 planet texture rotation / continuous spin / CSS keyframes rotate / timeline loop 作为每个行星球体的持续自转层。

因此左侧行星看起来像静态球体或只在入场时有过渡动效，而不是持续自转。

### 3. 音频没有进入生产和绑定链路

trace 中实际生产路线围绕 HyperFrames composition/render 展开，交付物是两个 `video/mp4`：

- `/projects/16540703029/workspace/assets/solar-part-a.mp4`
- `/projects/16540703029/workspace/assets/solar-part-b.mp4`

在收尾 trace 可见的 concat HTML 中只有两个视频元素：

```html
<video id="partA" ... muted playsinline></video>
<video id="partB" ... muted playsinline></video>
```

没有 `<audio>` 元素，也没有看到 `audio_produce`、`music_generate`、TTS/VO、BGM 生成和绑定的有效证据。对于非明确 silent 的科普视频，这应被交付门禁阻断。

### 4. 两段未合成是因为用了 HyperFrames 重新渲染 concat，而不是无重编码拼接

最终已有两个片段：

- `solar-part-a.mp4`：30.8s，Intro → Mercury → Venus → Earth → Mars。
- `solar-part-b.mp4`：39.7s，Jupiter → Saturn → Uranus → Neptune → Finale。

Agent 写了 `/projects/solar-explainer/workspace/assets/concat.html`，把两个 mp4 作为 `<video>` 顺序放进 70.5s 的 HTML composition 里再 `submit_render`。

多次 concat render 均停在 pending：

- `3evk1rrs`：standard，pending。
- `7yy1r4cv`：high，pending。
- `bmr9jakb`：draft，pending。

这不是“两个视频无法合成”的本质问题，而是合成方法错了：HyperFrames 需要重新渲染 70s HTML，触发长任务排队/超时问题。正确方式应使用 assembly/video-editor 的 concat 或 ffmpeg copy/mux 类无重编码拼接。

## 根因

### 根因 1：把“3D flip / sphere”理解成 HTML 动效，而没有补齐自转动画

需求中的 3D flip transition 被实现了，但“行星球体应持续自转”的视觉预期没有被建模为必要动画层。应显式为每个 planet sphere 添加持续 texture rotation / surface band movement / CSS custom property 驱动的转动效果。

### 根因 2：缺少音频意图门禁

科普视频默认不应无声。Agent 没有先建立 `audio_intent`，没有生成 VO/BGM，也没有在最终 HTML 或合成里挂载 `<audio>`。最终交付前也没有 probe 最终 mp4 的 audio stream。

### 根因 3：使用渲染型 concat，而不是装配型 concat

两个 mp4 已经产出后，后续需求是 post-production assembly。Agent 应切到 `assembly-skill` / video-editor 类工具做无重编码拼接和音频混合，而不是把两个视频塞进 HyperFrames 再渲染一遍。

### 根因 4：交付门禁接受了“两段即完整交付”

用户要的是一个视频，最终却把两段作为完整交付，并让用户用 QuickTime/CapCut/ffmpeg 自行拼接。这不符合成片交付标准。

## 修复方案

### 当前项目修复

1. 对 Part A / Part B 做无重编码拼接，产出单个 final mp4。
2. 若 Part A / Part B 本身无音轨，生成 70.5s 科普类 BGM，或加旁白 + BGM。
3. 将音频 mux 到最终单文件。
4. 如果要修复左侧行星自转，需要回到 HTML composition：
   - 为每个 planet sphere 增加持续 spin 动画。
   - 确保 split 后 Part A / Part B 都保留同样的自转逻辑。
   - 重新渲染两段，再用 assembly concat。
5. 交付前 `probe_media` 最终 mp4，确认 video + audio stream 均存在。

### 系统级修复

1. 非 silent 视频必须建立 `audio_intent`，并在交付前确认有 audio stream。
2. 已有视频片段合成时，优先走 assembly/video-editor concat，不要用 HyperFrames 重新渲染长视频。
3. 对“single final video”需求增加门禁：不能把多个 part 当最终交付。
4. 对 planet/sphere/rotate/spin 类视觉需求增加检查：如果用户期望自转，必须有持续旋转动画，而不只是 reveal transition。
