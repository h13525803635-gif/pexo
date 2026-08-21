---
name: modification-skill
description: Use whenever a user wants an already-delivered video changed, including Comments.texts / Mark what to fix, timestamped or circled feedback, and second-round follow-ups. Normalize comments and visual evidence, locate the exact target, choose the lightest valid fix, and verify it on the final output. For voice feedback, resolve and regenerate only affected spoken layers, then route locked results through Assembly and Motion. Route upstream only for fundamental direction or structure changes; not for creation from scratch.
compatibility: add_attachments, analyze_file_content, audio_produce, image_generate, music_generate, video_generate, voice_preview, voice_search
metadata:
  version: v5
---

# Read Gates - check before every tool call

Keep reading with `offset` until the tool output reaches the file's EOF; never stop based on a documented line count or expected number of reads. A partial read is a violation.

**Blocking** (calling the gated tool before completing the read is a violation):

| Before you... | Read completely first |
|---|---|
| Re-generate any segment (`video_generate`) | `references/video-models-routing.md` |
| Search, preview, resolve, produce, or fit a spoken layer | `references/voice-production-and-resolution.md` |
| Interpret annotated/second-round feedback, act on a subtitle/caption/text complaint, or deliver any modified video | `references/complaint-triage-and-reverification.md` | read to EOF |

Before the first gated tool call in a session, confirm in your internal plan which gates apply and that each gated file was read to its last line.

When a modification touches music, BGM, SFX, lyrics, timestamps, or audio/video synchronization, also read `references/music-audio-production-strategy.md` completely before classifying the change. Apply its change rules: song changes regenerate the complete song and recompute timing; mix-only changes preserve the song map; coupled visible speech/singing is not an audio-only replacement.

**Consult when relevant:** Read `references/image-generation-guide.md` completely to EOF before regenerating packaging key frames or cover images via `image_generate`.

---

# Usage

This skill is a **self-contained modification planner and regeneration engine**. Two entry points:

1. **Iteration on produced videos**: User gives feedback on a Pexo-generated video -> analyze feedback -> decide approach -> execute modification -> deliver updated video.
2. **Direct video editing request triage**: User brings an existing video and wants edits (trim, concat, add music, add narration, adjust pacing, crop, speed change, etc.) -> route single-clip rough-cut edits to assembly-skill and final composition to motion-skill with a concrete edit brief; use this skill only when the request also needs feedback interpretation, segment re-generation, or new audio/image generation.

**Input**: user feedback on the current video (including the real `Comments.texts.final_video_id`, `single_comments[]`, and `segment_comments[]` payloads) or direct editing instructions on any video, plus the sequence list and production context of the current video when available.

**Output**: The updated finished video. **Never expose internal terms** (sequence numbers, generation modes, sound on/off, model names) to the user - use natural creative language.

**Downstream split**: **single-clip rough-cut edits (trim, concat, speed, crop, scale, volume) belong to assembly-skill**; **final timeline assembly, transitions, subtitles, overlays, audio mix, cover, and render execution belong to motion-skill**. Hand off a concrete edit brief instead of executing those steps here. This skill directly executes only re-generation (`video_generate`), audio production (`audio_produce`, `music_generate`), image re-generation (`image_generate`), and analysis/inspection.

---

# Routing: Local vs Upstream

Before executing, determine whether this modification can be handled locally or needs to go upstream.

## Handle Locally (this skill)

- **Point impact** - single element fix: one accessory, one expression, one timestamped issue, one audio cue.
- **Line impact** - repeated class: multiple scenes of the same type need the same kind of fix.
- **Post-production adjustments** - analyze the requested change, then route by layer: single-clip rough-cut (trim, speed, crop, scale, volume, concat) to assembly-skill; composition-level changes (transitions, fades, subtitles, overlays, audio mix) to motion-skill.
- **Audio additions** - add voiceover/narration, add BGM, add SFX to existing footage.
- **Audio adjustments** - volume rebalancing, BGM re-shaping, VO timing fixes.
- **Segment re-generation** - re-generate one or more specific segments while preserving the rest.
- **Reassembly** - rough-cut original/regenerated segments via assembly-skill, then hand the locked material + edit decisions to motion-skill for recomposition (timeline/mix/subtitles/cover/render).
- **Direct editing requests** - per Usage entry point 2: purely non-generative -> route by layer; stay here only when interpretation or new generation is required.

## Route Upstream

- **Surface impact / rejection** (user hates the whole direction, wants a completely different concept) -> **brainstorm-skill** - the creative direction needs rethinking, not modification.
- **Structural change** (user wants different sequence structure, different story arc, different scene order) -> **script-skill** - the production plan needs redesigning.
- **Major asset gap** (user provides fundamentally new reference material that changes subject coverage) -> **subject-asset-skill** - the asset foundation needs rebuilding.
- **Full redo** (user explicitly asks to start fresh) -> full pipeline from brainstorm.

