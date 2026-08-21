---
name: motion-skill
description: Compose and render deterministic narrative-driven Motion videos in HTML from applicable locked upstream artifacts. Translate narrative intent into visual behavior, state, timing, material response, consequence, continuity, and final proof. Own HTML, packaging, motion, mix, and final QA; never regenerate or substitute primary media.
compatibility: analyze_file_content, audio_produce, cutout_image, edit_file, lint_composition, list_fonts, media_probe, media_process, music_generate, query_render, render_frame, show_final_video, submit_render, write_file
metadata:
  version: ""
---

## Scope

Motion writes `composition.html` and renders the final mp4. On a Generation-backed route, it runs only for `output_route.mode: motion_composition`. It packages locked footage, authors exact DOM-native material, and animates elements. Honor Script's `visual_stack`; never replace locked primary media.

When a composition contains music, song timestamps, BGM, ambience, or SFX, read `references/music-audio-production-strategy.md` before authoring audio DOM or final mix metadata. Motion consumes the locked audio assets and sentence-level timestamp data; it never regenerates music or changes the song segment map.

## Generation Route Intake Gate

Before the existing workflow begins, inspect every present Generation Handoff to EOF and parse its immutable `output_route`. This gate runs before the Visual Map branch and before any `write_file`, `edit_file`, lint, render, or delivery call.

Motion does not design media-processing steps or invent Motion work. Script is the authority for required visual/Motion intent; Generation may only carry the still-required items forward after inspecting its actual selected outputs. Motion verifies that carried intent against the Script Handoff before it acts.

- **`output_route.mode: direct_delivery`** - refuse Motion work and leave the exact selected revision unchanged for the active workflow's delivery step. Do not write/modify `composition.html`, submit a render, or package the generated asset.
- **`output_route.mode: media_processing`** - refuse Motion work and require Assembly to own it. Deterministic media/file processing must not be converted into an HTML composition.
- **`output_route.mode: motion_composition`** - enter the workflow only when `motion_requirements` is non-empty and agrees with Script. If upstream declares preparatory media work, require its completed, locked Motion inputs in the Assembly Handoff before composition.
- **Missing, unknown, or contradictory `output_route`** - block composition. Do not infer a route from `packaging_route`, `visual_map_required`, file type, probe success, or the availability of render tools.

If no Generation Handoff is present, the existing Script/Assembly-only Motion routes remain valid. A later request to revise an already delivered video belongs to Modification, not to a prior Generation route.

## Narrative Motion Core

Before HTML, translate each beat through:

`narrative function -> visual verb -> subject state -> spatial staging -> time curve -> frontend medium -> material response -> consequence -> transition bridge`

Choose one dominant visual verb per beat: `reveal`, `accumulate`, `connect`, `converge`, `propagate`, `collide`, `transform`, `traverse`, `focus`, or `resolve`. A beat is not ready when it only says "make it premium", "add energy", or names an effect preset.

For every beat, define `cause`, `state`, `response`, and `consequence`. Large movement must change a subject, path, material, or hierarchy. Use DOM/CSS for copy and stable planes, SVG for authored paths and masks, Canvas for seeded procedural behavior, WebGL for depth/material/deformation, and media elements for real subjects.

When no source website or locked primary media is required, use `evidence_type: authored` and record the design decision plus behavior rule; never invent source evidence. Use one paused timeline, preserve subject/path/material identity across transitions, and prove trigger, peak, consequence, settle, narrative read, and final hold. Read `references/narrative-to-frontend.md` for the beat worksheet.

## Hard gates

1. **Before `show_final_video`, probe declared audio.** If the composition declares audio, run `media_probe(mode=audio)` on the render; require an audio stream and duration within +/-1 s of the plan. If no audio is declared, get user-confirmed silent intent.
2. **Deterministic render.** No wall-clock callbacks or DOM-mutation callbacks. Use GSAP timeline ops only.
3. **Don't replay source video to pad VO.** Do not reuse one source across semantic scenes or adjacent beat windows unless explicitly `loopable_visual`. Use distinct clips, real offset trims, or an intentional still/card beat.
4. **Measured VO-first direct path intake.** Without an assembly handoff, require measured VO, probed assets, coverage, locked visual segments, and a timeline ledger before HTML. Every group must be `coverage_status: covers`.
5. **No long-form true-whiteboard DOM mimicry.** For true whiteboard footage >15s or >=2 beats, block pure `hf_dom_only` unless a documented prototype exception exists; route to the declared AIGC whiteboard path.
6. **Confirmed URL intake.** Read the confirmed Website Asset Brief, matching Script ledger, Subject bindings, and used-stage ledgers. Raw candidates and reselection are prohibited.
7. **Honor visual intent.** Non-information motion needs real action/spatial change, not still pan/zoom or packaging. Missing primary motion routes upstream.
8. **Preserve framing.** Follow `aspect_treatment`; never stretch or crop out must-show content.
9. **Visual continuity gate.** When strict continuity is declared, require locked anchors and a passing final `visual_continuity_qc`; failures route upstream.

