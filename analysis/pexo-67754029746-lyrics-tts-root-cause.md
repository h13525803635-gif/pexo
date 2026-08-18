# Project 67754029746: Lyrics Were Spoken by TTS

## Summary

The project requested a 45-second, 100 BPM cinematic pop duet. The delivered audio used spoken TTS for the lyrics because the planning contract classified sung lyrics as `audio_list_speech`, then explicitly required TTS assets. The background music was independently forced to be instrumental. The result was structurally guaranteed to be instrumental music plus spoken lyrics.

Confidence: confirmed.

## Evidence

Source: Metabase database 3 (`pg-server`), timezone `Asia/Shanghai`.

- Project `67754029746` was created at `2026-08-18 15:48:48`, with the user request containing `two singers`, `pop duet`, `100 BPM`, `sing every line`, and `synchronized audio`.
- Script handoff at `15:51:00` declared each lyric row as `planned_source: audio_list_speech` and `requires_tts_asset_before_video: true`.
- Music generation at `15:51:44` used `mode: text2music`, `force_instrumental: true`, and excluded `vocals`, `lyrics`, and `singing`.
- Seven `audio_produce` calls at `15:53:02-15:53:29` used `mode: text2speech`, ElevenLabs `eleven_v3`, and passed complete lyric lines as the `text` input.
- Video generation at `15:54:21` passed those TTS MP3 files as `audio_list` and asked the video model to synchronize mouth movement.
- QA at `16:17:15` inspected only visual frames. No audio inspection asserted singing versus speech, vocal identity, duet coverage, or BPM alignment.
- The project contains 316 persisted messages, one thread, 29 assets, and 8 audio assets. Project status is `ACTIVE` with execution status `INTERRUPTED`.

## Causal Chain

`singing/duet intent` -> `lyrics modeled as generic speech` -> `audio_list_speech` -> `text2speech` for every lyric block -> `instrumental-only BGM` -> `TTS MP3 used for lip-sync` -> `visual-only QA` -> spoken lyrics delivered.

## Root Cause

The earliest responsible decision was in the Script Handoff audio contract. The system had no first-class `sung_vocals` route, so it treated visible singing as visible speech. Generation then followed its contract correctly and invoked TTS. Voice-search wording such as "pop singing voice" and `[excited]` style tags could not turn a text-to-speech model into a melodic singing synthesizer.

## Contributing Failures

1. The audio schema conflates dialogue/VO and singing.
2. The router has no hard conflict check for `song/sing/lyrics/duet` plus `text2speech`.
3. `force_instrumental: true` was allowed even though vocals were required.
4. `speaker: Both` was not validated against two vocal stems or a verified duet master.
5. The 45-second target was allowed to drift to roughly 57 seconds after measured TTS durations.
6. Audio QA was omitted; only visual frame checks were performed.

## Correct Production Route

1. Classify the request as `performance_type: sung_vocals`, `vocal_structure: duet`, and preserve BPM/duration requirements.
2. Generate a vocal-capable song or singing-synthesis asset. Prefer separate instrumental, female, and male stems when the video requires per-singer lip-sync.
3. Use the female stem for female shots, the male stem for male shots, and both stems or a verified duet master for Together shots.
4. Generate visuals against those singing assets. Do not create lyric audio with `text2speech`.
5. Mix the final instrumental and vocal stems once, with one authoritative vocal clock.
6. Keep the final timeline within the requested 45-second tolerance; shorten/recompose when it does not fit.

## Required Product Fixes

### Script Skill owner

- Add `audio_intent.performance_type: sung_vocals` and `planned_source: generated_song_vocals`.
- Reserve `audio_list_speech` and `post_vo` for spoken dialogue, narration, and VO.
- Require a `music_lipsync_feasibility_declaration` for visible singing.

### Router / Generation Preflight owner

Block the call when any of these are true:

- singing intent plus lyric text passed to `audio_produce(mode=text2speech)`;
- vocals required plus `force_instrumental=true` without a separate vocal asset;
- `speaker=Both` without two voice stems or a verified duet master;
- actual duration outside the requested tolerance.

If the selected provider cannot produce controlled singing, surface a capability limitation instead of silently falling back to spoken TTS.

### QA / Assembly owner

For every vocal asset and final video, assert:

- sung rather than spoken delivery;
- correct female/male/both speaker assignment;
- lyric coverage and no duplicate/missing vocal layer;
- BPM and duration fit;
- final mix contains the intended vocal owner exactly once.

Sound-on video QA must use combined audio-visual inspection, not visual frames alone.

## Regression Assertions

- A request containing `song`, `sing`, `lyrics`, `duet`, or `chorus` cannot produce a lyric-bearing `text2speech` call.
- A song request cannot finish with `force_instrumental=true` unless a separately verified vocal master/stems are present.
- A Together section cannot pass with only one singer asset.
- A 45-second request cannot be delivered outside the declared duration tolerance.
- The final QA record must contain an audio verdict for `sung_vs_spoken`, speaker assignment, timing, and final mix ownership.

## Investigation Scope

- Database: `3` (`pg-server`)
- Project: `67754029746`
- Threads discovered/inspected: `1 / 1`
- Persisted messages inspected: `316`
- Assets: `29` total, `8` audio
- Reporting timezone: `Asia/Shanghai`
- No separate trace/observation tables were available in the operational schema query; conclusions are based on persisted project messages, tool arguments/results, project state, and asset metadata.
