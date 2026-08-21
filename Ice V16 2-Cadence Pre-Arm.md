# Ice V16 2-Cadence Pre-Arm

The ice (E024) V16 cos_dist outlier on `ice_no_music` (0.0259, the largest of the four voices in the cad-104 KON mask burst `0x370000`) was traced on 2026-05-21 to a PCSX mechanism the Godot reimplementation does not reproduce: two cadences before the multi-voice KON burst, the SMD interpreter primes every voice that will be keyed on — all four of 16/17/20/21 in one IRQ — with a WAVESET-bank park triplet (start_addr=0x1010, repeat_addr=0x30, pitch=slot residue), then the 0xAC Instrument opcodes resolve the real ADPCM addresses at cad 103, the real addresses plus an intentional pitch=0 silent prelude (V16) land at the cad-104 KON, and the first audible pitch (9329) begins at cad 106. Godot instead collapses this to a single cad-104 arm out of its SPU-init pitch=1 state. As of the doc date the fix (prime fan-out in the Godot dispatcher + pitch reset in the 0xAC handler) was in Planning status and is not implemented in the repo (2026-08-20 probe); the emitting opcode of the cad-102 walker bits (0x008|0x100) is still open — 0xAC is ruled out (it fires at cad 103) and play_sound is ruled out (fires at cad 1 / later), leaving the bytecode preamble (0xBA ReverbOn / 0xC4 / 0xD4) as the leading candidate.

## Points

- **PCSX primes all four voices that enter the cad-104 KON mask burst `0x370000` (16, 17, 20, 21) at cadence 102 in a single IRQ, writing start_addr=0x1010 and repeat_addr=0x30 to each — a mask-batched prime two cadences before the real arm, not a per-voice oddity (the cad-1 savestate-replay arms of voices 18/19/20/21 are separate and not part of the pattern).** — `[S·D] 2/3`
  - S: `spu_write_voice_start_addr` (FUN_8001B6A4, probe BP `0x8001B6A4`) and `spu_write_voice_repeat_addr` (probe BP `0x8001B720`) — `scus_disassembly.txt`
  - D: `probe_sample_start_addr_register` + `probe_sample_repeat_addr_register` — four rows of addr=4112/48 at cadence 102, one per voice 16/17/20/21 (`ice_no_music`, 2026-05-21)
  - R: none — the cad-102 prime not present in smd-player or godot-learning (probed: no pre-arm/prime handler or prime-loop constant; doc §4.2 code audit confirms no path pre-writes kRamInstrumentBase)
  - src: `research/effect_sound/working_documents/ICE_V16_TWO_CAD_PRE_ARM_FIX_PLAN.md`