## Technical floors

Use frame-fraction geometry and video-scale typography. Animated centering uses `top/left: 50%` plus GSAP `xPercent/yPercent: -50`, not CSS translate centering. Lock one visual style and the `DENSITY/VARIANCE/MOTION` dials before HTML. Use explicit transition z-index, keep the caption band exclusive, and route long or multi-beat hand-drawn footage to AIGC keyframes. The triggered references below own exact values and exceptions.

## References

### Must read
- `references/afc-multimodal-policy.md` - AFC routing and receipts
- `references/tech-html.md` - HTML contract (this file wins if it disagrees with SKILL.md)
- `references/tech-data-attributes.md` - `data-*` attribute catalog: authorable attributes vs runtime-emitted ones that must never be hand-written

### Audio production: use assembly-skill or generation Phase 0 - not motion
VO/TTS **production**, BGM generation, mix **planning**, audio coverage strategy, and cue sheets are owned upstream - **`assembly-skill`** on the standard path, **`generation-skill` Phase 0** when `vo_planning_mode: measured_vo_first` and assembly is skipped. Motion only owns the HTML-side audio DOM: `<audio>` / `<video data-has-audio>` shape and `data-volume` / `data-audio-role` / `data-loop` / `data-target-lufs` / `data-normalize-lufs` attributes. **Never call the audio-production tool here.** See `references/design-audio-and-assembly.md`.

### Upstream intake (before step 0 - read to EOF when paths are present)

| Path | When | Motion consumes |
|---|---|---|
| **Assembly** | Standard path: `artifact_context.artifacts.assembly_handoff.path` | Locked timeline/material package and authoritative `_sources` paths |
| **Script** | Assembly `_sources.script_handoff.path`; direct `artifact_context.artifacts.script_handoff.path` only on bypass | `packaging_route.visual_map_required`, `footage_clip_ids`, `shot_structure`, `text_layer_declaration`, `packaging_layout_manifest` |
| **Subject** | Assembly `_sources.subject_manifest.path`; direct context path only on bypass when `subject_required: true` | `continuity_anchors`, exact asset labels/revisions, sequence bindings |
| **Generation** | Assembly `_sources.generation_handoff.path`; direct context path only on bypass when generated clips are used | `visual_continuity`, per-unit anchor payload labels, selected output revisions and AFC evidence |
| **Assembly-bypass / generation direct** | No Assembly artifact; `vo_planning_mode: measured_vo_first`; generation finished Phase 0 + Phase 1 | Generation Handoff `measured_vo_timeline`, probed `vo_assets[]`, `measured_visual_coverage_plan`, `locked_visual_segments`, `planned_timeline_ledger`, `audio_intent`, and `voice_resolution`; Script Handoff remains exact-copy authority. |
| **Legacy estimate-first bypass** | No assembly; no `measured_vo_timeline` | Probed `vo_assets[]` + planned timeline from generation handoff; same placement rules, no char/sec estimates for `data-duration`. |

On the standard path, read Assembly first and then read every present `_sources` path to EOF. Do not require duplicate direct Subject or Generation paths in `artifact_context`; those are bypass-only inputs.

When a Generation Handoff is present, `output_route.mode` selects finalization ownership. `packaging_route.mode` only selects the Visual Map branch after Motion has already been authorized; it never selects direct delivery or media processing.

If `audio_intent` is not `explicit_silent` and probed VO/BGM assets are missing, **BLOCK** composition HTML. Route back to **generation-skill** (measured path) or **assembly-skill** (standard path) - do not produce audio in motion.

For `measured_vo_first` direct path, `render_frame` success is not coverage proof. Motion must treat the upstream `measured_visual_coverage_plan` as the proof source before writing HTML.

### Read when the trigger fires (checked at step 2, before writing HTML)

Each row's trigger is observable in the handoff or the task - when it fires, read the file (at the Section anchor if given) **before** writing the HTML that depends on it, not after QC fails.

