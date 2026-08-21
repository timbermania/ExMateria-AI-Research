# Cure 4 Audio Parity

State of knowledge for the `cure_4` effect-sound session (the for-each-spawn variant of the cure timeline), as of the 2026-05-13 bit-0x1000-gate experiment. Pair-slot allocation is time-aligned between Godot and PCSX: slot 4 holds pair 0 for the whole trace and slot 2 holds pair 1 then pair 2 after reuse, so the previously claimed "Godot allocates 3 pair slots vs PCSX's 2" divergence was an artifact of counting raw pool occupancy across all time (both sides allocate three times in spirit — PCSX's first two are pre-trace savestate residue). Known residual divergences at that date: voice 18 silent (the silent driver of pair 1's audible never KONs because `chan_92_value` stays 0 in Godot) and a 24-cadence drift on the first keyframe fire (PCSX cadence 96 vs Godot 72; the 2026-05-13 bisection traces it to PCSX's pre-trace sound-track ticker firing — see [[Effect Sound Timing]]). 2026-05-15 update: after the noise-LFO PRNG gate fix, the cure_4_no_music wf_idx=6/7 noise LFOs now match PCSX bit-exact — voice 18/19 pitch_bend value sets are element-for-element identical and the cadence-497 PRNG desync is resolved (see [[Noise LFO PRNG]]). The 2026-05-13 CAUSE_A_PRESTAGE_TIMING doc confirms voice 18's first KEYON sees `walker_flag_word=0x095` (no 0xAC before its first Note), with its first 0x1FF arm only at cad=242. 2026-05-14 resolution: voice 18's wrong-instrument prestage is root-caused and fixed — FFT's slot allocator seeds new slots with `instrument_idx = 0` (resolving to WAVESET inst[1] via the +1 loader rule), and Godot's `_prestage_first_instrument` now mirrors that `allocator_default` when the first Note precedes the first AC, so voice 18's first KEYON is bit-exact on `start_addr`/`loop_addr`/`adsr2`; because voice 18 is a noise voice (its sample registers don't gate the mix), the residual audio divergence now lives in the pitch baseline, not the sample (see [[SPU Voice Engine]]).

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
  - ⚠ SUPERSEDED (2026-08-20) by: the cure_4_no_music cadence-497 noise-LFO PRNG desync root-caused to Godot's never-cleared `slot.lfo_active` flag — after the 2026-05-15 gate fix, voice 18's 23 unique pitch_bend values match PCSX element-for-element (voice 18 no longer divergent/silent)
- **cure_4's first keyframe fire drifts 24 cadences between the two sides: PCSX fires at cadence 96, Godot at cadence 72 — it affects `cadence_index` alignment of stamped events, not raw row counts of independent events (documented in CADENCE_DRIFT_SPAWN_DELAY.md).** — `[D] 1/3`
  - D: cure_4 + PCSX capture cadence alignment (2026-05-13)
  - D: `probe_play_sound_call` @ 0x800125C0, cure_no_music PCSX cadence 200 vs Godot 176 (same 24-cadence drift) (2026-05-13)
  - R: none — no pre-trace warm-up-tick compensation present in smd-player or godot-learning
  - src: `research/effect_sound/working_documents/BIT_0X1000_GATE_NOT_THE_FIX.md`
  - src: `research/effect_sound/working_documents/CADENCE_DRIFT_SPAWN_DELAY.md`

- **The cure_4_no_music cadence-497 noise-LFO PRNG desync root-caused to Godot's `slot.lfo_active` flag (set by the D9 dispatch, never cleared by EndBar / 0xDB / slot reuse) keeping the noise callbacks — and the engine PRNG — advancing on 5 cadences where FFT's `lfo_handler_tick` outer gate (PC 0x800174D0) skips; switching the outer gate to `channel.channel_word_0 != 0` (2026-05-15) fixed it: voice 19's first divergent pitch_bend at call_index 565 became 65239 (== PCSX), voice 18's 23 unique pitch_bend values match PCSX element-for-element, total PRNG advances dropped 317 → 312, and `probe_lfo_swap` stayed 42/42.** — `[S·D·R] 3/3`
  - S: PC `0x800174D0` outer-gate skip (`beq v0, zero, LAB_800175f4`) — disassembly cited in the doc (§8.1)
  - D: post-fix Godot trace vs the prior PCSX snapshot (`run_effect_iteration.py --session cure_4_no_music`; orchestrator could not re-run the PCSX side) (2026-05-15)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd` FFT-aligned outer gate (doc §8.2) + `probe_pitch_inputs` / `probe_lfo_swap` pairs (`smd-player/workspace/orchestrator/probe_validation_manifest.py`)
  - src: `research/effect_sound/working_documents/CAD_497_WF_IDX_6_PRNG_DESYNC.md`

- **On PCSX `cure_4`, voice 18's first KEYON sees `walker_flag_word=0x095` (VOL_LR_RAW+PITCH+ADSR1_HIGH+ADSR2_LOW), not 0x1FF — its cad=0 dispatches (`0xB4 0xC2 0xE0 0xE2` + Note) contain no 0xAC before the first Note — so its first 0x1FF arm only arrives at cad=242, from the AC dispatch with instrument byte 15.** — `[D·R] 2/3`
  - D: `cure_4_no_music` capture, `probe_walker_flag_word_entry` BP @ PC `0x80014660` + `diag_walker_flag_word_writers.jsonl` watchpoints (2026-05-13)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd::_prestage_first_instrument` (AC-before-Note gate — Note-before-AC voices get only the narrow `WALKER_FLAG_ADSR1_HIGH` arm, consumed by `runtime/shared/flush_tick.gd::flush_kon_only_for_slot`) — validated by `smd-player/workspace/regression/verify_all.sh` cure_4 cluster-probe pairs (`probe_adsr1_high_register`)
  - src: `research/effect_sound/working_documents/CAUSE_A_PRESTAGE_TIMING_ISSUE.md`
- **Godot's `_prestage_first_instrument` now mirrors FFT's slot-allocator default: when no Instrument opcode precedes the first Note it stages WAVESET idx = 1 (source `allocator_default`) instead of the bytecode's first AC operand + 1 — post-fix, voice 18's first KEYON matches PCSX bit-exact on the instrument fields (`start_addr` 26064 → 4112, `loop_addr` 24592 → 48, `adsr2` 24518), and cure_no_music's two prestage rows both still take the `ac_before_note` path, so its behaviour is unchanged.** — `[S·D·R] 3/3`
  - S: `Hyp_instrument_data_loader` at PC `0x80017078`–`0x80017088` (`scus_disassembly.txt`) — the default seed (instrument_idx 0 → inst[1]) the new branch mirrors
  - D: post-fix render `--feds=cure_4_no_music.bin` (1680 pulses, entity_acc_cad0=0x3400) — voice 18 first-KEYON table vs PCSX + cure_no_music regression check (2026-05-14)
  - R: `smd-player/addons/exmateria_sound/runtime/effect_sound/play_sound.gd::_prestage_first_instrument` (branch table `ac_before_note` / `allocator_default` / early-return) + `prestage_first_instrument` trace event and `_probe_prestage_first_instrument_count` counter — validated by the post-fix cure_4_no_music render
  - src: `research/effect_sound/working_documents/CURE_4_CH2_DEFAULT_INSTRUMENT_INVESTIGATION.md`
- **On PCSX `cure_4`, voice 18's first-KEYON `adsr1` is 0x32FF: the high byte 0x32 is committed by Ch 2's `C2_decay(0x32)` opcode dispatching before the first Note (low byte 0xFF is inst[1]'s decay/sustain-rate byte) — at the 2026-05-14 capture Godot's `_key_on_voice` masked `adsr1 & 0x80FF` for `force_envelope_open` voices (attack-rate byte zeroed, 0x00FF), the source of the high-byte divergence.** — `[D·R] 2/3`
  - D: cure_4_no_music PCSX `spu_voice_events.jsonl` first-KEYON row (voice 18, sample 18630) + Ch 2 bytecode dump showing `C2_decay(0x32)` before the first Note (2026-05-14)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/flush_tick.gd::_key_on_voice` (the `force_envelope_open` `adsr1 & 0x80FF` mask; Godot adsr1 0x00FF vs PCSX 0x32FF at capture) — the mask has since been removed from the code, which now passes `slot.adsr1` through unmodified
  - src: `research/effect_sound/working_documents/CURE_4_CH2_DEFAULT_INSTRUMENT_INVESTIGATION.md`
## Notes

(empty — user territory)

## Related

- [[Noise LFO PRNG]]
- [[Effect Sound Audio Divergence]]
- [[FEDS Sound Definition Format]]
- [[KON KOFF Mask Dispatch]]
- [[SPU Voice Engine]]
- [[SFX Index]]
- [[Effect Sound Timing]]
