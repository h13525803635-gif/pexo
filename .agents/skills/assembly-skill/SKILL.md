---
name: assembly-skill
description: Prepare and lock actual video/audio materials for video projects, including probes, sequence coverage, rough-cut windows, measured post-VO fit, actual timeline, subtitles, and final audio truth before Motion. Use directly for standalone voice search, audition, TTS, and authorized voice cloning when the current request does not require a video pipeline, and for non-generative single-file edits. Consume Script, Subject, and Generation handoffs; preserve confirmed asset lineage and emit the Assembly Handoff.
compatibility: add_attachments, audio_produce, media_probe, media_process, music_generate, render_frame, submit_render, voice_preview, voice_search
metadata:
  version: v5
---

# Usage

Turn generated, supplied, and existing media into one measured timeline/material package for Motion. Assembly prepares and locks materials; Motion owns final composition, transitions, typography/layout/animation, multitrack render, and delivery QA.

Assembly owns:

- verifying every Script sequence is fulfilled by generated media, existing media, or Motion DOM
- probing actual media facts and STT when embedded speech is possible
- producing/fitting `post_vo` speech when it does not already exist
- rough-cut source windows and actual timeline placement
- actual audio assets, coverage, single final audible owner, subtitle artifacts, and protected native-clock rules

It does not rewrite Script intent, create generation outputs, design packaging geometry, or produce the final composition/render.

# Activation Gate

Activate Assembly for either route:

- **Video material route**: a Script, Subject, or Generation handoff needs media preparation, post VO, subtitle timing, timeline locking, audio ownership, or a package for Motion.
- **Standalone audio route**: the user asks to search, audition, select, clone, or directly produce a voice/audio asset without requesting a video pipeline.

The standalone audio route does not require Script, Subject, Generation, Website Asset Brief, visual coverage, timeline, subtitle, or Motion handoff fields. It still requires the full voice-resolution, production, probe, and delivery gates below.

# Blocking Reads

For the video material route, read all active paths to EOF before material decisions:

1. `artifact_context.artifacts.script_handoff.path`
2. `artifact_context.artifacts.subject_manifest.path` when present
3. `artifact_context.artifacts.generation_handoff.path` when generation occurred
4. `artifact_context.artifacts.website_asset_brief.path` for URL-backed work

Then read:

- `references/music-audio-production-strategy.md` before selecting `bgm`, `song`, or `bgm_sfx`, generating music, fitting a song, or making audio ownership decisions
- `references/voice-production-and-resolution.md` before `voice_search`, `voice_preview`, `add_attachments`, choosing the voice selector for `audio_produce`, or producing, replacing, or fitting post VO
- `references/design-audio-and-assembly.md` before audio truth/ownership or `replace_audio`
- `references/assembly-handoff-contract.md` before locking the package
- `references/editing-rhythm-design.md` when rough-cut rhythm changes source windows

For the standalone audio route, read `references/voice-production-and-resolution.md` completely before the first voice action. Use only the current user request, current-project voice state, and authorized current-project sample assets; do not invent a Script Handoff.

# Workflow