**Decision test**: "Can I fix this by changing specific segments, editing the timeline, or adding audio layers - without re-inventing the story or production plan?" If yes -> local. If no -> upstream.

---

# Constraints & Behavior

## Feedback Analysis (Rules 0-5)

0. **Modification Evidence Gate** (mandatory before intent classification whenever `Comments.texts` contains `single_comments`, `segment_comments`, or a second-round reference to the previous revision): normalize the input before interpreting it.
  - Follow `references/complaint-triage-and-reverification.md`: preserve every real payload field, distinguish annotated composite `_a.png` from overlay-only `_b.png`, recover the clean source frame from `final_video_id + frame.timestamp`, and emit `modification_context` plus `annotation_understanding_manifest` before Rule 1.
  - Every circled/annotated modification must call `analyze_file_content` with `mode: standard` at minimum; escalate to `mode: pro` for identity/global consistency, contradictory evidence, or ambiguity. Keep model/FPS/sampling parameters tool-owned. Rules 4-5 must consume the manifest.
  - Require exact set equality between input `comment_id` values and `annotation_understanding_manifest[].comment_id`, with actual consumed assets and evidence recorded. If an annotated asset cannot be recovered, mark spatial understanding `blocked`. Continue only when text/time evidence independently identifies the target and the requested edit does not depend on the missing visual evidence; otherwise stop and clarify. Tell the user exactly which evidence was unavailable.

1. **Feedback Analysis Pass** (mandatory, before deciding approach): Analyze the user's feedback as a structured requirement set rather than reacting to literal wording only. Classify **intent type** (confirmation / modification / supplementation / inquiry / rejection), **deliverable layer** (narrative / visual / audio / cross-layer), and **impact scope** (**point** - single local element or moment; **line** - repeated class or multiple scenes of the same type; **surface** - global tone, style, or whole-video structure). When multiple comments exist, determine whether they are independent, sequential, elaborative, or contradictory before planning execution. Use this pass to consolidate the true requirement before choosing approach.

2. **Production Context Recovery** (mandatory, before deciding approach): Do not choose a modification strategy until you have recovered how the current video was actually produced: the creation path (**direct audio-visual generation** / **silent visual generation + post audio assembly** / **hybrid assembly**), the sequence-level **audio narrative mode** (`on_camera_sync_speech`, `voiceover_narration`, `silent_broll_with_text`, `nat_sound_only`), whether the clip contains **valuable embedded audio** (co-generated SFX, dialogue, monologue, ambience, vocal events), and whether the requested change touches visual appearance only, post-produced audio only, or an **audio-visual coupling** where visible action and audible output must stay synchronized.

3. **Constraint Guard** (mandatory, before deciding approach): Before any modification, verify the proposed change against **all confirmed constraints** from the session:
  - **Duration**: If the user confirmed a specific duration, any modification must preserve it. If a proposed approach would alter duration, either compensate elsewhere or explicitly inform the user.
  - **Aspect ratio**: Modifications must preserve the confirmed aspect ratio.
  - **Creative direction**: Modifications should stay within the confirmed creative direction unless the user explicitly requests a direction change.
  - If a user's feedback inherently conflicts with a constraint (e.g., "make it faster" might imply shorter duration), **surface the tension** and propose a solution.

4. **Locate Sequences**: Map by time point / plot / visual content to the sequence list; identify which sequences to change. Default to **local modification** - never redo the entire video unless the user explicitly requests it.

5. **Localized Feedback Routing**: When feedback is timestamped, frame-anchored, or clearly targets a single local detail, default to the lightest viable local strategy. Route from `annotation_understanding_manifest.target` and `recommended_action`, not from raw user text. Every downstream handoff must retain `request_id`, `base_revision_id`, `comment_id`, `target_asset_id`, and `time_range`.
  - **Localized Visual Feedback**: a local accessory, prop, expression, object detail, or one moment in one shot.
  - **Localized Audio Feedback**: a local level issue, one specific sound cue, one isolated BGM adjustment.
  - **Audio-Visual Coupled Feedback**: a requested change where visible action and audible output must be correct **together** (e.g., lip sync, action-tied sound).
  - Routing priority for localized issues:
   1. post-production adjustment (direct edit),
   2. structure-preserving visual edit,
   3. selective re-generation,
   4. full re-generation only if lighter paths are not suitable.

### Subtitle/Text Complaint Triage

When the user complains about subtitles/captions/text being wrong, do not immediately choose ASS, manual captions, or subtitle reformatting. Classify the failure with media evidence first, per `references/complaint-triage-and-reverification.md` (blocking read gate): run its mandatory checks, assign one or more failure buckets, and follow its routing. If multiple buckets apply, fix the earliest owner in the pipeline first: script declaration -> generation prompt/key frame -> assembly timeline -> render styling.

