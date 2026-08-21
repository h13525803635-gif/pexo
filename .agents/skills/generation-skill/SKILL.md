---
name: generation-skill
description: Generate Script-declared production media from any required validated Subject bindings. When Script requires Subject, send exact locked references in every affected payload and reject prompt-only continuity or substitutions. Own generation, conditioning audio, QA, and Generation Handoff; standalone stills go to Image Production.
compatibility: add_attachments, analyze_file_content, audio_produce, image_generate, media_probe, media_process, music_generate, video_generate, voice_preview, voice_search
metadata:
  version: v9
---

# Usage

Convert Script sequences and Subject bindings into actual generated media. Generation decides how many calls to make; a Script sequence is not automatically a generation call, and one capable 4-15s call should cover multiple compatible narrative moments when possible.

Generation owns:

- `generation_unit_label`, call grouping, legal split reasons, provider/model/mode, prompt, payload, and output paths
- exact reference/audio payload bindings from Subject assets
- native-audio policy and sync-clock preservation
- frame-chain strategy, nodes, junction assets, and ordered unit bindings
- generated video/image/audio assets, measured probes, and post-generation QA facts
- one AFC acceptance receipt for every selected generated image/video revision, including reference and user-constraint comparisons

It does not rewrite Script story/copy/render routes, create Subject identities, assemble the final timeline/mix, design Motion layout, or deliver the final render.

# Activation Gate

Activate Generation when Script requires `generated_video`, `generated_image`, or `aigc_image`, or when audio must be produced/probed to condition a production generation call. Do not activate for Brainstorm previews, Subject supplementary references, standalone images, existing-media-only work, pure Motion DOM, or post-production audio that does not condition generation.

Subject is optional only for explicit description_only work with no activation trigger. When `subject_required: true`, require the validated manifest before any call.

# Blocking Read Order

Read active artifacts to EOF before planning or any generation call:

1. `artifact_context.artifacts.script_handoff.path`
2. `artifact_context.artifacts.subject_manifest.path` when Subject-relevant material exists
3. `artifact_context.artifacts.website_asset_brief.path` for URL-backed work

For URL-to-video work, require matching confirmed proposal ids in Website Asset Brief, Script, and Subject plus complete Script coverage and Subject URL binding rows. Require `audio_intent`. Missing or pending fields return to their owning stage before any production call.

Then read triggered references completely:

1. `references/afc-multimodal-policy.md`
2. `references/generation-preflight-gates.md`
3. `references/generation-blueprint-design.md`
4. `references/generation-execution-rules.md`
5. `references/video-models-routing.md` before each route/model decision or fallback
6. `references/voice-production-and-resolution.md` before `voice_search`, `voice_preview`, `add_attachments`, or choosing the voice selector for `audio_produce`
7. `references/generation-voice-strategy-execution.md` before any speech, visible performance, voice reference, audio list, or post-VO production
8. `references/video-generation-execution.md` for prompt, split, reference, SFX/BGM, or audio sync work
- `../script-skill/references/music-audio-production-strategy.md` before song generation, timestamp handling, music-first segmentation, or audio-conditioned video generation
9. `references/video-generation-strategy-genes.md` with the video execution contract
10. `references/image-generation-guide.md` for image/key-frame generation
11. `references/generation-deterministic-visual-payload-guide.md` for exact readable text or text-bearing assets
12. `references/generation-handoff-schema.md` for the exact Runtime View shape
13. `references/generation-handoff-contract.md` before declaring generation complete

# Workflow

