# Pexo 竖屏视频方向/构图异常问题汇总

**报告日期：** 2026-09-18（Asia/Shanghai）  
**范围：** Pexo 项目 `36676628347`、`67199287457`   
**数据源：** Metabase 数据库 `3`（`pg-server`）；项目消息、工具参数、工具结果、资产元数据；并对可获取的生成素材做了 `ffprobe` 与关键帧检查。  
**结论级别：** 两个项目的源素材方向异常和拼接透传链路为 confirmed；“模型对提示语的空间语义产生错误解释”是 strongly supported 的生成层根因；更上游的具体模型内部实现原因仍是 hypothesis。

## 结论先行

这不是一个单纯的“Pexo 导出时把视频旋转了”的问题，而是同一类质量事故在两个项目中的不同表现：

1. `36676628347`：Seedance 生成的源视频首段约 0–4 秒出现倒置/方向异常；源文件本身已经异常，后续 9:16 composition 仅用 `object-fit: cover` 铺满画布，没有做旋转修正。
2. `67199287457`：MiniMax H3 生成的素材出现两种异常：
   - `shot1`（0–8 秒）将要求的“上/中/下信息层级”生成成三块横向分栏，属于错误的构图解释；
   - `shot2`（约 8–16 秒，用户截图约 13 秒处）整段画面横置 90°，人物和车内空间侧躺进入竖屏画布。
   - `shot2` 文件元数据仍是 `768×1344`、无 `rotate` 标签，说明“容器是竖屏”不能证明“内容方向正确”。
3. 两个项目的最终拼接都没有制造旋转：`67199287457` 的最终成片是 concat 四个片段；`36676628347` 的 composition 直接播放生成素材。根因应优先归到“生成输出未满足方向/构图要求 + 验收漏检”，而不是默认归因给 Pexo assembly。

## 用户可见问题

| 项目 | 时间/素材 | 用户可见症状 | 分类 |
|---|---|---|---|
| `36676628347` | 首镜头约 0–4 秒，`cold_open_main_v2` | 咖啡馆建立镜头出现方向倒置；画面内容先呈现异常方向，之后恢复正常 | 生成内容方向异常 |
| `67199287457` | `shot1`，0–8 秒 | 乘客、司机/导航、手机被组织为三段横向画面，上下叠放，和“同一竖屏镜头中按景别/深度组织”不一致 | 生成构图解释异常 |
| `67199287457` | `shot2`，约 8–16 秒；截图约 13 秒 | 整个车内镜头侧躺 90° 进入竖屏画布 | 生成内容方向异常 |

## 根因链

### 共同链路

`9:16/竖屏意图` → `生成请求只约束了 aspect_ratio 和自然语言构图` → `模型返回了容器竖屏但内容方向/空间组织不合格的素材` → `Pexo 只做尺寸/时长/可播放性等基础检查，未做逐镜头方向检查` → `assembly/composition 直接透传` → `用户看到横置、倒置或错误分栏画面`。

### 项目 `36676628347`

- **Intent：** 用户请求 9:16 竖屏、连续 live-action 短片；提示中包含“Vertical 9:16 framing throughout”。
- **Generation：** `cold_open_main_v2` 使用 Seedance 2.5，参数 `aspect_ratio: "9:16"`，输出探测为 `720×1280`。
- **Observed artifact：** 视觉检查记录了 0–4 秒开场方向异常（椅子呈倒置），随后镜头恢复。
- **Propagation：** composition 画布为 `1080×1920`，视频元素 `width:100%; height:100%; object-fit:cover`，没有 `rotate` 或方向校正逻辑。
- **Detection gap：** 对素材做了尺寸/时长 probe 和内容检查，但没有把“每个镜头是否正向”作为硬验收条件。
- **Root cause confidence：** **strongly supported**。源素材的方向问题在进入 composition 前已存在；assembly 是传播者，不是首发原因。

### 项目 `67199287457`

- **Intent：** 用户明确要求 9:16、不能横向构图或横向裁切，并要求画面按上下信息层级组织。
- **Generation：** `shot1`、`shot2` 均使用 MiniMax H3，参数 `aspect_ratio: "9:16"`；两者资产元数据均为 `768×1344`。
- **Observed artifact：** `shot2` 下载后的关键帧在 0.5、4.5、7.5 秒均保持侧躺；`ffprobe` 未发现 `rotate` 标签。该素材就是用户截图约 13 秒处的来源。
- **Prompt risk：** `shot1` 的提示同时使用 “TOP third / BOTTOM third / stacked / vertical corridor”等空间描述，模型把它落实为三块横向面板；`shot2` 虽要求 “vertical gap”，实际生成仍为侧躺画面。这表明 `aspect_ratio` 控制了输出画布尺寸，但不能保证内容的重力方向或镜头坐标系。
- **Propagation：** 最终成片 `night_driver_final_cut.mp4` 由四段素材直接 concat；最终 probe 为 `768×1344`、约 33.45 秒，未发现旋转或再构图操作。
- **Detection gap：** `shot2` 的验收重点是“手是否靠近手机、手机是否仍可见、人物是否一致、是否有霓虹光”，没有把“画面正向/车内垂直线是否竖直”列为否决条件；最终成片检查也没有拦截方向异常。
- **Root cause confidence：** **confirmed**（源素材异常并被透传）；“模型对空间提示语的错误解释”**strongly supported**。

