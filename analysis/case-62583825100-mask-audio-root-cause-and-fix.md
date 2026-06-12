# Case 62583825100: Mask Overlay Timing and Silent Final Video

## User Prompt

> Generate a skincare product video using 01_ecommerce/02_skincare_serum.png. Starting at 00:08, add a 30% opacity black mask over the main video. Then, on the right side, slide in user review bubble cards from bottom to top every 1 second: "Skin feels so smooth! ⭐⭐⭐⭐⭐", "Buying my 3rd bottle", "Highly recommended". Arrange them in a staggered waterfall layout, keep them on screen for 5 seconds, and then slide them all out to the right.

## Symptoms

- The final video did not reliably follow the requested mask/overlay timing.
- The delivered video had no sound.

## Evidence From Trace

### 1. Base video was generated as silent

The base video generation call used Kling reference-to-video with `sound: "off"`:

```json
{
  "name": "skincare_serum_base",
  "provider": "kling",
  "model": "kling-v3-omni",
  "mode": "reference2video",
  "provider_param": {
    "kling_reference2video": {
      "duration": "15",
      "sound": "off"
    }
  }
}
```

The generated base asset was:

- `asset://a_abvLj5f`
- `/projects/62583825100/workspace/assets/skincare_serum_base_20260611T023405_624b4b14.mp4`

`probe_media` showed only one video stream and no audio stream.

### 2. Final HTML muted the base video and did not add any audio

The composition embedded the base video with `muted`:

```html
<video
  id="main-video"
  src="https://pexo-assets.../skincare_serum_base_20260611T023405_624b4b14.mp4"
  muted
  playsinline
></video>
```

There was no `<audio>` tag in the composition, and the trace does not show a BGM, VO, ambience, or SFX generation step.

The final delivered asset:

- render job: `4ivvw93m`
- final asset: `asset://a_8m3MKuM`

`probe_media` for the final asset again showed only a video stream and no audio stream.

### 3. The first successful render had timing warnings for mask and cards

The first successful render was submitted before the overlay timing warnings were fixed. HyperFrames returned warnings:

- `timed_element_missing_clip_class` for `<div id="mask">`
- `timed_element_missing_clip_class` for `<div id="cards-overlay">`
- `root_composition_missing_data_start`

The warning explained that timed elements without `class="clip"` could be visible for the entire composition instead of only during their scheduled time range.

The agent later updated the HTML to add `class="clip"` and root timing, but the second render job:

- `6bq9ray6`

remained pending. The delivered output was still the earlier successful render `4ivvw93m`, which was created from the warning-bearing HTML version.

## Root Cause

### Mask / overlay issue

The overlay was implemented in HTML, but the first completed render was produced from an HTML version with incomplete HyperFrames timing metadata. The missing `class="clip"` on timed overlay elements meant the mask and review cards were not guaranteed to obey `data-start="8"` / `data-duration="7"`.

Although the HTML was later corrected, the corrected version was not the one ultimately delivered.

### Silent final video

The final video was silent because the entire pipeline treated it as silent:

1. Base video generation used `sound: "off"`.
2. The composition embedded the video with `muted`.
3. No `<audio>` track was added.
4. No BGM, VO, ambience, or SFX asset was produced.
5. Final `probe_media` confirmed the delivered MP4 had no audio stream.

This was not a muxing failure at the final step. The audio layer was never created or attached.

## Fix For This Project

### 1. Re-render from the corrected HTML only

The overlay elements must include `class="clip"` and valid timing metadata:

```html
<div
  class="mask-layer clip"
  id="mask"
  data-start="8"
  data-duration="7"
  data-track-index="1"
></div>

<div
  id="cards-overlay"
  class="clip"
  data-start="8"
  data-duration="7"
  data-track-index="2"
></div>
```

The root wrapper should also include timeline ownership:

```html
data-start="0"
data-duration="15"
```

Do not deliver job `4ivvw93m`; it was rendered before the timing fix.

### 2. Add audio before final render

Because the user did not request silence, add at least one audio layer. For this skincare product video, the safest default is a 15-second premium beauty / soft luxury BGM track.

The final HTML should include a signed URL audio element, for example:

```html
<audio
  src="https://..."
  data-start="0"
  data-duration="15"
  data-track-index="3"
></audio>
```

If using generated ambience/SFX instead of music, the same rule applies: the final composition must contain an audio asset.

### 3. Final validation before delivery

Before delivery:

- Render from the latest corrected HTML.
- Confirm the render job belongs to the latest HTML version.
- Run `probe_media` on the final MP4.
- Block delivery unless the final MP4 contains an audio stream.
- Spot-check frames around:
  - `7.9s`: mask should not yet be visible.
  - `8.0s` to `8.6s`: mask fades to 30% opacity.
  - `8.3s`, `9.3s`, `10.3s`: review cards enter one by one.
  - `13.5s` onward: cards slide out to the right.

## System-Level Prevention

### 1. Treat overlay timing warnings as hard blockers

Do not allow final delivery when HyperFrames returns warnings such as:

- `timed_element_missing_clip_class`
- `root_composition_missing_data_start`
- timed overlay elements with `data-start` but no `class="clip"`

These warnings directly affect user-visible timing behavior and should block delivery.

### 2. Enforce latest-render ownership

If HTML is modified after a render job is submitted, that render job is stale. A stale successful render must not be delivered.

Final asset selection should require:

- latest HTML revision
- lint / render from that same revision
- successful final media probe

### 3. Add an audio intent gate

Unless the user explicitly requests silence:

- `audio_intent` should not be `explicit_silent`
- the final composition must include at least one audio source
- `probe_media(final)` must show an audio stream

If `video_generate` uses `sound: "off"`, the assembly/composition stage must add BGM, ambience, VO, or SFX before final delivery.

### 4. Final QA checklist

Add hard checks for this class of request:

- Mask starts exactly at requested timestamp.
- Mask opacity reaches requested target.
- Overlay cards enter at requested cadence.
- Overlay cards hold for requested duration.
- Overlay cards exit in requested direction.
- Final video contains an audio stream unless explicitly silent.
- Delivered asset is from the newest successful render.

## Summary

The project failed because the delivered render came from an HTML version with unresolved timed-overlay warnings, and because the audio layer was never created or attached. The immediate fix is to re-render from the corrected `clip`-based overlay HTML and add a BGM/audio track. The long-term fix is to make timed-overlay warnings and missing audio streams hard blockers for final delivery.
