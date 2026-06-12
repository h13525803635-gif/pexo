# Case 81821619614：跑车成片叠加层交付问题与根因

## 一、用户需求

用户要求基于 `01_ecommerce/05_sports_car.png` 生成一条跑车赛道视频，并额外叠加两类后期元素：

1. 左下角动态速度表，指针从 0 加速到 280 km/h。
2. 00:08 时从右侧滑入 70% 透明黑色参数卡，占画面右 1/3，文字为：
   `V8 4.0T Twin-Turbo / 650 HP / 0-100km/h in 3.2s / Carbon Ceramic Brakes`。

这不是单纯图生视频任务，核心交付依赖「底层视频 + HyperFrames 后期叠加渲染」。

## 二、实际链路

- `2026-06-11T02:35:37Z`：调用 `video_generate`，用上传跑车图生成 15s、16:9、无声跑车视频，返回 `asset://a_k1ySFGX`。
- `2026-06-11T02:43:11Z`：获取该视频 signed URL，准备嵌入 HTML。
- `2026-06-11T02:44:44Z`：写入 `/projects/81821619614/workspace/sports_car_overlay.html`，在 HTML 中实现速度表、参数卡、GSAP 时间线。
- 前两次 `submit_render` 被 lint 阻断：
  - `<video id="bg-video">` 缺少 `data-start`。
  - 给 `<video>` 加 timing 后，又嵌套在 timed wrapper 内，lint 提示会导致视频 frozen。
- 第三次 `submit_render` 通过但带 4 条 warnings，包括：root 缺 `data-start`、timed overlay 缺 `class="clip"`、spec-card CSS transform 与 GSAP transform 冲突。
- Agent 后续修复 warnings 后再次提交渲染，但多个 job 长时间 `pending`；其中 `cxv7orse` 后来返回 `hyperframes.job_not_found`。
- 最终在 `2026-06-11T04:47:12Z` 重新提交 job `7yeggwuc`，`2026-06-11T04:54:26Z` 完成，返回最终叠加成片 `asset://a_muhDt4C`，并通过 `show_final_video` 交付 `/projects/81821619614/workspace/assets/sports_car_racing_overlay_final.mp4`。

## 三、成片问题 / 交付风险

### 1. 最终成片没有被真正验收

最终 render 完成后，Agent 调用了 `probe_media`，但 probe 的对象是原始底层视频 `asset://a_k1ySFGX`，不是最终叠加成片 `asset://a_muhDt4C`。

因此它只验证了：

- 原始跑车视频是 1280x720；
- 时长约 15.04s；
- 只有 video stream；

但没有验证：

- 最终成片是否包含速度表；
- 参数卡是否在 00:08 出现；
- 右侧 1/3、70% 黑底、四行 specs 是否真实可见；
- 最终叠加渲染的分辨率、时长、是否黑屏/冻结；
- 最终文件是否存在音频或是否按预期无声。

这会导致 Agent 在最后回复中声称“速度表、参数卡都已完成”，但该说法没有最终产物级证据支撑。

### 2. HyperFrames render 队列 / job 状态处理不稳

本 case 至少经历了 6 次 `submit_render`，其中：

- 前 2 次是 lint 明确失败；
- 第 3 次虽然成功入队，但带 warnings；
- 后续 job 长时间 pending；
- `cxv7orse` 在长时间轮询后变成 `job_not_found`；
- Agent 最后靠重新提交才拿到 `a_muhDt4C`。

问题不是底层图生视频失败，而是后期合成渲染链路没有稳定的 timeout / retry / stale-job 处理策略。

### 3. 初版 HTML 违反了多条 HyperFrames 组合规则

初版 HTML 的问题包括：

- `<video>` 有 `src` 但缺 `data-start`，导致 HyperFrames 无法托管播放。
- 修复后又出现 video 嵌套 timed element，lint 明确提示 render 中视频可能冻结。
- timed overlay 缺 `class="clip"`，可能导致元素不按计划隐藏/显示。
- `#spec-card` 同时使用 CSS `transform: translateX(...)` 和 GSAP `x`，容易出现 transform 覆盖问题。
- root composition 起初缺 `data-start="0"`。

这些问题都被工具提示发现并大多修掉，但说明 Agent 的 HTML 生成没有先按 HyperFrames 必要骨架写，导致多次消耗渲染/轮询时间。

## 四、根因

### 根因 A：验收对象选错

最终交付资产是 `a_muhDt4C`，但 Agent probe 的是 `a_k1ySFGX`。这属于 QA 指针错误：把中间素材当成最终成片验收。

### 根因 B：缺少“最终 render asset 必须抽帧检查”的硬门槛

这个任务的核心是叠加层，而叠加层只有最终 render 才能确认。仅 `show_final_video` 不足以证明速度表和 spec card 出现在正确时间点。

### 根因 C：HyperFrames lint warnings 没有强制阻断

第三次 render 在 warnings 未处理时已经入队，后面又继续修 HTML 再提交，造成多个并行/陈旧 job 混杂。应当把影响 timing/clip/transform/root playback 的 warnings 视为必须修复项。

### 根因 D：render job 生命周期处理不足

长时间 pending、job_not_found 后没有统一策略：例如超过预计等待时间 N 倍即取消/弃用 job、重新 submit，并只追踪最新 job。结果 trace 中出现多个 job 交错，增加误交付风险。

## 五、解决方案

### 立即修复

1. 对最终 `asset://a_muhDt4C` 执行 `probe_media`，确认最终成片时长、分辨率、stream 信息，而不是 probe 原始 `a_k1ySFGX`。
2. 对最终成片抽帧验收：建议至少检查 `t=0.5s`、`t=7.5s`、`t=8.5s`、`t=10s`、`t=14.5s`。
3. 验收点：
   - 0.5s：速度表可见，读数开始增长。
   - 7.5s：速度表接近 280，参数卡尚未完全出现。
   - 8.5s：右侧黑色参数卡滑入，第一行 spec 可见。
   - 10s：四行 spec 均可见，卡片占右 1/3。
   - 14.5s：整体淡出。
4. 若抽帧失败或叠加层不可见，重渲染前先修 HTML，再重新 lint / render。

### 系统级修复

1. `show_final_video` 前新增硬性检查：若最终文件来自 HyperFrames，必须 probe `query_render.asset_id` 对应资产。
2. 增加最终资产帧级 QA：基于任务中声明的关键时间点自动 render/extract frame，并要求 Agent 明确检查 overlay 是否存在。
3. 将以下 HyperFrames warnings 升级为 blocking：
   - `root_composition_missing_data_start`
   - `timed_element_missing_clip_class`
   - `gsap_css_transform_conflict`
   - 所有 media timing / nested timed media 相关 warning/error
4. render job 管理只保留一个 active job：新 HTML 版本提交后，旧 job 标记 stale；最终只允许交付最新 HTML hash 对应的 job。
5. pending 超时策略：超过 `estimated_wait_seconds * 3` 或超过固定上限（如 10 分钟）仍 pending，应查询队列健康或重提，不要无限 sleep。
6. 用户回复中的“已完成元素”必须来自最终 QA 结果，而不是来自 HTML 设计意图。

## 六、结论

本 case 的底层跑车视频生成成功，最终也拿到了叠加成片资产 `asset://a_muhDt4C`。但交付链路存在明显 QA 缺陷：最终验收 probe 错了资产，只检查了原始跑车视频，没有验证真正交付的叠加成片。根因是 HyperFrames 任务缺少最终资产级 probe + 关键帧视觉验收硬门槛，同时 render job 生命周期管理不稳定，导致多 job pending / job_not_found 后靠重提才完成。
