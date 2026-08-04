# 项目 99151874044：用户确认角色细节后仍反复出现角色一致性问题的根因与解决方案

## 结论

这不是用户指令不清，也不是参考图完全没有传入生成调用。用户的角色说明、补充图片和逐轮确认都被读取，并在多个 prompt 中复述。

真正的问题是，系统把角色修改当成了局部镜头的 prompt 调整，没有把它们升级为全项目、双角色、版本化且可验证的角色契约。旧的双人参考图、仅描述狼男的局部参考图、尾巴参考图和镜头构图图被混合传入不同的独立生成调用；没有一份被所有镜头实际消费的完整角色 Manifest，也没有在拼接前检查每一段是否符合当前角色版本。

因此，每个 `reference2video` 调用都在重新解释不完整且互相重叠的条件。开场、后续片段和最终成片并不共享可强制执行的角色身份状态，最终出现狼尾缺失、猫娘多尾、猫娘 glitch、发型和耳朵漂移等问题。

因果链：

`用户提供双角色图和故事 -> 初版 Creative Brief 锁定早期外观 -> 用户后续补充双色发、刘海、狼尾、猫尾和小猫耳 -> 只写入狼男局部锁定文件 -> 新旧参考图在不同镜头中混用 -> 各段独立 reference2video 重生成 -> 未对每段执行角色一致性 QA -> 9 段未经身份门禁直接拼接 -> 用户在成片中发现尾巴、角色和解剖问题`

## 调查范围与数据完整性

- 项目 ID：`99151874044`
- 数据源：Langfuse 生产 trace
- trace 名称：`pexo:99151874044`
- 时间范围：2026-08-01 17:14:51Z 至 23:02:50Z
- 已发现 trace：30 条
- 已取得轻量 trace 记录：30/30 条
- 已恢复 observations：30/30 条，共 1,309 条
- 完整 trace detail：首条成功取得；其余 detail 接口响应持续卡顿，未将其作为唯一证据源
- 诊断证据来自：用户消息、trace output 中的工具调用参数、已写入的 brief/lock 文件、assembly 参数和最终逐帧评论
- 不提交原始 trace、用户图片、媒体文件或临时 URL 到 GitHub

## 用户已明确确认的角色要求

用户的要求不是一次性模糊描述，而是逐轮收敛：

1. 初始图和故事明确了两名核心角色及其关系：一名内向狼男与一名开朗猫娘。
2. 用户确认全曲、16:9，并明确“她是 neko girl，他是 wolf boy”。
3. 用户补充狼男的黑白狼尾、灰白双色内层、遮眼刘海，以及猫娘的小猫耳和猫尾。
4. 用户多次校正灰色发色的亮度、长度、刘海位置、镜头角度和开场动作。
5. 用户确认开场通过，并明确其他片段采用最新狼男发型参考。
6. 交付后的逐帧评论仍要求：狼尾全程可见、开场狼尾蜷在身侧、删除猫娘多余尾巴，并修复猫娘 glitch。

这些要求足以形成可执行角色规范；问题在于系统没有将它们汇总为统一、可追溯的生产状态。

## 关键执行时间线

