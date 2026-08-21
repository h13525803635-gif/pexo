# Music And Audio Production Strategy

Shared strategy for music-first and video-first work.

## Route Selection

- **Music-first**: use when picture follows a song structure, visible singing/lip-sync is required, or a supplied song must be preserved. Generate one complete song first; picture follows its measured timeline.
- **Video-first**: use when footage is recorded, uploaded, rough-cut, or locked before music. Assembly adds BGM, ambience, and SFX without changing locked picture.
- Do not replace audio coupled to visible speech or singing. Route that event through Generation.

## Ownership

- Script owns the relationship between music and picture: user-supplied or approved lyrics, singer/voice constraints, hard duration requirements, main clock, visible singing, lyric/beat relationships, visual energy, diegetic sound, and allowed sync downgrades. It declares semantic segments, not guessed final timecodes. Generation realizes these constraints and does not silently rewrite them.
- Generation owns complete-song production when a song or audio conditions generated footage. Returned audio is the time truth and its sync clock must be preserved.
- Assembly owns BGM/SFX generation, dialogue protection, ducking, fades, source-audio preservation, timeline fitting, mixing, and audio QA.
- Motion owns subtitles, lyric presentation, brand/package overlays, final composition, and render QA. It does not invent or regenerate music.

## Music Tool Scene Contract

The only request shape is:

```yaml
scene: bgm | song | bgm_sfx
duration_ms: integer
music_prompt: string
```

Routing is deterministic by `scene`: `bgm` and `song` use Music V2; `bgm_sfx` uses the sound-effects-capable route. Do not switch route based on prompt wording.

- `bgm`: instrumental bed only. Dialogue protection, ducking, echo, and final mix are post-production.
- `song`: one complete song, with lyrics/structure planning and timestamps. Never generate multiple songs for picture segments.
- `bgm_sfx`: one complete asset containing requested music, ambience, and timed diegetic events. Do not repeat events already present in preserved source audio.

## Music-First Timing

After the complete song returns, inspect the same version's actual duration, waveform/audio facts, vocal activity, lyric timestamps, section structure, BPM/beat/energy changes, breaths, phrase endings, and sustained notes. Treat actual audio as truth. If actual duration violates a hard delivery duration, regenerate or obtain an explicit scope change; never silently truncate or time-stretch the canonical song. Conflicting lyric timestamps block lyric/subtitle locking until the selected audio revision is inspected and the conflict is resolved or explicitly marked for review in the handoff.

Build a `music_rhythm_plan` before selecting cuts. Reject cuts inside words, syllables, sustained notes, tails, or uninterruptible actions. Prefer phrase endings, breaths, section boundaries, beat proximity, energy changes, and Script's cut motive, but visual transitions cannot override continuous vocal delivery.

Select all cuts globally. Cover `[0, song_duration]` continuously with adjacent, non-overlapping segments. A generation request is normally at most 15 seconds, but 15 seconds is an upper bound, not a target. If a continuous vocal phrase exceeds the limit, use a safe phrase/measure boundary or explicitly downgrade to non-lip-sync coverage before generation. When visible singing is mandatory, downgrade is forbidden: regenerate with a compatible song structure or block for an explicit scope decision.

Keep these windows distinct: `song_segment` is the exact song range, `generation_window` is optional context within model limits, and `final_edit` is the retained picture range mapped back to the song range. After generation, only make small boundary adjustments (about +/-300-500 ms), updating both adjacent segments together. Do not re-plan the song in Assembly.

## Timestamp And Subtitle Contract

For song generation, request timestamps and save the MP3 plus raw word-level timestamps under the same generation revision. Remove bracketed performance markers and section labels from visible text. Merge words using original lyric lines: sentence start is the first valid word and end is the last valid word. Preserve `section`, using `unknown` when unresolved. Internal times are milliseconds; convert only at presentation/export.

## Audio Layers And QA

Layer priority is main song/speech, action-tied SFX, then ambience. Preserve valuable source/co-generated audio. Never stack two versions of the same event or duplicate the main song waveform. Validate duration, continuous coverage, source-version consistency, dialogue/lyric/lip-sync alignment, beat/action alignment, transitions, loudness, silence, clipping, and final playback.

## Change Rules

- Change lyrics, song structure, style, BPM, singer, or song duration: regenerate the complete song and recompute timing.
- Change picture only: keep the song revision and cuts unless new picture invalidates the timing contract.
- Change ambience/SFX/mix only: keep the song segment map and re-run full audio QA.
