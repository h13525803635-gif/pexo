---
name: script-skill
description: Plan a new video from a locked creative direction or production-ready brief. Own narrative, sequence/shot, visual/audio/text intent, and downstream handoff. Route concrete assets, specific or recurring subjects, and shared environments through subject-asset-skill. Do not produce media here.
compatibility: analyze_file_content, audio_produce, media_probe, media_process, music_generate, read_file, video_generate, write_file
metadata:
  version: v7
---

# Usage

Script-skill is the bridge from locked idea to executable production contract.

Use it when the story direction is ready enough to plan production. Inputs may be a brainstorm creative script, a `brainstorm_artifact_path`, or a direct user brief with concept, duration/aspect, assets, dialogue/VO/music needs, visible text, or shot anchors.

Output is an internal `script_handoff` for subject-asset-skill, generation-skill, assembly-skill, and motion-skill. Do not execute production, render, publishing, or modification work here. This skill plans the contract only.

Core rule: never pass raw brainstorm prose or a loose user brief downstream. Compile the script, shot plan, subject brief, production strategy, continuity plan, and blockers. Script describes the required subjects and scopes; Subject locks the references.

# Read Gates

Read `references/script-handoff-contract.md` completely before finalizing any handoff. It is the canonical core schema and gate reference for this skill. Also read the directly relevant delegated contracts below.

Always read:

- `references/script-visual-routing-contract.md` for packaging, whiteboard, readable text, brand, and subject rules
- `references/script-production-contract.md` for URL asset consumption, production strategy, continuity, and generation contracts
- `references/script-audio-contract.md` when any audio, speech, visible performance, music, SFX, BGM, or explicit silence is present
- `references/music-audio-production-strategy.md` when any music, song, BGM, SFX, audio timestamp, or audio-first/video-first decision is present

For any URL-backed video project, read these active artifacts to EOF before planning:

- `artifact_context.artifacts.creative_brief.path`
- `artifact_context.artifacts.capture_context.path`
- `artifact_context.artifacts.website_asset_brief.path`

Require the Website Asset Brief to reference the exact active Creative Brief path. Require source URL and Capture Artifact id to match Capture Context, and require `user_asset_confirmation.status: confirmed` with `confirmed_proposal_id == proposal.proposal_id`. Missing or pending approval routes to Brainstorm.

Always read these before production-strategy or generation-contract decisions:

- Model capability, duration, and reference constraints: `references/video-models-routing.md`. Use them only for feasibility; Generation owns provider, model, mode, and provider-parameter selection.
- Generation prompt, audio ownership, sequence splitting, and BGM/SFX contract: `references/video-generation-execution.md`
- Production validation decisions paired with that contract: `references/video-generation-strategy-genes.md`

Read when relevant:

- Shot language, camera intent, montage mode tags: `references/shot-design-principles.md`
- Rhythm curve, beat density, cut motives, parallel continuity toolkit: `references/editing-rhythm-design.md`
- Frame-chain field names and junction modes: `references/frame-chain-handoff.md`
- Audio mood, SFX, music relationship, mix priority: `references/audio-design-guide.md`
- Visible dialogue and monologue writing: `references/dialogue-monologue-design-kb.md`
- Voiceover copy, voice requirements, measured VO-first planning, visible speech, talking-head, lip-sync, and audio-list clock ownership: `references/voice-strategy-execution.md`
- Measured VO-first paragraph grouping and Phase 0/1 gates: `references/script-audio-contract.md`
- Subject identity, attributes, state variants, recurring subject handling: `references/subject-definition-framework.md`
- Readable text, subtitles, UI, brand, packaging, safe zones, conflict-zone composition: `references/deterministic-visual-payload-guide.md`
- True whiteboard / hand-drawn explainer routing: `references/script-visual-routing-contract.md` Section "True Whiteboard / Hand-Drawn Explainer Routing"

# Contract

Final output must satisfy the `script_handoff` schema in `references/script-handoff-contract.md`, with delegated rules from the three responsibility-specific contracts.

Required contract groups:

- `video_script`
- `sequence_breakdown`
- `shot_breakdown`
- `shot_structure` for generated clips - declared shot facts (shot count, internal cut points, camera movement, subject position) that motion-skill's visual map consumes as its metadata prior; `unknown` for user-uploaded footage
- `subject_element_brief`
- `website_asset_route`, `asset_consumption_plan`, and `confirmed_asset_coverage_ledger` for URL work
- `required_sequence_manifest`
- `production_strategy`
- `duration_budget`
- `audio_intent`, `audio_strategy`, `audio_content.speech`, and `voice_requirements` when speech exists; `vo_script` when `vo_planning_mode: measured_vo_first`; `visible_speech_coverage_plan` when visible speech is requested or implied; `music_lipsync_feasibility_declaration` when a vocal track is planned with visible singing
- `text_layer_declaration` when readable text exists
- `brand_asset_declaration`, `packaging_plan`, and `packaging_layout_manifest` when relevant
- `packaging_route` - always emit; includes `requires_visual_map` when packaging over >=2 footage clips (see contract Section "Packaging Route And Visual Map Gate")
- `consistency_strategy`
- `continuity_pack` and `frame_chain_plan` when activated
- `generation_contracts` and prompt exclusions
- `fallback_strategy`
- `blockers` and `assumptions`

IDs are production keys. Preserve upstream `sequence_id`, `shot_id`, and `subject_id` when present. If an id is inferred, mark it as inferred and explain why.

# Workflow

1. Intake and lock inputs. If upstream provides any handoff artifact path marked `handoff_artifact_read_required: true`, call `read_file` on it until EOF before planning. For URL work read all three Brainstorm artifacts, verify the confirmation/proposal ids, and carry their exact paths. Reconcile active artifacts with the latest user instruction. A new instruction that changes an approved beat/material use routes to Brainstorm for a new confirmed revision; it is not a local Script override.
2. Write the full video script. Lock premise, viewer journey, hook, story beats, spoken script or blocker, packaging intent, and payoff/CTA. For URL work preserve the confirmed simple-script beats as anchors; production detail may expand them but cannot silently replace their intent or approved materials.
3. **VO planning mode** - set `audio_strategy.vo_planning_mode`:
   - **`measured_vo_first`** (`post_vo` + silent B-roll / explainer): follow the measured-VO sections in `references/script-audio-contract.md` and `references/voice-strategy-execution.md`; lock full VO copy, one `voice_requirements` row per perceived speaker, and semantic `paragraph_groups` (`semantic_beat` + `visual_intent`). Preserve target language, required versus preferred constraints, negative requirements, and explicit clone authorization without choosing a provider voice, candidate, or search route. Do not assign sequence ms from estimates. Hand off to generation-skill for voice resolution, TTS, and measurement before final sequence/shot breakdown. The downstream success condition is not merely "VO produced"; generation must derive visual beats from the measured VO and cover every group without default loop/reuse padding.
   - **`estimate_first`** (default - co-gen, `audio_list_speech`, music-first, legacy): plan sequences and shots from duration budget as today.
4. Break into sequences and shots when not deferred by `measured_vo_first`. Preserve user-provided shot order as anchors. Every sequence and shot must state what appears, what changes, what is heard, and what cannot be replaced. For each sequence, compile the compact `visual_stack` from the handoff contract: communication goal first, then primary visual, exact information layers, packaging, motion requirement, and any aspect treatment.
5. Declare production ownership. Cover audio, speech, readable text, brand layers, subjects, assets, production strategy, duration, consistency, continuity, and fallback. Route the three visual layers independently: website footage, AIGC, and Motion may be combined in one beat. Asset type alone never decides the route; UI, logo, charts, and exact text may be overlays on source or generated footage rather than standalone Motion scenes. **Compute `packaging_route`** per contract Section "Packaging Route And Visual Map Gate" - motion-skill treats `requires_visual_map` as authoritative; do not leave the decision to motion.
6. Route execution explicitly. If the plan needs video generation, image generation, TTS/audio production, music generation, cutout production, composition HTML, or final render, name the owning downstream skill and handoff artifact. Script context must not call those production tools directly. For `measured_vo_first`, generation-skill owns Phase 0 TTS/audio production before any video-generation call.
   - Set `handoff_metadata.subject_required: true`, `handoff_metadata.required_next_skill: subject-asset-skill`, and reason codes when the plan has a concrete asset path, specific/recurring identity, persistent state, visible-speaker binding, required reference/cutout, or strict/shared continuity anchor. Otherwise preserve the one-off `description_only` path and route directly to the next production owner.
   - For true whiteboard / hand-drawn explainer briefs, decide whether the visual footage is AIGC keyframe-controlled whiteboard, not just `hf_dom_only`. Long or multi-beat "hand draws on a whiteboard" work must route to generation-skill for keyframe/image-to-video coverage; motion may own only exact text, logo, captions, and final assembly.