### Second-round Conversation and Revision Safety

Before interpreting a follow-up message, require one valid complete-video receipt for the current `revision_id + content_hash`. Reuse it while that revision is unchanged; call `analyze_file_content` on the entire current revision with `mode: standard` only when the receipt is missing/stale or the video changed. Interpret each comment from that receipt plus targeted current-revision evidence for its time/region; do not reuse only the prior annotated frame or local window.

- Inherit the prior target only when unambiguous; otherwise clarify. Feedback such as "still too small" that disproves the prior fix invalidates the old conclusion and verification result. A new circle/timestamp creates a new `comment_id` against `current_revision_id`.
- Reject or rebase a stale `base_revision_id`; never modify an obsolete video silently.

### Modification Plan and Verification Contract

After AFC understanding and before execution, emit the `modification_plan` defined in `references/complaint-triage-and-reverification.md`, with one operation per comment and the evidence identity retained on every handoff. Do not deliver when its `modification_verification.overall_pass` is false or when the target cannot be judged from final-output evidence. Retry only the affected shot/window when possible.

## Modification Approach Decision (Rule 6)

6. **Decide: Direct Edit, Re-generate, or Combined?** Apply the **progressive modification principle** - start with the lightest-touch approach, escalate only if insufficient.

  **Non-generative edit handoff - route by layer** (fastest, preserves content):

  *Rough-cut / single-clip edits -> assembly-skill:*
  - **Pacing / timing** - trim dead air, adjust speed. When the user says "speed up the pacing", default to tighter editing (trim), not playback speed.
  - **"too long" / "trim"** - explicit duration reduction. Verify against confirmed duration constraint.
  - **Framing** (zoom in, crop) -> specify scale/crop parameters.

  *Composition / packaging changes -> motion-skill (HTML timeline):*
  - **Transitions** (jarring cuts, needs fades) -> specify transition type and duration.
  - **Audio levels** (music too loud, voice too quiet) -> specify per-track volume intent for the mix.
  - **Speed ramps, Ken Burns effects** -> specify keyframe animation intent.
  - **Post text/card overlays** (lower thirds, CTA cards, data callouts, title/chapter cards, PiP panels, watermarks) -> HyperFrames live HTML/CSS/GSAP layer requirements, when they do not need scene-depth occlusion or frame-accurate object tracking.
  - **Subtitles/captions** -> see Subtitle/Text Complaint Triage; subtitle render fixes are composition work.

  **Re-generation via `video_generate`** (when content must change):
  - **Content changes** (different visuals, scenes, characters) -> **Seedance preferred**; use **Seedance Fast** when the goal is a quick reference-based visual anchor; use **Kling only** for precision single-frame/key-frame work, copyrighted IP, fine-detail, or in-frame text.
  - **Visual style changes** (restyle, re-skin - keep structure) -> **edit video (base)** (Kling only). Input <=10s, preserves duration and timing.
  - **Specific element motion** (e.g., "slow down the leaves") -> re-generate with updated motion descriptions in the prompt, not playback speed.

  **Audio production via `voice_search` / `voice_preview` / `audio_produce` / `music_generate`**:
  - **Add voiceover** -> when exact copy is already supplied or approved, resolve the voice and generate TTS without another confirmation stop; ask only when the spoken copy or required voice conditions are genuinely unresolved. Mix the produced speech into the video.
  - **Add BGM** -> design audio direction, generate via `music_generate`, mix into video.
  - **Modify existing voiceover** -> re-generate TTS for affected segments when timing changed; reuse when timing preserved.
  - **BGM re-generation after visual changes** -> use video-first approach: analyze updated footage before writing new BGM prompt.
  - **Volume complaint triage** -> before adjusting gain, inspect the relevant BGM / VO / source video with an audio analysis tool; use `lufs`, `true_peak`, and `duration` to decide between whole-track normalization and local gain edits.

  **Audio-Visual Coupled Feedback** requires special handling:
  - If the request depends on visible action and audible output being synchronized, do **not** treat it as ordinary audio-only feedback.
  - **Speech replacement / dubbing**: Treat this as speech modification, not generic voiceover. Use clone TTS only after explicit clone or voice-match intent and usable samples; resolve it through `references/voice-production-and-resolution.md` before `audio_produce`.
  - **Fallback ordering**: authorized clone, reference-guided `audio_list`, localized re-generation, then newly searched and user-accepted public replacement audio.
  - **Explicit downgrade disclosure**: If the final path is replacement audio on top of fixed visuals, tell the user clearly. A requested clone must never fall back silently to a public voice; stop and ask whether the user accepts a new public-voice search.

  **Prefer non-generative approaches when applicable** - they're faster and preserve existing content.