| 时间（UTC） | Trace | 行为与证据 | 结论 |
|---|---|---|---|
| 17:14:51 | `7010a2e8199bda97f997c2451ef77a70` | 用户上传双角色参考图，定义狼男、猫娘、关系和剧情 | 初始意图和角色素材明确 |
| 17:21:39 | `891ab2dce692952f2bd5d2ae85cd3f05` | 用户确认全曲、16:9，并补充“她是猫娘、他是狼男”；系统写入 `creative_brief__locked__R01.md` | R01 只记录早期外观：狼男黑色乱发、猫娘棕色猫耳；尚未含后续的双色发和具体尾巴要求 |
| 17:30:38 | `8cfb51aeda84321c300942d7e9ba853c` | 用户新增黑白狼尾、灰白内层、遮眼刘海、小猫耳和猫尾 | 角色基线发生实质性变化，应该创建新的双角色版本 |
| 18:41:24 | `274e0a5d9d6992a9324782c36d168399` | 开场 `seg01_riverbank_v4` 的输入为镜头构图图、狼尾图和狼男参考图 | 该镜头没有猫娘的独立视觉参考，猫娘只能由文字补全 |
| 19:11-19:55 | `adab6f3107cd5e23d2936d93df49b5e7` 至 `5174e3ab190bd47be69b973f20679714` | 用户持续校正狼男的灰色调、发长、刘海、灰色仅限颈部发梢及开场镜头 | 修正只在单镜头和单角色层级迭代，没有更新全局角色契约 |
| 20:12:49 | `34cb2851906aef420bc16544a6ba7cef` | 用户确认开场，要求其他段采用新发型；系统仅写入 `wolf_boy_char_ref_LOCKED.md`，标注适用 Segments 2-9 | 锁定文件只覆盖狼男发型、耳朵、脸、衣服、饰品；不含狼尾，也不含猫娘完整定义 |
| 20:14:58 起 | `d3c639b6a99e10514bbd69394335fac9` 等 | Segments 2-9 分别调用 Kling `reference2video`；每段使用狼男单人参考图和早期双人原图 | 两名角色并未以明确的独立身份绑定输入；旧双人图继续与新狼男图混用 |
| 22:34:36 | `22b620f75cffc28ee9420904bdec9d84` | 9 段视频直接 convert 后 concat | 没有观察到逐段角色一致性验收作为 assembly 前置条件 |
| 22:44:32 | `a36b9c0ed4ef4a974965229acf07f230` | replace audio 后调用 `show_final_video` 交付 | 最终交付前没有观察到角色、尾巴数量或 glitch 的全片 QA |
| 23:02:50 | `799ec94b0794a9281914f741b17d741d` | 用户对最终视频逐帧标注狼尾缺失/姿态、猫娘多尾和猫娘 glitch | 成片暴露了此前未被验证的身份与解剖一致性问题 |

## 根因分层

### 1. 状态层：锁定的是过期的初版设定，而不是当前双角色版本

`creative_brief__locked__R01.md` 创建于用户后续外观修改之前，仍把狼男描述为黑色乱发。之后用户新增并反复细化的双色发、刘海、狼尾、猫尾和小猫耳没有生成新的 Creative Brief 或完整 Subject Manifest。

这导致系统同时保留：

- 初始双人原图及其早期外观；
- 狼尾单独参考图；
- 多个不同版本的狼男发型图；
- 面向 Segments 2-9 的狼男局部锁定文件；
- 未被版本化的猫娘文字描述。

这些不是同一个可消费的角色版本，也没有 lineage 表明哪个版本应覆盖旧状态。

### 2. 条件层：参考图被传入，但角色职责和版本互相冲突

开场 v4 的三张输入分别承担镜头构图、狼尾和狼男角色，没有猫娘的独立参考图。猫娘外观只能由 prompt 生成，因此无法保证与初始图或其他镜头保持同一身份。

Segments 2-9 使用“已生成的狼男单人参考图 + 初始双人原图”。初始双人图不是隔离的猫娘角色图，它既包含两名角色，也代表用户后来修改之前的外观。prompt 虽然用“image 1 是狼男、image 2 是猫娘”这样的文字指定角色，但第二张图本身包含两个角色，不能作为稳定的单角色绑定。

因此，模型收到的是多个版本、多个主体、不同职责的图像条件；系统没有为它提供“狼男必须使用版本 X，猫娘必须使用版本 X”的结构化约束。

### 3. 生成层：每段独立生成，没有跨镜头身份继承

Segments 2-9 是多个独立的 Kling `reference2video` 调用。调用之间没有可观察到的身份状态、manifest version 或角色 QA 结果传递。每段仅依赖当次 prompt 和当次 `image_list`，所以模型会在每段重新生成发型、耳朵、尾巴和面部细节。

“同一角色参考图被多次附带”只能提供软条件；在没有结构化身份绑定和输出验证时，不能等价于跨镜头角色锁定。

