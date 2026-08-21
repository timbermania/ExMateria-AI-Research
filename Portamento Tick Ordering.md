# Portamento Tick Ordering

FFT's 0xD4 portamento glide per-tick behaviour in `per_channel_tick` (FUN_80015138), and the deactivation-tick ordering that smd-player's dispatcher had to mirror to stay row-exact. While `porta_active` is set, `pre_pitch_acc` steps by `pre_pitch_delta` every tick and `target_counter` decrements; on the tick the counter reaches 0, FFT sets `CHAN1_PITCH_PRESTAGE` (chan_word_1 bit 0x200) in a delay slot BEFORE clearing `porta_active`, so the pitch-staging recompute and PITCH walker fan-out fire on the deactivation tick itself. smd-player originally snapshotted `porta_was_active` after `cadence_body` had already cleared `portamento_active`, missing that one fan-out on `cure_4_no_music` (voice 20, cad 242); arming the prestage from the pre-decrement snapshot inside `cadence_body` closed `probe_walker_flag_word_entry` to 924/924 PAIR (2026-05-14). The recompute itself only runs when `CHAN1_PITCH_PRESTAGE` is set (see [[PSX Pitch Conversion]]).

## Points

- **FFT's portamento tick (per_channel_tick, FUN_80015138) steps `pre_pitch_acc` by `pre_pitch_delta` every tick while `porta_active` is set (chan+0x6 bit 0x1) and decrements `target_counter` (chan+0xa0); on the tick `target_counter` reaches 0, `CHAN1_PITCH_PRESTAGE` (chan_word_1 bit 0x200) is ORed in the delay slot at 0x80015210 BEFORE the `porta_active` clear at 0x80015230, so the pitch-staging recompute and PITCH walker fan-out fire on the deactivation tick itself.** — `[S·D·R] 3/3`
  - S: `FUN_80015138` disasm PC `0x80015200–0x80015244` — porta gate `andi v0,a3,0x1` @ 0x80015200, delay-slot `_ori t0,t0,0x200` @ 0x80015210, `target_counter` `lhu/sh 0xa0(a0)` @ 0x80015214–0x80015220, `porta_active` clear `andi a3,a3,0xfffe` @ 0x80015230, `pre_pitch_acc` step `lw/addu/sw 0x7a(a0)` @ 0x80015234–0x80015244 (doc §2.6; raw a0-relative offsets)
  - D: `cure_4_no_music` capture (2026-05-14) — voice 20 portamento (`0xD4 96 60` at cad 0, target=96 rate=60): PCSX `probe_pitch_staging` +7 steps at cad 231/233/236/238/241 vs Godot stopping at cad 238; the missing cad-241 step corresponds to the missing voice-20 `wfw=0x0004` (PITCH) `probe_walker_flag_word_entry` row at call_index 387 / cad 242 (doc §2.5 bisection)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd:477-496` (`cadence_body`: pre-decrement-snapshot `pre_pitch_acc` step + `CHAN1_PITCH_PRESTAGE`/`WALKER_FLAG_PITCH` arm, in-code comment mirrors PC 0x80015200–0x80015244) — validated by the post-fix `probe_walker_flag_word_entry` 924/924 pair on `cure_4_no_music` (`smd-player/workspace/orchestrator/probe_validation_manifest.py`)
  - src: `research/effect_sound/working_documents/PROBE_WALKER_FLAG_WORD_ENTRY_924_923.md`

- **smd-player's pitch-staging recompute originally snapshotted `porta_was_active` in `tick()` AFTER `cadence_body` had already cleared `portamento_active`, so the deactivation tick never armed `CHAN1_PITCH_PRESTAGE`/`WALKER_FLAG_PITCH` and dropped one PITCH fan-out; setting both inside `cadence_body` from the pre-decrement snapshot `_cb_porta_was_active` closed the `cure_4_no_music` gap — `probe_walker_flag_word_entry` 924/923 → 924/924 PAIR, `probe_pitch_inputs` 803/802 → 803/803, `probe_pitch_staging` 803/802 → 803/803, `probe_pitch_register` 805/803 → 805/804 — with audio cos_dist within run-to-run variance (voice_18 0.375, voice_19 0.135, voice_20 0.487, full_mix 0.224 vs Part-A baseline 0.334–0.370 / 0.107–0.117 / 0.423–0.434 / 0.215–0.235).** — `[D·R] 2/3`
  - D: pre-/post-fix `cure_4_no_music` parity runs (2026-05-14) — probe table doc §2.8; the pre-fix missing row was identified by `diag_walker_entry_diff.py` row diff (voice 20, `wfw=0x0004`, call_index 387 / cad 242); residual post-fix issues (±2 drift on 6 walker rows, `probe_pitch_register` Δ=−1) tracked separately
  - R: `smd-player/addons/exmateria_sound/runtime/shared/dispatcher.gd:494-496` (arm gated on `_cb_porta_was_active` captured pre-decrement at dispatcher.gd:237; the post-snapshot `_recompute_pitch_staging` arm, gated on `porta_was_active && cadence_fired`, remains the setter on non-deactivation ticks) — validated by the post-fix parity re-run pairs above
  - src: `research/effect_sound/working_documents/PROBE_WALKER_FLAG_WORD_ENTRY_924_923.md`

## Notes

(empty — user territory)

## Related

- [[SPU Voice Engine]]
- [[PSX Pitch Conversion]]
- [[MIPS SPU Interleaving]]
- [[Cure 4 Audio Parity]]
- [[Effect Sound Timing]]
- [[Chan Base Offset Convention]]
- [[SFX Index]]
