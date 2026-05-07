# Case 14848151814: Root Cause and Fix Plan

## Scope

This note summarizes the confirmed root causes and concrete fixes for three issues observed in the generated video:

- Mixed visual ratio perception (`16:9` and `1:1` looking segments)
- On-screen text heavily overlapping with subtitle content
- Subtitle style appearing inconsistent over time

Data source: `analysis/langfuse-data/cases/14848151814/trace-1-07b98a04.json`.

## Problem 1: Mixed Ratio Perception (16:9 vs 1:1 Looking Parts)

### Root Cause

- Scene-level `video_generate` calls requested `aspect_ratio: "16:9"`, but multiple key scenes failed due to credit blocking.
- During assembly, failed scene videos were replaced by static scene images (`scene*.png`) while other scenes remained generated videos (`vid*.mp4`).
- Those static images were originally generated without strict ratio enforcement and were then scaled into a `1920x1080` timeline, creating an apparent ratio inconsistency.

### Fix

1. Enforce `16:9` on **both** `video_generate` and `image_generate` calls.
2. Add failure fallback chain for each scene:
   - retry same model once
   - retry fast model once
   - if still failed, mark scene as failed and stop final assembly
3. Add pre-assembly gate:
   - run `ffprobe` on every clip
   - reject assembly if any source is not `16:9`-compatible (or not normalizable by approved crop policy)

## Problem 2: On-Screen Text Overlaps with Subtitles

### Root Cause

- The timeline included a large `text` track set with many meme/commentary lines that duplicated narration meaning.
- Audio narration generation failed (credit blocked), so information delivery shifted even more to overlaid text.
- Result: when subtitles are also present (player subtitle or generated subtitle), semantic and spatial overlap becomes high.

### Fix

1. Define single information channel policy:
   - keep subtitles for narration, and reduce in-frame text to section headers only
   - or keep in-frame text only and disable subtitle generation
2. Introduce semantic de-duplication:
   - if subtitle sentence is similar to existing on-screen text in same time window, suppress one layer
3. Enforce safe zones:
   - title/header zone at top
   - subtitle zone at bottom
   - no shared zone occupancy in the same time range

## Problem 3: Subtitle Font Keeps “Changing”

### Root Cause

- Font family was mostly consistent (`montserrat`), but style parameters changed frequently clip-by-clip:
  - size
  - weight
  - color
  - stroke width and stroke color
  - vertical position
- Frequent style changes made subtitles look visually unstable, interpreted as font changes.

### Fix

1. Create one global subtitle style preset (single source of truth):
   - fixed family, weight, color, stroke, baseline position
2. Limit variation to controlled rules only:
   - long-line auto-shrink within a narrow range
   - optional distinct style for section title cards only
3. Disallow per-clip arbitrary style overrides for subtitle-type text clips.

## Implementation Checklist

- Update generation stage to always pass explicit ratio for images and videos.
- Add scene completion checker before edit stage (`all_required_scene_videos_ready == true`).
- Add ratio validator before `execute_edit_video`.
- Split text layers into:
  - `title_card` (optional, sparse)
  - `subtitle` (primary narrative text)
- Add style linter to reject subtitle tracks with too many unique style tuples.

## Priority

- P1: ratio enforcement + assembly gate + scene failure stop
- P1: text/subtitle de-dup and zoning
- P2: subtitle style preset + style linter