### 4. 确认语义层：用户确认没有升级为全局生产约束

用户说“opening scene is fine”“For the other segments, I want his hairstyle like this image”“the hair is perfect”时，表达的是当前确认的角色外观应覆盖后续镜头。系统只写了一个狼男局部文件，并没有建立双角色 `identity_version`，也没有使 R01 和已生成的不兼容素材失效。

此外，观察到该锁定文件被写入，但后续没有对应的 `read_file` 记录；即使 prompt 中手工复制了部分描述，也没有工件级消费和校验机制。

### 5. 验证与交付层：没有检查角色一致性，直接拼接和交付

链路中的 `analyze_file_content` 用于分析原始图、发型参考、尾巴参考、镜头构图和开场片段。没有看到 Segments 2-9 或最终成片被检查以下关键条件：

- 狼男是否保持同一发长、双色分布、遮眼刘海、衣着和饰品；
- 狼尾是否恰好一条、黑白配色正确并在要求的镜头中可见；
- 猫娘是否保持小猫耳、恰好一条猫尾、服装和外观一致；
- 是否有多尾、消失的尾巴、额外肢体或人物 glitch；
- 同一角色是否跨 9 段保持当前身份版本。

最终 assembly 直接拼接 9 段并替换音乐后交付，未将角色 QA 作为门禁。用户的逐帧反馈因此成为第一次系统性发现这些问题的检查。

## 已排除的替代解释

- **不是用户没有提供参考图**：初始双人图、尾巴图、发型图、开场视频和逐帧标注均被提供。
- **不是 Agent 完全忽略文字要求**：多个 prompt 复述了双色发、刘海、耳朵、服装和尾巴描述。
- **不是单一模型随机失败即可解释**：不同片段使用的参考组合并不一致，且没有统一状态和 QA；这是流程性问题，会系统性放大随机漂移。
- **不是只发生在开场**：最终成片评论覆盖开场、多个中段和后段，包含狼尾缺失、猫娘多尾和角色 glitch。
- **不是拼接本身创造了多尾或发型**：这些属于源视频视觉内容；拼接的错误是未检查并阻止不合格源片段进入成片。

## 解决方案

### P0：用完整 Subject Manifest 替代局部角色锁定文件

- **Owner**：subject-asset-skill / generation-skill
- **触发条件**：用户上传角色参考、修改角色特征，或说“后续片段都使用这个角色设计”。
- **行为**：创建并版本化 `Subject Manifest`，同时定义狼男和猫娘；每名角色有独立 canonical reference、当前 version、不可变特征、可变状态和禁止项。
- **必填字段**：

```json
{
  "identity_version": "duo_v4",
  "wolf_boy": {
    "canonical_ref": "wolf_boy_duo_v4.png",
    "required": ["grey nape tips only", "bangs cover left eye", "one black-to-white wolf tail"],
    "forbidden": ["missing tail when full body is visible", "extra tail", "pure black hair"]
  },
  "neko_girl": {
    "canonical_ref": "neko_girl_duo_v4.png",
    "required": ["small brown cat ears", "one slender cat tail", "white ruffled top", "denim jeans"],
    "forbidden": ["wolf ears", "more than one tail", "extra limbs"]
  }
}
```

- **门禁**：不得只写入单一角色的局部文档后继续生产双角色视频。
- **回归断言**：每个双角色生成单元必须引用同一 `identity_version`，且含两名角色的 canonical binding。

### P0：角色修改必须创建新版本并使旧素材失效

- **Owner**：modification-skill / project state
- **触发条件**：发型、耳朵、尾巴、服装、脸部、物种等身份特征发生修改。
- **行为**：创建 `duo_vN+1`，更新 Creative Brief、Subject Manifest 和 Generation Handoff；标记旧版本不兼容。
- **门禁**：新版本之后生成的镜头不得继续使用旧双人原图作为角色参考，除非该图被重新验收为新版本的 canonical asset。
- **回归断言**：角色变更后，任何 `video_generate` 输入中的 asset version 均等于当前 manifest version。

