# 拼贴动画风格偏差分析

## 范围

分析项目：`95437570412`、`43896078339`、`19554823058`、`81808640274`。

共同问题：用户明确要求拼贴动画，但最终成片更像普通照片/场景视频叠加纸张装饰，未形成“纸片主体 + 剪裁边缘 + 逐帧摆放/翻动”的运动语法。

## Case 分析

### 95437570412：旅行行李整理

**根因**

- 为修复 checklist 勾选位置，生成中的勾选被移到最终合成，破坏了纸片运动的一致性。
- 步骤转场被明确改成 smooth crossfade，而不是用户期待的 page-flip / stop-motion 翻页。
- 因此结果是连续动效加拼贴元素，而不是剪纸层被逐帧操作。

**解决方案**

- 将 page-flip、纸片滑入、贴纸落下、逐帧抖动设为不可降级的 motion primitives。
- 勾选位置在同一条拼贴时间轴中确定性绘制，不要把整个 beat 抽离到普通后期合成。
- 风格验收禁止 smooth crossfade 作为唯一转场。

### 43896078339：博物馆展览搭建

**根因**

- 用户要求使用展品照片、平面图、标签、档案剪报等真实材料，但素材未上传。
- 后续流程允许自行生成替代视觉，导致主体变成 AI 生成的博物馆场景，纸张、胶带、线条只剩装饰作用。
- 最终输出为 1280x720，也表明该项目走了替代生成/合成路径，未保持与目标拼贴制作的同等约束。

**解决方案**

- 缺少真实素材时，不能静默把“真实物件拼贴”降级成写实场景。
- 要么等待素材，要么明确切换成“纯虚构纸艺拼贴”并重新确认方向。
- 展品卡片、平面图、标签纸片必须成为画面主体和可移动层，而不是背景 overlay。

### 19554823058：夜市摊位开张

**根因**

- 规划把生成的夜市照片作为主体，再叠加纸灯笼、蒸汽、票根、贴纸。
- 执行记录说明这些 collage accents “small and scattered”，视觉重心因此仍是普通场景视频。
- 摊位、老板、食材、菜单、顾客没有被拆成可独立操作的剪纸层。

**解决方案**

- 将摊位、人物、食材、菜单、灯笼等拆成独立纸片/照片剪裁层。
- 每个 beat 必须包含放置、滑入、翻面、遮挡或纸张抖动等明确动作。
- 若主体仍是连续摄影、纸艺元素只占角落，应判定风格不合格并重生成。

### 81808640274：从手稿到书架

**根因**

- 采用“先生成 first draft beat，再续跑其余 beats”的分阶段流程，后续段落存在风格漂移。
- 文本保真和照片排版成为主约束；拼贴被简化成 grain、torn edges、proof marks 和 slow parallax。
- slow parallax 不能替代 stop-motion 的纸片摆放和离散运动。

**解决方案**

- 在第一段生成前锁定覆盖全部五段的统一 style manifest，后续 beat 不得重新解释风格。
- 将 manuscript、sticky notes、cover、proof marks、printer photos 转成独立纸片层。
- 禁止只用 slow parallax 代表拼贴动画；每段都必须通过同一套纸片动作、剪裁边缘和离散运动验收。

## 系统性修复

1. 规划层将 `collage_animation` 定义为结构化 style type，而非普通形容词。
2. 生成层强制声明纸片主体、剪裁边缘、贴放/翻页/胶带动作和离散或逐帧运动。
3. 验证层检测纯照片平移、parallax、overlay 或 crossfade，并将其判为不满足拼贴风格。
4. 交付层只有通过风格门禁才能标记 `FINAL`；失败时重生成或明确告知用户发生了降级。

## 对现有拼贴动画 Prompt 模板的修改建议

现有模板已经覆盖纸片材质、持续运动、镜头、转场、字幕、音频和 `DO NOT` 项，因此主要缺口不是继续增加形容词，而是把规则变成不可覆盖、可验证的约束。

### 1. 增加风格优先级和冲突覆盖规则

将以下内容放在模板最前面，防止通用模板或后续 beat 重新解释风格：

