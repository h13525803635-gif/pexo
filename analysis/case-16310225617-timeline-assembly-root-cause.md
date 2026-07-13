# Project 16310225617: Timeline Assembly Root Cause Analysis

## Executive Summary

Project `16310225617` has two distinct classes of defects:

1. The production plan contained three sequences, but the first generation batch submitted only sequence 1 and sequence 3. Sequence 2 was generated later during recovery, together with several sequence 3 retries.
2. The final backend concat path is not the source of the reported `1:19` clip. Langfuse proves that its three final source assets are approximately 7.04, 8.04, and 7.04 seconds long. A `1:19` timeline item and a frozen image after ten seconds must therefore come from a downstream editor/timeline persistence path that is using incorrect clip metadata or a stale asset list.

The backend final concat did include sequence 1. If sequence 1 is absent in the user-visible editor or delivered timeline, the omission happened when the timeline was saved, reconstructed, or displayed, rather than in the recorded concat call.

## Evidence Source

The findings below come from the real Langfuse trace and observation records for:

```text
trace name: pexo:16310225617
production trace: 6f7b60dccd1fc2e60499dff079baaf22
trace start: 2026-07-13T07:36:38.489Z
```

No conclusion about asset ordering or duration is inferred from similar historical cases.

## Actual Production Path

### Initial generation batch

At `2026-07-13T07:37:22Z`, the agent submitted only these two generation calls:

```text
rome_egypt_seg1_march
rome_egypt_seg3_awe
```

There was no initial `seg2_reveal` call. This is a real planning-to-execution integrity failure: a three-sequence plan reached generation with one sequence missing.

The agent subsequently attempted:

```text
rome_egypt_seg3_awe_v2
rome_egypt_seg3_awe_k
rome_egypt_seg2_reveal_k
rome_egypt_style_ref
rome_egypt_seg2_reveal_k2
rome_egypt_seg3_awe_k2
```

This recovery changed providers and asset versions. Without a canonical sequence manifest, those retries can easily diverge from the editor timeline's asset references.

### Final selected assets

| Order | Sequence | Asset | Requested duration | Probed video duration | Frames / FPS |
|---|---|---|---:|---:|---|
| 1 | `rome_egypt_seg1_march` | `a_Nooqypp` | 7s | 7.041667s | 169 / 24 |
| 2 | `rome_egypt_seg2_reveal_k2` | `a_xR7ZktL` | 8s | 8.041667s | 193 / 24 |
| 3 | `rome_egypt_seg3_awe_k2` | `a_hmYRKAe` | 7s | 7.041667s | 169 / 24 |

The exact generation observation IDs are:

```text
seg1: f4c7a66ab76dfe70
seg2: 38d2b3aaaf143af9
seg3: 40447d5b8f4cee5b
```

The corresponding media probe observation IDs are:

```text
a_Nooqypp: 2f404ca9f2291549
a_xR7ZktL: bc680a165e29c8e7
a_hmYRKAe: 2deba928381591f5
```

### Final concat

Langfuse observation `c219aacc3490b5a5` records this exact call:

```json
{
  "inputs": [
    "asset://a_Nooqypp",
    "asset://a_xR7ZktL",
    "asset://a_hmYRKAe"
  ],
  "steps": [
    {"op": "concat", "transition": "cut"}
  ],
  "output_name": "Rome_Sees_Egypt_Pyramid_Final",
  "output_format": "mp4"
}
```

The concat succeeded and returned final asset `a_cmZChFc`. Based on the probed input streams, the expected final duration is approximately:

```text
7.041667 + 8.041667 + 7.041667 = 22.125001 seconds
```

The trace did not probe the final output after concat. That is an important QA gap, but the recorded concat input explicitly contains the first sequence.

## Root Causes

### 1. Generation batch was not checked against the sequence plan

The first batch omitted sequence 2 and proceeded without a completeness gate. Later retries repaired the missing material opportunistically instead of reconciling the complete sequence plan.

### 2. The expected ten-second duration never reached generation

The final sequence 2 generation request explicitly used `duration: "8"`. The provider returned 8.041667 seconds, which matches the request. This is an upstream parameter propagation error, not a provider under-generation error.

### 3. Editor timeline metadata diverged from media truth

No generated or probed source in this trace is 1 minute 19 seconds long. The final backend assembly is also only about 22.13 seconds. Therefore a user-visible `1:19` second clip cannot originate from the recorded generation or concat parameters.

The most likely downstream failure mode is:

```text
incorrect/stale timeline duration
  -> source video reaches EOF after about 8 seconds
  -> player keeps the last decoded frame visible
  -> remaining timeline appears static
```

Possible sources of the incorrect duration include a stale retry record, project duration copied into a clip, a time-unit conversion error, or reconstruction of the timeline from a different asset version.

### 4. Input assets were probed, but the final output was not

The agent probed all three selected inputs immediately before concat. It did not probe or visually inspect `a_cmZChFc` after concat. As a result, the delivery gate could not catch missing opening content, an incorrect final duration, or long repeated-frame regions.

## Required Fixes

### Canonical sequence manifest

Create one authoritative manifest before generation and carry it through generation, retries, assembly, persistence, and editor playback:

```json
[
  {"sequence": 1, "asset": "a_Nooqypp", "duration": 7.041667},
  {"sequence": 2, "asset": "a_xR7ZktL", "duration": 8.041667},
  {"sequence": 3, "asset": "a_hmYRKAe", "duration": 7.041667}
]
```

Retries must replace the asset for the same sequence ID. They must not append an ambiguous new timeline item.

### Pre-assembly completeness gate

Before assembly, require:

```text
planned sequence IDs == successfully generated sequence IDs
```

If any sequence is missing, assembly must stop. This would have caught the missing initial sequence 2 immediately.

### Duration contract

- Preserve the confirmed requested duration in every generation call.
- Use `media_probe.derived.video_playable_seconds` as the clip's media truth.
- Reject a timeline item when its duration differs from probed media duration by more than 0.1 seconds, unless an explicit loop, speed, trim, freeze, or hold behavior is configured.
- Never infer clip duration from generation wall time, project duration, UI label text, or an older asset version.

### Timeline persistence contract

Persist these fields together and validate them on load:

```text
sequence_id
asset_id
source_duration_seconds
timeline_in_seconds
timeline_out_seconds
explicit_end_behavior
asset_version
```

If a timeline slot is longer than the source media, the system must require an explicit end behavior. Silent last-frame freezing should not be the default.

### Final delivery QA

After concat:

1. Probe the final asset and compare its duration with the manifest sum.
2. Decode frames near 0s, every sequence boundary, and the last second.
3. Confirm the first decoded section belongs to sequence 1.
4. Detect long repeated-frame runs and block delivery unless a freeze is explicitly designed.
5. Compare final sequence count and order against the canonical manifest.

## Final Diagnosis

The initial generation execution was incomplete, and the requested ten-second duration was changed to eight seconds before it reached the video provider. The recorded backend concat later selected three valid short assets in the correct order and included the first sequence. The reported missing opening segment, `1:19` duration, and static tail therefore indicate a separate timeline persistence or editor reconstruction defect, compounded by the lack of a post-concat duration and visual QA gate.