### P0：为每个角色建立清晰的参考职责

- **Owner**：generation router
- **行为**：狼男和猫娘分别使用隔离的 canonical reference；双人构图使用单独的 duo/layout reference；尾巴细节通过已合并到角色参考的图或 manifest 字段表达。
- **门禁**：禁止把包含两名角色的旧图同时当作单一角色参考使用；禁止用镜头构图图、尾巴图和单一角色图替代另一位角色的视觉参考。
- **回归断言**：对双角色镜头，payload 的每一个 `image_list` 条目都必须在 Subject Manifest 中声明 `subject_id` 和 `role`。

### P0：在每个片段和 assembly 前执行结构化角色 QA

- **Owner**：generation QA / assembly-skill
- **触发条件**：每次 video generation 成功后，以及最终 concat 前。
- **行为**：逐片段检测角色身份、尾巴数量、耳朵类型、发色区域、刘海位置、服装/饰品、重复肢体和人物 glitch。
- **门禁**：任何 required feature 缺失或 forbidden feature 出现时，禁止素材进入 concat。
- **回归断言**：9 段 source asset 必须各自有最新的 `identity_version` QA，且 `hard_constraint_pass = true`。

建议的 QA 输出：

```json
{
  "asset_id": "seg06_protector_v2",
  "identity_version": "duo_v4",
  "wolf_tail_count": 1,
  "wolf_tail_visible_when_required": true,
  "neko_tail_count": 1,
  "neko_ear_type": "small_cat",
  "hair_contract_pass": true,
  "anatomy_glitch_detected": false,
  "hard_constraint_pass": true
}
```

### P1：将用户确认区分为镜头确认与全局角色确认

- **Owner**：interaction / modification workflow
- 当用户说“其他片段使用此发型”“角色已经完美”等跨镜头表达时，默认更新全局角色版本，或用一句确认问题澄清范围。
- 确认后显示当前 `identity_version`、覆盖的角色和将失效的旧素材，避免用户以为局部预览已经锁定全片。
- **回归断言**：跨镜头确认之后的每次生成调用都携带新版本，不得继续读取旧版本 reference bundle。

## 回归测试

1. **完整性测试**：双角色项目没有同时覆盖两名角色的 Subject Manifest 时，阻断视频生成。
2. **版本失效测试**：用户修改发型或尾巴后，断言旧版本素材不能进入新的 concat。
3. **参考职责测试**：双角色生成 payload 中，每张参考图都必须有 `subject_id`、`role` 和 `identity_version`。
4. **狼尾测试**：用户要求全程可见时，逐帧或逐关键镜头断言狼尾存在且仅一条；开场支持“蜷在身侧、脏污”等状态字段。
5. **猫娘测试**：断言小猫耳、恰好一条猫尾、无额外尾巴和无额外肢体。
6. **跨段一致性测试**：比较 Segments 1-9 的角色 embedding、颜色区域、服装/饰品和发型结构，低于阈值时阻断 assembly。
7. **QA 门禁测试**：任何 source segment 缺少最新 `hard_constraint_pass: true` 记录时，`media_process concat` 必须失败。
8. **交付测试**：最终成片在 `show_final_video` 前，必须完成覆盖全片的角色一致性和 glitch 检查。

## 修复后的预期链路

`用户上传双角色图 -> 建立 duo_v1 Subject Manifest -> 用户修改外观 -> 生成 duo_v2 并使旧条件失效 -> 为狼男/猫娘/双人构图分别建立 canonical reference -> 每个片段携带同一 duo_vN -> 逐片段角色 QA -> 仅合格素材进入 assembly -> 全片一致性 QA -> 最终交付`

## 数据安全说明

本报告只记录项目 ID、trace ID、时间、工具类型、角色状态和必要的内部资产角色。未提交用户图像、视频、音频、原始 Langfuse payload、临时签名 URL、认证信息或大体积媒体文件。
