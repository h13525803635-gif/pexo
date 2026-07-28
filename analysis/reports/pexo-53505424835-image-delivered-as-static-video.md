# Pexo Project 53505424835: Image Request Delivered as a 1-Second Static Video

## 中文执行摘要

**结论已确认：这是交付类型路由错误，不是图片生成失败。** 用户明确要求一张 16:9 教育图片；Agent 错误进入以 MP4 为最终产物的 `motion-skill` 流程，并把 composition 显式设为 1 秒。过程中，系统已经成功生成且质检通过正确的 PNG，但没有交付该图片，反而删除全部动画后继续调用 `submit_render` 和 `show_final_video`，因此最终得到一段 1 秒静止视频。

建议在三个层面修复：在意图识别阶段锁定 `deliverable_type=image`；为 HTML 制图增加以 `render_frame` 和 PNG 交付为终点的静态分支；在最终交付前强制校验用户请求类型与文件 MIME/扩展名，图片请求遇到 MP4 时直接阻断。

## Scope and data completeness

- Project ID: `53505424835`
- Langfuse trace: `a86c59eee99793c138985816a2407171`
- Trace start: `2026-07-28T09:01:24.541Z`
- Discovered traces: 1
- Successfully fetched traces: 1
- Failed fetches: 0
- Observations inspected: 180
- Artifact inspection: not required for the root-cause conclusion; tool results and authored composition metadata directly establish the output type and duration

The raw Langfuse export is intentionally excluded from Git because it contains temporary signed asset URLs and potentially sensitive execution context.

## User-visible symptom

The user explicitly requested a simple 16:9 educational **image** explaining Transformer attention. The delivered artifact was instead `Transformer_Attention_Explainer.mp4`: a one-second video containing a static frame.

## Root cause

**Confirmed:** the agent selected an HTML-to-video workflow for an image-only request, successfully created and validated the requested PNG, but then ignored that valid image artifact and continued through the video-only `submit_render -> show_final_video` delivery path. The authored composition explicitly set its duration to one second, and the agent removed all animation before rendering, so the resulting MP4 was necessarily a one-second static video.

## Causal chain

```text
explicit image request
  -> motion-skill selected without locking the requested deliverable type
  -> HTML composition authored with data-duration="1"
  -> preview at t=0 exposed animation-start invisibility
  -> all entrance animation removed to make the image visible at t=0
  -> correct PNG rendered and quality-checked successfully
  -> PNG not delivered
  -> submit_render produced MP4
  -> show_final_video delivered the one-second static video
```

## Chronological evidence

| Time (UTC) | Observation | Evidence | Meaning |
|---|---|---|---|
| 09:01:24 | Trace input | User asked for a "16:9 educational image" | Requested deliverable type was unambiguously an image. |
| 09:01:29 | `read_file`, obs `a1df2155161744d2` | Agent loaded `motion-skill`, described as an HTML-to-video composition skill whose workflow ends in MP4 delivery | Earliest faulty routing decision. |
| 09:04:44 | `write_file`, obs `e4e656c77aeb0b56` | Root and scene both used `data-duration="1"` | The one-second duration was explicitly authored, not an accidental truncation by the renderer. |
| 09:04:51 | `render_frame`, obs `8867d973e3b2f2fa` | Generated `attn_preview_frame.png` at `time: 0` | The workflow already had a native PNG output mechanism. |
| 09:04:59 | `analyze_file_content`, obs `bda2a58e67df768f` | QC reported most intended content missing at the animation start state | This exposed a frame-timing problem, not a failure to generate an image. |
| 09:05:23 | `edit_file`, obs `6776938b39c21d29` | Agent replaced entrance animations with a static timeline and stated "static frame - no entrance animation" | This guaranteed that the subsequent video would contain no motion. |
| 09:05:26 | `render_frame`, obs `1321124afb3e38be` | Successfully generated `attn_frame_v2.png` | A valid candidate image artifact existed before video rendering. |
| 09:05:33 | `analyze_file_content`, obs `b8ecec6a1f06498c` | QC confirmed the title, steps, heatmap, observations, formula, colors, and legibility | The requested image was complete and ready to deliver. |
| 09:05:54 | `submit_render`, obs `43484a4f899f4864` | Submitted the HTML composition as `Transformer Attention Explainer` at 30 fps | The workflow unnecessarily converted the accepted image into video. |
| 09:06:16 | `show_final_video`, obs `b7e0d0a85f47e790` | Delivered `Transformer_Attention_Explainer.mp4` | Final delivery contradicted the user's requested artifact type. |

