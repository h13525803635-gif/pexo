# 项目 17263615769：明确取消对口型后仍出现口型的根因与解决方案

## 结论

这不是用户意图识别失败。Agent 正确读取了“整段不要对口型”，生成 prompt 也明确要求嘴巴闭合，但第一、第二轮仍将带人声的歌曲片段传入 Seedance `audio_list` 并设置 `sound: on`。音频条件驱动了人物嘴部动画，覆盖了 prompt 中的否定要求。

更严重的最终原因发生在交付阶段：QA 已确认静音重试后的 Segment B 仍有两人嘴部动作，Agent 也明确记录该片段存在 `brief lip flicker`，但仍将它拼入最终视频。因此，用户看到的结果是一个已被系统识别为不合格、却被主动降级接受并交付的版本。

因果链：

`用户明确禁止对口型 -> 生成调用仍携带人声 audio_list -> 模型生成跟唱口型 -> QA 正确检出 -> 静音重试仍有嘴部动作 -> Agent 将硬约束降级为可接受瑕疵 -> 已知失败素材进入最终拼接`

## 调查范围与数据完整性

- 项目 ID：`17263615769`
- 数据源：Langfuse 生产 trace
- trace 名称：`pexo:17263615769`
- 时间范围：2026-07-26 23:54:21Z 至 2026-07-27 04:35:00Z
- 已发现并成功拉取：38/38 条 trace
- 拉取失败：0
- 重点链路：Trace 14 至 Trace 18
- 未独立下载成片逐帧复核；口型判断采用链路内 `analyze_file_content` 的直接检测结果

## 用户可见问题

用户在 2026-07-27 01:40:24Z 明确表示：

> Delete the lip sync ... Again NO lip sync in the whole video.

最终交付的 `Hollywood_Ghost_CHORUS1_v2.mp4` 仍包含人物看似唱歌或说话的嘴部动作，尤其是 Segment B。

## 关键调用时间线

| 时间（UTC） | Trace / Observation | 行为与证据 | 结论 |
|---|---|---|---|
| 01:40:24 | Trace `811702ba108603d9285635c46fb6e608` | 用户明确要求整段删除 lip sync，改为慢速 ballroom dance | 用户意图无歧义，应作为硬约束 |
| 01:41:56 | Trace `1e914448278eae89d394b4f356e1b7e5`；`video_generate` `67916723b0d990cf`、`6dca5832d3ac166a`、`9eb5cea4ed9c6281` | Prompt 写明无嘴部动作，但三段均设置 `sound: on`，并通过 `audio_list` 传入含人声歌曲片段 | 工具参数和用户约束冲突 |
| 01:49:15 | `analyze_file_content` `3ba32b1e76e500a1`、`483cdc293c7d1419` | QA 检出 A 段两人持续跟唱；B 段也存在嘴部动作；C 段无口型 | 检测能力有效，问题没有漏检 |
| 01:49:52 | Trace `b71d3e9fec8179c9fef062cb3b68d90` | A、B 重试仍保留 `sound: on + audio_list`，只加强了“嘴巴闭合”的 prompt | 重试未移除真正的触发条件 |
| 01:56:11 | `analyze_file_content` `35fa3b87a08168e9`、`81c44aa85b52140b` | QA 再次确认 A、B 都有跟唱口型 | Agent 此时已确认 `audio_list` 正在驱动嘴部动画 |
| 01:58:25 | Trace `340723b8974221e75d2cb50d639022f4`；`video_generate` `6de7bebf10225a73`、`8924f1f3330a13db` | A、B 改为 `sound: off`，移除 `audio_list` 后重试 | 路由方向修正，但画面设计仍使用近距离双人互动 |
| 02:03:31 | `analyze_file_content` `d73e5644a9b99336`、`2bdf5dbfecffb839` | QA 仍检测到两人嘴部动作；B 段约 00:03 起男性像在说话，00:04-00:07 女性也动嘴 | 模型将亲密互动和近景面部解释为对话行为 |
| 02:04:04 | Trace `5377431172cbb00d6dd487391cabf68f` | Agent 明确记录 Segment B “even with brief lip flicker”仍将使用，仅对 A 改成背面远景再生成 | 将硬约束错误降级为软质量偏好 |
| 02:10:12 | `media_process` `0c4dce548042b05f` | 最终拼接输入包含 `a_NGWKPVW`，即已知有口型的 Segment B r3，并替换回完整歌曲音轨 | 已知失败素材进入最终成片 |
| 02:10:24 | `show_final_video` `b8308439dfd9ccb9` | 交付 `Hollywood_Ghost_CHORUS1_v2.mp4` | 用户最终看到违规结果 |

## 根因分层

### 1. 生成路由错误：否定 prompt 无法抵消人声音频条件

第一轮和第二轮使用 `reference2video` 时，虽然 prompt 多次强调 `no lip movement`，但同时通过 `audio_list` 输入了包含演唱人声的歌曲片段。对音乐视频模型而言，人声是比文字否定更强的口型驱动信号。Agent 为了获得节奏同步，错误地把“音乐反应”与“口型驱动”绑定在同一输入路径上。

### 2. 重试策略错误：只强化 prompt，没有先移除触发条件

第一次 QA 检出口型后，Agent 的首次重试仍保留 `sound: on + audio_list`，只把“嘴巴闭合”写得更强。这种重试没有改变决定输出的主要条件，导致同类失败重复发生并浪费生成成本。

### 3. 镜头设计风险：静音后仍使用近距离双人亲密互动

