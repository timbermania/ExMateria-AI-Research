# Dormant Slot Residue

Chan-side residue held by dormant SPU slots in music-muted parity sessions: on `haste_no_music`, slots 0/1/6/7 were allocated to music tracks before the savestate, but no chan-side opcodes ever run on them during the capture (music muted), so PCSX keeps frozen music-engine values in the core pitch fields — pre_pitch_acc_u32 halves at chan+0x80/0x82, word_86 at chan+0x86, pitch_bend at chan+0x88 — constant across all 207 captured cadences, while Godot's inactive LFO-handler emit originally emitted zeros there, producing 828 rows of `pre_pitch_hi` divergence (4 slots × 207 cadences). smd-player now seeds the dormant slots from a `chan_lfo_residue.json` snapshot that also captures those core fields, so the inactive emit emits the frozen PCSX values; dormant slots are never dispatched (chan_word_0 == 0 early-return), so the seed is trace-only with no audio change.

## Points

- **On the PCSX `haste_no_music` capture, SPU slots 0/1/6/7 are dormant — allocated to music tracks before the savestate, music muted so no chan-side opcodes run on them during the capture — and PCSX preserves frozen music-engine residue in the core chan-side pitch fields (pre_pitch_acc_u32 high/low halves at chan+0x80/0x82, word_86 at chan+0x86, pitch_bend at chan+0x88), constant across all 207 captured cadences: ch0 pre_pitch_hi 0x3C00; ch1 pre_pitch_lo 0xEC72 / pre_pitch_hi 0x3BB4; ch6 pre_pitch_hi 0x3B7C with word_86 0xFFE0 (signed −32); ch7 pre_pitch_hi 0x3B9C; word_86 and pitch_bend zero on all other slots.** — `[D·R] 2/3`
  - D: `probe_chan_pitch_state` (BP `0x800174C8`) — `pre_pitch_hi` divergence 828 rows = 4 dormant slots × 207 cadences, `haste_no_music` capture (2026-05-24)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/probes/probe_emit.gd` (`emit_lfo_handler_inactive` emits the seeded residue values for these dormant slots instead of zeros) + `smd-player/workspace/harness/render_effect_sound.gd` (seeds `EffectPlaySound._slot_residue` from `chan_lfo_residue.json`) — validated by the `probe_chan_pitch_state` paired entry in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/DORMANT_SLOT_PROBE_RESIDUE_SEED.md`
- **Godot's dormant-slot LFO-handler emit is residue-seeded rather than zero-based: `diag_chan_lfo_residue_snapshot.lua` captures per-slot `chan_word_0`, `pre_pitch_lo`/`pre_pitch_hi`, `word_86`, `pitch_bend` in addition to the LFO sub-slots (chan+0xE0/0x100/0x120/0x140), `render_effect_sound.gd` loads that `chan_lfo_residue.json` into `EffectPlaySound._slot_residue` keyed by slot_idx, and `emit_lfo_handler_inactive(slot_idx, residue)` (called from `play_sound.gd:1312/1318`) emits the looked-up values — the doc's pre-fix state (sub-slots-only seeding via `mixer.set_voice_lfo_subslot` plus zero-emitting helper) is what produced the 207/207 dormant `pre_pitch_hi` divergence on `haste_no_music`, plus partials on ch1 `pre_pitch_lo` and ch6 `word_86`; no dormant-channel stubs are created, so dormant slots still never receive opcode dispatch.** — `[D·R] 2/3`
  - D: `probe_chan_pitch_state` / `probe_lfo_subslot{0,1,2,3}_state` dormant-slot divergences (207/207 `pre_pitch_hi` each, 828 total; sub-sampling jitter expected ≤ 8 post-fix), `haste_no_music` capture (2026-05-24)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/probes/probe_emit.gd:181` (`emit_lfo_handler_inactive(slot_idx, residue)`; doc §4.4 option B) + `runtime/effect_sound/play_sound.gd:1312,1318` (`_slot_residue.get(slot_idx, {})`) + `smd-player/workspace/diagnostics/diag_chan_lfo_residue_snapshot.lua` (core-field capture) — validated by the `probe_lfo_handler_entry` / `probe_lfo_subslot*_state` / `probe_chan_pitch_state` paired entries in `smd-player/workspace/orchestrator/probe_validation_manifest.py`
  - src: `research/effect_sound/working_documents/DORMANT_SLOT_PROBE_RESIDUE_SEED.md`

## Notes

(empty — user territory)

## Related

- [[Savestate Residue Voice]]
- [[SPU Voice Engine]]
- [[LFO Sub-Slot Period Reset]]
- [[Effect Sound Slot Allocator]]
- [[SFX Index]]