1. Validate Script/Subject sources and label references. Preserve every upstream planning/subject/asset label.
2. Build generation units from each sequence's `visual_stack`, material sources, duration, reference compatibility, audio clocks, and model capabilities. Generate only when the primary route or a declared bridge requires it. Preserve the exact-information and packaging layers for their owners. Prefer one call within model capability. Every additional video unit boundary needs an allowed `split_reason`.
3. Read each Script `audio_content.speech` row's `planned_source` before audio work. Reuse an exact supplied or already-produced speech asset, and do not search for `co_gen_dialogue`, `audio_list_voice_ref`, or `audio_list_music`. Only when `audio_list_speech` or `post_vo` requires a new TTS asset, resolve that row's `voice_identity_id` through the matching Script `voice_requirements` row and `references/voice-production-and-resolution.md`, then produce it with `audio_produce`. Probe speech before video when Script requires `measured_vo_first` or audio-conditioned generation, bind returned assets to `speech_label`, and preserve the realized `voice_resolution`.
4. Resolve each Subject binding by exact `asset_label -> path` and matching role. For strict continuity, include every Subject anchor in each affected unit payload. For URL work, include every Generation-owned required `asset_use_id` in the actual payload and write its lineage row. Prompt words never replace payload proof.
5. Choose and execute frame-chain strategy only when Script continuity requirements need it. Generate/register node assets here; do not expect Subject or Script to provide frame-chain nodes.
6. Run pre-call gates, execute calls, and probe every output. For each selected generated revision, call `analyze_file_content` with a business-specific acceptance `query`: image or silent video -> `scope: visual`, `mode: standard`; sound-on video -> `scope: combined`, `mode: standard`. Compare the actual output with its generation unit, user hard constraints, and every payload-bound reference. Require the selected output and every comparison reference claimed by the verdict in `consumed_assets[]`. Every time-bound video requirement result, defect, text/logo instability, identity drift, and audio-visual sync issue must reference `video_evidence.evidence_id`; visual claims require their source timestamp and viewable representative frame. A `kill` or `insufficient` result that would trigger regeneration gets exactly one targeted `analyze_file_content` confirmation on the same revision and smallest failed time/evidence range, using the original finding's modality as `scope` and `mode: pro`. Rerun only the smallest failed unit/time range; never fill missing duration by duplicating/stretching another clip.
7. Validate expected subject fidelity, reference fidelity, required action/motion, gross integrity, duration, and audio lineage from the probe plus AFC receipt. Record every consumed receipt/evidence id in the Generation Handoff. Exact readable text assigned to Motion is not a generation success criterion. `subject_motion`, `process_demo`, and `spatial_motion` pass only when the requested motion actually occurs; camera movement over a still does not count. For `measured_vo_first`, materialize the measured VO timeline, VO asset bindings, subtitle assets, measured visual coverage, locked visual segments, and planned timeline ledger required by `references/generation-handoff-contract.md`. Write `<project_label>__generation_handoff__post_qa__RNN.md` with exactly one `yaml generation_handoff_v2` block, read it back to EOF, parse and rerun the handoff validation, then carry `artifact_context.artifacts.generation_handoff.path`.

# Direct Delivery

A selected generated video may be delivered without Assembly or Motion only when it already covers the complete Script duration and sequence content, its native audio is the final audio, and it needs no source edit, separate VO/BGM/SFX, subtitle, exact text layer, transition, overlay, packaging, or HTML composition. Generation Handoff write/read-back and final media QA remain mandatory before delivery. Any failed condition routes to Assembly and/or Motion instead of silently doing their work here.

# Hard Gates

Block before calls or handoff when:

- an active Script/Subject artifact was not read to EOF or their labels conflict
- `subject_required` work lacks a validated Subject Manifest
- Script required fields, exact speech/text, or continuity requirements are unresolved
- a strict/recurring Subject binding lacks the exact usable reference asset
- strict continuity lacks an approved anchor in the actual payload or its output comparison; prompt text is not evidence
- a specific real subject is routed to prompt-only generation, or its verified reference is absent from the actual generation payload
- the planned motion requirement is `subject_motion`, `process_demo`, or `spatial_motion` but the call or QA result provides only static-image camera movement
- a named asset binding does not resolve to one exact usable path with a matching declared role
- URL approval/proposal ids disagree, a required Generation-owned use lacks a Subject binding or actual payload label, or the payload silently substitutes another asset
- URL work lacks complete `url_asset_lineage` evidence for every required Generation-owned use
- an additional generation unit has no legal split reason
- prompt/payload asks the video model to typeset exact copy assigned to another render route
- visible speech lacks exact text, compatible planned source, visual reference, or protected audio clock
- a new public TTS asset lacks a realized current-project `voice_resolution`, or a requested clone silently falls back to a public voice
- handoff or direct delivery is attempted while a direct-production or required new TTS line lacks a successful, probed `audio_produce` result; search or audition preview does not satisfy the line
- `measured_vo_first` lacks produced/probed VO before visual timing is finalized
- a frame-chain route lacks generated nodes, output paths, or ordered unit bindings
- any required generated unit has no successful output, probe, or usable QA result
- a selected generated image/video revision lacks a `mode: standard` AFC acceptance receipt, a blocking/insufficient receipt lacks the required targeted `mode: pro` confirmation or repair, or the receipt does not compare all required user constraints and reference asset labels
- an AFC receipt omits the selected output or a claimed comparison reference from `consumed_assets[]`, or the Generation Handoff omits consumed receipt/evidence ids
- a generated video verdict contains a time-bound issue without revision-bound `video_evidence`, valid source timestamps, or required representative-frame evidence
- audio lineage would let Assembly duplicate speech or replace a protected native clock
- Generation Handoff contains unresolved blockers, assumptions/history, or exceeds tool/file limits
- a production generation call starts before the active Script and required Subject artifacts were read to EOF
- Assembly, Motion, or direct user delivery starts before the Generation Handoff was written, read back, parsed, and validated

# User-Facing Behavior

Execute silently and describe progress in creative language. Do not expose internal labels, model/provider names, payloads, split mechanics, sound flags, or prompt internals unless the user asks.