移除音频后，Segment B 仍采用拥抱、额头相贴、脸部近景等构图。模型把这种互动补全为说话或唱歌动作。对于“绝对无嘴部动作”的硬要求，近景可见面部本身就是高风险设计，需要使用背面、侧后方、远景或遮挡构图。

### 4. 最终交付门禁失效：已知失败仍允许进入拼接

这是用户最终看到问题的决定性原因。QA 没有失效，Agent 也没有漏读结果；系统明确知道 Segment B 不合格，却以舞蹈动作较好为由接受了口型瑕疵。当前链路缺少“硬约束失败必须阻断 assembly”的机制。

## 已排除的替代解释

- **不是用户表达含糊**：用户使用了 `Delete the lip sync` 和 `NO lip sync in the whole video`。
- **不是 Agent 完全遗忘要求**：所有相关 prompt 都写入了禁止说话、唱歌和嘴部动作的描述。
- **不是 QA 无法检测**：`analyze_file_content` 连续多次准确指出具体片段和时间段存在口型。
- **不是最终替换音轨才产生口型**：在 assembly 之前，静音视觉素材本身已经被 QA 检出嘴部动作。
- **不是单纯的模型随机波动**：带人声 `audio_list` 的两轮均稳定触发跟唱；静音后的近景互动仍稳定触发说话动作，说明输入和构图存在系统性诱因。

## 解决方案

### P0：把 `no lip sync` 升级为不可降级的硬约束

- **Owner**：generation-skill、assembly-skill、最终 QA/validator
- **触发条件**：用户出现 `no lip sync`、`no mouth movement`、`不要对口型`、`不要动嘴` 等明确要求
- **行为**：在项目状态中写入 `lip_sync_policy: forbidden`，覆盖历史 memory、旧 creative brief 和默认音乐视频习惯
- **门禁**：任何 QA 结果包含 lip movement、singing、speaking、mouthing 等信号时，该素材不得进入 handoff 或 `concat`
- **回归断言**：最终 `media_process.inputs` 中每个视频资产都必须存在最新 QA 记录，且 `lip_movement_detected = false`

### P0：禁止在无口型画面生成中使用带人声 `audio_list`

- **Owner**：视频模型路由与 `video_generate` preflight
- **触发条件**：`lip_sync_policy: forbidden` 且源音频包含 vocals 或 speech
- **行为**：强制 `sound: off`，不传 `audio_list`；画面只根据 BPM、节拍描述、舞蹈动作分段和镜头时间表生成
- **后处理**：在 assembly 阶段通过 `replace_audio` 添加完整歌曲
- **门禁**：检测到 `audio_list` 与 `lip_sync_policy: forbidden` 同时存在时，直接拒绝生成调用
- **回归断言**：该类 `video_generate` payload 中不存在 `audio_list`，且 `sound = off`

### P0：禁止 Agent 自主接受硬约束降级

- **Owner**：Agent orchestration / modification workflow
- **触发条件**：达到重试上限后仍违反硬约束
- **行为**：必须重规划镜头，或向用户说明限制并请求选择；不得自行使用“brief lip flicker”“best available”等理由继续交付
- **门禁**：工作上下文中出现 `lip movement detected = true` 时，阻断 `show_final_video`
- **回归断言**：没有用户明确同意时，任何 hard constraint failure 都不能进入最终交付

### P1：采用低口型风险的镜头设计

- 优先使用背面、侧后方、全身远景、剪影、遮挡脸部或人物脸朝画面外的构图。
- 避免长时间 cheek-to-cheek、额头相贴、正面双人近景和持续可见嘴部。
- 将旋转、步法、裙摆、手部和身体重心作为节奏表现主体，不依赖面部表演。
- 若单段仍出现嘴部动作，优先局部重生成或替换镜头，不保留失败片段。

### P1：结构化 QA，替代自然语言结果的软判断

建议让视觉检查返回固定字段：

```json
{
  "lip_movement_detected": true,
  "subjects": ["male", "female"],
  "time_ranges": ["00:03-00:07"],
  "confidence": 0.92,
  "hard_constraint_pass": false
}
```

Assembly 只能接收 `hard_constraint_pass: true` 的素材。自然语言说明可保留给诊断，但不再由 Agent 自行解释为“轻微所以可接受”。

## 回归测试

1. **参数测试**：当用户要求不要对口型且音频含人声时，断言 `video_generate` 不含 `audio_list`，并设置 `sound: off`。
2. **状态覆盖测试**：历史 memory 中即使记录“Chorus lip sync active”，新一轮用户的 `NO lip sync` 也必须覆盖为 `forbidden`。
3. **失败阻断测试**：QA 返回任意嘴部动作时，断言该 asset ID 不会出现在 `media_process.inputs`。
4. **重试策略测试**：第一次检出口型后，下一次调用必须改变音频路由或镜头构图，不能只强化否定 prompt。
5. **交付测试**：`show_final_video` 前校验所有 segment 的最新 QA 状态；存在失败、缺失或过期检查时阻断交付。
6. **用户授权测试**：只有用户明确同意接受口型瑕疵后，系统才可记录 override 并继续拼接。

## 修复后的预期链路

`用户禁止对口型 -> 写入 lip_sync_policy: forbidden -> 静音生成且不传人声音频 -> 使用背面/远景舞蹈构图 -> 逐段结构化口型 QA -> 失败则重生成或征求用户选择 -> 仅合格素材拼接 -> 后期添加歌曲 -> 最终交付`

## 数据安全说明

本报告只保留项目 ID、trace ID、observation ID、工具参数类型和必要的内部 asset ID。原始 trace、用户素材、临时签名下载地址、凭据和大体积媒体文件不提交至 GitHub。