| Trigger (observable) | Read |
|---|---|
| Any overlay (title / subtitle / card / logo) will exist | `references/design-house-style.md` Section Packaging zones & safe area - authoritative for overlay placement and Overlay QC |
| `packaging_route.visual_map_required: true` | `references/design-visual-map.md` - Pass A measurement, schema, receipt, and map gate |
| `subject_required: true` + strict visual continuity | Upstream continuity anchors and AFC evidence, then `references/visual-continuity-qc.md` |
| Composition authors any designed look (i.e. not a bare pass-through) | `references/design-visual-styles.md` - pick the style id for the design receipt |
| >= 2 scenes | `references/design-beat-planning.md` - dials + beat plan feed the design receipt |
| Any element animates | `references/design-motion.md` Section Easing convention, Section Duration scale |
| >= 2 scenes meet at a boundary | `references/design-transitions.md` |
| `text_layer_declaration` includes subtitles, or VO assets present | `references/design-captions.md` |
| Any text element at all | `references/design-typography.md`, heading "Video-medium type sizing" (CJK content -> heading "CJK-specific notes") |
| `<audio>` / `<video data-has-audio>` in the DOM | `references/design-audio-and-assembly.md` (motion's slice only) |
| Camera move, cursor demo, or 3D entrance requested | `references/design-camera-cursor.md` |
| Cutout assets or brand logo in the manifest | `references/design-cutout-and-brand.md` |
| Charts, KPI numbers, counters, or stats | `references/design-data-viz.md` |

## Workflow

Read upstream handoffs first, measure when Script requires it, then design, lint, verify, and render.

0. **Read the resolved Script Handoff and lock the map gate.** Read it to EOF before any composition work. Take `packaging_route.visual_map_required` as the only authority for Pass A - **do not self-grant skip** (`no_footage_overlay_zone`, `no_footage_overlay_only`, and similar invented statuses are invalid). Branch:

   - **`visual_map_required: false` + `mode: none`** -> annotate `<!-- visual-map: none | pass_a=no_overlays_planned | pass_b=skipped_trivial -->`. Motion is active for a non-packaging task such as transitions or Motion-created visuals; this branch only skips Visual Map work. Skip to step 2.
   - **`visual_map_required: false` + `mode: motion_scene`** -> Motion creates the full DOM/HTML visual with no video clips to measure. Annotate `pass_a=no_footage`; Pass B still runs when overlays exist. Skip to step 2 (no map file).
   - **`visual_map_required: true` + `mode: overlay_on_video`** -> Pass A is **mandatory** before any overlay or packaging HTML. Proceed to step 1.

0a. **True-whiteboard route check.** Before accepting `hf_dom_only`, scan the handoff and current user brief for true whiteboard triggers: hand-draw, marker draws, real-time drawing, visible hand, eraser/wipe, paper/board texture, or a hand-drawn explainer longer than 15s / >=2 beats.

   - If `production_strategy.visual_route` is `aigc_keyframe_whiteboard` or `hybrid_aigc_whiteboard_plus_hf_text`, require locked generated visual segments before composition. Motion places those clips and owns exact text/brand/subtitle overlays only.
   - If no visual route is declared and triggers are present, BLOCK and route back to script-skill. Do not self-upgrade to `hf_dom_only`.
   - If the route is explicitly `hf_dom_only`, require a short-DOM/prototype reason and verify total DOM visual coverage: every scene window has a live timeline event inside its clip window, no duplicate old `<script>` block, and no blank board tail unless declared as an intentional end-card hold.

0b. **Measured VO-first direct-path gate.** When `vo_planning_mode: measured_vo_first` and no assembly package is present, verify the generation handoff before writing any HTML:

   - `measured_vo_timeline` exists.
   - probed `vo_assets[]` exists.
   - `measured_visual_coverage_plan` exists.
   - every group has `coverage_status: covers`.
   - locked/probed visual segments and the planned timeline ledger cover those group windows.
   - any looped visual segment is explicitly declared as `source_strategy: intentional_loop` with an intentional design reason in the coverage plan.

0c. **Confirmed URL binding gate.** Proposal ids must match. Script's ledger is the expected set; bind only confirmed Motion-owned Subject rows or footage with stage lineage. Every URL file must trace `capture_ref -> source_asset_id -> asset_use_id -> Subject asset_label -> landed path`. Missing lineage or cross-owner substitution blocks HTML.

0d. **Visual-stack gate.** Read each sequence's stack before authoring. Exact-information and packaging layers may overlay any route but cannot replace missing primary content or motion.

0e. **Adjacent-cut continuity preflight.** When strict continuity is declared, read anchors, selected revisions, and upstream AFC evidence before writing HTML. Compare both sides of every planned adjacent cut with the approved anchor; missing or failed upstream evidence blocks composition. The final render still requires a new receipt under `references/visual-continuity-qc.md`.

1. **Visual map (Pass A) - only when `visual_map_required: true`.** Build `visual_map.json` per `references/design-visual-map.md` and `references/afc-multimodal-policy.md`. Inspect every `footage_clip_ids` entry for time ranges, occupancy, safe zones, motion/contrast risk, and timestamped evidence. `clips_measured` and `clips_total` must equal the list length; trace each field to the current revision and source time. Priors and render PNGs never replace source inspection.

2. **Catalog binding gate.** Read the Asset Catalog and Binding Manifest before using a catalog `workspace_path`. Path, role, and `motion_overlay` / `motion_ui` / `brand_overlay` lane must agree. Reject uncertain or cross-class bindings; UI evidence cannot replace a person or product hero. This is a manifest check, not new multimodal analysis.

3. Write `composition.html` with `write_file` in <=400-line calls and extend it with `edit_file`. Here `edit_file` is only an internal file-authoring tool for `composition.html`, never an output route or a video-editing category. Follow `references/design-house-style.md`. Copy required map measurements, put the map receipt first, and put `<!-- design: style=<id> dials=<D>/<V>/<M> beats=<n> -->` second. Use legal library values, keep each packaging unit self-contained, and let measurements win over the layout manifest.

   Under `measured_vo_first`, HTML may only place the already-covered plan. Do not split one semantic source clip into repeated `v2a` / `v2b` / `v2c` windows or add `data-loop="true"` to compensate for a measured VO group that is longer than its visual source. If a loop is present, it must correspond to an upstream `source_strategy: intentional_loop` row.

4. `lint_composition` - always run before rendering. Fix every error; intentional warnings get `<!-- lint-override: <code> reason=<one line> -->`.

5. **Overlay QC (Pass B).** Verify against the map, or check layout/legibility for `packaging_route.mode: motion_scene`. Skip only `mode: none` with no overlays. Otherwise use only the questions in `references/design-house-style.md` Section "Overlay QC" and keep Pass B pending until every verdict is recorded.

   1. Sample timestamps: overlay window boundaries, subtitle+overlay overlaps, closing frame (4-6 typical).
   2. `render_frame` at each timestamp.
   3. Analyze each frame with those questions using `scope: visual`, `mode: fast`; record evidence, consumed assets, and verdict links. Escalate only blocking/insufficient timestamps once with targeted `mode: pro` analysis.
   4. On fail: fix composition, update map if measurement was wrong, re-lint.
   5. All pass -> `pass_b.status: pass` in map + update header annotation.

6. **Hold-length check.** Closing/persistent overlays (`cta-card`, `end-card`, `selling-point`, final `lower-third`, held title) with `data-duration` over **8s** -> split beats or `<!-- cta-hold-override: reason=<one line> -->`.

6a. **Final URL reconciliation.** Emit one `confirmed_asset_consumption_ledger` row per required use. Generation needs payload lineage; Assembly needs locked media/timeline evidence; Motion needs a DOM/media binding plus sampled-frame proof. Only `consumed` or Website-Brief-backed `user_approved_omission` may pass. Missing, substituted, repurposed, or pending rows block render.

7. `submit_render` -> `query_render`. Required-map routes need `pass_a=complete`, a measured count matching `footage_clip_ids`, and `pass_b=pass`; other routes need `pass_b=pass` or `skipped_trivial`. Require a legal style id. Invented map or style tokens violate protocol.

8. Post-render, probe duration/audio and required subject/action. Under strict continuity, inspect every final boundary and write/read `<project_label>__visual_continuity_qc__RNN.json` per `references/visual-continuity-qc.md`. Only a final-render receipt with verdict `pass` may reach `show_final_video`.

**Escape annotations.** Record lint-missed collisions as top-level `<!-- lint-escape: reason=<why> -->` comments.

## Source-to-Motion

For website films, read `references/narrative-to-frontend.md`, `references/source-to-motion.md`, `references/behavioral-choreography.md`, `references/media-provenance.md`, `references/stage-integration.md`, `references/behavior-proof.md`, and `references/visual-qa.md`. Preserve source-backed state, response laws, identity, consequence, and proof time.