```text
STYLE PRIORITY: If the user requests paper collage, scrapbook, cut-paper,
paper-craft, mixed-media collage, or stop-motion collage, this style is a
hard requirement. It overrides generic photorealistic video, isometric,
parallax, flat-vector, slideshow, and motion-graphics treatments.
The style must remain consistent across every beat and every revision.
```

### 2. 明确“拼贴层是主体”，禁止照片加装饰的降级

在 `STYLE` 段落后加入：

```text
The collage layers are the primary subjects and occupy the visual focus.
Do not generate a normal photorealistic scene and add small paper decorations
on top. Every beat must be constructed from independently movable paper,
photo-cutout, clipping, label, tape, or card layers.
```

这条直接针对 `43896078339` 和 `19554823058` 的失败模式。

### 3. 把“unseen pair of hands”改成可执行的交互要求

现有措辞容易被模型理解为氛围描述，建议改为：

```text
Human manipulation is continuous and causally visible. In every beat, show
hands, fingers, tweezers, pins, tape pulls, or another physical mechanism
actively rearranging the layers. Do not replace human manipulation with
autonomous object motion alone.
```

### 4. 增加后处理保真规则

在 `TRANSITIONS` 或 `DO NOT` 后加入：

```text
POST-PROCESSING MUST PRESERVE THE COLLAGE MOTION LANGUAGE.
Deterministic corrections such as checkmarks, labels, or text must be
rendered as moving paper layers inside the same timeline. Do not replace a
paper action with a vector overlay, static composite, smooth crossfade,
slow parallax, or isolated UI correction.
```

这条针对 `95437570412` 中为修复勾选位置而牺牲拼贴运动的情况。

### 5. 限制分阶段生成的风格漂移

在 `STORY` 前加入：

```text
Before generating any beat, lock one style manifest for the complete video.
All later beats, retries, revisions, and assembled segments must inherit the
same paper materials, layer behavior, motion vocabulary, camera language,
and transition rules. A continuation must not switch to photo layout,
parallax, slideshow, or generic explainer motion.
```

这条针对 `81808640274` 的 first-draft / continuation 风格漂移。

### 6. 明确缺少用户素材时的两种合法路径

在素材相关指令后加入：

```text
If user-supplied reference material is required but missing, do not silently
switch to a normal generated scene. Either wait for the material, or obtain
explicit approval to use fully fictional paper-collage source material.
When fictional material is approved, it must still be built as movable paper
layers, not as a complete scene with collage overlays.
```

### 7. 增加最终风格门禁

现有 `DO NOT` 是生成提示，不足以阻止错误结果交付。建议在模板末尾增加：

```text
FINAL DELIVERY GATE: Do not mark the video final if any beat has fewer than
three independently moving paper elements, if landed elements freeze, if the
main subject is a normal photorealistic scene, if collage pieces are only
decorative overlays, or if dissolves, smooth crossfades, slow pans, or pure
parallax are the primary motion language. Regenerate the failed beat or
report the degradation explicitly.
```

### 8. 建议同步传给路由和验证器的结构化字段

长 prompt 之外，建议由模板编译成结构化参数，避免关键规则在长文本中被忽略：

```json
{
  "style": "paper_collage_stop_motion",
  "style_hard_requirement": true,
  "continuous_layer_motion": true,
  "min_active_paper_elements": 3,
  "min_visible_changes_per_second": 2,
  "required_manipulation": true,
  "allowed_transitions": ["paper_swipe", "tear_away", "hand_clear", "hard_cut"],
  "forbidden_primary_motion": ["dissolve", "smooth_crossfade", "slow_pan", "pure_parallax"],
  "final_style_gate": true
}
```

这样模板修改才会真正贯穿“意图识别 → 路由 → 生成 → 后期 → 验收 → 交付”，而不是只改变 agent 对用户的描述。

## 证据说明

本报告基于项目历史、执行消息和资产元数据整理。直接记录包括：`95437570412` 使用 smooth crossfade、`43896078339` 输出 1280x720、`19554823058` 使用小面积分散的 collage accents、`81808640274` 采用分阶段续跑。Metabase MCP 在分析时鉴权失败，因此未将未验证的数据库字段作为证据。
