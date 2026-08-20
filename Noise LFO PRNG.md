# Noise LFO PRNG

FFT's shared effect-engine PRNG (xorshift on `DAT_80032a18`, `FUN_800178F4`) and how the D9-armed noise-waveform pitch LFOs (wf_idx 6/7, callbacks `FUN_8001780C`/`FUN_80017878`) consume it through `lfo_handler_tick`. The 2026-05-15 cure_4_no_music investigation (CAD_497) pinned the handler's outer gate (`chan_word_0 != 0` at PC 0x800174D0), the per-firing advance count, and the savestate seeding (`engine_prng_state`), and closed the cadence-497 PRNG desync in smd-player by replacing a never-cleared Godot `slot.lfo_active` gate with the FFT `channel_word_0` gate. After the fix, cure_4's noise-LFO voices (18/19) match PCSX bit-exact on pitch_bend.

## Points

- **FFT's `lfo_handler_tick` (0x800174C8–0x800175E0) walks every chan-pool slot on every cadence but fires the sub-slot-0 LFO callback only when `chan_word_0 != 0` (PC 0x800174D0: `beq v0, zero, LAB_800175f4`) and the sub-slot-0 active bit `chan+0xFE & 0x1` is set (PC 0x800174EC) — EndBar clears `chan_word_0`, so once a channel's bytecode terminates the LFO is disarmed, PRNG advance included.** — `[S·D·R] 3/3`
  - S: PC `0x800174C8`–`0x800175E0` (outer iterator), `0x800174D0` (`beq → LAB_800175f4`), `0x800174EC` — disassembly cited in the doc (§5.1, §8.1)
  - D: cure_4_no_music `probe_lfo_handler_entry` (PCSX BP @ `0x800174C8`) — slot 3 gate_pass=1 rows 264 (PCSX) vs 266 (Godot; +2 are gate_pass=0 rows) (2026-05-15)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd` outer gate `channel.channel_word_0 != 0 and channel.lfo_sub_active[0] != 0` (replaces the never-cleared `slot.lfo_active` gate, doc §8.2) + `probe_pitch_inputs` pair (BP `0x80017368`, `smd-player/workspace/orchestrator/probe_validation_manifest.py`)
  - src: `research/effect_sound/working_documents/CAD_497_WF_IDX_6_PRNG_DESYNC.md`
- **FFT's effect-engine PRNG is a 32-bit xorshift at `FUN_800178F4` (0x800178F4–0x8001791C) on state `DAT_80032a18`; the noise-waveform pitch-LFO callbacks (wf_idx 6 = `FUN_8001780C`, wf_idx 7 = `FUN_80017878`, selected by D9 param[2]'s low nibble through jumptable `PTR_LAB_80028F54` — 9 entries, wf_idx 0–7 + default) each consume one PRNG value per firing, and wf_idx 6 computes `acc = (step_src >> 15) * prng` with `>> 15` a signed arithmetic shift (PC 0x80017850: `sra v1, v1, 0xf`) — e.g. `D9 04 D7 06`: byte1 0xD7 = −41 signed, rate² = 1681, sign-preserved lo = −1681, step_src = −1681 << 14 = −27,541,504.** — `[S·D·R] 3/3`
  - S: `FUN_8001780C`/`FUN_80017878`/`FUN_800178F4` ranges, PC `0x80017850`, `DAT_80032a18`, `PTR_LAB_80028F54` — disassembly cited in the doc (§3 H4, §6 Q3, §7)
  - D: savestate seed `engine_prng_state = 0x71A4E3C8` (`extract_noise_seed.py`, 2026-05-15); first 5 xorshift outputs of a standalone GDScript driver match the Python reference port; `probe_pitch_inputs` voice 19 bit-exact call_index 1..501 (cadence 0..419) (2026-05-15)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/helpers/lfo_prng.gd` (three-step xorshift port of `FUN_800178F4`) + noise-callback math in `runtime/shared/per_tick/advance_lfo.gd` (wf_idx 6: `sra_s32(step_src, 15) * prng`) — validated by the `probe_pitch_inputs` pair (BP `0x80017368`)
  - src: `research/effect_sound/working_documents/CAD_497_WF_IDX_6_PRNG_DESYNC.md`
- **A noise-LFO callback firing advances the engine PRNG once unconditionally, and a second time only when its internal countdown wraps — so when two wf_idx=6 LFOs arm on the same cadence (cadence 497 on cure_4_no_music) the PRNG advances 4× (2 LFOs × unconditional + first-tick conditional); the full 7 s cure_4_no_music capture totals 317 PRNG advances pre-fix (312 post-gate-fix).** — `[D·R] 2/3`
  - D: per-cadence PRNG-advance trace via the `_diag_lfo_prng_step` static emit (cad 497: 4 advances; cad 422 EndBar: 0; total 317 pre-fix) (2026-05-15)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/per_tick/advance_lfo.gd` (wf_idx 6: `_LfoPrng.step()` unconditionally at the top of the callback + a second step on countdown-wrap reload) — validated by the post-gate-fix `probe_pitch_inputs` bit-exact voice-18/19 sets (doc §8.3, 2026-05-15)
  - src: `research/effect_sound/working_documents/CAD_497_WF_IDX_6_PRNG_DESYNC.md`
- **Both engines seed the noise-LFO PRNG from the PCSX savestate's `DAT_80032a18` (0x71A4E3C8 on cure_4), so PCSX and Godot draw the same xorshift sequence — the seed is loaded on both sides from the `engine_prng_state` u32 emitted by the savestate extractor, and the shared-sequence claim is conclusive via voice-19 pitch_bend bit-exact over 501 consecutive rows.** — `[D·R] 2/3`
  - D: savestate seed extraction `extract_noise_seed.py` → `engine_prng_state = 0x71A4E3C8` + `probe_pitch_inputs` voice-19 match through call_index 501 (2026-05-15)
  - R: `smd-player/workspace/harness/render_effect_sound.gd` seeds `SharedLfoPrng.set_state` at session start + `runtime/shared/helpers/lfo_prng.gd` — validated by the `probe_pitch_inputs` pair
  - src: `research/effect_sound/working_documents/CAD_497_WF_IDX_6_PRNG_DESYNC.md`

## Notes

(empty — user territory)

## Related

- [[SFX Index]]
- [[SPU Voice Engine]]
- [[SMD Opcodes]]
- [[Cure 4 Audio Parity]]
- [[Effect Sound Audio Divergence]]
- [[FEDS Sound Definition Format]]