- **The prime parks each voice on the WAVESET ADPCM bank: 0x1010 is the SPU-RAM WAVESET bank base (verified via PCSX-Redux readSpuRam), 0x30 = 3 × 16-byte ADPCM blocks, so under the LOOP_REPEAT flag the SPU decodes the first block and loops back at 0x30 — a silent in-place loop (env_vol≈0 from the prior spell's Release tail) that leaves the voice in a steady decode state before the cad-104 real-address retarget, without an interpolator transient.** — `[D] 1/3`
  - D: `ice_no_music` probe traces (2026-05-21) + bank-base readSpuRam verification recorded in `smd-player/src/shared/fft_spu_core_runtime.h` (kRamInstrumentBase comment)
  - R: none — the 0x30 prime loop not present in smd-player or godot-learning (probed; only the `kRamInstrumentBase = 0x1010` constant exists at `smd-player/src/shared/fft_spu_core_runtime.h:32`)
  - src: `research/effect_sound/working_documents/ICE_V16_TWO_CAD_PRE_ARM_FIX_PLAN.md`
- **The prime's pitch is a slot residue written deterministically by PCSX: cad 102 writes pitch=5096 to V16 and pitch=148 to V17, then the cad-104 KON cadence carries an intentional pitch=0 silent prelude on V16 (V17 gets pitch=1625), and the first audible pitch 9329 lands at cad 106 — Godot instead holds pitch=1 from SPU init across its KON, with the 9329 ramp beginning ~2 cads after KON.** — `[S·D] 2/3`
  - S: `spu_write_voice_pitch` (FUN_8001B628, probe BP `0x8001B628`) — `scus_disassembly.txt`
  - D: `probe_pitch_register` (PCSX cad 102/104/106 rows) + Godot `spu_voice_events.jsonl` V16 (pitch=1 from sample 69640 through the KON at 69849, 9329 at sample 70402) — `ice_no_music` (2026-05-21)
  - R: none — no intentional pitch=0 silent-prelude write in smd-player or godot-learning (probed; `smd-player/addons/exmateria_sound/runtime/shared/per_tick/pitch_staging.gd` carries slot residue only)
  - src: `research/effect_sound/working_documents/ICE_V16_TWO_CAD_PRE_ARM_FIX_PLAN.md`
- **At cadence 103 — one cadence after the prime — the 0xAC Instrument opcode dispatches four times (instrument bytes 115, 114, 119, 117), resolving the real ADPCM start/repeat addresses for the four voices, which are then committed together with the KON mask `0x370000` at cadence 104.** — `[S·D] 2/3`
  - S: `Hyp_smd_op_instrument` (0xAC handler, PC `0x80015E30`, XREF opcode table `0x80028BBC`) — `scus_disassembly.txt`
  - D: `probe_opcode_instrument` cad-103 rows (`ice_no_music`, 2026-05-21)
  - R: none — the cad-103 4×0xAC burst is PCSX-side timing; smd-player and godot-learning probed (0xAC handler present at `smd-player/addons/exmateria_sound/runtime/shared/opcodes/instrument.gd` but fires at a different Godot cadence — see [[Effect Sound Timing]])
  - src: `research/effect_sound/working_documents/ICE_V16_TWO_CAD_PRE_ARM_FIX_PLAN.md`
- **Godot has no prime: on `ice_no_music` V16's KON (sample 69849) carries the resolved real start_addr=442176 directly with pitch=1, and no start_addr=4112 event exists anywhere in the Godot trace — the 14/14 `probe_sample_start_addr_register` pair count masks the structural divergence (4 PCSX prime writes missing, 4 other Godot writes drifting in late) — and V16's audible onset is sample 531 (12.0 ms) vs PCSX's 832 (18.9 ms), aligned-window time-domain correlation 0.21, FFT-magnitude cosine 0.83 (the "right notes, wrong timing" signature).** — `[D·R] 2/3`
  - D: Godot `spu_voice_events.jsonl` V16 + `spu_voice_16.wav` onset/correlation diff (`ice_no_music`, 2026-05-21)
  - R: `smd-player/addons/exmateria_sound/runtime/shared/opcodes/instrument.gd` (0xAC writes the resolved final address immediately, no kRamInstrumentBase pre-write) + `runtime/shared/spu_irq_walker.gd::_fan_sample_addr` (commits on the next walker pass — the same IRQ as the KON) — divergence confirmed by the 2026-05-21 `ice_no_music` trace pair
  - src: `research/effect_sound/working_documents/ICE_V16_TWO_CAD_PRE_ARM_FIX_PLAN.md`
- **The 2-cadence pre-arm gap affects voices 16/17/20/21 identically; only V16 shows up in cos_dist (0.0259 vs 0.000/0.002/0.014 for V17/V20/V21) because its fast descending-pitch envelope makes it the lowest-abs_mean voice in the `0x370000` group (874.7 vs 6847.5 for V18), and the early-onset high-frequency divergence skews the time-averaged-spectrum cosine disproportionately.** — `[D] 1/3`
  - D: `ice_no_music` per-voice abs_max/abs_mean/cos_dist metrics (`ice_no_music`, 2026-05-21)
  - R: none — scoring observation, no implementation involved; smd-player and godot-learning probed
  - src: `research/effect_sound/working_documents/ICE_V16_TWO_CAD_PRE_ARM_FIX_PLAN.md`

## Notes

(empty — user territory)

## Related

- [[Savestate Residue Voice]]
- [[KON KOFF Mask Dispatch]]
- [[Effect Sound Audio Divergence]]
- [[Effect Sound Timing]]
- [[SPU Voice Engine]]
- [[Effect Sound Slot Allocator]]
- [[SFX Index]]