7. **Confirmed URL consumption design.** Map every confirmed required `asset_use_id` into `asset_consumption_plan` with its unchanged source id/ref, landed path, approved beat/use, final owner, legal lane, and `usage_commitment: required`. Optional uses need a documented production reason when omitted. Emit one `confirmed_asset_coverage_ledger` row per confirmed required use with `status: planned`. Any non-empty confirmed asset-use set requires Subject; a user-confirmed gap-only proposal may route directly to execution without an empty Subject Manifest. Script may refine execution ownership but cannot substitute another capture asset, change the intended use, or downgrade required to optional.
8. **Inventory-aware brief calibration.** Before finalizing `subject_element_brief`, inspect the persisted Capture Context and confirmed Website Asset Brief. Use only confirmed selected rows with an exact landed path, original verification receipt, and matching role. UI screenshots can satisfy requested UI/website evidence only, never a product/person/environment role. Do not call analysis or capture tools in Script. Every URL asset row must require Subject binding.
9. Write the handoff artifact. Use `write_file` when available, usually `/workspace/script-handoff.md` or `/projects/<project_id>/workspace/script-handoff.md`. Carry its immutable path in `artifact_context.artifacts.script_handoff.path`; URL work also preserves all three Brainstorm artifact paths.

# Hard Gates

Block and route back instead of guessing when any required contract group is missing or contradicted.

- Missing locked creative direction: route to brainstorm-skill.
- Missing upstream artifact read: read it before planning.
- URL present without matching Creative Brief, Capture Context, and confirmed Website Asset Brief paths read to EOF: route to Brainstorm.
- URL proposal pending/revision-requested, source URL/artifact/proposal ids disagree, or any confirmed required asset use is absent/downgraded/replaced/repurposed: do not hand off.
- URL handoff missing `asset_consumption_plan`/coverage ledger, or a non-empty confirmed asset-use set does not route next to `subject-asset-skill`: do not hand off.
- Missing full script, sequence plan, or shot plan: do not hand off - except `measured_vo_first` Phase A (see workflow step 3).
- A sequence lacks a complete `visual_stack`, maps an asset type directly to an owner without a communication reason, or uses `text_to_video` for a specific real subject without a verified faithful reference: do not hand off.
- `subject_motion`, `process_demo`, or `spatial_motion` is requested but the planned primary visual is only a static image placement or pan/zoom: do not hand off.
- `measured_vo_first` without a complete `vo_script.voice_identity_id`, matching `voice_requirements`, and semantic `paragraph_groups`: do not hand off.
- Missing audio ownership for requested or implied speech, BGM, SFX, narration, talking-head, or promo audio: do not hand off.
- Missing text ownership for exact readable copy, subtitles, UI, labels, CTA, prices, brand wordmarks, or factual payloads: do not hand off.
- Missing or contradictory `packaging_route` when packaging or multi-clip footage is planned: do not hand off.
- `requires_visual_map: true` but `footage_clip_ids` empty or count < 2: do not hand off.
- Missing subject brief for recurring people, products, objects, scenes, UI carriers, or state changes: do not hand off.
- A concrete asset path, specific/recurring identity, persistent state, visible-speaker binding, required reference/cutout, or strict/shared continuity anchor requires Subject; if `handoff_metadata.subject_required` or `handoff_metadata.required_next_skill` is missing or points directly to Generation, do not hand off.
- Missing continuity or frame-chain fields when seamless joins, strict continuity, storyboard control, or entry/exit frame control is required: do not hand off.
- True whiteboard / hand-drawn explainer marked as pure `hf_dom_only` when it requires a real hand, marker strokes, wipe gestures, or >15s multi-beat drawing footage: do not hand off. Route visual coverage through AIGC keyframe/video generation and reserve motion for post text/packaging.
- Production tool bypass: do not call video-generation, image-generation, audio-production, music-generation, cutout, composition, render, or delivery tools from Script context. Hand off to the owning skill instead.
- Open blocker affects production feasibility: write the blocker and stop at script handoff.

# User-Facing Response

The user-facing summary should read like a creative script and production outline. Remove internal provider names, model names, route mechanics, API/tool details, sequence ids, and other system trace terms unless the user explicitly asks for implementation detail.

# Out Of Scope

- Developing or choosing the creative concept: brainstorm-skill.
- Asset coverage audit, role labeling, and supplementary reference generation: subject-asset-skill.
- Production generation and media creation: generation-skill.
- HTML/HyperFrames implementation and render QA: motion-skill.
- Material prep, rough cut, and VO production: assembly-skill; timeline assembly and final mix: motion-skill.
- Existing video edits: modification-skill.
- Publishing copy, cover, and social packaging: publishing-skill.