0. Classify the route. For standalone audio, follow the Direct Audio Delivery section and do not run video coverage, timeline, subtitle, or Motion packaging steps. For video material work, continue below.
1. Build a coverage map from Script `sequence_breakdown` and each sequence's `visual_stack`. Resolve the primary visual through edited website footage, Generation output, a verified real image, or `motion_dom`; never duplicate another clip to cover missing content. For URL work, require matching proposal ids and the complete Script coverage ledger, Subject bindings, and Generation lineage. Consume every required Assembly-owned `assembly_source`; carry Motion-owned rows as pending passthrough.
2. Probe every media asset used in the timeline. Run STT for possible embedded speech, visible dialogue, audio-list output, or any sound-on clip that overlaps post VO.
3. Audit existing speech before adding `post_vo`. Reuse an exact usable speech asset when one already realizes the line. When a new TTS asset is required, resolve its `voice_identity_id` from matching Script `voice_requirements` or the current approved voice request through `references/voice-production-and-resolution.md`, then produce by `speech_label` with `audio_produce`. Use the same reference to probe measured duration and finish the voice-fit loop without truncation or speed hacks.
4. Rough-cut only with `media_process`; record exact source windows and post-edit probes. `direct_clean` keeps suitable website footage unadorned; `direct_packaged` prepares the clean edit and preserves room for Motion layers. Do not claim action or spatial coverage by adding pan/zoom to a still. Apply the declared `aspect_treatment`; never stretch media or destructively crop must-show content merely to fill the output frame.
5. Build the actual visual timeline, audio asset ledger/timeline, subtitle tracks, coverage facts, Motion constraints, and confirmed-asset consumption ledger. Each audible item gets one owner. Every confirmed required URL use receives one status: `consumed_in_generation`, `consumed_in_assembly`, `pending_motion`, or `user_approved_omission`.
6. Write `<project_label>__assembly_handoff__locked__RNN.md` with exactly one `yaml assembly_handoff_v2` block. Read it back to EOF, parse it with duplicate-key rejection, rerun the contract validation, then carry `artifact_context.artifacts.assembly_handoff.path`.

# Direct Audio Delivery

For a standalone request:

1. Classify it as direct production, audition, or selection only using `references/voice-production-and-resolution.md`.
2. Resolve the requested voice without inventing language, demographic, accent, use-case, or timbre evidence. Preserve the realized `voice_resolution` in active current-project state.
3. For direct production, require exact approved copy, call `audio_produce`, probe the returned final asset with `media_probe`, and surface that final asset through `add_attachments`. Do not call `voice_preview` or stop for confirmation unless the user explicitly requested an audition.
4. For an audition, surface only the preview asset and wait for the user's selection before formal TTS. For selection only, record the selected candidate without manufacturing a speech asset.
5. A successful standalone audio operation produces no Assembly Handoff unless it is later incorporated into a video project.

# Direct Edit Exception

For a user-requested non-generative edit on an existing standalone file, use `media_process` for trim/concat/convert/crop/resize/speed/volume/loudnorm/extract/replace audio. Verify before/after facts. A request to revise an already delivered Pexo video enters modification-skill first.

# Hard Gates

Do not lock when:

- active upstream artifacts required by the selected route were not read to EOF or label/path conflicts remain
- any Script sequence lacks real coverage or is covered by unrelated duplicate/stretch filler
- a `subject_motion`, `process_demo`, or `spatial_motion` sequence is covered only by static-image pan/zoom, or cross-aspect media ignores its declared treatment and loses must-show content
- an asset/path/duration needed for timeline math is unresolved or unprobed
- post VO is incomplete, unmeasured, or exceeds its planned window
- a required direct-production or post-VO line stops after search or audition preview without a produced and probed final speech asset
- a new public TTS asset lacks a realized current-project `voice_resolution`, or a requested clone silently falls back to a public voice
- one `speech_label` would be audible from both embedded and separate speech
- a protected generated-native clock would be muted/replaced without waveform-preserving proof
- a `keep_nat_sound` segment lacks matching Script SFX intent and actual event windows
- non-silent work lacks an actual audio completion route/coverage to the visual endpoint
- subtitle files/timing are missing for `subtitle_track` requirements
- URL proposal ids disagree, a required Assembly-owned use is absent from locked media, Generation lineage is missing, or a Motion-owned row is dropped/changed instead of passed through
- any confirmed required URL use lacks exactly one intermediate ledger status
- the package copies Script exact text, Subject inventories, Generation prompts, retry history, or Motion layout

# User-Facing Behavior

Keep internal timeline/audio mechanics hidden. Surface only material gaps, failed voice production, or user decisions that change creative outcome.