## Re-generation Rules (Rules 7-11)

7. **Execution Truthfulness**: If the path preserves structure and applies a true edit, describe it as an edit. If it re-generates footage, describe it as regenerated. Never tell the user "this is an edit" when the actual path is a new generation pass.

8. **Retry Limit**: After **2 consecutive failures** with the same approach, **stop and switch strategy** or inform the user of the limitation.

9. **Cascade Re-generation** (when create-video re-generation is needed):
  - **Serial-generated videos**: Modifying Sequence N requires re-generating N and all subsequent sequences. Cascade can be truncated at scene-change boundaries.
  - **Parallel-generated videos**: Modifying Sequence N requires only re-generating N.
  - **Hybrid videos**: Cascade applies within serial blocks; parallel sequences are independent.
  - **Edit video does not trigger cascade** - it preserves structure and timing.

10. **Re-generation Audio & Evidence Preservation**:
  - Preserve embedded audio for visual-only changes; regenerate coupled audio-visual events together. Match the surrounding audio character.
  - Reuse existing reference assets and key frames as visual anchors; character identity relies on references, not prompt text alone.
  - Fix text that belongs physically inside a scene by regenerating the dependent key frame/segment, not by disguising it with a post overlay.
  - Generation-model routing details live in `references/video-models-routing.md`.

11. **Asset Organization** (before re-generation):
  - When assets exceed capacity limits, classify material vs. reference, budget material first.
  - Only 1 reference video per call, **must be <=10s**.
  - Rank by role priority; keep within limits; silently drop lowest-priority items.

## Assembly & Delivery (Rules 12-16)

12. **Downstream Handoff**: This skill executes neither rough cuts nor final renders. Send original/regenerated assets plus the evidence-bound operation brief to **assembly-skill** for single-clip material prep/VO fit, and locked materials plus timeline/mix/text decisions to **motion-skill** for composition and delivery QA. Follow Motion's audio/assembly and caption gates.
  - **Reassembly source rule**: Always reassemble from **original individual segment files**, not from a previously rendered composite. Never set `volume: 0` on a source with co-generated audio.
  - **Preserved-source A/V clock rule**: When preserving a source video's original audio in a separate audio track, moving picture and sound must share the same source clock: `video.start_ms - video.source.in_ms == audio.start_ms - audio.source.in_ms`. Crossfade overlap does not permit video to start at `2000ms` while matching source audio starts at `3000ms`; use audio fade-in or a still/freeze-frame transition instead.
  - **BGM masking**: When co-generated audio conflicts with BGM, prefer BGM re-shaping (sectional volume changes) over re-generating video.

13. **Voiceover & Audio Layer Decisions**: inspect existing speech/audio/subtitles before adding a layer; never double spoken content. Resolve, produce, and fit new TTS through `references/voice-production-and-resolution.md`. A direct voice-add or replacement operation is incomplete if it stops after search or audition preview without a successful, probed `audio_produce` result. Reuse unchanged VO; regenerate only affected lines when copy or timing changes. Follow `references/complaint-triage-and-reverification.md` for subtitles.

14. **Interruption Recovery**: When a tool call is cancelled mid-execution, inventory surviving assets before re-planning, reuse successfully generated assets (never re-generate them), preserve successful parameter patterns, scope recovery to the exact failure point, and tell the user what survived and what remains.

15. **Cover Preservation**: After any reassembly, ensure the video remains preview-safe on frame 0.
  - If opening unchanged -> preserve existing cover.
  - If opening changed -> follow motion-skill's audio-and-assembly design reference (Cover bake-in) for cover source priority and bake-in.

16. **Targeted Re-verification** (blocking, before delivering any modified video): the specific point the user complained about must be proven fixed **on the final output**, not on an intermediate asset. Verify per `references/complaint-triage-and-reverification.md` (Targeted Re-verification): final-video frames as the verification surface, fail-seeking QA prompts only, character-by-character comparison for exact text (uncertain reading = FAIL), burned subtitles checked on the final video. If re-verification fails, the round is not complete - fix and re-verify, or report the remaining gap honestly. Never re-deliver with the complained defect unverified.

# References

For model routing (Quick Contract, Strategy Genes, Seedance/Kling decision tree, degradation chain), see `references/video-models-routing.md`.
For image generation (routing strategy, prompt rules, reference discipline), see `references/image-generation-guide.md`.
For voice search, preview, candidate switching, clone authorization, spoken-layer production, measured fit, continuity, and anti-doubling, see `references/voice-production-and-resolution.md`.
For subtitle/text complaint triage, subtitle artifact handling, and pre-delivery re-verification, see `references/complaint-triage-and-reverification.md`.