## 关键证据

| 时间（Asia/Shanghai） | 项目/资产 | 证据 | 结论 |
|---|---|---|---|
| 2026-09-17 10:23–11:05 | `36676628347` / `cold_open_main_v2` | 用户提示要求 `9:16`；`video_generate` 使用 Seedance 2.5、`aspect_ratio: "9:16"`；`media_probe` 返回 `720×1280` | 参数层面是竖屏，不能证明内容方向正确 |
| 2026-09-17 11:09 | `36676628347` / `cold_open_main_v2` | `analyze_file_content` 记录 0:00–0:04 开场方向异常，之后恢复 | 方向异常在生成素材内部发生 |
| 2026-09-17 11:06 | `36676628347` / composition | 组合 HTML 使用 `1080×1920`，`video { width:100%; height:100%; object-fit:cover; }`，无 rotate | composition 只铺满/裁切，未负责方向修复 |
| 2026-09-17 19:00–19:03 | `67199287457` / `shot1_wontwake` | `video_generate` 使用 MiniMax H3、`aspect_ratio: "9:16"`；资产 metadata `768×1344`；视觉验收描述三段上下堆叠 | 首镜头是构图解释异常，不是导出尺寸异常 |
| 2026-09-17 19:06–19:10 | `67199287457` / `shot2_reach_withdraw` | 初次带视频作 image reference 的调用失败；随后无 reference 重试成功，使用 MiniMax H3、`aspect_ratio: "9:16"` | 重试解决了调用合法性，但没有保证方向质量 |
| 2026-09-17 19:10–19:11 | `67199287457` / `shot2_reach_withdraw` | `media_probe` 资产为 `768×1344`；视觉验收只确认手、手机、人物、霓虹和 live-action | 漏掉正向检查 |
| 2026-09-17 19:40 | `67199287457` / `night_driver_final_cut.mp4` | concat 输入为 `shot1 + shot2 + shot3_v4 + shot4`；最终 probe `768×1344`、约 `33.45s` | assembly 直接传播 shot2 异常 |
| 用户截图 | `67199287457` / 约 13 秒 | 司机、乘客和车顶整体侧躺 | 与 shot2 关键帧一致，定位到生成素材而非最终播放器偶发旋转 |

## 排除项与仍需保留的假设

### 已基本排除

- **最终 concat 统一旋转：** 不符合证据。`shot2` 原始文件在下载后已经侧躺；最终 concat 没有旋转步骤。
- **仅仅是播放器 EXIF/rotate 标签：** 不符合证据。`shot2` `ffprobe` 未返回 `rotate` 标签，画面像素本身已横置。
- **仅仅是最终画布尺寸错误：** 不符合证据。相关文件均是 portrait dimensions（`720×1280`、`768×1344`、`1080×1920`），但内容方向仍可能错误。

### 仍需验证

- MiniMax H3 和 Seedance 在何种提示词组合、参考图/参考视频状态下更容易出现方向漂移，需要建立带“重力方向/车内垂直线”标注的回归样本。
- 是否存在 provider 侧偶发的 latent rotation 或 camera-coordinate failure，当前项目数据不足以拆分模型内部机制。

## 修复建议（按最早责任层排序）

### 1. Generation acceptance：新增方向硬门槛

- **Owner：** generation-skill / media acceptance validator
- **Trigger：** 输出声明为 portrait（`height > width`），且用户意图包含竖屏、人物/建筑/车辆等有明确重力方向的内容。
- **Behavior：** 自动抽取首帧、镜头切点附近帧和末帧，做视觉方向检查；对于车窗、车顶、人物头部、文字 UI 等强方向线索逐帧判定。
- **Guardrail：** “编码尺寸为 9:16”不得作为通过条件；检测到 90°/180° 旋转、侧躺主体或明显错误坐标系时，候选标记为 rejected，不得进入 assembly。
- **Regression assertion：** 一个 `768×1344` 但像素内容侧躺的 fixture 必须失败；一个正常 9:16 竖屏 fixture 必须通过。

### 2. Prompt/router：把抽象的 top/bottom/vertical 改成可执行约束