## Trigger, propagation, and detection gap

### Trigger

The agent interpreted "educational image" as permission to use the motion composition skill without first persisting `deliverable_type=image`.

### Propagation

The selected skill's normal completion path expects `submit_render` followed by `show_final_video`. Once this path was selected, the agent treated the successfully rendered PNG as only a preview/QC frame rather than the final deliverable.

### Detection gap

There was no final contract check comparing the requested MIME/type with the artifact being delivered. Neither lint nor visual QC checks output-type correctness. The task list also used ambiguous wording such as "render the final image" while the actual final tool remained video-only.

## Ruled-out alternatives

- **Image generation failure:** ruled out. `attn_frame_v2.png` was generated successfully and passed detailed visual QC.
- **Renderer unexpectedly truncated a longer video:** ruled out. Both the composition root and its scene explicitly declared a one-second duration.
- **Animation failed during export:** ruled out as the primary cause. The agent intentionally removed entrance animation and converted the composition to a static frame before export.
- **Wrong artifact caused by a stale revision:** not supported. The final MP4 was produced directly from the corrected static HTML after PNG validation.

## Corrective actions

### 1. Lock deliverable type during intake

- **Owner:** intent router / task planner
- **Trigger:** explicit terms such as `image`, `picture`, `poster`, `infographic`, `still`, `PNG`, `JPG`, or `WebP`, without a request for animation or video
- **Behavior:** persist `deliverable_type=image` and route to an image-capable workflow
- **Guardrail:** downstream skills may change the authoring method, but not the deliverable type without explicit user approval

### 2. Add a static-output branch to HTML composition

- **Owner:** `motion-skill` or its successor HTML composition skill
- **Trigger:** `deliverable_type=image`
- **Behavior:** author and lint the HTML, call `render_frame` at the intended final state, perform image QC, then deliver the resulting PNG
- **Guardrail:** block `submit_render`, `query_render`, and `show_final_video` for image-only tasks

### 3. Enforce delivery contract validation

- **Owner:** final delivery layer
- **Trigger:** immediately before any `show_final_*` call
- **Behavior:** compare requested deliverable type with the artifact extension and MIME type
- **Guardrail:** reject `.mp4`, `.mov`, or `.webm` when `deliverable_type=image`; reject image files when `deliverable_type=video`

### 4. Treat successful image QC as a terminal state

- **Owner:** agent workflow / task state machine
- **Trigger:** an image artifact exists and passes all requested visual assertions
- **Behavior:** mark the image task complete and deliver that exact artifact
- **Guardrail:** any subsequent format conversion must be justified by the user request or an explicit compatibility requirement

## Regression checks

1. Given "make a 16:9 educational image", the final tool is an image-delivery tool and the delivered MIME type begins with `image/`.
2. An HTML-authored infographic may call `render_frame`, but must not call `submit_render`.
3. If `render_frame` returns a PNG that passes QC, that same asset ID is delivered.
4. A mismatch between `deliverable_type=image` and an MP4 artifact fails before delivery.
5. Adding words such as "colorful", "steps", or "attention grid" must not cause the router to infer motion.
6. A request that explicitly asks to animate the infographic routes to video and is unaffected by the image-only guardrail.

## Conclusion

This was a routing and delivery-contract failure, not a content-generation failure. The earliest effective fix is to lock the requested artifact type at intake; the strongest backstop is a final MIME/type assertion that makes it impossible to deliver a video for an explicit image request.
