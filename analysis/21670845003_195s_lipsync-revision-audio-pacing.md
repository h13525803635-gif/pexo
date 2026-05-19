# Case 21670845003 — Silent Hearts Lipsync MV

**Project**: Performance Emocionante: Cantora no Estúdio Cinematográfico com Lipsync  
**User ID**: 23697417113  
**Date**: 2026-05-19  
**Duration**: ~2h20m (38 Langfuse traces)  
**QA tags**: Revision made it worse · Audio issue · Bad pacing  

---

## Session overview

| Phase | Time (UTC) | User input | Agent action | Deliverable |
|-------|------------|------------|--------------|-------------|
| Kickoff | 03:52 | Character + cinematic performance prompt | 3× `video_generate` + BGM | `silent_hearts_performance.mp4` |
| Lipsync start | 04:03–04:06 | Has music ready; upload MP3 | Plan 15s segments; gen seg1 | Preview seg1 |
| QA audio | 05:02 | Comment: duplicate audio, remove advanced track | Remount seg1, gen seg2–3 | `preview_lipsync_45s` |
| QA sync | 05:16 | Video slower than music | Speed/sync investigation | — |
| Audio strategy | 05:18 | Use co-generated audio, drop original BGM | Batch lipsync generation | `preview_lipsync_45s_v2` |
| Full mount V1 | 05:34 | “Pode montar” | Assemble 16 clips (with dupes) | `silent_hearts_full_clip.mp4` |
| Revision V2 | 05:40 | Disjointed; wrong “let’s go” at 3:20 | Remove seg15, crossfade; **duplicate filler segs** | `silent_hearts_full_v2.mp4` (195s) |
| Lyrics fix | 05:44 | Full song lyrics | TTS + `v2_lip_seg1–10` — **never assembled** | Still shows `full_v2` |
| Revision V3 | 05:59 | Remove music; silent lip expression only | Regenerate `v3_seg1–10` (sound off) | **No `full_v3` delivered** |
| Credits | 06:07–06:11 | Too many gens; credits exhausted | Propose cheap assembly with seg1–14 + TTS | Incomplete |

---

## Problem 1 — Revision made it worse (P1)

### Phenomenon

Each user-driven revision degraded quality: V2 still had repeated segments and wrong embedded vocals; V3 burned credits on full silent regen without a new final.

### Root cause

1. **V1→V2 patched symptoms, not structure**  
   User: clip feels like “vários pedaços juntados”; wrong “let’s go” at 3:20 (trace-19).  
   Agent removed seg15 and added 800ms crossfades → `silent_hearts_full_v2`.  
   But **5 missing segments (4, 6, 8, 11, 16) were filled by copying neighbors**:
   - Seg 4 → reuse seg5 (×2 in timeline)
   - Seg 6, 8 → reuse seg7 (×2 each)
   - Seg 11 → reuse seg10 (×2)

2. **V2 lyrics/TTS never shipped**  
   Trace-21: user pasted full lyrics → `silent_hearts_vocals` TTS + 10× `v2_lip_seg*`.  
   **Zero** `execute_edit_video` calls used `v2_lip` clips. User still saw `full_v2` with old co-generated audio.

3. **V3 misread “sem audio”**  
   Trace-25: user wanted no music track, only visual lip expression to lyrics.  
   Agent launched 10× `v3_seg*` regeneration (`sound` off) instead of **muting/stripping** on existing timeline.  
   No `silent_hearts_full_v3.mp4` in any trace; credits exhausted at trace-37.

### Attribution

| Layer | Issue |
|-------|--------|
| Agent | Chose duplicate-fill over generating missing segments; regen instead of post mute |
| Skill gap | No rule blocking “neighbor duplicate” assembly; no revision gate requiring new edit spec |

### Recommendations

| Priority | Skill / rule | Change |
|----------|--------------|--------|
| P0 | `assembly-skill` | **FORBID** using the same clip file twice in one timeline unless user explicitly requests repeat |
| P0 | `modification-skill` | Revision workflow: (1) diff timeline, (2) user confirm, (3) render — never re-show old final while new assets exist |
| P1 | `modification-skill` | “Remove music” → `audio_tracks: []` + strip segment audio; do **not** default to bulk `video_generate` |
| P1 | `generation-skill` | If segments missing, generate **only** missing indices before remount |

---

## Problem 2 — Audio issue (P1)

### Phenomenon

Duplicate audio; lip sync behind music; co-generated song ≠ user’s track; V3 path would be fully silent.

### Root cause

1. **Duplicate audio (trace-9)**  
   User annotation: `AUDIO DUPLICADO. EXCLUIR O AUDIO ADIANTADO.`  
   Co-generated audio inside Seedance clips + user MP3 `Waiting_On_To_Stay_ok.mp3` layered in edit.