- **Owner：** video-generation route / prompt compiler
- **Trigger：** prompt 同时出现 `top third`、`bottom third`、`stacked`、`vertical corridor` 或类似空间词。
- **Behavior：** 编译成明确的“相机保持 upright；人物头部朝画面上方；车辆车顶线水平；不要分屏/拼贴/多面板；所有内容来自同一连续镜头”的约束，并减少可能诱发分栏的 panel/section 语义。
- **Guardrail：** 对需要同一镜头的请求，不要只依赖自然语言的“上/中/下”，必须附带“single continuous frame / no split screen / no collage / no rotated camera”约束。
- **Regression assertion：** `shot1` 类提示不得产出三段横向面板；`shot2` 类提示不得产出侧躺车内画面。

### 3. Reference/retry：调用失败重试不能丢失关键质量约束

- **Owner：** generation-skill retry logic
- **Trigger：** reference2video 调用因参数类型失败后重试。
- **Behavior：** 修正非法输入（例如视频误传给 image-only 字段）时，保留原始方向约束、镜头坐标约束和验收标准；重试后必须重新做方向检查。
- **Guardrail：** “调用成功”不是“内容通过”；任何 retry 生成的候选都视为新 revision，不能继承旧候选的视觉验收结论。
- **Regression assertion：** retry 资产必须有新的 `media_probe` + visual orientation check，且没有 orientation evidence 时不能进入 final concat。

### 4. Assembly：增加入口断言，而非承担盲目修复

- **Owner：** assembly/media_process
- **Trigger：** concat、HTML composition 或最终交付前。
- **Behavior：** 对每个输入片段读取 width/height/rotate，并要求通过方向验收标记；在时间线上抽取每个片段边界帧做最终 sanity check。
- **Guardrail：** assembly 可以拒绝不合格素材，但不应无提示地自动旋转/裁切，因为自动修复可能改变构图；如要自动旋转，必须有明确的 provider orientation metadata 或用户授权的修复策略。
- **Regression assertion：** 含一段侧躺 portrait 视频的 concat 请求必须被阻断并指出具体片段和时间范围。

## 建议的最小回归集

1. **Portrait upright：** 正常人物/车辆竖屏，检查头部、车顶、文字方向。
2. **90° rotated portrait container：** 文件为 `768×1344`，像素内容侧躺；必须 reject。
3. **180° opening segment：** 仅首段倒置、后段正常；必须报告时间区间，不得被全片平均分数掩盖。
4. **Single-frame vertical stack：** 同一连续镜头中包含上/中/下信息；必须保持单一画面，不得生成三段 panel/collage。
5. **Reference retry：** 第一次调用失败、第二次调用成功；第二次仍需完整方向验收。
6. **Assembly propagation：** 输入片段之一失败时，concat/HTML delivery 必须阻断。

## 数据完整性说明

- 本报告使用数据库 `3`（`pg-server`），业务时区 `Asia/Shanghai`。
- 发现并检查了 2 个项目、至少 2 条相关 generation trace/tool-call 链；项目消息和资产查询均成功完成；部分大 JSON 查询在 MCP 展示层被截断，因此报告只引用了必要字段，未把截断内容当作完整证据。
- 关键素材的 `shot2` 已通过可获取源文件做 `ffprobe` 和多时间点关键帧检查；`36676628347` 的最新 Seedance 源文件下载链接已过期（HTTP 403），因此其“源素材已异常”的结论主要依赖已持久化的 `media_probe` 与视觉分析记录，置信度标为 strongly supported，而不是将无法复核的像素检查写成 confirmed。
- 所有报告中的 signed URL、签名参数、请求 token 和不必要的个人数据均已省略。

## 复现/审计 SQL（已脱敏）

```sql
-- 项目状态
SELECT project_id, project_name, status, execution_status,
       execution_progress, final_videos_count,
       created_at AT TIME ZONE 'Asia/Shanghai' AS created_at_cn,
       updated_at AT TIME ZONE 'Asia/Shanghai' AS updated_at_cn,
       completed_at AT TIME ZONE 'Asia/Shanghai' AS completed_at_cn
FROM public.projects
WHERE project_id IN ('36676628347', '67199287457')
ORDER BY created_at;

-- 资产尺寸、provider、模型与源文件身份
SELECT project_id, asset_id, asset_name, file_name, mime_type,
       asset_status::text, metadata
FROM public.project_assets
WHERE project_id IN ('36676628347', '67199287457')
  AND (mime_type LIKE 'video/%'
       OR asset_type ILIKE '%video%'
       OR file_name ILIKE '%.mp4')
ORDER BY project_id, created_at, id;

-- 生成、probe、视觉验收和 assembly 记录索引
SELECT project_id, id,
       created_at AT TIME ZONE 'Asia/Shanghai' AS created_at_cn,
       role, event_type, left(content::text, 9000) AS content_text
FROM public.project_messages
WHERE project_id IN ('36676628347', '67199287457')
  AND (content::text ILIKE '%video_generate%'
       OR content::text ILIKE '%media_probe%'
       OR content::text ILIKE '%analyze_file_content%'
       OR content::text ILIKE '%concat%'
       OR content::text ILIKE '%object-fit%')
ORDER BY project_id, created_at, id
LIMIT 160;
```
