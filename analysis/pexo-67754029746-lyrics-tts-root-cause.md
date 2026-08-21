# 项目 67754029746：歌词被 TTS 念出来的根因与解决方案

## 结论

该项目原本要求生成 45 秒、约 100 BPM 的电影感流行男女对唱，但最终音频使用了普通 TTS 来处理歌词。原因是规划阶段把“唱歌词”错误建模成了 `audio_list_speech`，随后又明确要求生成 TTS 音频；背景音乐则被单独强制生成为纯伴奏。因此最终结果在结构上就是“纯伴奏 + 朗读歌词”。

根因置信度：已确认。

## 证据

数据来源：Metabase 数据库 3（`pg-server`），时区为 `Asia/Shanghai`。

- 项目 `67754029746` 创建于 `2026-08-18 15:48:48`。用户请求明确包含“两位歌手”“pop duet”“100 BPM”“按标记演唱每一行”和“音画同步”。
- `15:51:00` 生成的 Script Handoff 将每一段歌词声明为 `planned_source: audio_list_speech`，并设置 `requires_tts_asset_before_video: true`。
- `15:51:44` 的音乐生成调用使用了 `mode: text2music`、`force_instrumental: true`，并排除了 `vocals`、`lyrics` 和 `singing`。
- `15:53:02-15:53:29` 共执行 7 次 `audio_produce`，均使用 `mode: text2speech`、ElevenLabs `eleven_v3`，并把完整歌词作为 `text` 输入。
- `15:54:21` 的视频生成把这些 TTS MP3 作为 `audio_list`，并要求视频模型根据它们做口型同步。
- `16:17:15` 的质检只检查了人物、灯光、构图和字幕画面，没有检查“唱歌还是说话”、声部归属、对唱覆盖或 BPM 对齐。
- 项目共保存 316 条消息、1 个线程、29 个资产和 8 个音频资产。项目状态为 `ACTIVE`，执行状态为 `INTERRUPTED`。

## 因果链

`唱歌/对唱意图` -> `歌词被建模为普通语音` -> `audio_list_speech` -> `text2speech` 处理所有歌词 -> `纯伴奏 BGM` -> `TTS MP3 用于口型同步` -> `只做视觉质检` -> `最终交付朗读歌词`。

## 根因

最早的责任点是 Script Handoff 的音频契约。现有契约没有一等的 `sung_vocals`（歌声）路由，于是把“画面中演唱”归入了“画面中说话”。Generation Skill 后续是按错误契约正常执行的，因此调用了 TTS。

代理搜索了“pop singing voice”，并加入 `[excited]` 等情绪标记，但实际调用仍然是：

```json
{
  "mode": "text2speech",
  "model": "eleven_v3"
}
```

声音风格和情绪提示只能影响朗读方式，不能把普通文本转语音模型变成具有旋律、音高和节拍的歌唱模型。

## 伴随失败

1. 音频类型模型将对白/旁白和歌唱混在了一起。
2. 路由器没有禁止“歌曲意图 + 歌词文本 + `text2speech`”这一冲突组合。
3. 需要人声时仍允许 `force_instrumental: true`。
4. `speaker: Both` 没有校验是否真的存在两个人声 stem 或经过验证的男女对唱母带。
5. TTS 实测时长超出预算后，45 秒目标被放宽到约 57 秒。
6. 没有执行音频质检，只执行了画面质检。

## 正确生产链路

1. 将请求分类为 `performance_type: sung_vocals`、`vocal_structure: duet`，同时锁定 BPM 和总时长。
2. 使用支持歌词/歌声的音乐生成或歌声合成能力。若需要按歌手做口型同步，优先同时生成 instrumental、female 和 male stems。
3. 女声画面使用 female stem，男声画面使用 male stem，Together 画面使用两个 stem 或经过验证的 duet master。
4. 视频生成直接使用歌声音频，不得用 `text2speech` 重新制作歌词。
5. 最终只使用一个权威人声时钟，将伴奏和人声 stem 混音一次。
6. 若无法在 45 秒目标内完整覆盖歌词，应重新缩短歌词或重新编排，不得静默放宽时长。

## 必须实施的产品修复

### Script Skill

- 增加 `audio_intent.performance_type: sung_vocals` 和 `planned_source: generated_song_vocals`。
- `audio_list_speech` 与 `post_vo` 仅允许用于对白、旁白和 VO。
- 画面中存在演唱时，必须生成 `music_lipsync_feasibility_declaration`。

### Router / Generation Preflight

出现以下任一情况时必须阻断：

- 意图包含 `song`、`sing`、`lyrics`、`duet` 或 `chorus`，但歌词被传入 `audio_produce(mode=text2speech)`；
- 需要人声却设置了 `force_instrumental=true`，且没有独立的歌声音频；
- `speaker=Both`，但没有两个声部 stem 或经过验证的对唱母带；
- 实际时长超出用户声明的容差。

如果当前提供商不支持可控歌唱，应明确报告能力限制，不能静默降级成朗读 TTS。

### QA / Assembly

每个歌声音频和最终视频都必须验证：

- 是歌唱而不是朗读；
- 女声、男声和双人声部归属正确；
- 歌词完整，没有缺失、重复或叠加的人声层；
- BPM 和时长符合要求；
- 最终混音中人声只由一个权威音轨负责。

所有带声音视频必须进行 combined 音画检查，不能只检查静帧画面。

## 回归断言

- 包含 `song`、`sing`、`lyrics`、`duet` 或 `chorus` 的请求，不得产生歌词文本进入 `text2speech` 的调用。
- 歌曲请求不能在没有独立歌声母带/stems 的情况下以 `force_instrumental=true` 完成。
- Together 段不能只通过单个歌手资产验收。
- 45 秒请求不得超出声明的时长容差。
- 最终 QA 记录必须包含 `sung_vs_spoken`、声部归属、时序和最终混音所有权结论。

## 调查范围

- 数据库：`3`（`pg-server`）
- 项目：`67754029746`
- 发现/检查线程数：`1 / 1`
- 检查的持久化消息：`316`
- 资产：共 `29` 个，其中音频 `8` 个
- 报告时区：`Asia/Shanghai`
- 在本次运行中未发现独立的 trace/observation 业务表；结论基于项目消息、工具参数/结果、项目状态和资产元数据。