2. **Sync drift (trace-11)**  
   User: `O video está mais lento que a musica`  
   Fixed 15s video slots not aligned to music BPM/lyric phrases; no per-clip speed adjustment.

3. **Wrong music source (trace-13/15)**  
   User chose co-generated embedded audio for perfect lips — but vocals/BGM no longer match uploaded song.

4. **V2 has no master audio track**  
   `silent_hearts_full_v2` edit spec: **video track only**, 13 clips each with embedded co-gen audio stitched — no user MP3, no TTS bed.

5. **V3 extreme**  
   Silent regen removes all audio — worse than V2 for any music-video intent.

### Attribution

| Layer | Issue |
|-------|--------|
| Agent | Skipped assembly-skill “probe segment audio first”; stacked tracks |
| Skill gap | Lipsync projects lack mandatory single master (`user_mp3` or `tts_vo`) |

### Recommendations

| Priority | Skill / rule | Change |
|----------|--------------|--------|
| P0 | `generation-skill` | Lipsync: default `sound=off` on `video_generate` |
| P0 | `assembly-skill` | Mandatory `audio_tracks[0]` = user MP3 or TTS; mute embedded speech in clips |
| P1 | `assembly-skill` | Pre-mix checklist: single dialogue/music bus; detect dual active audio |
| P1 | `generation-skill` | TTS path: one VO file → timecoded slice map → per-segment reference audio |

---

## Problem 3 — Bad pacing (P1)

### Phenomenon

Choppy, repetitive feel; ~195s output vs ~4min song; visible looped segments.

### Root cause

1. **Fixed 15s grid** — every clip `out_ms: 15000` regardless of lyric line length.  
2. **Missing + duplicate segments** — timeline `1,2,3,5,5,7,7,9,10,10,12,13,14` (~195s).  
3. **Premature assembly (trace-17)** — mounted with 11/16 segments ready; duplicated neighbors to “complete” clip.  
4. **Crossfade ≠ pacing** — 800ms crossfade cannot fix repeated 30s of identical footage.  
5. **No speed correction** after user reported slow video vs music.

### `silent_hearts_full_v2` timeline (from trace-19 edit spec)

| # | Clip | Note |
|---|------|------|
| 1–3 | seg1, seg2, seg3 | OK |
| 4–5 | seg5, seg5 | **duplicate** (stand-in for missing seg4) |
| 6–7 | seg7, seg7 | **duplicate** (stand-in for seg6, seg8) |
| 8 | seg9 | OK |
| 9–10 | seg10, seg10 | **duplicate** (stand-in for seg11) |
| 11–13 | seg12, seg13, seg14 | OK; seg15 removed (wrong “let it go” lyrics) |

Output: **195s**, 720×1280, 24fps — no separate `audio_tracks` in spec.

### Recommendations

| Priority | Skill / rule | Change |
|----------|--------------|--------|
| P0 | `creative-skill` / `script-skill` | Build segment map from **lyric timestamps** on user MP3 before generation |
| P0 | `assembly-skill` | Validator: no duplicate `source.file`; contiguous segment indices or documented gaps |
| P1 | `generation-skill` | Long MV: 30–45s preview + locked segment map before batch gen |
| P1 | `assembly-skill` | Support per-clip `speed` to match music after user sync complaint |

---

## Recommended recovery (if redoing this project)

1. Use user MP3 as **single master**; generate only missing lipsync segments with `sound=off`.  
2. Rebuild timeline from lyric timecodes — no duplicate clips.  
3. Mount with explicit `audio_tracks: [{ user_mp3 }]`, all segment embedded audio muted.  
4. For “expression only, no music”: export mute from corrected timeline — do not regen `v3_seg*`.  
5. Integrate `v2_lip` + TTS only if replacing co-gen vocals entirely, then **new** `full_v3` edit.

---

## Artifacts

| Path | Description |
|------|-------------|
| `analysis/langfuse-data/cases/21670845003/trace-*.json` | 38 Langfuse traces |
| `analysis/langfuse-data/cases/21670845003/assets.json` | Extracted asset metadata |
| `analysis/langfuse-data/cases/21670845003/media/` | Downloaded samples |
| `analysis/langfuse-data/cases/21670845003/qa-report.html` | Visual QA dashboard |

---

## Skill change summary

| File | Action |
|------|--------|
| `assembly-skill/SKILL.md` | Ban duplicate clip fill; enforce audio master + segment mute |
| `generation-skill/SKILL.md` | Lipsync: sound off; lyric-based segment map before batch |
| `modification-skill/SKILL.md` | Revision diff + confirm; “remove music” = mute not regen |
