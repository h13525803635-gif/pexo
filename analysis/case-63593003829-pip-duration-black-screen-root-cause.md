# Case 63593003829: PiP Duration Miscalculation and Black Screen

## Summary

Project `63593003829` requested a sneaker product showcase video:

- Main visual: close-up rotating shot of the sneaker.
- PiP visual: bottom-right window showing the shoe being worn while running.
- Overlay: product spec card sliding in from the left with `280g`, `Cushion Tech`, and `Breathable Mesh`.

The user reported two symptoms:

1. The first delivery only appeared to provide one image.
2. After the user asked again, the delivered video had a long black section at the end.

The root cause is a timeline planning error plus insufficient delivery QA:

> The two generated 8-second clips were treated as if they could contribute roughly `8s + 8s` of sequential duration, but the requested design was picture-in-picture. The two clips should have played in parallel on the same timeline. The final HyperFrames render was made as an 18-second video, while the visible source videos only covered about 8 seconds unless explicitly looped, frozen, or replaced by a fallback end frame. Once the embedded videos ended, the composition exposed black background.

## Langfuse Evidence

Local export path:

- `analysis/langfuse-data/cases/63593003829/trace-2-fb3a282a.json`
- `analysis/langfuse-data/cases/63593003829/trace-index.json`

Trace index after refetch:

```json
{
  "fetched": [
    {
      "idx": 2,
      "trace_id": "fb3a282a12766747788078f5b392a883",
      "start_time": "2026-06-11T06:14:29.960Z",
      "file": "analysis/langfuse-data/cases/63593003829/trace-2-fb3a282a.json"
    }
  ],
  "failed": [
    {
      "idx": 1,
      "trace_id": "c94fc20eda733f5aecd5bcf7c0968220",
      "start_time": "2026-06-11T02:27:16.059Z",
      "error": "GET failed after 5 attempts: HTTP Error 500: Internal Server Error"
    }
  ]
}
```

The first trace, which likely corresponds to the initial delivery, could not be fetched from Langfuse because the trace detail endpoint returned HTTP 500. The second trace contains the user's follow-up message:

```text
Please deliver the video.
```

The recovered context in that trace contains the original video request:

```text
Generate a product showcase video using the sneaker image. Start with a close-up rotating shot of the shoe, then add a picture-in-picture window in the bottom-right showing the shoe being worn while running. Overlay a product spec card sliding in from the left with key features: weight 280g, cushion tech, breathable mesh.
```

## Timeline

All times are from `2026-06-11` UTC.

| Time | Event | Meaning |
|---|---|---|
| `06:14:29` | User says `Please deliver the video.` | User had to ask again after the first delivery. |
| `06:14:34` | `query_render(job_id="0x7otdim")` | Agent tried to recover an old render job. |
| `06:14:34` | `hyperframes.job_not_found` | The old render job was gone or invalid. |
| `06:14:42` | `submit_render(html_file="/projects/63593003829/workspace/sneaker_showcase.html", fps=30, output_name="sneaker_product_showcase")` | Agent resubmitted the HTML composition. |
| `06:22:52` | render status `done` | New render completed. |
| `06:23:01` | `probe_media(asset://a_D3UAoia)` | Agent checked container metadata only. |
| `06:23:15` | `show_final_video("/projects/63593003829/workspace/assets/sneaker_product_showcase.mp4")` | Final video was delivered. |

Final video metadata from `probe_media`:

```text
file: asset://a_D3UAoia
duration: 18.000000 seconds video, 18.048000 seconds audio
resolution: 1920x1080
fps: 30
codec: H.264 + AAC stereo
```

Generated media in local cache:

- `sneaker_rotating_closeup_20260611T023503_c04bb093.mp4`
- `sneaker_running_pip_20260611T023442_564acae9.mp4`
- `sneaker_bgm_20260611T023523_75c34f08.mp3`
- `sneaker_product_showcase.mp4`

## Why the First Delivery Looked Like One Image

The visible second trace shows the agent attempted to query a previous render job:

```text
query_render(job_id="0x7otdim") -> hyperframes.job_not_found
```

That means the first attempt did not leave a valid, recoverable render job for the follow-up turn. Because the failed first trace cannot be fetched due to Langfuse HTTP 500, the exact final action from the first turn is not visible. However, the available evidence supports this explanation:

- The user had to ask again with `Please deliver the video`.
- The next turn tried to recover an old render job.
- That job was missing.
- The agent then resubmitted the render from HTML.

So the first issue was a delivery-state failure: the system did not reliably deliver the completed video asset on the first turn and fell back into a state where only a preview/image-like artifact may have been visible to the user.

## Why the Second Video Had a Long Black Screen

The production logic generated two 8-second clips:

1. Main clip: rotating sneaker close-up.
2. PiP clip: running/worn shoe footage.

But those clips were not supposed to be sequential scenes. The user specifically asked for a picture-in-picture window, so the correct structure was:

```text
0s ---------------- 8s
main rotating shoe:  [================]
PiP running shoe:       [=============]
spec card overlay:         [==========]
```

The mistaken structure treated the clips more like duration inventory:

```text
main clip 8s + PiP clip 8s + intro/outro packaging ~= 18s
```

That is wrong for a PiP composition because the main and PiP videos overlap in time. The final render was 18 seconds, but the visual source clips only covered roughly the first 8 seconds unless the HTML explicitly extended them.

Expected safeguards were missing:

- No `loop` for the embedded video elements.
- No freeze-frame or poster frame after source video end.
- No fallback product image/background for the tail.
- No visual black-frame QA before delivery.

The agent did run `probe_media`, but this only confirmed the file was technically valid:

```text
18s, 1920x1080, 30fps, H.264 video + AAC audio
```

It did not verify whether the frames were visually non-black.

## Correct Design Interpretation

The correct interpretation of the request:

- The rotating close-up is the main background layer.
- The running/worn shoe footage is an overlaid PiP layer.
- The spec card is another overlay layer.
- These layers should coexist on one timeline.

The two 8-second clips should not be summed to determine final duration.

If the final video must be 18 seconds, then at least one of the following is required:

- Generate a true 18-second main visual.
- Loop the 8-second main/PiP clips cleanly.
- Freeze the last good frame after 8 seconds.
- Transition to a static product end card.
- Shorten the final render to the actual visible duration.

## Fixes

### 1. Timeline Planning Rule

Add a planning rule for PiP and split-screen compositions:

```text
When clips are layered simultaneously, final duration is the max of layer durations, not the sum.
```

For this case:

```text
max(main 8s, PiP 8s) = 8s usable visual duration
```

The final 18-second target was only valid if additional visual coverage was intentionally created.

### 2. HyperFrames Authoring Rule

For every embedded `<video>` in HyperFrames:

- Define what happens after `video.duration`.
- Use one of:
  - `loop`
  - freeze last frame
  - replace with poster/end card
  - fade to branded still frame
  - trim composition duration

Never allow the root background to become the only visible layer unless that background is intentionally designed.

### 3. Render Recovery Rule

If `query_render` returns:

```text
hyperframes.job_not_found
```

Then:

- Do not continue as if the prior render exists.
- Resubmit the render.
- Store the new job id.
- Deliver only the new completed asset.
- Avoid exposing stale preview assets as final output.

### 4. Visual QA Before Delivery

`ffprobe` or `probe_media` is not enough. Before `show_final_video`, run frame-level QA:

- Sample frames at `0s`, `25%`, `50%`, `75%`, and `duration - 1s`.
- Compute average luminance and non-black pixel ratio.
- Fail the render if multiple late frames are near black.
- Re-render or trim before delivery.

Suggested rule:

```text
If the final 20% of a video has consecutive frames with very low luminance and no intentional fade-to-black marker, block delivery.
```

### 5. Case-Specific Remediation

For project `63593003829`, the clean fix is:

1. Reopen the HTML composition.
2. Treat `sneaker_rotating_closeup` as the main layer.
3. Treat `sneaker_running_pip` as a concurrent PiP layer.
4. Either shorten the render to about 8-10 seconds or loop/freeze the visuals through 18 seconds.
5. Add a final product end card if 18 seconds is required.
6. Re-render.
7. Run visual black-frame QA.
8. Deliver the new mp4.

## Final Diagnosis

This was not a user asset problem.

It was caused by:

1. A lost or expired first render job, which made the first delivery appear incomplete.
2. A PiP duration-planning mistake, where two parallel 8-second videos were treated like sequential duration coverage.
3. Missing post-render visual QA, so a technically valid 18-second mp4 with a black tail passed delivery.
