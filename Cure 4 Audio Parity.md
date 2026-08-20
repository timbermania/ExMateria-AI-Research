# Cure 4 Audio Parity

State of knowledge for the `cure_4` effect-sound session (the for-each-spawn variant of the cure timeline), as of the 2026-05-13 bit-0x1000-gate experiment. Pair-slot allocation is time-aligned between Godot and PCSX: slot 4 holds pair 0 for the whole trace and slot 2 holds pair 1 then pair 2 after reuse, so the previously claimed "Godot allocates 3 pair slots vs PCSX's 2" divergence was an artifact of counting raw pool occupancy across all time (both sides allocate three times in spirit — PCSX's first two are pre-trace savestate residue). Known residual divergences at that date: voice 18 silent (the silent driver of pair 1's audible never KONs because `chan_92_value` stays 0 in Godot) and a 24-cadence drift on the first keyframe fire (PCSX cadence 96 vs Godot 72; the 2026-05-13 bisection traces it to PCSX's pre-trace sound-track ticker firing — see [[Effect Sound Timing]]).

## Points

- **FFT's play_sound entry path tests `DAT_80032A54 & 0x1000` at PC 0x800125C0.** — `[S] 1/3`
  - S: PC `0x800125C0` + `DAT_80032A54` (working doc; gate sequence `lhu DAT_80032A54 / andi v0, 0x1000 / beq → LAB_800125F8` recorded in the header of `smd-player/workspace/diagnostics/diag_play_sound_gate_state.lua`)
  - R: none — the `0x1000` gate not present in smd-player or godot-learning (probed; only the 0x800125C0 entry counter for `probe_play_sound_call` remains in `play_sound.gd`)
  - src: `research/effect_sound/working_documents/BIT_0X1000_GATE_NOT_THE_FIX.md`
- **cure_4 pair-slot occupancy is time-aligned between Godot and PCSX — slot 4 = pair 0 the whole trace, slot 2 = pair 1 then pair 2 after reuse; Godot's `pool.find_free_pair_slot()` consults `slot.active_word & 1` (commit `3cf6c165`), so at frame 71 it returns slot 2, exactly what PCSX's allocator does at cadence 494; the previously claimed "Godot allocates 3 pair-slots vs PCSX's 2" divergence is an artifact of counting raw all-time pool occupancy including the reused slot (PCSX's allocator also runs three times in spirit: two pre-trace savestate-residue allocations + one call at cadence 494, so `probe_play_sound_alloc` fires once on PCSX).** — `[S·D·R] 3/3`
  - S: FFT slot allocator `play_sound_callee_12d40` @ `0x80012D40`, busy-mask pass 2 @ PC `0x80012E04` (`scus_decompilation.c`; mirrored in `pool.gd`)
  - D: cure_4 slot-occupancy trace + `probe_play_sound_alloc` (2026-05-13)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/pool.gd::find_free_pair_slot` (busy = `_slot_free == 0` AND `active_word & 1`), consumed by `godot-learning/src/audio/EffectSfxEngine.gd` — validated by the `probe_play_sound_alloc` BP pair at `0x80012D40`/`0x80012E74` (`smd-player/workspace/probes/probe_play_sound_alloc.lua`)
  - src: `research/effect_sound/working_documents/BIT_0X1000_GATE_NOT_THE_FIX.md`
- **cure_4's timeline fires three keyframes — phase1.track0 (channel_index 0, sid 2 → pair 0), phase1.track1 (channel_index 1, sid 3 → pair 1), animate_tick.track0 (channel_index 0, sid 4 → pair 2); phase1.track0 and animate_tick.track0 share `channel_index=0` because animate_tick.track0 is the for-each-spawn equivalent of phase1.track0, re-using the same channel index.** — `[D·R] 2/3`
  - D: cure_4 runtime trace (2026-05-13): pairs 0/1 fire at tick 9, pair 2 at tick 71 on channel 0
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound_controller.gd` (`_build_channels`, `channel_index` read from the sound-tracks JSON entries) + `runtime/effect_json_loader.gd` (`sound.json`); validated by the cure_4 probe suite (`smd-player/workspace/orchestrator/probe_validation_manifest.py`)
  - src: `research/effect_sound/working_documents/BIT_0X1000_GATE_NOT_THE_FIX.md`
- **cure_4 voice 18 remains silent in Godot (cos_dist 1.0) as of the 2026-05-13 probe state: the silent driver of pair 1's audible doesn't KON because `chan_92_value` stays 0 in Godot — a documented separate issue in the `play_sound.gd` chan_92 logic.** — `[D·R] 2/3`
  - D: voice-18 audio diff in the cure_4 probe run (2026-05-13), cos_dist 1.0
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd` (chan_92 logic; comment references `CURE_4_V18_WALKER_MISS_AT_CAD_495_INVESTIGATION.md`) + `smd-player/workspace/diagnostics/diag_chan_92_writers.lua`
  - src: `research/effect_sound/working_documents/BIT_0X1000_GATE_NOT_THE_FIX.md`
- **cure_4's first keyframe fire drifts 24 cadences between the two sides: PCSX fires at cadence 96, Godot at cadence 72 — it affects `cadence_index` alignment of stamped events, not raw row counts of independent events (documented in CADENCE_DRIFT_SPAWN_DELAY.md).** — `[D] 1/3`
  - D: cure_4 + PCSX capture cadence alignment (2026-05-13)
  - D: `probe_play_sound_call` @ 0x800125C0, cure_no_music PCSX cadence 200 vs Godot 176 (same 24-cadence drift) (2026-05-13)
  - R: none — no pre-trace warm-up-tick compensation present in smd-player or godot-learning
  - src: `research/effect_sound/working_documents/BIT_0X1000_GATE_NOT_THE_FIX.md`
  - src: `research/effect_sound/working_documents/CADENCE_DRIFT_SPAWN_DELAY.md`

## Notes

(empty — user territory)

## Related

- [[Effect Sound Audio Divergence]]
- [[FEDS Sound Definition Format]]
- [[KON KOFF Mask Dispatch]]
- [[SPU Voice Engine]]
- [[SFX Index]]
- [[Effect Sound Timing]]
